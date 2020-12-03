# linux
### 1、linux服务器load的具体含义是什么？
> 先说一下cpu的状态，按照进程执行过程中的不同情况分为 3态模型：
* 运行： 进程占用处理器，正在运行
* 就绪： 具备运行条件，等待系统分配处理器以便运行
* 等待：又称为阻塞或者睡眠状态，不具备运行条件，等待某个事件（eg io）完成的状态
  
>状态转移：
* 运行态→等待态：等待使用资源；如等待外设传输；等待人工干预。
* 等待态→就绪态：资源得到满足；如外设传输结束；人工干预完成。
* 运行态→就绪态：运行时间片到；出现有更高优先权进程。
* 就绪态—→运行态：CPU 空闲时选择一个就绪进程
[load 参考1](https://www.cnblogs.com/nandi001/p/11643414.html)
[load 参考2](https://blog.csdn.net/Qgwperfect/article/details/108521790)

* load是一段时间内 cpu 正在处理 以及 等待cpu处理的进程数之和的统计。
* load average 表示的是CPU的负载，包含的信息不是CPU的使用率状况，而是在一段时间内CPU正在处理以及等待CPU处理的进程数之和的统计信息，也就是CPU使用队列的长度的统计信息。
* 如果没有开启超线程，那么cpu线程数和cpu核数相同

### 2 linux系统中如何让进程在后台运行？
* 一般情况下 命令后加 & 即可
* 对于已经在执行的命令，可以 ctrl+z 暂停， 然后使用 bg 放到后台执行
* 前两种的缺点是退出终端shell时会发hangup信号给子进程，子进程收到后会退出。
如果想在退出shell终端后继续执行，需要使用nohup忽略hangup信号。eg : nohup java -jar xx.jar &

### 3 xargs 是做什么的？uniq？sed？
* xargs: 从标准输入中读取内容，将此内容传递给它需要协助的命令，作为那个命令的参数
一般与管道一起使用，可以将前一个命令的标准输出转换为命令参数。 而且xargs的标准输入中的 制表符，换行符，空格都将被空格取代
[xargs参考资料](http://c.biancheng.net/linux/xargs.html)
* uniq: 去重，一般与sort一起使用
* sed： 处理文本文件
* 
### 4 shell脚本统计java代码行数  
wc -l xx.txt

### 5 如何杀死服务器上所有的java进程？ 
ps aux |grep java |grep -v "grep" |awk '{print $2}' |xargs kill -9
