---
title: '微软的CFI实现:CFG&RFG'
date: 2020-06-12 00:55:23
tags:
	- others
---

前向：CFG

后向：RFG

<!--more-->

## CFG(Control Flow Guard)

前向CFI的实现：调用函数之前检查函数的有效性


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