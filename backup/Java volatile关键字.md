# 准备

在了解volatile关键字之前，必须先了解 [Java内存模型](https://huangxiaocheng93.github.io/blog/post/Java-nei-cun-mo-xing.html) 有关的知识，所以后续会默认在Java内存模型的基础上说明。

当一个变量被volatile关键字修饰时，它将具备两项特性：

- 保证此变量对所有线程的可见性；
- 禁止指令重排序优化；

# 实现原理

volatile变量的可见性是如何实现的呢？实际上是Java内存模型对volatile修饰的变量定义了一些特殊规则。

假定T表示一个线程，V和W分别表示两个volatile型变量，那么在进行read、load、use、assign、store和write操作时需要满足如下规则：

## 规则一

**只有当线程T对变量V执行的前一个动作是load的时候，线程T才能对变量V执行use动作；并且，只有当线程T对变量V执行的后一个动作是use的时候，线程T才能对变量V执行load动作。线程T对变量V的use动作可以认为是和线程T对变量V的load、read动作相关联的，必须连续且一起出现。**

用大白话来说，就是变量每次在use的时候必须先read和load，从主内存刷新最新的值，用于保证能看见其他线程对变量所做的修改。

## 规则二

**只有当线程T对变量V执行的前一个动作是assign的时候，线程T才能对变量V执行store动作；并且，只有当线程T对变量V执行的后一个动作是store的时候，线程T才能对变量V执行assign动作。线程T对变量V的assign动作可以认为是和线程T对变量V的store、write动作相关联的，必须连续且一起出现**。

用大白话来讲，就是变量在每次修改（assign）以后，必须马上store和write同步回主内存中，用于保证其他线程可以看到自己对变量所做的修改。

## 规则三

**假定动作A是线程T对变量V实施的use或assign动作，假定动作F是和动作A相关联的load或store动作，假定动作P是和动作F相应的对变量V的read或write动作；与此类似，假定动作B是线程T对变量W实施的use或assign动作，假定动作G是和动作B相关联的load或store动作，假定动作Q是和动作G相应的对变量W的read或write动作。如果A先于B，那么P先于Q。**

这条规则写的比较绕，简单解释，假设一个场景：线程T对变量V实施read → load → use操作，同时线程T对变量W也实施read → load → use操作；

这种情况下，如果是先read变量V，也一定会先use变量V，不会出现先read变量V，然后read → load → use变量W，然后再load → use变量W的情况。

如果不理解**指令重排**，可能很难理解这条规则在说什么，所以可以先简单理解这条规则要求volatile修饰的变量不会被指令重排序优化，从而保证代码的执行顺序与程序的顺序相同。

# 可见性

如果理解了上面的规则一和规则二就很容易理解可见性是怎么做到的了，线程1对变量V的更改会立即同步回主内存，同时线程2在使用变量V的时候每次都会从主内存读取，这样就保证了线程2一定能读取到线程1修改的数据。

## 硬件层面是如何实现的呢？

上面的解释都是在说Java内存模型层面的实现，那具体到硬件层面是如何实现的呢？这实际上就和CPU的内存一致性协议有关系了。

在volatile变量赋值之后，会使用一个CPU指令（IA32下是lock），它的作用是将本处理器的缓存写入了内存，该写入动作也会引起别的处理器或者别的内核无效化（Invalidate）其缓存，这种操作相当于对缓存中的变量做了一次前面介绍Java内存模式中所说的“store和write”操作。所以通过这样一个空操作，可让前面volatile变量的修改对其他处理器立即可见。
这条指令就是**内存屏障**。

## 可见就一定线程安全吗？

这里很容易误解，即使写操作对其他线程立即可见，也不代表就是线程安全的，是否线程安全取决于对变量的操作是不是**原子**的，如果对volatile变量实施的是原子操作，那这个操作就是线程安全的，但是很可惜，Java里面的操作并非原子操作，这导致volatile变量的运算在并发下一样是不安全的。

所以我们如果需要原子操作，即使变量被volatile修饰，也一定要加锁操作。

# 指令重排

指令重排是CPU的优化，指令重排的意思是，CPU可能会不按照程序编写的顺序执行指令，只保证最终的结果和顺序执行是一致的。CPU的这种优化措施的目的还是为了提高执行效率。

但是这种优化在多线程环境下可能就会有问题，借用《深入理解Java虚拟机》的示例：

```java
Map configOptions;
char[] configText;
// 此变量必须定义为volatile
volatile boolean initialized = false;

// 假设以下代码在线程A中执行
// 模拟读取配置信息，当读取完成后
// 将initialized设置为true,通知其他线程配置可用
configOptions = new HashMap();
configText = readConfigFile(fileName);
processConfigOptions(configText, configOptions);
initialized = true;

// 假设以下代码在线程B中执行
// 等待initialized为true，代表线程A已经把配置信息初始化完成
while (!initialized) {
    sleep();
}
// 使用线程A中初始化好的配置信息
doSomethingWithConfig();
```

假如线程A执行的代码被重排序，`initialized = true` 被放到了proess方法前面，可能就会导致配置没有初始化完成B线程就在读取配置了。这就是为什么volatile关键字会限制指令重排。