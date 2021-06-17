<<<<<<< HEAD
# 第1章 Kafka概述

## 1.1 定义

Kafka是一个分布式的基于发布/订阅模式的消息队列（Message Queue），主要应用于大数据实时处理领域。

## 1.2 消息队列

### 1.2.1 使用消息队列的好处

1）解耦
两边可以独立修改，只需要同样的接口约束。

2）可恢复性
部分组件失效不会影响整个系统。

消息队列降低了进程间的耦合度，即使一个处理消息进程挂掉，加入队列中的消息仍可在系统恢复后被处理。

3）缓冲
解决生产消息和消费消息的处理速度不一致的情况。

4）灵活性 & 峰值处理能力
使用消息队列能够使关键组件顶住突发访问压力，而不会因为突发超负荷请求而崩溃。

5）异步通信
消息队列提供了异步处理机制，允许用户把消息放入队列，但不立即处理。想向队列中放入多少消息就放多少，然后在需要的时候再去处理它们。
=======
# 1.Kafka概述

## 1.1  kafka是什么?

​		kafka是基于发布/订阅的消息队列

## 1.2  kafka应用场景?

​		实时:kafka一般用于实时场景
​		离线:kafka在工作中有时候会结合flume进行使用,此时kafka用于削峰

## 1.3  基础架构

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

# 2.常用指令

## 2.1  topic相关

### 	2.1.1  创建topic

```shell
bin/kafka-topics.sh --create --topic topic名称 --partitions 分区数 --replication-factor 副本数 --bootstrap-server borker主机:9092,..
```

### 	2.1.2  查看kafka集群所有的topic

```shell
bin/kafka-topics.sh --list --bootstrap-server borker主机:9092,..
```

### 	2.1.3  查看某个topic的详细信息

```shell
bin/kafka-topics.sh --describe --topic topic名称 --bootstrap-server borker主机:9092,..
```

### 	2.1.4  修改topic信息

[只能修改分区数,而且是只能增加分区数,不能减少分区数]

```shell
bin/kafka-topics.sh --alter --topic topic名称 --bootstrap-server borker主机:9092,.. --partitions 分区数
```

### 	2.1.5  删除topic

```shell
bin/kafka-topics.sh --delete --topic topic名称 --bootstrap-server borker主机:9092,..
```

## 2.2  消费者/生产者相关	

### 	2.2.1  消费者相关

```shell
bin/kafka-console-consumer.sh --topic topic名称 --bootstrap-server borker主机:9092,..
```

### 	2.2.2  生产者相关

```shell
bin/kafka-console-producer.sh --topic topic名称 --broker-list borker主机:9092,..
```

### 	2.2.3  数据相关

```shell
bin/kafka-dump-log.sh --files 待查看的文件路径 --print-data-log
```

# 3.原理



## 3.1  存储结构



**Topic:** 逻辑上的概念

**partition:** 物理上,在磁盘中以目录的形式的存在,目录名: topic名称-分区号

**segment:** 逻辑上的概念,是分区的一个段

**index:** log文件的索引

**log:** 数据存储文件

**timeindex:** log文件数据的时间索引

**segment的命名规范:**
1.每个分区第一个segment的文件名=00000000000000000
2.第N个segment的文件名 = 第N-1个segment中最后一个offset+1

比如此时有一个分区有三个segment: 0000000000000000.0000000000000019.0000000000000031
0000000000000000segment中存放的是offet从0到18的数据
0000000000000019segment中存放的是offet从19到30的数据
0000000000000031segment中存放的是offet 31之后的数据

如何根据offset找到数据?
	1.根据segment文件名依据二分查找法,确定offset数据在哪个segment中
	2.根据segment文件的Index索引确定数据处于log文件哪个区间
	3.根据log文件扫描区间得到offset数据



## 3.2  生产者

### 3.2.1  分区的规则<生产者的数据究竟发到哪个分区中>

1.直接指定分区号: 数据发到指定分区中
2.没有指定分区号,但是有key: 数据发到 (key.hashCode % 分区数) 所在分区中
3.既没有分区号,也没有key:

**新版本:**
	1.第一个批次发送的时候,会生成一个随机数,第一个批次的数据发送 随机数% 分区数 所在分区中
	2.第N个批次的数据发送的时候,此时会排除掉第N-1个批次发送的分区号,然后从剩余分区中随机选择一个发送

**旧版本<轮询>:**
	1.第一个批次发送的时候,会生成一个随机数,第一个批次的数据发送 随机数% 分区数 所在分区中
	2.第N个批次的数据发送的时候,此时会发送到 (第一个批次生成的随机数+(N-1))% 分区数 所在分区中

### 3.2.2  数据可靠性<生产者发送的数据是否真实的到达kafka>

kafka通过ack机制<leader会返回确认消息给生产者,如果生产者没有收到leader的确认消息会重新发送数据>保证数据的可靠性

ack=0: leader接受到数据之后发送返回确认消息给生产者,此时leader数据可能还没有落盘
问题: leader返回确认消息之后宕机,因为数据还没有落盘,所有出现数据丢失

ack=1: leader接受到数据并且落盘之后在返回确认消息给生产者
问题: leader接受到数据并且落盘之后在返回确认消息给生产者之后宕机了,此时副本还没有同步数据,所以数据丢失

ack=-1: leader接受到数据并且落盘,而且所有的follower都同步数据之后才会返回消息给生产者

问题: leader接受到数据并且落盘,而且所有的follower都同步数据之后正准备返回确认消息的时候,leader宕机,此时会从follower中选举出新leader,因为原来的leader没有返回确认消息,生产者会认为数据丢失会重新发送数据给新leader，此时新leader有两份数据,数据重复

### 3.2.3  数据一致性<分区副本之间数据一致>

kafka通过ISR机制保证分区副本的一致性
后续选举leader的时候从ISR里面选举

ISR: 与leader同步到了一定程度<副本的LEO>=集群副本的HW>的follower集合

LEO: 每个副本最后一个offset

HW: 所有副本中最小的LEO[表述的是已经同步完成并且返回确认消息给了生产者]

副本故障之后重新加入ISR:
	1.leader故障: leader故障之后,首先会选举出新leader,此时其他的follower会将HW之后的所有数据清除重新从新leader同步数据,原来的leader故障解决之后,会清除宕机之前HW之后的所有数据,重新从新leader同步数据,同步到LEO>=集群HW的时候会重新加入ISR列表
	2.follower故障: follower故障之后,此时会清除HW之后的所有数据,重新从leader同步数据,同步到LEO>=集群HW的时候会重新加入ISR列表

### 3.2.4  exactly once

三种容错语义:

exactly-once: 同一条数据发到kafka的时候有且仅有一条[数据不重复也不丢失]
at-lest-once: 同一条数据发到kafka的时候至少一条[数据可能重复不丢失]
at-most-once： 同一条数据发到kafka的时候最多一条[数据可能丢失]

**exactly once的原理:** kafka借鉴mysql主键思想,每次发送数据的时候都会带上数据的主键[producerid+partition+sequenceNumber],broker接受到数据之后会将主键缓存,后续发送数据的时候首先会查看数据的主键在缓存中是否存在,如果存在代表该数据之前已经发送过,
此时会将数据标记为无效,如果不存在则标记为有效



**exactly once的前提:** ack=-1 and 开启幂等性 and 生产者不宕机
producerid: 生产者的唯一标识[producerid是在生产者启动的时候生成的]
partition： 分区号
sequenceNumber: 序列化号,代表发送到分区的第几条数据

## 3.3  消费者

​	3.3.1.消费数据方式: kafka采用主动拉取数据的方式
​	3.3.2.分区的分配策略:
​		1.轮询: 一个分区分配给一个消费者,轮着来
​			比如: 
​			Topic[partition0.partition1.partition2.partition3.partition4] 
​			consumer group[consumer1.consumer2]
​			此时轮询的分配结果:
​				consumer1消费partition0.partition2.partition4
​				consumer2消费partition1.partition3
​		2.range: 
​			1.确定每个消费者平均消费几个分区: 分区数/消费者个数
​			2.确定前几个消费者多消费一个分区: 分区数%消费者个数
​			比如: 
​			Topic[partition0.partition1.partition2.partition3.partition4] 
​			consumer group[consumer1.consumer2]
​			1.确定每个消费者平均消费几个分区: 分区数/消费者个数 = 5/2=2
​			2.确定前几个消费者多消费一个分区: 分区数%消费者个数 = 5%2=1
​			此时range的分配结果:
​				consumer1消费partition0.partition1.partition2
​				consumer2消费partition3.partition4
​	3.3.3.offset的保存
​		消费者组每次消费topic分区数据之后,都会记录该消费者组消费该分区的时候应该从哪个offset开始消费
​		0.9版本之前offset是保存在zookeeper里面
​		0.9版本之后offset是保存在kafka 内部topi****c<consumer_offsets>中

4、kafka高效读写
	1、分区: 提高并发,提高消费、写入效率
	2、顺序写磁盘: 减少磁头寻址的时间
	3、pagecache缓存
		1、生产者每个批次数据首先写入broker pagecache中,等到pagecache缓存空间不足的时候统一写入磁盘,
  相当于将单次写磁盘改为批次写磁盘,减少和磁盘打交道的次数
		2、pagecache中的批次数据会根据物理地址进行排序,排序之后写入磁盘可以减少磁头寻址的时间
		3、如果网络够好,生产者写入速率 = 消费者消费速率,此时数据刚刚写入page缓存中,消费者就直接从缓存中拉取数据消费了,不用经过磁盘
		4、pagecache不属于JVM管理,不会增加GC的负担
	4、零拷贝
		内存分为两块: 内核区、用户区
		从磁盘读取数据发送网络正常流程:
			1、通过io流读取磁盘数据,将数据读到内核区的pagecache中
			2、将内核区的pagecache中的数据拷贝到用户区,在用户区中对数据做处理
			3、将处理完成的数据通过io流发送到内核区的socket缓存区中
			4、将数据从socket缓存区发送到网卡
		零拷贝流程:
			1、通过io流读取磁盘数据,将数据读到内核区的pagecache中
			2、将数据从pagecache缓存区发送到网卡
5、zookeeper在kafka作用
	kafka集群的broker节点启动的时候会选举出contoller,controller负责broker监听、leader选举等都是依赖zookeeper
	leader选举流程:
		1、启动kafka broker节点,首先会从broker中选举出一个controller
		2、broker启动之后会在zk上注册: /brokers/ids, controller会监听该路径
		3、broker中保存有很多分区,这些分区在broker启动之后也会在zk上注册: /brokers/topics/topic名称/partitions/分区号/state
		4、如果broker宕机,该broker在 /brokers/ids上的临时节点消失,controller因为在监听在路径,所以第一时间就可以得知
		5、宕机的broker中如果有leader,此时会选举出新leader, controller会更新/brokers/topics/topic名称/partitions/分区号/state中分区leader信息
6、事务
	1、producer事务
		上述的exactly once的前提: ack=-1 and 开启幂等性 and 生产者不宕机
		如果生产者宕机,producerid以及分区号是会改变,所以生产者重启之后再发送数据,此时数据的主键已经改变,没办法保证数据有且仅有一条。
		所以kafka提供了一个事务机制。
		kafka消费者代码中会写死一个transaction.id配置，后续会将transaction.id的值与producerid绑定，将绑定结果保存在kafka内部topic中<transaction_state>。该topic中还会保存事务的状态。
		如果生产者宕机，宕机重启之后,首先会根据transaction.id从__transaction_state中获取producerid,然后再查看事务的状态,如果事务的状态不是commit,此时会回滚事务<将上一个事务的所有数据的isValid设置为false>,此时再次开启事务进行提交
	2、消费者事务: 精准一次消费
		kafka不能保证消费者精准一次消费

4、生产者消息发送流程
	1、创建producer,将消息封装成ProducerRecord对象
	2、通过send发送消息
	3、消息首先通过拦截器进行拦截处理
	4、通过序列化器对数据进行序列化
	5、通过分区器得知数据应该发送到哪个分区
	6、将数据发送到共享变量accumulator中,等到该topic 该分区积累一个批次之后由sender线程统一发送到broker中
5、消费者拉取数据消费完成之后会提交offset,提交的offset = 消费的数据的offset+1

# 4.生产者消息发送流程

​	1.创建producer,将消息封装成ProducerRecord对象
​	2.通过send发送消息
​	3.消息首先通过拦截器进行拦截处理
​	4.通过序列化器对数据进行序列化
​	5.通过分区器得知数据应该发送到哪个分区
​	6.将数据发送到共享变量accumulator中,等到该topic 该分区积累一个批次之后由sender线程统一发送到broker中
5.消费者拉取数据消费完成之后会提交offset,提交的offset = 消费的数据的offset+1

kafka系列-kafka调优篇-高并发高吞吐架构设计
boat824109722 2017-12-15 16:33:20 7221 收藏 4
分类专栏： kafka 文章标签： kafka 大数据 并发 消息队列
版权

 kafka的PageCache读写

 不同于Redis和MemcacheQ等内存消息队列，Kafka的设计是把所有的Message都要写入速度低容量大的硬盘，以此来换取更强的存储能力。实际上，Kafka使用硬盘并没有带来过多的性能损失（这一点是有条件限制的，这个条件是，消费者的消费速度要高于或等于生产者的速度）。

 


 kafka重度依赖底层操作系统提供的PageCache功能。（文件缓存，速度相当于操作内存）当上层有写操作时，操作系统只是将数据写入PageCache，同时标记Page属性为Dirty。当读操作发生时，先从PageCache中查找，如果发生缺页才进行磁盘调度，最终返回需要的数据。写入PageCache的数据被定期批量保存到文件系统，减少了磁盘的操作次数，减少系统开销。

 


 实际上PageCache是把尽可能多的空闲内存都当做了磁盘缓存来使用。同时如果有其他进程申请内存，回收PageCache的代价又很小，所以现代的OS都支持PageCache。使用PageCache功能同时可以避免JVM设计带来的GC开销大的问题。

 


 总结：

 1、kafka一开始是把数据写到PageCache，也就是缓存，如果消费者一直在消费，而且速度大于等于kafka的生产者发送数据的速度，那么消费者会一直从PageCache读取数据，速度等同于内存的操作，不会因为kafka写入磁盘的操作影响吞吐量。

 


 2、当kafka的消费者消费速度不及生产者生产速度时，PageCache存的数据已经是最新的数据了，kafka消费端需要的数据已经被存储磁盘中，这时，kafka的消费速度会受到磁盘的读取速度影响。

 


 问题：

 kafka的数据一开始就是存储在PageCache上的，定期flush到磁盘上的，也就是说，不是每个消息都被存储在磁盘了，如果出现断电或者机器故障等，PageCache上的数据就丢失了。

 


 kafka数据完整性保障：

 kafka是多副本的，当你配置了同步复制之后。多个副本的数据都在PageCache里面，出现多个副本同时挂掉的概率比1个副本挂掉的概率就很小了。（官方推荐是通过副本来保证数据的完整性的）

 


 同时kafka也提供了相关的配置参数，来让你在性能与可靠性之间权衡（一般默认）：

 #
 当达到下面的消息数量时，会将数据
 flush
 到日志文件中。默认
 10000

 log.flush.interval.messages=10000

 #
 当达到下面的时间
 (ms)
 时，执行一次强制的
 flush
 操作。
 interval.ms
 和
 interval.messages
 无论哪个达到，都会
 flush
 。默认
 3000ms

 log.flush.interval.ms=1000

 #
 检查是否需要将日志
 flush
 的时间间隔

 log.flush.scheduler.interval.ms = 3000

>>>>>>> 2f7d4eb929649ca91457c26ba851f10dde270a7e
