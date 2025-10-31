# 1. 简介

Go 的 sync.RWMutex 是一个读写互斥锁，它允许任意数量的读锁，或者一个写锁。

对于很多场景来说，读操作次数远多于写操作，且读操作允许并发执行，而写操作则应当同时只有一个，在这种使用场景下，我们可以使用读写互斥锁进行并发控制。一个协程拥有写锁时，其他获取读锁或写锁的协程需要阻塞等待，一个协程拥有读锁时，其他协程可以拥有读锁，但不可以拥有写锁。

RWMutex 基于写操作优先进行设计，如果有写操作请求上锁，则会等待已经申请了读锁的协程继续执行，同时对于新的读操作只有在所有的写操作完成之后才可以开始进行操作。这个设计保证了写操作能够及早获取写锁，但对于频繁写入的情况，可能会导致读操作长时间阻塞。

读锁和写锁用法如下，在上锁后应该使用 defer 语句立刻进行解锁。可以在任何正常退出的分支或 panic 造成的异常退出时，保证能够执行到对应的解锁操作，避免上锁后未执行对应的解锁而导致的死锁。

```go
var lock sync.RWMutex

func read() {
	lock.RLock()
	defer lock.RUnlock()
	// do sth
}

func write() {
	lock.Lock()
	defer lock.Unlock()
	// do sth
}
```

RWMutex 还支持使用常识上锁的方法，如果无法获取锁时会返回 false，而非阻塞等待。

```go
var lock sync.RWMutex

func tryRead() {
	if lock.TryRLock() {
		defer lock.RUnlock()
		// do sth
	}
}

func tryWrite() {
	if lock.TryLock() {
		defer lock.Unlock()
		// do sth
	}
}
```

# 2. 原理解析

RWMutex 结构体定义如下：

```go
type RWMutex struct {
	w           Mutex        // held if there are pending writers
	writerSem   uint32       // semaphore for writers to wait for completing readers
	readerSem   uint32       // semaphore for readers to wait for completing writers
	readerCount atomic.Int32 // number of pending readers
	readerWait  atomic.Int32 // number of departing readers
}
```

它是基于 sync.Mutex 的封装，所以其结构体中包含了一个 Mutex 成员变量，并且添加了另外四个变量来实现读写锁的粒度拆分。

* writerSem：写阻塞等待的信号量，最后一个读锁释放时会释放该信号量
* readerSem：读阻塞等待过的信号量，最后一个写锁释放时会释放该信号量
* readerCount：获取到读锁的协程数量
* readerWait：上写锁时等待获取读锁的协程数量

```go
const rwmutexMaxReaders = 1 << 30
```

常量 rwmutexMaxReaders 表示支持并发获取读锁的协程数量的最大值。它同时是一个足够大的值，可以用于将记录获取读锁的 reader 的数量减去它，变为负数后，即表明有写锁在尝试上锁，又仍然能记录获取读锁数量信息。

使用变量 readerCount 和常量 rwmutexMaxReaders，将获取读锁的 reader 数量和写锁成功上锁两个状态的更新，融合进了一个变量里，从而可以原子化地进行判断和更新。

## 2.1 Lock

Lock 方法用于上写锁。

```go
func (rw *RWMutex) Lock() {
	// 加互斥锁，和其他写锁进行互斥阻塞
	rw.w.Lock()

	// 将 readerCount 减去大数 rwmutexMaxReaders，变为负数表示有 writter 正在请求写锁，同时还能继续记录 reader 数量
	// 将 readerCount 实际的值赋值给 r
	r := rw.readerCount.Add(-rwmutexMaxReaders) + rwmutexMaxReaders

	// r 不为 0 时表示有 reader 持有锁，readerWait 加上持有读锁的 reader 数量
	// 如果有 reader 持有读锁，当前 writter 阻塞等待，否则加锁成功直接返回
	if r != 0 && rw.readerWait.Add(r) != 0 {
		runtime_SemacquireRWMutex(&rw.writerSem, false, 0)
	}
}
```

## 2.2 Unlock

Unlock 方法用于释放写锁。

```go
func (rw *RWMutex) Unlock() {
	// 将 readerCount 加上大数 rwmutexMaxReaders，变为正数表示当前 writter 不再持有写锁，或供另一个写锁进行减操作
	r := rw.readerCount.Add(rwmutexMaxReaders)
	if r >= rwmutexMaxReaders { // 未上写锁时释放写锁，触发异常
		fatal("sync: Unlock of unlocked RWMutex")
	}

	// 释放信号唤醒所有阻塞的 reader
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false, 0)
	}

	// 释放互斥锁，允许其他 writter 进行上锁操作
	rw.w.Unlock()
}
```

## 2.3 RLock

RLock 方法用于上读锁。

```go
func (rw *RWMutex) RLock() {
	// readerCount 加一，如果小于 0，表示有一个 writer 正在申请或持有写锁
	if rw.readerCount.Add(1) < 0 {
		// 休眠阻塞，等待唤醒
		runtime_SemacquireRWMutexR(&rw.readerSem, false, 0)
	}
}
```

## 2.4 RUnlock

RUnlock 方法用于释放读锁。

```go
func (rw *RWMutex) RUnlock() {
	// readerCount 减一，如果小于 0，表示有一个 writer 正在申请或持有写锁
	if r := rw.readerCount.Add(-1); r < 0 {
		rw.rUnlockSlow(r)
	}
}

func (rw *RWMutex) rUnlockSlow(r int32) {
	if r+1 == 0 || r+1 == -rwmutexMaxReaders { // r 等于 -1，表示未上读锁时释放读锁，触发异常
		fatal("sync: RUnlock of unlocked RWMutex")
	}

	// readerWait 减一，如果变为 0，表示最后一个写锁释放
	if rw.readerWait.Add(-1) == 0 {
		// 唤醒等待的 writer 来获取锁
		runtime_Semrelease(&rw.writerSem, false, 1)
	}
}
```

## 2.5 TryLock

TryLock 方法用于尝试上写锁，如果上锁失败，则不阻塞，直接返回 false。

```go
func (rw *RWMutex) TryLock() bool {
	if !rw.w.TryLock() { // 尝试上锁失败时返回 false
		return false
	}

	// 持有读锁 reader 数量不为 0 时，将之前的互斥锁解锁，返回 false
	if !rw.readerCount.CompareAndSwap(0, -rwmutexMaxReaders) {
		rw.w.Unlock()
		return false
	}

	// 成功上写锁
	return true
}
```

## 2.6 TryRLock

TryRLock 方法用于尝试上读锁，如果上锁失败，则不阻塞，直接返回 false。

```go
func (rw *RWMutex) TryRLock() bool {
	for {
		c := rw.readerCount.Load()
		if c < 0 { // 如果 readerCount < 0，表示有写锁正在上锁，返回
			return false
		}
		// readerCount 没有变化时加 1，成功表示成功上锁返回，否则继续循环等待
		if rw.readerCount.CompareAndSwap(c, c+1) {
			return true
		}
	}
}
```

# 3. 参考

* [读写锁 - sync.RWMutex | 深入Go语言之旅](https://go.cyub.vip/concurrency/sync-rwmutex/)

