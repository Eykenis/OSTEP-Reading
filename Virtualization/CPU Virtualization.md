# 虚拟化 CPU

如何对 CPU 进行虚拟化，来创造有很多 CPU 可用的假象？

对于 CPU 来说，通常使用的是**时分共享**机制，让一个实体使用 CPU 一小段时间，然后再切换到另一个实体，如此往复，CPU 资源就可以被多个实体共享。使用 CPU 的程序通常会被抽象为**进程**。进程拥有它的生命周期，拥有**创建、销毁、等待、其他控制和状态五个 API**，并且在其生命周期拥有**运行、就绪、阻塞三个状态**。

一个程序是如何作为进程完整地在 CPU 上完成其工作的呢？

- 首先，程序的存储结构是 代码-静态数据。这个结构将被加载到内存上去（通常一个程序不会被完整地加载，而这是内存虚拟化导致的结果），并为其分配相应的堆和栈空间。想一想，在学习程序设计的时候，是否提到过堆和栈这两种内存空间，例如 C 中的栈局部变量以及通过 `malloc()` 获取的堆空间？它们的当然是需要被分配的。除此之外，程序还有一个重要的功能就是相应 I/O. 因此还需要设置一些其他工作来准备 I/O. 例如在 UNIX 系统中的文件描述符。
- 这一切都准备好后，操作系统将 CPU 的控制权移交给程序，一个进程由此被创建，并进入了就绪状态。
- 就绪的进程在调度程序的调度下进入运行状态，而在遇到 I/O 时，它将等待 I/O 完成（除了 I/O 也可能是其他需要等待的事件），等待时，进程将进入阻塞状态。当请求完成后，进程将回到就绪状态或者运行状态。如果一个**正在运行**的进程被取消调度，那么它也将回到就绪状态。
- 当程序运行完其所有的代码，进程就会开始等待被清理。清理后，进程的生命周期就可以正常结束了。