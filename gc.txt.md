并行收集线程数可以通过-XX:ParallelGCThreads=N来指定。默认情况下，当CPU少于8核时将使用与核心数相同的线程数进行垃圾回收，当CPU核心数大于8时将使用核心数的5/8的线程数进行垃圾回收。
并行收集算法server类型的机器上是默认的垃圾收集算法，使用该算法时可以通过-XX:MaxGCPauseMillis=N指定垃圾收集时暂停应用线程的最大时间，通过-XX:GCTimeRatio=N指定应用线程运行时间与垃圾回收时间的比例，指定了这两者之一后虚拟机将自动为了满足对应的目标进行自动调优，包括自动调整堆大小等一些与垃圾收集相关的属性，在调整的过程中不一定可以满足对应的目标。在没有指定堆大小或者堆大小不是固定值的情况，JVM自动调优的目标还包括使用最小的堆内存。

调优目标优先级如下：
1、满足最大的停顿时间的目标
2、满足吞吐量的目标
3、满足最小使用堆内存的目标

Growing and shrinking are done at different rates. By default a generation grows in increments of 20% and shrinks in increments of 5%. The percentage for growing is controlled by the command-line option -XX:YoungGenerationSizeIncrement=<Y> for the young generation and -XX:TenuredGenerationSizeIncrement=<T> for the tenured generation. The percentage by which a generation shrinks is adjusted by the command-line flag -XX:AdaptiveSizeDecrementScaleFactor=<D>. If the growth increment is X percent, then the decrement for shrinking is X/D percent.
这种增量比在虚拟机启动的时候不一定按照指定的值来，在虚拟机启动的时候如果需要增大一个世代的堆大小，为了更快的启动，虚拟机会在指定的增量比的基础上再补充一些百分比。

If the maximum pause time goal is not being met, then the size of only one generation is shrunk at a time. If the pause times of both generations are above the goal, then the size of the generation with the larger pause time is shrunk first.

If the throughput goal is not being met, the sizes of both generations are increased. Each is increased in proportion to its respective contribution to the total garbage collection time. For example, if the garbage collection time of the young generation is 25% of the total collection time and if a full increment of the young generation would be by 20%, then the young generation would be increased by 5%.



CMS
年轻代收集时会暂停应用线程，老年代收集时是并行的，一个老年代收集周期分好几个阶段，只有少部分时候是会暂停应用线程的。老年代的垃圾收集周期中是可以交替出现年轻代垃圾收集的。
CMS进行老年代垃圾收集时会暂停应用线程两次，第一次是初始标记阶段（标记那些存活的对象），第二次发生在再次标记阶段（标记那些由于引用线程更新而丢失的并发跟踪对象）。再次标记阶段结束后，就会进行并发的垃圾清理，真正清除那些不再需要用到的对象。一个垃圾收集过程完成后，垃圾收集器就会等待下一次垃圾收集过程的开始，在此期间是不会占用系统资源的。
并行收集失败（发生并发收集失败时将暂停应用线程）：
	CMS在收集老年代的垃圾时是并发的，不需要暂停应用线程的。但是当CMS收集老年代垃圾的速度赶不上内存分配的速度时，即老年代被填满时就会发生并发收集失败，此时CMS将暂停所有的应用线程，以期能够快速的收集垃圾。当老年代没有被填满，但是需要申请一块比较大的内存时，老年代由于没有连续的内存空间也会导致分配失败，此时也会发生并发收集失败。
	
	如果太多的时间花在GC上了，CMS将抛出一个OutOfMemoryError。如果超过98%的时间是花在GC上的，而且回收的空间是少于2%的，就会抛出OutOfMemoryError。如果需要可以使用-XX:-UseGCOverheadLimit来禁用该功能。


什么时候会开始一次并发收集呢？
	默认在老年代的空间占用到92%时会开始一次并发收集，这个值可以通过-XX:CMSInitiatingOccupancyFraction=N来调整，比如-XX:CMSInitiatingOccupancyFraction=50。
	另外JVM会在内部记录过往的垃圾回收情况，然后动态的评估老年代的空间被占满还需要多长时间，每次垃圾回收需要多少时间，然后会以评估的数据在老年代空间被占满之前能够完成一次垃圾回收周期为目标发起一次垃圾回收周期。


The concurrent collection cycle typically includes the following steps:

	Stop all application threads, identify the set of objects reachable from roots, and then resume all application threads.

	Concurrently trace the reachable object graph, using one or more processors, while the application threads are executing.

	Concurrently retrace sections of the object graph that were modified since the tracing in the previous step, using one processor.

	Stop all application threads and retrace sections of the roots and object graph that may have been modified since they were last examined, and then resume all application threads.

	Concurrently sweep up the unreachable objects to the free lists used for allocation, using one processor.

	Concurrently resize the heap and prepare the support data structures for the next collection cycle, using one processor.


-XX:+CMSIncrementalMode





G1
	both young and old regions are garbage collected in a mixed collection. To collect old regions, G1 does a complete marking of the live objects in the heap. Such a marking is done by a concurrent marking phase. A concurrent marking phase is started when the occupancy of the entire Java heap reaches the value of the parameter InitiatingHeapOccupancyPercent. Set the value of this parameter with the command-line option -XX:InitiatingHeapOccupancyPercent=<NN>. The default value of InitiatingHeapOccupancyPercent is 45.
		
	Pause Time Goal
	
		Set a pause time goal for G1 with the flag MaxGCPauseMillis. G1 uses a prediction model to decide how much garbage collection work can be done within that target pause time. At the end of a collection, G1 chooses the regions to be collected in the next collection (the collection set). The collection set will contain young regions (the sum of whose sizes determines the size of the logical young generation). It is partly through the selection of the number of young regions in the collection set that G1 exerts control over the length of the GC pauses. You can specify the size of the young generation on the command line as with the other garbage collectors, but doing so may hamper the ability of G1 to attain the target pause time. In addition to the pause time goal, you can specify the length of the time period during which the pause can occur. You can specify the minimum mutator usage with this time span (GCPauseIntervalMillis) along with the pause time goal. The default value for MaxGCPauseMillis is 200 milliseconds. The default value for GCPauseIntervalMillis (0) is the equivalent of no requirement on the time span.


	The region sizes can vary from 1 MB to 32 MB depending on the heap size. The goal is to have no more than 2048 regions. 
	
	
	The marking cycle has the following phases:

		Initial marking phase: The G1 GC marks the roots during this phase. This phase is piggybacked on a normal (STW) young garbage collection.

		Root region scanning phase: The G1 GC scans survivor regions marked during the initial marking phase for references to the old generation and marks the referenced objects. This phase runs concurrently with the application (not STW) and must complete before the next STW young garbage collection can start.

		Concurrent marking phase: The G1 GC finds reachable (live) objects across the entire heap. This phase happens concurrently with the application, and can be interrupted by STW young garbage collections.

		Remark phase: This phase is STW collection and helps the completion of the marking cycle. G1 GC drains SATB buffers, traces unvisited live objects, and performs reference processing.

		Cleanup phase: In this final phase, the G1 GC performs the STW operations of accounting and RSet scrubbing. During accounting, the G1 GC identifies completely free regions and mixed garbage collection candidates. The cleanup phase is partly concurrent when it resets and returns the empty regions to the free list.



Table 10-1 Default Values of Important Options for G1 Garbage Collector

Option and Default Value	Option
-XX:G1HeapRegionSize=n

Sets the size of a G1 region. The value will be a power of two and can range from 1 MB to 32 MB. The goal is to have around 2048 regions based on the minimum Java heap size.

-XX:MaxGCPauseMillis=200

Sets a target value for desired maximum pause time. The default value is 200 milliseconds. The specified value does not adapt to your heap size.

-XX:G1NewSizePercent=5

Sets the percentage of the heap to use as the minimum for the young generation size. The default value is 5 percent of your Java heap.Foot1

This is an experimental flag. See How to Unlock Experimental VM Flags for an example. This setting replaces the -XX:DefaultMinNewGenPercent setting.

-XX:G1MaxNewSizePercent=60

Sets the percentage of the heap size to use as the maximum for young generation size. The default value is 60 percent of your Java heap.Footref1

This is an experimental flag. See How to Unlock Experimental VM Flags for an example. This setting replaces the -XX:DefaultMaxNewGenPercent setting.

-XX:ParallelGCThreads=n

Sets the value of the STW worker threads. Sets the value of n to the number of logical processors. The value of n is the same as the number of logical processors up to a value of 8.

If there are more than eight logical processors, sets the value of n to approximately 5/8 of the logical processors. This works in most cases except for larger SPARC systems where the value of n can be approximately 5/16 of the logical processors.

-XX:ConcGCThreads=n

Sets the number of parallel marking threads. Sets n to approximately 1/4 of the number of parallel garbage collection threads (ParallelGCThreads).

-XX:InitiatingHeapOccupancyPercent=45

Sets the Java heap occupancy threshold that triggers a marking cycle. The default occupancy is 45 percent of the entire Java heap.

-XX:G1MixedGCLiveThresholdPercent=85

Sets the occupancy threshold for an old region to be included in a mixed garbage collection cycle. The default occupancy is 85 percent.Footref1

This is an experimental flag. See How to Unlock Experimental VM Flags for an example. This setting replaces the -XX:G1OldCSetRegionLiveThresholdPercent setting.

-XX:G1HeapWastePercent=5

Sets the percentage of heap that you are willing to waste. The Java HotSpot VM does not initiate the mixed garbage collection cycle when the reclaimable percentage is less than the heap waste percentage. The default is 5 percent.Footref1

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









-XX:+DisableExplicitGC
















