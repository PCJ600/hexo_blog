---
layout: next
title: Redis学习笔记(三)——字典
date: 2021-12-21 20:40:34
categories: Redis
tags: Redis
---

## 前言

字典是一种用于保存键值对的数据结构，Redis数据库使用字典做为底层实现，字典也是哈希键的底层实现之一。

C语言中并没有内置字典这个数据结构，Redis自己实现了字典。

以下结合源码分析Redis字典的设计与实现，源码版本：[Redis 6.0.10](https://github.com/redis/redis/releases/tag/6.0.10)

<!-- more -->

## 问题思考

C语言中如何设计一个通用的字典和哈希表？

* [字典的设计与实现](#jump1)

  * [哈希表和哈希表节点的设计](#jump2)
  * [字典的数据结构设计](#jump3)
  * [如何设计多态字典](#jump4)

* [哈希算法](#jump5)

  * [为什么数组长度设计为2的n次方](#jump6)

* [如何解决键冲突](#jump7)

* [rehash操作](#jump8)

  * [为什么要rehash](#jump9)
  * [rehash的实现步骤](#jump10)
  * [为什么不直接复制ht[0]的节点到ht[1]，而是要重新计算哈希索引值再rehash](#jump11)

* [rehash的触发条件是什么](#jump12)

  * [负载因子的概念](#jump13)
  * [哈希表扩展策略](#jump14)
  * [哈希表收缩策略](#jump15)

* [渐进式rehash](#jump16)

  * [为什么要渐进式](#jump17)
  * [渐进式rehash的实现](#jump18)
  * [增、删、改、查场景中的rehash实现](#jump19)

* [字典API](#jump20)

  * [随机返回一个键值对](#jump21)
  * [字典迭代器的设计与实现](#jump22)
  * [dictScan算法](#jump23)

  

### 字典的设计与实现<span id="jump1"></span>

Redis中使用哈希表来实现字典，一个哈希表中有多个哈希表节点，每个节点存储一个键值对。

#### 哈希表和哈希表节点的设计<span id="jump2"></span>

哈希表的结构体定义如下：

```c
typedef struct dictht {
    dictEntry **table;			// 哈希表数组，每个数组是一个链表 (链地址法)
    unsigned long size;			// 记录哈希表大小
    unsigned long sizemask;		// 掩码，用于计算哈希索引，值总是等于size - 1
    unsigned long used;			// 记录已有哈希表节点个数。
} dictht;
```

* table是一个指针数组，每个数组相当于一个链表；**链地址法**实现哈希表
* size表示哈希表大小，取值总是为2的幂
* sizemask为掩码，用于计算索引，取值总是为size - 1
* used记录已有哈希表节点个数

一个大小为4的空哈希表如下图：
![](image1.png)

哈希表节点的结构体定义如下：

```c
typedef struct dictEntry {
	void *key;
	union {
		void *val;
		uint64_t u64;
		int64_t s64;
	} v;
    struct dictEntry *next;
};
```

* key保存键，对于哈希键场景，这个key存储的实际类型是sds
* v保存值，设置为联合类型用于保存不同类型的值
* next保存链表的下一个节点，以链地址法解决哈希键冲突问题
* value保存值，用联合类型的好处是可以保存不同类型的值



#### 字典的数据结构设计<span id="jump3"></span>

字典的结构体定义如下：

```c
typedef struct dict {
    dictType *type;		     // 类型特定函数
    void *privdata;
    dictht ht[2];			 // 两个哈希表, ht[1]用于rehash
    long rehashidx; 		 /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```

#### 如何设计多态字典？<span id="jump4"></span>

type和privdata用于实现多态字典，dictType结构体定义如下：

```c
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);			// 计算hash值
    void *(*keyDup)(void *privdata, const void *key);	// 复制键
    void *(*valDup)(void *privdata, const void *obj);	// 复制值
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);	// 比较键
    void (*keyDestructor)(void *privdata, void *key);	// 销毁键
    void (*valDestructor)(void *privdata, void *obj);	// 销毁值
} dictType;
```

考虑字典的增、删、改、查场景，每个场景必定会涉及如下动作：

* 哈希值的计算
* 键、值的复制
* 键、值的销毁

* 比较两个键是否相同

对于不同类型的键值对，以上这些动作的具体实现必定是有差异的。设计dictType的好处在于，通过挂接钩子函数消除这个差异，简化了字典的增、删、改、查等API的实现。（比如复制键的场景，不同类型的键值对只需调用同一个钩子函数`keyDup`即可）

钩子函数的挂接，在字典创建时完成，参考`dictCreate`， 时间复杂度O(1)

```c
dict *dictCreate(dictType *type, void *privDataPtr) {
    dict *d = zmalloc(sizeof(*d));
    _dictInit(d,type,privDataPtr);
    return d;
}

int _dictInit(dict *d, dictType *type, void *privDataPtr) {
    _dictReset(&d->ht[0]);
    _dictReset(&d->ht[1]);
    d->type = type;				// 挂接钩子函数
    d->privdata = privDataPtr;
    d->rehashidx = -1;
    d->iterators = 0;
    return DICT_OK;
}
```

* ht是一个数组，包含2个哈希表。一般情况下只会用到ht[0]，ht[1]用于字典的rehash操作。

* rehashidx表示字典rehash的进度，取值为-1表示没有进行rehash。



**举例：**

新增一个哈希键key，4个键值对。

```
127.0.0.1:6379> hmset key key1 value1 key2 value2 key3 value3 key4 value4
OK
127.0.0.1:6379> object encoding key
"hashtable"
127.0.0.1:6379> hgetall key
1) "key2"
2) "value2"
3) "key4"
4) "value4"
5) "key1"
6) "value1"
7) "key3"
8) "value3"
```

注：必须先修改redis配置项`hash-max-ziplist-value`（这个例子中要设置为4），否则这里的哈希键默认编码方式为`ziplist`，而不是`hashtable` ！

**GDB打印这个字典，具体方法如下：**

```c
// 1、打印哈希键key，在0号数据库键空间中先找到这个key。
(gdb) p (sds)server.db[0].dict.ht[0].table[1].key
$1 = (sds) 0x564377c118c1 "key"

// 2、打印哈希键的值, 注意值也是一个字典结构，有4个键值对
(gdb) p *(robj *)server.db[0].dict.ht[0].table[1].v
$2 = {type = 4, encoding = 2, lru = 12185874, refcount = 1, ptr = 0x564377c20450} // 字典位于0x564377c20450地址处
#define OBJ_HASH 4 			/* Hash object. type = 4*/
#define OBJ_ENCODING_HT 2   /* Encoded as hash table， encoding = 2 */

// 3、打印这个字典
(gdb) set print pretty on
(gdb) p *(dict *)0x564377c20450
$3 = {
  type = 0x5643777e4860 <hashDictType>,
  privdata = 0x0,
  ht = {{							
      table = 0x564377c10eb0, // 哈希表地址0x564377c10eb0
      size = 4,
      sizemask = 3,
      used = 4
    }, {			// ht[1]用于rehash，一般情况下rehashidx=-1，ht[1]用不上
      table = 0x0,
      size = 0,
      sizemask = 0,
      used = 0
    }},
  rehashidx = -1,	// 等于-1表示没有进行rehash
  iterators = 0
}

// 4、打印字典中的哈希表，哈希表中的指针数组长度为4
// 索引0的链表长度为1，内容：(key, value2) -> NULL
(gdb) p ((dictEntry **)0x564377c10eb0)[0]
$4 = (dictEntry *) 0x564377c1a250
(gdb) p (sds)((dictEntry *) 0x564377c1a250)->key
$5 = (sds) 0x564377c10ee1 "key2"
(gdb) p (sds)((dictEntry *) 0x564377c1a250)->v
$6 = (sds) 0x564377c1a231 "value2"
(gdb) p ((dictEntry *) 0x564377c1a250)->next
$7 = (struct dictEntry *) 0x0

// 索引1的链表长度为0
(gdb) p ((dictEntry **)0x564377c10eb0)[1]
$8 = (dictEntry *) 0x0

// 索引2的链表长度为2，内容：(key4, value4) -> (key1, value1)
(gdb) p ((dictEntry **)0x564377c10eb0)[2]
$9 = (dictEntry *) 0x564377c0f530
(gdb) p (sds)((dictEntry *) 0x564377c0f530)->key
$10 = (sds) 0x564377c1a1b1 "key4"
(gdb) p (sds)((dictEntry *) 0x564377c0f530)->v
$11 = (sds) 0x564377c1a1d1 "value4"
(gdb) p ((dictEntry *) 0x564377c0f530)->next
$12 = (struct dictEntry *) 0x564377c1a500
(gdb) p (sds)((dictEntry *) 0x564377c1a500)->key
$13 = (sds) 0x564377c10dd1 "key1"
(gdb) p (sds)((dictEntry *) 0x564377c1a500)->v
$14 = (sds) 0x564377c1a4e1 "value1"
(gdb) p ((dictEntry *) 0x564377c1a500)->next
$15 = (struct dictEntry *) 0x0

// 索引3的链表长度为1, 内容：(key3, value3) -> NULL
(gdb) p ((dictEntry **)0x564377c10eb0)[3]
$16 = (dictEntry *) 0x564377c1a190
(gdb) p (sds)((dictEntry *) 0x564377c1a190)->key
$17 = (sds) 0x564377c1a151 "key3"
(gdb) p (sds)((dictEntry *) 0x564377c1a190)->v
$18 = (sds) 0x564377c1a171 "value3"
(gdb) p ((dictEntry *) 0x564377c1a171)->next
$19 = (struct dictEntry *) 0x0
```



下图展示了这个字典：
![](image2.png)


### 哈希算法<span id="jump5"></span>

Redis内置两种哈希算法：

* `dictGenHashFunction` 用于字符串哈希，区分大小写。
* `dictGenCaseHashFunction`用于字符串哈希，不区分大小写。

```c
/* The default hashing function uses SipHash implementation in siphash.c. */

uint64_t dictGenHashFunction(const void *key, int len) {
    return siphash(key,len,dict_hash_function_seed);
}

uint64_t dictGenCaseHashFunction(const unsigned char *buf, int len) {
    return siphash_nocase(buf,len,dict_hash_function_seed);
}
```

**索引值如何计算？**

先通过哈希算法求出64位的哈希值，再利用这个哈希值和sizemask掩码得到索引值，源码参考`_dictKeyIndex`，这里简单给出伪代码：

```c
// 返回key的索引值。
long GetKeyIndex(dict *d, const void *key) {
    uint64_t hash = dictGenHashFunction((unsigned char *)key, strlen((char *)key));
	return hash & dict->ht[0].sizemask;
}
```

举例：一个大小为4的空哈希表，新增一个键值对(k0, v0)， 假设通过哈希算法求出的哈希值为666，按照上面的伪代码，求出索引值为 666 % ( 4 - 1) = 0，这说明这个节点需要插入到第0个哈希表数组，如图所示：

![](image3.png)


**REDIS 3.0中的哈希算法**

使用Murmurhash2算法计算键的哈希值，返回uint32_t类型的哈希值。

```c
unsigned int dictGenHashFunction(const void *key, int len);
```



#### 为什么哈希表数组长度设计为2的n次方，有什么好处？<span id="jump6"></span>

**好处1**：保证哈希表元素分布均匀，减少哈希冲突。以下举例说明：

* 如果哈希表长度size不设置为2的n次方（比如这里设为15），那sizemask = 14, 二进制表示：1110；这时假设我要插两个元素，哈希值分别是6 (110), 7 (111)，分别与1110进行与运算后，两者得到的结果是相同的，因此这两个元素存储到同一个链表中，导致效率低下。

* 如果size设计为2的n次方(比如16），sizemask的二进制为15，二进制表示：1111，此时哈希值6和7分别与1111运行与运算后得到结果110和111，因此这两个元素存储到不同的链表，保证了哈希表元素分布均匀。

**好处2**：利用位运算比取模运算更快的特点，提高求哈希索引的效率。

计算哈希索引的本质是取模求余数，但计算机不擅长做模运算，更擅长做位运算。我们注意到，只有在size是2的幂的条件下， 等式`hash & (size - 1) == hash % size  `才成立。

附：[位运算和取模运算的运算效率对比](https://www.codeprj.com/blog/acae4c1.html)



### 如何解决键冲突<span id="jump7"></span>

常见的解决哈希冲突方法：

* 开放寻址法 （线性探查法，平方探查法等）
* 链地址法
* 再哈希法
* ......

Redis中使用**链地址法**解决哈希键冲突，即将同一个索引值的多个哈希表节点通过单链表连接起来。

链地址法优点是实现简单，适用于预先不知道哈希表数据大小，插入删除频繁的场景。



### rehash操作<span id="jump8"></span>

#### 为什么要rehash？<span id="jump9"></span>

哈希表的扩容和收缩操作通过rehash完成，rehash的目的在于将哈希表的负载因子(键的个数 / 哈希表大小)始终维持在合理范围内，以达到最优的时间和空间性能。

如果不设计字典的rehash动作，试考虑以下两个场景中的问题：

* 新增大量的键值对到哈希表，查询某个键的时间复杂度从O(1)退化到O(N)，时间性能差。
* 考虑一个保存大量键值对的哈希表，删除这个表的所有键后，哈希表数组的空间并没有释放，出现内存占用高甚至内存不足的问题，空间性能差。

综上，必须设计字典的rehash动作，当哈希表保存键值对过多或过少时，对哈希表进行动态地扩容和收缩。

#### rehash的实现步骤<span id="jump10"></span>

1、为字典的ht[1]分配空间，分配的空间大小取决于执行的操作(扩展或收缩）, 以及当前键值对数目（即ht[0].used的值），源码参考`_dictExpandIfNeeded`

* 如执行扩展操作，ht[1]的size设置为ht[0].used * 2, 参考`dictExpand`
* 如执行收缩操作，ht[1]的size设置为ht[0].used，参考`dictResize`

2、将ht[0]的所有键值对rehash到ht[1]，即重新计算ht[0]中所有键值对在ht[1]上的索引值，再将所有键值对加入ht[1]

3、ht[0]中所有键值对完成rehash后，释放ht[0], 将ht[0]指向ht[1]，ht[1]置为空哈希表，为下一次rehash做准备。 



**举例：**

假设这里对一个大小为4的字典做扩展操作，字典初始状态如下图：
![](image4.png)


1、为字典的ht[1]分配空间，大小为 4 * 2 = 8，分配空间后的字典如下图：

![](image5.png)


2、将ht[0]的所有4个键值对rehash到ht[1]，假设重新计算得到的索引值为2, 3, 6, 7，如下图：
![](image6.png)


3、释放ht[0]，将ht[0]指向ht[1], ht[1]置为空哈希表。和最初的字典对比，我们成功将哈希表的大小从4扩展到了8，如下图：

![](image7.png)


至此，我们搞清楚了为啥REDIS要在dict结构体中设计两个哈希表，原因如下：

* rehash过程中简化两个哈希表之间的数据迁移替换操作。

* 用于实现渐进式的rehash，参考[渐进式rehash](#jump16)



#### 为什么不直接复制ht[0]的节点到ht[1]，而是要重新计算哈希索引值再rehash？<span id="jump11"></span>

先看下索引值如何计算的: idx = `hashcode & (dict->size - 1)`

注意到，rehash之后哈希数组的大小发生了变化，所以索引值必须重新计算。



### rehash的触发条件是什么<span id="jump12"></span>

rehash操作会导致哈希表的扩展或收缩，以下分别讨论哈希表扩展、哈希表收缩两个场景是如何触发的。

#### 负载因子的概念<span id="jump13"></span>

这里先给出哈希表负载因子的定义：

```c
load_factor = ht[0].used / ht[0].size;
```

可以看出，负载因子大于1时，说明一定出现了哈希冲突。

#### 哈希表扩展策略<span id="jump14"></span>

以下条件中满足任意一个，即自动触发哈希表的扩展操作：

* Redis服务器没有执行`BGSAVE`或`BGREWRITEAOF`命令，且哈希表负载因子大于等于1
* Redis服务器正在执行`BGSAVE`或`BGREWRITEAOF`命令，且哈希表负载因子大于等于5

**说明**：执行`BGSAVE`或`BGREWRITEAOF`操作时，Redis会fork一个子进程。为了充分利用写时复制(Copy On Write)技术，在子进程存在期间，Redis服务器会提高负载因子，避免子进程存在期间进行哈希表扩展操作，引入不必要的内存写入，从而提高性能。

哈希表扩展策略的源码参考`_dictExpandifNeeded`

```c
static int dict_can_resize = 1;
static unsigned int dict_force_resize_ratio = 5;
static int _dictExpandIfNeeded(dict *d) {
    /* Incremental rehashing already in progress. Return. */
    if (dictIsRehashing(d)) return DICT_OK;

    /* If the hash table is empty expand it to the initial size. */
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    /* If we reached the 1:1 ratio, and we are allowed to resize the hash
     * table (global setting) or we should avoid it but the ratio between
     * elements/buckets is over the "safe" threshold, we resize doubling
     * the number of buckets. */
    if (d->ht[0].used >= d->ht[0].size &&		// 哈希表扩展的条件
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, d->ht[0].used*2);	// 扩展操作
    }
    return DICT_OK;
}

void dictEnableResize(void) {
    dict_can_resize = 1;
}

void dictDisableResize(void) {
    dict_can_resize = 0;
}

void updateDictResizePolicy(void) {
    // 如果没有活跃的子进程，dict_can_resize写1，否则写0
    if (!hasActiveChildProcess())	// (bgsave, bgwriteaof这些后台持久化操作在fork出的子进程中进行！
        dictEnableResize();
    else
        dictDisableResize();
}

int hasActiveChildProcess() {
    return server.rdb_child_pid != -1 ||
           server.aof_child_pid != -1 ||
           server.module_child_pid != -1;
}
```

#### 哈希表收缩策略<span id="jump15"></span>

哈希表的负载因子小于0.1时，Redis自动对哈希表做收缩操作。具体场景分析如下：

* 删除哈希表中的某个键值对之后，通过`htNeedsResize`判断是否需对哈希表做收缩，参考`hdelCommand`

* Redis服务器的定时事件中，周期性地检查每个数据库键空间字典，调用`htNeedsResize`判断是否需对哈希表做收缩操作。

哈希表收缩策略参考源码`htNeedsResize`

```c
#define HASHTABLE_MIN_FILL 10      /* Minimal hash table fill 10% */
#define dictSlots(d) ((d)->ht[0].size+(d)->ht[1].size)
#define dictSize(d) ((d)->ht[0].used+(d)->ht[1].used)
int htNeedsResize(dict *dict) {
    long long size, used;
    size = dictSlots(dict);
    used = dictSize(dict);
    return (size > DICT_HT_INITIAL_SIZE &&		
            (used*100/size < HASHTABLE_MIN_FILL));	// 负载因子小于0.1，表示需要收缩
}
```

哈希表收缩的实现参考源码`dictResize`

```c
#define DICT_HT_INITIAL_SIZE 4
int dictResize(dict *d) {
    unsigned long minimal;

    if (!dict_can_resize || dictIsRehashing(d)) return DICT_ERR;
    minimal = d->ht[0].used;			// 收缩后的哈希表大小为max(ht[0].used, 4)
    if (minimal < DICT_HT_INITIAL_SIZE)
        minimal = DICT_HT_INITIAL_SIZE;
    return dictExpand(d, minimal);
}
```

定时事件中判断每个数据库键空间字典是否需要收缩，参考源码`databasesCron`：

```c
void databasesCron(void) {
	// ....
	if (!hasActiveChildProcess()) {	// 如果没有执行bgsave, bgwriteaof
        for (j = 0; j < 16; j++) {
            tryResizeHashTables(resize_db % server.dbnum); // 尝试对每个数据库键空间字典和过期字典做哈希表收缩
            resize_db++;
        }		
    }
    // ...
}

void tryResizeHashTables(int dbid) {
    if (htNeedsResize(server.db[dbid].dict))
        dictResize(server.db[dbid].dict);
    if (htNeedsResize(server.db[dbid].expires))
        dictResize(server.db[dbid].expires);
}
```



### 渐进式rehash<span id="jump16"></span>

#### 为什么需要渐进式rehash<span id="jump17"></span>

考虑一个很大的字典（比如有1亿个key），当扩展或收缩哈希表时，需要将ht[0]中所有键值对rehash到ht[1]，这个计算量是非常庞大的，对Redis服务器性能有一定影响，甚至会导致服务器在一段时间内停止服务。

因此，Redis服务器并不是一次性将ht[0]键值对全部rehash到ht[1]，而是分多次地，渐进式地完成rehash动作。

#### 渐进式rehash的步骤<span id="jump18"></span>

* 为ht[1]分配空间，并设置字典的rehashIdx为0，表示rehash从ht[0]的第0号链表开始。

* 每执行一次字典的增、删、改、查操作时，Redis调用一次`_dictRehashStep`，将ht[0]哈希表在rehashIdx索引上的所有键值对rehash到ht[1]，rehash完成后将rehashIdx加1
* 当ht[0]所有键值对都rehash到ht[1]后，将rehashIdx置为-1，表示rehash完成。
* 另外，Redis服务器的定时任务中，会周期性地调用`dictRehashMilliseconds`方法，在指定毫秒事件内对字典进行主动的rehash。

可以看出，渐进式rehash采用分治思想，将rehash计算量平摊到对字典的增、删、改、查操作上，避免了一次性rehash的庞大计算量。

**rehash源码分析：**

1、通过`_dictRehashStep`执行一次单步的rehash，源码如下：

```c
static void _dictRehashStep(dict *d) {
    if (d->iterators == 0) dictRehash(d,1);
}

/* Performs N steps of incremental rehashing. Returns 1 if there are still
 * keys to move from the old to the new hash table, otherwise 0 is returned.
 *
 * Note that a rehashing step consists in moving a bucket (that may have more
 * than one key as we use chaining) from the old to the new hash table, however
 * since part of the hash table may be composed of empty spaces, it is not
 * guaranteed that this function will rehash even a single bucket, since it
 * will visit at max N*10 empty buckets in total, otherwise the amount of
 * work it does would be unbound and the function may block for a long time. */
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    if (!dictIsRehashing(d)) return 0;

    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            uint64_t h;

            nextde = de->next;
            /* Get the index in the new hash table */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;											// rehash完成后，将rehashidx计数加1
    }

    /* Check if we already rehashed the whole table... */
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```

Q：为什么要设置单步rehash中最大空桶访问次数限制(`empty_visit`)？ 

A：考虑一个非常大的哈希表，比如有10亿个key, 其中连续1亿个空桶。如果不设置这个限制，会导致rehash中遍历空桶耗时太多，出现服务器阻塞。



2、通过定时任务中，周期性调用`dictRehashMilliseconds`，对字典执行主动rehash

```c
long long timeInMilliseconds(void) {
    struct timeval tv;

    gettimeofday(&tv,NULL);
    return (((long long)tv.tv_sec)*1000)+(tv.tv_usec/1000);
}

/* Rehash in ms+"delta" milliseconds. The value of "delta" is larger 
 * than 0, and is smaller than 1 in most cases. The exact upper bound 
 * depends on the running time of dictRehash(d,100).*/
int dictRehashMilliseconds(dict *d, int ms) {
    long long start = timeInMilliseconds();
    int rehashes = 0;

    while(dictRehash(d,100)) {
        rehashes += 100;
        if (timeInMilliseconds()-start > ms) break;
    }
    return rehashes;
}
```

实现细节分析：

* 通过C语言库函数`gettimeofday`计算rehash前后的耗时，好处是跨平台，且时间精度可以达到微秒级。

* 通过循环多次执行一个短任务，每次比较实际用时与规定用时的方式，近似地实现规定时间内执行某个任务的目的。这种实现技巧简化了代码逻辑，值得学习借鉴。



#### 增、删、改、查场景中的rehash实现<span id="jump19"></span>

在rehash过程中，字典会同时使用ht[0], ht[1]两个哈希表，此时字典的增、删、改、查操作也会同时在两个字典中进行。

* 查 —— 依次在ht[0], ht[1]中查找。

* 增 —— 新增的键值对总是只加入到ht[1]，目的是保证ht[0]中键值对只减不增，加速rehash的执行。
* 删 —— 依次在ht[0], ht[1]中查找节点，并删除。

**源码分析：**

1、`dictAdd` 将给定键值对添加到字典，时间复杂度O(1)

```c
int dictAdd(dict *d, void *key, void *val) {
    dictEntry *entry = dictAddRaw(d,key,NULL);

    if (!entry) return DICT_ERR;
    dictSetVal(d, entry, val);
    return DICT_OK;
}

dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing) {
    long index;
    dictEntry *entry;
    dictht *ht;

    if (dictIsRehashing(d)) _dictRehashStep(d);	// 执行一次rehash
	
    // 计算哈希索引值
    if ((index = _dictKeyIndex(d, key, dictHashKey(d,key), existing)) == -1)
        return NULL;

    /* Allocate the memory and store the new entry.
     * Insert the element in top, with the assumption that in a database
     * system it is more likely that recently added entries are accessed
     * more frequently. 
     */
    ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0]; // rehash中，新增的键值对加入ht[1]
    entry = zmalloc(sizeof(*entry));
    entry->next = ht->table[index];	 // 头插法，理由是最先插入的节点被频繁访问的可能性越大
    ht->table[index] = entry;
    ht->used++;

    /* Set the hash entry fields. */
    dictSetKey(d, entry, key);
    return entry;
}
```

分析：

* 每执行一次字典的插入操作，都会调用一次`_dictRehashStep`，做一次rehash
* rehash过程中，新增的键值对总是加入到ht[1]，`ht = dictIsRehashing(d) ? &d->ht[1] : &d->ht[0];`

* 使用头插法，理由是最先插入的节点后续被访问的可能性越大，而且实现也非常简单，节省内存。



2、`dictDelete`从字典中删除给定键的键值对，时间复杂度O(1)

```c
int dictDelete(dict *ht, const void *key) {
    return dictGenericDelete(ht,key,0) ? DICT_OK : DICT_ERR;
}

static dictEntry *dictGenericDelete(dict *d, const void *key, int nofree) {
    uint64_t h, idx;
    dictEntry *he, *prevHe;
    int table;

    if (d->ht[0].used == 0 && d->ht[1].used == 0) return NULL;	// 表空不能删除，直接返回

    if (dictIsRehashing(d)) _dictRehashStep(d);	// 执行一次rehash
    h = dictHashKey(d, key);	// 计算哈希值

    for (table = 0; table <= 1; table++) {	// 删除操作同时使用ht[0], ht[1]两张表
        idx = h & d->ht[table].sizemask;	// 计算索引值
        he = d->ht[table].table[idx];
        prevHe = NULL;
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key)) {
                if (prevHe)
                    prevHe->next = he->next;
                else
                    d->ht[table].table[idx] = he->next;
                if (!nofree) {
                    dictFreeKey(d, he);
                    dictFreeVal(d, he);
                    zfree(he);
                }
                d->ht[table].used--;
                return he;
            }
            prevHe = he;
            he = he->next;
        }
        if (!dictIsRehashing(d)) break;
    }
    return NULL; /* not found */
}
```

分析：

* 每执行一次字典的删除操作，都会调用一次`_dictRehashStep`，做一次rehash
* 删除操作同时使用ht[0], ht[1]两张表



3、`dictFetchValue`用于查找字典，返回给定键的值，时间复杂度O(1)

```c
void *dictFetchValue(dict *d, const void *key) {
    dictEntry *he;
    he = dictFind(d,key);
    return he ? dictGetVal(he) : NULL;
}

dictEntry *dictFind(dict *d, const void *key) {
    dictEntry *he;
    uint64_t h, idx, table;

    if (dictSize(d) == 0) return NULL;
    if (dictIsRehashing(d)) _dictRehashStep(d); // 执行一次rehash
    h = dictHashKey(d, key);
    for (table = 0; table <= 1; table++) { 		// 查询操作同时使用ht[0], ht[1]两张表
        idx = h & d->ht[table].sizemask;
        he = d->ht[table].table[idx];
        while(he) {
            if (key==he->key || dictCompareKeys(d, key, he->key))
                return he;
            he = he->next;
        }
        if (!dictIsRehashing(d)) return NULL;
    }
    return NULL;
}
```

分析：

* 每执行一次字典的查询操作，都会调用一次`_dictRehashStep`，做一次rehash
* 查询操作同时使用ht[0], ht[1]两张表

### 字典API<span id="jump20"></span>

| API            | 功能                       |
| -------------- | -------------------------- |
| dictCreate     | 创建字典                   |
| dictRelease    | 释放字典                   |
| dictAdd        | 新增一个键值对到字典       |
| dictFetchValue | 查找字典，返回给定键的值   |
| dictDelete     | 从字典中删除给定键的键值对 |

#### 随机返回一个键值对<span id="jump21"></span>

源码参考`dictGetRandomKey`

```c
dictEntry *dictGetRandomKey(dict *d) {
    dictEntry *he, *orighe;
    unsigned long h;
    int listlen, listele;

    if (dictSize(d) == 0) return NULL;
    if (dictIsRehashing(d)) _dictRehashStep(d);
    if (dictIsRehashing(d)) {
        do {
            /* We are sure there are no elements in indexes from 0
             * to rehashidx-1 */
            h = d->rehashidx + (random() % (d->ht[0].size +
                                            d->ht[1].size -
                                            d->rehashidx));
            he = (h >= d->ht[0].size) ? d->ht[1].table[h - d->ht[0].size] :
                                      d->ht[0].table[h];
        } while(he == NULL);
    } else {
        do {
            h = random() & d->ht[0].sizemask;
            he = d->ht[0].table[h];
        } while(he == NULL);
    }

    /* Now we found a non empty bucket, but it is a linked
     * list and we need to get a random element from the list.
     * The only sane way to do so is counting the elements and
     * select a random index. */
    listlen = 0;
    orighe = he;
    while(he) {
        he = he->next;
        listlen++;
    }
    listele = random() % listlen;
    he = orighe;
    while(listele--) he = he->next;
    return he;
}
```

分析：

* 实现思路是：先随机选取一个非空的桶，再随机选取链表中的一个节点。

* 理想情况下，哈希表中的每个链表的长度为1，所以这里用O(N)复杂度随机获取链表节点是完全可以接受的，不记录链表长度目的是为了节省内存。



#### 字典迭代器的实现<span id="jump22"></span>

##### 1、先看下字典迭代器的结构体定义

```c
/* If safe is set to 1 this is a safe iterator, that means, you can call
 * dictAdd, dictFind, and other functions against the dictionary even while
 * iterating. Otherwise it is a non safe iterator, and only dictNext()
 * should be called while iterating. */
typedef struct dictIterator {
    dict *d;
    long index;						// 当前遍历到的哈希表中的索引值
    int table;						// 取值只能是0或1
    int safe;						// 表示这个迭代器是否是安全的，1表示安全，0表示不安全
    dictEntry *entry, *nextEntry;	// 迭代器指向的当前元素，以及下一个要遍历的元素
    /* unsafe iterator fingerprint for misuse detection. */
    long long fingerprint;			// 字典的指纹
} dictIterator;
```

1、safe成员为1，表示这个迭代器是安全的。安全指的是遍历过程中允许对字典做修改操作，且迭代中不会出现元素重复。所以当字典绑定了安全迭代器时，Redis不允许出现rehash操作，源码参考：

```c
static void _dictRehashStep(dict *d) {
    if (d->iterators == 0) dictRehash(d,1);	// 只有未绑定安全迭代器时，才执行dictRehash
}
```

safe成员为0，表示这个迭代器是不安全的。不安全指的是遍历过程中不允许修改这个字典，且遍历过程中可能出现重复元素。它的优点在于允许rehash。

2、fingerprint为字典的指纹，在首次迭代时生成指纹，销毁迭代器时比较指纹，如果两者不一致，说明出现异常，程序退出。源码参考：

```c
void dictReleaseIterator(dictIterator *iter) {
    if (!(iter->index == -1 && iter->table == 0)) {
        if (iter->safe)
            iter->d->iterators--;		// 计数减1，允许rehash操作
        else
            assert(iter->fingerprint == dictFingerprint(iter->d));	// 比较指纹
    }
    zfree(iter);
}
```

指纹的生成方法参考`dictFingerprint`

```c
long long dictFingerprint(dict *d) {
    long long integers[6], hash = 0;
    int j;

    integers[0] = (long) d->ht[0].table;
    integers[1] = d->ht[0].size;
    integers[2] = d->ht[0].used;
    integers[3] = (long) d->ht[1].table;
    integers[4] = d->ht[1].size;
    integers[5] = d->ht[1].used;

    /* We hash N integers by summing every successive integer with the integer
     * hashing of the previous sum. Basically:
     *
     * Result = hash(hash(hash(int1)+int2)+int3) ...
     *
     * This way the same set of integers in a different order will (likely) hash
     * to a different number. */
    for (j = 0; j < 6; j++) {
        hash += integers[j];
        /* For the hashing step we use Tomas Wang's 64 bit integer hash. */
        hash = (~hash) + (hash << 21); // hash = (hash << 21) - hash - 1;
        hash = hash ^ (hash >> 24);
        hash = (hash + (hash << 3)) + (hash << 8); // hash * 265
        hash = hash ^ (hash >> 14);
        hash = (hash + (hash << 2)) + (hash << 4); // hash * 21
        hash = hash ^ (hash >> 28);
        hash = hash + (hash << 31);
    }
    return hash;
}
```



##### 2、迭代器初始化

```c
dictIterator *dictGetIterator(dict *d) {
    dictIterator *iter = zmalloc(sizeof(*iter));
    iter->d = d;
    iter->table = 0;
    iter->index = -1;
    iter->safe = 0;
    iter->entry = NULL;
    iter->nextEntry = NULL;
    return iter;
}

dictIterator *dictGetSafeIterator(dict *d) {
    dictIterator *i = dictGetIterator(d);

    i->safe = 1;	// 安全迭代器和非安全迭代器初始化的区别就是safe成员不同而已！
    return i;
}
```

可以看出，安全迭代器和非安全迭代器初始化的区别就是safe成员不同而已。



##### 3、遍历下一个节点 

```c
dictEntry *dictNext(dictIterator *iter) {
    while (1) {
        if (iter->entry == NULL) {		// 如果为空，遍历下一个索引
            dictht *ht = &iter->d->ht[iter->table];
            if (iter->index == -1 && iter->table == 0) {
                if (iter->safe)
                    iter->d->iterators++;
                else
                    iter->fingerprint = dictFingerprint(iter->d);	// 如果是非安全迭代器，计算指纹
            }
            iter->index++;
            if (iter->index >= (long) ht->size) {	// 如果当前哈希表是ht[0]且遍历完成，尝试遍历ht[1]
                if (dictIsRehashing(iter->d) && iter->table == 0) {
                    iter->table++;
                    iter->index = 0;
                    ht = &iter->d->ht[1];
                } else {
                    break;
                }
            }
            iter->entry = ht->table[iter->index];
        } else {	// 否则，遍历当前链表的下一个元素
            iter->entry = iter->nextEntry;
        }
        if (iter->entry) {
            /* We need to save the 'next' here, the iterator user
             * may delete the entry we are returning. */
            iter->nextEntry = iter->entry->next;
            return iter->entry;
        }
    }
    return NULL;
}
```

简要概括，这个遍历操作就是从ht[0]到ht[1]，再沿着哈希表数组往下遍历。



#### dictScan算法<span id="jump23"></span>

dictScan算法用于遍历字典，是Redis字典源码中最为精妙，同时也是最难理解的算法。

**字典遍历算法的复杂性分析**

Redis字典涉及Rehash操作，如果在两次遍历键之间，字典发生了扩展或收缩，就会导致哈希键的索引值变化。字典遍历算法实现的难点在于<font color = 'red'>**如何保证字典中原有元素都能被遍历到，且迭代到的重复元素尽可能地少**</font>

Redis实现的dictScan算法特点如下：

* 开始遍历时的所有元素，只要不被删除都能被返回

* rehash过程中，算法可能返回重复元素，遍历过程中新增或删除的key可能返回，可能不返回。

**具体实现分析：**

先挖个坑，后续再填。。。。。。

网上已有非常深入的分析，推荐这篇：[美团针对Redis Rehash机制的探索和实践](https://tech.meituan.com/2018/07/27/redis-rehash-practice-optimization.html)



## 参考资料

【1】《Redis设计与实现》 第4章 字典

【2】[Copy On Write机制了解一下](https://juejin.cn/post/6844903702373859335)

【3】[美团针对Redis Rehash机制的探索和实践](https://tech.meituan.com/2018/07/27/redis-rehash-practice-optimization.html)

【4】[解决哈希冲突的常用方法分析](https://cloud.tencent.com/developer/article/1672781)
