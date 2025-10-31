# 1. 简介

Redis 源码仓库地址：https://github.com/redis/redis

Redis 是使用 C 实现的，以下基于 Redis 6.0 进行介绍。

# 2. SDS

Redis 构建了一种名为 SDS（simple dynamic string，简单动态字符串）的类型，作为默认的字符串实现。这个结构体定义在 sds.h 中。

Redis 旧版本的 SDS 结构体定义：

```c
struct sdshdr {
    int len;    // 记录buf数组中已使用的字节数（字符串实际长度）
    int free;   // 记录buf数组中未使用的字节数
    char buf[]; // 数组，存储实际字符串内容，末尾保留'\0'
};
```

在 Redis 3.2 开始，针对不同长度的字符串，优化为了 5 种数据结构，以减少内存占用。通过 flags 字段的低三位表示 SDS 类型，不同类型通过字段表示长度。

```c
struct __attribute__ ((__packed__)) sdshdr5 { // 实际不会用到，只是为了访问 flags 成员
    unsigned char flags; // 低3位存储类型，高5位存储长度，可表示最大长度 31 字节
    char buf[];          // 数组
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len;         // 已使用长度，1 字节，可表示最大长度 255 字节
    uint8_t alloc;       // 总分配空间，1 字节，不包含头部和'\0'
    unsigned char flags; // 低3位存储类型，高5位未使用
    char buf[];          // 数组
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len;        // 已使用长度，2 字节
    uint16_t alloc;      // 总分配空间，2 字节，不包含头部和'\0'
    unsigned char flags; // 低3位存储类型，高5位未使用
    char buf[];          // 数组
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len;        // 已使用长度，4 字节
    uint32_t alloc;      // 总分配空间，4 字节，不包含头部和'\0'
    unsigned char flags; // 低3位存储类型，高5位未使用
    char buf[];          // 数组
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len;        // 已使用长度，8 字节
    uint64_t alloc;      // 总分配空间，8 字节，不包含头部和'\0'
    unsigned char flags; // 低3位存储类型，高5位未使用
    char buf[];          // 数组
};
```

以下为 sdshdr8 的对象示例图，len=5 表示 buf 数组中已使用字符串的长度，alloc=20 表示实际申请的 buf 数组长度，flags=1 表示这是一个 sdshdr8 对象：

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/database/redis_struct_sdshdr.jpg)

SDS 结构体设计的好处在于：

* 可以在 O(1) 时间快速获取字符串的使用长度；
* 通过空间预分配和惰性释放，减少频繁的内存重新分配；
* 支持在需要时自动检查和扩容，防止内存溢出，无需手动修改；

SDS 并非将 buf 数组视为以 '\0' 结束的字符串，而是一个二进制数组，以 len 成员来标记数组长度，因此可以在这里存入任何数据，包括 '\0'。但为了兼容 C 字符串，Redis 总是会多申请一个字节来容纳这个字符串，并在最后写入一个 '\0' 字节，因此它可以使用一部分 strings.h 的函数。

# 3. 字典

哈希和集合的底层实现是字典，字典同时也是 Redis 数据库保存 key-value 的底层实现，结构体定义在 dict.h 中。

字典包含哈希表、rehash 索引及一些其他成员。ht 是长度为 2 的哈希表数组，一般只使用 ht[0]，当哈希表在进行 rehash 时才会用到 ht[1]。

```c
typedef struct dict {
    dictType *type;          // 指向 dictType 结构的指针
    void *privdata;          // 私有数据
    dictht ht[2];            // 哈希表
    long rehashidx;          // rehash 索引，当值为 -1 时表示不在 rehash
    unsigned long iterators; // 正在执行的迭代器数量
} dict;
```

dictType 结构体保存了一些用于操作特定类型键值对的函数。

```c
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);                             // 计算哈希值
    void *(*keyDup)(void *privdata, const void *key);                      // 复制键
    void *(*valDup)(void *privdata, const void *obj);                      // 复制值
    int (*keyCompare)(void *privdata, const void *key1, const void *key2); // 对比键
    void (*keyDestructor)(void *privdata, void *key);                      // 销毁键
    void (*valDestructor)(void *privdata, void *obj);                      // 销毁值
} dictType;
```

哈希表保存哈希表节点数组和大小。

```c
typedef struct dictht {
    dictEntry **table;      // 指向哈希表节点数组的指针
    unsigned long size;     // 哈希表大小
    unsigned long sizemask; // 哈希表大小-1，用于计算索引值
    unsigned long used;     // 已有节点数量
} dictht;
```

哈希表节点保存着一个键值对，以及指向下个哈希表节点的指针，用于存在多个哈希值相同的键连接成链表，以解决键冲突的问题。

```c
typedef struct dictEntry {
    void *key; // 键
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v; // 值
    struct dictEntry *next; // 下个哈希表节点指针
} dictEntry;
```

以下为一个 dict 对象的例子。并未进行 rehash，所以数据全部在哈希表 ht[0] 中，哈希表大小为 4，其中存在 3 个节点，由于其中两个节点存在哈希值冲突，所以用链表连接起来。

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/database/redis_struct_dict.jpg)

## 3.1 哈希算法

将键值对添加到字典时，需要用键计算出哈希值，然后根据哈希值，将键值对放到哈希表数组的指定索引位置上。

Redis 使用 MurmurHash2 算法来计算键的哈希值。

当不同的键被分配到哈希表数组的同一个索引时，就会产生键冲突。Redis 哈希表使用链地址法（separate chaining）来解决键冲突，这些分配到同一个索引的键值对节点，通过指针构成一个单向链表。

## 3.2 rehash

哈希表的负载因子（load factor）是已存储键值对数量和哈希表容量的比值，较高时会增加键冲突概率，较低时内存利用率低，此时需要对哈希表的大小进行扩展或收缩，这个过程称为 rehash。

为了避免 rehash 对服务器性能以及对应哈希的查询造成影响，rehash 不是一次性集中完成的，而是分多次渐进地完成。

rehash 步骤如下：

1. 根据实际存储的键值对的数量，为哈希表 ht[1] 分配一定大小的空间；
2. 索引计数器变量 rehashidx 设为 0，表示 rehash 开始；
3. 将保存在 ht[0] 的所有键值对逐个重新计算哈希值和索引值，放到 ht[1] 上；
4. 在 rehash 进行期间，对字典的增删改查操作还会顺带将 ht[0] 索引上的所有键值对 rehash 到 ht[1]，并将 rehashidx 加一；
5. 当 ht[0] 的所有键值对都迁移到 ht[1] 后，将 rehashidx 设为 -1，表示 rehash 完成，释放 ht[0]，将 ht[1] 设置为 ht[0]，在 ht[1] 新建一个空白哈希表；

在渐进式 rehash 的过程中，对字典的查找、更新、删除操作会在两个哈希表上进行，而新增的键值对则直接保存到 ht[1]。

# 4. 列表

列表的底层实现是双向链表，结构体定义在 adlist.h 中。

链表本身包含了表头节点、表尾节点、链表长度，还有实现多态链表需要的函数：

```c
typedef struct list {
    listNode *head;    // 表头节点
    listNode *tail;    // 表尾节点
    unsigned long len; // 链表长度
    void *(*dup)(void *ptr);            // 节点值复制函数
    void (*free)(void *ptr);            // 节点值释放函数
    int (*match)(void *ptr, void *key); // 节点值对比函数
} list;
```

链表中的节点包含值的指针和向前、向后的指针：

```c
typedef struct listNode {
    struct listNode *prev; // 向前指针
    struct listNode *next; // 向后指针
    void *value;           // 值指针
} listNode;
```

以下为一个 list 对象的例子：

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/database/redis_struct_list.jpg)

# 5. 跳跃表

跳跃表（skiplist）时有序集合的底层实现之一，结构体定义在 server.h 中。它是一种有序的链表结构，并且在每个节点维持多个指向后面节点的指针，加快访问速度。

跳跃表支持平均 O(logN)、最坏 O(N) 复杂度的节点查找，还支持顺序性批量处理节点。它在大部分情况效率可以媲美平衡树，且实现比平衡树更简单。

跳跃表本身保存了表头节点、表尾节点、跳跃表长度和节点的最大层数。

```c
typedef struct zskiplist {
    struct zskiplistNode *header, *tail; // 表头节点、表尾节点
    unsigned long length;                // 跳跃表长度
    int level;                           // 节点的最大层数
} zskiplist;
```

跳跃表节点保存了节点元素、分数、后退指针和多个层，每个层有前进指针以及跨度。

节点有一到多层，最多可以到 32 层，每层有前进指针用于访问表尾方向的其他节点，跨度则记录两个节点间的距离。后退指针指向当前指针的前一个节点。元素和分数则表示该节点记录的值和设置的分数。

```c
typedef struct zskiplistNode {
    sds ele;                           // 元素
    double score;                      // 分数
    struct zskiplistNode *backward;    // 后退指针
    struct zskiplistLevel {
        struct zskiplistNode *forward; // 前进指针
        unsigned long span;            // 跨度
    } level[];                         // 层
} zskiplistNode;
```

跳跃表可以通过节点记录的多层来加快访问其他节点的速度，越高层跳跃的距离越大。创建新的跳跃表节点时，会随机生成 1 到 32 之间的值作为 level数组大小，也就是层的高度，每多一层的概率是 1/4。

以下为一个跳跃表的例子：

![](https://blog-1304941664.cos.ap-guangzhou.myqcloud.com/article_material/database/redis_struct_skiplist.jpg)

# 6. 参考

* [《Redis设计与实现》](https://book.douban.com/subject/25900156/)

