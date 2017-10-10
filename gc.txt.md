
## 并行收集器
并行收集线程数可以通过-XX:ParallelGCThreads=N来指定。默认情况下，当CPU少于8核时将使用与核心数相同的线程数进行垃圾回收，当CPU核心数大于8时将使用核心数的5/8的线程数进行垃圾回收。

### 并行收集算法
server类型的机器上是默认的垃圾收集算法，使用该算法时可以通过-XX:MaxGCPauseMillis=N指定垃圾收集时暂停应用线程的最大时间，通过-XX:GCTimeRatio=N指定应用线程运行时间与垃圾回收时间的比例，指定了这两者之一后虚拟机将自动为了满足对应的目标进行自动调优，包括自动调整堆大小等一些与垃圾收集相关的属性，在调整的过程中不一定可以满足对应的目标。在没有指定堆大小或者堆大小不是固定值的情况，JVM自动调优的目标还包括使用最小的堆内存。

调优目标优先级如下：
1、满足最大的停顿时间的目标
2、满足吞吐量的目标
3、满足最小使用堆内存的目标

堆大小的增加与减少是以不同的速率进行的，默认情况下一个代的大小以20%的速率增大，以5%的速率减少，年轻代的增长速率可以通过-XX:YoungGenerationSizeIncrement=<Y>来指定，老年代的增长速率可以通过-XX:TenuredGenerationSizeIncrement=<T>来指定。代的大小的减少速率可以通过-XX:AdaptiveSizeDecrementScaleFactor=<D>来指定增速与减速的比率，如果一个代的增长速率是X，那么它的减少速率就是X/D。这种增量比在虚拟机启动的时候不一定按照指定的值来，在虚拟机启动的时候如果需要增大一个世代的堆大小，为了更快的启动，虚拟机会在指定的增量比的基础上再补充一些百分比。

如果最大暂停时间的目标没有满足，JVM将缩小代的大小，但是同一时间只会有一个代被缩小。如果年轻代和老年代的垃圾回收的最大暂停时间目标都没有得到满足，则优先缩小垃圾回收时间最多的代的大小。如果吞吐量的目标没有满足，则JVM将增大堆的大小，年轻代和老年代都将增大。每个代的增加将按照其在总的垃圾回收时间中的占比进行增加，比如说年轻代的垃圾回收时间占总的垃圾回收时间的25%，年轻代增长的速率是20%，则年轻代的增长将是25% * 20% = 5% 。



## CMS
年轻代收集时会暂停应用线程，老年代收集时是并行的，一个老年代收集周期分好几个阶段，只有少部分时候是会暂停应用线程的。老年代的垃圾收集周期中是可以交替出现年轻代垃圾收集的。CMS进行老年代垃圾收集时会暂停应用线程两次，第一次是初始标记阶段（标记那些存活的对象），第二次发生在再次标记阶段（标记那些由于引用线程更新而丢失的并发跟踪对象）。再次标记阶段结束后，就会进行并发的垃圾清理，真正清除那些不再需要用到的对象。一个垃圾收集过程完成后，垃圾收集器就会等待下一次垃圾收集过程的开始，在此期间是不会占用系统资源的。

### 并行收集失败（发生并发收集失败时将暂停应用线程）：
CMS在收集老年代的垃圾时是并发的，不需要暂停应用线程的。但是当CMS收集老年代垃圾的速度赶不上内存分配的速度时，即老年代被填满时就会发生并发收集失败，此时CMS将暂停所有的应用线程，以期能够快速的收集垃圾。当老年代没有被填满，但是需要申请一块比较大的内存时，老年代由于没有连续的内存空间也会导致分配失败，此时也会发生并发收集失败。如果太多的时间花在GC上了，CMS将抛出一个OutOfMemoryError。如果超过98%的时间是花在GC上的，而且回收的空间是少于2%的，就会抛出OutOfMemoryError。如果需要可以使用-XX:-UseGCOverheadLimit来禁用该功能。


什么时候会开始一次并发收集呢？
默认在老年代的空间占用到92%时会开始一次并发收集，这个值可以通过-XX:CMSInitiatingOccupancyFraction=N来调整，比如-XX:CMSInitiatingOccupancyFraction=50。另外JVM会在内部记录过往的垃圾回收情况，然后动态的评估老年代的空间被占满还需要多长时间，每次垃圾回收需要多少时间，然后会以评估的数据在老年代空间被占满之前能够完成一次垃圾回收周期为目标发起一次垃圾回收周期。


并发收集周期默认会包含以下几个阶段:
1.暂停所有的应用线程，标记所有从根开始可达的对象，即还存活的对象，然后会恢复应用线程。
2.并发的跟踪这些可达的对象图表，即在应用线程运行的同时启用一到多个线程进行跟踪。
3.再次并发的跟踪那些在第一次跟踪以后有过变更的对象。
4.暂停所有的应用线程，再次跟踪自从上次检查后那些可能有过变更的对象，然后恢复应用线程。
5.并发的清除那些不可达的对象。
6.并发的重置堆，并为下一次的垃圾回收周期准备好数据结构。


CMS垃圾收集器也可以通过-XX:MaxGCPauseMillis和-XX:GCTimeRatio来指定自动调优的目标。


## G1
G1垃圾收集器是Garbage First的简称，其会将堆分配为最多不超过2048个区，每个区的大小在1-32MB之间。可以通过-XX:G1HeapRegionSize=N来指定每个分区的大小，应该尽量指定的是2的N次方，如果指定的不是2的N次方，则JVM将向下取整取到最接近2的N次方的数字。一个区可能属于年轻代的eden区，也可能属于年轻代的survivor区，或者属于老年代。老年代的垃圾回收始于并发标记阶段，用于标记存活的对象。当整个堆的占用比达到了参数InitiatingHeapOccupancyPercent指定的比率时将开始并发标记阶段，该参数可以通过-XX:InitiatingHeapOccupancyPercent=<NN>指定，默认是45。
	

MaxGCPauseMillis

可以通过MaxGCPauseMillis指定垃圾回收停顿时间的目标。G1收集器使用预见式的方式来决定每次垃圾回收时能回收的垃圾量以达到MaxGCPauseMillis的目标，在一次垃圾回收完成后，G1会选择在下一次垃圾回收时将会回收的区。这些区中也会包含年轻代的区。
	
		Set a pause time goal for G1 with the flag MaxGCPauseMillis. G1 uses a prediction model to decide how much garbage collection work can be done within that target pause time. At the end of a collection, G1 chooses the regions to be collected in the next collection (the collection set). The collection set will contain young regions (the sum of whose sizes determines the size of the logical young generation). It is partly through the selection of the number of young regions in the collection set that G1 exerts control over the length of the GC pauses. You can specify the size of the young generation on the command line as with the other garbage collectors, but doing so may hamper the ability of G1 to attain the target pause time. In addition to the pause time goal, you can specify the length of the time period during which the pause can occur. You can specify the minimum mutator usage with this time span (GCPauseIntervalMillis) along with the pause time goal. The default value for MaxGCPauseMillis is 200 milliseconds. The default value for GCPauseIntervalMillis (0) is the equivalent of no requirement on the time span.


	The region sizes can vary from 1 MB to 32 MB depending on the heap size. The goal is to have no more than 2048 regions. 
	
	
标记周期有如下这些阶段:
1.初始标记阶段。
2.根区扫描阶段。
3.并发标记阶段。G1收集器将寻找整个堆中存活的对象，这个阶段是跟应用线程一起运行的，它可以被年轻代的垃圾收集中断。
4.重新标记阶段。这个阶段会暂停所有的应用线程。
5.清理阶段。
		


Table 10-1 Default Values of Important Options for G1 Garbage Collector

Option and Default Value	Option
-XX:G1HeapRegionSize=n 指定一个区域的大小

-XX:MaxGCPauseMillis=200 指定最大的暂停时间

-XX:G1NewSizePercent=5 指定年轻代的最小占比


-XX:G1MaxNewSizePercent=60 指定年轻代的最大占比


-XX:ParallelGCThreads=n 指定进行年轻代垃圾收集或者是Full GC时的线程数


-XX:ConcGCThreads=n 指定并发标记时使用的线程数。建议指定该值为ParallelGCThreads的1/4。


-XX:InitiatingHeapOccupancyPercent=45 指定触发垃圾收集的堆的占用比，默认是45。


-XX:G1MixedGCLiveThresholdPercent=85 指定允许进行混合式的垃圾回收时老年代占比的阈值，超过这个阈值就会进行Full GC了。


-XX:G1HeapWastePercent=5 指定允许浪费的堆空间占比。


-XX:G1MixedGCCountTarget=8

Sets the target number of mixed garbage collections after a marking cycle to collect old regions with at most G1MixedGCLIveThresholdPercent live data. The default is 8 mixed garbage collections. The goal for mixed collections is to be within this target number.Footref1

-XX:G1OldCSetRegionThresholdPercent=10

Sets an upper limit on the number of old regions to be collected during a mixed garbage collection cycle. The default is 10 percent of the Java heap.Footref1

-XX:G1ReservePercent=10

Sets the percentage of reserve memory to keep free so as to reduce the risk of to-space overflows. The default is 10 percent. When you increase or decrease the percentage, make sure to adjust the total Java heap by the same amount.Footref1



How to Unlock Experimental VM Flags
To change the value of experimental flags, you must unlock them first. You can do this by setting -XX:+UnlockExperimentalVMOptions explicitly on the command line before any experimental flags. For example:

java -XX:+UnlockExperimentalVMOptions -XX:G1NewSizePercent=10 -XX:G1MaxNewSizePercent=75 G1test.jar		







Recommendations
When you evaluate and fine-tune G1 GC, keep the following recommendations in mind:

Young Generation Size: Avoid explicitly setting young generation size with the -Xmn option or any or other related option such as -XX:NewRatio. Fixing the size of the young generation overrides the target pause-time goal.

Pause Time Goals: When you evaluate or tune any garbage collection, there is always a latency versus throughput trade-off. The G1 GC is an incremental garbage collector with uniform pauses, but also more overhead on the application threads. The throughput goal for the G1 GC is 90 percent application time and 10 percent garbage collection time. Compare this to the Java HotSpot VM parallel collector. The throughput goal of the parallel collector is 99 percent application time and 1 percent garbage collection time. Therefore, when you evaluate the G1 GC for throughput, relax your pause time target. Setting too aggressive a goal indicates that you are willing to bear an increase in garbage collection overhead, which has a direct effect on throughput. When you evaluate the G1 GC for latency, you set your desired (soft) real-time goal, and the G1 GC will try to meet it. As a side effect, throughput may suffer. See the section Pause Time Goal in Garbage-First Garbage Collector for additional information.

Taming Mixed Garbage Collections: Experiment with the following options when you tune mixed garbage collections. See the section Important Defaults for information about these options:

-XX:InitiatingHeapOccupancyPercent: Use to change the marking threshold.

-XX:G1MixedGCLiveThresholdPercent and -XX:G1HeapWastePercent: Use to change the mixed garbage collection decisions.

-XX:G1MixedGCCountTarget and -XX:G1OldCSetRegionThresholdPercent: Use to adjust the CSet for old regions.

Overflow and Exhausted Log Messages
When you see to-space overflow or to-space exhausted messages in your logs, the G1 GC does not have enough memory for either survivor or promoted objects, or for both. The Java heap cannot because it is already at its maximum. Example messages:

924.897: [GC pause (G1 Evacuation Pause) (mixed) (to-space exhausted), 0.1957310 secs]

924.897: [GC pause (G1 Evacuation Pause) (mixed) (to-space overflow), 0.1957310 secs]

To alleviate the problem, try the following adjustments:

Increase the value of the -XX:G1ReservePercent option (and the total heap accordingly) to increase the amount of reserve memory for "to-space".

Start the marking cycle earlier by reducing the value of -XX:InitiatingHeapOccupancyPercent.

Increase the value of the -XX:ConcGCThreads option to increase the number of parallel marking threads.

See the section Important Defaults for a description of these options.








## 禁用显示的垃圾回收
-XX:+DisableExplicitGC参数可以禁用显示的垃圾回收，即禁用System.gc()的调用响应。

## 垃圾日志
最基本的-verbose:gc和-XX:+PrintGC就可以开启基本的垃圾回收日志。使用-XX:+PrintGCDetails可以输出更加详细的垃圾回收日志。-XX:+PrintGCTimeStamps会输出垃圾回收时从JVM运行开始的时间。-XX:+PrintGCDteStamps会输出垃圾回收时的格式化的时间。

默认情况下垃圾回收的日志是输出到标准输出的，可以通过-Xloggc:filepath指定垃圾回收的日志输出到某个文件。通过-XX:+UseGCLogfileRetation可以开启日志的循环输出，-XX:NumberOfGCLogfiles=N指定日志文件的数目，不指定默认是0,表示无限制；通过-XX:GCLogfileSize=N指定单个日志文件的大小，不指定默认是0,表示无限制，如-XX:GCLogfileSize=100M。
















