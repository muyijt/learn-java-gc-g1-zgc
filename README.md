# learn-java-gc-g1-zgc
> 学习g1和zgc垃圾收集器

## G1收集器
![G1堆内存模型图](https://edrawcloudpubliccn.oss-cn-shenzhen.aliyuncs.com/viewer/self/38637691/share/2026-3-26/1774509182/main.svg)
```text
G1将java堆划分为多个大小相等的独立区域(region)，jvm最多可以有2048个region。
一般region大小等于堆大小除以2048。比如堆大小为4096M，则region大小为4096/2048=2M。当然也可以用参数“-XX:G1HeapRegionSize”手动指定region大小，但是推荐默认的计算方式，不手动指定大小。
G1保留了年轻代和老年代的概念，但不再是物理隔阂了，他们都是（可以不连续）region的集合。
默认年轻代初始占堆内存的5%，可以通过“-XX:G1NewSizePercent”设置新生代初始占比，在系统运行中，jvm会不停的给年轻代增加更多的region，但最多新生代的占比不会超过堆内存的60%，可以通过“-XX:G1MaxNewSizePercent”来调整。
年轻代中的Eden和Survivor对应的region数量比为8:1:1。
一个region可能之前是年轻代，进行垃圾回收后，这个region可能变成老年代。
G1垃圾收集器对于对象什么时候会转移到老年代跟之前（learn-java-memory-allocation）原则一样，唯一不同的是对于大对象的处理，G1有专门分配大对象的region叫Humongous区。一个对象大小超过一个region大小的50%就被认为是大对象，放入Humongous区。如果一个大对象太大会跨多个region来存放。
Full GC的时候除了收集年轻代，老年代之外，也将Humongous区一并回收

工作过程
  1. 初始标记: 暂停所有工作线程，记录下gc roots能直接引用的对象，速度很快
  2. 并发标记: 同CMS的并发标记
  3. 最终标记: 同CMS的重新标记
  4. 筛选回收: 首先对各个region的回收价值和成本进行排序，根据用户所期望的GC停顿时间(-XX:MaxGCPauseMillis来指定，默认200ms)来制定回收计划

垃圾收集分类
  YoungGC
    Eden区放满后，G1会计算下现在回收Eden区需要的时间，如果时间远小于参数-XX:MaxGCPauseMills设定的值，那么增加年轻代的region。如果时间接近-XX:MaxGCPauseMills设定的值就会触发Young GC
  MixedGC
    不是FullGC，老年代的堆占有率达到参数(-XX:InitiatingHeapOccupancyPercent)设定的值则触发，回收所有的Young和部分Old（根据期望的GC停顿时间确定Old区垃圾收集的优先顺序）以及大对象区。使用复制算法把各个region中存活的对象拷贝到别的region里去，拷贝过程中如果发现没有足够的空region能够承载拷贝对象就会触发一次Full GC。
  FullGC
    暂停所有工作线程，采用单线程进行标记、清理和压缩整理，好空闲出空region供下一次MixedGC使用，这个过程非常耗时。

参数
  -XX:+UseG1GC 使用G1收集器
  -XX:ParallelGCThreads 指定GC工作的线程数量
  -XX:G1HeapRegionSize 指定单个region大小(1M~32M，且必须是2的n次幂)，默认是将整个堆划分为2048个分区
  -XX:MaxGCPauseMillis 目标暂停时间（默认200ms）
  -XX:G1NewSizePercent 新生代内存初始空间(默认整堆5%)
  -XX:G1MaxNewSizePercent 新生代内存最大空间
  -XX:TargetSurvivorRatio to Survivor区的填充容量（默认50%），to Survivor区域里的一批对象（年龄1+年龄2+……+年龄n）综合超过了to Survivor区域的50%，此时就会把年龄>=n的对象放入老年代
  -XX:MaxTenuringThreshold 最大年龄阈值（默认15）
  -XX:InitiatingHeapOccupancyPercent 老年代占用空间达到整堆内存阈值(默认45%)，则执行新生代和老年代的混合收集(MixedGC)
  -XX:G1MixedGCLiveThresholdPercent(默认85%) region中的存活对象低于这个值时才会回收该region，如果超过这个值，存活对象过多，回收的的意义不大
  -XX:G1MixedGCCountTarget 在一次回收过程中指定做几次筛选回收(默认8次)，在最后一个筛选回收阶段可以回收一会，然后暂停回收，恢复系统运行，一会再开始回收，这样可以让系统不至于单次停顿时间过长
  -XX:G1HeapWastePercent(默认5%) gc过程中空出来的region是否充足阈值，在混合回收的时候，对Region回收都是基于复制算法进行的，都是把要回收的Region里的存活对象放入其他Region，然后这个Region中的垃圾对象全部清理掉，这样的话在回收过程就会不断空出来新的Region，一旦空闲出来的Region数量达到了堆内存的5%，此时就会立即停止混合回收，意味着本次混合回收就结束了。
```

## ZGC垃圾收集器
```text

```

## 垃圾收集器选择
1. 有限调整堆的大小让服务器自己来选择
2. 如果内存小于100m，使用穿行收集器
3. 如果是单核、没有停顿时间要求，串行或jvm自己选择
4. 如果运行停顿时间超过1秒，选择并行或者jvm自己选择
5. 如果相应时间最重要，并且不能超过1秒，使用并发收集器
6. 4G以下使用parallel，4~8G可以用ParNew+CMS，8G以上可以用G1，几百G以上用ZGC