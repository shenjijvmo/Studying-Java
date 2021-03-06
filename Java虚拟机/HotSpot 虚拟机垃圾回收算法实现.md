> 作为使用范围最广的虚拟机之一HotSpot，必须对垃圾回收算法的执行效率有严格的考量，只有这样才能保证虚拟机高效运行

## 枚举根节点
从可达性分析中从 GC Roots 节点找引用链这个操作为例，可以作为 GC Roots 的节点主要在全局性的引用（例如常量或者类静态属性）与执行上下文（例如栈帧中的本地变量表）中。

但是现在很多应用仅仅方法区就有数百兆，如果要逐个检查这里面的引用，那么必然会消耗很多的时间。

另外，可达性分析对执行时间的敏感还体现在 GC 停顿上，因为这项分析工作必须在一个能确保一致性的快照中进行 —— 这里的“一致性”指的是在整个分析过程中整个执行系统看起来就像是被冻结在某个时间点上，不可以出现分析过程中对象引用关系还在不停变化的情况，该点不满足的话分析结果准确性就无法得到保证。

这点是导致 GC 进行时必须停顿所有 Java 执行线程的其中一个重要原因，即使是号称不会发生停顿的 CMS 收集器中，枚举根节点也是必须要停顿的。

由于目前主流 Java 虚拟机使用的都是准确式 GC，所以当执行系统停顿下来后，并不需要一个不漏的检查完成所有执行上下文和全局的引用位置，**虚拟机应当是有办法直接得知那些地方存放着对象引用的。** 在HotSpot的实现中，是使用一组被称为**OopMap**的数据结构来达到这个目的的，在类加载的时候，HotSpot就把对象内什么偏移量上是什么类型的数据计算出来，在JIT 编译过程中，也会在特定的位置记录下栈和寄存器中哪些位置是引用。这样， GC 在扫描时就可以直接得知这些信息了。

## 安全点
在 OopMap 的帮助下，HotSpot 可以快速并且准确的完成 GC Roots 枚举，但是一个很现实的问题随之而来：可能导致引用关系变化，或者说 OopMap 内容变化的指令非常多，如果为每一条指令都生成对应的 OopMap，那么将需要大量的额外空间，这样 GC 的空间成本将会变得很高。

实际上，HotSpot 也的确没有为每一条指令都生成 OopMap，只是在“特定的位置” 记录了这些信息，这些位置被称为是安全点，**即程序执行时并非是在所有地方都能停顿下来开始 GC ，只有到达安全点时才能暂停。**

安全点的选择既不能太少以至于让 GC 等待太长时间，也不能过于频繁以至于过分增大运行时负荷。

**所以，安全点的选定基本上是以程序“是否具有让程序长时间执行的特征”为标准进行选定的——因为每条指令执行的时间都非常短暂，程序不太可能因为指令流长度太长这个原因而过长时间运行，“长时间执行”的最明显特征就是指令序列复用，例如方法调用、循环跳转、异常跳转等，所以具有这些功能的指令才会产生安全点。**

对于安全点，另一个需要考虑的问题就是如何让 GC 发生时让所有线程（这里不包括执行 JNI 调用的线程）都“跑”到最近的安全点再停顿下来。这里有两种方案可供选择：抢先式中断和主动式中断。

* 抢先式中断：无需线程的执行代码主动配合，在 GC 发生时，首先把所有线程全部中断，如果发现有线程中断的地方不再安全点上，就会发线程，让它“跑”到安全点上。现在几乎没有虚拟机实现采用抢先式中断来暂停线程从而响应GC事件
* 主动式中断：当 GC 需要中断线程时，不直接对线程操作，仅仅简单地设置一个标志，各个线程执行时主动去轮询这个标志，发现中断标志为真时就自己中断挂起。轮询标志的地方和安全点是重合的，另外再加上创建对象需要分配内存的地方。

## 安全区域
使用安全点似乎已经完美解决了如何进入 GC 的问题，但实际情况却并不一定，安全点机制保证了程序执行，在不太长的时间内就会遇到可以进入 GC 的安全点。

但是线程“不执行”的时候呢？所谓不执行就是没有分配 CPU 时间，典型的例子就是线程处于 Sleep 状态或者 Blocked状态，这时候线程无法响应 JVM 的中断请求，“走”到安全点去中断挂起，JVM显然也不太可能等待线程重新被分配 CPU 时间。对于这种状况，就需要安全区域来解决。

**安全区域就是在一段代码片段中，引用关系不会发生变化，在这个区域中的任意地方开始 GC 都是安全的。**

在线程执行到安全区域中的代码时，首先标识自己已经进入了安全区域，那样，当这段时间里 JVM 要发起 GC 时，就不用管标识自己为安全区域状态的线程了。

当线程要离开安全区域时，它要检查系统是否已经完成了根节点枚举（或者是整个 GC 过程），如果完成了，那线程就继续执行，否则它就必须等待直到收到可以安全离开安全区域的信号为止。