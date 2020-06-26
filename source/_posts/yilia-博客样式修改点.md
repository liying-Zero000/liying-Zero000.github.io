---
title: yilia 博客样式修改点
date: 2020-06-14 22:31:55
tags:
	- 随笔
---

博客主题：yilia

<!--more-->

- 头像上侧图片

  

- 添加栏目

  大致就是新建了一个md保存具有相应tag的文件，最终所有文件还是存在_post中。

- ~~左侧显示总文章数~~ 太丑了被我删掉了

  将themes\yilia\layout_partial\left-col.ejs文件的

  ```js
  <nav class="header-menu">
      <ul>
      <% for (var i in theme.menu){ %>
          <li><a href="<%- url_for(theme.menu[i]) %>"><%= i %></a></li>
      <%}%>
      </ul>
  </nav>
  ```

  后面加上

  ```js
  <nav>
      总文章数 <%=site.posts.length%>
  </nav>
  ```

- 社交分享按钮更改

  https://blog.csdn.net/qq_40922859/article/details/100878573

- 字数统计

  https://blog.csdn.net/fmk1023/article/details/100073217

- Typora+PicGo+Gitee 搭建图床

  https://www.cnblogs.com/focksor/p/12402471.html

- 修改代码块为Mac的panel

  参考链接：[http://oltremare.cc/2019/09/20/%E7%A7%81%E4%BA%BA%E8%AE%A2%E5%88%B6-%E4%B8%BAYilia%E4%B8%BB%E9%A2%98%E4%BB%A3%E7%A0%81%E5%9D%97%E6%B7%BB%E5%8A%A0Mac-Panel%E6%95%88%E6%9E%9C/](http://oltremare.cc/2019/09/20/私人订制-为Yilia主题代码块添加Mac-Panel效果/)

  https://blog.ihoey.com/posts/Hexo/2018-05-27-hexo-code-block.html

  主要做法：在yilia文件夹下创建scripts文件夹，新建codeblock.js文件，代码内容就是上边两个写的那些

  在Yilia主题路径\source-src\css下，新建css文件，文件名为code-block.scss(第一个写错了，他写的css)。上面第一个链接写的代码：

  ```scss
  .highlight-wrap[data-rel] {
    position: relative;
    overflow: hidden;
    border-radius: 5px;
    margin: 0px 0;
    ::-webkit-scrollbar {
      height: 5px;
    }
    ::-webkit-scrollbar-track {
      -webkit-box-shadow: inset 0 0 6px rgba(0, 0, 0, 0.3);
      border-radius: 10px;
    }
  
    ::-webkit-scrollbar-thumb {
      border-radius: 10px;
      -webkit-box-shadow: inset 0 0 6px rgba(0, 0, 0, 0.5);
    }
    &::before {
      color: white;
      content: attr(data-rel);
      height: 38px;
      line-height: 38px;
      background: #21252b;
      color: #fff;
      font-size: 16px;
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      font-family: 'Source Sans Pro', sans-serif;
      font-weight: bold;
      padding: 0px 70px;
      text-indent: 15px;
      float: left;
    }
    &::after {
      content: ' ';
      position: absolute;
      -webkit-border-radius: 50%;
      border-radius: 50%;
      background: #fc625d;
      width: 12px;
      height: 12px;
      top: 0;
      left: 20px;
      margin-top: 13px;
      -webkit-box-shadow: 20px 0px #fdbc40, 40px 0px #35cd4b;
      box-shadow: 20px 0px #fdbc40, 40px 0px #35cd4b;
      z-index: 3;
    }
  }
  ```

  然后在Yilia主题路径\source-src\css下的main.scss引入codeblock

  ```scss
  @import "./codeblock";
  ```

  修改同路径下的hilight.scss

  ```scss
  .article-entry .highlight {
    background: #272822;
    margin: 35px 0;
    padding: 10px 10px;
    overflow: auto;
    color: #272822;
    font-size: 0.9em;
    line-height: 35px;
  }
  ```

  

  打包过程中碰到的问题：

  解决npm install 报错 check python checking for Python executable python2 in the PATH：

  [https://blog.csdn.net/qq_42886417/article/details/103123659](https://blog.csdn.net/qq_42886417/article/details/103123659)

  打包步骤：

  ![](https://upload-images.jianshu.io/upload_images/19092361-4d45b8a21eb0e827.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  [node-sass安装失败解决](https://blog.csdn.net/liul99/article/details/95603254?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase)

  疯球了，这样子我孙子来看我博客的时候那些依赖包都不服务了咋办。。几十年呢！！！



- 错误修改

  ![image-20200626005027694](C:\Users\liyin\AppData\Roaming\Typora\typora-user-images\image-20200626005027694.png)

  next 页面有问题，查了一下source

  这个应该是右箭头的意思吧，，找一下在哪里生成

  ![image-20200626010302703](C:\Users\liyin\AppData\Roaming\Typora\typora-user-images\image-20200626010302703.png)

  如果是最后一页的话，Prev前面的左箭头标识就不对了。