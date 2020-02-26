# 监控（未整理）



0.0 说zabbix的监控粒度不够细，原来是指的不能监控到cgroup这个层面、

zabbix cons：

* 大量数据时，关系型数据库压力大
* 监控容器需要通过api 或者自己写脚本获取，不够原生
* 监控粒度，无法直接监控到cgroup层面
* agent方式，push不好

Prometheus pro:

*  时序数据存在自带的**时序数据库**，便于聚合分析

  Prometheus 提供了两种数据持久化方式：**一种是本地存储，通过 Prometheus 自带的 TSDB（时序数据库），将数据保存到本地磁盘**，为了性能考虑，建议使用 SSD。但本地存储的容量毕竟有限，建议不要保存超过一个月的数据。Prometheus 本地存储经过多年改进，自 Prometheus 2.0 后提供的 V3 版本，TSDB 的性能已经非常高，可以支持单机每秒 1000 万个指标的收集。**另一种是远端存储，适用于大量历史监控数据的存储和查询**。通过中间层的适配器的转化，Prometheus 将数据保存到远端存储。适配器实现 Prometheus 存储的 remote write 和 remote read 接口，并把数据转化为远端存储支持的数据格式。目前，远端存储主要包括 OpenTSDB、InfluxDB、Elasticsearch、M3db、Kafka 等，其中 M3db 是目前非常受欢迎的后端存储。

* 云原生，pull方式
* 直接从cgroup获取信息（kubelet--cadvisor？）
* end

