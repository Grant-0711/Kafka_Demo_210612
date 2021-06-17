

# 1.基础架构

https://www.processon.com/diagraming/60cb1a6a1e0853498cf2c84c

producer: 生产者[向kafka中写入消息]
topic: 主题,一般工作中一个业务对应一个主题
partition: 分区

**作用:**

1  分布式存储,便于扩容

2  提交并发,从而提高效率

consumer group: 消费者组。工作中一个消费者组消费一个topic数据

topic中一个分区的数只能被一个消费者组中的一个消费者所消费

consumer group中包含多个消费者,消费者的个数最好是与topic 分区数一致

​	如果consumer group中消费者个数>topic分区数,此时会有消费者没有分区数据可以消费

​	如果consumer group中消费者个数<topic分区数,此时会有消费者消费多个分区的数据

副本: 每个分区保存在不同的服务器上,如果服务器宕机,分区数据丢失,所以为了保证数据的安全性,提交了副本机制

leader: 副本的一个角色,Producer发送数据以及消费者消费数据都找leader

follower:副本的一个角色,主要负责同步leader的数据,如果leader宕机,会从follower中选举出新leader

broker: kafka的一台节点,分区的存放节点

offset: 偏移量,offset是数据在分区中的唯一标识,后续消费者组消费topic 分区数据的时候,会通过offset记录下一次应该从哪个位置开始消费        

到一定的个数，磁盘也达到一定转数，往磁盘里面顺序写（追加写）数据的速度和写内存的速度差不多`生产者生产消息，经过kafka服务先写到os cache 内存中，然后经过sync顺序写到磁盘上`



# 2.Kafka零拷贝机制保证读数据高性能

https://www.processon.com/diagraming/60cb24b75653bb3c31edd358

## **常规消费者读取数据流程**

1. 消费者发送请求给kafka服务
2. kafka服务去os cache缓存读取数据（缓存没有就去磁盘读取数据）
3. 从磁盘读取了数据到os cache缓存中
4. os cache复制数据到kafka应用程序中
5. kafka将数据（复制）发送到socket cache中
6. socket cache通过网卡传输给消费者

## **Kafka linux sendfile技术 — 零拷贝**

​	1.消费者发送请求给kafka服务 

​	2.kafka服务去os cache缓存读取数据（缓存没有就去磁盘读取数据） 

​	3.从磁盘读取了数据到os cache缓存中 

​	4.os cache直接将数据发送给网卡 

​	5.通过网卡将数据传输给消费者

## 二者对比

**Kafka linux sendfile技术 — 零拷贝技术不需要将数据读入socket cache中，而是直接从os cache发到网卡**



