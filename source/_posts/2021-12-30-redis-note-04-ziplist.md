---
layout: next
title: Redis学习笔记(四)——ziplist
date: 2021-12-30 20:45:24
categories: Redis
tags: Redis
---

## 背景

ziplist是一种为节约内存而开发的数据结构，本质是一个字节数组。

ziplist是列表键和哈希键的底层实现之一，也用于quicklist的实现。

<!-- more -->

## 问题思考

双向链表结构，在<font color = 'red'>存储数据本身长度远小于链表节点大小</font>的场景下，有严重的内存浪费问题。针对这种情况，Redis设计了ziplist这种节约内存的数据结构。以下给出ziplist相关的思考问题，了解ziplist的实现原理和设计思路。

* [ziplist的数据结构](#jump1)
* [ziplist节点构成](#jump2)
* [为什么要设计ziplist](#jump3)
* [ziplist相关操作](#jump4)
  * [查找指定节点](#jump5)
  * [连锁更新问题](#jump6)
* quicklist的数据结构
* 为什么要设计quicklist
* quicklist的增、删、改、查操作



### ziplist的数据结构<span id="jump1"></span>

Redis没有专门定义结构体来表示ziplist，因为ziplist本质就是一个空间连续的字节数组。

ziplist中包含多个节点(entry)，每个节点存储一个字符串值或整数值，每个节点通过`struct zlentry`结构表示。

ziplist的各组成部分参考如下：
![](image1.png)

可以看出，ziplist由列表头 + 列表节点 + 列表尾这三部分组成。每个组成部分作用说明如下：

* `zlbytes`占4字节，用于记录整个ziplist占用内存的总字节数，对于空表来说，`zlbytes`等于11，源码参考`ziplistNew`

```c
#define ZIPLIST_HEADER_SIZE     (sizeof(uint32_t)*2+sizeof(uint16_t)) // 8 + 2 = 10字节
#define ZIPLIST_END_SIZE        (sizeof(uint8_t)) // 1字节
#define ZIPLIST_BYTES(zl)       (*((uint32_t*)(zl)))  // zlbytes

/* Create a new empty ziplist. */
unsigned char *ziplistNew(void) {
    unsigned int bytes = ZIPLIST_HEADER_SIZE+ZIPLIST_END_SIZE;	// zlbytes = 10 + 1 = 11
    unsigned char *zl = zmalloc(bytes); // 可以看出，空表分配11字节大小空间
    ZIPLIST_BYTES(zl) = intrev32ifbe(bytes);
    ZIPLIST_TAIL_OFFSET(zl) = intrev32ifbe(ZIPLIST_HEADER_SIZE);
    ZIPLIST_LENGTH(zl) = 0;
    zl[bytes-1] = ZIP_END;
    return zl;
}
```

ziplist为空时的内存空间如下图：
![](image2.png)
* `zltail`占4字节，用于记录表尾节点首地址距离ziplist起始地址有多少字节。设计这个字段的目的是为了快速定位表尾节点地址（ZIPLIST_ENTRY_TAIL)。
* `zllen`占2字节，用于记录列表节点总数。注意`zllen`等于65535时，表示这个列表长度太大，必须通过遍历整个ziplist才能得到真实的长度。参考`ziplistLen`实现：

```c
#define UINT16_MAX 65535
unsigned int ziplistLen(unsigned char *zl) {
    unsigned int len = 0;
    if (intrev16ifbe(ZIPLIST_LENGTH(zl)) < UINT16_MAX) {
        len = intrev16ifbe(ZIPLIST_LENGTH(zl)); // zllen < 65535时，O(1)复杂度获取ziplist长度
    } else {
        unsigned char *p = zl+ZIPLIST_HEADER_SIZE;
        while (*p != ZIP_END) {			// zllen = 65535时，O(N)复杂度获取ziplist长度
            p += zipRawEntryLength(p);
            len++;
        }

        /* Re-store length if small enough */
        if (len < UINT16_MAX) ZIPLIST_LENGTH(zl) = intrev16ifbe(len);
    }
    return len;
}
```

* `zlend`占1字节，用于标记ziplist的结束，内容固定为0xFF(十进制的255)

### ziplist节点构成<span id="jump2"></span>

ziplist中包含多个节点(entry)，每个节点存储一个字符串值或整数值，每个节点通过struct zlentry结构表示。

```c
typedef struct zlentry {
    unsigned int prevrawlensize; /* Bytes used to encode the previous entry len*/
    unsigned int prevrawlen;     /* Previous entry len. */
    unsigned int lensize;        /* Bytes used to encode this entry type/len.
                                    For example strings have a 1, 2 or 5 bytes
                                    header. Integers always use a single byte.*/
    unsigned int len;            /* Bytes used to represent the actual entry.
                                    For strings this is just the string length
                                    while for integers it is 1, 2, 3, 4, 8 or
                                    0 (for 4 bit immediate) depending on the
                                    number range. */
    unsigned int headersize;     /* prevrawlensize + lensize. */
    unsigned char encoding;      /* Set to ZIP_STR_* or ZIP_INT_* depending on
                                    the entry encoding. However for 4 bits
                                    immediate integers this can assume a range
                                    of values and must be range-checked. */
    unsigned char *p;            /* Pointer to the very start of the entry, that
                                    is, this points to prev-entry-len field. */
} zlentry;
```

前面提到，ziplist本质是一个字节数组，Redis为了操作方便，才专门定义zlentry结构体，并解析某个entry的信息到zlentry中，**注意ziplist字节数组本身并不存储zlentry；** ziplist的每个entry结构都由三部分组成：

* 前一节点长度信息：`previous_entry_length`
* 当前节点编码信息：`encoding`
* 当前节点内容：`content`

ziplist节点的各组成部分示意图：
![](image3.png)

以下分别介绍这三个组成部分：

#### 1、前一节点长度信息 previous_entry_length

`previous_entry_length`本身占1字节或5字节，用于记录前一个节点的长度，单位为字节：

* 如果前一个节点长度小于254字节，`previous_entry_length`就只占1字节，直接保存前一个节点长度。
* 如果前一个节点长度大于等于254字节，`previous_entry_length`就占5字节，其中第一个字节固定为0xFE，后4个字节保存前一个节点长度。

源码参考`ZIP_DECODE_PREVLENSIZE`，这个宏根据前一个节点长度，返回`previous_entry_length`占用的字节数：

```c
#define ZIP_BIG_PREVLEN 254
#define ZIP_DECODE_PREVLENSIZE(ptr, prevlensize) do {                          \
    if ((ptr)[0] < ZIP_BIG_PREVLEN) {                                          \
        (prevlensize) = 1;                                                     \
    } else {                                                                   \
        (prevlensize) = 5;                                                     \
    }                                                                          \
} while(0)
```

Q：为什么要设计`previous_entry_length`字段，有什么作用？

A：用于支持从表尾向表头方向遍历。比如我们有指向某个节点起始地址的指针p，用p减去这个节点的`previous_entry_length`，就能得到前一节点的起始地址。

访问ziplist当前节点的前一个节点，参考源码`ziplistPrev`的实现：

```c
/* Return pointer to previous entry in ziplist. */
unsigned char *ziplistPrev(unsigned char *zl, unsigned char *p) {
    unsigned int prevlensize, prevlen = 0;
    if (p[0] == ZIP_END) {	// p指向ZIPLIST_ENTRY_END，前一个节点就是最后一个节点，即ZIPLIST_ENTRY_TAIL
        p = ZIPLIST_ENTRY_TAIL(zl);
        return (p[0] == ZIP_END) ? NULL : p;
    } else if (p == ZIPLIST_ENTRY_HEAD(zl)) {	// p为表头，说明前一个节点为NULL
        return NULL;
    } else {
        ZIP_DECODE_PREVLEN(p, prevlensize, prevlen);	// p减去这个节点的previous_entry_length
        assert(prevlen > 0);
        return p-prevlen;
    }
}
```

#### 2、当前节点编码信息 encoding，当前节点内容content

 `encoding`用于记录当前节点内容的实际类型（字符串还是整数），以及长度：

* `encoding`长度可以是1字节、2字节、或5字节。其中`encoding`最高两位为00, 01, 或10时表示存储的值类型为字符串；`encoding`最高两位为11表示存储的值类型为整数。

每一种`encoding`对应的编码长度和content类型，参考如下源码：

```c
// ziplist.c
/* Different encoding/length possibilities */
#define ZIP_STR_MASK 0xc0			// 11000000
#define ZIP_INT_MASK 0x30			// 00110000
#define ZIP_STR_06B (0 << 6)		// 00bbbbbb （长度小于等于63）
#define ZIP_STR_14B (1 << 6)		// 01bbbbbb bbbbbbbb  (长度大于63且小于等于16383)
#define ZIP_STR_32B (2 << 6)		// 10______ bbbbbbbb bbbbbbbb bbbbbbbb bbbbbbbb（长度大于16384）
#define ZIP_INT_16B (0xc0 | 0<<4)	// 11000000  16位整数
#define ZIP_INT_32B (0xc0 | 1<<4)	// 11010000  32位整数
#define ZIP_INT_64B (0xc0 | 2<<4)	// 11100000  64位整数
#define ZIP_INT_24B (0xc0 | 3<<4)	// 11110000  24位整数
#define ZIP_INT_8B 0xfe				// 11111110  8位整数

/* 4 bit integer immediate encoding |1111xxxx| with xxxx between
 * 0001 and 1101. */
#define ZIP_INT_IMM_MASK 0x0f   /* Mask to extract the 4 bits value. To add
                                   one is needed to reconstruct the value. */
#define ZIP_INT_IMM_MIN 0xf1    /* 11110001 */
#define ZIP_INT_IMM_MAX 0xfd    /* 11111101 */

#define INT24_MAX 0x7fffff
#define INT24_MIN (-INT24_MAX - 1)

/* Macro to determine if the entry is a string. String entries never start
 * with "11" as most significant bits of the first byte. */
#define ZIP_IS_STR(enc) (((enc) & ZIP_STR_MASK) < ZIP_STR_MASK)  // 判断是否为字节数组编码！！！

/* Extract the encoding from the byte pointed by 'ptr' and set it into
 * 'encoding' field of the zlentry structure. */
#define ZIP_ENTRY_ENCODING(ptr, encoding) do {  \
    (encoding) = (ptr[0]); \
    if ((encoding) < ZIP_STR_MASK) (encoding) &= ZIP_STR_MASK; \
} while(0)

/* Decode the entry encoding type and data length (string length for strings,
 * number of bytes used for the integer for integer entries) encoded in 'ptr'.
 * The 'encoding' variable will hold the entry encoding, the 'lensize'
 * variable will hold the number of bytes required to encode the entry
 * length, and the 'len' variable will hold the entry length. */
#define ZIP_DECODE_LENGTH(ptr, encoding, lensize, len) do {                    \
    ZIP_ENTRY_ENCODING((ptr), (encoding));                                     \
    if ((encoding) < ZIP_STR_MASK) {                                           \
        if ((encoding) == ZIP_STR_06B) {                                       \
            (lensize) = 1;                                                     \
            (len) = (ptr)[0] & 0x3f;                                           \
        } else if ((encoding) == ZIP_STR_14B) {                                \
            (lensize) = 2;                                                     \
            (len) = (((ptr)[0] & 0x3f) << 8) | (ptr)[1];                       \
        } else if ((encoding) == ZIP_STR_32B) {                                \
            (lensize) = 5;                                                     \
            (len) = ((ptr)[1] << 24) |                                         \
                    ((ptr)[2] << 16) |                                         \
                    ((ptr)[3] <<  8) |                                         \
                    ((ptr)[4]);                                                \
        } else {                                                               \
            panic("Invalid string encoding 0x%02X", (encoding));               \
        }                                                                      \
    } else {                                                                   \
        (lensize) = 1;                                                         \
        (len) = zipIntSize(encoding);                                          \
    }                                                                          \
} while(0)
```

整数编码：

| encoding               | encoding长度 | content长度                                                  |
| :---------------------- | :------------ | :------------------------------------------------------------ |
| 11111110 (ZIP_INT_8B)  | 1字节        | 8位整数                                                      |
| 11000000 (ZIP_INT_16B) | 1字节        | 16位整数                                                     |
| 11110000(ZIP_INT_24B)  | 1字节        | 24位整数                                                     |
| 11010000(ZIP_INT_32B)  | 1字节        | 32位整数                                                     |
| 11100000(ZIP_INT_64B)  | 1字节        | 64位整数                                                     |
| 1111xxxx               | 1字节        | 0-12之间的整数。此时没有content部分，值存储在encoding的xxxx四个位 |

字节数组编码：

| encoding                                                   | encoding说明                            | encoding长度 | content长度                                    |
| :------------------------------------------------------ | :--------------------------------------- | :------------- | :---------------------------------------------- |
| 00xxxxxx (ZIP_STR_06B)                                     | 后6位表示字节数组的长度                 | 1字节        | 保存长度小于等于63字节的字节数组               |
| 01xxxxxx xxxxxxxx (ZIP_STR_14B)                            | 后14位表示字节数组的长度                | 2字节        | 保存长度大于63， 小于等于16383字节的字节数组   |
| 10______ xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx (ZIP_STR_32B) | 最后24位表示字节数组的长度，第3-8位留空 | 5字节        | 保存长度大于16384，小于 2^32 - 1字节的字节数组 |

总结：Redis根据ziplist节点存储内容的类型和大小，使用不同的编码表示，目的在于最大限度地节省空间。



<font color = 'red'>思考问题：</font>创建一个ziplist编码的空列表键，并依次添加两个值"hello"，"10086"，问此时ziplist的长度是多少，内容是什么样的？

```
127.0.0.1:6379> info  # Redis版本必须为3.2之前的，否则列表键的默认编码为quicklist，而非ziplist!
# Server redis_version:3.0.0
127.0.0.1:6379> flushdb
127.0.0.1:6379> rpush key hello 10086 # 新增列表键，依次插入两个key "hello" "10086"
(integer) 2
127.0.0.1:6379> object encoding key   # 确认列表键编码为ziplist
"ziplist"
```

如果你真的掌握了ziplist节点构成，那么不需要运行Redis，你也可以得出答案：这个ziplist占用22字节，内存如下图所示：
![](image4.png)
以下给出GDB的验证方法和验证结果，感兴趣的可以参考：

```c
// 0、确认ziplist的首地址
(gdb) p (sds)server.db[0].dict.ht[0].table[3].key
$1 = (sds) 0x7fc6ac086288 "key" // 键
(gdb) p *(robj *)server.db[0].dict.ht[0].table[3].v
$2 = {type = 1, encoding = 5, lru = 13392586, refcount = 1, ptr = 0x7fc6ac035d60} // ptr即为ziplist的首地址！

// 1、打印zlbytes, zltail, zllen
(gdb) p *(int *)0x7fc6ac035d60
$3 = 22 // zlbytes = 22
(gdb) p *(int *)0x7fc6ac035d64
$4 = 17 // zltail = 17
(gdb) p *(short *)0x7fc6ac035d68
$5 = 2  // zlen = 2

// 2、以下开始打印第一个节点
(gdb) x/xb 0x7fc6ac035d6a
0x7fc6ac035d6a: 0x00 // previous_entry_length本身占1字节，长度为0
(gdb) x/xb 0x7fc6ac035d6b
0x7fc6ac035d6b: 0x05 // encoding为00000101, 本身占1字节，表示长度为5的字节数组
(gdb) x/5c 0x7fc6ac035d6c
0x7fc6ac035d6c: 104 'h' 101 'e' 108 'l' 108 'l' 111 'o'	// content占5字节，存储"hello"
// 第一个节点总长度为 1 + 1 + 5 = 7 字节

// 3、以下开始打印第二个节点
(gdb) x/xb 0x7fc6ac035d71
0x7fc6ac035d71: 0x07 // previous_entry_length本身占1字节，长度为7
(gdb) x/xb 0x7fc6ac035d72
0x7fc6ac035d72: 0xc0 // encoding为11000000, 本身占1字节, 表示16位的整数
(gdb) p *(short *) 0x7fc6ac035d73
$6 = 10086	// content本身占2字节，存储内容正是10086

// 4、标志zlend，占1字节
(gdb) x/xb 0x7fc6ac035d75
0x7fc6ac035d75: 0xff
```

### 为什么要设计ziplist<span id="jump3"></span>

还是考虑这个列表键： `"hello" -> "10086"`

* 如果用ziplist编码，仅需22个字节存储。
* 如果用linkedlist编码，以64位环境为例，光链表节点就需要2 * sizeof(listNode) = 48个字节，这个大小远超过了数据本身的大小，导致严重的内存浪费。

可以看出，ziplist设计思想是以时间换空间，目的是节省宝贵的内存资源。

Redis中，ziplist是列表键和哈希键的底层实现之一，也用于quicklist的实现。

### ziplist相关操作<span id="jump4"></span>

压缩列表的增、删、改、查API如下：

| API           | 功能                     | 时间复杂度                                               |
| :------------- | :------------------------ | :-------------------------------------------------------- |
| ziplistPush   | 插入指定节点到表头或表尾 | 平均O(N)，连锁更新场景为O(N^2)                           |
| ziplistDelete | 删除指定节点             | 平均O(N)，连续更新场景为O(N^2)                           |
| ziplistFind   | 查找指定节点             | O(N^2)，因为节点值可能是字符串，而字符串比较复杂度为O(N) |
| ziplistNext   | 返回给定节点的下一个节点 | O(1)                                                     |
| ziplistPrev   | 返回给定节点的前一个节点 | O(1)                                                     |

#### 查找指定节点<span id="jump5"></span>

源码参考`ziplistFind`，时间复杂度为O(N^2)，因为节点值可能是字符串，而字符串比较复杂度为O(N)

```c
// 功能：寻找ziplist中，节点值和vstr相等的节点并返回
// skip表示跳过的节点数
unsigned char *ziplistFind(unsigned char *p, unsigned char *vstr, unsigned int vlen, unsigned int skip) {
    int skipcnt = 0;
    unsigned char vencoding = 0;
    long long vll = 0;

    while (p[0] != ZIP_END) {	// 如果没有达到列表尾，就一直循环遍历！
        unsigned int prevlensize, encoding, lensize, len;
        unsigned char *q;

        ZIP_DECODE_PREVLENSIZE(p, prevlensize);
        ZIP_DECODE_LENGTH(p + prevlensize, encoding, lensize, len);
        q = p + prevlensize + lensize;	// 此时q指向节点p的content

        if (skipcnt == 0) {
            /* Compare current entry with specified entry */
            if (ZIP_IS_STR(encoding)) {	// 如果是字符串，memcmp比较是否相等，复杂度O(N)
                if (len == vlen && memcmp(q, vstr, vlen) == 0) {
                    return p;
                }
            } else {
                /* Find out if the searched field can be encoded. Note that
                 * we do it only the first time, once done vencoding is set
                 * to non-zero and vll is set to the integer value. */
                // 对于传入的vstr, 解码动作只需做一次， vencoding相当于一个flag
                if (vencoding == 0) {
                    if (!zipTryEncoding(vstr, vlen, &vll, &vencoding)) {
                        /* If the entry can't be encoded we set it to
                         * UCHAR_MAX so that we don't retry again the next
                         * time. */
                        vencoding = UCHAR_MAX;
                    }
                    /* Must be non-zero by now */
                    assert(vencoding);
                }

                /* Compare current entry with specified entry, do it only
                 * if vencoding != UCHAR_MAX because if there is no encoding
                 * possible for the field it can't be a valid integer. */
                // 如果解码成功，比较整数值是否相等
                if (vencoding != UCHAR_MAX) {
                    long long ll = zipLoadInteger(q, encoding);
                    if (ll == vll) {
                        return p;
                    }
                }
            }

            /* Reset skip count */
            skipcnt = skip;
        } else {
            /* Skip entry */
            skipcnt--;
        }
        /* Move to next entry */
        // q指向content, len就是content的长度, 所以 q + len即为下一个节点的首地址！！
        p = q + len;
    }
    return NULL;
}
```

#### 连锁更新问题<span id="jump6"></span>

前面提到，ziplist中每个节点的`previous_entry_length`记录了前一个节点的长度，考虑在表头插入节点的场景：

如果列表所有节点长度都在250-253字节之前，且插入节点大于等于254字节，就会导致所有节点的`previous_entry_length`都必须从1字节扩展为5字节, 即触发了**连锁更新**。

以下举例说明这个问题：给定一个长度为3的，每个节点大小均为253字节的ziplist，往表头插入一个300字节大小的节点。
![](image5.png)
注：除了新增节点会引发连锁更新，删除节点操作也可能引发连锁更新，此处不再赘述。

<font color = 'red'>思考问题：</font>既然连锁更新的最坏复杂度为O(n^2)，为什么Redis还是放心使用ziplist？
* 因为连锁更新触发条件苛刻，只有满足存在多个连续长度为250-253之间的节点才能触发。
* ziplist只应用于节点数少且数据小的场景，即使出现了连续更新，需要更新的节点数量也很少，不会出现性能问题。


## 参考资料

【1】《Redis设计与实现》 第7章 压缩列表

【2】[Redis源码解析-基础数据-ziplist(压缩列表)](
