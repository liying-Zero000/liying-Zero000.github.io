---
title: github.io+reveal.js
date: 2020-07-07 00:23:22
tags:
	- others
---

起源：之前就看到过reveal.js做的个人简历，当时是一个技术博客，想着一定要给自己加上。

我的slides链接：

[my-slides](https://liying-zero000.github.io/slides/)

参考：

http://jr0cket.co.uk/2014/01/share-your-revealjs-slides-on-github-pages.html

如何用 Github 免费在线播放你的幻灯？ - 王树义的文章 - 知乎 https://zhuanlan.zhihu.com/p/77840879

（上边这位老师我本科毕设的时候开始关注的……,会分享很多实用的自然语言处理教程）



大体步骤：

- github中新建一个repo，pull到本地
- 下载reveal.js相关文件
- 把reveal.js的js、css、dst、assets(这个存图跟media吧)拷贝到repo目录下，在reveal.js中随便找一个示例html放到repo主目录
- git push
- repo的setting 中打开github pages 页面



注意点：

因为我自己开了 github用户名.github.io ，一直想的是怎样加到博客中，尝试了第一个参考链接的在github.io中加入一个gh-pages分支的方法，但是建好之后由于切换分支以及页面问题，放弃了这个思路。

无论你是否建立了基于用户名的github.io博客，都建议新建一个repo，存reveal.js生成的slides，可以在github.io博客中增加一个post，内容为slides的链接（或者把slides的repo的read m内容写为所有slides 的链接）。



下一步：如何使用markdown编写slides，生成reveal.js的html页面？