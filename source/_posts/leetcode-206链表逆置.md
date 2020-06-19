---
title: '[leetcode]206链表逆置'
date: 2020-06-18 23:13:30
tags:
	- Java
---

## 思路1: 最笨的方法,链表节点全部入栈,然后出栈

<!--more-->

问题:入栈之后节点之间还是相连的,再出栈会有很多next的问题
最笨的解决方法:入栈的时候直接拷贝节点值,然后新建节点 ━┳━　━┳━
新建链表要考虑头插法还是尾插法.头插法是逆序的
```java
public ListNode reverseList(ListNode head) {
        Stack<ListNode> temp = new Stack<>();
        while (head!=null){
            temp.push(new ListNode(head.val));
            head = head.next;
        }
        ListNode result = null;
        ListNode tail = null;
        while (!temp.empty()){
            ListNode getNode = temp.pop();
            if (tail==null){
                result = getNode;
            }else{
                tail.next = getNode;
            }
            tail = getNode;
        }
        return result;
    }
```
##  思路2 原地逆置
这个应该也是可以实现的
分析问题先从普通的开始看,先不要看头和尾

node1&larr;node2&nbsp; &nbsp; node3 &rarr; node4

假设node2之前的是已经逆转完毕的
现在要逆转的是node3
用3个指针记录位置
node2&larr; pre
node3&larr; now
node4&larr; next(这个是为了记录走到哪里了,不加这个的话链表直接断掉,都不知道遍历到哪里)
![](https://gitee.com/liying000/blogimg/raw/master/19092361-af67a198d16dc47c.png)
现在node3已经逆置完成了,只需要把三个指针后移就可以了

然后考虑首尾的情况 :
- 头:pre就一个null,(注意为了处理统一,相当于这个是已经处理好的链表,所以不要跟第一个节点连起来)
- 尾:循环条件肯定是next等于null的时候退出,但是现在最后一个节点还没有逆置,所以要推出循环后再加处理
```java
public ListNode reverseList(ListNode head){
        if (head==null||head.next==null) return head;
        ListNode pre = null;
        ListNode now = head;
        ListNode next = now.next;
        while (next!=null){
            now.next = pre;
            pre = now;
            now = next;
            next = next.next;
        }
        now.next = pre;
        pre = now;
        return pre;
    }
```
## 思路3 一边遍历一遍新建另一个链表
不写了.