---
layout: next
title: Redis学习笔记(一)——简单动态字符串
date: 2021-12-11 20:28:06
categories: Redis
tags: Redis
---

## 前言

Redis是用C语言开发的，但并没有直接使用C语言数组去表示字符串，而是使用简单动态字符串(Simple dynamic String，简称SDS)作为字符串的底层实现。

以下给出SDS相关的一些常见问题，通过源码分析和实际验证，思考这些问题的答案，了解实现原理和设计思路。

<!-- more -->

**源码版本：**

[Redis 3.0.0](https://github.com/redis/redis/releases/tag/3.0.0)

[Redis 6.0.10](https://github.com/redis/redis/releases/tag/6.0.10)



## 思考问题<span id="jump0"></span>

* [SDS的数据结构，字符串如何表示的](#jump1)
  * [SDS结构体各成员的作用](#jump2)
  * [创建给定C字符串的SDS场景，sdshdr结构体各成员初值是多少](#jump3)
* [SDS相较于C风格字符串的优点](#jump4)
  * [SDS的空间分配策略是如何杜绝缓冲区溢出问题的](#jump5)
  * [SDS是如何减少修改字符串时带来的内存重分配次数](#jump6)
    * [字符串增长场景，SDS扩容策略](#jump7)
    * [字符串缩短场景，SDS空间释放策略](#jump8)
* [SDS最大长度是多少](#jump9)
* [字符串键的三种编码方式](#jump10)
  * [int编码](#jump11)
  * [embstr编码](#jump12)
  * [raw编码](#jump13)
  * [设计embstr编码的用意是什么](#jump14)
  * [三种编码之间的转换规则](#jump15)
* [对于很短的字符串，并不需要4字节表示长度，REDIS 3.2中的SDS实现是如何优化，节约内存的](#jump16)
  * [给定一个长度为n的sds，它的底层通过哪个sdshdr类型表示？](#jump17)
* [REDIS字符串命令](#jump18)



## 源码分析

### SDS的数据结构，字符串是如何表示的 <span id="jump1"></span>
REDIS 3.0中，SDS的数据结构：

```c
struct sdshdr {
	int len;		// 字符串长度，即对C风格字符串调用strlen的结果
	int free;		// buf数组中的未使用字节数
	char buf[];		// '\0'结尾
};
```

注意sizeof(struct sdshdr)的结果等于8，不是9；这里的buf为柔性数组成员。至于为什么用柔性数组成员可以参考这篇文章：[https://www.cnblogs.com/davygeek/p/5748852.html](https://www.cnblogs.com/davygeek/p/5748852.html)

#### SDS结构体各成员的作用<span id="jump2"></span>

* len: 表示buf数组中已使用的字节数(**不包括'\0'**)，即字符串长度。好处是获取字符串长度时间为O(1)
* free: 表示buf数组中未使用的字节数。
* buf: 用于保存字符串，遵循C风格字符串原则，以'\0'结尾。

#### 创建给定C字符串的SDS场景，sdshdr结构体各成员初值是多少？<span id="jump3"></span>

**思考问题**：写入一个字符串键key, 值为长度为5的字符串"Redis"，它的SDS表示中len, free, buf成员值各是多少？

**源码分析：**

1、Redis调用`sdsnew`创建一个包含给定C字符串的SDS，这里的initlen = strlen("Redis") = 5。

```c
sds sdsnew(const char *init) {
    size_t initlen = (init == NULL) ? 0 : strlen(init);	// 调用strlen获取长度
    return sdsnewlen(init, initlen);
}
```

2、再调用`sdsnewlen`，返回SDS。

```c
sds sdsnewlen(const void *init, size_t initlen) {
    struct sdshdr *sh;

    // sds遵循C风格字符串原则，以'\0'结尾，额外1字节不计数len
    if (init) {
        sh = zmalloc(sizeof(struct sdshdr)+initlen+1);
    } else {
        sh = zcalloc(sizeof(struct sdshdr)+initlen+1);
    }
    if (sh == NULL) return NULL;
    sh->len = initlen;			// len = strlen(init)
    sh->free = 0;				// 创建一个包含C字符串SDS场景，free = 0
    if (initlen && init)
        memcpy(sh->buf, init, initlen);
    sh->buf[initlen] = '\0';	// sds遵循C风格字符串原则，以'\0'结尾，额外1字节不计入len
    return (char*)sh->buf;
}
```

可以看出，创建给定C字符串的SDS场景：

* len的值为strlen("Redis") = 5
* free的值为0

* buf的内容为 "Redis", 以'\0'结尾，6字节大小

SDS实例图：
![](image1.png)

**思考问题：** 初始化SDS时free为啥给0？—— 平时使用字符串还是只读场景偏多，这样能节约空间。

**GDB验证：**

Redis是基于内存的数据库，可以通过GDB打印内存，查看这个key对应的SDS结构体，是否和源码分析结果一致：

```
(gdb) p server.db[0].dict.ht[0]
$1 = {table = 0x7f3008d40900, size = 4, sizemask = 3, used = 1}

(gdb) p *(struct redisObject *) (server.db[0].dict.ht[0].table[0].v)
$2 = {type = 0, encoding = 8, lru = 11494144, refcount = 1, ptr = 0x7f3008d40838}
# type = REDIS_STRING(0), encoding = REDIS_ENCODING_EMBSTR(8)

(gdb) p  (*(robj *)server.db[0].dict.ht[0].table[0].v).ptr
$3 = (void *) 0x7f3008d40838

(gdb) p  (sds) 0x7f3008d40838
$4 = (sds) 0x7f3008d40838 "redis"		# 存储字符串键的值

(gdb) p *(struct sdshdr *)(0x7f3008d40838 - 0x8)			# len 4字节， free 4字节，减去8字节正好是struct sdshdr的首地址
$5 = {len = 5, free = 0, buf = 0x7f3008d40838 "Redis"}  	# 和以上源码分析的SDS结构体内容一致
```

**上述GDB调试操作的依据说明：**

* Redis将所有数据库保存在服务器状态`server`变量中，默认创建16个数据库，默认目标数据库为0号(db[0])

* dict为数据库键空间，保存所有键值对，底层实现为哈希表，其中ht[0]存储key-value，ht[1]用于rehash

* table类型为`dictEntry **`, 链地址法实现哈希表，table[0]不为NULL，说明这个key的hashcode % size的结果为0

* ptr类型为void *。对于字符串键而言，ptr实际类型为char *，存储内容为"Redis"串，而不是struct sdshdr的首地址。

**注：**Redis并没有直接使用sds, list这些基本数据结构去实现数据库，而是在这些基本数据结构上构筑了一个**对象系统**，统一使用redisObject对象：

```c
typedef struct redisObject {
    unsigned type:4;			    // 类型
    unsigned encoding:4;			// 编码
    unsigned lru:24;				// 记录对象最后一次被命令程序访问的时间
    int refcount;					// 引用计数
    void *ptr;						// 指向底层实现数据结构的指针
} robj;
```

* ptr为void *泛型指针，指向底层实现的数据结构。void *是C语言实现泛型编程的常用手段。
* type为对象类型。type属性设计目的很简单，因为仅通过ptr这个泛型指针无法获取这个对象真正的类型。对于字符串键，type值取0（REDIS_STRING）

* encoding为编码类型。encoding属性设计目的在于，根据不同场景，为对象设置不同的底层数据结构实现来优化性能。此例中，encoding值取8(REDIS_ENCODING_EMBSTR)，表示编码方式为embstr。



### SDS相较于C风格字符串的优点<span id="jump4"></span>

* 获取字符串长度的时间复杂度为O(1) （对于SDS来说，获取长度只需访问len成员）

* 杜绝缓冲区溢出问题

* 减少字符串修改时导致的内存重分配次数

* 二进制安全，除了能保存文本数据，还可以保存二进制数据

#### SDS的空间分配策略是如何杜绝缓冲区溢出问题的？<span id="jump5"></span>

以SDS拼接函数`sdscat`为例，源码如下：

```c
sds sdscat(sds s, const char *t) {
    return sdscatlen(s, t, strlen(t));
}

sds sdscatlen(sds s, const void *t, size_t len) {
    struct sdshdr *sh;
    size_t curlen = sdslen(s);		// 得到源字符串长度

    s = sdsMakeRoomFor(s,len);		// 拼接前检查s的剩余空间是否足够，如空间不足需先扩展空间
    if (s == NULL) return NULL;
    sh = (void*) (s-(sizeof(struct sdshdr)));
    memcpy(s+curlen, t, len);
    sh->len = curlen+len;
    sh->free = sh->free-len;
    s[curlen+len] = '\0';
    return s;
}
```

可以看出，`sdscat`拼接字符串前，会先通过`sdsMakeRoomFor`检查s的剩余空间是否足够，**如果空间不足，会先调用`realloc`扩展出足够空间后**，再通过`memcpy`拼接字符串。 所以杜绝了缓冲区溢出问题。

#### SDS是如何减少修改字符串时带来的内存重分配次数<span id="jump6"></span>

对于C字符串，每次增长或缩短操作，都会导致一次内存重分配，性能较差。

SDS中引入free属性，通过未使用空间，优化字符串的增长或缩短操作，减少内存重分配次数。

#### 字符串增长场景，SDS扩容策略<span id="jump7"></span>

对于字符串增长场景，REDIS采用空间预分配的思想，即不仅分配修改后的SDS必需的空间，**还会额外分配一定的未使用空间**。源码参考`sdsMakeRoomFor`

```c
#define SDS_MAX_PREALLOC (1024 * 1024)
sds sdsMakeRoomFor(sds s, size_t addlen) {
    struct sdshdr *sh, *newsh;
    size_t free = sdsavail(s);
    size_t len, newlen;
	if (free >= addlen) return s;			// 如果剩余空间足够直接返回
	len = sdslen(s);
	sh = (void*) (s-(sizeof(struct sdshdr)));
	newlen = (len+addlen);
	if (newlen < SDS_MAX_PREALLOC)			// 如修改后的SDS长度小于1M,realloc重新分配两倍空间
   		newlen *= 2;
	else
    	newlen += SDS_MAX_PREALLOC;			// 如修改后的SDS长度大于等于1M, realloc重新分配1M的空间
	newsh = zrealloc(sh, sizeof(struct sdshdr)+newlen+1);
	if (newsh == NULL) return NULL;

	newsh->free = newlen - len;
	return newsh->buf;
}
```

可以看出，额外分配的未使用空间大小由修改后的SDS长度决定：

* 如果对SDS修改后，SDS长度(即len属性的值)小于1MB，REDIS会额外分配len个字节的空间。例如，给定SDS串s("hello")，调用sdscat(s, "world")之后，len = 10，free = 10, buf数组大小变为 10 + 10 + 1 = 21字节。
* 如果对SDS修改后，SDS长度大于等于1MB， REDIS只额外分配1MB的空间，目的是避免内存出现太大的浪费。

相比C风格字符串，**SDS的扩容策略将增长N次字符串需要的内存重分配次数从N次降低为最多N次**。

GDB验证结果如下，和分析源码得出的结论一致：

```
127.0.0.1:6379> set key hello
[OOK
(gdb) p *(struct sdshdr *)((*(robj *)server.db[0].dict.ht[0].table[3].v).ptr - 0x8)
$1 = {len = 5, free = 0, buf = 0x7f571f1407f8 "hello"}

127.0.0.1:6379> APPEND key world
(integer) 10
(gdb) p *(struct sdshdr *)((*(robj *)server.db[0].dict.ht[0].table[3].v).ptr - 0x8)
$2 = {len = 10, free = 10, buf = 0x7f571f1407e8 "helloworld"}
```

#### 字符串缩短场景，SDS空间释放策略<span id="jump8"></span>

对于字符串缩短场景，REDIS采用惰性空间释放策略，即并不立即回收空闲内存，而是仅使用free属性记录空闲字节数，如果将来需对SDS做增长操作，可以直接使用这部分空闲内存，无需做内存重分配。

源码分析：`sdsclear`用于清空SDS保存的字符串内容，采用惰性空闲释放策略，复杂度仅为O(1)

```c
void sdsclear(sds s) {
    struct sdshdr *sh = (void*) (s-(sizeof(struct sdshdr)));
    sh->free += sh->len;
    sh->len = 0;
    sh->buf[0] = '\0';
}
```

同时，REDIS提供API `sdsRemoveFreeSpace`，通过realloc仅分配实际大小的内存，真正地回收空闲内存，解决惰性空间释放策略带来的内存浪费问题。

```c
sds sdsRemoveFreeSpace(sds s) {
    struct sdshdr *sh;
    sh = (void*) (s-(sizeof(struct sdshdr)));
    sh = zrealloc(sh, sizeof(struct sdshdr)+sh->len+1);	// 仅分配实际大小的内存
    sh->free = 0;										// free写0
    return sh->buf;
}
```

### SDS最大长度是多少？<span id="jump9"></span>

Redis 3.0

* struct sdshdr的len成员记录SDS长度，类型为int，4字节，理论上最大长度 2^32 / 2^10 / 2^10 = 4096MB

* 在set, append操作中通过硬编码写死字符串的最大长度为512MB，超过这个长度会报错，源码参考`checkStringLength`。

```c
static int checkStringLength(redisClient *c, long long size) {
    if (size > 512*1024*1024) {
        addReplyError(c,"string exceeds maximum allowed size (512MB)");
        return REDIS_ERR;
    }
    return REDIS_OK;
}
```

综上， Redis 3.0中SDS最大长度为 512MB。

Redis 6.0.10

* 通过配置项`proto-max-bulk-len`指定SDS长度，默认是512MB，用户可以自行配置这个值，这点和Redis 3.0有区别。

```c
static int checkStringLength(client *c, long long size) {
    if (!(c->flags & CLIENT_MASTER) && size > server.proto_max_bulk_len) {
        addReplyError(c,"string exceeds maximum allowed size (proto-max-bulk-len)");
        return C_ERR;
    }
    return C_OK;
}
```

综上， Redis 6.0中SDS最大长度默认为512MB，用户可以自行配置这个值。

### 字符串键的三种编码方式<span id="jump10"></span>

Redis中字符串对象有三种编码，分别是int , embstr, raw。以下分别介绍这三种编码：

#### int编码<span id="jump11"></span>

如果一个字符串对象保存的内容是整数值，且这个整数可以用long表示，Redis就把它的编码设置为int

举例：执行 set key "123"命令，会创建一个int编码的字符串对象

```
127.0.0.1:6379> set key 123
OK
127.0.0.1:6379> object encoding key
"int"
```

内存中的`redisObject`对象内容如下：

![](image2.png)

int类型编码的字符串，要求整数落在long的范围内。在64位环境上，long范围：-9223372036854775808～+9223372036854775807，不同的键值和编码结果参考下表：

| 键值                 | 编码   |
| :-------------------- | :------ |
| +123                 | embstr |
| -123                 | int    |
| --123                | embstr |
| -9223372036854775808 | int    |
| -9223372036854775809 | embstr |
| 9223372036854775807  | int    |
| 9223372036854775808  | embstr |

**源码分析：**

`createStringObjectFromLongLong`根据传入的long long类型的整数值，创建一个字符串对象。如果入参在long范围之内，就创建int编码的字符串对象，源码如下：

```c
#define REDIS_SHARED_INTEGERS 10000
robj *createStringObjectFromLongLong(long long value) {
    robj *o;
    // value 的大小在 0 - 10000之间，直接返回一个共享对象
    if (value >= 0 && value < REDIS_SHARED_INTEGERS) {
        incrRefCount(shared.integers[value]);
        o = shared.integers[value];

    } else {
		// 如果在long范围内，就创建编码为int的字符串
        if (value >= LONG_MIN && value <= LONG_MAX) {
            o = createObject(REDIS_STRING, NULL);
            o->encoding = REDIS_ENCODING_INT;
            o->ptr = (void*)((long)value);	// ptr实际指向一个long类型的value

        // 如果在long之外，就创建一个编码为embstr的字符串
        } else {
            o = createObject(REDIS_STRING,sdsfromlonglong(value));
        }
    }
    return o;
}
```

#### REDIS中的对象共享机制

Redis服务器初始化的时候，会预先创建1万个字符串对象 (0 ~ 9999), 当服务器需要使用值为 0 - 9999的字符串对象时，服务器会直接使用这些共享对象，而不是去新创建一个对象。这个用意在于节约内存。

**举例：**

创建字符串键A，B，值都写“1”, 那么这两个键共享同一个redisObject对象，且这个redisObject对象的ptr指向的内容为1

**GDB验证结果：**

```
(gdb)  p *(robj *)server.db[0].dict.ht[0].table[0].v
$15 = {type = 0, encoding = 1, lru = 11668863, refcount = 3, ptr = 0x1}
(gdb)  p *(robj *)server.db[0].dict.ht[0].table[2].v
$16 = {type = 0, encoding = 1, lru = 11668863, refcount = 3, ptr = 0x1}
(gdb)  p &(*(robj *)server.db[0].dict.ht[0].table[0].v)
$17 = (robj *) 0x7fa7c1457360
(gdb)  p &(*(robj *)server.db[0].dict.ht[0].table[2].v)
$18 = (robj *) 0x7fa7c1457360		# 和$17相同，都是0x7fa7c1457360，说明这是一个共享对象
```

此时，两个key和共享字符串对象的内存示意图：

![](image3.png)


#### raw编码<span id="jump13"></span>

如果字符串对象保存的是一个字符串值，且这个字符串长度大于39个字节，REDIS就使用SDS存储这个字符串，并设置编码类型为raw。

**举例：**

```
127.0.0.1:6379> set key 1234567891234567891234567891234567891234
OK
127.0.0.1:6379> object encoding ket
"raw"
127.0.0.1:6379> strlen key
(integer) 40
```

raw编码的字符串示意图：

![](image4.png)

**源码分析：**

`createStringObject`用于创建一个SDS表示的字符串对象。当字符串长度大于39字节时使用raw编码， 否则用embstr编码，源码如下：

```c
/* The current limit of 39 is chosen so that the biggest string object
 * we allocate as EMBSTR will still fit into the 64 byte arena of jemalloc. */
#define REDIS_ENCODING_EMBSTR_SIZE_LIMIT 39
robj *createStringObject(char *ptr, size_t len) {
    if (len <= REDIS_ENCODING_EMBSTR_SIZE_LIMIT)
        return createEmbeddedStringObject(ptr,len);
    else
        return createRawStringObject(ptr,len);
}

robj *createRawStringObject(char *ptr, size_t len) {
    return createObject(REDIS_STRING,sdsnewlen(ptr,len));
}

robj *createObject(int type, void *ptr) {
    robj *o = zmalloc(sizeof(*o));
    o->type = type;
    o->encoding = REDIS_ENCODING_RAW;	// 设置编码类型为raw
    o->ptr = ptr;
    o->refcount = 1;
    o->lru = LRU_CLOCK();
    return o;
}
```



#### embstr编码<span id="jump12"></span>

如果字符串对象保存的是一个字符串值，且这个字符串长度小于等于39个字节，REDIS就使用SDS存储这个字符串，并设置编码类型为embstr，**embstr是专门用于保存短字符串的一种优化编码方式。**

**举例：**

```
127.0.0.1:6379> set key hello
OK
127.0.0.1:6379> object encoding key
"embstr"
```

和raw编码类似，embstr编码也使用`redisObject`和`sdshdr`保存字符串，但差别在于：

* raw编码需调用**2**次malloc创建`redisObject`和`sdshdr`对象，且`redisObject`和`sdshdr`内存不连续
* 而embstr编码只需**1**次malloc创建`redisObject`和`sdshdr`对象，且`redisObject`和`sdshdr`内存是连续的

GDB查看embstr编码字符串的内存：

```
(gdb) p &(*(robj *)server.db[0].dict.ht[0].table[3].v)
$1 = (robj *) 0x7f43ebd3b9c0
(gdb) p *(robj *)server.db[0].dict.ht[0].table[3].v
$2 = {type = 0, encoding = 8, lru = 11755503, refcount = 1, ptr = 0x7f43ebd3b9d8}
(gdb) p *(struct sdshdr *)0x7f43ebd3b9d0
$3 = {len = 5, free = 0, buf = 0x7f43ebd3b9d8 "hello"}
```

观察结果发现robj和sdshdr内存确实是连续的，embstr编码的内存示意图：

![](image5.png)


**源码分析：**

`createEmbeddedStringObject`用于创建一个embstr编码的字符串

```c
robj *createEmbeddedStringObject(char *ptr, size_t len) {
    robj *o = zmalloc(sizeof(robj)+sizeof(struct sdshdr)+len+1); // 仅1次malloc
    struct sdshdr *sh = (void*)(o+1);

    o->type = REDIS_STRING;
    o->encoding = REDIS_ENCODING_EMBSTR;
    o->ptr = sh+1;
    o->refcount = 1;
    o->lru = LRU_CLOCK();

    sh->len = len;
    sh->free = 0;
    if (ptr) {
        memcpy(sh->buf,ptr,len);
        sh->buf[len] = '\0';
    } else {
        memset(sh->buf,0,len+1);
    }
    return o;
}
```



#### 设计embstr编码的用意是什么<span id="jump14"></span>

相比于raw编码，embstr编码存储短字符串的优点：

* 创建字符串对象时，malloc次数从2次变为1次，释放字符串对象时，free次数从2次变成1次。

* embstr编码中，`redisObject`和`sdshdr`内存连续，可以更好利用缓存，提升效率。

#### 三种编码之间的转换规则<span id="jump15"></span>

**规则1：** embstr对象执行修改命令后，总是会变成一个raw编码对象。

**规则2：** 对于int对象，如果在这个对象执行的操作导致其保存的值不在long范围内，这个对象编码总是变成raw

**源码分析：**

以APPEND命令为例，源码参考`appendCommand`，此函数最终调用`dbUnshareStringValue`，总是创建一个raw编码的对象。

```c
void appendCommand(redisClient *c) {
    size_t totlen;
    robj *o, *append;
    o = lookupKeyWrite(c->db,c->argv[1]);
    if (o == NULL) {
        // 键值对不存在就创建一个新的 ......
    } else {
        // 键值对存在 ......
        /* "append" is an argument, so always an sds */
        append = c->argv[2];
        totlen = stringObjectLen(o)+sdslen(append->ptr);
        if (checkStringLength(c,totlen) != REDIS_OK)
            return;
        /* Append the value */
        o = dbUnshareStringValue(c->db,c->argv[1],o);
        o->ptr = sdscatlen(o->ptr,append->ptr,sdslen(append->ptr));
        totlen = sdslen(o->ptr);
    }
	// ......
}

robj *dbUnshareStringValue(redisDb *db, robj *key, robj *o) {
    redisAssert(o->type == REDIS_STRING);
    if (o->refcount != 1 || o->encoding != REDIS_ENCODING_RAW) {
        robj *decoded = getDecodedObject(o);
        o = createRawStringObject(decoded->ptr, sdslen(decoded->ptr)); // embstr对象执行修改命令后，总是会变成一个raw编码对象。
        decrRefCount(decoded);
        dbOverwrite(db,key,o);
    }
    return o;
}
```

**验证结果：**

```
127.0.0.1:6379> set key "hello"
OK
127.0.0.1:6379> object encoding key
"embstr"
127.0.0.1:6379> append key " world!"
(integer) 12
127.0.0.1:6379> object encoding key
"raw"
```



#### REDIS3.2中的SDS实现<span id="jump16"></span>
REDIS 3.2中，根据SDS的长度又细分为5类，对于不同长度的字符串，用不同的sdshdrX结构体存储，实现节约内存的目的。

以REDIS 6.0.10源码为例，sdshdr结构体定义如下：

```c
typedef char *sds;

/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the he
    ader and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

**各结构体成员作用：**

* len表示字符串实际长度。
* alloc表示为sds分配的大小，不包括'\0'。

* flags表示sdshdr类型，用于判断sds的类型。flags本身是char类型有8位，其中高5位保留，只用低3位足以这表示5种sdshdr类型，参考源码：

```C
#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
#define SDS_TYPE_MASK 7
#define SDS_TYPE_BITS 3
```

**`__attribute__ ((__packed__))`作用？**

`__attribute__ ((__packed__))`是GCC特有的语法，作用是取消结构体的字节对齐，采用内存紧凑模式排列。

这里给出加上或不加关键字时，各sdshdr结构体的大小：

| 结构体   | 加GCC关键字`__attribute__ ((__packed__))` | 不加关键字 |
| :-------- | :----------------------------------------- | :---------- |
| sdshdr5  | 1字节                                     | 1          |
| sdshdr8  | 3                                         | 3          |
| sdshdr16 | 5                                         | 6          |
| sdshdr32 | 9                                         | 12         |
| sdshdr64 | 17                                        | 24         |

**问题思考**：取消结构体字节对齐的用意是什么，有什么优点？

* 1个好处是节约了内存，时间换空间。
* 另1个好处是使得**通过内存直接访问结构体内部变量非常方便**，比如通过buf[-1]这种骚操作可以直接访问到`flags`成员，从而判断sds类型，实现非常简洁。源码参考如下：

```c
sds s;
char type = s[-1] & SDS_TYPE_MASK;
```

#### 给定一个长度为n的sds，它的底层通过哪个sdshdr类型表示？<span id="jump17"></span>

以REDIS 6.0.10源码为例，参考`sdsReqType`的实现：

```c
static inline char sdsReqType(size_t string_size) {
    if (string_size < 32)
        return SDS_TYPE_5;
    if (string_size < 0xff)
        return SDS_TYPE_8;
    if (string_size < 0xffff)
        return SDS_TYPE_16;
    if (string_size < 0xffffffff)
        return SDS_TYPE_32;
    return SDS_TYPE_64;
}
```

得出如下结论：

* SDS长度小于32，用sdshdr5类型表示
* SDS长度在 [32, 255），用sdshdr8类型表示
* 依次类推 ......



**思考问题：**`set key hello`， 键key和值hello都是用sdshdr5表示的吗？

详细分析过程参考文章：[https://segmentfault.com/a/1190000017450295](https://segmentfault.com/a/1190000017450295)

以下仅给出结论和验证结果：

**结论：**

* **对于长度小于32的字符串键和值，键通过sdshdr5表示，而值通过sdshdr8表示**

* 对于值，使用`createEmbeddedStringObject`总是创建一个sdshdr8类型的对象。
* 对于键，通过调用链`setGenericCommand-->genericSetKey-->dbAdd`，最终调用`sdsdup`，创建一个sdshdr5类型的对象。调用栈参考：

```
(gdb) bt
#0  dbAdd (db=0x557b1c4d9690, key=0x557b1c4f5720, val=0x557b1c4f4730) at db.c:185
#1  0x0000557b1b95751d in genericSetKey (c=c@entry=0x557b1c4ecfa0, db=0x557b1c4d9690, key=key@entry=0x557b1c4f5720,
    val=val@entry=0x557b1c4f4730, keepttl=keepttl@entry=0, signal=signal@entry=1) at db.c:252
#2  0x0000557b1b9649e7 in setGenericCommand (c=c@entry=0x557b1c4ecfa0, flags=flags@entry=0, key=0x557b1c4f5720,
    val=0x557b1c4f4730, expire=expire@entry=0x0, unit=unit@entry=0, ok_reply=0x0, abort_reply=0x0) at t_string.c:87
#3  0x0000557b1b964c51 in setCommand (c=0x557b1c4ecfa0) at t_string.c:146
#4  0x0000557b1b93bd5e in call (c=0x557b1c4ecfa0, flags=15) at server.c:3368
#5  0x0000557b1b93c7a5 in processCommand (c=c@entry=0x557b1c4ecfa0) at server.c:3797
#6  0x0000557b1b94a7b0 in processCommandAndResetClient (c=c@entry=0x557b1c4ecfa0) at networking.c:1895
#7  0x0000557b1b94f09a in processInputBuffer (c=0x557b1c4ecfa0) at networking.c:1978
#8  0x0000557b1b9cbd48 in callHandler (handler=<optimized out>, conn=0x557b1c4fdd60) at connhelpers.h:79
#9  connSocketEventHandler (el=<optimized out>, fd=<optimized out>, clientData=0x557b1c4fdd60, mask=<optimized out>)
    at connection.c:296
#10 0x0000557b1b9356c7 in aeProcessEvents (eventLoop=eventLoop@entry=0x557b1c479aa0, flags=flags@entry=27) at ae.c:479
#11 0x0000557b1b935a0d in aeMain (eventLoop=0x557b1c479aa0) at ae.c:539
#12 0x0000557b1b932216 in main (argc=2, argv=0x7ffe8e366c18) at server.c:5498
```

**GDB验证结果：**

```
(gdb) x/tb ((*(robj *)server.db[0].dict.ht[0].table[1].v).ptr - 1) // value
0x557b1c4f4742: 00000001	// 注意value的最后三位为001，表示SDS_TYPE_8
$2 = {len = 5 '\005', alloc = 5 '\005', flags = 1 '\001', buf = 0x557b1c4f4743 "hello"}

(gdb) p (sds)server.db[0].dict.ht[0].table[1].key
$3 = (sds) 0x557b1c502f61 "key"
(gdb) x/tb 0x557b1c502f61 - 0x1
0x557b1c502f60: 00011000	// 注意key的最后三位为000，表示SDS_TYPE_5
```

**思考问题**：对于短字符串，为什么键底层类型为sdshdr5，值底层类型设置却成sdshdr8？

个人分析：实际应用场景中，通常键的更新次数远小于值的更新次数。所以对键采用最小的内存存储，以节省空间；对值用更大的内存存储，减少内存重分配的次数，提高性能。



#### REDIS字符串命令<span id="jump18"></span>

REDIS 字符串命令参考官方网站：[https://redis.io/commands#string](https://redis.io/commands#string)

以下仅给出几个最常用的命令:

* set key value
* get key
* append key value



## 参考资料

【1】《Redis设计与实现》第2章 简单动态字符串，第8章 对象

【2】[如何阅读Redis源码](https://blog.huangz.me/diary/2014/how-to-read-redis-source-code.html)

【3】[【Redis源码分析】一个对SDSHDR5是否使用的疑问](https://segmentfault.com/a/1190000017450295)

【4】[柔性数组](https://www.cnblogs.com/davygeek/p/5748852.html)
