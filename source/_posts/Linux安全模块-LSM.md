---
title: Linux安全模块(LSM)
date: 2020-06-16 00:47:07
tags:
	- SELinux
---

# 1. 什么是LSM
一种轻量级的安全访问控制框架，主要利用Hook函数对权限进行访问控制，并在部分对象中内置了透明的安全属性。
# 2. 设计思想
Linux 系统调用接口提供了用户空间与内核进行交互的抽象，可以通过**截获系统调用**来实施访问控制。
LSM 的策略执行点刚好在对象被访问之前，如下图所示：

![](https://gitee.com/liying000/blogimg/raw/master/20191121144419367.png)

# 3. 实现原理

Linux 安全模块通过提供一系列的安全钩子函数来控制对内核对象的访问，同时通过放置在内核数据结构中的不透明安全域来维护对象的安全属性。安全域都是 void*类型的指针，一般命名为security，并用于存储安全属性，通过安全域，安全模块可以把安全属性和内核内部对象联系起来，以实施各自的安全策略。LSM修改了的内核结构如下表所示
| 内核结构        | 对象                             |
| --------------- | -------------------------------- |
| cred            | 认证或令牌                       |
| super_block     | 文件系统                         |
| inode           | 管道、文件、或套接字             |
| file            | 打开的文件                       |
| sk_buff         | 网络缓冲区(包)                   |
| net_device      | 网络设备                         |
| kernel_ipc_perm | 信号量、共享内存段、或者消息队列 |
| msg_msg         | 单个消息                         |
这些内核对象在各个安全模块中都有具体的实现
## 3.1 安全模块的注册

LSM框架下访问决策模块包括selinux，smack，tomoyo，yama，apparmor.

![](https://gitee.com/liying000/blogimg/raw/master/20191121151442404.png)

操作系统初始化 Linux 安全模块及各个安全模块的注册到 Linux 安全模块的过程如下图所示：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191122120356810.png)
其中，内核init进程的初始化在/init/main.c::start_kernel()中
安全模块的初始化函数为security_init()：

![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191121165939882.png)
security_init()函数如下，在/security/security.c中

```c
/**
 * security_init - initializes the security framework
 *
 * This should be called early in the kernel initialization sequence.
 */
int __init security_init(void)
{
	int i;
	struct hlist_head *list = (struct hlist_head *) &security_hook_heads;

	pr_info("Security Framework initializing\n");

	for (i = 0; i < sizeof(security_hook_heads) / sizeof(struct hlist_head);
	     i++)
		INIT_HLIST_HEAD(&list[i]);

	/*
	 * Load minor LSMs, with the capability module always first.
	 */
	capability_add_hooks();
	yama_add_hooks();//yama是用于控制进程追踪的安全模块
	loadpin_add_hooks();

	/*
	 * Load all the remaining security modules.
	 */
	major_lsm_init();

	return 0;
}
```
备注：在4.x的稳定版本中的security函数之间有一些差异，此处分析的是4.20版本，这个函数与5.0版本的security_init()函数相同。
首先是一些轻量的安全模块的初始化
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191122092324537.png)
将该安全模块的所有hooks函数添加到在security_hook_list的数据结构里
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191122104630622.png)
接下来是主要的安全模块的注册 major_lsm_init
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191122103700724.png)
lsm_info是一个包含 安全模块的名称与相应的init函数的结构体
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/2019112210400277.png)
从前面minor LSMs的加载以及后面major LSMs的加载可以看出，目前linux已经支持多个安全模块共同起作用。

## 3.2 Hooks函数的应用点
在 linux/include/security.h 中定义的 security_xxx 函数为 Linux 安全模块的上层框架，它们被放在内核数据结构的内核控制路径上，当某个内核对象被访问时，它们将被调用，将访问请求转发到特定安全模块的 Hooks 函数中。
以目录的创建为例分析其具体的实现过程：
在用户空间中，创建目录的命令为mkdir(),它在内核中的调用入口为sys_mkdir()
具体的目录创建函数为 vfs_mkdir()，其实现如下

```c
//dir 是父目录的inode结构，dentry是要建文件的dentry结构
int vfs_mkdir(struct inode *dir, struct dentry *dentry, umode_t mode)
{
	int error = may_create(dir, dentry);//检查是否对父目录有读写权限
	unsigned max_links = dir->i_sb->s_max_links;
	......
	error = security_inode_mkdir(dir, dentry, mode);
	//检查是否具有创建目录的权限
	......
	error = dir->i_op->mkdir(dir, dentry, mode);
	//调用具体文件系统实现函数
	......
	return error;
}
```
这个函数主要做了三件事：
（1）调用may_create()检查主体对父目录的读写权限（这部分进行DAC与capability的检查）
（2）调用 security_inode_mkdir 函数检查进程是否有创建目录的权限
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191121162845425.png)
这个插入了安全模块的钩子，这个hook会将请求转给相应的安全模块进行判断。
（3）调用底层文件系统的目录创建函数
（安全模块为selinux的情况下）
整个创建目录过程中，最终调用hooks的检查函数的过程大致如下：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191121163256610.png)