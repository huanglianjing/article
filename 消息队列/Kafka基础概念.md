# 1. 介绍

![kafka_logo](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/message_queue/kafka_logo.jpg)

Kafka是LinkedIn采用Scala开发的一个多分区、多副本、基于ZooKeeper协调的分布式消息系统，已被捐献给Apache基金会。kkokokokKafka定位为一个分布式流式处理平台，包含高吞吐、可持久化、可水平扩展、支持流数据处理等特性。

Kafka官网：[Apache Kafka](https://kafka.apache.org/)

Kafka源码：[github.com/apache/kafka](https://github.com/apache/kafka)

Kafka有三大角色：

- 消息系统：Kafka具备系统解耦、冗余存储、流量削峰、缓冲、异步通信、扩展性、可恢复性等功能。
- 存储系统：Kafka把消息持久化到磁盘，并且有多副本机制，相比内存存储的系统降低了数据丢失的风险。
- 流式处理平台：Kafka为流式处理框架提供了可靠的数据来源。



# 2. 基本概念

## 2.1 体系架构

![kafka_architecture](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/message_queue/kafka_architecture.png)

上图为Kafka的体系架构。一个Kafka体系架构包含若干Producer、若干Broker、若干Consumer，以及一个ZooKeeper集群。

- ZooKeeper：负责集群元数据的管理、控制器的选举。
- 生产者Producer：将消息发送到Broker。
- 消费者Consumer：从Broker订阅主题并消费消息。
- 服务代理节点Broker：将收到的消息存储到磁盘。Broker可以看作一个Kafka服务节点或Kafka服务实例，可以将多个Broker运行在不同的服务器上，也可以运行在同一个服务器但是配置不同的端口。

## 2.2 消息和批次

#### 消息

Kafka的数据单元被称为消息，消息由字节数组组成，对与Kafka来说，消息里的数据没有特别的格式或含义。消息又一个可选的元数据，也就是键，键也是一个字节数组。当消息写入不同分区时会用到键，如为键生成一个散列值，再对主题的分区数取模，同一个键的消息就会被写入相同的分区。

#### 批次

批次就是一组消息，且它们属于同一个主题和分区。将消息分批次写入Kafka，可以减少网络开销，提高效率。

## 2.3 主题和分区

#### 主题Topic

Kafka的消息以主题进行归类，这是一个逻辑上的概念。生产者将消息发送到特定主题，而消费则订阅指定的主题并进行消费以获取消息。

#### 分区Partition

一个主题可以分为多个分区，而一个分区只属于某个主题。同一个主题下，不同分区包含的消息是不同的，分区在存储层面可以看做一个可追加的日志文件，消息追加到分区日志文件后会分配一个特定的偏移量offset。offset是消息在分区的唯一标识，offset可以保证消息在分区内的顺序性，每个分区有分别的offset，在同一个分区的消息将是先进先出的，因此Kafka保证的是分区有序而不是主题有序。

一个主题的各个分区可以分布在不同的broker上，每条消息被发送到broker前，会根据分区规则选择存储到哪个具体的分区。因此在创建主题的时候通过参数指定分区的个数，或者创建完成后修改分区数量，以实现分区水平扩展，突破机器IO的性能瓶颈。

目前Kafka只支持增加分区数，不支持减少分区数。

#### 副本Replica

分区的多副本机制可以增加副本数量以提升容灾能力，同一个分区拥有的副本数量称为复制系数/副本因子。同一个分区的不同副本保存相同的消息，副本之间为一主多从的关系，其中leader副本负责处理读写请求，follower副本只负责与leader副本的消息同步。副本处于不同的broker重，当leader副本出现故障时，便从follower副本重新选举新的leader副本对外提供服务。

下图展示了在一个有4个broker的集群中，一个主题配置了3个分区P1、P2、P3。复制系数为3，即每个分区包含有3个副本，一个是leader副本两个是follower副本，这3个副本存储在不同的broker中。

![kafka_partition_replica](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/message_queue/kafka_partition_replica.png)

分区的所有副本统称为AR（Assigned Replicas），leader副本以及与leader副本保持一定程度同步的副本组成ISR（In-Sync Replicas），与leader副本同步滞后过多的副本组成OSR（Out-of-Sync Replicas），因此AR=ISR+OSR。leader副本维护和跟踪ISR集合与自己的滞后状态，滞后太多或失效的副本会被从ISR剔除，变为失效副本，OSR中的副本追上leader副本则会被转移至ISR。默认配置下，leader副本发生故障后只会在ISR集合中选举新leader。

#### 偏移量offset

偏移量指消息在日志文件中的相对位置，一个分区中每个消息的偏移量都是唯一的。偏移量会被保存在Kafka内部主题__consumer_offsets中。

以下是三个有含义的偏移量：LogStartOffset、HW、LEO。

- LogStartOffset为0，是日志文件的起始处，也就是第一条消息。
- HW（High Watermark）俗称高水位，标识了一个offset，消费者只能拉取到HW之前的消息。
- LEO（Log End Offset）标识当前日志文件下一条待写入消息的offset，相当于当前日志分区最后一条消息的offset加1。分区的ISR集合每个副本都会维护自身的LEO，而ISR集合中最小的LEO就是分区的HW。

下图展示了一个日志文件，其中HW为6，LEO为9，因此消费者只能拉取偏移量0到5的消息，而下一条写入的消息偏移量将会是9。

![kafka_offset](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/message_queue/kafka_offset.png)

## 2.4 生产者和消费者

#### 生产者

生产者创建消息，并且将消息发不到一个特定的主题。生产者会把消息均衡地分布到主题的所有分区上，分区的选择是通过消息键和分区器来实现的，也可以通过自定义分区器来实现分区选择。

#### 消费者

消费者读取消息，消费者需要订阅一个或多个主题，并按消息生成的顺序读取它们，消费者通过检查消息的偏移量来区分已经读过的消息，从而从该处继续往后读取消息。消费者把每个分区最后读取的消息偏移量保存在ZooKeeper或Kafka上，即使消费者关闭或重启，读取状态也不会丢失。

## 2.5 broker和集群

#### broker

broker是一个独立的Kafka服务器，它接收来自生产者的消息，设置消息的偏移量，并提交消息到磁盘保存。broker为消费者提供服务，对读取分区请求作出响应，返回提交到磁盘上的消息。

#### 集群

broker是集群的组成部分，每个集群都有一个broker充当集群控制器（Controller）。控制器负责管理工作，如将分区分配给broker，监控broker状态。



# 3. 服务端

## 3.1 协议设计

Kafka自定义了一组基于TCP的二进制协议，只要遵守这组协议的格式，就可以向Kafka收发消息。目前包含数十种协议类型，每种协议类型都有对应的请求（Request）和响应（Response）。

#### 协议请求头

![kafka_request_format](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/message_queue/kafka_request_format.png)

协议请求头包含四个域（Field）：

| 域（Field）    | 描述（Description）                                          |
| -------------- | ------------------------------------------------------------ |
| api_key        | API标识，如PRODUCE、FETCH等表示发送和拉取消息的请求          |
| api_version    | API版本号                                                    |
| correlation_id | 客户端指定的唯一标识请求id，服务端处理完请求会把同样correlation_id回写到响应 |
| client_id      | 客户端id                                                     |

#### 协议响应头

![kafka_response_format](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/message_queue/kafka_response_format.png)

协议响应头只有correlation_id，对应发送时的请求头中的correlation_id。

## 3.2 控制器

Kafka集群中有一个或多个broker，其中有一个broker会被选举为控制器，负责管理整个集群所有分区和副本的状态。其职责有：

- 从ZooKeeper读取主题、分区、broker相关信息并管理
- 更新集群元数据
- 监听主题变化
- 监听分区变化
- 监听broker变化
- 启动并管理分区状态机和副本状态机 
- 分区的新leader副本选举
- 分区ISR集合变化元数据同步
- 维护分区的leader副本的均衡

#### 控制器的选举

Kafka基于ZooKeeper选举控制器，成功竞选控制器的broker在ZooKeeper中创建临时节点/controller。每个broker启动的时候会尝试读取/controller节点的brokerid值，若不是-1则表示已经有控制器，若不存在该节点则会尝试创建。创建成功的broker称为控制器，创建失败的broker竞选失败。

每个broker会持续监听/controller节点。在控制器broker故障或退出时，注册的/controller节点失效，其他broker会抢占地再去重新注册/controller节点，成功注册的一个broker成为新的控制器，其他broker竞选失败。也可以手动删除/controller节点来触发新一轮的选举。

ZooKeeper中还有一个持久节点/controller_epoch，存放整型的controller_epoch值，用于记录控制器发生变更的次数，起始值为1，每次选出新控制器就会加1。每个和控制器交互的请求都会携带controller_epoch，小于该值的请求被认为是无效的请求。通过controller_epoch保证了控制器的唯一性。

#### 监听ZooKeeper

只有控制器会在ZooKeeper注册相应的监听器，其他broker很少需要监听ZooKeeper。但每个broker还是会对/controller节点添加监听器，以选举控制器。

#### 优雅关闭

通过脚本工具kafka-server-stop.sh关闭kafka，实际上是发送SIGTERM信号给Kafka进程。Kafka进程有个关闭勾子kafka-shutdown-hock，捕获终止信号后，会正常关闭一些必要的资源，消息完全同步到磁盘，执行控制关闭（ControllerShutdown）的动作，ControllerShutdown包含有重新选举leader副本和ISR副本、leader副本的迁移等操作，然后才关闭进程。



# 4. 生产者

## 4.1 整体架构

Kafka生产者客户端整体架构如下图所示：

![kafka_producer](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/message_queue/kafka_producer.png)

整个生产者客户端有两个线程协调运行，分别为主线程和Sender线程。

**主线程**中有KafkaProducer创建消息，然后通过可能的拦截器、序列化器和分区器的作用之后缓存到消息累加器（RecordAccumulator，也称为消息收集器）中，拦截器不是必须的，序列化器是必须的，分区器在消息指定了partition字段时就不需要。

**消息累加器**主要用来缓存消息以便Sender线程可以批量发送，进而减少网络传输的资源消耗以提升性能。在RecordAccumulator的内部为每个分区都维护了一个双端队列Deque\<ProducerBatch\>，主线程中发送来的消息会被追加到某个双端队列的尾部，Sender线程则从双端队列的头部读取消息。这里ProducerRecord是生产者创建的消息，而ProducerBatch是一个消息批次，可以减少网络请求次数以提升整体吞吐量。

**Sender线程**负责从消息累加器中获取消息并将其发送到 Kafka 中，其中的InFlightRequests缓存已经发往Kafka但还没收到响应的请求。

## 4.2 生产者参数

#### acks

这个参数用来指定分区中必须有多少个副本收到这条消息，生产者才会认为消息是成功写入的，acks的值涉及消息的可靠性和吞吐量之间的权衡。

- acks = 1。默认值，是消息可靠性和吞吐量的折中方案。生产者发送消息之后，只要分区的leader副本成功写入消息，就会收到服务端的成功响应。
- acks = 0。生产者发送消息之后不需要等待任何服务端的响应。可以达到最大吞吐量。
- acks = -1 或 acks = all。生产者发送消息之后，需要等待ISR的所有副本成功写入消息才能收到服务端的成功响应。可以达到最强可靠性。

#### max.request.size

限制生产者客户端能发送的消息的最大值，默认为1048576B，即1MB。

#### retires和retry.backoff.ms

生产者重试次数，默认值为0。

## 4.3 拦截器

生产者拦截题（Interceptor）既可以用来在消息发送前做一些准备工作，比如按照某个规则过滤不符合要求的消息、修改消息的内容等，也可以用来在发送回调逻辑前做一些定制化的需求，比如统计类工作。

生产者拦截器的使用也很方便，主要是自定义实现org.apache.kafka.clients.producer. ProducerInterceptor接口。ProducerInterceptor接口中包含3个方法：

```java
public ProducerRecord<K, V> onSend(ProducerRecord<K, V> record);
public void onAcknowledgement(RecordMetadata metadata, Exception exception);
public void close();
```

KafkaProducer在将消息序列化和计算分区之前会调用生产者拦截器的onSend()方法来对消息进行相应的定制化操作。KafkaProducer会在消息被应答（Acknowledgement）之前或消息发送失败时调用生产者拦截器的onAcknowledgement()方法，优先于用户设定的Callback之前执行。close()方法用于在关闭拦截器时执行一些资源的清理工作。

## 4.4 序列化器

序列化器（Serializer）把对象转换成字节数组，以通过网络发送给Kafka。而在对侧，消费者需要用反序列化器（Deserializer）把从Kafka收到的字节数组转换成相应的对象。

生产者使用的序列化器和消费者使用的反序列化器是需要一一对应的，否则将复发解析出想要的数据。

序列化器都需要实现org.apache.kafka.common.serialization.Serializer 接口，此接口有3个方法：

```java
public void configure(Map<String, ?> configs, boolean isKey)
public byte[] serialize(String topic, T data)
public void close()
```

configure()方法用来配置当前类，serialize()方法用来执行序列化操作，而close()方法用来关闭当前的序列化器。

如果Kafka客户端提供的几种序列化器都无法满足应用需求，则可以选择使用如Avro、JSON、Thrift、ProtoBuf和Protostuff等通用的序列化工具来实现，或者使用自定义类型的序列化器来实现。

## 4.5 分区器

分区器（Partitioner）根据key的字段来计算partition的值，也就是索要发往的分区号，但是若消息ProducerRecord中指定partition字段时则不需要分区器。

Kafka中提供的默认分区器是org.apache.kafka.clients.producer.internals.DefaultPartitioner，它实现了org.apache.kafka.clients.producer.Partitioner接口，这个接口中定义了2个方法，具体如下所示。

```java
public int partition(String topic, Object key, byte[] keyBytes, 
                     Object value, byte[] valueBytes, Cluster cluster);
public void close();
```

其中partition()方法用来计算分区号，返回值为int类型。close()方法在关闭分区器的时候用来回收一些资源。

除了使用Kafka提供的默认分区器，还可以使用自定义的分区器，只需同DefaultPartitioner一样实现Partitioner接口即可。



# 5. 消费者

## 5.1 消费者与消费组

消费者负责订阅Kafka中的主题，并从主题中拉取消息。

每个消费者都有一个对应的消费组，又称为消费者群组，消息发布到主题后，只会被投递给订阅了该主题的每个消费组中的一个消费者。多个消费组订阅了某个主题后，互相之间不影响，每个消费组自行决定如何分配包含的各个消费者能消费的分区，而每个消费者只能消费被消费组分配的分区。

![kafka_2_consumer_group](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/message_queue/kafka_2_consumer_group.png)

如上图所示，该主题有4个分区。有两个消费组对其进行消费，他们分别决定如何给自己的消费者分配分区。消费组A的四个消费者分别消费一个分区，而消费组B的两个消费者分别消费两个分区。

当消费组的消费者数量为1时，该消费者消费主题所有的分区。当消费者数量小于分区数量时，分区被平均地分配给消费者，但是有时候无法完全平均，如10个分区分配给3个消费者，3个消费者会分别消费4、3、3个分区。当消费者数量等于分区数量时，正好每个消费者消费一个分区。当消费者数量大于分区数量时，由于每个分区只能由消费组的一个分区消费，因此多出的消费者将不会被分配分区，如下图所示。

![kafka_consumer_more_than_partition](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/message_queue/kafka_consumer_more_than_partition.png)

消费组可以通过增加或减少消费者个数来提高或降低整体的消费能力，具有横向伸缩性。但是超过分区数的消费者将没有意义。

实现消息中间件的两种消息投递模式：

- 点对点模式（P2P）：所有消费者都属于一个消费组，所有消息都会被均衡地投递给每个消费者，即每条消息只会被一个消费者处理。
- 发布/订阅模式（Pub/Sub）：所有消费者都属于不同的消费组，所有消息都会被广播给所有消费者，即每条消息会被所有消费者处理。

## 5.2 分区再均衡

消费组的消费者共同读取一个主题的分区，在一个新的消费者加入消费组时，消费者被关闭或发生崩溃离开群组时，还有主题添加分区时，会发生分区的重新分配。分区的所有权从一个消费者转移到另一个消费者，这种行为被称为再均衡。

再均衡给消费者群组带来了高可用性和伸缩性。在再均衡期间，消费者无法读取消息，造成整个消费组内的消费者一小段时间的不可用。当某个消费者消费完部分消息时，还没来得及提交消费位移就发生再均衡操作时，这部分消息将会被重复消费。所以一般应尽量避免不必要的再均衡的发生。

消费者通过向群组协调器broker发送心跳来维持和消费组的从属关系和对分区的所有权关系，如果消费者停止发送心跳时间足够长，会话就会过期，群组协调器会认为它已经死亡，从而触发一次再均衡。

## 5.3 消费者客户端

### 5.3.1 参数配置

#### bootstrap.servers

指定Kafka集群的broker地址清单。

#### group.id

索菲着所属的消费组名称，默认值为""。

#### key.deserializer和value.deserializer

与生产者客户端的key.serializer和value.serializer参数对应，才能将消息的字节数组反序列化称原有的对象格式。

#### auto.offset.reset

从何处开始进行消费，默认值latest表示从分区末尾开始消费消息，earliest表示从0开始消费。

#### enable.auto.commit

让消费者基于任务调度自动提交偏移量。若消费者中途宕机，会出现这段时间消费了却没有提交offset，造成重复消费。

#### auto.commit.interval.ms

自动提交偏移量的频度，默认每5秒钟一次。

### 5.3.2 消费逻辑

- 订阅主题：可以以集合或正则表达式的形式订阅多个主题。
- 反序列化
- 消息消费：Kafka的消费是基于拉模式的，由消费者主动向服务端发起请求拉取消息。通过不断轮训调用poll()方法拉取消息。
- 位移提交：调用poll()方法返回的是还没被消费过的消息集，消费完消息后需要执行消费位移的提交，提交的消费位移是已经消费的位置+1，表示下一条需要拉取的消息的位置。默认提交方式为自动提交，不是每消费一条提交一次，而是定时提交，有重复消费和消息丢失的风险。
- 关闭消费

## 5.4 分区分配策略

分区分配策略指的是，一个主题的多个分区，以何种策略分配给一个消费组的多个消费者。由消费者客户端参数partition.assignment.strategy设置消费者与订阅主题之间的分区分配策略，默认为RangeAssignor分配策略，也可配置多个分配策略并以逗号分隔。

### RangeAssignor分配策略

按照消费者总数和分区总数整除运算获得跨度，将分区按跨度平均分配。将消费组内所有订阅该主题的消费者按名称字典序排序，每个消费者划分固定分区范围，不够平均分配时字典序靠前的消费者会多分配一个分区。

假设有2个消费者C0和C1，订阅了主题t0和t1，每个主题都有4个分区p0、p1、p2、p3，分配结果如下：

消费者C0：t0p0、t0p1、t1p0、t1p1

消费者C1：t0p2、t0p3、t1p2、t1p3

但是当每个主题都有3个分区时，分配结果如下，会出现分配不均：

消费者C0：t0p0、t0p1、t1p0、t1p1

消费者C1：t0p2、t1p2

### RoundRobinAssignor分配策略

将消费组的所有消费者及其订阅的所有主题的分区按字典许排序，通过轮询逐个将分区分配给消费者。

解决了RangeAssignor分配策略中每个主题3个分区的不均衡问题，分配结果如下：

消费者C0：t0p0、t0p2、t1p1

消费者C1：t0p1、t1p0、t1p2

但是当消费组内各消费者订阅的信息不同时，还有可能导致分区分配不均。

### StickyAssignor分配策略

黏性的分配策略，既要使分配尽可能均匀，也在重新分配时尽可能与上次分配保持相同，而不是每次重新排序分配。该分配策略更优异，代码实现也异常复杂。

### 自定义分配策略

通过实现PartitionAssignor接口来自定义分配策略。

## 5.5 事务

消息中间件的消息传输保障有3个层级：

- at most once：至多一次，消息可能丢失，但不会重复传输
- at least once：最少一次，消息绝不会丢失，但可能重复传输
- exactly once：恰好一次，每条消息肯定而且只会传输一次

生产者向Kafka发送消息时，一旦消息被提交到日志文件，由于多副本机制，这条消息就不会丢失。在网络问题造成生产者无法判断消息是否已经提交，可以进行多次重试来确保消息写入，重试可能会造成消息的重复写入，这里的消息保障为at least once。

对于消费者，如果消费者在拉取完消息后，先处理消息再提交消费位移，则中途消费者宕机，重启后会从同一位置拉取，出现重复消费，此时是at least once。如果消费者在拉完消息，先提交消费位移再处理消息，中途宕机重启后从提交的位移开始重新消费，之前有部分消息未消费，发生消息丢失，此时是at most once。

#### 幂等

生产者客户端参数enable.idempotence设置为true开启幂等性功能，可以避免生产者重试时重复写入消息。

#### 事务

幂等性不能跨多个分区运作，事务可以弥补这个缺陷，保证对多个分区写入操作的原子性，事务需要生产者开启幂等性。

事务的实现过程为：

- 生产者查找事务协调器（TransactionCoordinator）所在broker节点
- 生产者提供唯一的transactionalId，请求事务协调器获取PID，transactionalId与PID一一对应
- 开启事务
- 事务处理过程
- 提交或终止事务



# 6 日志

Kafka的中的消息是以主题进行归类的，各个主题在逻辑上相互独立，每个主题又可以分为一个或多个分区，每条消息会根据分区规则追加到指定分区。Kafka中的消息是存储在磁盘上的，每个分区的每个副本对应一个日志（Log）。为了防止Log过大，将Log切分为多个日志分段（LogSegment）。

## 6.1 日志结构

Log在物理上以文件夹的形式存储，而每个LogSegment对应于磁盘上的一个日志文件和两个索引文件，以及可能的其他文件如.deleted、.cleaned、.swap等。文件结构如下图所示：

![kafka_log_structure](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/message_queue/kafka_log_structure.png)

每个LogSegment的.log（日志文件）都对应两个索引文件：.index（偏移量索引文件）和.timeindex（时间戳索引文件），每个LogSegment都有一个基准偏移量baseOffset，表示当前LogSegment的第一条消息的偏移量，偏移量是一个64位长整形数，日志文件和两个索引文件都以基准偏移量命名，长度为20位数字，前面以0填充。

如第一个LogSegment的基准偏移量为0，则有00000000000000000000.log、00000000000000000000.index、00000000000000000000.timeindex这三个文件。当当前LogSegment到达一定条件时会创建新的LogSegment，追加消息只能写到最新的LogSegment，也称为activeSegment，就会在文件夹中新建如00000000000000001000.log的三个文件，文件以新的基准偏移量命名。

## 6.2 日志存储文件

日志文件存储的文件夹由配置文件决定，在配置文件config/server.properties中可以看到：

```properties
log.dirs=/tmp/kafka-logs
```

在日志文件夹/tmp/kafka-logs中，分区的文件存储在命名为`<topic>-<partition>`的文件夹中，如现在有主题test以及分区0，对应文件夹test-0：

```bash
$ cd /tmp/kafka-logs/test-0/
$ l
总用量 8
-rw-r--r-- 1 root root 10485760 8月   7 17:45 00000000000000000000.index
-rw-r--r-- 1 root root      218 8月   7 17:45 00000000000000000000.log
-rw-r--r-- 1 root root 10485756 8月   7 17:45 00000000000000000000.timeindex
-rw-r--r-- 1 root root        8 8月   7 17:45 leader-epoch-checkpoint
```

这三个文件就是基准偏移量为0的三个LogSegment文件。

## 6.3 日志格式

日志格式指的是消息的格式，最新的v2版本格式如下：

![kafka_log_format](https://article-1304941664.cos.ap-guangzhou.myqcloud.com/message_queue/kafka_log_format.png)

#### 消息压缩

Kafka会将多条消息一起进行压缩，生产者发送的压缩数据在broker中保持压缩状态进行存储，消费者从服务端获取到的也是压缩的消息，消费者在处理消息之前才会解压消息，这样保持了端到端的压缩。使用的压缩方式由配置参数compression.type决定，默认producer指保留生产者使用的压缩方式，还可以配置为gzip、snappy、lz4，分别对应不同压缩算法，而配置uncompressed表示不压缩。

#### 变长字段

Kafka引入了变长整形（Varints）和ZigZag编码。

Varints使用一个或多个字节来序列化整数，每个字节都以最高1位作为msb位（most significant bit），后面7位勇于存储数据。前面各个字节的msb位为1，最后一个字节的msb位为0。因此这种表示整形又称为Base 128。

Varints使用ZigZag的编码方式，以一种锯齿形的方式来回穿梭正负数，以使绝对值较小的负数有较小的编码值。

## 6.4 日志索引

Kafka的索引文件以稀疏索引的方式构造消息的索引，它并不保证每个消息在索引文件中都有对应索引项。根据配置log.index.interval.bytes（默认为4096），，每当写入一定量的消息，偏移量索引文件和时间戳索引文件分别增加一个索引项。稀疏索引通过MappedByteBuffer将索引文件映射到内存中，加快查询速度，由于偏移量是单调递增的，因此会使用二分查找快速定位。

#### 偏移量索引

偏移量索引的索引项占8字节，包含：

- relativeOffset：相对偏移量，4字节，消息相对baseOffset的偏移量
- position：物理地址，4字节，消息在日志分段文件的物理地址

#### 时间戳索引

时间戳索引的索引项占12字节，包含：

- timestamp：当前日志分段最大的时间戳
- relativeOffset：时间戳对应消息的相对偏移量

## 6.5 日志清理

Kafka有两种日志清理策略：

- 日志删除（Log Retention）：按一定策略删除不符合条件的日志分段
- 日志压缩（Log Compaction）：对每个消息的key进行整合，相同key指保留最后一个版本的value值

由配置参数log.cleanup.policy设置日志清理策略，delete位日志删除，compact为日志压缩。

#### 日志删除

Kafka的日志管理器有一个专门的日志删除任务来周期性地检测和删除不符合保留条件的日志分段文件，时间周期由log.retention.check.interval.ms配置。日志保留策略有3种：基于时间、基于日志大小、基于日志起始偏移量。

日志分段中的activeSegment永远不会被删除。

## 6.6 磁盘存储

相比传统的消息中间件RabbitMQ使用内存作为存储介质，Kafka使用磁盘存储和缓存消息，Kafka通过以下几方面实现高吞吐：

1. Kafka使用文件追加的方式写入消息，并且不允许修改已写入的消息，这种属于顺序写盘操作，能承载不小的吞吐量。
2. Kafka大量使用页缓存，由操作系统负责具体的刷盘任务。Kafka提供了同步刷盘及间断性强制刷盘（fsync）的功能，提高消息可靠性，但不建议这么做，同步刷盘严重影响性能，消息可靠性应该由多副本机制来保障。
3. Kafka还使用零拷贝（Zero-Copy）进一步提升性能。将数据直接从磁盘文件复制到网卡设备中，不需要经过应用程序，从而减少内核和用户模式之间的上下文切换。
4. 生产者将多条消息一起进行压缩发送，broker收到后保持压缩状态存储，消费者获取到压缩的消息，在处理时才解压，减少了网络传输消耗和传输次数。



# 参考

- [《深入理解Kafka：核心设计与实践原理》](https://book.douban.com/subject/30437872/)
- [《Kafka权威指南》](https://book.douban.com/subject/27665114/)

