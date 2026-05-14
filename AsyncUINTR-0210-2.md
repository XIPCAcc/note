
编译支持uintr-wait的linux

make menuconfig

然后在菜单里：

在Processor type and features  ---> 设置User Interrupts (UINTR) 

找到类似：

User Interrupts support (X86_USER_INTERRUPTS)
Support blocking for user interrupts (X86_UINTR_BLOCKING)
至少要把：

User Interrupts support 设为 [*]（= CONFIG_X86_USER_INTERRUPTS=y）
可选：

如果你需要 UINTR 能打断 read/sleep 等阻塞系统调用，再把

Support blocking for user interrupts 设为 [*]（= CONFIG_X86_UINTR_BLOCKING=y）
保存配置退出（.config 文件中就会写入这些项）。

Enabling support for blocking would allow system calls like read(),
      sleep() etc; to be interrupted when a user interrupt is initiated.
      This behavior is similar to the mechanism provided by signals.
      Note: ... experimental and flaky ...