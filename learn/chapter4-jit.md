
# 即时编译（JIT(Just-In-Time)）概览
计算机，更具体的说是cpu，只能执行汇编码或者二进制码。 因此高级语言必须“翻译”成这种指令。
* 编译型语言：源代码静态编译生成二进制文件。 例如 c++。 
* 解释型语言：解释器 将源代码 转换成 二进制代码。  例如 perl/php
* 解释型 vs 编译型 ：解释型可移植性高，执行速度比编译型慢。 
* java走的是一条中间路线，结合了脚本语言的平台独立性和编译型语言的本地性能。 java源代码 -（编译）-》java字节码 -（解释+jit编译）-》二进制机器码
  
# 调优入门：
大多数情况下的编译器调优，只是为目标机器上的java选择正确的jvm和编译器开关（-client 、 -server 、 -xx:+TieredCompilation）. 分层编译是长期运行应用的最佳选择。
程序不必指定编译器，而是仰仗平台支持的编译器。
确认默认编译器：java -version。
查看操作系统是64位or32位：uname -a

编译器类型：
* c1: 又叫client编译器
* c2: 又叫server编译器
* 分层编译：启动时用client编译器，随着代码变热用server编译器。
  
c1和c2最主要的差别是编译时机不同，client编译器开始的早，server编译器可以做更多优化。
安装java前就需要考虑如何选择编译器，因为不同的java安装包包含了不同的编译器。同时，可以使用参数进行设置 

表格-不同的jvm安装版本和不同参数下的编译器:
|jvm版本| -client| -server| -d64|
|:----  | :---- | ---- | ---- |
|linux32位| 32位client编译器|32位server编译器|出错|
|linux64位| 64位server编译器|64位server编译器|64位server编译器|
|mac os x|64位server编译器|64位server编译器|64位server编译器|

表格-不同os和机器上的默认编译器：
|操作系统 | 默认编译器|
|----|---|
|mac os，任意数量cpu| -server|
|linux64位，任意数量cpu| -server|

# 调优中级：
调优代码缓存
* 代码缓存是有最大值的资源，它会影响jvm可运行的编译代码的总量。 
* 分层编译很容易达到默认配置的上限（特别是java7），使用分层编译时应该监控代码缓存，必要时增大
* 代码缓存最大值： -XX:ReservedCodeCacheSize=N
* 代码缓存初始值： -XX:InitialCodeCacheSize=N
* 确认缓存设置：jcmd 47 VM.flags -all |grep CodeCacheSize
  
调优编译阈值
* 编译基于2种jvm计数器：方法调用计时器、方法中的循环回边计数器
* 标准编译：jvm执行方法时，检查两种计数器，然后判定是否适合编译。如果适合，加入编译队列。
* OSR(On-Stack-Replacement 栈上替换)：判定回边计数器 在循环进行时能编译循环，编译结束后替换栈上代码。
* 标准编译阈值设置：-XX:CompileThreshold=N client编译器默认值是1500，server默认值是10000
* 确认编译阈值：jcmd 47 VM.flags -all |grep CompileThreshold
* 检查编译过程最好的方法是开启PrintCompilation：  -XX:+PrintCompilation      
* 检查编译过程也可以使用jstat： jstat -printcompilation 47 1000
  
# 调优高级：
编译线程
* 分层编译时默认开启多个client和server线程，线程数根据cpu计算出来来的。
* 设置编译线程总数：-XX:CICompilerCount=N . 1/3(至少一个)用来出来client编译器队列，其余（至少1个）用来处理server编译器队列

内联：编译器最重要的优化是内联
* 内联默认开启， 可通过 -XX:-Inline 关闭
* 方法是否内联取决于它有多热以及它的大小
 * 方法很小时会内联（ -XX:MaxInlineSize=N 设置，默认35字节）
 * 方法很热并且字节码小于325字节（-XX:MaxFreqInlineSize=N 设置，默认325字节）  

逃逸分析
* 逃逸分析是编译器做的最复杂的优化
* 默认开启（ -XX:+DoEscapeAnalysis）
  
# 小结
* 从调优的角度，简单的选择是对所有应用使用server编译器和分层编译。这将解决90%与编译相关的性能问题。 只要代码缓存足够大，编译器就能提供尽善尽美的性能
*  关于final关键字影响性能的说法，在落伍的过去或许有一些价值，对于现在的编译器，是一个流传甚广的谣言。
*  不用担心小方法（eg getter setter） 因为编译器有内联优化
*  需要编译的代码在编译队列中，队列中代码越多，程序达到最佳性能的时间越久
*  虽然代码缓存大小可调整，但它仍然是有限资源
*  代码越简单，编译器优化越多，分析反馈和逃逸分析使代码更快。 复杂的循环结构和大方法限制它的有效性



