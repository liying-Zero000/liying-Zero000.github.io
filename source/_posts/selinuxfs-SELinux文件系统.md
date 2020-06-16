---
title: 'selinuxfs:SELinux文件系统'
date: 2019-12-03 01:16:21
tags:	
	- SELinux
---

SELinux 文件系统是用户空间与 SELinux 交互的接口，在 libselinux 库中封装了许多 SELinux 文件系统的操作，需要使用SELinux 提供的安全机制的用户空间程序可以通过 libselinux 方便的完成 SELinux 相关操作。

<!--more-->

# 文件系统的挂载
selinuxfs挂载位置：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191203105531344.png)
/sys/fs/selinux目录树如下：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191203105645911.png)

## init_call函数在内核启动中的调用过程
如果在内核启动命令行中指定“selinux=1”,则在（_initcall(init_sel_fs)）中会注册并挂载SELinux文件系统
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191203152314608.png)
在 /include/linux/init.h中，涉及到的函数如下：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191203142125974.PNG)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191203142341730.png)

最终我们看到的是的：`__define_initcall(fn,id,__sec)`。
这个宏的含义是：
1) 声明一个名称为`__initcall_##fn`的函数指针；

2) 将这个函数指针初始化为fn；

3) 编译的时候需要把这个函数指针变量放置到名称为 ".initcall" id".init"的section中。
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191203145936959.png)

可以看出INIT_CALL一共有8个等级，而我们的 _initcall(init_sel_fs) 通过 device_initcall注册在 level =6 中，比较靠后，

下面我们来看看，内核是什么时候调用存储在 .initcall"id".init 的section内的函数的。
内核是通过do_initcalls函数 调用执行 .initcall"id".init ，整个函数调用链如下：
kernel 启动过程中会执行 init/main.c 的 start_kernel()，最终会调用 do_initcalls() 执行这些初始化函数：

start_kernel()->rest_init()->kernel_thread(kernel_init)->kernel_init_freeable()->do_basic_setup()->do_initcalls()->do_initcall_level()

而执行的顺序就跟各个宏的定义有关，比如 pure_initcall 实际上是 __define_initcall(fn, 0) 所以它会被首先执行，而 late_initcall 是 __define_initcall(fn, 7) 所以会最后执行。具体的执行如下：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191203151437789.png)
do_initcall_level() 会执行每个 section 内部的所有初始化函数 ：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191203151939412.png)
do_initcall_level() 按顺序遍历每个 section 执行所有函数。

两个相关的数组，initcall_levels 指向了每个 section 的起始地址，也就是第一个函数地址，initcall_level_names 则将 section 的顺序号（就是 0,1,2,3 这些）和初始化函数的含义关联起来（比如 0 对应 early ，1 对应 core）

![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191203151742801.PNG)
## init_sel_fs——selinuxfs挂载过程
整个 init_sel_fs函数的过程如下所示：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191203154119850.png)

```c
static int __init init_sel_fs(void)
{
	...
	err = sysfs_create_mount_point(fs_kobj, "selinux");
	if (err)
		return err;
	...
	err = register_filesystem(&sel_fs_type);
	if (err) {
		sysfs_remove_mount_point(fs_kobj, "selinux");
		return err;
	}

	selinux_null.mnt = selinuxfs_mount = kern_mount(&sel_fs_type);
	...
	return err;
}
```

主要有两步：

- register_filesystem(&sel_fs_type)，将sel_fs_type注册到Linux文件系统中
 sel_fs_type定义如下：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191203154534527.png)
其中sel_mount 是挂载函数，会在挂载文件系统时被调用

- 挂载文件系统，使用的是 kern_mount 函数，函数返回 vfsmount 对象selinuxfs_mount
sel_fs_type 中的 sel_mount 函数在这个挂载过程中被调用。sel_mount函数中直接调用了 mount_single 函数，  mount_single 属于 Linux 文件系统的一个功能函数，其要做的工作是，创建超级节点 super_block，并回调 fill_super()指针函数，在 SELinux 文件系统的实现中就是 sel_fill_super()函数
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191203160621496.PNG)
sel_fill_super()函数初始化文件系统超级块，创建根目录和导出文件系统。从selinux_files 数组可以看到，SELinux 文件系统创建的文件有 load，enforce，context，status，mls，disable，policyvers，policy 等，并提供了相应的文件操作集合
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191203160109737.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191203160124929.png?)
例如我们看到，对 policy 文件提供了打开、读、注销等操作.
整个 sel_fill_super所创建的文件与我们最开始 用 tree命令看到的 selinux 目录结构树相吻合。

除了上面的文件外，还会创建一些目录，有booleans，avc，class，initial_contexts,policy_capabilities,后续会在这里文件中创建对应的文件，此过程发生在加载策略配置文件 policy.X 时，如在 class 目录中会为策略中的每个客体类别都创建一个以其名称命名的类目录，并在类目录下创建对应客体类别的所有权限的文件

# 关键的功能接口
SELinux 文件系统通过创建一系列相关的文件为用户空间提供了丰富的策略管理接口，虽说数量众多，但它们的实现原理都是相似的，因此只研究与策略加载有关的load 文件操作。load 文件操作集合为
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191203161344170.png)
load 文件就提供了一个写操作，用于完成用户配置的二进制安全策略库的加载功能。
其过程如下所示：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191203171353364.png)

该函数的第一步是检查当前进程是否具有加载策略的权限：有则继续，否则退出。
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191203172450445.PNG)
接着把配置的二进制策略数据拷贝到内核空间 data。
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191203175048249.png)
调用 security_load_policy()函数加载配置的安全策略。
后续通过相关的 sel_make_policy_nodes函数在文件系统挂载时创建的 booleans、class、policycap 目录中创建相关的目录与文件
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191203175312633.png)
其中，sel_make_policy_nodes如下所示：
![在这里插入图片描述](https://gitee.com/liying000/blogimg/raw/master/20191203175411654.png)
load 文件提供的 sel_write_load 接口在系统中扮演着重要的角色，Linux 系统启动时，init 进程将初始化 SELinux，此时，策略文件并没有被加载，因此只能通过预定义的客体“初始安全标识”来描述相应内核设施的初始安全属性（在前面sid）。

比如 init 进程在装再policy.X（X 为版本号）策略文件之前，使用的便是初始的安全标识“kernel”，此时它处于 kernel_t 类型上（初始化函数 selinux_init 中调用 cred_init_security 函数设置）。

之后 init 进程通过 load 文件提供的策略加载功能接口加载策略文件 policy.X，在部署SELinux 环境时，通过 restorecon 命令给整个文件系统上所有正规文件和目录打标签。

这样，SELinux 的安全策略才能真正起作用，init 进程在加载了策略库后会重启自己，并且依策略库中的定义的策略规则转换为 init_t 类型。