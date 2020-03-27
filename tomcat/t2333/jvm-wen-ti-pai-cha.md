# JVM问题排查

java 应用太多了，作为一个Operator难免要和java打交道。

**分析java 耗CPU的原因**

1. 查出耗CPU的线程

   `jps -v` 可以拿到java pid

   ① `top && input 1` 查出耗CPU的进程

   ②定位线程：`top -Hp pid_num`

2. 分析线程

   ①将线程tid转换成十六进制:`printf "0x%x\n" thread_pid`

   ②查看该线程的运行情况: `jstack -l java_pid |grep 0x7866` \(0x+thread\_pid\_十六进制\)

    分析runnable状态的线程调用栈，这里分为两种情况：用户线程和系统线程

   常见的系统线程"VM Thread" 容易引起高CPU消耗:

   `"VM Thread" os_prio=0 tid=0x00007f25bc120800 nid=0xa26 runnable`

   或者

   `"Concurrent Mark-Sweep GC Thread" prio=10 tid=0x00007f1448029800 nid=0x2202 runnable`

    VM Thread"就是该cpu消耗较高的线程，VM Thread是JVM层面的一个线程，主要工作是对其他线程的创建，分配和对象的清理等工作的。

   用户线程引起的CPU Hang：

   可以分享jvm的调用栈（先进后出）：

   ```text
   [...]
   ​
   "TP-Processor31" daemon prio=10 tid=0x00002b1e4cb77000 nid=0x740f runnable [0x00002b1e5255f000]
    java.lang.Thread.State: RUNNABLE
    at java.net.SocketInputStream.socketRead0(Native Method)
   [...]
    at com.mysql.jdbc.util.ReadAheadInputStream.read(ReadAheadInputStream.java:189)
    - locked <0x00000007b79aa940> (a com.mysql.jdbc.util.ReadAheadInputStream)
    at com.mysql.jdbc.CompressedInputStream.readFully(CompressedInputStream.java:296)
   [...]
    at com.mysql.jdbc.MysqlIO.sqlQueryDirect(MysqlIO.java:2172)
    at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2696)
    - locked <0x00000007b79a5740> (a java.lang.Object)
    at com.mysql.jdbc.PreparedStatement.executeInternal(PreparedStatement.java:2105)
    at com.mysql.jdbc.PreparedStatement.executeQuery(PreparedStatement.java:2264)
    - locked <0x00000007b79a5740> (a java.lang.Object)
    at org.apache.tomcat.dbcp.dbcp.DelegatingPreparedStatement.executeQuery(DelegatingPreparedStatement.java:96)
    at org.apache.tomcat.dbcp.dbcp.DelegatingPreparedStatement.executeQuery(DelegatingPreparedStatement.java:96)
    at com.boxjar.MyService.doSomething(MyService.java:102)
    at com.boxjar.MyServlet.handleRequest(MyServlet.java:33)
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:617)
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:717)
   [...]
    at org.apache.tomcat.util.threads.ThreadPool$ControlRunnable.run(ThreadPool.java:690)
    at java.lang.Thread.run(Thread.java:662)
 
    -----
 
    at com.boxjar.MyService.doSomething(MyService.java:102) 表示在执行MyService.doSomething这方法，确这个方法在MyService这个类里面的102行。
   ​
   这个是java调用的堆栈（先进后出的堆栈，与之对应的是队列先进先出）。
   ​
   分析线程nid=0x740f的堆栈调用就可以知道那个块代码有问题了。
   ​
   这个可能不准，但是你可以学着分析
   ```

   end

3. 分析GC

   ①使用jstat观察gc情况: `jstat -gcutil java_pid 1000`

   &lt;interval&gt; 默认是ms，1000ms即1s.

   ```text
   $ jstat -gcutil 1 1000
   0.0 这个明显是有问题的-- 这个三个100 问题贼大--
     S0     S1     E      O      P     YGC     YGCT    FGC    FGCT     GCT   
     0.00 100.00 100.00 100.00      ? 133634 1103.378 90527 248406.829 249510.207
     0.00 100.00 100.00 100.00      ? 133634 1103.378 90527 248406.829 249510.207
     0.00 100.00 100.00 100.00      ? 133634 1103.378 90527 248406.829 249510.207
     0.00 100.00 100.00 100.00      ? 133634 1103.378 90527 248406.829 249510.207
     0.00 100.00 100.00 100.00      ? 133634 1103.378 90527 248406.829 249510.207
     0.00 100.00 100.00 100.00      ? 133634 1103.378 90527 248406.829 249510.207
     0.00 100.00 100.00 100.00      ? 133634 1103.378 90527 248406.829 249510.207
   ```

   end

    ②使用jmap查看哪些对象消耗这堆内存

    histo\[:live\] to print histogram of java object heap; if the "live" suboption is specified, only count live objects

    `jmap -histo java_pid`

   `jmap -heap java_pid` ： 可以看到jvm heap的详细使用情况。

4. end

