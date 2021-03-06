# 虚拟化 CPU

如何对 CPU 进行虚拟化，来创造有很多 CPU 可用的假象？

## 概念：进程

对于 CPU 来说，通常使用的是**时分共享**机制，让一个实体使用 CPU 一小段时间，然后再切换到另一个实体，如此往复，CPU 资源就可以被多个实体共享。使用 CPU 的程序通常会被抽象为**进程**。进程拥有它的生命周期，拥有**创建、销毁、等待、其他控制和状态五个 API**，并且在其生命周期拥有**运行、就绪、阻塞三个状态**。

> Process API
>
> • Create: An operating system must include some method to create new processes. When you type a command into the shell, or double-click on an application icon, the OS is invoked to create a new process to run the program you have indicated.
>
> • Destroy: As there is an interface for process creation, systems also provide an interface to destroy processes forcefully. Of course, many processes will run and just exit by themselves when complete; when they don' t, however, the user may wish to kill them, and thus an interface to halt a runaway process is quite useful.
>
> • Wait: Sometimes it is useful to wait for a process to stop running; thus some kind of waiting interface is often provided.
>
> • Miscellaneous Control: Other than killing or waiting for a process, there are sometimes other controls that are possible. For example, most operating systems provide some kind of method to suspend a process (stop it from running for a while) and then resume it (continue it running).
>
> • Status: There are usually interfaces to get some status information
> about a process as well, such as how long it has run for, or what
> state it is in.

一个程序是如何作为进程完整地在 CPU 上完成其工作的呢？

- 首先，程序的存储结构是 代码-静态数据。这个结构将被加载到内存上去（通常一个程序不会被完整地加载，而这是内存虚拟化导致的结果），并为其分配相应的堆和栈空间。想一想，在学习程序设计的时候，是否提到过堆和栈这两种内存空间，例如 C 中的栈局部变量以及通过 `malloc()` 获取的堆空间？它们当然是需要被分配的。除此之外，程序还有一个重要的功能就是响应 I/O. 因此还需要设置一些其他工作来准备 I/O. 例如在 UNIX 系统中的文件描述符。
- 这一切都准备好后，操作系统将 CPU 的控制权移交给程序，一个进程由此被创建，并进入了就绪状态。
- 就绪的进程在调度程序的调度下进入运行状态，而在遇到 I/O 时，它将等待 I/O 完成（除了 I/O 也可能是其他需要等待的事件），等待时，进程将进入阻塞状态。当请求完成后，进程将回到就绪状态或者运行状态。如果一个**正在运行**的进程被取消调度，那么它也将回到就绪状态。
- 当程序运行完其所有的代码，进程就会变为僵尸进程，开始等待被清理。清理后，进程的生命周期就可以正常结束了。

## 虚拟化机制：受限地直接执行

### Limited Direct Execution, problems, and solution

讨论完进程的概念，再来讨论一个关键问题：如何高效、可控地虚拟化 CPU？

一种最直接的机制是**直接执行**。就像之前进程的运行过程所描述的，我们直接在 CPU 上运行程序即可。操作系统将程序代码加载到内存中，当遇到需要执行的程序时，就跳到对应代码的入口处。

这样的好处是快速，但不妨考虑两个问题：

一是当进程希望进行 I/O 和其他操作时，我们将难以做出决定。如果完全允许 I/O，存储器将失去其安全性（没人知道进程会进行什么样的读写，有没有可能破坏存储器结构）

二是，我们如何进行切换？如果我们所有的进程都是按照顺序一个一个地执行，那么 CPU 虚拟化将失去其多核的假象。因此，什么时候进行进程切换，如何进行切换，也是我们需要考虑的重要问题。

所以，我们提出**受限的直接执行**。

#### Restricted Operations

一个解决特殊操作的权限问题的方法是为处理器设置**内核态**和**用户态**。在用户态下，进程不能进行 I/O；而当需要 I/O 的时候，处理器将切换到内核态，允许运行的代码进行 I/O 和其他特权操作。除此以外，在用户态下，也会为用户暴露一些系统调用的关键功能，例如文件访问，进程创建与销毁。我们规定，要执行这些调用，必须使程序执行 trap 指令，完成后，将进行 return trap. 中文我们通常称其为陷阱调用，执行 trap 指令的时候称为 “陷入” 内核态（不过翻译不重要）。

而且！在 trap 指令中，我们也不能想当然地像一般过程那样去进行地址跳转来执行相应代码，这样做同样将导致存储器失去保护！我们通常会使用一个 **trap table**, 这个 table 由操作系统设置，在每一次启动机器时由 OS 告知硬件。

#### Switching Between Process

在进程之间切换，显然是需要操作系统介入的。但是当一个程序在运行时，操作系统其实并没有在 CPU 上运行。所以我们需要想办法将控制权转交给操作系统。一种简单的想法是在进程内部进行系统调用来将控制权转交给 OS. 但如果进程不愿意呢（恶意进程）？因此我们采用**中断 (interrupt)** 机制。每隔一小段时间产生一个中断，停止当前运行的进程，并运行**中断处理程序**，以此来让操作系统定时决定进程切换。中断产生时，也要注意保存上下文。即之前运行的程序的一些临时变量，以及其执行的位置，等等。当中断结束后，重新加载保存的上下文，这样就可以继续运行进程了。

---

## 调度

简单来说，调度解决的就是 CPU 系统中断后，如何切换进程的问题。

为了比较各种调度算法，我们给出两个指标：

**响应时间**：进程到达队列和其被第一次运行的时间差。

**周转时间**：进程到达队列和其运行完成的时间差。

### FIFO

朴实无华，先进先出。

### Shortest Job First, SJF

最短任务优先，要注意其只是解决了 FIFO 中同时到达任务的顺序问题。即同时到达的任务中取运行时间最短的。但是如果一个长任务先于短任务到达，短任务仍然需要等到长任务执行完毕。

### Shortest Time-to-Completion First, STCF

与 SJF 不同，STCF 是一种**抢占式**的调度算法。每当新工作进入系统时，就会确定所有没有完成工作的剩余需要时间，并调度剩余时间最少的程序去运行。

> 个人哔哔：
>
> 为什么不添加一个时间片，每过一段时间就检查所有进程的运行时间，然后去运行剩余时间最短的进程呢？自我攻略：这样做效率几乎肯定是不如 RR 算法的。其很容易产生许多无用的时钟，即遍历了所有进程之后该运行的进程还是没变，浪费了诸多时间在遍历上。但，选择进程的时间开销如何呢？我不知道二者之间的数量级关系，因此也不好作判断。

### Round Robin

确定一个时间片，然后在未完成的进程之间按某个确定的顺序反复切换，直到所有进程完成。

### MLFQ

之前讨论的三种算法都只有一个进程队列。要继续优化不妨可以使用多队列。在 MLFQ 中，如果多个队列都有任务，总是优先执行优先级高的队列。当然，每个队列的优先级都是固定的。

具体来说，MLFQ 有以下规则：

1. 优先运行优先级高的队列中的任务
2. 如果任务的优先级相同，按照轮转 RR 模式运行
3. 工作进入系统时，默认是放在最高优先级队列的
4. 一旦工作用完了其在对应优先级的时间配额，就会降低其优先级
5. 经过一段时间，所有的工作都重新被放回最高优先级队列

使用 MLFQ 的一大目的就是区分出那些经常使用 I/O 和不常使用 I/O 的程序。通常，经常使用 I/O 的程序总是在阻塞状态，因此很少占用 CPU. 让它们在使用 CPU 时拥有高优先级是很有必要的，毕竟它们使用的少。相应地，如果 I/O 少，说明需要经常使用 CPU，这个时候降低其优先级即可。而为了防止程序通过 I/O 来恶意长时间获取高优先级（例如一个时间片只花很少时间进行象征性的 I/O，这会使其总是获得较高优先级，从而导致其他进程发生 “饥饿” ），我们为每个工作限定其配额，并且每隔一段时间就重置优先级（规则 5）。

### Lottery Scheduling

给每个进程一定量的彩票，然后每个时间片都进行一次抽奖。抽到的进程将获得这个时间片的使用权。例如 A 有 0\~74 共 75 张彩票，而 B 有 75\~100 共 25 张彩票，抽奖抽取一个 0\~100 的数，抽到彩票的值将决定运行 A 还是 B.
