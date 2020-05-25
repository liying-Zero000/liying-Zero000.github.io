---
title: SELinux命令行操作
date: 2019-08-25 23:27:16
tags:
    - SELinux
---
@[toc](SELinux的策略与规则管理相关命令)

下面的命令中的policy（如果有的话）参数都是policy路径，否则就是默认分析内核中的，policy可以是以下两种文件中的一种
(下面是manpage的说法，版本不一定是正确的，《selinux学习笔记》中提到过版本最好不要超过内核中的版本，
疑问：***如果是自己编写的策略文件，版本号是直接指定的嘛***？)

- 源文件：
   包含单一策略源的单个文本文件。 该文件通常名为policy.conf。
- 二进制文件：
  包含二进制策略的单个文件。 此文件通常以Linux系统上的版本命名，例如policy.30。 此文件通常在Android系统上命名为sepolicy。

# 1.seinfo
得到当前规则库中各种SELinux语法使用情况的摘要信息
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190823140910418.png)
分析某个编译好的policy文件，在seinfo后面加上policy文件的路径
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190823141053377.png)
## （1）seinfo -u 
查看策略中的user信息
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190823141300735.png)
### seinfo -uXXX -x
查询某个linux用户关联了哪些selinux的user以及role
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019082314171195.png)
## （2）seinfo -r
查询策略的role信息
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190823142111574.png?)
### seinfo -rXXX_t -x
查询某个角色关联的类型信息
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190823142424200.png?)
## （3）seinfo -tXXX_t -x
查询一个type所属于的所有domain的信息
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190823144212309.png?)
## （4）seinfo -b
查询布尔值的名字，布尔值详细内容由semanage给出
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190823154419270.png?)
# 2.sesearch
精细地查找一条具体的SELinux规则
语法：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190823150054735.png)
其中
-A 并不是表示all的意思，而是
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190823151935977.png)
两个permission属性的集合
## sesearch [-A/--allow/...] -s [name] -t[name]
s跟t可以同时指定。

从下面的图可以看出，s(source)跟t(target)并不是alllow规则中的主体跟客体的对应，**而是具有source指定的名称作为类型或者属性的意思（存疑）**，如果要精确查找某些allow规则中的主体或者客体是某个名称的规则， 用正则表达式 |grep name（当然上面的有些命令也可以用正则表达式。）

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190823145542826.png?)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190823153516242.png)
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019082315364738.png)
查看某一对主体&客体产生的日志（如果被标记为dontaudit也是可以检索出来）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190825231137447.png)
## sesearch -T
查询域转移的命令
sesearch -T命令会显示系统中所有的转移命令，包括域转移，角色转移等
举例如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191223153854719.png)
其中：
 -s 参数表示要发生迁移的域
 -t 参数表示的是在迁移点（注意不是迁移后的域）
# 3.semanage
查询与修改SELinux默认目录的安全上下文
semanage可管理的内容很多，具体有：
## import
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190823173740952.png)
不是很重要，略
## export
略
## login

该属性用于定义等Unix用户登录系统后，所得到的的对应SElinux User，以及context标记。
### semanage login -l
-l, --list            List records of the login object type
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190823173952587.png)
这个命令对于那些权利分离的系统很有用（例如红旗安全操作系统，三权分离，直接管理登录的selinux用户）
## user
控制SELinux用户角色和MLS / MCS级别之间的映射。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190823151321280.png?)
## semanage user -a
Modify groups for staff_u user
下面这条命令的意思是给staff_u这个selinux的user增加三个角色。
 #semanage user -m -R "system_r unconfined_r staff_r" staff_u
 Add level for TopSecret Users
 #semanage user -a -R "staff_r" -rs0-TopSecret topsecret_u
 使用**semanage user**创建**Selinux User**的命令：
 

```
semanage user -a -R user_r -P user -r s0-s5:c1.c20 -L s0 chuzhang_u
```
其中，-R是selinux Roles，-P是selinux Prefix（selinux 前缀，链接到selinux 用户id，并反映用户角色），-L 是selinux缺省的level，-r是MLS/LCS 的范围
注意：-r和-L两个值的设定，对**semanage login**的 -r参数是有影响的，其值不能超过**semanage user** 中定义的某个**Selinux User** 的MCS范围
注意：给semanage user设定-L参数，可能会引发许多意向不到的问题，建议设置时保留其默认值。

semanage user与semanage login设定的selinux user冲突，解决方法是：
删除掉semanage user 设置的selinux user，重新设置。

## port
定义了一些针对某个服务的类型（Type）所使用的端口。除已定义的属性端口外，该服务就不能使用额外的端口
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190825190317341.png)

## interface
对应接口对象对应的context

## fcontext（重要）
### semanage fcontext -l 
查询所有的默认安全上下文
一旦对文件的安全上下文进行了修改，就可以使用 restorecon 命令进行恢复，因为默认安全上下文已经明确定义了
**问题：对于一些新建文件或者复制文件，安全上下文是如何默认的？**
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019082316291573.png?)

```
思考：对于后期维护selinux的人员来说，可能需要查看某个主体或者客体对应的资源是哪些，
这时候用semanage fcontext -l 就很方便，
另外，如果想要开发管理工具，可以调用semanage fcontext显示出相应的type对应的文件是哪些。例子见下：
```
### semanage fcontext -l |grep XXX
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190828152726519.png?)
restorecon举例：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190823164116721.png)
### semanage fcontext -a/-d 
-a表示增加安全上下文
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190823165241282.png)
这个语句的意思是给/test目录下所有以 . 开头的文件增加“httpd_sys_content_t”的type
但是manpage中说的是**增加或者删除一条record**，需要实际操作一下。
-d表示删除安全上下文
同理。
## boolean
semanage boolean -l
长格式查看所有布尔变量
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019082518291582.png?)

```
/***
布尔值可以理解为是把一组规则（rules）合并起来，定义一个名称，以便在运行时可修改，实时生效，
以满足某些许可的要求。
***/
```
修改bool值
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190825184917545.png?)
## permissive
selinux中可以单独将某些类型设置为permissive，也就是说，整个系统运行在enforce模式时，此类型是permissive的
semanage permissive -l是列出当前策略中所有permissive类型
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191223151957800.png)
semanage permissive -a XXX_t
将XXX_t设置为permissive模式
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191223152529309.png)
实质上进行的操作应该是编译了一个pp文件，该文件的内容是将XXX_t设置为permissive
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191223152649895.png)
semanage permissive -d XXX_t
从系统策略中删除某一个permissive的type可以用semanage permissive -d XXX_t这个命令
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191223152934557.png)
从返回结果看，这是讲我们看到的permissive-XXX_t删除，二者是一样的效果
（semodule -r permissive_sandbox_min_t）
现在系统中不存在permissive的type
## dontaudit
设定某个规则访问或者拒绝访问之后是否记录。
## ibendport
# 4.setsebool
## setsebool XXX on/off
设定某个布尔值打开或者关闭
# 5.getsebool
获取某个布尔值的状态
## getsebool XXX

# 6.chcon
chcon [选项]... [-u 用户] [-r 角色] [-l 范围] [-t 类型] 文件
比较有用的部分是这个命令，修改某个文件的安全上下文
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190823164437455.png)
用法：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190823164239876.png?)
# 日志
系统会对主体访问客体的动作是否允许进行判断，当拒绝发生时，除规则动作定义为dontaudit之外，会生成AVC消息记录在审计日志 /var/log/audit/audit.log文件，当允许发生时，只有匹配规则动作为auditallow的消息才写入日志。

## 日志字段
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190825223804190.png)
- type 
前缀是AVC，标识出这条消息是一条AVC消息，其它类型包括USER_AUTH，LOGIN，SYSCALL，和PATH
- msg
时间戳：从基准时间依赖的以纳秒为单位的时长
序列号：1770，用来标识由相同事件产生的多个相关的审核消息，例如，一个事件可以产生两个系统调用和AVC审核消息，这些消息的序列号是相同的
- avc
标识出审计消息时来自允许还是拒绝访问，以及允许或拒绝的许可，关键词denied标识这个消息来自访问被拒绝，允许访问的字段是granted
- scontext
主体的上下文
- tcontext
客体的上下文
- tclass
## 辅助信息
对于特定的客体类别，例如，与文件有关的客体类别的审计消息通常包括客体的inode号，与网络有关的客体类别的审计消息通常包括IP地址和端口号

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190825225923604.png?)
查看日志服务的信息
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190825230434455.png?)
## 日志命令
### ausearch
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190825230726458.png?)
还可以加入时间等命令
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190825230650809.png)
### audit2why
从审计日志得到一些解决建议。
### audit2allow
只能对日志中出现的一些拒绝访问给出建议，对于MLS/MCS模式下造成的拒绝访问不能给出建议。



