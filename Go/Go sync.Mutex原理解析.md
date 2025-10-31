# 1. 简介

Go 的 sync.Mutex 作为互斥锁，在程序并发运行时，用于对某一段代码或某一种资源访问进行控制，防止同时执行。

如 Go 原生的 map 存在多个协程并发地进行修改，会引发 panic，需要使用互斥锁进行保护。又如代码中存在对某个变量进行先读取后更新的操作，当不进行互斥访问时，可能存在两个协程先后读取再先后更新，造成数据的更新不稳定符合预期，需要通过互斥锁将读取更新的逻辑隔离开来。

Mutex 的使用方法如下，定一个 sync.Mutex 对象，然后调用 Lock 方法加锁，执行完一段逻辑代码后，调用 Unlock 方法解锁。当某个协程加锁后，别的协程再调用 Lock 方法将会被阻塞，直至改变量调用 Unlock 方法解锁后，才能获取锁。

```go
var lock sync.Mutex

func f() {
	lock.Lock()
	// do sth
	lock.Unlock()
}
```

我们中间一段互斥访问的逻辑代码可能会存在函数返回，或引发 panic 等情况，以上写法有可能会执行不到 Unlock，或在某些返回分支条件中忘记加上 Unlock 方法调用，造成死锁。更好的写法是在调用 Lock 方法之后，立刻就 defer 调用 Unlock 方法，无论函数返回或是引发 panic，Unlock 方法都能得到调用。

```go
var lock sync.Mutex

func f() {
	lock.Lock()
	defer lock.Unlock()
	// do sth
}
```

Mutex 还存在一个 TryLock 方法，当无法获取锁时，将会返回 false，而不是一直阻塞。

```go
var lock sync.Mutex

func f() {
	if lock.TryLock() {
		defer lock.Unlock()
		// do sth
	}
}
```

# 2. 原理解析

## 2.1 Mutex

Mutex 结构体定义如下：state 表示当前互斥锁的状态，sema 是一个信号量变量，用来控制等待协程的阻塞休眠和唤醒。

```go
type Mutex struct {
	state int32
	sema  uint32
}
```

state 字段有 32 位长度，可以分为四部分。最低的三位分别表示 mutexLocked、mutexWoken、mutexStarving 三个状态，mutexLocked 表示 Mutex 是否已上锁， mutexWoken 表示从普通模式被唤醒，mutexStarving 表示是否属于饥饿模式。后面 29 位用于记录等待互斥锁协程数量的计数器 waiter，最多可以表示 2^29 个协程正在等待该互斥锁。

* mutexLocked：互斥锁是否已上锁，为 1 表示已被锁定
* mutexWoken：是否有等待的协程已被唤醒，防止多个等待的协程被同时唤醒来竞争锁
* mutexStarving：互斥锁是否处于饥饿模式
* waiter：有多少协程在等待获取互斥锁

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/mutex_state.png)

```go
const (
    mutexLocked = 1 << iota // mutex is locked
    mutexWoken
    mutexStarving
    mutexWaiterShift = iota // 3，用于 state 移位获取等待互斥锁的协程数量
)
```

Mutex 有两种模式：普通模式和饥饿模式。

在普通模式下，所有争取锁的协程都加入到先进先出的等待队列中，被唤醒的协程将会和新来的协程进行竞争，但新的协程已经在 CPU 中于是有优势获取锁，而被唤醒的协程往往可能无法获取到锁，而重新插入队列前面。如果一个等待锁的协程超过 1ms 无法获取锁时，该锁转为饥饿模式。

在饥饿模式下，当一个协程解锁后，将会将锁的拥有权直接传递给等待队列开头的协程。新来的协程即使在未上锁时也不会尝试去获取锁，也不会进行自旋，而只是将他们排到等待队列的队尾。当某个协程获取锁后，如果它是等待队列的最后一个，或着它等待时间小于 1ms，将锁变回普通模式。

普通模式有着更好的获取锁的性能，而饥饿模式则更注重公平，避免一些协程一直无法获取锁导致的饿死情况。

```go
const (
	starvationThresholdNs = 1e6 // 1ms，普通模式和饥饿模式转换的阈值
)
```

## 2.2 Lock

Lock 方法用于给 Mutex 加锁，如果已经锁上了，将会一直阻塞等待。

```go
func (m *Mutex) Lock() {
	// 快速通过 CAS 原子操作获得锁
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		return
	}
	// 无法获取锁，通过自旋或等待来获取锁
	m.lockSlow()
}

func (m *Mutex) lockSlow() {
	var waitStartTime int64 // 等待时间
	starving := false // 饥饿标记
	awoke := false // 唤醒标记
	iter := 0 // 自旋次数
	old := m.state // 锁的当前状态

	// 循环直至成功上锁
	for {
		// 尝试自旋：已上锁且不是饥饿模式，判断可以自旋时
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
			// 不是唤醒状态，没有被其他人唤醒，等待的协程数不为0
			// 尝试将 state 的 mutexWoken 位置 1 成功时
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true // 设置唤醒标记
			}
            // 进行自旋并更新记录的状态
			runtime_doSpin()
			iter++
			old = m.state
			continue
		}

		new := old
		// 如果不是饥饿模式，将 state 的 mutexLocked 位置 1，表示加锁
		if old&mutexStarving == 0 {
			new |= mutexLocked
		}
		// 如果已上锁或是饥饿模式，将等待计数器加一
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift
		}
		// 如果是饥饿模式且已上锁，将 state 的 mutexStarving 位置 1
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving // 当前协程的锁变为饥饿模式，当解锁时不主动获取锁
		}
		// 如果自旋时成功设置唤醒标记
		if awoke {
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken // 清除唤醒标志位
		}

		// 尝试通过 CAS 原子操设置新的状态
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
			// 原先没有上锁也不是饥饿模式，表示已经获取到锁，退出循环返回
			if old&(mutexLocked|mutexStarving) == 0 {
				break
			}
			// 如果等待时长大于 0，放到等待队列的开头
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 { // 第一次执行到这里，设置等待时间
				waitStartTime = runtime_nanotime()
			}
			// 阻塞等待获取 mutex，当前协程进行休眠
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)
			// 检查等待时间，超过阈值时设置为饥饿状态
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state
			// 如果已经是饥饿状态
			if old&mutexStarving != 0 {
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}
				// 加锁并将等待计数器减一
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				if !starving || old>>mutexWaiterShift == 1 {
					// 如果当前协程不是饥饿模式，就将锁转为正常模式
					delta -= mutexStarving
				}
				atomic.AddInt32(&m.state, delta)
				break
			}
			awoke = true
			iter = 0
		} else {
			// 设置状态失败，重置 old，继续循环
			old = m.state
		}
	}
}
```

Lock 方法通过自旋锁机制，在某些情况下允许协程进行自旋等待，当期间成功加锁时，可以免除协程调度的时间开销，提高性能。

还通过普通模式和饥饿模式的切换，兼顾性能与公平。普通模式下所有协程争抢加锁，这时候往往是正在 CPU 中调度执行的协程获取到锁，而这会造成一些协程迟迟无法获取锁的情况。切换饥饿模式又能保证公平性，避免了部分协程获取锁的时间变得不合理地长。而在最后一个等待协程获取锁或等待获取锁时间小于 1ms 时，又切换回普通模式，保证了上锁的性能。

上锁的流程图如下：

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/mutex_lock.jpg)

## 2.3 TryLock

TryLock 方法也是给 Mutex 加锁，但与 Lock 方法的区别是，Lock 方法获取不到锁时会阻塞等待，而 TryLock 方法会返回 false。

```go
func (m *Mutex) TryLock() bool {
	old := m.state
	// 判断已上锁或是饥饿模式，直接返回 false
	if old&(mutexLocked|mutexStarving) != 0 {
		return false
	}

	// 尝试将 mutexLocked 位置 1，成功返回 true，失败返回 false
	if !atomic.CompareAndSwapInt32(&m.state, old, old|mutexLocked) {
		return false
	}
	return true
}
```

TryLock 方法相比之下非常简单，通过 CAS 尝试加锁，随后将成功或失败结果直接返回，不需要考虑多个等待协程间的竞争关系。

## 2.4 Unlock

Unlock 方法用来给 Mutex 释放锁。

```go
func (m *Mutex) Unlock() {
	// 将 state 减去 mutexLocked，即减一
	new := atomic.AddInt32(&m.state, -mutexLocked)
	if new != 0 {
		// 如果 state 不等于 0，表示其他位有值，有额外工作需要处理
		m.unlockSlow(new)
	}
}

func (m *Mutex) unlockSlow(new int32) {
	// 对未上锁的 mutex 调用 Unlock 将导致 fatal error
	if (new+mutexLocked)&mutexLocked == 0 {
		fatal("sync: unlock of unlocked mutex")
	}

	if new&mutexStarving == 0 {
		// 正常模式
		old := new
		for {
			// 如果等待计数器为 0，或者已经有在处理的情况，直接返回
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			// 等待计数器减一，设置 mutexWoken 为 1
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			// 尝试更新 state，成功时返回，否则继续循环
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				// 唤醒等待队列中的 waiter 协程，返回
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else {
		// 饥饿模式
		// 直接唤醒等待队列中的 waiter 协程
		runtime_Semrelease(&m.sema, true, 1)
	}
}
```

Unlock 方法释放锁，并通过信号量来唤醒阻塞获取锁的协程。根据 Mutex 属于普通模式还是饥饿模式，区别在于唤醒等待队列的任意一个或者是队首的等待协程。

由于 Mutex 本身并不会记录持有锁的协程信息，所以 Unlock 方法不会对解锁操作判断协程是否已加锁，对于未加锁的 Mutex 进行解锁将会触发 fatal error。我们需要在代码逻辑中保证先加锁后解锁的顺序。

解锁的流程图如下：

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/mutex_unlock.jpg)

# 3. 参考

* [多图详解Go的互斥锁Mutex - luozhiyun`s Blog](https://www.luozhiyun.com/archives/413)
* [【Go基础篇】 彻底搞懂 Mutex 实现原理 - 掘金](https://juejin.cn/post/7166237022058151972)
* [深入理解 go Mutex | Dra-M]()

