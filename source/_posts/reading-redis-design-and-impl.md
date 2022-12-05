---
title: '[读书笔记]-redis设计与实现'
date: 2022-12-05 19:27:34
tags:
	- reading
    - redis
---

知其然，知其所以然之：redis

<!--more-->

# 第一部分 数据结构与对象
## 第二章 简单动态字符串
- 简单动态字符串（SDS）：可以被修改的字符串

结构如下：
```C
struct sdshdr { 
    int len;    // 记录buf数组中已使用字节的数量    // 等于SDS所保存字符串的长度  
    int free; // 记录buf数组中未使用字节的数量
    char buf[]; // 字节数组，用于保存字符串
```

- 个人理解：封装过的C字符串数组

用途：
1. 空间预分配
- 修改后 buf.size(need) < 1m，分配给free以buf.size(need)大小的内存。buf.size = 2* buf.size(need)
- 修改后 buf.size(need) >= 1m，分配给free以buf.size(need)大小的内存。buf.size = buf.size(need)+ 1m
2. 惰性空间释放
- sds的字符删减之后，不会释放空间。而是继续留着，等待下次添加时使用

## 第三章 链表
C语言没有内置链表，所以redis自己实现了。

链表节点：

```C
typedef struct listNode { 
    struct listNode * prev; // 前置节点
    struct listNode * next; // 后置节点
    void * value; // 节点的值
} listNode;
```

list:链表

```C
typedef struct list { 
    listNode * head; // 表头节点
    listNode * tail; // 表尾节点
    unsigned long len; // 链表包含节点数量
    void *(*dup)(void *ptr); // 节点值复制函数
    void (*free)(void *ptr); // 节点值释放函数
    int (*match)(void *ptr,void *key); // 节点值对比函数
} list;
```
list是双向不循环链表

## 第四章 字典
C语言没有内置map，所以redis自己实现了。
redis实现的map底层使用哈希表 (HashMap?)
### 4.1 结构
1. 哈希表

```C
typedef struct dictht {
    dictEntry **table; // 哈希表数组
    unsigned long size; // 哈希表大小(上面这个数组的大小)
    unsigned long sizemask; //哈希表大小掩码，用于计算索引值    //总是等于size-1
    unsigned long used; // 已有节点的数量
} dictht;
```

2. 哈希表节点：一个键值对
```C
typedef struct dictEntry { 
    void *key; // 键
    union{
        void *val;        
        uint64_tu64;        
        int64_ts64;    
        } v;   // 值.可以是其中一种   
    struct dictEntry *next; // 指向下个哈希表节点，形成链表
} dictEntry;
```

3. 字典
```C
typedef struct dict {
    dictType *type; // 类型特定函数
    void *privdata; // 私有数据
    dictht ht[2]; // 哈希表
    in trehashidx; // rehash索引    //当rehash不在进行时，值为-1
} dict;
```

type 和 privdata 是针对不同类型的键值对，为创建多态字典设置
- dictType 结构保存了一些用于操作指定类型的的函数，根据dictType不同，dict可以使用不同的函数
- privdata，保存需要传给类型特定函数的数据

### 4.2 哈希算法
ht[0]主要使用，ht[1]用作rehash

放入一个键值对 k0v0
- 计算哈希值为8
- 8和dictht.sizeMark计算：放入 ht[0].dictEntry[0]

### 4.3 解决键冲突
盲猜，使用k0v0的next指针

bingo。链地址法解决冲突

### 4.4 rehash

问题：
- 什么时候需要rehash？
- rehash是由谁执行的？
>1)服务器目前没有在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1。

>2）服务器目前正在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5。

这个要一直探测？

### 4.5 渐进式哈希

## 第五章 跳跃表

“Redis使用跳跃表作为有序集合键的底层实现之一”

- 有序表包含元素数量比较多
- 有序集合的member是较长字符串

