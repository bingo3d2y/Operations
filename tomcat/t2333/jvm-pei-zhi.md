# jvm配置

 thanks for servyou

```text
/usr/local/logs/heapdump admin.admin
/usr/local/logs/gc_log    admin.admin
/usr/local/logs/tomcat/
JAVA_HOME=/usr/local/java/default/
PATH=$PATH:$JAVA_HOME/bin
​
​
JDK：# ll /usr/local/java/
total 8
lrwxrwxrwx 1 admin admin   13 Nov 10 12:05 default -> jdk1.8.0_121/
drwxr-xr-x 8 admin admin 4096 Dec 19  2014 jdk1.7.0_75
drwxr-xr-x 8 admin admin 4096 Dec 13  2016 jdk1.8.0_121
Tomcat：$ ll -d /usr/local/tomcat/
drwxr-xr-x 9 admin admin 4096 Nov 10 12:02 /usr/local/tomcat/
​
下面这些加到catalina.sh
MEM="-server -Xms2048m -Xmx2048m -XX:MaxPermSize=256m -XX:MaxMetaspaceSize=256m"
GC_OPTS="-XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSInitiatingOccupancyFraction=70  -XX:MaxTenuringThreshold=15 -XX:+Disa3bleExplicitGC -XX:+CMSParallelRemarkEnabled -verbose:gc -XX:+PrintGCDateStamps -XX:+PrintGCDetails -Xloggc:/usr/local/logs/gc_log/gc_$(date +%Y%m%d-%H%M).log  -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=20M"
OOM_OPTS="-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/usr/local/logs/heapdump/heapdump_$(date +%Y%m%d-%H%M).hprof -XX:OnOutOfMemoryError='curl http://192.168.9.217:8080/opmc-web/oom/warn?hostName=$(hostname)'"
JMX_OPTS="-Dcom.sun.management.jmxremote.port=18090 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false"
​
​
CATALINA_OPTS="$JAVA_OPTS $MEM $GC_OPTS $JMX_OPTS $OOM_OPTS -Djava.security.egd=file:/dev/./urandom"
​
MaxMetaspaceSize默认为-1，无限大。不过如果没有限制的话，一直增大会被系统干掉进程
PS： -Djava.security.egd=file:/dev/./urandom 
这个好屌0.0启动时随机时间，这个选项不设定tomcat/weblogic启动很慢
​
 UseConcMarkSweepGC：开启此参数使用ParNew & CMS（serial old为替补）搜集器。
 CMSInitiatingOccupancyFraction：触发CMS收集器的内存比例。比如70%的意思就是说，当内存达到70%，就会开始进行CMS并发收集。
 UseCMSCompactAtFullCollection：这个前面已经提过，用于在每一次CMS收集器清理垃圾后送一次内存整理。
 MaxTenuringThreshold：晋升老年代的最大年龄。默认为15，比如设为10，则对象在10次普通GC后将会被放入年老代。
```

end

