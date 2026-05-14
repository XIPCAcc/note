# uintr_wait 返回值

syscall(__NR_UINTR_WAIT, usec, flags)
如果返回0，说明是没有收到用户态中断超时返回。如果返回-1，还有检查是否是因为收到了用户态中断，如果返回的是EINTR，那么说明是正确收到了用户态中断，而不是发生了错误
ret = uintr_wait(UINTR_WAIT_MAX_USEC, 0);
		printf("uintr_wait ret = %d\n", ret);

		if (ret == -1) {
			perror("uintr_wait");
			switch (errno) {
				case ENOSYS:
					printf("错误: CPU 或内核不支持 UINTR\n");
					break;
				case EINVAL:
					printf("错误: 参数无效\n");
					break;
				case EOPNOTSUPP:
					printf("错误: 进程未注册为 UINTR 接收者\n");
					break;
				case EINTR:
					printf("正确: 被信号中断\n");
					break;
			}
		}

注意参数需要包含 usec，接收到uintr返回值也是-1

注册handler的时候要有标志位
uintr_register_handler(client_ui_handler, UINTR_HANDLER_FLAG_WAITING_ANY)

# handler的编译

编译handler的标志位很重要
uintr-test/build.rs
fn main() {
    println!("cargo:rerun-if-changed=src/handler.c");
    
    cc::Build::new()
        .file("src/handler.c")
        .flag("-muintr")
        .flag("-O3")
        .flag("-std=gnu11")
        .compile("handler");
}