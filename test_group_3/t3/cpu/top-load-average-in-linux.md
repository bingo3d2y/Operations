# top load average in Linux

**top load average in Linux**

Most UNIX systems count only processes in the running \(on CPU\) or runnable \(waiting for CPU\) states. However, Linux also includes processes in uninterruptible sleep states \(usually waiting for disk activity\), which can lead to markedly different results if many processes remain blocked in I/O due to a busy or stalled I/O system.

Load average的概念源自UNIX系统，虽然各家的公式不尽相同，但都是用于衡量正在使用CPU的进程数量和正在等待CPU的进程数量，一句话就是runnable processes的数量。所以load average可以作为CPU瓶颈的参考指标，如果大于CPU的数量，说明CPU可能不够用了。

但是，GNU/Linux上不是这样的！

Linux上的load average除了包括正在使用CPU的进程数量和正在等待CPU的进程数量之外，还包括uninterruptible sleep的进程数量。通常等待IO设备、等待网络的时候，进程会处于uninterruptible sleep状态。Linux设计者的逻辑是，uninterruptible sleep应该都是很短暂的，很快就会恢复运行，所以被等同于runnable。然而uninterruptible sleep即使再短暂也是sleep，何况现实世界中uninterruptible sleep未必很短暂，大量的、或长时间的uninterruptible sleep通常意味着IO设备遇到了瓶颈。众所周知，sleep状态的进程是不需要CPU的，即使所有的CPU都空闲，正在sleep的进程也是运行不了的，所以sleep进程的数量绝对不适合用作衡量CPU负载的指标，Linux把uninterruptible sleep进程算进load average的做法直接颠覆了load average的本来意义。所以在**GNU/Linux系统上，load average这个指标基本失去了作用，因为你不知道它代表什么意思，当看到load average很高的时候，你不知道是runnable进程太多还是uninterruptible sleep进程太多，也就无法判断是CPU不够用还是IO设备有瓶颈。**

```text
$ taskset 1  dd if=/dev/vda of=/dev/null bs=1MB
$ taskset 2  dd if=/dev/vda of=/dev/null bs=1MB
$ top //取一段top的输出
top - 16:27:21 up 266 days,  7:00,  7 users,  load average: 1.97, 1.50, 1.02 //load几乎满了
Tasks: 130 total,   1 running, 129 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.2 us,  1.7 sy,  0.0 ni,  0.0 id, 98.2 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :  3881936 total,   133852 free,   327432 used,  3420652 buff/cache
KiB Swap:        0 total,        0 free,        0 used.  3280100 avail Mem 
​
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                                                    
 1116 root      20   0  108940   1364    500 D   1.3  0.0   0:03.72 dd                                                                                                                         
 1119 root      20   0  108940   1364    500 D   1.3  0.0   0:03.00 dd  
 <truncated>
 
0.0 D 表示的是进程什么状态呢。答案如下：
# man top 
S  --  Process Status
           The status of the task which can be one of:
               D = uninterruptible sleep 
               R = running
               S = sleeping
               T = stopped by job control signal
               t = stopped by debugger during trace
               Z = zombie
# D 即上面说的 Linux also includes processes in uninterruptible sleep states.所以，top的
load average 不能直接反应出是CPU瓶颈，也可能是IO瓶颈，不不应该说是IO wait
ps -ef |grep defunct|wc -l  //defunct表示僵尸进程
​
0.0 wow，就是不知道sar统计的包不包括Uninterruptible process了
$ sar -q 1
Linux 4.4.0-117-generic (fishong)   02/22/19    _x86_64_    (2 CPU)
​
14:11:28      runq-sz  plist-sz   ldavg-1   ldavg-5  ldavg-15   blocked
14:11:29            0       412      0.00      0.04      0.06         0
14:11:30            0       415      0.00      0.04      0.06         0
​
```

end

番外：

如果IO操作是需要CPU不停轮询访问检测有没有数据到来或者需要发送的那就会导致cpu占用率很高，得看IO具体实现。（0.0 一般都是 i/o wait~~ 不耗CPU，但是GNU/linux 会算到 cpu load ）

```text
# taskset 1  dd if=/dev/vda of=/dev/null bs=1MB
这样的属于I/O等待，不耗cpu的
# dstat -C 0 //查看cpu的状态 idl=0且usr+sys=0；wai+idl=100
You did not select any stats, using -cdngy by default.
-------cpu0-usage------ -dsk/total- -net/total- ---paging-- ---system--
usr sys idl wai hiq siq| read  writ| recv  send|  in   out | int   csw 
  0   0  99   0   0   0|5717B 3702B|   0     0 |   0     0 | 334   492 
  0   2   0  98   0   0|  46M    0 | 272B 1012B|   0     0 | 399   448 
  0   3   0  97   0   0|  56M   16k| 758B  658B|   0     0 | 468   479 
  0   3   0  97   0   0|  56M    0 | 758B  654B|   0     0 | 454   469 
  1   2   0  97   0   0|  42M    0 | 552B  658B|   0     0 | 401   453 
  0   2   0  98   0   0|  54M    0 | 478B  490B|   0     0 | 439   447 
  0   3   0  97   0   0|  40M    0 | 552B  564B|   0     0 | 394   438 
  0   2   0  98   0   0|  57M    0 | 832B  732B|   0     0 | 462   460 
  0   3   0  97   0   0|  54M    0 | 272B  490B|   0     0 | 427   427 
  0   1   0  99   0   0|  17M    0 | 890B  658B|   0     0 | 304   384 
  0   2   0  98   0   0|  55M    0 | 758B  658B|   0     0 | 452   463 
  0   2   0  98   0   0|  54M    0 | 140B  436B|   0     0 | 410   425 
  <truncated>
Another interesting experiment is to run a CPU bound task at the same time on the same CPU:
# taskset 1 sh -c "while true; do true; done"  //再给cpu指定一个任务可以充分利用cpu。
但是有些情况是真的CPU一直在检查io是不是结束了从而，系统真正是被io消耗了cpu
# dstat -C 0 //启动while任务，再次查看CPU状态。idl 全部是0且usr+sys=100
You did not select any stats, using -cdngy by default.
-------cpu0-usage------ -dsk/total- -net/total- ---paging-- ---system--
usr sys idl wai hiq siq| read  writ| recv  send|  in   out | int   csw 
  0   0  99   0   0   0|6189B 3702B|   0     0 |   0     0 | 334   492 
 99   1   0   0   0   0|  55M    0 | 478B 1086B|   0     0 |1365   392 
 99   1   0   0   0   0|  54M    0 | 758B  712B|   0     0 |1374   413 
 99   1   0   0   0   0|  54M   16k| 552B  584B|   0     0 |1364   436 
 99   1   0   0   0   0|  53M    0 | 684B  580B|   0     0 |1363   386 
100   0   0   0   0   0|  10M    0 | 346B  490B|   0     0 |1215   325 
 99   1   0   0   0   0|  44M    0 | 950B  826B|   0     0 |1337   387 
 98   2   0   0   0   0|  54M    0 | 684B  584B|   0     0 |1380   390 
 99   1   0   0   0   0|  54M    0 | 272B  510B|   0     0 |1365   410 
 99   1   0   0   0   0|  54M    0 |1038B  806B|   0     0 |1389   397 
 99   1   0   0   0   0|  54M  100k| 478B  510B|   0     0 |1356   386 
 99   1   0   0   0   0|  48M    0 | 272B  526B|   0     0 |1296   345 
 98   2   0   0   0   0|  54M    0 | 964B  806B|   0     0 |1335   355 
 99   1   0   0   0   0|  52M    0 | 272B  436B|   0     0 |1341   387 
<truncated>
综上，可知：I/O wait time is a sub-category of idle time. sub-category 子状态
wai 属于CPU idle的一种.
```

These findings allow us to deduce the exact definition of I/O wait time:

_For a given CPU, the I/O wait time is the time during which that CPU was idle \(i.e. didn’t execute any tasks\) and there was at least one outstanding disk I/O operation requested by a task scheduled on that CPU \(at the time it generated that I/O request\)_

> 当I/O wait time 发生时，CPU不执行任何task且至少有一个task \(outstanding disk I/O operation\)被调度到CPU上。
>
> at the time it generated that I/O request？？？0.0 这个time产生是由于I/O request。

end

