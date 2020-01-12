为了解决prometheus上k8s节点漂移 数据丢失的问题，调研了thanos，cortex，m3db三种开源解决方案

### thanos

#### 架构图

![architecture overview](https://thanos.io/img/arch.jpg)

#### 组件

##### Sidecar

和prometheus server一起部署 作为sidecar 负责读取持久化的数据 并上传到远端存储

##### Store Gateway 

负责在远端存储与Querier/Query沟通并提供指标

##### Compactor

在远端存储压缩和保留数据

##### Receiver

负责接收prometheus的remote write请求(这个在官方文档写的比较少 毕竟有了sidecar感觉这个作用就不是很大了 只是提供了remote write的能力)

##### Ruler/Rule

由于sidecar并没有足够的数据或者想对所有的数据执行一些告警或者聚合的时候 就需要使用到ruler这个模块

##### Querier/Query

实现了prometheus的v1版本的api 负责从远端存储查询数据 与store gateway沟通 通常是grafana与其对接

#### 总结

thanos解决了prometheus的数据长期远端问题 但是其也依赖一个对象存储 目前支持的有

![image-20191224170550701](/Users/swift/Library/Application Support/typora-user-images/image-20191224170550701.png)

参考哗啦啦的方案 他们是使用了ceph(支持s3)  但是ceph本身也没上云 在360这套行不通

考虑到之前有做过minio这套方案 如果把数据存在mino里面 不知道是否可行

##### 缺点

依赖对象存储 需要维护一套

##### 优点

Querier/Query层可以水平拓展 且对于多个prometheus实例的数据可以自动去重到对象存储 提供了全局的数据查询

### cortex

#### 架构图

![img](https://www.cncf.io/wp-content/uploads/2018/12/Cortex-blog-post-1024x921.png)

#### 组件

##### Distributor

负责处理prometheus的remote write请求 并把数据发送给ingesters(grpc协议) 多个distributor通过一致性hash位于一个环上 

The hash itself is based on one of two schemes:

1. The metric name and tenant ID
2. All the series labels and tenant ID

##### Ingesters

从distributor接收数据 并把数据压缩以及写到支持的存储方案中 目前支持四种 [AWS DynamoDB](https://aws.amazon.com/dynamodb), [AWS S3](https://aws.amazon.com/s3), [Apache Cassandra](https://cassandra.apache.org/), and [Google Cloud Bigtable](https://cloud.google.com/bigtable/).

##### Ruler

执行聚合规则 并生成报警 把他们发送给Alertmanager(与thanos比较类似)

##### Querier

负责处理客户端的promql请求

#### 总结

以上任意一个组件都可以被独立的管理，cortex的特点是支持多租户管理，其中后期的grafana上云可以考虑这个方案，比较适合。

像thanos一样 cortex也需要依赖远端存储方案(本地存储也可以 但是不够可靠) 其官方文档指出 生产级别的部署不应该使用单点部署 而是每个组件都可以独立部署 并且可以扩展 同样的 目前在360落地的可能也只有minio了 需要考虑维护成本

##### 优点

生产级别多个组件独立可拓展 支持多租户 没有thanos的sidecar 只是采用了remote write模式

##### 缺点

同thanos维护一套对象存储 提供的remote write&read功能是否会对写入和查询效率降低需要测试

引入了etcd 这里又多了一个operator

### m3

#### 架构图

![img](http://1fykyq3mdn5r21tpna3wkdyi-wpengine.netdna-ssl.com/wp-content/uploads/2018/08/image4-1.png)

#### 组件

##### Coordinator

sidecar组件 负责向后端存储执行读或者写操作

##### M3DB

分布式的时序数据库 提供了可横向扩展的存储能力

##### Query

分布式的查询引擎 针对一些实时和历史指标 支持不同的查询语言

##### Aggregator

基于etcd中的数据 提供降采样能力 减少采集指标的能力

#### 总结

根据m3的官方文档 如果是部署在单机 比较简单 挂载宿主机目录 启动m3db就可以了 对于生产级别的部署 官方提供了operator 这点比较友好  但是就像前两种方案一样 m3同样需要一个StorageClass

目前支持 

AWS EBS (class io1)  

Azure premium LRS

GCE Persistent SSD

如果想用自研的StorageClass 则需要自己维护对象存储

##### 优点

m3对于k8s的支持比较好 原生就是operator操作 同cortex一样 采用了remote read&write方案 虽然引入了sidecar 但是并没有引入本地存储 而是直接存在了m3db里面

##### 缺点

提供的remote write&read功能是否会对写入和查询效率降低需要测试 自行维护一套对象存储 

也可以考虑用minio

### 三种方案的总结

以上三种方案任意一种想要上k8s都需要维护一套对象存储 这套对象存储可以考虑用minio去做

m3自带operator 比较 cloud native 其他的在官方文档没有见到k8s相关的

结合现有的情况 prometheus上云有一下参考方案

1.通过把几台带ssd的机器加入k8s集群 并打上污点 专供监控实例调度 启动的时候以挂载hostpath的形式持久化保存监控数据

2.把之前做的那套minio作为存储实现 选择比较cloud native的m3(也可以其他 只要有了对象存储就行了)

参考qihoo现有的minio 还是基于ceph rbd做的...

参考别家minio的方案 有个人是用的local storage做的storage class。。。感觉又绕回去了

3.自行再开发一套新的对象存储...再选型...

4.参考prometheus的remote read&write支持方案 选择一套适用的

![image-20191225102635931](/Users/swift/Library/Application Support/typora-user-images/image-20191225102635931.png)