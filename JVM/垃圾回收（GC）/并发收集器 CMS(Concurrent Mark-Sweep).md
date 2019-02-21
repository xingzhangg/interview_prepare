## 1.总体介绍：

CMS(Concurrent Mark-Sweep)是以牺牲吞吐量为代价来获得最短回收停顿时间的垃圾回收器。对于要求服务器响应速度的应用上，这种垃圾回收器非常适合。**CMS是用于对tenured generation的回收，也就是年老代的回收**，目标是尽量减少应用的暂停时间，减少full gc发生的几率，利用和应用程序线程并发的垃圾回收线程来标记清除年老代。在启动JVM参数加上-XX:+UseConcMarkSweepGC ，这个参数表示对于老年代的回收采用CMS。CMS采用的基础算法是：标记—清除。

## 2.CMS过程：

- 初始标记(STW initial mark) ***暂停应用 
- 并发标记(Concurrent marking)
- 并发预清理(Concurrent precleaning)
- 重新标记(STW remark) *** 暂停 应用
- 并发清理(Concurrent sweeping)
- 并发重置(Concurrent reset)

**初始标记** ：在这个阶段，需要虚拟机停顿正在执行的任务，官方的叫法STW(Stop The Word)。这个过程从垃圾回收的"根对象"开始，只扫描到能够和"根对象"直接关联的对象，并作标记。所以这个过程虽然暂停了整个JVM，但是很快就完成了。

**并发标记** ：这个阶段紧随初始标记阶段，在初始标记的基础上继续向下追溯标记。并发标记阶段，应用程序的线程和并发标记的线程并发执行，所以用户不会感受到停顿。

**并发预清理** ：并发预清理阶段仍然是并发的。在这个阶段，虚拟机查找在执行并发标记阶段新进入老年代的对象(可能会有一些对象从新生代晋升到老年代， 或者有一些对象被分配到老年代)。通过重新扫描，减少下一个阶段"重新标记"的工作，因为下一个阶段会Stop The World。

**重新标记** ：这个阶段会暂停虚拟机，收集器线程扫描在CMS堆中剩余的对象。扫描从"跟对象"开始向下追溯，并处理对象关联。

**并发清理** ：清理垃圾对象，这个阶段收集器线程和应用程序线程并发执行。

**并发重置** ：这个阶段，重置CMS收集器的数据结构，等待下一次垃圾回收。

 

CSM执行过程： 
![点击查看原始大小图片](../pics/2b525609-ce63-3a42-bf19-b2fbcd42f26c.png)

## 3.CMS缺点

- CMS回收器采用的基础算法是Mark-Sweep。所有CMS不会整理、压缩堆空间。这样就会有一个问题：经过CMS收集的堆会产生空间碎片。 CMS不对堆空间整理压缩节约了垃圾回收的停顿时间，但也带来的堆空间的浪费。为了解决堆空间浪费问题，CMS回收器不再采用简单的指针指向一块可用堆空 间来为下次对象分配使用。而是把一些未分配的空间汇总成一个列表，当JVM分配对象空间的时候，会搜索这个列表找到足够大的空间来hold住这个对象。

- 需要更多的CPU资源。从上面的图可以看到，为了让应用程序不停顿，CMS线程和应用程序线程并发执行，这样就需要有更多的CPU，单纯靠线程切 换是不靠谱的。并且，重新标记阶段，为空保证STW快速完成，也要用到更多的甚至所有的CPU资源。当然，多核多CPU也是未来的趋势！

- CMS的另一个缺点是它需要更大的堆空间。因为CMS标记阶段应用程序的线程还是在执行的，那么就会有堆空间继续分配的情况，为了保证在CMS回 收完堆之前还有空间分配给正在运行的应用程序，必须预留一部分空间。也就是说，CMS不会在老年代满的时候才开始收集。相反，它会尝试更早的开始收集，已 避免上面提到的情况：在回收完成之前，堆没有足够空间分配！默认当老年代使用68%的时候，CMS就开始行动了。 – XX:CMSInitiatingOccupancyFraction =n 来设置这个阀值。

总得来说，CMS回收器减少了回收的停顿时间，但是降低了堆空间的利用率。

## 4.啥时候用CMS

如果你的应用程序对停顿比较敏感，并且在应用程序运行的时候可以提供更大的内存和更多的CPU(也就是硬件牛逼)，那么使用CMS来收集会给你带来好处。还有，如果在JVM中，有相对较多存活时间较长的对象(老年代比较大)会更适合使用CMS。



=================================================================



下面是参数介绍和遇到的问题总结，

1、启用CMS：**-XX:+UseConcMarkSweepGC**。 

 

2。CMS默认启动的回收线程数目是  (ParallelGCThreads + 3)/4) ，如果你需要明确设定，可以通过**-XX:ParallelCMSThreads**=20来设定,其中ParallelGCThreads是年轻代的并行收集线程数


3、CMS是不会整理堆碎片的，因此为了防止堆碎片引起full gc，通过会开启CMS阶段进行合并碎片选项：**-XX:+UseCMSCompactAtFullCollection**，开启这个选项一定程度上会影响性能，阿宝的blog里说也许可以通过配置适当的CMSFullGCsBeforeCompaction来调整性能，未实践。

4.为了减少第二次暂停的时间，开启并行remark: **-XX:+CMSParallelRemarkEnabled**。如果remark还是过长的话，可以开启**-XX:+CMSScavengeBeforeRemark**选项，强制remark之前开始一次minor gc，减少remark的暂停时间，但是在remark之后也将立即开始又一次minor gc。

5.为了避免Perm区满引起的full gc，建议开启CMS回收Perm区选项：
**+CMSPermGenSweepingEnabled -XX:+CMSClassUnloadingEnabled**

6.默认CMS是在tenured generation沾满68%的时候开始进行CMS收集，如果你的年老代增长不是那么快，并且希望降低CMS次数的话，可以适当调高此值：
**-XX:CMSInitiatingOccupancyFraction=80**

这里修改成80%沾满的时候才开始CMS回收。

7.年轻代的并行收集线程数默认是(cpu <= 8) ? cpu : 3 + ((cpu * 5) / 8)，如果你希望降低这个线程数，可以通过**-XX:ParallelGCThreads=** N 来调整。

8.进入重点，在初步设置了一些参数后，例如：

 ```
1. -server -Xms1536m -Xmx1536m -XX:NewSize=256m -XX:MaxNewSize=256m -XX:PermSize=64m  
2. -XX:MaxPermSize=64m -XX:-UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection  
3. -XX:CMSInitiatingOccupancyFraction=80 -XX:+CMSParallelRemarkEnabled  
4. -XX:SoftRefLRUPolicyMSPerMB=0  
 ```


需要在生产环境或者压测环境中测量这些参数下系统的表现，这时候需要打开GC日志查看具体的信息，因此加上参数：

**-verbose:gc -XX:+PrintGCTimeStamps -XX:+PrintGCDetails -Xloggc:/home/test/logs/gc.log**

在运行相当长一段时间内查看CMS的表现情况，CMS的日志输出类似这样：

```

1. 4391.322: [GC [1 CMS-initial-mark: 655374K(1310720K)] 662197K(1546688K), 0.0303050 secs] [Times: user=0.02 sys=0.02, real=0.03 secs]  
2. 4391.352: [CMS-concurrent-mark-start]  
3. 4391.779: [CMS-concurrent-mark: 0.427/0.427 secs] [Times: user=1.24 sys=0.31, real=0.42 secs]  
4. 4391.779: [CMS-concurrent-preclean-start]  
5. 4391.821: [CMS-concurrent-preclean: 0.040/0.042 secs] [Times: user=0.13 sys=0.03, real=0.05 secs]  
6. 4391.821: [CMS-concurrent-abortable-preclean-start]  
7. 4392.511: [CMS-concurrent-abortable-preclean: 0.349/0.690 secs] [Times: user=2.02 sys=0.51, real=0.69 secs]  
8. 4392.516: [GC[YG occupancy: 111001 K (235968 K)]4392.516: [Rescan (parallel) , 0.0309960 secs]4392.547: [weak refs processing, 0.0417710 secs] [1 CMS-remark: 655734K(1310720K)] 766736K(1546688K), 0.0932010 secs] [Times: user=0.17 sys=0.00, real=0.09 secs]  
9. 4392.609: [CMS-concurrent-sweep-start]  
10. 4394.310: [CMS-concurrent-sweep: 1.595/1.701 secs] [Times: user=4.78 sys=1.05, real=1.70 secs]  
11. 4394.310: [CMS-concurrent-reset-start]  
12. 4394.364: [CMS-concurrent-reset: 0.054/0.054 secs] [Times: user=0.14 sys=0.06, real=0.06 secs]  
```

其中可以看到CMS-initial-mark阶段暂停了0.0303050秒，而CMS-remark阶段暂停了0.0932010秒，因此两次暂停的总共时间是0.123506秒，也就是123毫秒左右。两次短暂停的时间之和在200以下可以称为正常现象。

但是你很可能遇到**两种fail引起full gc**：Prommotion failed和Concurrent mode failed。

Prommotion failed的日志输出大概是这样：

```
1. [ParNew (promotion failed): 320138K->320138K(353920K), 0.2365970 secs]42576.951: [CMS: 1139969K->1120688K(  
2. 166784K), 9.2214860 secs] 1458785K->1120688K(2520704K), 9.4584090 secs]  
```

这个问题的产生是由于救助空间不够，从而向年老代转移对象，年老代没有足够的空间来容纳这些对象，导致一次full gc的产生。解决这个问题的办法有两种完全相反的倾向：**增大救助空间、增大年老代或者去掉救助空间**。 增大救助空间就是调整-XX:SurvivorRatio参数，这个参数是Eden区和Survivor区的大小比值，默认是32，也就是说Eden区是 Survivor区的32倍大小，要注意Survivo是有两个区的，因此Surivivor其实占整个young genertation的1/34。调小这个参数将增大survivor区，让对象尽量在survitor区呆长一点，减少进入年老代的对象。去掉救助空 间的想法是让大部分不能马上回收的数据尽快进入年老代，加快年老代的回收频率，减少年老代暴涨的可能性，这个是通过将-XX:SurvivorRatio 设置成比较大的值（比如65536)来做到。在我们的应用中，将young generation设置成256M，这个值相对来说比较大了，而救助空间设置成默认大小(1/34)，从压测情况来看，没有出现prommotion failed的现象，年轻代比较大，从GC日志来看，minor gc的时间也在5-20毫秒内，还可以接受，因此暂不调整。

Concurrent mode failed的产生是由于CMS回收年老代的速度太慢，导致年老代在CMS完成前就被沾满，引起full gc，避免这个现象的产生就是调小**-XX:CMSInitiatingOccupancyFraction**参数的值，让CMS更早更频繁的触发，降低年老代被沾满的可能。我们的应用暂时负载比较低，在生产环境上年老代的增长非常缓慢，因此暂时设置此参数为80。在压测环境下，这个参数的表现还可以，没有出现过Concurrent mode failed。