---
title: '[leetcode]160两个单链表的交点'
date: 2020-06-18 23:19:24
tags:
	- Java	
	- leetcode
---

leetcode 160

<!--more-->

# 思路1:
>链表逆置，但是notes里说不要改变链表结构

停滞了...
# 思路2:
网上看思路,是跟环有关的
[网上的思路](https://wangxin1248.github.io/algorithm/2019/05/leetcode-160.html):

>a 链表与 b 链表相连，将 b 链表与 a 链表相连。这样形成两个新链表的长度是一样的，并且让这两个链表首尾相接成环。接着我们分别从两个链表的开头开始遍历链表，肯定会存在两个指针指向同一个节点的情况.当两个指针指向的内容相同时，便是相交节点或者是空节点（表示不是相交链表）

根据思路画个图:
链表1: a1 &rarr; a2&rarr; c1&rarr; c2 &rarr; c3
链表2: b1 &rarr; b2&rarr;b3 &rarr; c1&rarr; c2 &rarr; c3

链表1+链表2:
![](https://gitee.com/liying000/blogimg/raw/master/19092361-cb9fcb5469b6c8af.png)

链表2+链表1:
![](https://gitee.com/liying000/blogimg/raw/master/19092361-2b4f034da729f42f.png)
两个链表长度相等,如果存在两个指针指向同一个节点,那么这个节点就是相交节点

理想很丰满┑(￣Д ￣)┍,实现很麻烦

问题1:由于Java中的对象都是引用,链表1＋链表2之后,两个链表已经成为了下面的结果:

![](https://gitee.com/liying000/blogimg/raw/master/19092361-e79f7022155b2b43.png)

![](https://gitee.com/liying000/blogimg/raw/master/19092361-85ce0957f9a0a44b.png)


按照别人的C++写的java:

~~我还没理解......to do list~~
youtube上有个帅哥在讲:[https://www.youtube.com/watch?v=IpBfg9d4dmQ](https://www.youtube.com/watch?v=IpBfg9d4dmQ)

>是我理解错了,所以上面的图画错了.
>不是把链表连起来,是用两个指针遍历两个链表,上面那个说法是指针遍历的顺序,
>也就是说,第一个指针:
> a1 &rarr; a2&rarr; c1&rarr; c2 &rarr; c3 (key point)&rarr;b1 &rarr; b2&rarr;b3 &rarr; c1&rarr; c2 &rarr; c3 <br>第二个指针:
>b1 &rarr; b2&rarr;b3 &rarr; c1&rarr; c2 &rarr; c3 (key point)&rarr;a1 &rarr; a2&rarr; c1&rarr; c2 &rarr; c3 
>两个指针同步遍历,第一个指针遍历顺序 第一个链表＋第二个链表
>第二个指针遍历第二个链表＋第一个链表
>从这个例子来讲while循环执行8次,第九次肯定就是相交节点
>＞︿＜

```
public ListNode getIntersectionNode(ListNode headA, ListNode headB) {
        ListNode temp1 = new ListNode();
        temp1 = headA;
        ListNode temp2 = new ListNode();
        temp2 = headB;
        while (temp1!=temp2){
            if (temp1==null){
                temp1 = headB;
            }else {
                temp1 = temp1.next;
            }
            if (temp2==null){
                temp2 = headA;
            }else {
                temp2 = temp2.next;
            }
        }
        return temp1;
    }
```

还有一些其他的解法:

# 思路3:
两个链表的长度,把长的那部分截取,然后两个指针就可以同步进行了,两个指针指向相同对象,找到交点.来实现一下.
```
public ListNode getIntersectionNode2(ListNode headA, ListNode headB) {
        if (headA==null||headB==null){
            return null;
        }
        ListNode temp1 = headA;
        ListNode temp2 = headB;
        int lenA = getLength(headA);
        int lenB = getLength(headB);
        if (lenA>lenB){
            int step = lenA-lenB;
            while (step>0){
                temp1 = temp1.next;
                step--;
            }
        }else if (lenB>lenA){
            int step = lenB-lenA;
            while (step>0){
                temp2 = temp2.next;
                step--;
            }
        }
        while (temp1!=temp2){
            temp1 = temp1.next;
            temp2 = temp2.next;
        }
        return temp1;
    }
    public static int getLength(ListNode listNode){
        int len = 1;
        while (listNode!=null){
            listNode = listNode.next;
            len++;
        }
        return len;
    }
```
害,这个思路通俗易懂,连我都能把算法实现了d=====(￣▽￣*)b