---
title: SELinux中的avc源码分析
date: 2019-11-08 11:08:16
tags:
	- SELinux
---

/security/selinux/avc.c

<!--more-->

# avc的存储
AVC的缓存由类似文件系统的node和entry结构组成，结构avc_cache、avc_node、avc_entry、av_decision的关系图如下所示：
实箭头表示指针(所有指针没有完全画出，只画出了重要的逻辑)，白色箭头表示结构体的内容。
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191108090214403.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191108090420513.png?)

#define AVC_CACNE_SLOTS  512
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/2019110720170651.png)

## avc节点的插入
avc_insert()
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191108104028882.png)
重点看了avc节点的插入，一个新的avc节点插入到avc的哪个位置首先要找到将它放到哪个哈希槽中，计算 `hvalue=avc_hash(ssid,tsid,tclass)`作为哈希链表 头结点数组的下标，通过`avc_node_populate`将ssid，tsid，tclass与相应的avd信息赋值到 node相应的位置。
`hlist_for_each_entry`是linux中哈希链表的一个遍历函数，如果在当前的list中，存在ssid、tsid、tclass于node完全相同的节点，用node替代该位置上节点，goto语句跳出。
如果不存在重复节点，在该list上新增一个节点`（hlist_add_head_rcu）`。

# 函数avc_has_perm分析
函数avc_has_perm从avc中查找决策信息，如果没找到，使用内核SELinux计算决策，并将决策信息相应条目插入到avc中，记录审核信息。函数avc_has_perm列出如下：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191107170208878.png)
ssid表示主体的sid，tsid表示客体的sid，class是客体的类别，requested是请求的类别
avc_has_perm_noaudit获取决策信息
然后把返回值rc传到avc_audit中，如果决策否决了操作，则将安全上下文、决策信息等写入log文件或者输出
先看 avc_has_perm_noaudit函数：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191107161151131.png)
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191107165906523.png)
最终都可以获得相应的avd（决策信息），将`u32 requested`与 `u32`的`avd->allowed`取反进行 按位与
`denied`为`true`的情况是，`requested`与`avd->allowed`的二进制形式至少存在一个位置，`requested`的这一位为1，`avd->allowed`的这一位为0（疑问：这个32位的数值代表什么？为什么不用数值判断而是用位操作符判断？需要查看av的存储形式做判断）
大部分情况下 `denied`为`false`，也就是二进制形式中，`requested`为1的位，对应的`avd->allowed`一定为1.
之后的处理如下：

```c
	denied = requested & ~(avd->allowed);                        
	if (unlikely(denied))                         
		rc = avc_denied(state, ssid, tsid, tclass, requested, 0, 0,
				flags, avd);

	rcu_read_unlock();
	return rc;
```
如果 denied是false，直接返回rc =0 ；
如果 denied是true，调用 avc_denied()后返回rc=0;
（都是在一般情况下，像没有足够内存分配或者其他错误返回值暂时不考虑）
avc_denied()函数如下：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191108162518610.png)
回到avc_has_perm函数，avc_audit是安全审计函数，不作重点分析。

```c
rc2 = avc_audit(state, ssid, tsid, tclass, requested, &avd, rc,
			auditdata, 0);
```
下面我们来看avc_compute_av()如何通过ssid，tsid，tclass计算avd
# avc_compute_av()函数分析

```c
/*
 * Slow-path helper function for avc_has_perm_noaudit,
 * when the avc_node lookup fails. We get called with
 * the RCU read lock held, and need to return with it
 * still held, but drop if for the security compute.
 *
 * Don't inline this, since it's the slow-path and just
 * results in a bigger stack frame.
 */
static noinline
struct avc_node *avc_compute_av(struct selinux_state *state,
				u32 ssid, u32 tsid,
				u16 tclass, struct av_decision *avd,
				struct avc_xperms_node *xp_node)
{
	rcu_read_unlock();
	INIT_LIST_HEAD(&xp_node->xpd_head);
	security_compute_av(state, ssid, tsid, tclass, avd, &xp_node->xp);
	rcu_read_lock();
	return avc_insert(state->avc, ssid, tsid, tclass, avd, xp_node);
}
```
主要调用了security_compute_av，最终在avc中插入一个节点
对于如何计算access_vector比较复杂（见下图），还没有整理，计划下周进行
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191107200032844.png)