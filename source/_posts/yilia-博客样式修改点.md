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

  