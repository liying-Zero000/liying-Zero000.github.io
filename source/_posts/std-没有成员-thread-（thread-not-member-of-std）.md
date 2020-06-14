---
title: std 没有成员 thread （thread not member of std）
date: 2020-04-07 12:59:57
tags:
	- others
---

问题描述：

系统： win10 32位

编辑器： VScode

<!--more-->

问题：在配置好includePath以及安装好mingw之后，仍然显示 std 没有成员 thread （thread not member of std），在编辑界面写thread时，不会有自动提示（这应该是有问题？），示例代码如下

```cpp
#include <iostream>       // std::cout
#include <future>
#include <thread>

void foo() 
{
  // do stuff...
}

void bar(int x)
{
  // do stuff...
}

int main() 
{
  std::thread first (foo);     // spawn new thread that calls foo()
  std::thread second (bar,0);  // spawn new thread that calls bar(0)

  return 0;
}
```

![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

错误提示：

![img](https://gitee.com/liying000/blogimg/raw/master/20200407142124670.png)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

可能的原因：

不支持C++11，因为thread类是C++11的特性，将vscode修改为默认用C++11 ，没有解决；尝试C++11 的其他新特性，成功。

说明是thread这个类的问题，在Stack Overflow上查找到相似问题，链接如下：

https://stackoverflow.com/questions/2519607/stdthread-error-thread-not-member-of-std

提到的解决方法：

mingw的问题，下载mingw为thread提供的头文件，include当前项目中，解决。

![img](https://gitee.com/liying000/blogimg/raw/master/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xpeWluZzk2,size_16,color_FFFFFF,t_70)![点击并拖拽以移动](data:image/gif;base64,R0lGODlhAQABAPABAP///wAAACH5BAEKAAAALAAAAAABAAEAAAICRAEAOw==)

内心OS：为什么配环境要出这么多问题！！我已经装过了vscode运行c++ 在windows xp上，mac上，win10上！开学用自己电脑还要再装一遍。