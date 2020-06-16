---
title: security_compute_av源码分析
date: 2019-11-15 11:16:53
tags:
	- SELinux
---

security_compute_av是负责计算ssid对tsid的访问权限决策的函数，函数位置在：
/security/selinux/ss/service.c ::security_compute_av

<!--more-->

security_compute_av整体过程：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191108162518611.png)

1. 首先获取ssid 的安全上下文结构
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/2019111411394111.png)
如果sidtab中存在相应的ssid，再判断这个ssid对应的scontext是否被确定为permissive

![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191114114108434.png)
找到相关的解释如下：
A domain running with a permissive type will be allowed to perform any action similarly to when the system is globally set permissive.
如果域是 permissive 类型，那么 这个域将被允许任何访问，（相当于系统对这个域开的是permissive模式）。
实现方式是：
在policydb中存储了 bitmap类型的permissive_map，通过ebitmap_get_bit函数判断，主体的type是否在bitmap中（也就是type这个32位的数值是否为permissive_map中的一个key）

2. 获取tsid 的安全上下文结构
    ![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191114114739587.png)

3. 获取客体类别 tclass的策略值
    ![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191114141316278.png)

4. 基于前面获取的主体和客体安全上下文及客体类别的策略值调用安全服务器的
    context_struct_compute_av()计算访问向量决策avd
    ![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191114144952764.png)
    下面来看context_struct_compute_av()函数：
    4.1 该函数首先初始化访问控制向量。
    ![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191114145711911.png)
    4.2 第二步是检查tclass 是否定义，如果没有定义
    就直接返回错误信息。
    ![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191114153509699.png)
    如果能正常识别，则通过tclass 在 policydb 的 class_val_to_struct数组中查找相应的 tclass_datum 数据
    ![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191114153620346.png)
    在policydb中可以看到class_val_to_struct的定义，并且user、role、type的datum都是由 他们的 value-1作为索引
    ![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/2019111415422510.png)
    class_datum 的结构图如下所示：

  

  ![](https://gitee.com/liying000/blogimg/raw/master/image-20200616231435922.png)

4.3 获取scontext和tcontext的属性：
sattr与tattr都是ebitmap结构
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191114160438450.png)
4.4 填充avd 中的数据
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191114164821227.png)
调用ebitmap_for_each_positive_bit()宏，该宏实际上是一个for循环。（n是ebitmap_node）
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191114171510606.png)
这里两次调用该宏分别对源和目标属性进行组对，然后以其作为键值来查找策略中的访问向量规则，如果存在相应的规则，则根据规则的类型设置访问向量决策

我们先来看avtab结构：

结构avtab是描述策略中所有向量访问规则的表，表的每个节点对应策略中一条访问向量规则。

avtab链表结构如下所示：

![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191115111626123.png)
可见avtab_node是用avtab_key作为索引的，而avtab_key是由source_type、target_type等计算得到，
遍历 sattr 和 tattr 位图，查找 sattr 和 tattr 位图中的置 1 位，分别加 1 之后初始化avkey，for循环结束条件是直到找到 avkey.apecified(在前面的代码中是与xperms相关联)

 然后根据avtab_node->key的specified字段更新avd。

![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191115095422394.png)

> 思考：context是跟sid一一对应的，context中的type保存在type_attr_map_array中，可以通过这一部分得到type对应的位图，而在计算avd时，通过遍历位图得到avkey，avkey中保存了 sourcetype对应targettype的specified字段就是相应的访问规则。
> 整体比较复杂，大量使用哈希链表与位图。