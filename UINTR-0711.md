# Skyloft: A General High-Efficient Scheduling Framework  in User Space

Skyloft 如何在用户态处理硬件定时器中断。

第一步是将定时器中断向量配置至 UINV（User Interrupt Vector Register，用户中断向量寄存器）​ 中，使核心能够将定时器中断识别为用户中断。然而当定时器事件发生时，PIR（Posted Interrupt Request，投递中断请求寄存器）并不会自动更新；由于 PIR 为空，CPU便不会触发用户中断处理流程。

因此，第二步——更新 PIR——虽然可利用 SENDUIPI指令更新 PIR，但这会不必要地生成一个处理器间中断（IPI）。发现 UPID（User Posted Interrupt Descriptor，用户投递中断描述符）​ 中有一个名为 SN（Suppress Notification，抑制通知）​ 的控制位，可用于阻止实际 IPI 的生成。基于此，Skyloft 让每个核心向自身发送一个设置了 SN 位的 IPI，从而在不触发真实 IPI 的前提下，有效地更新了 PIR。

（1）初始化每个线程的 UPID（用户投递中断描述符）​ 并设置 SN（抑制通知）位；

（2）执行 SENDUIPI指令，向 PIR（投递中断请求寄存器）​ 写入非空值（使用任意中断号），从而使首个硬件中断能在用户态得到处理；

（3）进入中断处理函数后，再次执行 SENDUIPI，以确保后续硬件中断均在用户态处理。

缺点：这样普通的进程发送用户态中断就失效了。

和Aeolia的区别：Aeolia 是通过把UPID直接mmap到用户态空间，然后手动写PIR。Skyloft是利用senduipi指令写PIR。

和Aeolia的好处是用户态中断可以复用，而不是只能给Timer用。

同时对比了Aspen，分析Aspen使用一个单独的核专门运行timer thread向其他核发送用户态中断的缺点。