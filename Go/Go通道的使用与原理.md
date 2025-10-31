# 1. 简介

channel 称为通道/管道，是 Go 中可以在协程之间发送和接收消息的通信机制，就像一个消息队列，每一个通道会指定一个固定的数据类型，这个通道只能收发这个类型的对象。

与其他语言通过共享内存来仔线程之间实现通信不同，Go 提倡“不要通过共享内存的方式进行通信，而是通过 Channel 通信的方式共享内存”。

## 1.1 类型

一个 int 类型的通道，它的类型是 chan int，同种类型的通道可以使用 == 进行比较。通道的零值是 nil，对 nil 通道发送和接收将永远阻塞。

通道是使用 make 函数创建的数据结构的引用，当复制或者作为参数传递给函数时，复制的是引用。

## 1.2 初始化

```go
// 声明变量，值为nil
var ch chan int

// make函数
ch = make(chan int)      // 无缓冲通道
ch := make(chan int， 0) // 无缓冲通道
ch := make(chan int， 3) // 容量为3的缓冲通道
```

## 1.3 操作

通道有两个主要操作：发送（send）和接收（receive）。

```go
ch <- x  // 发送

x := <-ch     // 接收，赋值变量
x, ok := <-ch // 接收，赋值变量且判断是否成功读取
<-ch          // 接收，丢弃结果
```

通道还有一个操作：关闭（close）。关闭后发送操作将导致 panic，而接收操作将获取缓冲区的值，直至通道为空，后续再接收将获取到通道元素类型对应的零值。

```go
close(ch) // 关闭通道
```

通道关闭可以通过 select 语句接收到。

```go
done := make(chan struct{})
close(done)
select {
case <-done:
	fmt.Println("done")
}
```

通道默认可以读和写，而在传递到函数参数中时，可以限制通道在函数中是否可以读或写数据。

```go
// 通道可读写
func f(ch chan int) {
}

// 通道只能读
func f(ch <-chan int) {
}

// 通道只能写
func f(ch chan<- int) {
}
```

## 1.4 缓冲

通道创建时可以指定缓冲区容量，在其内会维护一个元素队列，队列的最大长度是通道的容量。发送操作向队列尾部插入元素，若通道满了则阻塞，接收操作从通道头部移除一个元素，若通道空了则阻塞。

缓冲区容量为0是无缓冲通道，也称为同步通道。发送操作会阻塞，直至另一个协程对通道执行接收操作，接收操作也会阻塞，直至另一个协程对通道发送一个值。

无缓冲通道提供强同步保障，发送和接收同步，而缓冲通道将发送和接收解耦。当发送快于接收和接收快于发送，缓冲区保持满或者空，缓冲通道都是没有价值的。

获取通道缓冲区容量和当前元素个数：

```go
cap(ch) // 获取通道容量
len(ch) // 获取通道当前元素个数
```

读取通道阻塞的条件有：

* 通道无缓冲区
* 通道的缓冲区无数据
* 通道为nil

写入通道阻塞的条件有：

* 通道无缓冲区
* 通道的缓冲区已满
* 通道为nil

不同操作对于不同状态的通道，将会遇到的情况如下表所示：

| 操作       | 通道为nil | 已关闭通道                                       | 未关闭、非空通道 |
| ---------- | --------- | ------------------------------------------------ | ---------------- |
| 从通道读取 | 阻塞      | 获取一个缓冲区的值，如果取完则返回通道类型的零值 | 正常读取或阻塞   |
| 写入通道   | 阻塞      | panic                                            | 正常写入或阻塞   |
| 关闭通道   | panic     | panic                                            | 成功关闭         |

## 1.5 无阻塞读写

无论是非缓冲还是带有缓冲区的通道，在读写操作到达通道空或满时，都会阻塞住，直到有另一个协程写或者读。

可以通过 select 实现非阻塞的通道读写操作。

非阻塞读：

```go
func TryRead(ch chan string) (string, bool) {
	select {
	case str := <-ch:
		return str, true
	default:
		return "", false
	}
}
```

非阻塞写：

```go
func TryWrite(ch chan string, str string) bool {
	select {
	case ch <- str:
		return true
	default:
		return false
	}
}
```

# 2. 实现原理

## 2.1 数据结构

通道的数据结构定义在 go 源码的 src/runtime/chan.go 中：

```go
type hchan struct {
	qcount   uint           // 队列当前元素个数
	dataqsiz uint           // 环形队列长度
	buf      unsafe.Pointer // 环形队列指针
	elemsize uint16         // 每个元素大小
	closed   uint32         // 标识关闭状态
	elemtype *_type         // 元素类型
	sendx    uint           // 写入时的队列下标
	recvx    uint           // 读取时的队列下标
	recvq    waitq          // 等待从通道读取的协程队列
	sendq    waitq          // 等待向通道写入的协程队列
	lock     mutex          // 互斥锁
}
```

可以看到 go 使用一个环形队列来实现通道，通过两个下标来标记读写在环形队列中的位置，以充分利用内存。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_channel_struct.png)

等待读取和写入的协程队列结构体为 waitq，它是一个双向链表，由指向链表首个元素和最后一个元素的指针组成，而这里元素结构体 sudog 代表着在等待队列中的一个 goroutine。实际上的一个 G 可以出现在许多等待队列上，因此有可能不同的 sudog 实际是同一个 G。

```go
type waitq struct {
	first *sudog
	last  *sudog
}
```

## 2.2 创建通道

编译器将 make 创建通道的表达式进行转换，进而调用 runtime.makechan 函数。

创建时根据通道是否有缓冲区、元素类型是否为指针，分配对应的内存。

```go
func makechan(t *chantype, size int) *hchan {
	elem := t.Elem
	mem, overflow := math.MulUintptr(elem.Size_, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}
	var c *hchan
	switch {
	case mem == 0: // 无缓冲通道，分配通道默认内存
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		c.buf = c.raceaddr()
	case elem.PtrBytes == 0: // 元素类型不是指针，分配通道加缓冲区以及底层数组的内存
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default: // 元素类型是指针，分配通道加缓冲区的内存
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}
	c.elemsize = uint16(elem.Size_)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)
	return c
}
```

## 2.3 发送数据

编译器将对通道发送数据的语句转为调用 runtime.chansend 函数。

```go
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	lock(&c.lock) // 对当前通道加锁，防止被其它协程并发修改数据
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

	// 如果存在等待写的协程，从队列取出第一个协程，调用send函数
	if sg := c.recvq.dequeue(); sg != nil {
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

	// 若缓冲区未满，将发送的数据写入缓冲区
	if c.qcount < c.dataqsiz {
		qp := chanbuf(c, c.sendx)
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

	// 无缓冲区或缓冲区已满，阻塞等待
	if !block {
		unlock(&c.lock)
		return false
	}
	gp := getg() // 获取发送数据的协程
	mysg := acquireSudog()
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg) // 将这次发送阻塞信息加到等待写的协程队列
	gp.parkingOnChan.Store(true)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceBlockChanSend, 2)
	KeepAlive(ep)

	gp.waiting = nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true
}
```

send 函数将数据写入等待接收数据的协程，并将该协程标记为可运行状态。

```go
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if sg.elem != nil {
		sendDirect(c.elemtype, sg, ep) // 将数据写入
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	goready(gp, skip+1) // 将协程标记为可运行状态
}

func sendDirect(t *_type, sg *sudog, src unsafe.Pointer) {
	memmove(dst, src, t.Size_)
}
```

## 2.4 接收数据

编译器将从通道接收数据的语句转为调用 runtime.chanrecv 函数。

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	lock(&c.lock) // 对当前通道加锁，防止被其它协程并发修改数据
	if c.closed != 0 {
		if c.qcount == 0 {
			unlock(&c.lock)
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	} else {
		// 如果存在等待读的协程，从队列取出第一个协程，调用recv函数
		if sg := c.sendq.dequeue(); sg != nil {
			recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
			return true, true
		}
	}

	// 若缓冲区已有数据，从缓冲区将数据取出
	if c.qcount > 0 {
		qp := chanbuf(c, c.recvx)
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

	// 无可以获取的数据，阻塞等待
	if !block {
		unlock(&c.lock)
		return false, false
	}
	gp := getg() // 获取接收数据的协程
	mysg := acquireSudog()
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg) // 将这次接收阻塞信息加到等待读的协程队列
	gp.parkingOnChan.Store(true)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceBlockChanRecv, 2)

	gp.waiting = nil
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, success
}
```

recv 函数将数据写入等待接收数据的协程，并将该协程标记为可运行状态。

```go
func recv(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	if c.dataqsiz == 0 {
		if ep != nil {
			// 无缓冲区，直接将数据写入
			recvDirect(c.elemtype, sg, ep)
		}
	} else {
		qp := chanbuf(c, c.recvx)
		if ep != nil {
        	// 将队列的数据复制到接收方的内存地址
			typedmemmove(c.elemtype, ep, qp)
		}
        // 将发送队列头数据复制到缓冲区，释放一个阻塞的发送方
		typedmemmove(c.elemtype, qp, sg.elem)
		c.recvx++
		c.sendx = c.recvx // c.sendx = (c.sendx+1) % c.dataqsiz
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	goready(gp, skip+1) // 将处理器分配至发送数据的协程
}

func recvDirect(t *_type, sg *sudog, dst unsafe.Pointer) {
	memmove(dst, src, t.Size_)
}
```

## 2.5 关闭通道

编译器将关闭通道转为调用 runtime.closechan 函数。

```go
func closechan(c *hchan) {
	if c == nil {
		panic(plainError("close of nil channel"))
	}

	lock(&c.lock) // 对当前通道加锁，防止被其它协程并发修改数据
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	c.closed = 1

	var glist gList

	// 释放从通道读取的协程队列
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		glist.push(gp)
	}

	// 释放向通道写入的协程队列
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		glist.push(gp)
	}
	unlock(&c.lock)

	for !glist.empty() {
		goready(gp, 3) // 触发调度
	}
}
```

# 3. 参考

* [《Go专家编程》](https://book.douban.com/subject/35144587/)
* [Go 语言 Channel 实现原理精要 | Go 语言设计与实现](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-channel/)

