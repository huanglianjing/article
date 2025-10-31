# 1. 简介

github仓库地址：https://github.com/IBM/sarama

文档地址：https://pkg.go.dev/github.com/IBM/sarama

sarama 是 Go 语言的 Kafka 客户端。

# 2. 使用

## 2.1 安装

 使用 go get 将 sarama 包下载到 GOPATH 指定的目录下。

```bash
go get github.com/IBM/sarama
```

## 2.2 生产消息

先设置配置，然后连接至 Kafka 服务器，然后进行消息发送，最后在程序退出时调用方法关闭连接。

```go
	// 配置
	config := sarama.NewConfig()
	config.Producer.Retry.Max = 3                    // 最大尝试发送次数
	config.Producer.RequiredAcks = sarama.WaitForAll // 发送完数据需要leader和follow都确认
	config.Producer.Return.Successes = true          // 成功交付的消息将在success channel返回

	// 连接 Kafka 服务器
	producer, err := sarama.NewSyncProducer([]string{"127.0.0.1:9092"}, config)
	if err != nil {
		panic(err)
	}
	defer producer.Close()

	// 发送一条消息
	msg := &sarama.ProducerMessage{
		Topic: "topic",
		Value: sarama.ByteEncoder("msg"),
	}
	if _, _, err := producer.SendMessage(msg); err != nil {
		fmt.Printf("send error: %#v\n", err)
	}

	// 发送多条消息
	msgs := make([]*sarama.ProducerMessage, 0)
	for _, str := range []string{"msg1", "msg2", "msg3"} {
		msg := &sarama.ProducerMessage{
			Topic: "topic",
			Value: sarama.ByteEncoder(str),
		}
		msgs = append(msgs, msg)
	}
	if err := producer.SendMessages(msgs); err != nil {
		fmt.Printf("send error: %#v\n", err)
	}
```

## 2.3 消费消息

首先需要定义一个结构体，实现接口 sarama.ConsumerGroupHandler。

```go
// sarama.ConsumerGroupHandler
type ConsumerGroupHandler interface {
	// Setup 消费者启动前的操作
	Setup(ConsumerGroupSession) error

	// Cleanup 消费者关闭时的操作
	Cleanup(ConsumerGroupSession) error

	// ConsumeClaim 消费消息时触发
	ConsumeClaim(ConsumerGroupSession, ConsumerGroupClaim) error
}
```

我们定义相应的结构体实现该接口。

```go
// ConsumerGroupHandler 消费者组处理器
type ConsumerGroupHandler struct {
	// 也可以在结构体中包含函数具柄成员，在初始化时定义传入，在 ConsumeClaim 方法调用
}

// Setup 消费者启动前的操作
func (c *ConsumerGroupHandler) Setup(_ sarama.ConsumerGroupSession) error {
	return nil
}

// Cleanup 消费者关闭时的操作
func (c *ConsumerGroupHandler) Cleanup(_ sarama.ConsumerGroupSession) error {
	return nil
}

// ConsumeClaim 消费消息时触发
func (c *ConsumerGroupHandler) ConsumeClaim(sess sarama.ConsumerGroupSession, claim sarama.ConsumerGroupClaim) error {
	for msg := range claim.Messages() {
		fmt.Printf("topic=%s, partition=%d, offset=%d\n", msg.Topic, msg.Partition, msg.Offset)
		fmt.Printf("consume msg: %s\n", msg.Value)
		// 处理消息具体逻辑
	}
	return nil
}
```

先设置配置，然后通过消费者组连接至 Kafka 服务器，然后进行消息消费，最后在程序退出时调用方法关闭连接。

可以使用优雅退出的方式，监听相应信号，在需要退出程序时主动关闭连接。

```go
	// 配置
	config := sarama.NewConfig()
	config.Producer.Return.Successes = true               // 成功发送的消息将写到 Successes 通道
	config.Consumer.Return.Errors = true                  // 消费时错误信息将写到 Errors 通道
	config.Consumer.Fetch.Default = 3 * 1024 * 1024       // 默认请求的字节数
	config.Consumer.Offsets.Initial = sarama.OffsetNewest // 从最新的 offset 读取，如果设置为 OffsetOldest 则从最旧的 offset 读取
	config.Consumer.Offsets.AutoCommit.Enable = true      // 将已消费的 offset 发送给 broker，默认为 true

	// 连接 Kafka 服务器，可以传入多个 broker，用逗号连接
	consumer, err := sarama.NewConsumerGroup([]string{"127.0.0.1:9092"}, "consumer-group", config)
	if err != nil {
		panic(err)
	}
	defer consumer.Close()

	// 消费消息
	for {
		ctx := context.Background()
		err := consumer.Consume(ctx, []string{"topic"}, &ConsumerGroupHandler{}) // 传入定义好的 ConsumerGroupHandler 结构体
		if err != nil {
			fmt.Printf("consume error: %#v\n", err)
		}
	}
```

# 3. 参考

* [Go 使用 sarama 包操作 kafka - oldme](https://oldme.net/article/28)

