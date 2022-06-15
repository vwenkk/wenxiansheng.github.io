---
title: Redis设计与实现
date: 2018-03-05 21:13:33
tags:
- 学习笔记

categories:
- Redis
    - 设计与实现
---
    
Redis 设计与实现

<!-- more -->

#### SDS 的定义

C语言的字符串不能满足Redis字符串在安全性，效率以及功能方面的要求

sds 的结构

```c
// sdshdr 结构
struct sdshdr {

    // buf 已占用长度
    int len;

    // buf 剩余可用长度
    int free;

    // 实际保存字符串数据的地方
    // 利用c99(C99 specification 6.7.2.1.16)中引入的 flexible array member,通过buf来引用sdshdr后面的地址，
    // 详情google "flexible array member"
    char buf[];
};
```

- free 属性值为0，表示这个sds没有分配任何未使用的空间
- len 属性的值表示字符串的长度
- buf 属性是一个char类型的数组，数组的最后一个字节保存空字符 '\0'



> 常数复杂度获取字符串长度

C字符串不记录自身的长度信息，所以获取字符串的长度需要遍历整个字符串，复杂度为O（n）

因为SDS在len属性中记录了SDS本身的长度，所以获取sds长度的复杂度为O（1）

> 空间预分配

空间预分配用于优化SDS的字符串增长操作：当SDS的API对一个SDS进行修改并且需要对SDS进行空间扩展的时候，程序不仅会对SDS分配修改所必须的空间，还会为SDS分配额外的未使用空间，通过预分配策略，Redis可以减少连续执行字符串增长操作所需的内存重分配次数



- 如果对SDS进行修改之后，SDS的长度（len属性的值）小于1M（宏SDS_MAX_PREALLOC），那么SDS len属 性的值和free属性的值将和free属性的值相同 举个例子，如果进行修改之后，SDS的len将变成13字节，那么程序也会分配13字节的未使用空间，SDS的buf数组的实际长度将变成13+13+1=27字节（额外的一字节用于保存空字符）
- 如果SDS进行修改之后SDS的长度将大于等于1M，那么程序会分配1M未使用的空间，举个例子，如果进行修改之后，SDS的len将变成30MB，那么程序会分配1MB的未使用空间，SDS的buf数组的实际长度将为30MB+1MB+1byte。



```c
/* 
 * 对 sds 的 buf 进行扩展，扩展的长度不少于 addlen 。
 *
 * T = O(N)
 */
sds sdsMakeRoomFor(
    sds s,
    size_t addlen   // 需要增加的空间长度
) 
{
    struct sdshdr *sh, *newsh;
    size_t free = sdsavail(s);
    size_t len, newlen;

    // 剩余空间可以满足需求，无须扩展
    if (free >= addlen) return s;

    sh = (void*) (s-(sizeof(struct sdshdr)));

    // 目前 buf 长度
    len = sdslen(s);
    // 新 buf 长度
    newlen = (len+addlen);
    // #define SDS_MAX_PREALLOC (1024*1024)
    // 如果新 buf 长度小于 SDS_MAX_PREALLOC 长度
    // 那么将 buf 的长度设为新 buf 长度的两倍
    if (newlen < SDS_MAX_PREALLOC)
        newlen *= 2;
    else
        newlen += SDS_MAX_PREALLOC;

    // 扩展长度
    newsh = zrealloc(sh, sizeof(struct sdshdr)+newlen+1);

    if (newsh == NULL) return NULL;

    newsh->free = newlen - len;

    return newsh->buf;
}
```

> 惰性空间释放

当需要缩短SDS保存的字符串时，程序并不立即使用内存重分配来回收缩短后多出来的字节，而是使用free属性将这些字节的数量记录下来，并等待将来使用

> SDS 优点

- 常数复杂度获取字符串长度
- 杜绝缓冲区溢出
- 减少修改字符串长度时所需的内存重分配次数
- 二进制安全
- 兼容部分C字符串函数

#### 链表和链表节点

每个链表节点使用一个adlist.h/listNode结构来表示

```c
/*
 * 链表节点
 */
typedef struct listNode {

    // 前驱节点
    struct listNode *prev;

    // 后继节点
    struct listNode *next;

    // 值
    void *value;

} listNode;
```

多个listNode通过prev和next指针组成双端链表

```c
/*
 * 链表
 */
typedef struct list {

    // 表头指针
    listNode *head;

    // 表尾指针
    listNode *tail;

    // 节点数量
    unsigned long len;

    // 复制函数
    void *(*dup)(void *ptr);
    // 释放函数
    void (*free)(void *ptr);
    // 比对函数
    int (*match)(void *ptr, void *key);
} list;
```

list结构为链表提供了表头指针head、表尾指针tail，以及链表长度计数器len，而dup、free和match成员则是用于实现多态链表所需的类型特定函数

- dup函数复制链表节点时调用，用于复制链表节点所保存的值
- free函数释放链表节点时调用，用于释放链表节点所保存的值
- match函数在列表中查找和 key 匹配节点时调用，如果列表带有匹配器，那么匹配通过匹配器来进行。如果列表没有匹配器，那么直接将 key 和节点的值进行比对。

Redis的链表实现的特性总结如下：

- 双端：链表节点带有prev和next指针，获取某个节点的前置节点和后置节点的复杂度都是O(1)
- 无环：表头节点的prev指针和表尾节点的next指针都指向NULL，对链表的访问以NULL为终点
- 带表头指针和表尾指针：通过list结构的head指针和tail指针，程序获取链表的表头节点和表尾节点的复杂度为O(1)
- 带链表长度计数器：程序使用list结构的len属性来对list持有的链表节点进行计数，程序获取链表中节点数量的复杂度为O(1)
- 多态：链表节点使用void * 指针来保存节点值，并且可以通过list结构的dup、free、match三个属性为节点值设置类型特定函数，所以链表可以用于保存各种不同类型的值

#### 字典

字典，又称为符号表、关联数组或映射，用于保存键值对的抽象数据结构。在字典中，一个键可以和一个值进行关联（或者说将键映射为值），这些关联的键和值就称为键值对。

在字典中，一个键可以和一个值进行关联，这些关联的键和值就称之为键值对。

字典中的每个键都是唯一的。

> 字典的实现

Redis的字典使用哈希表作为底层实现，一个哈希表里面可以有多个哈希表节点，而每个哈希表节点就保存了字典中的一个键值对

> 哈希表

Redis字典所使用的的哈希表由dict.h/dictht结构定义：

```c
/*
 * 哈希表
 */
typedef struct dictht {

    // 哈希表节点指针数组（俗称桶，bucket）
    dictEntry **table;      

    // 指针数组的大小
    unsigned long size;     

    // 指针数组的长度掩码，用于计算索引值
    // 值总是等于size-1
    unsigned long sizemask; 

    // 哈希表现有的节点数量
    unsigned long used;     

} dictht;
```

哈希表节点使用dictEntry结构表示，每个dictEntry结构保存一个键值对

```c
/*
 * 哈希表节点
 */
typedef struct dictEntry {

    // 键
    void *key;

    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    // 链往后继节点
    struct dictEntry *next; 

} dictEntry;
```

key 保存键值对中的键，v 保存值，next属性是指向另一个哈希表节点的指针，这个指针可以将多个哈希值相同的键值对连接在一起，用来解决键冲突的问题

![image-20210802114749082](http://imgbed-wk.oss-cn-beijing.aliyuncs.com/img/image-20210802114749082.png)

> 字典

```c
/*
 * 字典
 *
 * 每个字典使用两个哈希表，用于实现渐进式 rehash
 */
typedef struct dict {

    // 特定于类型的处理函数
    dictType *type;

    // 类型处理函数的私有数据
    void *privdata;

    // 哈希表（2个）
    dictht ht[2];       

    // 记录 rehash 进度的标志，值为-1 表示 rehash 未进行
    int rehashidx;

    // 当前正在运作的安全迭代器数量
    int iterators;      

} dict;
```

type 属性和 privdata属性是针对不同类型的键值对，为创建多态字典而设置的

privdata属性保存了需要传给那些类型特定函数的可选参数

ht 属性是一个包含两个项的数组，数组中的每个项都是一个dictht哈希表，一般情况下字典只使用ht[0]哈希表，ht[1]哈希表只会对ht[0]哈希表进行rehash时使用

> 解决键冲突

当有两个或以上数量的键被分配到了哈希表数组的同一个索引时，我们称这些键发生了冲突

Redis 的哈希表使用链地址法来解决冲突，每个哈希表节点都有一个next指针，多个哈希表节点可以用next指针构成一个单向链表，被分配到同一个索引上的多个节点可以用这个单向链表链接起来，这样来解决冲突的问题

#### 跳跃表

```c
/*
 * 跳跃表
 */
typedef struct zskiplist {
    // 头节点，尾节点
    struct zskiplistNode *header, *tail;
    // 节点数量
    unsigned long length;
    // 目前表内节点的最大层数
    int level;
} zskiplist;

/* ZSETs use a specialized version of Skiplists */
/*
 * 跳跃表节点
 */
typedef struct zskiplistNode {
    // member 对象
    robj *obj;
    // 分值
    double score;
    // 后退指针
    struct zskiplistNode *backward;
    // 层
    struct zskiplistLevel {
        // 前进指针
        struct zskiplistNode *forward;
        // 这个层跨越的节点数量
        unsigned int span;
    } level[];
} zskiplistNode;
```

#### 整数集合

整数集合是集合键的底层实现之一，一个集合中只包含整数值元素，并且元素数量不多时，Redis会使用整数集合键的底层实现，它可以保存类型为int16_t、int32_t或者int64_t的整数值，并且保证集合中不会出现重复元素。

```c
typedef struct intset {

    // 保存元素所使用的类型的长度
    uint32_t encoding;

    // 元素个数
    uint32_t length;    

    // 保存元素的数组
    int8_t contents[];  

} intset;
```

contents的属性声明为int8_t类型的数组，但是实际上contents 数组中并不保存任何int8_t类型的值，contents数组的真正类型由encoding类型的值决定

- 如果encoding属性的值为INTSET_ENC_INT16，那么数组里的每个项都是一个int16_t类型的整数值
- 如果encoding属性的值为INTSET_ENC_INT32，那么数组里的每个项都是一个int32_t类型的整数值

- 如果encoding属性的值为INTSET_ENC_INT64，那么数组里的每个项都是一个int64_t类型的整数值

> 升级

每当我们要将一个新元素添加到整数集合中，并且新元素的类型比整数集合现有所有元素的类型都要长时，整数集合需要先进行升级，然后才能将新元素添加到整数集合里面。

> 降级

整数集合不支持降级操作，一旦对数组进行了升级，编码就会一直保持升级后的状态。

#### 压缩列表

压缩列表是列表键和哈希键的底层实现之一。当一个列表键只包含少量列表项，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压缩列表做列表键的底层实现

```bash
127.0.0.1:6379> rpush lst 1 3 5 666 "hello" "world"
(integer) 6
127.0.0.1:6379> object encoding lst
"ziplist"
```

当一个哈希键只包含少量键值对，并且每个键值对的键和值要么是小整数值，要么是长度比较短的字符串，那么redis就会使用压缩列表来做哈希键的底层实现

```bash
127.0.0.1:6379> hmset profile "name" "wk" age 28 job php
OK
127.0.0.1:6379> object encoding profile
"ziplist"
127.0.0.1:6379>
```

