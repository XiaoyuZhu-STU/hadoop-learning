# HDFS 数据迁移

## 使用场景 
#### 冷热集群数据同步 分类存储
#### 集群数据整体搬迁
#### 数据准实时同步

## 考量因素
#### bandwidth 带宽
#### performance 性能
#### data increment 增量同步 （针对变化的增量数据同步）
#### syncable 数据迁移的同步性


## 分布式拷贝工具 DistCp(底层使用mapreduce 在集群之间或并行在同一集群内复制) 


## 安全模式 （safe mode）
#### 安全模式下文件系统处于一种可读不可写的特殊状态（针对NameNode 的维护状态）
#### 安全模式进入(启动后自动进入；)
#### 手动 hdfs dfsadmin -safemode get/enter/leave; 手动进入安全模式对于集群维护或者升级的时候非常有用，因为这时候HDFS上的数据是只读的。

## 负载平衡器 Balancer
#### 计算： 每个DataNode的利用率（本机已用空间与本机总容量之比）与集群的利用率（HDFS整体已用空间与HDFS集群总容量的比）之间相差不超过给定阈值百分比。
#### 运行 1、hdfs dfsadmin -setBalancerBandwith newbandwith(数据) 2、hdfs balancer (-threshold 5) 默认值 10%


## 磁盘均衡器 Disk Balancer (datanode 内部磁盘不均衡)
#### volume data density计算：1、ideal storage = total used/total capacity； 2、 volume data density = ideal storage - dfs used ratio;
#### volume Data Density的正值表示磁盘未充分利用，而负值表示磁盘相对于当前理想存储目标的利用率过高
#### 
$$
\test{Node Data Density}= \sum |\text{volume data density}| 
$$
