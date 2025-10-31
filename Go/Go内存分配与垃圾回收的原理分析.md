# 1. 内存管理

每个进程都会在计算机中占用一定的内存，在其占用的内存空间中包含两个重要区域：栈区（Stack）和堆区（Heap）。函数调用的参数、返回值以及局部变量大都会被分配到栈上，这部分内存会由编译器进行管理；而用户程序手动申请的内存则主要被分配到堆区，对于C、C++等编程语言需要程序员手动申请和释放内存，而对Java、Go等编程语言有垃圾收集器对不再用到的内存进行回收。

内存管理一般包含三个不同的组件，分别是用户程序（Mutator）、内存分配器（Allocator）和垃圾收集器（Collector），当用户程序申请内存时，它会通过内存分配器申请新内存，而分配器会负责从堆中初始化相应的内存区域。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_memory_manage_3_part.png)

# 2. 栈内存分配

栈区的内存一般由编译器自动分配和释放，其中存储着函数的入参以及局部变量，这些参数会随着函数的创建而创建，函数的返回而消亡，一般不会在程序中长期存在。

## 2.1 寄存器

栈寄存器是 CPU 寄存器中的一种，它的主要作用是跟踪函数的调用栈，Go 语言的汇编代码包含 BP 和 SP 两个栈寄存器，它们分别存储了栈的基址指针和栈顶的地址。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_stack_registers.png)

因为历史原因，栈区内存都是从高地址向低地址扩展的，当应用程序申请或者释放栈内存时只需要修改 SP 寄存器的值，时间开销非常小。

## 2.2 线程栈

如果我们在 Linux 操作系统中执行 pthread_create 系统调用，进程会启动一个新的线程，操作系统会根据架构选择不同的默认栈大小，多数架构上默认栈大小都在 2 ~ 4 MB 左右，极少数架构会使用 32 MB 的栈。Go 语言在设计时认为执行上下文是轻量级的，所以它在用户态实现 Goroutine 作为执行上下文。

## 2.3 逃逸分析

在编译器优化中，逃逸分析是用来决定指针动态作用域的方法，Go 语言的编译器使用逃逸分析决定哪些变量应该在栈上分配，哪些变量应该在堆上分配，其中包括使用 `new`、`make` 和字面量等方法隐式分配的内存，Go 语言的逃逸分析遵循以下两个不变性：

1. 指向栈对象的指针不能存在于堆中；
2. 指向栈对象的指针不能在栈对象回收后存活；

逃逸分析是静态分析的一种，在编译器解析了 Go 语言源文件后，它可以获得整个程序的抽象语法树，编译器可以根据抽象语法树分析静态的数据流。遍历对象分配图并查找违反两条不变性的变量分配关系，如果堆上的变量指向了栈上的变量，那么该变量需要分配在堆上。

为了保证内存的绝对安全，编译器可能会将一些变量错误地分配到堆上，但是因为堆也会被垃圾收集器扫描，所以不会造成内存泄露以及悬挂指针等安全问题。

## 2.4 栈内存空间

Go 语言使用用户态线程 Goroutine 作为执行上下文，它的额外开销和默认栈大小都比线程小很多，Goroutine 的初始栈内存在最初的几个版本中多次修改，最开始为 4KB，为了减轻分段栈中的栈分裂对程序的性能影响从 4KB 到 8 KB，又在引入连续栈之后降低到 2KB。

**分段栈**

分段栈是 Go 语言在 v1.3 版本之前的实现，所有 Goroutine 在初始化时都会分配一块固定大小的内存空间，运行时会在全局的栈缓存链表中找到空闲的内存块并作为新 Goroutine 的栈空间返回。

当 Goroutine 调用的函数层级或者局部变量需要的越来越多时，运行时会创建一个新的栈空间，这些栈空间虽然不连续，但是当前 Goroutine 的多个栈空间会以链表的形式串联起来，运行时会通过指针找到连续的栈片段。

分段栈机制存在两个比较大的问题：

1. 如果当前 Goroutine 的栈几乎充满一个栈空间，那么任意函数调用都会触发栈扩容，当函数返回后又会触发栈的收缩。在这个边缘频繁调用和返回函数，栈的分配和释放会造成巨大的额外开销，这被称为热分裂问题（Hot split）；
2. 一旦 Goroutine 使用的内存越过了分段栈的扩缩容阈值，运行时会触发栈的扩容和缩容，带来额外的工作量；

**连续栈**

每当程序的栈空间不足时，初始化一片更大的栈空间并将原栈中的所有值都迁移到新栈中，新的局部变量或者函数调用就有充足的内存空间。连续栈可以解决分段栈中存在的两个问题。

在扩容的过程中，需要将指向旧栈对应变量的指针重新指向新栈，这增加了栈扩容时的额外开销。

# 3. 堆内存分配

当用户在程序中手动申请内存时，Go 通过内存分配器将会从堆区申请对应的内存，以供程序使用。

## 3.1 分配方法

编程语言的内存分配器一般包含两种分配方法，一种是线性分配器（Sequential Allocator，Bump Allocator），另一种是空闲链表分配器（Free-List Allocator），这两种分配方法有着不同的实现机制和特性。

### 3.1.1 线性分配器

线性分配器是一种高效的内存分配方法，但是有较大的局限性。当我们使用线性分配器时，只需要在内存中维护一个指向内存特定位置的指针，如果用户程序向分配器申请内存，分配器只需要检查剩余的空闲内存、返回分配的内存区域并修改指针在内存中的位置。

如下图，在申请了三块内存之后：

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_bump_allocator_1.png)

回收释放其中一块内存后：

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_bump_allocator_2.png)

虽然线性分配器实现为它带来了较快的执行速度以及较低的实现复杂度，但是线性分配器无法在内存被释放时重用内存。它需要与合适的垃圾回收算法配合使用，例如：标记压缩（Mark-Compact）、复制回收（Copying GC）和分代回收（Generational GC）等算法，它们可以通过拷贝的方式整理存活对象的碎片，将空闲内存定期合并。

对于C、C++等直接暴露指针地址的语言无法使用该分配器。

### 3.1.2 空闲链表分配器

空闲链表分配器会对空闲内存维护一个链表当用户程序申请内存时，空闲链表分配器会依次遍历空闲的内存块，找到足够大的内存，然后申请新的资源并修改链表。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_free_list_allocator.png)

分配内存时需要遍历链表，时间复杂度为O(n)，在链表中选择内存块有不同策略：

1. 首次适应（First-Fit）：从链表头开始遍历，选择第一个大小大于申请内存的内存块
2. 循环首次适应（Next-Fit）：从上次遍历的结束位置开始遍历，选择第一个大小大于申请内存的内存块
3. 最优适应（Best-Fit）：从链表头遍历整个链表，选择最合适的内存块
4. 隔离适应（Segregated-Fit）：将内存分割成多个链表，每个链表中的内存块大小相同，申请内存时先找到满足条件的链表，再从链表中选择合适的内存块

Go 使用的内存选择策略类似于隔离适应，隔离适应的分配策略减少了需要遍历的内存块数量，提高了内存分配的效率。

## 3.2 虚拟内存布局

Go 程序在启动时会虚拟化整片虚拟内存区域，这些内存并不是真正存在的物理内存，而是虚拟内存。包含 spans、bitmap、arena 三部分：

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_heap_1.png)

spans 区域存储了指向内存管理单元 runtime.mspan 的指针，每个内存单元会管理几页的内存空间，每页大小为 8KB。bitmap 用于标识 arena 区域中的那些地址保存了对象，每一位表示一个字节是否空闲。arena 区域是真正的堆区，运行时会将 8KB 看做一页，这些内存页中存储了所有在堆上初始化的对象。

对于任意一个地址，我们都可以根据 arena 的基地址计算该地址所在的页数并通过 spans 数组获得管理该片内存的管理单元 runtime.mspan。

以上是 Go 1.10 及之前的版本，堆区内存空间是连续的，在 1.11 后 arena 改为使用稀疏内存，解除了堆大小的 512GB 内存上限，但内存不连续也使得内存管理 变得更加复杂。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_heap_2.png)

## 3.3 分级分配

Go 语言的内存分配器借鉴了 TCMalloc（Thread-Caching Malloc，线程缓存分配）的设计思想，使用多级缓存将对象根据大小分类，并实施不同的分配策略。

分配器使用线程缓存（Thread Cache）、中心缓存（Central Cache）和页堆（Page Heap）三个组件分级管理内存。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_multi_level_cache.png)

线程缓存属于每一个独立的线程，因为不涉及多线程，所以也不需要使用互斥锁来保护内存，这能够减少锁竞争带来的性能损耗。当线程缓存不能满足需求时，运行时会使用中心缓存作为补充解决小对象的内存分配，中心缓存是多个线程共享的，访问时需要加锁。在遇到 32KB 以上的对象时，内存分配器会选择页堆直接分配大内存，当页堆内存不够时，则会通过系统调用向操作系统申请更多内存。

Go 程序启动时，将会初始化如下图所示的内存布局。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_memory_layout.png)

每个处理器会分配一个线程缓存 runtime.mcache，它会持有内存管理单元 runtime.mspan，用于处理微对象和小对象的分配。

每个类型的内存管理单元都会管理特定大小的对象，当内存管理单元中不存在空闲对象时，它们会从 runtime.mheap 持有的 134 个中心缓存 runtime.mcentral 中获取新的内存单元，中心缓存属于全局的堆结构体 runtime.mheap，它会从操作系统中申请内存。

### 3.3.1 内存管理单元

runtime.mspan 是 Go 语言内存管理的基本单元。每个 mspan 管理 npages 个大小为 8KB 的页，这里的页是操作系统内存页的整数倍。

```go
type mspan struct {
	next *mspan
	prev *mspan

	startAddr uintptr // 起始地址
	npages    uintptr // 页数
	freeindex uintptr // 扫描页中空闲对象的初始索引

	allocBits  *gcBits // 标记内存的占用情况
	gcmarkBits *gcBits // 标记内存的回收情况
	allocCache uint64  // allocBits 的补码，可以用于快速查找内存中未被使用的内存

	state       mSpanStateBox // 状态

	spanclass   spanClass // 跨度类
	...
}
```

作为节点，mspan 通过 next 和 prev 指针构成一个双向链表。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_mspan_linked_list.png)

当用户程序或者线程向 mspan 申请内存时，它会使用 allocCache 字段以对象为单位在管理的内存中快速查找待分配的空间。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_mspan_search.png)

当 mspan 管理的内存不足时，会以页为单位向堆申请内存。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_mspan_alloc.png)

mspan.spanclass 表示对象的跨度类，它决定了内存管理单元中存储的对象大小和个数。

Go 内存管理设计了 67 种跨度类，每一个跨度类都会存储特定大小的对象并且包含特定数量的页数以及对象，所有的数据都会被预选计算好并存储在 runtime.class_to_size 和 runtime.class_to_allocnpages 等变量中。

| class | bytes/obj | bytes/span | objects | tail waste | max waste |
| ----- | --------- | ---------- | ------- | ---------- | --------- |
| 1     | 8         | 8192       | 1024    | 0          | 87.50%    |
| 2     | 16        | 8192       | 512     | 0          | 43.75%    |
| 3     | 24        | 8192       | 341     | 0          | 29.24%    |
| 4     | 32        | 8192       | 256     | 0          | 46.88%    |
| 5     | 48        | 8192       | 170     | 32         | 31.52%    |
| 6     | 64        | 8192       | 128     | 0          | 23.44%    |
| 7     | 80        | 8192       | 102     | 32         | 19.07%    |
| …     | …         | …          | …       | …          | …         |
| 67    | 32768     | 32768      | 1       | 0          | 12.50%    |

跨度类为1-67，byte/obj 表示对象大小上限的字节数，objects 表示最多可以存储多少个对象。当存储的对象刚好比上一级跨度的对象大小大1字节，如 33 字节对象要存在跨度类 5 而不是 4，就会存在最大的空间浪费率。

除了上述 67 个跨度类之外，运行时中还包含 ID 为 0 的特殊跨度类，它能够管理大于 32KB 的特殊对象。

跨度类中除了存储类别的 ID 之外，它还会存储一个 noscan 标记位，表示对象是否包含指针，垃圾回收会对包含指针的 mspan 结构体进行扫描。spanClass 是一个 uint8 整数，前 7 存储跨度类的 id，最后一位表示是否包含指针。

### 3.3.2 线程缓存

线程缓存对应结构体 runtime.mcache，它会与线程上的处理器一一绑定，主要用来缓存用户程序申请的微小对象。每一个线程缓存都持有 68 * 2 个 runtime.mspan，它们存储在 mcache 对象的 alloc 字段中。线程缓存还有一个微分配器，用于管理 16 字节以下的对象，只会用于分配非指针类型的内存，tiny 指向堆中的一片内存，tinyoffset 是下一个空闲内存所在的偏移量，tinyAllocs 记录内存分配器中分配的对象个数。

```go
type mcache struct {
	// 微分配器
	tiny       uintptr
	tinyoffset uintptr
	tinyAllocs uintptr

	// mspan 数组
	alloc [numSpanClasses]*mspan
	...
}
```

runtime.allocmcache() 函数对线程缓存 mcache 进行初始化，运行时在初始化处理器时将调用，使用 runtime.mheap 中的线程缓存分配器来初始化。初始化后 mspan 数组所有成员都是空的占位符 emptyspan。

```go
func allocmcache() *mcache {
	var c *mcache
	systemstack(func() {
		lock(&mheap_.lock)
		c = (*mcache)(mheap_.cachealloc.alloc())
		c.flushGen = mheap_.sweepgen
		unlock(&mheap_.lock)
	})
	for i := range c.alloc {
		c.alloc[i] = &emptymspan
	}
	c.nextSample = nextSample()
	return c
}
```

当用户程序申请内存时，如果对应跨度类为空，会从上一级组件即中心缓存中获取新的 runtime.mspan，填入 alloc 数组。

```go
func (c *mcache) refill(spc spanClass) {
	s := c.alloc[spc]
	s = mheap_.central[spc].mcentral.cacheSpan()
	c.alloc[spc] = s
}
```

### 3.3.3 中心缓存

中心缓存对应结构体 runtime.mcentral，每个中心缓存都会管理某个跨度类的内存管理单元，它会同时持有两个 runtime.spanSet，分别存储包含空闲对象和不包含空闲对象的内存管理单元。

访问中心缓存中的内存管理单元需要使用互斥锁。

```go
type mcentral struct {
	spanclass spanClass
	partial  [2]spanSet
	full     [2]spanSet
	...
}
```

### 3.3.4 页堆

页堆对应结构体 runtime.mheap，它是全局变量存储，堆上初始化的所有对象都由该结构体统一管理。

```go
type mheap struct {
	arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena

	central [numSpanClasses]struct {
		mcentral mcentral
		pad      [(cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize) % cpu.CacheLinePadSize]byte
	}
	...
}
```

成员变量 central 是全局的中心缓存列表，包含一个长度为 136 的 runtime.mcentral 数组，其中 68 个为跨度类需要 scan 的中心缓存，另外的 68 个是 noscan 的中心缓存。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_mheap_central.png)

成员变量 arenas 负责管理所有稀疏内存空间的二维数组，它是 heapArena 指针的数组的指针的数组，其中一个 heapArena 对象负责管理 64MB 的内存空间。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_mheap_arenas.png)

## 3.4 对象分类与内存分配

根据申请内存的大小，将对象分为微对象、小对象和大对象三种。其中微对象不可以是指针类型，如果是指针将被当作小对象。

| 类别   | 大小        |
| ------ | ----------- |
| 微对象 | (0, 16B)    |
| 小对象 | [16B, 32KB] |
| 大对象 | (32KB, +∞)  |

堆上所有的对象都会通过调用 runtime.newobject 函数分配内存，该函数会调用 runtime.mallocgc 分配指定大小的内存空间。

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	mp := acquirem()
	mp.mallocing = 1

	c := gomcache()
	var x unsafe.Pointer
	noscan := typ == nil || typ.ptrdata == 0
	if size <= maxSmallSize {
		if noscan && size < maxTinySize {
			// 微对象分配
		} else {
			// 小对象分配
		}
	} else {
		// 大对象分配
	}

	publicationBarrier()
	mp.mallocing = 0
	releasem(mp)

	return x
}
```

对于微对象，先使用微型分配器，再依次尝试线程缓存、中心缓存和堆分配内存。对于小对象，依次尝试使用线程缓存、中心缓存和堆分配内存。对于大对象，直接在堆上分配内存。

微分配器可以将多个较小的内存分配请求合入同一个内存块中，只有当内存块中的所有对象都需要被回收时，整片内存才可能被回收。例如，微分配器已经在 16 字节的内存块中分配了 12 字节的对象，如果下一个待分配的对象小于 4 字节，它会直接使用上述内存块的剩余部分，减少内存碎片。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_tiny_allocator.png)

# 4. 垃圾回收

垃圾回收算法针对用户程序在堆中手动申请的内存。不同编程语言通常会使用手动和自动两种方式管理内存，C、C++ 以及 Rust 等编程语言使用手动的方式管理内存，工程师需要主动申请或者释放内存；而 Python、Ruby、Java 和 Go 等语言使用自动的垃圾收集机制释放无用内存。

Go 采用的是三色标记法标记和清理内存，使用混合写屏障策略，优化思路是尽可能减少 STW 的时间。

## 4.1 标记清除

标记清除（Mark-Sweep）算法是最常见的垃圾收集算法，标记清除收集器是跟踪式垃圾收集器，其执行过程可以分成标记（Mark）和清除（Sweep）两个阶段：

1. 标记阶段：从根对象出发查找并标记堆中所有存活的对象；
2. 清除阶段：遍历堆中的全部对象，回收未被标记的垃圾对象并将回收的内存加入空闲链表；

在标记阶段，内存空间中包含多个对象，我们从根对象出发依次遍历对象的子对象并将从根节点可达的对象都标记成存活状态，即 A、C 和 D 三个对象，剩余的 B、E 和 F 三个对象因为从根节点不可达，所以会被当做垃圾。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_mark_sweep_mark_phase.png)

标记阶段结束后会进入清除阶段，在该阶段中收集器会依次遍历堆中的所有对象，释放其中没有被标记的 B、E 和 F 三个对象并将新的空闲内存空间以链表的结构串联起来，方便内存分配器的使用。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_mark_sweep_sweep_phase.png)

以上是最基础的标记清除算法，垃圾收集器从垃圾收集的根对象出发，递归遍历这些对象指向的子对象并将所有可达的对象标记成存活；标记阶段结束后，垃圾收集器会依次遍历堆中的对象并清除其中的垃圾。整个过程需要标记对象的存活状态，有长时间的 STW（stop the world），用户程序在垃圾收集的过程中不能执行。

## 4.2 三色标记法

为了解决原始标记清除算法带来的长时间 STW，多数现代的追踪式垃圾收集器都会实现三色标记算法的变种以缩短 STW 的时间。

三色标记算法将程序中的对象分成黑色、灰色和白色三类：

- 黑色对象：已经被标记过，且该对象指向的对象也都被标记过，可以看做程序正在引用的对象
- 灰色对象：已经被标记过，但该对象指向的对象还不一定全被标记，可以看做临时的标记对象，一次标记循环后会被置为黑色活跃的对象；
- 白色对象：还未被标记，可以看做有可能被清理的对象，如果一次标记循环后未被标记为黑色，则可以视为未被引用而被 GC 清理；

在垃圾收集器开始工作时，程序中不存在任何的黑色对象，垃圾收集的根对象会被标记成灰色，垃圾收集器只会从灰色对象集合中取出对象开始扫描，当灰色集合中不存在任何对象时，标记阶段就会结束。

三色标记法的工作原理包含以下步骤：

1. 将根对象标记为灰色；
2. 从灰色对象的集合中选择一个灰色对象并将其标记成黑色，将其指向的所有对象都标记成灰色；
3. 重复上述两个步骤直到对象图中不存在灰色对象；
4. 对没有被标记的白色对象进行 GC；

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_tri_color_mark_sweep.png)

当三色的标记清除的标记阶段结束之后，应用程序的堆中就不存在任何的灰色对象，我们只能看到黑色的存活对象以及白色的垃圾对象，垃圾收集器可以回收这些白色的垃圾，下面是使用三色标记垃圾收集器执行标记后的堆内存，堆中只有对象 D 为待回收的垃圾：

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_tri_color_mark_sweep_after_mark_phase.png)

因为用户程序可能在标记执行的过程中修改对象的指针，所以三色标记清除算法本身是不可以并发或者增量执行的，它仍然需要 STW。

当在三色标记过程中，如果已经不存在灰色对象了，而这时候用户程序建立了从黑色对象到白色对象的引用，被引用的白色对象有可能会被错误地回收，这种错误称为悬挂指针，即指针没有指向特定类型的合法对象，影响了内存的安全性。想要并发或者增量地标记对象还是需要使用屏障技术。

## 4.3 屏障技术

内存屏障技术是一种屏障指令，它可以让 CPU 或者编译器在执行内存相关操作时遵循特定的约束，目前多数的现代处理器都会乱序执行指令以最大化性能，但是该技术能够保证内存操作的顺序性，在内存屏障前执行的操作一定会先于内存屏障后执行的操作。

想要在并发或者增量的标记算法中保证正确性，我们需要达成以下两种三色不变性（Tri-color invariant）中的一种：

- 强三色不变性：黑色对象不会指向白色对象，只会指向灰色对象或者黑色对象；
- 弱三色不变性：黑色对象指向的白色对象必须包含一条从灰色对象经由多个白色对象的可达路径；

垃圾收集中的屏障技术更像是一个钩子方法，它是在用户程序读取对象、创建新对象以及更新对象指针时执行的一段代码，根据操作类型的不同，我们可以将它们分成读屏障（Read barrier）和写屏障（Write barrier）两种，因为读屏障需要在读操作中加入代码片段，对用户程序的性能影响很大，所以编程语言往往都会采用写屏障保证三色不变性。

Go 语言中主要使用了两种写屏障技术，分别是 Dijkstra 提出的插入写屏障和 Yuasa 提出的删除写屏障。

**插入写屏障**

当执行类似 *slot = ptr 的指针赋值表达式时，执行以下写屏障代码，先通过 shade 函数尝试改变新指向的对象的颜色，如果是白色则置为灰色。

```go
writePointer(slot, ptr)
	shade(ptr)
	*slot = ptr
```

如下图示例，当 A 从指向 B 修改为指向 C 时，将 C 置为灰色，它们最后都会被标记为黑色，避免了 C 被错误地回收。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_dijkstra_insert_write_barrier.png)

这是一种相对保守的屏障技术，它会将有存活可能的对象都标记成灰色以满足强三色不变性，实际上不再存活的 B 对象在这个循环中仍然存活，直至下一个循环才会被标记回收。

**删除写屏障**

当执行类似 *slot = ptr 的指针赋值表达式时，执行以下写屏障代码，通过 shade 函数尝试改变原本指向对象的颜色，如果是白色则置为灰色。

```go
writePointer(slot, ptr)
	shade(*slot)
	*slot = ptr
```

如下图示例，当 A 从指向 B 改为指向 C 时，A 原本指向的 B 已经是灰色，所以不会改变 B 的颜色。当 B 指向 C 引用删除时，B 原本指向的 C 从白色改为灰色，保证了 C 以及下游对象在垃圾收集循环中存活。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_yuasa_delete_write_barrier.png)

在老对象的引用被删除时，将白色的老对象涂成灰色，这样删除写屏障就可以保证弱三色不变性，老对象引用的下游对象一定可以被灰色对象引用，避免了出现悬挂指针。

**混合写屏障**

Go 语言在 v1.8 组合 Dijkstra 插入写屏障和 Yuasa 删除写屏障构成了如下所示的混合写屏障，该写屏障会将被覆盖的对象标记成灰色并在当前栈没有扫描时将新对象也标记成灰色。

```go
writePointer(slot, ptr)
	shade(*slot)
	if current stack is grey:
		shade(ptr)
	*slot = ptr
```

在垃圾收集的标记阶段，我们还需要将创建的所有新对象都标记成黑色，防止新分配的栈内存和堆内存中的对象被错误地回收，因为栈内存在标记阶段最终都会变为黑色，所以不再需要重新扫描栈空间。

## 4.4 增量和并发

传统的垃圾收集算法会在垃圾收集的执行期间暂停应用程序，垃圾收集器会抢占 CPU 的使用权占据大量的计算资源以完成标记和清除工作，为了减少应用程序暂停的最长时间和垃圾收集的总暂停时间，我们会使用下面的策略优化现代的垃圾收集器：

- 增量垃圾收集：增量地标记和清除垃圾，降低应用程序暂停的最长时间；
- 并发垃圾收集：利用多核的计算资源，在用户程序执行时并发标记和清除垃圾；

增量垃圾收集是减少程序最长暂停时间的一种方案，它可以将原本时间较长的暂停时间切分成多个更小的 GC 时间片，虽然从垃圾收集开始到结束的时间更长了，但是这也减少了应用程序暂停的最大时间。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_incremental_collector.png)

并发垃圾收集不仅能够减少程序的最长暂停时间，还能减少整个垃圾收集阶段的时间，通过开启读写屏障、利用多核优势与用户程序并行执行。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/go/go_concurrent_collector.png)

# 5. 参考

* [《Go语言设计与实现》](https://book.douban.com/subject/35635836/)

