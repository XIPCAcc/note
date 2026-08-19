# AeoDriver

AeoDriver编译生成 libstorage_kernel.ko 内核模块。

在主线 nvme.ko 驱动之上的旁路内核模块，不重写 NVMe 驱动，而是复用主线驱动已经枚举好的控制器、admin 队列、MSI-X 向量表，为用户态 libstorage.so 提供一套 ioctl 控制平面 ，字符设备 mmap 数据平面，让应用进程能直接把命令写进 SQ、直接从 CQ 取完成、通过 UINTR 硬件直通机制零 syscall 收到中断。把必须 ring0 才能做的事（申请 DMA 内存、配 MSI-X、注册 IRQ、创建字符设备 mmap）封装给用户态，而把真正的 I/O 提交/完成路径 100% 留在用户态。

## ls_init

Aeolia-src/AeoDriver/kernel-module/core.c

```c
static int __init ls_init(void)
{
    // 注册字符设备主 /dev/libdriver 节点
	ret = ls_init_cdev();
    // 初始化 QP 创建销毁互斥锁
	mutex_init(&qp_mutex);
    // 使用NVMe.ko 的API 枚举系统上所有 NVMe 设备，在内核里主动请求加载主线 nvme.ko 驱动模块，等价于在用户态执行 modprobe nvme
	ret = ls_nvme_probe();
    // 申请全局 4KB UPID 页
	ret = ls_init_uintr();

	ls_migrate_wq = alloc_workqueue("ls_migrate_wq", WQ_UNBOUND | WQ_HIGHPRI, 1);
	
}

kernel/aeolia-kernel-uintr/arch/x86/kernel/uintr.c
struct uintr_upid *init_upid_mem(void)
{
	upid_page = (struct uintr_upid *)get_zeroed_page(GFP_KERNEL);
	return upid_page;
}
EXPORT_SYMBOL(init_upid_mem);
```

## 控制平面

通过 /dev/libdriver 主字符设备的 ioctl提供。

核心职责是把用户态驱动做不了的 4 件事（NVMe 管理命令：申请 CQ/SQ 队列；中断注册：MSI-X virq + UINTR 绑定 + fallback 唤醒；内存分配：dma_alloc_coherent 一致性队列内存 + 全局 UPID 页；字符设备 + mmap：把 SQ/CQ/PCI Doorbell/UPID 4 段关键内存安全地映射到具体用户进程的页表）通过 7 条 ioctl 暴露给上层，复用主线 nvme.ko 已经初始化好的控制器与 admin 队列资源（寄生又兼容，不像 SPDK 那样需要 vfio-pci 接管设备、unbind 原驱动）。
真正的 I/O 提交（填 SQE + 写 doorbell）和 I/O 完成（读 CQE + callback）路径在 mmap 成功后完全不再进内核，配合 UINTR 硬件中断直通实现"零 syscall 零 kernel 路径"的超低延迟存储栈。

```c
static struct file_operations fops = {
	.owner = THIS_MODULE,
	.open = ls_open,
	.release = ls_release,
	.unlocked_ioctl = ls_ioctl,
};

static long ls_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
	int ret = 0;
	switch (cmd) {
	case LIBSTORAGE_OPEN_DEVICE:
		LOG_DEBUG("Open Device");
		ret = ls_open_device((void __user *)arg);
		break;
	
	case LIBSTORAGE_REG_UINTR:
		LOG_DEBUG("Register UINTR");
		ret = ls_reg_uintr((void __user *)arg);
		break;
	
	case LIBSTORAGE_SCHED_YIELD:
		LOG_DEBUG("Sched Yield");
		kern_ls_sched_out((void __user *)arg);
		break;
	
}
```

### ls_open_device

```c

static int ls_open_device(void __user *__args)
{
	ret = path_to_nid(args.path);

    // 分配 MSI-X 向量 + 注册内核 fallback IRQ handler
	ret = ls_nvme_set_irq(nvme, dev_instance, &args);
    //  创建 /dev/ls_nvme/nvme0instance1 等节点，用于后续调用该设备的mmmap映射UPID page到用户态
	ls_map_nvme(dev_instance, args.instance_id);
	
	ret = copy_to_user(__args, &args, sizeof(ls_device_args));
	
}
```

#### ls_nvme_set_irq

```c
int ls_nvme_set_irq(ls_nvme_dev *nvme, ls_nvme_dev_instance *instance,
		    ls_device_args *args)
{
    // NVME内部自己组织了一个中断地址空间，现在在找第一个空闲的中断号
	irq_vec = find_first_zero_bit(nvme->vec_bmap, nvme->vec_bmap_size) + 1;
	set_bit(irq_vec - 1, nvme->vec_bmap);
	instance->irq_vector = irq_vec;
	// 把空闲中断号转换为内核统一编制的 virq，将硬件层的 MSI-X 向量号 转换为 Linux 内核抽象的 虚拟 IRQ
    // MSI-X 向量号是 NVMe PCIe 设备层面的概念。设备通过写 PCIe MSI-X 表的某个 entry 来触发中断，每个 entry 对应一个向量号（从 0 开始编号）。
	instance->virq = pci_irq_vector(nvme->pdev, irq_vec);
	
	ret = ls_register_irq(nvme, instance, args);
}

static int ls_register_irq(ls_nvme_dev *nvme, ls_nvme_dev_instance *instance,
			   ls_device_args *args)
{
    // IRQF_SHARED 标志：允许同一 virq 被多个 handler 共享。因为主线 NVMe 驱动可能已经注册过该向量的 handler，AeoDriver 不能独占，必须用共享方式挂上自己的 handler。
	unsigned int flags = IRQF_SHARED; 
	instance->irq_name = kzalloc(PATH_SIZE, GFP_KERNEL);
    // 中断名格式为 nvme%duirq%d，在 /proc/interrupts 中可以看到，方便调试。
    sprintf(instance->irq_name, "nvme%duirq%d", nvme->id,
		instance->upid_idx);
    // Linux 标准 API，绑定 virq → handler（ls_nvme_irq_kernel_handler）。最后一个参数 instance 是传给 handler 的 dev_id，handler 里用它来判断"这次中断是否属于我的实例"。
	ret = request_irq(instance->virq, ls_nvme_irq_kernel_handler, flags,
			  instance->irq_name, instance);
}

// 用户态中断到达时候，如果用户态中断等待线程不在运行，那么中断被当成普通中断，执行ls_nvme_irq_kernel_handler 唤醒等待线程
static irqreturn_t ls_nvme_irq_kernel_handler(int irq, void *dev_instance)
{
	ls_nvme_dev_instance *instance = (ls_nvme_dev_instance *)dev_instance;
	ls_nvme_qp *qp = find_qp_by_qid(instance->nvme, instance->qid);
	
	struct task_struct *task = instance->task;
	
    // 中断处理唤醒task
	WRITE_ONCE(instance->in_waiting, 0);
	int ret = wake_up_process(task);
	WRITE_ONCE(task->thread.waiting_uintr, 1);
	task->thread.waiting_uintr = 1;

	if (task_nice(task) < task_nice(current)) {
		instance->num_uintr_kernel++;

		if (current->flags & PF_EXITING ||
		    current->flags & PF_SUPERPRIV) {
			LOG_DEBUG("Task %d is exiting", current->pid);
			return IRQ_HANDLED;
		}
        // 标记当前正在 CPU 上执行的进程（current）"需要被调度"，即触发一次调度点，让更高优先级的用户进程尽快获得 CPU。
		set_tsk_need_resched(current);
	}

	return IRQ_HANDLED;
}

```

#### ls_map_nvme

为一个设备实例（instance）创建并注册其专属的 UPID 字符设备，使得用户态可以通过 open() + mmap() 把内核的全局 upid_page 映射到自己的地址空间。    

```c

static struct file_operations upid_fops = {
	.owner = THIS_MODULE,
	.mmap = upid_mmap,
	.open = upid_open,
};
int ls_map_nvme(ls_nvme_dev_instance *instance, int instance_id)
{

    // ls_nvme/nvme%dinstance%d
	snprintf(upid_name, PATH_SIZE, LIBSTORAGE_UPID_PATH_FORMAT,
		 instance->nvme->id, instance_id);
	upid_dev = device_create(ls_class, NULL,
				 MKDEV(ls_cdev_major, instance_id * 4 + 4),
				 NULL, upid_name);
	

	cdev_init(&instance->upid_cdev, &upid_fops);
	instance->upid_cdev.owner = THIS_MODULE;
	ret = cdev_add(&instance->upid_cdev,
		       MKDEV(ls_cdev_major, instance_id * 4 + 4), 1);

}

static int upid_mmap(struct file *filp, struct vm_area_struct *vma)
{
    // upid_page映射到用户态空间
	vma->vm_page_prot = pgprot_noncached(vma->vm_page_prot);
	return io_remap_pfn_range(vma, vma->vm_start,
				  virt_to_phys(upid_page) >> PAGE_SHIFT,
				  map_size, vma->vm_page_prot);
}
```

### ls_reg_uintr

ioctl 接收LIBSTORAGE_REG_UINTR 参数后调用 ls_reg_uintr 注册用户态中断。

```c
// ls_reg_uintr()调用ls_nvme_register_uintr()绑定中断和注册用户态中断处理程序 Aeolia-src/AeoDriver/kernel-module/irq.c
int ls_reg_uintr(void __user *__args)
{
	dev_instance = nvme_instances + args.instance_id;

	nvme = nid_to_nvme(args.nid);

	ret = ls_nvme_register_uintr(nvme, dev_instance, &args);

	ret = copy_to_user(__args, &args, sizeof(ls_device_args));
}

int ls_nvme_register_uintr(ls_nvme_dev *nvme, ls_nvme_dev_instance *instance,
			   ls_device_args *args)
{
    // irq_data 描述一条具体的中断（比如 AeoDriver 申请的某个 virq）的元信息：
    // 这个 virq 对应哪个硬件中断号
    // 由哪个 irq_chip（中断控制器）管理
    // 通过 virq 找到对应的 irq_desc，再返回它的 irq_data 字段。
	struct irq_data *irq_data = irq_get_irq_data(instance->virq);
    // 目的是拿到 irq_chip，然后调它的 irq_set_affinity 方法。irq_chip 是一组函数指针表，描述"如何操作某种中断控制器"。每种中断控制器（IOAPIC、MSI-X、ARM GIC、GPIO 控制器）都注册自己的 irq_chip 实现。
	struct irq_chip *irq_chip = irq_data->chip;
	const struct cpumask *mask = cpumask_of(smp_processor_id());
    // 固定中断亲和性到当前 CPU 
	irq_chip->irq_set_affinity(irq_data, mask, true);
	set_cpus_allowed_ptr(current, mask);

	ret = do_uintr_register_irq_handler(args->uintr_handler,
					    instance->virq);
	instance->upid_idx = ret;
	args->upid_idx = ret;
	instance->owner_pid = current->pid;
	instance->task = current;

}

// kernel/aeolia-kernel-uintr/arch/x86/kernel/uintr.c
int do_uintr_register_irq_handler(u64 handler, u32 irq)
{
	uintr_pid = current->pid;

    // 关键就在于 get_vec_by_msi_irq(irq) 把NVMe的设置中断号当做uinv写入upid->nc.nv和msr_uintr_misc
	return do_uintr_register_handler(handler, get_vec_by_msi_irq(irq));
}
EXPORT_SYMBOL(do_uintr_register_irq_handler);

int do_uintr_register_handler(u64 handler, u8 uinv)
{
	upid_ctx = task->thread.upid_ctx;
	
	upid_ctx = alloc_upid_ctx();
	
	task->thread.upid_ctx = upid_ctx;

	xstate = start_update_xsave_msrs(XFEATURE_UINTR);
	upid = upid_ctx->upid;
	upid->nc.nv = uinv;
	upid->nc.ndst = cpu_to_ndst(smp_processor_id());
	g_vis_uinv = uinv;
	xsave_wrmsrl(xstate, MSR_IA32_UINTR_HANDLER, handler);
	xsave_wrmsrl(xstate, MSR_IA32_UINTR_PD, (u64)upid);
	xsave_rdmsrl(xstate, MSR_IA32_UINTR_MISC, &msr_uintr_misc);
	msr_uintr_misc |= (u64)uinv << 32;
	xsave_wrmsrl(xstate, MSR_IA32_UINTR_MISC, msr_uintr_misc);
	msr_uintr_stackadj = 256;
	xsave_wrmsrl(xstate, MSR_IA32_UINTR_STACKADJUST, msr_uintr_stackadj);
	task->thread.upid_activated = 1;
	end_update_xsave_msrs();

    // 手动把puir置为1
	upid->puir = 0x1;

    // 返回的是upid的下标，这里用减法计算出来
	return upid - (struct uintr_upid *)upid_page;
}
```

用户态中断处理程序的上下文负责读取NMVe CQ的数据，最重要的是puir手动重新置1
```c
// Aeolia-src/AeoDriver/libstorage/io.c
void __attribute__((interrupt))
ls_nvme_irq_user_handler(struct __uintr_frame *ui_frame, uint64_t vector)
{
	uint32_t pre_key = __rdpkru();
	LOG_DEBUG("IRQ handler, vector : %lu", vector);
	enter_protected_region(local_qp->in_trust);
	g_interrupt_times++;
	ls_qpair_process_completions(local_qp, 0);
	exit_protected_region(pre_key);
}

inline int ls_qpair_process_completions(ls_nvme_qp *qp, int maxn)
{
	if (qp->interrupt == LS_INTERRPUT_UINTR) {
		qp->nvme->upid->puir = 1;
	}
```
