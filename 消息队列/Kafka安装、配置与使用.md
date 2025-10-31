# 1. 安装

## 1.1 Linux下安装

Kafka运行环境需要先安装好Java环境。

进入官网http://kafka.apache.org/downloads，选择相应的版本的Kafka链接并下载：

```bash
$ wget https://downloads.apache.org/kafka/2.8.0/kafka_2.13-2.8.0.tgz
```

解压安装包

```bash
$ tar zxf kafka_2.13-2.8.0.tgz -C /usr/local/
$ ln -s /usr/local/kafka_2.13-2.8.0/ /usr/local/kafka
$ cd /usr/local/kafka
```

启动ZooKeeper

```bash
$ ./bin/zookeeper-server-start.sh config/zookeeper.properties

# 后台运行
$ nohup ./bin/zookeeper-server-start.sh config/zookeeper.properties >> zookeeper.log 2>&1 &
```

启动Kafka

```bash
$ ./bin/kafka-server-start.sh config/server.properties

# 后台运行
$ nohup ./bin/kafka-server-start.sh config/server.properties >> kafka.log 2>&1 &
```

## 1.2 MacOS下通过brew安装

在MacOS下，还可以通过brew来安装和运行Kafka，并且可以很方便地启动。

安装：

```bash
$ brew install zookeeper
$ brew install kafka
```

启动服务：

```bash
$ brew services start zookeeper
$ brew services start kafka
```

如果只是临时启动的话：

```bash
$ zkServer start
$ kafka-server-start /usr/local/etc/kafka/server.properties
```

这里需要注意的是，由于Kafka是依赖ZooKeeper来运作的，所以需要先启动ZooKeeper再启动Kafka，关闭的时候也注意要先关闭Kafka再关闭ZooKeeper。

而Kafka对应的一系列脚本工具，可以直接用命令的方式进行调用，如以下的一些管理常用命令：

```
kafka-topics
kafka-console-producer
kafka-console-consumer
...
```



# 2. 配置系统服务单元

这一步是可选的，配置了之后通过systemctl命令启动和停止，也可以直接执行脚本来启动停止。

## 2.1 Zookeeper

创建系统服务单元

```bash
$ cd /etc/systemd/system
$ vi zookeeper.service
```

贴上以下内容

```properties
[Unit]
Description=Apache Zookeeper server
Documentation=http://zookeeper.apache.org
Requires=network.target remote-fs.target
After=network.target remote-fs.target

[Service]
Type=simple
ExecStart=/usr/local/kafka/bin/zookeeper-server-start.sh /usr/local/kafka/config/zookeeper.properties
ExecStop=/usr/local/kafka/bin/zookeeper-server-stop.sh
Restart=on-abnormal
User=root
Group=root

[Install]
WantedBy=multi-user.target
```

操作命令

```bash
# 启动ZooKeeper
$ systemctl start zookeeper

# 查看ZooKeeper状态
$ systemctl status zookeeper

# 关闭ZooKeeper
$ systemctl stop zookeeper
```

## 2.2 Kafka

创建系统服务单元

```bash
$ cd /etc/systemd/system
$ vi kafka.service
```

贴上以下内容

```properties
[Unit]
Description=Apache Kafka Server
Documentation=http://kafka.apache.org/documentation.html
Requires=zookeeper.service

[Service]
Type=simple
ExecStart=/usr/local/kafka/bin/kafka-server-start.sh /usr/local/kafka/config/server.properties
ExecStop=/usr/local/kafka/bin/kafka-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target
```

操作命令

```bash
# 启动Kafka
$ systemctl start kafka

# 查看Kafka状态
$ systemctl status kafka

# 关闭Kafka
$ systemctl stop kafka
```



# 3. 目录结构

下面进入Kafka的目录，也就是`/usr/local/kafka`，看一下目录的结构。

```
|-- bin                           // Kafka和ZooKeeper的脚本工具
|   |-- kafka-console-consumer.sh
|   |-- kafka-console-producer.sh
|   |-- kafka-server-start.sh
|   |-- kafka-server-stop.sh
|   |-- kafka-topics.sh
|   |-- windows                   // windows下的bat脚本
|   |-- zookeeper-server-start.sh
|   |-- zookeeper-server-stop.sh
|   `-- ...
|-- config                        // Kafka和ZooKeeper的配置文件
|   |-- kraft                     // Kafka2.8开始移除ZooKeeper依赖的新启动配置，本文暂不介绍
|   |-- server.properties
|   |-- zookeeper.properties
|   `-- ...
|-- libs                          // 一些依赖的jar包
|-- LICENSE
|-- licenses
|-- logs                          // 日志
|-- NOTICE
`-- site-docs                     // 文档
```



# 4. 脚本工具

Kafka提供了很多脚本工具，可以用来进行主题创建和查看、生产者、消费者等操作。

以下脚本执行需要先进入Kafka目录进行操作，脚本工具都在bin目录下。

#### kafka-server-start.sh

Kafka启动脚本。

```bash
# 启动Kafka
$ ./bin/kafka-server-start.sh config/server.properties
```

#### kafka-server-stop.sh

Kafka关闭脚本。通过发送SIGTERM信号给Kafka进程实现优雅关闭。

```bash
# 关闭Kafka
$ ./bin/kafka-server-stop.sh
```

#### kafka-topics.sh

与主题相关的脚本，用于查看主题、创建主题。

```bash
# 查看已创建的主题
$ ./bin/kafka-topics.sh --zookeeper localhost:2181 --list

# 创建主题
$ ./bin/kafka-topics.sh --zookeeper localhost:2181 --create --topic test --replication-factor 1 --partitions 1

# 增加主题的分区数
$ ./bin/kafka-topics.sh --zookeeper localhost:2181 --alter --topic test --partitions 3

# 删除主题
$ ./bin/kafka-topics.sh --zookeeper localhost:2181 --delete -topic test

# 查看主题分区
$ ./bin/kafka-topics.sh --zookeeper localhost:2181 --describe -topic test
```

#### kafka-console-producer.sh

生产者脚本。

```bash
# 通过生产者发送消息，在终端输入然后回车发送消息
$ ./bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
```

#### kafka-console-consumer.sh

消费者脚本。

```bash
# 通过消费者接收消息
$ ./bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test

# 通过消费者接收消息，从头开始
$ ./bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning

# 使用消费组
$ ./bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --group consumer-group1
```

#### kafka-configs.sh

配置管理脚本。

#### kafka-consumer-groups.sh

消费组管理脚本。

```bash
# 列出当前集群所有消费组
$ ./bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# 展示消费组的详细信息
$ ./bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group groupname
$ ./bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group groupname --state # 状态
$ ./bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group groupname --members # 消费者成员信息
$ ./bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group groupname --members --verbose # 消费者分配情况

# 删除消费组
$ ./bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --delete --group groupname

# 将消费组所有分区的消费位移置0
$ ./bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group groupname --all-topics --reset-offsets --to-earliest --execute

# 将消费组某个分区的消费位移置为分区末尾
$ ./bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --group groupname --topic test --reset-offsets --to-latest --execute
```

#### kafka-delete-records.sh

删除消息脚本。

以下是一个示例delete.json，该json指明删除主题topic-monitor下，分区0中偏移量为10、分区1中偏移量为11、分区2中偏移量为12的消息：

```json
{
  "partitions": [
    {
      "topic": "topic-monitor",
      "partition": 0,
      "offset": 10
    },
    {
      "topic": "topic-monitor",
      "partition": 1,
      "offset": 11
    },
    {
      "topic": "topic-monitor",
      "partition": 2,
      "offset": 12
    }
  ],
  "version": 1
}
```

```bash
# 根据json文件规则删除消息
$ ./bin/kafka-delete-records.sh --bootstrap-server localhost:9092 --offset-json-file delete.json
```

#### kafka-configs.sh

配置管理脚本。

#### kafka-preferred-replica-election.sh

分区leader副本选举脚本。

```bash
# 分区leader副本选举
$ ./bin/kafka-preferred-replica-election.sh --zookeeper localhost:2181
```

#### kafka-reassign-partitions.sh

分区脚本。

```bash
# 分区重分配
$ ./bin/kafka-reassign-partitions.sh --zookeeper localhost:2181 --generate --topic-to-move-json-file reassign.json

# 修改副本因子
$ ./bin/kafka-reassign-partitions.sh --zookeeper localhost:2181 --execute --reassignment-json-file add.json
```

#### 性能测试工具

kafka-producer-perf-test.sh用于生产者性能测试。

kafka-consumer-perf-test.sh用于消费者性能测试。



# 5. 配置

## 5.1 server.properties

Kafka服务端配置文件，启动Kafka时用到。

```properties
# 集群中broker的唯一标识，默认值为0，各个broker不同，设置为0开始的枚举值
broker.id=0

# 端口，默认为9092
port=9092

# broker连接的ZooKeeper集群的地址和端口，多个节点用逗号分隔
zookeeper.connect=localhost:2181

# 监听客户端连接的地址列表
# protocol 协议，支持的协议有PLAINTEXT、SSL、SASL_SSL等
# host     主机名，不指定表示默认网卡，0.0.0.0表示所有网卡
# port     端口，默认值为null
listeners=protocol1://host1:port1,protocol2://host2:port2,protocol3://host3:port3
listeners=PLAINTEXT://:9092

# 日志文件目录
# log.dirs存放多个目录，以逗号分隔，优先级更高
# log.dir存放单个目录
log.dirs=/tmp/kafka-logs

# 单个消息的最大值
message.max.bytes=1000000

# 创建新主题默认的分区数
num.partitions=1

# 数据可以被保留多久，默认为一周
log.retention.hours=168

# 数据可以被保留多久
log.retention.minutes = 100
log.retention.ms = 100

# 根据保留消息字节数判断消息是否过期，默认为1GB
log.retention.bytes=1073741824

# 日志片段大小上限，默认为1GB，达到上限时会打开新的日志片段
log.segment.bytes=1073741824

# 多长时间后日志片段会被关闭
log.segment.ms
```



# 参考

- [《深入理解Kafka：核心设计与实践原理》](https://book.douban.com/subject/30437872/)
- [《Kafka权威指南》](https://book.douban.com/subject/27665114/)

