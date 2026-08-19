# libstorage

libstorage.so 动态链接库提供基本的NVMe读写API。

```c
Aeolia-src/AeoDriver/libstorage/libstorage.c

1	open_device	ioctl 打开 NVMe 设备，拿到 nsid、lba_shift 等
2	register_uintr	注册 MSI-X、syscall 绑用户 handler、mmap UPID
3	read_blk 阻塞读，POLLING 自旋 / UINTR 靠中断回调
4	read_blk_async 非阻塞读，完成调 callback
5	ls_sched_yield	自定义让出 CPU
6	ls_qpair_process_completions 扫 CQ + 触发 callback
7	create_qp	建 SQ/CQ、mmap SQE/CQE/DB、配 iod/req_buf
8	delete_qp	撤销 QP

```

## 示例程序

```c
Aeolia-src/AeoDriver/apps/test/test1_sync_polling.c

int main()
{
	strcpy(dev_args.path, "/dev/nvme0");
	ret = open_device(&dev_args, &nvme); 
	ret = register_uintr(&dev_args, &nvme);
	
	return 0;
}

static void async_callback(void *data)
{
	cb_cnt++;
}

static void test_read_write(int upid_idx)
{
	
	qp_args.interrupt = LS_INTERRPUT_UINTR;
	ret = create_qp(&qp_args, &nvme, &qp);

	for (int i = 0; i < batch; i++) {
		ret = write_blk_async(&qp, HUGE_PAGE_SIZE + iosize * i, iosize,
				      wrbuf + i, async_callback, NULL);
	}
    // busy loop 轮询等待完成
	while (cb_cnt < batch) {
	}

	cb_cnt = 0;
	printf("write success\n");
	for (int i = 0; i < batch; i++) {
		ret = read_blk_async(&qp, HUGE_PAGE_SIZE + iosize * i, iosize,
				     rdbuf + i, async_callback, NULL);
		if (ret) {
			printf("failed to send req!\n");
		}
	}
    // busy loop 轮询等待完成
	while (cb_cnt < batch) {
	}

    // 校验结果
	for (int i = 0; i < batch; i++) {
		char *buf = (char *)rdbuf[i].buf;
		for (int j = 0; j < iosize; j++) {
			byte_cnt += (buf[j] == 0x2f);
			assert(buf[j] == 0x2f);
		}
	}
	printf("%d. Successfully read/write block, byte_cnt : %d, expected : %d\n",
	       ++test_cnt, byte_cnt, batch * iosize);

}
```

## open_device

open_device 是libstorage动态链接库提供的API，通过 ioctl与内核模块 （ libstorage_kernel.ko ）进行通信。

打开控制设备 /dev/libdriver，是内核模块 libstorage_kernel.ko 创建的字符设备，用于发 ioctl。

在模块加载时候创建的该设备 (ls_init() -> ls_init_cdev())。


```c
Aeolia-src/AeoDriver/libstorage/libstorage.c

int open_device(ls_device_args *args, ls_nvme_dev *nvme)
{
    // 1) 打开字符设备 /dev/libdriver（全局只开一次，fd 缓存到 ls_dev_fd）
    if (!ls_dev_fd) {
        ls_dev_fd = open("/dev/" DEVICE_NAME, O_RDWR);   // DEVICE_NAME="libdriver"
    }

    // 2) 把用户态中断处理函数地址塞进 args，随 ioctl 一起传给内核
    args->uintr_handler = (void *)ls_nvme_irq_user_handler;

    // 3)ioctl_open_device 宏展开后 使用字符设备 /dev/libdriver的 ioctl发送 LIBSTORAGE_OPEN_DEVICE命令，执行libstorage_kernel.ko的ls_ioctl()
    // ls_ioctl 根据参数LIBSTORAGE_OPEN_DEVICE，调用ls_open_device()，最后path_to_nid("/dev/nvme0")  打开传入的块设备路径 nvme
    // 打开以后还会ls_nvme_set_irq 为当前打开的 NVMe 设备实例分配并注册一个内核态 MSI-X 中，用于任务已 sched_yield 让出 CPU 时的唤醒。注册的中断处理函数的ls_nvme_irq_user_handler
    // ls_open_device()然后调用ls_map_nvme()创建 UPID 字符设备/dev/ls_nvme/nvme%dinstance%d （每次open都对应一个instance），每个 instance 占 4 个次设备号位（预留给 UPID + CQ + SQ + DB 共 4 个字符设备），UPID 设备	/dev/ls_nvme/nvmeXinstanceY CQ 设备	/dev/ls_nvme/cqX_Y SQ 设备	/dev/ls_nvme/sqX_Y DB 设备	/dev/ls_nvme/dbX_Y
    ret = ioctl_open_device(args);    // 宏: ioctl(ls_dev_fd, LIBSTORAGE_OPEN_DEVICE, args)

    // 4) 内核返回后，args 被 copy_to_user 回填（含 instance_id/nid/nsid/lba_shift...）
    //    用回填的字段构造用户态 ls_nvme_dev 结构
    create_nvme_info_from_args(args, nvme);    // L48-L57
    // 打开内核为该 instance 创建的 UPID 字符设备/dev/ls_nvme/nvme%dinstance%d，并 mmap 一页（64 字节对齐的 uintr_upid 数组），让 用户态和内核共享同一块 UPID 内存 ——这是 UINTR 投递中断的物理基础。
    create_uintr_info_from_args(args, nvme);   // L27-L47
    create_dma_for_nvme(nvme, args->instance_id); // L84-L105
}
```

```c
};
int ls_map_nvme(ls_nvme_dev_instance *instance, int instance_id)
{
	upid_dev = device_create(ls_class, NULL,
				 MKDEV(ls_cdev_major, instance_id * 4 + 4),
				 NULL, upid_name);
}

static void create_uintr_info_from_args(ls_device_args *args, ls_nvme_dev *nvme)
{
	nvme->upid_fd = open(path, O_RDWR);
	
	uintr_upid *upid_page = mmap(NULL, PAGE_SIZE, PROT_READ | PROT_WRITE,
				     MAP_SHARED, nvme->upid_fd, 0);
	nvme->upid_page = upid_page;
}

static int upid_mmap(struct file *filp, struct vm_area_struct *vma)
{
	// 把内核的upid_page 映射的用户态内存空间，
    // struct uintr_upid *upid_page;  upid_page 内存由init_upid_mem申请的4KB页，由libstorage_kernel.ko初始化的时候，调用ls_init_uintr，之后调用init_upid_mem 申请和初始化的内存
    // EXPORT_SYMBOL(upid_page);
	return io_remap_pfn_range(vma, vma->vm_start,
				  virt_to_phys(upid_page) >> PAGE_SHIFT,
				  map_size, vma->vm_page_prot);
}
```

do_uintr_register_irq_handler() 在注册的时候，从upid_page中申请一个upid。upid_page 是全局的，所有进程共享一个4KB页。
```c
static struct uintr_upid *alloc_upid(void)
{
	if (!upid_page)
		return NULL;
	int upid_idx = atomic_inc_return(&upid_alloc_cnt);
	upid_idx--;
	return upid_page + upid_idx;
}
```

## register_uintr

register_uintr通过ioctl调用ls_reg_uintr()
```c
int register_uintr(ls_device_args *args, ls_nvme_dev *nvme)
{
	int ret = ioctl_register_uintr(args);
	nvme->upid = (uintr_upid *)nvme->upid_page + args->upid_idx;
	_stui();
	return 0;
}

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

## read_blk_async

构造req提交到SQ后返回，callback将在 用户态中断处理程序中被调用。
```c
int read_blk_async(ls_nvme_qp *qp, uint64_t lba, uint32_t lba_count,
		   dma_buffer *buf, void (*callback)(void *), void *data)
{
	io_request *req = ls_prepare_io_request(qp, lba, lba_count, buf->buf,
						buf->pa, callback, data,
						nvme_cmd_read);

	ret = ls_submit_req(qp, req);
	return ret;
}
```

## read_blk

同步读取，调用的是read_blk_async 异步提交请求，但是callback 使用的是全局的 ls_sync_default_callback，提交完请求就busy loop等待。
```c
int read_blk(ls_nvme_qp *qp, uint64_t lba, uint32_t count, dma_buffer *buf)
{
	read_blk_async(qp, lba, count, buf, ls_sync_default_callback, NULL);

    while (received_reqs < total_reqs) {
        // 如果采用Pooling方法，那么需要自己主动轮询CQ状态
		if (qp->interrupt != LS_INTERRPUT_UINTR)
			ls_qpair_process_completions(qp, 0);
	}
	return ret;
}

// ls_sync_default_callback本质上就是一个计数，使得read blk的busy loop能够退出
static void ls_sync_default_callback(void *data)
{
	received_reqs++;
}
// *
```

## ls_sched_yield

ioctl_sched_yield 通过libdriver的 ioctl调用kern_ls_sched_out()

```c
Aeolia-src/AeoDriver/libstorage/libstorage.c

int ls_sched_yield(ls_device_args *args)
{
	ioctl_sched_yield(args);
}

Aeolia-src/AeoDriver/kernel-module/core.c
void kern_ls_sched_out(void __user *__args)
{
	ret = copy_from_user(&args, __args, sizeof(ls_device_args));

	instance = nvme_instances + args.instance_id;
	WRITE_ONCE(instance->in_waiting, 1);

	ls_nvme_qp *qp = find_qp_by_qid(instance->nvme, instance->qid);

    // 禁止本 CPU 的所有外部中断 (包括当前 virq 的 MSI-X)，进入临界区
	local_irq_disable();

    // 关 UINTR Posted Interrupt,
	del_uinv();

    // 检查当前 CPU 上是否有 Posted User Interrupt Request 未被投递
	if (check_uirr() || READ_ONCE(current->thread.waiting_uintr)) {
		WRITE_ONCE(instance->in_waiting, 0);
		local_irq_enable();
		WRITE_ONCE(current->thread.waiting_uintr, 0);
		return;
	}
	if (READ_ONCE(instance->in_waiting)) {
		might_sleep();
		set_current_state(TASK_INTERRUPTIBLE);
		local_irq_enable();
		schedule();
	}
	WRITE_ONCE(current->thread.waiting_uintr, 0);
}

kernel/aeolia-kernel-uintr/arch/x86/kernel/uintr.c
void del_uinv(void)
{
	u64 misc_msr;
	rdmsrl(MSR_IA32_UINTR_MISC, misc_msr);
	misc_msr &= ~GENMASK_ULL(39, 32);
	wrmsrl(MSR_IA32_UINTR_MISC, misc_msr);
}

u64 check_uirr(void)
{
	u64 uirr;
	rdmsrl_safe(MSR_IA32_UINTR_RR, &uirr);
	return uirr;
}
```