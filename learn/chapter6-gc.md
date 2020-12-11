垃圾收集算法篇
# throughput收集器
## 新生代垃圾收集
新生代垃圾回收发生在eden空间快用尽时。一部分移动到survivor空间，一部分移动到老年代，一部分回收掉了。
新生代垃圾回收是多线程操作的。
* 问题1 快用尽怎么定量？
* 问题2 怎么判断还在使用的对象移动到老年代还是s0？
* 问题3 回收掉是怎么个操作？

## 老年代垃圾收集
老年代垃圾收集会回收新生代中所有对象（包括survivor空间中的对象）。只有活跃对象或者已经压缩整理的对象会在老年代继续保持。
* 问题1 怎么定义活跃对象？
* 问题2 什么时候进行的压缩整理？
* 问题3 为什么老年代垃圾收集会回收新生代对象呀？ 老年代垃圾收集线程和新生代垃圾收集线程会冲突吗？
  
## 开启PrintGCDetails 可以看到gc过程
* minor gc 日志 xxtimestamp: [GC [PSYoungGen:xx新生代回收前大小->xx新生代回收后大小(xx新生代总大小)] xxx]
* full gc 日志  xxtimestamp:[Full GC [PSYoungGen:xx->xx(xx)][ParOldGen:xx->xx] xx->xx(xx)[PSPermGen:xx->xx(xx)] xx secs] [Times: user=xx sys=xx, real=xx secs] 其中 ninor gc和 full gc的耗时打印中 可以看到cpu耗时user=xx 值大于 real时间，因为使用了多个线程，所以总的cpu耗时更长。

## 堆大小堆自适应调整和静态调整
throughput收集器的调优是围绕停顿时间进行的，寻求堆总体大小、新生代、老年代大小的平衡。
* -XX:MaxGCPauseMillis=N 最大停顿时间
* -XX:GCTimeRatio=N  eg 95 代表 1-1/（1+19）=95% 说明期望应用程序线程工作的时间95%    默认99

# cms收集器
## 新生代垃圾收集
与throughput收集器新生代垃圾收集相似。对新生代对象进行回收时，所有应用线程都会被暂停。
日志格式:  xxtimestamp: [GC  xxtime: [ParNew: xx->xx(xx),xx secs] xx -> xx(xx) , xx secs][Times: user=xx sys=xx, real=xx secs]

## cms并发回收周期
### 阶段一 初始标记  stw（stop the world）
xxtime:[GC [1 CMS-initial-mark: xx老年代占用大小(xx老年代堆大小)] xx整个堆占用大小（xx整个堆大小）][Times: user=0.08 sys=0.00,real=0.08 secs]

### 阶段二 标记阶段 应用线程不会被中断
xxtime: [CMS-concurrent-mark-start]
xxtime: [CMS-concurrent-mark: xx secs][Times:.....]
该阶段仅标记，堆大小无实质改变

### 阶段三 预清理 应用线程不会被中断
xxtime: [CMS-concurrent-preclean-start]
xxtime: [CMS-concurrent-preclean: xx secs][Times:.....]

### 阶段四 重新标记
#### 阶段4.1 可中断预清理 应用线程不会被中断
目的是为了避免新生代垃圾收集完成后紧接着进行重新标记，导致停顿时间过长。
xxtime: [CMS-concurrent-abortable-preclean-start]
xxtime: 新生代垃圾回收。。。。
xxtime: [CMS-concurrent-abortable-preclean: xxsecs] [times:...]
#### 阶段4.2 重新标记 stw
xxtime: [GC [YG occupancy: xx (xx)]
   xxtime: [Rescan(parallel),xx secs]
   xxtime: [weak refs processing,xx secs]
   xxtime: [scrub string table,xx secs]
   xxtime: [1 CMS-remark xx(xx)]
   xx(xx) xx secs]
   [Times:.....]
### 阶段五 清除阶段
xxtime: [CMS-concurrent-sweep-start]
xxtime: [CMS-concurrent-sweep: xx secs][Times:...]
### 阶段六 并发重置
xxtime: [CMS-concurrent-reset-start]
xxtime: [CMS-concurrent-reset: xx secs][Times:...]

## 异常情况进行fullgc
* 并发模式失效（concurrent mode failure）：新生代垃圾回收，老年代没有足够空间容纳晋升的对象
* 晋升失败（promotion failed）：有足够空间容纳，但是空间碎片化导致晋升失败

发生并发失效时，cms垃圾回收退化为full gc，stw，对老年代无效对象进行回收。这个操作是单线程的，非常耗时
默认情况下，cms收集器不会对永久代进行垃圾回收。一旦用尽，需要进行full gc。

# G1 垃圾收集器
* G1垃圾收集器是一种工作在堆内不同分区上的并发收集器。分区可以属于老年代，也可以属于年轻代（默认一个堆被划分为2048个分区）。同一个代的分区不一定是连续的。
* 分区设计的初衷是在老年代垃圾回收时只处理垃圾最多的分区，花费较少的时间
* 老年代分区处理的算法不适用于新生代，新生代之所以使用分区是为了便于代大小调整

## 新生代垃圾收集
* eden空间耗尽时触发g1垃圾收集器进行新生代垃圾收集。
  
> xxtime: [GC pause(young), xx secs] ...
> [Eden: xx(xx)->xx(xx) Survivors: xx->xx Heap: xx(xx)->xx(xx)] [Times: user=xx sys=xx,real=xx secs]
  
## 并发周期
并发周期的清理阶段真正回收的内存数量很少， 真正做的事情是定位出哪些老年代分区可回收垃圾最多（标记X分区）

### 阶段一 初始标记 stw
> xxtime: [GC pause(young) (initial-mark), xx secs] 
### 阶段二 扫描根分区 无需暂停 不可中断
> xxtime: [GC concurrent-root-region-scan-start]
> xxtime: [GC concurrent-root-region-scan-end, xxx]
### 阶段三 并发标记 无需暂停 可中断
> xxtime: [GC concurrent-mark-start]
> xxtime: [GC concurrent-mark-end, xx sec]
### 阶段四 重新标记 stw
> xxtime: [GC remark xxtime [GC ref-PRC, xx secs], xx secs]
### 阶段五 清理阶段
> xxtime: [GC cleanup xx->xx(xx),xx secs]
### 阶段六 并发清理
> xxtime: [GC concurrent-cleanup-start]
> xxtime: [GC concurrent-cleanup-end,xx]

## 混合式垃圾收集
* 进行正常的新生代垃圾收集，同时回收并发周期后台扫描线程标记的分区。
* 标记分区（x）中的绝大部分对象被回收，活跃数据被移动到另一个分区。g1的这种移动方式伴随着压缩，所以相比cms更少产生碎片
## 必要时的full gc


  
