---
title: SELinux安全上下文与sid的映射
date: 2019-10-24 01:22:48
tags:
	- SELinux
---

# 存储
用户态所使用的安全上下文都以“user:role:type[:mls_range]”字符串表示
在 SELinux 内核空间用context 数据结构来描述 SC，其中各个域都以 u32 来表示

<!--more-->

![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191024160745283.png)
SELinux 内核空间给每个有效的 context 分配一个 u32 sid，所有内核数据结构的安全扩展中都通过 sid
来描述当前对象的安全上下文。sidtab_node 用于建立 context 数据结构和其 sid 之间的关联，组织为
sidtab 哈希表。

## sidtab
SID表用一个hash表存储了SID和安全上下文的节点，提供了SID和安全上下文的对应关系。通过查找表sidtab，可方便地由SID找到安全上下文，hash表的hash值就是SID。
结构sidtab列出如下（在security/selinux/ss/sidtab.h中）：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191024162633740.png)
SID表节点由结构sidtab_node 的实例组成，它描述了SID和安全上下文的对应关系，这样，知道了SID或者安全上下文，就可以找到安全上下文或者SID。结构sidtab_node 列出如下：

```c
struct sidtab_node {
	u32 sid;		/* SID,security identifier */
	struct context context;	/* 安全上下文结构security context structure */
	struct sidtab_node *next; //下一个节点
};
```

结构context是安全上下文的描述结构，安全上下文包括用户、角色、类型和安全级别，结构context列出如下（在selinux/ss/context.h中）：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191024163128426.png)
SID表sidtab的组织结构图如下图所示：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/2019102510031012.png)
在selinux/ss/sidtab.c中提供了SID表sidtab的管理函数，例如，函数sidtab_init初始化sidtab表，函数sidtab_insert在表中插入一个节点；函数sidtab_search从表中查找到SID对应的安全上下文；函数sidtab_search_context从表中找到安全上下文对应的SID；函数sidtab_context_to_sid将安全上下文转换到对应的SID，如果表中不存在这个节点，将其加入到表中。

看一下**sidtab的初始化：**

```c
int sidtab_init(struct sidtab *s)
{
	int i;

	s->htable = kmalloc(sizeof(*(s->htable)) * SIDTAB_SIZE, GFP_ATOMIC);
	if (!s->htable)
		return -ENOMEM;
	for (i = 0; i < SIDTAB_SIZE; i++)
		s->htable[i] = NULL;
	s->nel = 0;
	s->next_sid = 1;  ## 初始化sidtab时，下一个可以分配的sid是1 unsigned int next_sid;	
	s->shutdown = 0;
	spin_lock_init(&s->lock);
	return 0;
}
```
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191025091749196.png)
可以看出SIDTAB_HASH_BUCKETS 是128，也就是有128个哈希槽。
# 操作
## sitab_node 的插入

```c
int sidtab_insert(struct sidtab *s, u32 sid, struct context *context)
{
	int hvalue, rc = 0;
	struct sidtab_node *prev, *cur, *newnode;
```
看输入参数，插入的前提是**已经得到了sid与相应的context**

```c
	hvalue = SIDTAB_HASH(sid);
	prev = NULL;
	cur = s->htable[hvalue];
```
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191025101130912.png)
哈希值是直接取sid的后七位
第一个例子：SID是28

hvalue = 28
prev = NULL
cur = s->htable[28] =NULL

```c
	while (cur && sid > cur->sid) {
		prev = cur;
		cur = cur->next;
	}

	if (cur && sid == cur->sid) {
		rc = -EEXIST;
		goto out;
	}

	newnode = kmalloc(sizeof(*newnode), GFP_ATOMIC);
	if (newnode == NULL) {
		rc = -ENOMEM;
		goto out;
	}
	newnode->sid = sid;
	if (context_cpy(&newnode->context, context)) { //context复制成功会返回0
		kfree(newnode);
		rc = -ENOMEM;
		goto out;
	}

	if (prev) {
		newnode->next = prev->next;
		wmb();
		prev->next = newnode;
	} else {
		newnode->next = s->htable[hvalue];
		wmb();
		s->htable[hvalue] = newnode;
	}

	s->nel++;
	if (sid >= s->next_sid)
		s->next_sid = sid + 1;
out:
	return rc;
```
while跟if都走不到，直接分配一个newnode
newnode的sid是已知的sid，将context拷贝到newnode中
prev是null
那么最终执行的是：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191025104247596.png)
sid为28的节点作为 s->htable[28]的链表的头结点

如果是sid是156，
hvalue = 28
prev = NULL
cur = s->htable[28] =【sid=28 | * next = NULL】
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191025114103837.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191025114052522.png)
在s->htable[28]这个链表用尾插法插入新节点。
## 创建 context 并注册到 sidtab 以获得 sid 的时机
任何内核数据结构的安全属性中都以 u32 sid 描述相应数据结构的安全上下文。sid 有效说明对应的context 数据结构已经存在并且在 sidtab 中注册过了。如果 Security Server 尚未初始化，则有效的 sid 只能是固化在 SELinux 内核驱动中的 Initial SID。如
果参数 sid 属于 Initial SID，则直接从 initial_sid_to_string 数组中拷贝相应的字符串

Initial SID /security/selinux/include/flask.h
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191025093554178.png)

每当得到一个新的 context 时（比如进入到新的 domain 或者创建新的文件或内核对象时），都需要调用 sidtab_context_to_sid 函数向 sidtab 查询它所对应的 sid，如果不存在则注册之并返回相应的 sid。

由 sidtab_context_to_sid 函数的调用者创建 context 数据结构，所有的可能调用如下：

1. Initial SID string 向 sidtab 注册时
security_load_policy  
---> policydb_load_isid 
---> sidtab_insert
2. Domain Transition 时
当进程 exec 可执行文件时可能会发生 Domain Transition，根据当前进程 tsec->sid 以及可执行文件 isec->sid 查询 policy 得到新的 domain 和新的 context。
selinux_bprm_set_security
--->security_transition_sid
---> security_compute_sid         # 根据 subject/target context，计算 newcontext
---> sidtab_context_to_sid
3. 创建新文件时
vfs_create
---> may_create
---> security_transition_sid
---> security_compute_sid
---> sidtab_context_to_sid
vfs_create
---> ...
---> security_inode_init_security == selinux_inode_init_security
---> security_compute_sid
---> sidtab_context_to_sid
4. 当用户写入/proc/<$pid>/attr/*文件时
用户通过 attr/下的 current/exec 文件来显式指定当前进程的 SC 字符串，或者下一次执行 exec 时切换
到的 SC 字符串。
selinux_setprocattr
---> security_context_to_sid # 将 SC 字符串转化为一个 context 数据结构
---> sidtab_context_to_sid

其中，通过context获得sid的函数是 sidtab_context_to_sid

```c
int sidtab_context_to_sid(struct sidtab *s, struct context *context, u32 *out_sid)
```
输入是sidtab 与context ，输出out_sid

![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191025162437850.png)
过程如下：

- 先搜索sidtab的cache，如果cache中存在相应的context，那么返回相应的sid
- 如果cache中不存在相应的context，那么查找整个sidtab
- 如果整个sidtab中存在相应的context，返回相应的sid
- 如果整个sidtab中不存在相应的context，分配一个新的sid

分配的过程先检查有没有超过无符号整数的最大界限，如果没超过，将 sidtab结构中保存的 next_sid赋值给sid，同时next_sid+1
然后把context与对应的sid作为一个新的sidtab_node插入到sidtab中

从表中查找到SID对应的安全上下文是一个简单的查找哈希表的过程；
## initial SID注册到sidtab中
函数sidtab_search_context从表中找到安全上下文对应的SID

![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191025164010766.png)
是一个数组链表遍历的过程，不再分析。

在策略数据库初始化的过程中，装载SID的第一步就是在sidtab中注册初始SID：

/security/selinux/ss/policydb.c::policydb_load_isids(struct policydb *p, struct sidtab *s)
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191025165712887.png)
接着调用了sidtab_insert函数
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191025165741773.png)
初始化之后，sidtab中的next_sid是28
非系统定义的context生成时，可以使用的第一个sid就是28了。