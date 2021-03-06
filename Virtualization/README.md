# 虚拟化

在开始了解虚拟化之前，需要搞清楚一些问题。

- **虚拟化是什么？**

	其实是把物理的 CPU 和内存加上一层抽象，对于有限的几个 CPU，创造一种 “你有很多 CPU 可以用” 的假象；对于有限的内存空间，创造一种 “你有多到几乎无穷尽的地址空间” 的假象。

- **为什么需要虚拟化？**

	操作系统为什么要这样自欺欺人呢？很显而易见的一个道理，通过这样的虚拟化之后，更高层次的程序，就不用再关心 CPU 现在能不能用，或是内存够不够用，所给的内存地址在物理内存中有没有这种问题了。它们都将交给操作系统统一处理，不需要其他程序单独去处理。实际上，书的第二页已经说明了，**Why the OS does this is not the main question, as the answer should be obvious: it makes the system easier to use.**

- **如何将资源虚拟化？**

	实际上这才是接下来我们最关心的，实践永远是最重要的。我们通过一系列虚拟化方法，物理资源抽象成为看起来更加强大的虚拟形式。实际上它们并没有真的变得更强大，反而因为一系列虚拟化过程性能降低了。但是却极大地降低了我们工作的难度，就好比我们需要高级语言来写现代程序，而不是使用汇编一样。本书主要介绍的是 CPU 和内存的虚拟化方法。

