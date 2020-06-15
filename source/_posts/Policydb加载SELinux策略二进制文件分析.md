---
title: Policydb加载SELinux策略二进制文件分析
date: 2019-12-01 16:30:16
tags:
	- SELinux
---

@[toc](Policydb加载策略二进制文件分析)
Linux 操作系统启动时，init 进程首先初始化 LSM 模块并将SELinux 作为它的一个可选安全模块加载到系统中，最后 init 通过文件系统的策略加载接口读取 policy.x 文件。策略的加载过程便是将二进制策略配置文件中的数据加载到内存中表示为策略数据库结构的过程，由于策略数据结构比较复杂，解析的过程将是一个很复杂 和细致的工作。
# security_load_policy函数
解析二进制策略配置文件的功能是通过int security_load_policy(void  *data,size_t  len)函数实现的，该函数位于 security/selinux/ss/services.c中，在运行时被selinux伪文件系统加载：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201160744277.png)

security_load_policy函数原型为：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201160926148.png)
参数data是指向二进制文件数据的指针，参数len是该二进制文件的数据长度，解析过程如下图所示：
## 解析过程
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/2019120116261661.png)
加载策略时，需要分两种情况：第一种是没有加载策略，即内核中不存在任何其它的安全策略库；第二种是已经加载了策略，即内核中已经存在安全策略库。
	
如果没有加载策略，则执行的是 if  (!state->initialized)里的语句，首先读取二进制策略配置文件，这通过调用 policydb_read()函数实现。

其次是调用访问控制向量表提供的功能函数 avc_ss_reset()重置访问控制向量表高速缓存区。
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201161102342.png)
下面重点分析policydb_read()函数
## policy_read()函数
由于 policydb 结构比较大，涉及到许多比较复杂的数据结构，因此对这个过程的具体实现步骤需要进行详细分析。
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201162954103.png)

首先是由 policydb_init()函数实现的初始化 policydb 结构的过程，具体步骤如下： 
### policy_init()函数
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201163101375.png)
1)memset(p,  0,  sizeof(*p))；将策略数据库结构中所有的内存空间都置 0，防止数
据污染。
2）symtab_init(&p->symtab[i],  symtab_sizes[i])；初始化策略数据库结构中的各个符号表对象，具体为借助 symtab_init()创建符号表中的哈希表，并将基本名称数量初始化为 0。
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201164311516.png)
3)  avtab_init(&p->te_avtab)；初始化avtab

4）roles_init(p)；初始化角色表，具体为在策略数据库对象中插入第一个角色对象，它的键值和内部角色值分别为宏 OBJECT_R 与 OBJECT_R_VAL 的值，也即为“object_r”与 1，这是为初始角色所定义的值。
5） cond_policydb_init(p)；初始化策略数据库结构中与条件式有关的成员，包括bool_val_to_struct,cond_list 和 te_cond_avtab。te_cond_avtab 和 te_avtab 是同一结构的两个不同数据成员，它们都是调用 avtab_init 进行初始化，不同的是初始化的对象。
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201164038820.png)
policydb_init之后接下来是对二进制策略文件数据的读取，这个才是该 policydb_read()函数所做的主要事情
### policy_read()函数读取二进制文件
文件读取的过程如下： 
1)  读取文件的魔数和策略库字符串长度，验证魔数是否为 POLICYDB_MAGIC；
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201164646772.png) 
2)  验证字符串长度，如果不符合要求，则返回错误代码， 该长度必须与POLICYDB_STRING 相等； 
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201165145604.png)
3)  读取策略库字符串 policydb_str，并验证其是否为 POLICYDB_STRING；
 ![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201165521459.png)
4)  读取策略库的版本信息、配置和符号表大小； 
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201165430957.png)
5)  初始化策略库版本 policyvers； 
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201165818702.png)
6)  检查是否启用多级安全架构； 
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201171753954.png)
7)  验证版本信息的正确性； 
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201171915433.png)
8)  读取符号表信息，并填充到策略数据库对象的各个符号表 symtab[i]中； 
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201172005372.png)
9)  接着是调用 avtab_read 函数，读取文件中的 TE 策略规则，将所读取的规则保存到 avtab_node 节点，并插入 te_avtab 表中； 
10) 调用 cond_read_list 读取条件访问向量规则链表，条件链表中的 TE 规则存储到te_cond_avtab 中； 

11) 读取角色转换属性数据，这些规则限定了用户在运行时能否转换角色； 
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201172239530.png)
12) 读取角色授权属性(role_allow)数据；

13) 读取 filename_trans 规则集合； 
15) 设置策略数据库结构对象的进程转换权限；
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201172556332.png) 
16) 读取对象的初始安全上下文并填充到 ocontext[OCON_NUM]成员中；
17) 读取 genfs 数据结构； 
18) 读取 MLS 相关信息，包括安全等级转换及 MLS 安全等级； \
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201172639299.png)
19) 读取类型所属的属性（attribute，它与类型公用相同的命名空间），保存类型到属性的倒转映射关系到 type_attr_map[]数组中； 
20) 进行策略数据库结构的边界检查。 
至此 policydb_read()操作完成
这些操作完成后，二进制策略文件就读取到内核中了。

## 其余工作
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191201172953241.png)
第三步是：调用selinux_set_mapping()建立客体类别的策略值（policy value）与内核值（kernel value）之间的一一映射关系。具体过程为： 
1)  找到输入 mapping 中类别的数量； 
2)  为 current_mapping 数据结构分配内存空间，需多分配一个成员空间给“类别零”； 
3)  循环遍历所有的客体类别，也即 seclass_map 集合中的各个元素值，在循环中通过string_to_security_class()函数获取类别的策略值，string_to_av_perm()获取对应类别的内核值，将两者关联，最终将完成设置的 current_mapping 返回。 

第四步是调用 policydb_load_isid()填充 sidtab 数据，该步骤可以分为以下几步： 
1）调用 sidtab_init()函数初始化 SID 表； 
2）将 p->ocontexts[OCON_ISID]中的对象初始安全上下文，填充到 SID 表中。 

最后一步是调用相关的函数处理加载策略库后的影响：
1)  建立策略加载前所有的超级块，完成超级块的初始化；
2)  刷新访问向量缓存 AVC，并更新许可；
3)  通过netlink 通信机制广播 seqno 到用户空间； 
4)  更新策略重新加载的次数及当前deny_unkown 的设置，也即对未知权限的处理方式。 

## 内核已经存在策略的情况
到目前为止，本文只讲了整个问题的一半，下面讲如果内核已经存在策略该怎么
办。 
第一步：policydb_read 读取新的策略文件； 
第二步：policydb_load_isids 填充 sidtab 数据； 
第三步：selinux_set_mapping 建立客体类别的内核值与策略值之间的一一映射关
系； 
第四步：调用 security_preserve_bools()以维持布尔值集合不变，也即将已存在的策
略中的布尔值保存到新加载的策略库中。 
第五步：利用 sidtab_map 用新 sid 表 newsidtab 替换原来的 sidtab，在这之前必须先
屏蔽掉外部对 sidtab 的查询。 
第六步：保存原来的 policydb 和 sid 表，以备释放其占用的内存空间。 
第七步：装载新的 policydb 和 sid 表。 
第八步：处理其它事情，这部分的工作与第一种情况中的收尾工作大致不差，主要有：
1)  释放为旧的策略数据库结构分配的内存空间；
2)  释放为旧的 SID 表分配的内存空间；
3)  刷新访问向量缓存并更新许可；
4)  通过 netlink 通信机制广播 seqno