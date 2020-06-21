---
title: SELinux进程安全上下文源代码分析
date: 2019-10-23 14:36:55
tags:
	- SELinux
---

下面的内容主要梳理安全上下文如何成为进程的属性
进程的安全上下文保存在 /proc/pid/attr

<!--more-->

![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191016111613899.PNG)



# 存储

Linux内核通过一个被称为进程描述符的task_struct结构体来管理进程，这个结构体包含了一个进程所需的所有信息。它定义在include/linux/sched.h文件中。
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191018170046416.png)
关于进程安全部分，保存在real_cred与cred两个cred的对象中
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191018170209321.png)
其中，real_cred指的是进程作为客体时的credentials
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;cred指的是进程作为主体时的credentials
在cred中，保存了各种UID与GID，以及进程拥有的capbility。
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191018170650635.png)
cred中的 secutiry指针保存了LSM的内容
![](https://gitee.com/liying000/blogimg/raw/master/20191018171248188.png)![在这里插入图片描述](https://img-blog.csdnimg.cn/20191017163913784.png?)
备注：上图画的有错误，task_struct中有一个 stack指针，指向进程内核栈的栈底。

# init进程的安全上下文
/security/selinux/hook.c::selinux_init(void)
init进程的selinux初始化函数如下：

```c
static __init int selinux_init(void)
{
	//判断boot代码中设置的安全模块是否为SELinux
	if (!security_module_enable("selinux")) {
		selinux_enabled = 0;
		return 0;
	}
	//如果不是selinux，输出提示信息“boot中设置的安全模式不是selinux”
	if (!selinux_enabled) {
		printk(KERN_INFO "SELinux:  Disabled at boot.\n");
		return 0;
	}
	/* Set the security state for the initial task. */
	//设置init进程的安全状态
	cred_init_security();

	default_noexec = !(VM_DATA_DEFAULT_FLAGS & VM_EXEC);

	sel_inode_cache = kmem_cache_create("selinux_inode_security",
					    sizeof(struct inode_security_struct),
					    0, SLAB_PANIC, NULL);
	file_security_cache = kmem_cache_create("selinux_file_security",
					    sizeof(struct file_security_struct),
					    0, SLAB_PANIC, NULL);
	//初始化avc，
	avc_init();
	security_add_hooks(selinux_hooks, ARRAY_SIZE(selinux_hooks));

	if (avc_add_callback(selinux_netcache_avc_callback, AVC_CALLBACK_RESET))
		panic("SELinux: Unable to register AVC netcache callback\n");
	
	//selinux模式
	if (selinux_enforcing)
		printk(KERN_DEBUG "SELinux:  Starting in enforcing mode\n");
	else
		printk(KERN_DEBUG "SELinux:  Starting in permissive mode\n");

	return 0;
}
```

其中 cred_init_security()函数如下：

```c
/*
 * initialise the security for the init task
 */
static void cred_init_security(void)
{
	struct cred *cred = (struct cred *) current->real_cred;
	struct task_security_struct *tsec; //进程安全信息结构tsec
	/*
	struct task_security_struct {
	u32 osid;		/* SID prior to last execve 最后一次执行前的SID
	u32 sid;		/* current SID  目前的SID
	u32 exec_sid;		/* exec SID  exec的SID
	u32 create_sid;		/* fscreate SID 创建文件系统的SID
	u32 keycreate_sid;	/* keycreate SID 创建密钥的SID
	u32 sockcreate_sid;	/* fscreate SID 创建套接字的SID
	};
	*/
	//给tsec分配内存，
	tsec = kzalloc(sizeof(struct task_security_struct), GFP_KERNEL);
	if (!tsec)
		panic("SELinux:  Failed to initialize initial task.\n");
	
	//把tsec的 osid与SID都设为SECINITSID_KERNEL
	//SECINITSID_KERNEL 为 Linux 内核中的预定义宏，值为 1
	tsec->osid = tsec->sid = SECINITSID_KERNEL; 
	/*
     * Security identifier indices for initial entities
	#define SECINITSID_KERNEL 1 
	*/  
	//把init进程的security 设为tsec                          
	cred->security = tsec;
}
```
init进程的cred初始化过程做了下面的工作：

- 定义 进程安全信息结构tsec
- 将tsec的osid与sid都设为 SECINITSID_KERNEL
- 把tsec赋值给real_cred的security部分

最终，init进程的 current->real_cred->security->osid=SECINITSID_KERNEL=1
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp; &nbsp;->sid = SECINITSID_KERNEL =1

疑问！！！！！！
为什么只定义了init进程的real_cred.....待解决。

# 安全上下文与进程关联
把一个domain与一个进程关联分为两种：
1）fork出来进程以后，然后通过传递参数的方式动态的修改新进程的domain；
2）通过exec某个binary文件启动进程，该新启动进程的domain是由其对应的binary object的context决定。

在fork过程中，最终调用_do_fork()，在前面我们已经了解过，fork过程会将父进程完全复制一份给子进程，那么父进程的 task_struct结构体指向的cred也同样是复制的

## fork过程中对进程安全内容的复制
前面我们已经了解到，进程安全部分存储在 task_struct 的 real_cred与cred 对象中，这是两个 cred 结构的对象，cred结构定义了进程安全部分的内容

在 `_do_fork_()` $\rightarrow$ `copy_process()` 中，在`dup_task_struct` 之后，进行了`cred`的复制，
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191018150655687.png)
在cred之前主要进行了以下工作：

1. 该用户是否已经创建了太多的进程，如果没有超出了resource limit，那么继续fork
2. 如果超出了resource limit，但user ID是root的话，没有问题，继续
3. 如果user ID 不是root，但是有 SYS_RESOURCE或者SYS_ADMIN的capbility的话，也可以继续fork
4. 否则fork失败

检查完成之后就正式开始copy credential，kernel/cred.c::copy_creds()

```c
int copy_creds(struct task_struct *p, unsigned long clone_flags)
{
	struct cred *new;
	int ret;

	if (
#ifdef CONFIG_KEYS
		!p->cred->thread_keyring &&
#endif
		clone_flags & CLONE_THREAD
	    ) {
		p->real_cred = get_cred(p->cred);
		get_cred(p->cred);
		alter_cred_subscribers(p->cred, 2);
		kdebug("share_creds(%p{%d,%d})",
		       p->cred, atomic_read(&p->cred->usage),
		       read_cred_subscribers(p->cred));
		atomic_inc(&p->cred->user->processes);
		return 0;
	}
...
}
```
看注释部分：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191018162545327.png)
说明子进程的real_cred与cred都是父进程的real_cred

在dup_task_struct时，已经完全复制了父进程的内容
子进程已经有了cred与real_cred，但是完全是父进程的，子进程的cred与父进程的cred相同，只需要将子进程的real_cred更新为自己cred的内容。
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/2019101816432623.png)

## exec过程中对进程安全内容的处理
系统调用exec作为入口，最终在内核中执行`do_execveat_common()`,代码如下fs/exe.c所示：

```c
/*
 * sys_execve() executes a new program.
 */
static int do_execveat_common(int fd, struct filename *filename,
			      struct user_arg_ptr argv,
			      struct user_arg_ptr envp,
			      int flags)
{
	char *pathbuf = NULL;
	struct linux_binprm *bprm; //描述可执行程序的结构
	struct file *file; 
	struct files_struct *displaced;
	int retval;
	
	retval = prepare_bprm_creds(bprm);
    bprm->file = file;
...
    retval = bprm_mm_init(bprm);
...
    retval = prepare_binprm(bprm);
...
	retval = exec_binprm(bprm);
	
```

我们关注的关于selinux的部分在这几个函数中。

prepare_bprm_creds:

创建struct cred对象，并创建struct task_security_struct对象赋予struct cred的security成员变量，并将struct cred对象与struct linux_binprm对象关联；

具体过程如下：

```flow
st=>start: 开始
e=>end: 结束
op=>operation: prepare_bprm_creds(bprm)
op2=>operation: bprm->cred = prepare_exec_creds()
op3=>operation: return struct cred new = prepare_creds()
op4=>operation: return struct cred new； step1:memcpy(new, current->cred, sizeof(struct cred)); step2:new->security=NULL


st->op->op2->op3->op4
op4->e
```
也就是说这一步主要是创建相关的结构，并给二进制可执行程序的对象关联上 cred 结构

第二步：初始化struct linux_binprm中的file成员变量，将其对应真正的文件，用于从从file映射到inode，然后从node获取object的security context，然后从security context变成security id；

第三步：

prepare_binprm，真正初始化struct cred，将其所有成员变量都对应到正确的值；

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191023144251414.png?)

其中 `security.c::security_bprm_set_creds`会调用钩子函数，

![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/201910231442045651.png)
在selinux的security/selinux/hooks.c 中，定义为：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/201910231440067511.png)
实际上对cred进行赋值的就是selinux_bprm_set_creds:

```c
static int selinux_bprm_set_creds(struct linux_binprm *bprm)
{
	const struct task_security_struct *old_tsec;
	struct task_security_struct *new_tsec;
	struct inode_security_struct *isec;
	struct common_audit_data ad;
	struct inode *inode = file_inode(bprm->file);
	int rc;

	old_tsec = current_security();  // = current->security 
	new_tsec = bprm->cred->security;
	isec = inode_security(inode); 

	/* Default to the current task SID. */ 
	//先把当前进程安全部分的secutiry部分的sid赋值到 bprm->cred>security 部分的sid跟osid中
	new_tsec->sid = old_tsec->sid;
	new_tsec->osid = old_tsec->sid;


	/* Reset fs, key, and sock SIDs on execve. */ 
	// 把bprm->cred>security 的 create_sid、keycrete_sid、sockcreate都设为0，
	//还剩下exec_sid
	new_tsec->create_sid = 0;
	new_tsec->keycreate_sid = 0;
	new_tsec->sockcreate_sid = 0;
```