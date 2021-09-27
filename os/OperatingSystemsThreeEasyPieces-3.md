# Virtualization \| CPU

> 2021.09.22 \| 《Operating Systems: Three Easy Pieces》- 3


虚拟化CPU，另看似同时运行的多个作业之间共享物理CPU。

必须保证 **performance** 和 **control**。前者表示CPU的利用率，后者表示需要控制进程的权限与资源。

## Limited Direct Execution

直接在CPU上运行程序。

在进程列表中创建一个process entry，为其分配内存，从磁盘中将代码加载到内存，找到main()入口开始执行；完毕后执行return，释放内存，将进程从列表中删除。

问题：
1. ***Restricted Operations***: 怎么保证程序在运行时，不执行不被允许的操作？
2. ***Switching Between Processes***: 怎么将当前程序停止，并运行其他程序？

### Restricted Operations

通过两种模式来进行权限控制

1. **User Mode**: 运行的代码功能受限。
2. **Kernal Mode**: 允许执行一些特权命令，例如I/O。

如果 **User Mode** 的进程想进行特权操作，需要执行系统调用，即 **system call**

要执行 **system call**，进程需要执行一条特殊的 **trap** 指令，**trap** 指令会跳转到内核并将对应的权限提高到 **Kernal Mode**。**system call** 结束后，操作系统调用 **return from trap** 指令，返回到用户进程并将权限降低到 **User Mode**。

在系统最开始启动时，操作系统会设置 **trap table**，记录每个 **trap** 对应的 **trap handler**，并为每个系统调用分配一个 **system-call number**。

当执行 **system call** 时，用户代码需要将 **system-call number** 放入指定的位置（寄存器或堆栈），OS在内核模式时，会检查是否有效。通过这种方式，保证用户代码不知道具体的 **handler** 地址，无法直接跳转指令。

### Switching Between Processes

当一个进程在CPU上运行时，**OS**（可以被视为另一个进程）是没有在运行的。

那么 **OS** 是怎么获得 **CPU** 运行权的呢？

1. **OS** 等待进程中断，进程主动发起 **trap** 指令，将 CPU 交给 **OS**。

> 一旦进程不主动释放CPU资源，OS永远无法运行。只能重启机器。

2. 定时中断 (timer interrupt)。一旦中断发生，**OS** 会获得 CPU 的运行权限，可以切换进程。

> 在启动阶段，OS 需要初始化定时器；并告知硬件当计时器中断发生时，需要运行哪段代码。

#### 上下文切换（Context Switch）

切换进程时，将上一个进程的寄存器等保存 ，并恢复下一个进程的寄存器等内容。

在进程切换期间，有两种类型的寄存器保存/恢复。

1. 在定时器中断发生时，进程的用户寄存器由硬件使用该进程的内核堆栈隐式保存。

> 即由用户态切换到内核态时，保存用户态

2. 当操作系统决定从进程A切换到进程B时，内核寄存器由软件（即操作系统）显式保存，保存到进程的进程结构中的内存中。

> 进程切换时，保存进程的上下文