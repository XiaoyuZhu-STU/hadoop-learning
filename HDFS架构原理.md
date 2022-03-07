# HDFS 读写流程
## HDFS 写数据流程
#### pipeline ：客户端将数据块写入第一个数据节点，第一个数据节点保存数据之后再将块复制到第二个数据节点，后者保存后将其复制到第三个数据节点。（这样能够充分利用每个机器的带宽，避免网络瓶颈和高延迟时的连接，最小化推送所有数据的延时）
#### ACK 应答响应 ：在数据通信中，接收方发给发送方的一种传输类控制字符。表示发来的数据已确认接收无误。
#### 副本存储策略：第一优先本地，第二不同于第一的机架，第三与第二相同机架但不同机器。
####  流程：
###### 1、客户端通过Distributed FileSystem模块向namenode请求上传文件，namenode检查目标文件是否已存在，父目录是否存在。
###### 2、namenode返回是否可以上传。
###### 3、客户端请求第一个 block上传到哪几个datanode服务器上
###### 4、namenode返回3个datanode节点，分别为dn1、dn2、dn3
###### 5、客户端通过FSDataOutputStream模块请求dn1上传数据，dn1收到请求会继续调用dn2，然后dn2调用dn3，将这个通信管道建立完成。
###### 6、dn1、dn2、dn3逐级应答客户端。
###### 7、客户端开始往dn1上传第一个block（先从磁盘读取数据放到一个本地内存缓存，然后才会写入本地磁盘），以packet为单位，dn1收到一个packet就会传给dn2，dn2传给dn3；dn1每传一个packet会放入一个应答队列等待应答。
###### 8、当一个block传输完成之后，客户端再次请求namenode上传第二个block的服务器。（重复执行3-7步）。


## HDFS 读数据流程
####  流程：
###### 1、客户端通过Distributed FileSystem向namenode请求下载文件，namenode通过查询元数据，找到文件块所在的datanode地址。
###### 2、挑选一台datanode（就近原则，然后随机）服务器，请求读取数据。
###### 3、datanode开始传输数据给客户端（从磁盘里面读取数据输入流，以packet为单位来做校验）。
###### 4、客户端以packet为单位接收，先在本地缓存，然后写入目标文件。

# NameNode 元数据管理
#### 数据类型 1、文件自身属性信息（名称、权限、修改时间、文件大小、复制因子、数据块大小）；2、文件块位置映射信息（文件块位于哪个节点）。
#### 储存形式 1、内存元文件（内存中的元数据是最完整的，包括文件自身属性信息、文件块位置映射信息） 2、 元数据文件。
#### fsimage内存镜像文件 ：fsimage中仅包含Hadoop文件系统中文件自身属性相关的元数据信息，但不包含文件块的位置信息。
#### edits log 编辑日志：文件中记录的是HDFS所有更改操作（文件创建，删除或修改）的日志，文件系统客户端执行的更改操作首先会被记录到edits文件中。
#### 加载元数据文件顺序： 将fsimage文件加载到内存中，再执行edits文件中的操作。
####  当客户端对HDFS中的文件进行新增或者修改操作，操作记录首先被记入edits日志文件中，当客户端操作成功后，相应的元数据会更新到内存元数据中。
#### fsimage 文件查看 hdfs oiv -i (fsimage_00000000000000000000050) -p XML -o fsimage.xml.
#### edits 文件查看 hdfs oev -i (edits_00000000000000000-0000000000000000) -oedits.xml.

## SecondaryNameNode
#### 职责： 合并Namenode的edits logs 到fsimage文件中（减小edits logs文件大小和得到一个新的fsimage文件，这样也会减小Namenode上的压力）。
#### checkpoint 流程：
###### 1、当触发checkpoint操作条件时，SNN发送请求给NN滚动edits log。然后NN会生成一个新的编辑日志文件：edits new，便于记录后续操作记录。
###### 2、SNN会将旧的edits log文件和上次fsimage复制到自己本地（使用HTTP GET方式）。
###### 3、 SNN首先将fsimage载入到内存，然后一条一条地执行edits文件中的操作，使得内存中的fsimage不断更新，这个过程就是edits和fsimage文件合并。合并结束，SNN将内存中的数据dump生成一个新的fsimage文件。
###### 4、 SNN将新生成的Fsimage new文件复制到NN节点。至此刚好是一个轮回，等待下一次checkpoint触发SecondaryNameNode进行工作，一直这样循环操作。

## NameNode 元数据恢复
#### 从secondarynamenode恢复： SecondaryNameNode在checkpoint的时候会将fsimage和edits log下载到自己的本机上本地存储目录下。并且在checkpoint之后也不会进行删除。

# HDFS小文件解决方案
####  HDFS并不擅长存储小文件，因为每个文件最少一个block，每个block的元数据都会在NameNode占用内存，如果存在大量的小文件，它们会吃掉NameNode节点的大量内存。
## Hadoop archive 文件归档
#### Hadoop  archive可以把多个文件归档成为一个文件，归档成一个文件后还可以透明的访问每一个文件。
####  hadoop archive -archiveName name -p （parent） （src）* （dest） -archivenamenode 指创建文档的名称，拓展名为*.har
#### 查看归档之前的样子   hadoop fs -ls har://scheme-hostname:port/archivepath/fileinarchive (har://hdfs-node1:8020)
#### 提取 hadoop fs -cp har:///outputdir/test.har/* /smallfile1 （并行） hadoop distcp har:///outputdir/test.har/* /smallfile2

## sequence file 序列化文件

