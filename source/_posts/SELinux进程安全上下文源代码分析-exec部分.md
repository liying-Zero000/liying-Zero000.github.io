---
title: SELinux进程安全上下文源代码分析-exec部分
date: 2019-10-24 12:33:18
tags:
	- SELinux
---

## exec过程中对进程安全内容的处理
系统调用exec作为入口，最终在内核中执行`do_execveat_common()`,代码如下fs/exe.c所示：

<!--more-->

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

![image-20200621173404283](https://gitee.com/liying000/blogimg/raw/master/image-20200621173404283.png)

也就是说这一步主要是创建相关的结构，并给二进制可执行程序的对象关联上 cred 结构

第二步：初始化struct linux_binprm中的file成员变量，将其对应真正的文件，用于从从file映射到inode，然后从node获取object的security context，然后从security context变成security id；

第三步：

prepare_binprm，真正初始化struct cred，将其所有成员变量都对应到正确的值；

![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191023144251414.png)

其中 `security.c::security_bprm_set_creds`会调用钩子函数，

![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191023144204565.png)
在selinux的security/selinux/hooks.c 中，定义为：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191023151424366.png)
实际上对cred进行赋值的就是selinux_bprm_set_creds:

终极目标是确定 bsec->sid = newsid
要么newsid等于current->cred->security->sid
要么等于一个默认域转换的相关新的domain相关的sid
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

如果当前进程存在 exec_id说明已经定义好了当前进程执行exec操作时要切换成的域，继续进行的操作如下：

```c
if (old_tsec->exec_sid) { //如果当前进程存在 exec_sid
		//bprm->cred->security的sid等于当前进程的exec_sid
		new_tsec->sid = old_tsec->exec_sid; 
		/* Reset exec SID on execve. */
		new_tsec->exec_sid = 0;  //bprm->cred->security的exec_sid=0
		...
else{
	/* Check for a default transition on this program. */
		rc = security_transition_sid(old_tsec->sid, isec->sid,
					     SECCLASS_PROCESS, NULL,
					     &new_tsec->sid);    
}
```
这一部分比较复杂，主要是通过当前进程的sid与二进制文件的sid确定 bprm的sid

![image-20200621173629798](https://gitee.com/liying000/blogimg/raw/master/image-20200621173629798.png)

分析一下security_compute_sid(old_tsec->sid,  isec->sid ,SECCLASS_PROCESS, AVTAB_TRANSITION,NULL, &new_tsec->sid, true);

```c
static int security_compute_sid(u32 ssid,  //old_tsec->sid
				u32 tsid,                  //isec->sid
				u16 orig_tclass,           //SECLASS_PROCESS
				u32 specified,             //AVTAB_TRANSITION
				const char *objname,       //NULL
				u32 *out_sid,              //&new_tsec->sid         
				bool kern)                 //true
{
	struct class_datum *cladatum = NULL;
	struct context *scontext = NULL, *tcontext = NULL, newcontext;
	struct role_trans *roletr = NULL;
	struct avtab_key avkey;
	struct avtab_datum *avdatum;
	struct avtab_node *node;
	u16 tclass;
	int rc = 0;
	bool sock;
	...
	if (kern) {
		tclass = unmap_class(orig_tclass);
		sock = security_is_socket_class(orig_tclass);
	}
	...
	//sidtab_search()函数获取old_tsec->sid和isec->sid对应的安全上下文
	scontext = sidtab_search(&sidtab, ssid);
	if (!scontext) {
		printk(KERN_ERR "SELinux: %s:  unrecognized SID %d\n",
		       __func__, ssid);
		rc = -EINVAL;
		goto out_unlock;
	}
	tcontext = sidtab_search(&sidtab, tsid);
	if (!tcontext) {
		printk(KERN_ERR "SELinux: %s:  unrecognized SID %d\n",
		       __func__, tsid);
		rc = -EINVAL;
		goto out_unlock;
	}
	...
	switch (specified) {
	case AVTAB_TRANSITION:
	case AVTAB_CHANGE:                                                
		if (cladatum && cladatum->default_user == DEFAULT_TARGET) {  
			newcontext.user = tcontext->user;
		} else {
			/* notice this gets both DEFAULT_SOURCE and unset */
			/* Use the process user identity. */
			newcontext.user = scontext->user;                
		}
		break;
	
	...

	/* Set the role to default values. */
	if (cladatum && cladatum->default_role == DEFAULT_SOURCE) {
		newcontext.role = scontext->role;
	} else if (cladatum && cladatum->default_role == DEFAULT_TARGET) {
		newcontext.role = tcontext->role;
	} else {
		if ((tclass == policydb.process_class) || (sock == true))
			newcontext.role = scontext->role;  
		else
			newcontext.role = OBJECT_R_VAL;
	}

	...
	/* Set the type to default values. */
	if (cladatum && cladatum->default_type == DEFAULT_SOURCE) {
		newcontext.type = scontext->type;
	} else if (cladatum && cladatum->default_type == DEFAULT_TARGET) {
		newcontext.type = tcontext->type;
	} else {
		if ((tclass == policydb.process_class) || (sock == true)) {
			/* Use the type of process. */
			newcontext.type = scontext->type;  
		} else {
			/* Use the type of the related object. */
			newcontext.type = tcontext->type;
		}
	}
```
过程大致如下：

- 通过current->cred->security->sid与bprm->inode->isec->sid获取当前进程的安全上下文以及二进制可执行文件的安全上下文
- 确定newcontext的user、role、type（根据一些条件判断user、role、type分别等于当前进程的，还是二进制文件的相关user、role、type）
	![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191023144006751.png)
- 确定了 bprm的newcontext，调用sidtab_context_to_sid(&sidtab, &newcontext, out_sid)函数获取newcontext对应的SID。

&new_tsec->sid即为最后一步中输出的out_sid

至此，new_tsec的sid已经确定，

下面的部分是回到selinux_bprm_set_creds()

```c
//如果不需要发生 domain transition，
//则检查new_tsec->sid对相应的二进制文件的sid是否具有execute_no_trans 权限
	if (new_tsec->sid == old_tsec->sid) {
		rc = avc_has_perm(old_tsec->sid, isec->sid,
				  SECCLASS_FILE, FILE__EXECUTE_NO_TRANS, &ad);
		if (rc)
			return rc;
	} else { //如果需要发生域转移，也就是通过计算得到了新的sid，则检查相应的权限，
		/* Check permissions for the transition. */

		//检查是否许可 domain transition
		rc = avc_has_perm(old_tsec->sid, new_tsec->sid,
				  SECCLASS_PROCESS, PROCESS__TRANSITION, &ad);
		if (rc)
			return rc;
		//检查新 domain 是否以当前可执行程序为其 entrypoint
		rc = avc_has_perm(new_tsec->sid, isec->sid,
				  SECCLASS_FILE, FILE__ENTRYPOINT, &ad);
		if (rc)
			return rc;
```

至此，新建的cred结构的security部分已经得到真正的值
第四步：exec_binprm，将新创建的struct cred对象与进程对应的task对象关联；

exec_binprm==> search_binary_handler==>load_elf_binary==>install_exec_creds（bprm）==>commit_creds

```c
/*
 * install the new credentials for this executable
 */
void install_exec_creds(struct linux_binprm *bprm)
{
	security_bprm_committing_creds(bprm);

	commit_creds(bprm->cred);
	bprm->cred = NULL;         

	/*
	 * Disable monitoring for regular users
	 * when executing setuid binaries. Must
	 * wait until new credentials are committed
	 * by commit_creds() above
	 */
	if (get_dumpable(current->mm) != SUID_DUMP_USER)
		perf_event_exit_task(current);
	/*
	 * cred_guard_mutex must be held at least to this point to prevent
	 * ptrace_attach() from altering our determination of the task's
	 * credentials; any time after this it may be unlocked.
	 */
	security_bprm_committed_creds(bprm);
	mutex_unlock(&current->signal->cred_guard_mutex);
}
```
其中，将bprm的cred部分与当前进程的cred相关联的函数是commit_creds（bprm->cred）

```c
/**
 * commit_creds - Install new credentials upon the current task
 * @new: The credentials to be assigned
 *
 * Install a new set of credentials to the current task, using RCU to replace
 * the old set.  Both the objective and the subjective credentials pointers are
 * updated.  This function may not be called if the subjective credentials are
 * in an overridden state.
 *
 * This function eats the caller's reference to the new credentials.
 *
 * Always returns 0 thus allowing this function to be tail-called at the end
 * of, say, sys_setgid().
 */
int commit_creds(struct cred *new){
	...
	rcu_assign_pointer(task->real_cred, new);
	rcu_assign_pointer(task->cred, new);
	...
}
```
当前task 的real_cred与cred都变为了bprm的cred部分。