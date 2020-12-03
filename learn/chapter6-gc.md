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
* full gc 日志  xxtimestamp:[Full GC [PSYoungGen:xx->xx(xx)][ParOldGen:xx->xx] xx->xx(xx)[PSPermGen:xx->xx(xx)] xx secs [Times: user=xx sys=xx, real=xx secs]] 其中 ninor gc和 full gc的耗时打印中 可以看到cpu耗时user=xx 值大于 real时间，因为使用了多个线程，所以总的cpu耗时更长。


  
