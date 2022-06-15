# 持久性

计算机的持久性由处于存储器结构最底层的外部存储支持（在现代计算机中，外存之下还有一层存取速度更慢但空间更加庞大的存储级——网络存储）。实际上，外存，内存，包括高速缓存的很多存储策略都是类似的。但除了关心存储策略以外，还要着重理解设备交互的机制，知道外存是如何将自己的操作封装成一系列接口供软件使用的。