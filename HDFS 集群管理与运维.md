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
#### 计算 $\text{Node Data Density}= \sum |\text{volume data density in this datanode}| $
#### 较低的node Data Density值表示该机器节点具有较好的扩展性，而较高的值表示节点具有更倾斜的数据分布

## 纠删码技术
#### 3副本弊端：在复制因子为N时，存在N-1个容错能力，但存储效率仅为1/N。
#### 纠删码技术（ec）通过对数据进行分块，然后计算出校验数据，使得各个部分的数据产生关联性.
#### 编码和解码工作会消耗HDFS客户端和DataNode上的额外CPU。

## 动态扩容 节点上线
#### step1 新机环境准备： 主机名 ip  hosts映射 防火墙 时间同步 ssh免密 jdk
#### step2 hadoop配置
#### step3 手动启动datanode 进程
#### Step4：Web页面查看情况
#### Step5：DataNode负载均衡服务

## 动态缩容 节点下线
#### step1 添加退役节点 需要提前配置dfs.hosts.exclude属性
#### step2 Step2：刷新集群 hdfs dfsadmin -refreshNodes，等待退役节点状态为decommissioned（所有块已经复制完成）
#### step3 手动关闭datanode进程
#### Step4：DataNode负载均衡服务

## HDFS HA(高可用)
#### 给单点故障设置备份，形成主备架构。
#### 设计核心问题 脑裂问题（是当联系主备节点的"心跳线"断开时(即两个节点断开联系时)，本来为一个整体、动作协调的HA系统，就分裂成为两个独立的节点。由于相互失去了联系，主备节点之间像"裂脑人"一样，使得整个集群处于混乱状态）。 数据状态同步问题（主备节点之间的状态、数据是一致） 做法：通过日志重演操作记录。
#### Hadoop 单点故障 namenode。
#### 解决方案 QJM
