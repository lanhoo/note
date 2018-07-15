<h1 class="mb-0 align-center">Go 与 Node 内存分配与垃圾回收</h1>


### Go 内存分配

Go 的内存分配采用的是空间换时间的策略，在系统初始化的时候，它会先预留一大块连续的内存区域，之后所有的大小对象的内存分配，需要向操作系统申请内存的时候，都会使用这块预留的内存。

那么，Go 为什么要预留这一段地址呢？

因为它的内存分配器也好，垃圾回收器也好，都要依赖于连续的内存地址，因为它把它当成一个数组来用。

go 以32kb为界，将大小对象分为两种，分配策略也根据大小对象分为两种。

1. 大对象一般生命周期很长，很有可能和线程一样，对于这种大对象的处理策略就是，直接申请内存，放入堆中。

2. 小对象优先的策略会先去cache上去取，是无锁的，为什么说是无锁的呢，因为这个cache是每个工作线程的cache，所以是不需要做任何的锁的操作；假如cache上的块用完了，就需要扩张，就会去堆上的cache上去申请已经切分好的对应等级的小对象。

堆上小对象内存切分策略是：把小对象按照8字节对齐，分为N种等级。存入对应cache上。


### Go 垃圾回收

Go 在1.3之前，采用的是普通的标记清除法。这个算法分为两步，标记和清除。

1. 标记：从程序的根节点开始，递归地遍历所有对象，将能遍历到的对象打上标记。
2. 清除：将所有未标记的对象当做垃圾销毁

这个算法有个缺陷是，在执行标记和清除的时候，会暂停整个程序，否则其他线程的代码可能会改变对象状态，从而可能把不应该回收的对象当做垃圾收集掉。

于是，在1.3的时候，Go 做了一些改进，就是把清除的过程改为了并行，就是程序只需要在标记的过程暂停。

1.5版本则进行了较大的改进，使用了三色标记算法。
三色标记法是传统Mark-Sweep的一个改进，它是一个并发的GC算法。原理如下：
1. 首先创建白、灰、黑三个集合；
2. 将所有对象放入白色集合中；
3. 然后从根节点开始遍历所有对象最外层，把遍历到的对象从白色集合放入灰色集合；
4. 之后遍历灰色集合，将灰色对象引用的对象从白色集合放入灰色集合，之后将此灰色对象放入黑色集合；
5. 重复4直到灰色中无任何对象；
6. 收集所有白色对象（垃圾）

这个算法在程序执行的同时进行收集，并不需要暂停整个程序。但是也会有一个缺陷。可能程序中的垃圾产生速度会大于垃圾收集的速度，这样会导致程序中的垃圾越来越多无法被收集掉。

### Node 内存分配

Node 基于V8构建，所以在Node中使用的Javascript对象基本上都是通过V8自己的方式来进行分配和管理的。在V8中，所有的JavaScript对象都是通过堆来进行分配的，普通变量则分配在栈上。

在一般的后端开发语言中，在基本的内存使用上没有什么限制，然而在Node中通过JavaScript使用内存时就会发现只能使用部分内存（64位系统下约为1.4GB，32位系统下约为0.7GB）。在这样的限制下，将会导致Node无法直接操作大内存对象。

V8为何要限制堆的大小呢？

1. 表层原因是V8最初就是为浏览器设计的，不太可能遇到用大量内存的场景。

2. 深层原因是V8的垃圾回收机制的限制。按官方的说法，以1.5GB的垃圾回收堆内存为例，V8做一次小的垃圾回收需要50毫秒以上，做一次非增量式的垃圾回收甚至要1秒以上。这是垃圾回收中引起JavaScript线程暂停执行的时间，在这样的时间花销下，应用的性能和响应能力都会直线下降。

### Node 垃圾回收机制

V8的垃圾回收策略主要基于分代式垃圾回收机制。在V8中，主要将内存分为新生代和老生代两代。新生代中的对象为存活时间较短的对象，老生代中的对象为存活时间较长或常驻内存的对象。V8堆的整体大小就是新生代所用内存空间加上老生代的内存空间。

那么，我们就先说说新生代。

新生代中使用的是一种采用复制的方式实现的垃圾回收算法。它将堆内存一分为二，一个处于使用中暂且称为From空间，另一个处于闲置状态暂且称为To空间。当我们分配对象时，先是在From空间进行分配。当开始进行垃圾回收时，会检查From空间中的存活对象，这些存活对象将被复制到To空间中，而非存活对象占用的空间将会被释放。完成复制后，From空间和To空间的角色发生对换。简而言之，在垃圾回收的过程中，就是通过将存活对象在两个空间之间进行复制。

显而易见，这种方式只能使用堆内存的一半，这是划分空间和复制机制所决定的，但是这种方式由于只复制存活的对象，并且对于生命周期短的场景搓火对象只占少部分，所以它在时间效率上有优异德 表现。和Go 一样，都是利用空间换时间的算法。

新生代在一定情况下，也会晋升为老生代。

当一个对象经过多次复制依然存活时，它将会被认为是生命周期较长的对象。这种生命周期较长的对象随后会被移动到老生代中，采用新的算法管理。对象从新生代中移动到老生代中的过程称为晋升。

那么，什么是老生代中的垃圾回收是怎么做的呢？

在老生代中，主要采用的是标记清除算法，与新生代中的那种算法相比，不需要将内存空间划分为两半，所以不存在浪费一半空间的行为。关于标记清除算法，这里不再赘述。

> 为了避免出现JavaScript应用逻辑与垃圾回收器看到的不一致的情况，垃圾回收的算法都是需要将应用逻辑暂停下来，待执行完垃圾回收后再恢复执行应用逻辑，所以，想要高性能的执行效率，需要注意让垃圾回收尽量少的进行，尤其是全堆垃圾回收。

其他常见的垃圾回收算法，可参考这篇博文：[golang 垃圾回收机制](https://lengzzz.com/note/gc-in-golang)