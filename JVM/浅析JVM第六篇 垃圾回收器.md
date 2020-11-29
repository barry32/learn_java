### 浅析JVM第六篇: 垃圾回收器

上一篇文章我们一起学到复制算法、标记删除、标记整理算法，复制算法被应用在年轻代垃圾回收之中，标记整理、标记删除被应用在老年代的垃圾回收当中。

**那我们想一下，如果让你来编写一个垃圾回收器你会有什么样的思路？**

* Serial GC

第一个思路就是串行的方式来执行垃圾回收算法。针对于年轻代时候复制算法，对于老年代使用标记整理算法。在执行垃圾回收的时候触发STW挂起应用线程，然后开启一个线程来执行相应的垃圾回收算法。启用这种算法对应的JVM参数是` -XX:+UseSerialGC`

我们来具体操作一下，为了方便操作查看GC时间和GC具体的日志，我们配上`-XX:+PrintGCDetails` 、`-XX+PrintGCDateStamps`、`-XX:+PrintGCTimeStamps`再加上`-XX:+UseSerialGC`一起来看查看GC日志

```java
2020-11-28T15:10:56.108+0800: 0.464: [GC (Allocation Failure) 2020-11-28T15:10:56.108+0800: 0.464: [DefNew: 7656K->904K(9216K), 0.0032368 secs] 7656K->3976K(29696K), 0.0032952 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-11-28T15:11:01.112+0800: 5.468: [GC (Allocation Failure) 2020-11-28T15:11:01.112+0800: 5.468: [DefNew: 8394K->0K(9216K), 0.0019440 secs] 11466K->5922K(29696K), 0.0019782 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-11-28T15:11:06.114+0800: 10.470: [GC (Allocation Failure) 2020-11-28T15:11:06.114+0800: 10.470: [DefNew: 7319K->0K(9216K), 0.0006347 secs] 13241K->6946K(29696K), 0.0006662 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-11-28T15:11:11.115+0800: 15.471: [GC (Allocation Failure) 2020-11-28T15:11:11.115+0800: 15.471: [DefNew: 7322K->0K(9216K), 0.0007164 secs] 14268K->6946K(29696K), 0.0008338 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-11-28T15:11:11.118+0800: 15.474: [GC (Allocation Failure) 2020-11-28T15:11:11.118+0800: 15.474: [DefNew: 7324K->0K(9216K), 0.0097433 secs] 14270K->14114K(29696K), 0.0098379 secs] [Times: user=0.00 sys=0.02, real=0.01 secs] 
2020-11-28T15:11:16.128+0800: 20.484: [GC (Allocation Failure) 2020-11-28T15:11:16.128+0800: 20.484: [DefNew: 7327K->7327K(9216K), 0.0000165 secs]2020-11-28T15:11:16.128+0800: 20.484: [Tenured: 14114K->6945K(20480K), 0.0026746 secs] 21441K->6945K(29696K), [Metaspace: 3958K->3958K(1056768K)], 0.0027372 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

上面是一串GC信息，我们来挑两条看一下。

```java
2020-11-28T15:10:56.108+0800: 0.464: [GC (Allocation Failure) 2020-11-28T15:10:56.108+0800: 0.464: [DefNew: 7656K->904K(9216K), 0.0032368 secs] 7656K->3976K(29696K), 0.0032952 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

`2020-11-28T15:10:56.108+0800` 表示GC开始时间为2020-11-28，`0.464`这边已经精确到秒的级别了。

`GC (Allocation Failure` GC是用来区别Minor GC 和Major GC的标志，此处是Minor GC，`2020-11-28T15:10:56.108+0800: 0.464`这边是Minor GC的开始时间，`[DefNew: 7656K->904K(9216K), 0.0032368 secs]` ,这边的信息比较多，`DefNew`指的是**default new generation**也就是使用的垃圾收集器的名字，此处为Serial，内存使用量由7656k变为904k，年轻代的总容量是9216K，耗时0.0032368秒。` 7656K->3976K(29696K), 0.0032952 secs] ` 对应的信息是，整个堆使用量由7656K变为3976K，此外堆容量的总大小是29696K，GC耗时为0.0032952 secs。` [Times: user=0.00 sys=0.00, real=0.00 secs] ` GC耗时的类别衡量，user是垃圾收集器消耗的CPU总时间，sys是OS调用或者等待系统事件的时间花费。real是应用被停止的时间，由于当前垃圾回收器是串行，所以real = sys + user

我们再来看另外一条，

```java
2020-11-28T15:11:16.128+0800: 20.484: [GC (Allocation Failure) 2020-11-28T15:11:16.128+0800: 20.484: [DefNew: 7327K->7327K(9216K), 0.0000165 secs]2020-11-28T15:11:16.128+0800: 20.484: [Tenured: 14114K->6945K(20480K), 0.0026746 secs] 21441K->6945K(29696K), [Metaspace: 3958K->3958K(1056768K)], 0.0027372 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
```

`2020-11-28T15:11:16.128+0800` 这边还是表示的是GC的开始时间， `20.484`这边将开始时间具体到秒数，`[GC (Allocation Failure) 2020-11-28T15:11:16.128+0800: 20.484` 这边依旧是Young GC，GC的原因是内存分配失败，`2020-11-28T15:11:16.128+0800: 20.484: [DefNew: 7327K->7327K(9216K), 0.0000165 secs]` 表示 年轻代串行垃圾收集，内存使用量7327变为7327，耗时为`0.0000165 secs`， 这边其实指的是，当前年轻代已经满了。`[Tenured: 14114K->6945K(20480K), 0.0026746 secs] 21441K->6945K(29696K)` 这边指的是 单线程的标记整理的垃圾收集器的名字，老年代的内存使用量由14114K变成6945K，老年代的总容量为20480K，老年代串行垃圾回收耗时为 0.0026746 secs，整个堆内存使用量由21441K变成6945K，整个堆内存的总容量是29696K。`[Metaspace: 3958K->3958K(1056768K)], 0.0027372 secs` , 元空间的信息，当前没有垃圾被回收，耗时为0.0026746 secs , `Times: user=0.00 sys=0.00, real=0.00 secs` 垃圾收集器消耗的CPU总时间为0， OS调用时间为0，应用被停止的时间为0。

```java
2020-11-29T20:31:20.868+0800: 0.807: [Full GC (Allocation Failure) 2020-11-29T20:31:20.868+0800: 0.807: [Tenured: 903K->880K(20480K), 0.0024428 secs] 903K->880K(29696K), [Metaspace: 3956K->3956K(1056768K)], 0.0024834 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

这个例子是直接触发了Full GC，这边不再赘述。

* Parallel GC

串行方式的垃圾回收往往不能令人满意，于是人们就会想到并行方式的垃圾回收。并行式的垃圾回收适用于多核的机器可以用来提高吞吐量，使用并行方式的垃圾回收，JVM还是会触发STW，来挂起应用线程，并启动多个线程来进行垃圾回收，至于使用多少线程来进行垃圾回收可以通过`-XX:ParallelGCThreads=XXX` (`-XX:ParallelGCThreads=4`)来进行配置，默认为机器的核心数。对此我们的JVM参数配置先设定为`-XX:+UseParallelGC`、`-XX:+UseParallelOldGC`、`-XX:ParallelGCThreads=4` ,为了更好的来观察并行方式的GC，我们继续使用`-XX:+PrintGCDetails`、`-XX:+PrintGCDateStamps`、`-XX:+PrintGCTimeStamps`这些参数来配合使用，然后我们一起来查看GC日志。

```java
2020-11-29T21:09:38.022+0800: 0.533: [GC (Allocation Failure) [PSYoungGen: 7663K->1023K(9216K)] 7663K->4103K(29696K), 0.0016500 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-11-29T21:09:43.025+0800: 5.535: [GC (Allocation Failure) [PSYoungGen: 8513K->888K(9216K)] 11593K->6016K(29696K), 0.0009643 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-11-29T21:09:48.027+0800: 10.537: [GC (Allocation Failure) [PSYoungGen: 8289K->856K(9216K)] 13417K->7008K(29696K), 0.0007085 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-11-29T21:09:53.028+0800: 15.538: [GC (Allocation Failure) [PSYoungGen: 8177K->888K(9216K)] 14330K->7040K(29696K), 0.0006895 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-11-29T21:09:53.029+0800: 15.539: [GC (Allocation Failure) [PSYoungGen: 8212K->872K(9216K)] 14364K->14192K(29696K), 0.0017337 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-11-29T21:09:58.031+0800: 20.541: [GC (Allocation Failure) [PSYoungGen: 8198K->888K(9216K)] 21519K->20352K(29696K), 0.0024651 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-11-29T21:09:58.034+0800: 20.544: [Full GC (Ergonomics) [PSYoungGen: 888K->0K(9216K)] [ParOldGen: 19464K->6945K(20480K)] 20352K->6945K(29696K), [Metaspace: 3959K->3959K(1056768K)], 0.0081908 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
2020-11-29T21:10:03.042+0800: 25.552: [GC (Allocation Failure) [PSYoungGen: 7319K->32K(9216K)] 14264K->14145K(29696K), 0.0009307 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
2020-11-29T21:10:03.043+0800: 25.553: [Full GC (Ergonomics) [PSYoungGen: 32K->0K(9216K)] [ParOldGen: 14113K->5898K(20480K)] 14145K->5898K(29696K), [Metaspace: 3960K->3960K(1056768K)], 0.0045204 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
```

当前的GC的日志比较多，我们还是老规矩，挑两条具有代表性的GC日志一起来分析一下。

```java
2020-11-29T21:09:38.022+0800: 0.533: [GC (Allocation Failure) [PSYoungGen: 7663K->1023K(9216K)] 7663K->4103K(29696K), 0.0016500 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

`2020-11-29T21:09:38.022+0800: 0.533` 这边依旧指的是GC开始的时间且开始时间精确到秒。`GC (Allocation Failure)` 这边指的是Minor GC也就是Young GC，GC的原因是内存分配失败。`[PSYoungGen: 7663K->1023K(9216K)] 7663K->4103K(29696K), 0.0016500 secs] ` ， 首先PSYoungGen这边指的垃圾收集器的名字，全名是**Parallel Scavenge Young Generation* ，内存使用量由7663K变成1023K，年轻代的容量是9216k，整个堆的内存使用情况是7663K变成4103k，整个堆的容量是29696K。` [Times: user=0.00 sys=0.00, real=0.00 secs] ` 这边 GC的花费的CPU时间是0，OS调用时间是0，real此时应该是接近于user+sys再除以GC线程数。

我们再来看一个例子。

```java
2020-11-29T21:10:03.043+0800: 25.553: [Full GC (Ergonomics) [PSYoungGen: 32K->0K(9216K)] [ParOldGen: 14113K->5898K(20480K)] 14145K->5898K(29696K), [Metaspace: 3960K->3960K(1056768K)], 0.0045204 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
```

`2020-11-29T21:10:03.043+0800: 25.553`这边还是GC开始时间并且开始时间精确到秒，`Full GC (Ergonomics) `这边是触发了Full GC 原因是JVM内部的Ergonomics认为当前是触发Full GC的最佳时机。`[PSYoungGen: 32K->0K(9216K)]` 这边PSYoungGen 指的是垃圾收集器的名字，年轻代内存使用情况由32k变成0k，年轻代的总容量是9216k。`[ParOldGen: 14113K->5898K(20480K)] 14145K->5898K(29696K)` 这边ParOldGen指的是Parallel Old Generation，内存使用量由14113k变成5898k，老年代的容量为2048k。整个堆的内存使用情况由14145k变成5898k且堆的容量是29696K。` [Metaspace: 3960K->3960K(1056768K)], 0.0045204 secs] `元空间内存使用情况没有发生变化，元空间的内存容量为1056768K，此次GC耗时为0.0045204 secs。` [Times: user=0.00 sys=0.00, real=0.00 secs]` GC消耗CPU时间为0，OS调用时间为0，real时间为0。

--------

（未完成，待更新）