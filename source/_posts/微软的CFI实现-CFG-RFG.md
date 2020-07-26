---
title: '微软的CFI实现:CFG&RFG'
date: 2020-06-12 00:55:23
tags:
	- others
	- CFI
---

前向：CFG

后向：RFG

<!--more-->

## CFG(Control Flow Guard)

前向CFI的实现：调用函数之前检查函数的有效性

微软的CFG实现主要集中在间接调用保护上。
### VS打开CFG选项

程序编译过程：
(.c) -> 预处理器 cpp (.c) -> 编译器 cc1 (.s) -> 汇编器 as (.o) -> 链接器 ld -> (.exew)

Visual Studio 开启CFI
Project | Properties | Configuration Properties | C/C++ | Code Generation and choose Yes (/guard:cf)  打开CFI选项
![](https://upload-images.jianshu.io/upload_images/19092361-72cdd71878a122a8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
编译时加入命令行参数：
![](https://upload-images.jianshu.io/upload_images/19092361-a6efa218faf87072.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
linker选项也要加上
![image.png](https://upload-images.jianshu.io/upload_images/19092361-4ce5ee43300c2c5b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>疑问：把编译器关掉再打开就没有这些配置了？
>解决了。Code Generation选了CFI之后没有点应用。

Visual Studio如何生成汇编代码？
Project | Properties | Configuration Properties | C/C++|Output Files 选中下图内容
![](https://upload-images.jianshu.io/upload_images/19092361-c6d9fc08294db8d3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在项目中用release运行代码，文件目录如下：

![](https://upload-images.jianshu.io/upload_images/19092361-a147223e8b7cc0c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
直接用Visual Studio打开asm文件。

下面是分析：
代码来自[TrendMicro](http://sjc1-te-ftp.trendmicro.com/assets/wp/exploring-control-flow-guard-in-windows10.pdf)
```
#include <iostream>
#include <tchar.h>

typedef int(*fun_t)(int);
int foo(int a)
{
	printf("hello jack:%d\n",a);
	return a;
}
class CTargetObject
{
public:
	fun_t _fun;
};
int _tmain(int argc, _TCHAR* argv[])
{
	int i = 0;
	CTargetObject* o_array = new CTargetObject[5];
	for (i = 0; i < 1000; i++)
	{
		o_array[i]._fun = foo;
	}
	o_array[0]._fun(1); // <- indirect call
	return 0;
}
```
开启CFG选项之后的编译结果：
![](https://upload-images.jianshu.io/upload_images/19092361-ec13373bfefefd8c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

>截图是因为VS的字体好看(～￣▽￣)～

未开启CFG的编译结果：
>tmd没人运行过吗？i<1000？？？
>改了。i<5
>(╬◣д◢)不改：内存问题导致编译错误。改了，内存没问题编译不出来理想结果QAQ

![](https://upload-images.jianshu.io/upload_images/19092361-ea6566f68669598e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


简介到此为止。
### 如何检查程序是否打开了CFG？
Run the [dumpbin tool](https://docs.microsoft.com/cpp/build/reference/dumpbin-reference) (included in the Visual Studio 2015 installation) from the Visual Studio command prompt with the */headers* and */loadconfig* options: **dumpbin /headers /loadconfig test.exe**.
Visual Studio 2019下，dumpbin.exe在\VC\Tools\MSVC\14.26.28801\bin\Hostx86\x64中，执行命令行执行就可以，后面test.exe是要检查的二进制可执行程序。
红色框中显示CFG开启。
![image.png](https://upload-images.jianshu.io/upload_images/19092361-284d2b50ffe08701.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从下图可以看出，现在的CFG更加复杂。
![image.png](https://upload-images.jianshu.io/upload_images/19092361-81abb25557e4f384.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
下面是MicroSoft官网中 2015年的CFG保护。
![image.png](https://upload-images.jianshu.io/upload_images/19092361-f5d7c69059f5090f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

IDA中：004020DC
>P.s 第一次知道有ida这个软件，太神奇了ヽ(ﾟ∀ﾟ)ﾒ(ﾟ∀ﾟ)ﾉ 

![](https://upload-images.jianshu.io/upload_images/19092361-bddec9a9f350228b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
双击跟过去那个检查函数是没有代码的。


0040210C 
Guard CF function table：RVA列表的指针，其包含了程序的代码。每个函数的RVA将转化为 CFGBitmap中的“1”位。CFGBitmap 的位信息来自Guard CF function table。
Guard CF function count： RVA的个数。
![image.png](https://upload-images.jianshu.io/upload_images/19092361-0816714e7539297a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

[https://www.giantbranch.cn/2019/10/28/CFG%E9%98%B2%E6%8A%A4%E6%9C%BA%E5%88%B6%E7%AE%80%E5%8D%95%E5%AE%9E%E8%B7%B5%E4%B8%8E%E4%BB%8B%E7%BB%8D/](https://www.giantbranch.cn/2019/10/28/CFG%E9%98%B2%E6%8A%A4%E6%9C%BA%E5%88%B6%E7%AE%80%E5%8D%95%E5%AE%9E%E8%B7%B5%E4%B8%8E%E4%BB%8B%E7%BB%8D/)

>┓( ´∀` )┏我真的没找到ntdll!LdrpValidateUserCallTarget



### 原理分析
CFG检查基于CFGBitmap，它表示在进程空间内**所有函数的起始位置**。在进程空间内每8个字节的状态对应CFGBitmap中的一位。如果函数的地址是合法有效的，那么这个函数对应在CFGbitmap的位置会被设置位1，否则是0。 一个Bitmap的大小是4字节


win10 edge 的CFG
![](https://upload-images.jianshu.io/upload_images/19092361-de8ffc8d1f5a925f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](https://upload-images.jianshu.io/upload_images/19092361-4d5bfd4bd1689da5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

CFG的验证过程。
它会以我们将要跳转的目标函数地址作为自己的参数，并进行以下的操作：

1.访问CFGbitmap，它表示在进程空间内所有函数的起始位置。在进程空间内每8个字节的状态对应CFGBitmap中的一位。如果函数的地址是合法有效的，那么这个函数对应在CFGbitmap的位置会被设置位1
2.进行函数地址校验，会把这个要校验的地址通过计算转换为CFGbitmap中的一位。
提取目标地址对应位的过程如下：

- 取目标地址的高24位作为索引i；
- 将CFG Bitmap当作32位整数的数组，用索引i取出一个32位整数bits；
- 取目标地址的第4至8位作为偏移量n；
- 如果目标地址不是0x10对齐的，则设置n的最低位；
- 取32位整数bits的第n位即为目标地址的对应位。

计算过程举例：
```
Mov ecx, 0x00b613a0 
Mov esi , ecx  
Call _guard_check_icall_fptr 
call esi
```
上面这段汇编代码，间接跳转的地址是：0x00b613a0  。把这个地址放到了ecx和esi中，然后调用_guard_check_icall_fptr函数。
在这个函数中，操作如下：


目标地址：
0x00b613a0 = ‭00000000 10110110 00010011 10100000‬ (b)
那么首先取高位的3字节：00000000 10110110 00010011 = 0x00b613
CFGbitmap的基地址加上0x00b613就是指向一个字节单元的指针
这个指针最终取到的假设是：
CFGBitmap ：0x10100444 = 0001 0000 0001 0000 0000 0100 0100 0100(b)
接着判断目标地址是否以0x10对齐：地址 & 0xf 是否等于 0 ；
- 如果等于0，那偏移是：最低字节二进制位的前5位，这里就是 10100；
- 若不等于0，则偏移是： 10100|0x1。

这里的例子，0x00b613a0 & 0xf = 0 ，所以偏移： 1010 0 = 20 (d)
这个偏移就是该函数在CFGbitmap中第20位的位置上，若这个位置是1，说明该函数调用是合法有效的，反之则不是。
那么：
CFGBitmap ：0x10100444 = 0001 0000 0001 0000 0000 0100 0100 0100(b)
第20位加粗的 1 ，说明这个调用合法有效。
上述例子参考[https://xz.aliyun.com/t/2587](https://xz.aliyun.com/t/2587)

整体流程如图所示：

![CFG防护原理示意图](https://upload-images.jianshu.io/upload_images/19092361-99b3cba0d51b0143.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


上面这些工作是编译器的工作，剩下的是操作系统的支持。

现在，我们已经有了CFG工作机制的基本认识。但是这个机制带来了下面的问题：

1. CFGBitmap的位信息来自哪里？

2. 何时且怎么生成CFGBitmap？

3. 系统怎么处理不可靠的间接调用触发的异常？
 答：直接中断程序
![image](https://upload-images.jianshu.io/upload_images/19092361-379c0865019bb36a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

1. 在OS引导阶段，第一个CFG相关的函数是MiInitializeCfg。这个进程是system。调用堆栈如下：
![Call stack](https://upload-images.jianshu.io/upload_images/19092361-8ccf6f9422212ace.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


相关文章：
[【技术分享】探索Windows 10的CFG机制](https://www.anquanke.com/post/id/85493)

[https://www.giantbranch.cn/2019/10/28/CFG%E9%98%B2%E6%8A%A4%E6%9C%BA%E5%88%B6%E7%AE%80%E5%8D%95%E5%AE%9E%E8%B7%B5%E4%B8%8E%E4%BB%8B%E7%BB%8D/](https://www.giantbranch.cn/2019/10/28/CFG%E9%98%B2%E6%8A%A4%E6%9C%BA%E5%88%B6%E7%AE%80%E5%8D%95%E5%AE%9E%E8%B7%B5%E4%B8%8E%E4%BB%8B%E7%BB%8D/)

[http://www.panshy.com/articles/201508/security-2512.html](http://www.panshy.com/articles/201508/security-2512.html)

[https://xz.aliyun.com/t/2587](https://xz.aliyun.com/t/2587)

[https://www.anquanke.com/post/id/85493](https://www.anquanke.com/post/id/85493)

[https://lucasg.github.io/2017/02/05/Control-Flow-Guard/](https://lucasg.github.io/2017/02/05/Control-Flow-Guard/)


## RFG（Return Flow Guard）
后向CFI的实现：函数返回之前检查返回地址的有效性

关于RFG，[腾讯玄武安全实验室做出了详细的分析](https://xlab.tencent.com/cn/2016/11/02/return-flow-guard/#more-199)。就是一种影子栈思想，把返回地址保存到影子栈中，只不过这个影子栈启用了FS寄存器，是一种软件实现。
Intel的控制流完整性技术（CFI的硬件实现）的原理也是影子栈，但是是修改Call指令，返回地址同时存入传统栈与影子栈。
而微软的RFG是在汇编时加入额外的指令，是一种软件实现。
![](https://gitee.com/liying000/blogimg/raw/master/19092361-25f24eee4d660307.png)
图片来自[https://eyalitkin.wordpress.com/2017/08/18/bypassing-return-flow-guard-rfg/](https://eyalitkin.wordpress.com/2017/08/18/bypassing-return-flow-guard-rfg/)




>P.s2018年以后RFG已经弃用了。[https://stackoverflow.com/questions/54945409/return-flow-guard-implementation-in-windows-sdk-10](https://stackoverflow.com/questions/54945409/return-flow-guard-implementation-in-windows-sdk-10)



问题：
为什么要在函数头添加指令？（我就是看不懂汇编QAQ）