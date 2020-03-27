# CPU

CPU 可压缩资源，CPU资源不足，应用不会直接挂掉，只是延时增加，体验感下降。

**CPU架构**

从系统架构来看，目前的商用服务器的CPU大体可以分为三类。

**SMP\(UMA\)**

对称多处理器结构\(SMP：Symmetric Multi-Processor\)

所谓对称多处理器结构，是指服务器中多个 CPU 对称工作，无主次或从属关系。各 CPU 共享相同的物理内存，每个 CPU 访问内存中的任何地址所需时间是相同的，因此 SMP 也被称为一致存储器访问结构 \(UMA ： Uniform. Memory Access\) 。 啧啧，NUMA

SMP 服务器的主要特征是共享，系统中所有资源 \(CPU 、内存、 I/O 等 \) 都是共享的。也正是由于这种特征，导致了 SMP 服务器的主要问题，那就是它的扩展能力非常有限（eg，cpu增加内存访问冲突会变多）。

当我们打开服务器的背板盖，如果发现有多个cpu的槽位，但是却连接到同一个内存插槽的位置，那一般就是smp架构的服务器，日常中常见的pc啊，笔记本啊，手机还有一些老的服务器都是这个架构。

> `ls /sys/devices/system/node/# 如果只看到一个node0 那就是smp架构`
>
> 0.0

实验证明， SMP 服务器 CPU 利用率最好的情况是 2 至 4 个 CPU 。

**NUMA（重点）**

非一致存储访问结构\(NUMA：Non-Uniform Memory Access\)

在NUMA架构出现前，CPU欢快的朝着频率越来越高的方向发展（现在已经被4GHz所限制）。受到物理极限的挑战，又转为核数越来越多的方向发展。

NUMA中，虽然内存直接attach在CPU上，但是由于内存被平均分配在了各个die上。只有当CPU访问自身直接attach内存对应的物理地址时，才会有较短的响应时间（后称`Local Access`）。而如果需要访问其他CPU attach的内存的数据时，就需要通过inter-connect通道访问，响应时间就相比之前变慢了（后称`Remote Access`）。所以NUMA（Non-Uniform Memory Access）就此得名。

![cpu-numa](file://D:/data_files/MarkDown/Images/cpu-numa.png?lastModify=1582441923)

由于这个特点，为了更好地发挥系统性能，开发应用程序时需要尽量减少不同CPU模块之间的信息交互。利用NUMA技术，可以较好地解决原来SMP系统的扩展问题，在一个物理服务器内可以支持上百个CPU。比较典型的NUMA服务器的例子包括HP的Superdome、SUN15K、IBMp690等。

0.0

**NUMA带来的问题**

上面提到了NUMA架构的CPU会导致内存分配不均匀，如何优化呢？？

Linux之父Linus针对这个问题给了建议：_既然CPU只有在Local-Access时响应时间才能有保障，那么我们就尽量把该CPU所要的数据集中在他local的内存中就OK啦~_

没错，事实上Linux识别到NUMA架构后，默认的内存分配方案就是：优先尝试在请求线程当前所处的CPU的Local内存上分配空间。如果local内存不足，优先淘汰local内存中无用的Page（Inactive，Unmapped）。





**CPU基本常识**

 [https://www.cnblogs.com/taiyonghai/p/7244878.html](https://www.cnblogs.com/taiyonghai/p/7244878.html)

**CPU个数**

CPU 的个数是指物理上，也就是硬件上存在着几颗物理 CPU，指的是真实存在的处理器的个数，1个代表1颗、2个代表2颗 CPU 处理器。

4路服务器即指有4个cpu

**CPU核心数**

 一个核心就是一个物理线程，**单核**、**双核**、**多核**，指的就是物理核心的数目。

eg： 个人PC一般是双核，高端点4核。服务器多核，比如4路服务器18核。

**CPU线程数**

 CPU 的线程数概念仅仅只针对 Intel 的 CPU 才有用，因为它是通过 Intel 超线程技术来实现的 。 线程数是一种逻辑的概念，简单地说，就是模拟出的 CPU 核心数。一个核心最少对应一个线程 。 英特尔的**超线程**技术可以把一个**物理线程模拟出两个线程**来用，充分发挥 CPU 性能，即一个核心可以有两个到多个线程。

0.0 超线程即 核数 \* 2

> 超线程： Intel Hyper-Threading Technology . 超线程就是 **通过调整AS而模拟出来的‘逻辑核（PU）’**。
>
> 超线程就是一个物理核里面，有两个AS，一个PU。两个AS共享一个PU。：
>
> PU: **Processing Unit（运算处理单元），简称PU**
>
> AS: **Architectual State（架构状态单元），简称AS**

\*\*\*\*

