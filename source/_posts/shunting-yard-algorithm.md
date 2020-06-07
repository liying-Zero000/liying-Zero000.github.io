---
title: shunting yard algorithm
date: 2020-06-03 01:34:00
tags:
    - Java
---

为什么要看这个？
答：写一个24点的小demo，需要得到中缀表达式的计算结果。

## 后缀表达式计算过程
后缀表达式特点：
- 操作符置于被操作数之后
- 不需要括号，不需要定义优先级，从左到右计算即可。

计算过程：
从左到右扫描，操作数保存，遇到一个操作符，停下来计算该操作符需要的数字计算。得到计算结果后返回序列。
代码：
>没有做错误处理，也没有考虑单目运算符。
```
public static int PostfixToValue(String string) {
		Stack<Integer> number = new Stack<>();
		for(int i = 0;i<string.length();i++) {
			if (string.charAt(i)>='0'&&string.charAt(i)<='9') {
				number.push(Character.getNumericValue(string.charAt(i)));
			}else {
				if(number.size()>1) {
					int second = number.pop();
					int first = number.pop();
					//只算加减乘除
					switch (string.charAt(i)) {
					case '+':
						number.push(first+second);
						break;
					case '-':
						number.push(first-second);
						break;
					case '*':
						number.push(first*second);
						break;
					case '/':
						number.push(first/second);
						break;
					default:
						break;
					}
				}
			}
		}
		return number.pop();
	}
```
## 中缀表达式转后缀表达式——调度场算法
计算机中计算的是后缀表达式，那么如何把中缀表达式转换为后缀表达式？
┓( ´∀` )┏好多教程写的都是过程，突然找到一个总结，这个转换有专有名词——调度场算法。

版本1-不带括号的计算，算法思路：

- 数字直接入栈Postfix栈(也可以用队列，或者List，总之是用来保存结果的)
- 扫描到操作符入操作符栈Operation
（1）如果Operation栈空，直接入栈
（2）栈不空，比较当前操作符与栈顶操作符的优先级
    --------当前操作符优先级高：入栈
    --------否则：Operation栈顶元素出栈，入Postfix栈，重复（2）

最后，把Operation栈中所有元素出栈，按出栈顺序入Postfix栈。

一个利于理解的图：
![](https://liam.page/uploads/images/ACS/Shunting_yard.png)
```
public static String infixToSufix(String string) {
		Stack<Character> operation = new Stack<>();
		Stack<Character> postfix = new Stack<>();
		for(int i = 0;i<string.length();i++) {
			if (string.charAt(i)>'0'&&string.charAt(i)<='9') {
				postfix.add(string.charAt(i));//如果是数字直接入栈
			}else {
				if (operation.empty()) {//操作符栈为空
					operation.push(string.charAt(i));
				}else {//操作符栈不空
					if (Compare(string.charAt(i), operation.peek())) {//当前的操作符比栈顶的操作符优先级高
						operation.add(string.charAt(i));
					}else {
						while(!operation.empty()&&!Compare(string.charAt(i), operation.peek())) {//跟栈顶操作符出栈，一直比较，直到当前的操作符优先级高
								postfix.add(operation.pop());
						}
						operation.add(string.charAt(i));
					}
				}
			}
		}
		while(!operation.empty()) {
			postfix.add(operation.pop());
		}
		return postfix.toString();
	}

public static boolean Compare(char first, char second) {
		//大于0说明前边的char优先级更高
		return definepriority(first)-definepriority(second)>0;
	}
public static int definepriority(char c) {
        int priority = 0;
		switch (c) {
		case '+':
			priority = 1;
			break;
		case '-':
			priority = 1;
			break;
		case '*':
			priority = 2;
			break;
		case '/':
			priority = 2;
			break;
		default:
			break;
		}
		return priority;
	}
```
那么问题来了：
加括号的怎么处理？

算法版本升级——加括号的表达式：
思路：

- 数字直接入栈Postfix栈(也可以用队列，或者List，总之是用来保存结果的)
- 扫描到操作符入操作符栈Operation
（1）如果栈为空，或者当前符号为‘ ( ’，或者栈顶为' ( '，直接入栈。
（2）栈不空，比较当前操作符与栈顶操作符的优先级
--------如果当前符号是' ) '，Operation栈顶元素出栈，入Postfix栈，直到取出的操作符是' ( '为止。
    --------当前操作符优先级高：入栈
    --------否则：Operation栈顶元素出栈，入Postfix栈，重复（2）


最后，把Operation栈中所有元素出栈，按出栈顺序入Postfix栈。
```
if (operation.empty()||string.charAt(i)=='(') {//操作符栈为空或者是左括号
					operation.push(string.charAt(i));
				}else {//操作符栈不空
					if (string.charAt(i)==')') {
						while(operation.peek()!='(') {
							postfix.add(operation.pop());
						}
						operation.pop();//把左括号pop出来扔掉
					}else {
						if (Compare(string.charAt(i), operation.peek())) {//当前的操作符比栈顶的操作符优先级高
							operation.add(string.charAt(i));
						}else {
							while(!operation.empty()&&!Compare(string.charAt(i), operation.peek())) {//跟栈顶操作符出栈，一直比较，直到当前的操作符优先级高
									postfix.add(operation.pop());
							}
							operation.add(string.charAt(i));
						}
					}
				}
```
只修改了操作符栈的部分，多加了一个是否为右括号的分支。
在添加的时候，如果是左括号直接入栈，所以没有定义左括号的优先权。

>问题：为什么括号这么操作就是对的呢？

我自己的理解：括号因为优先级最高，所以可以看做是把括号中的操作看做是另一个中缀表达式转后缀表达式。如下图所示：
![](https://img.alicdn.com/tfs/TB12TZZbbvpK1RjSZFqXXcXUVXa-672-414.png_640x640.jpg)
图片来自：[boycgit.github.io](https://boycgit.github.io/algorithm-shunting-yard/)

括号里的是一个子栈。

遗留问题：
- String中计算的数字是小数，或者是几位数（这个应该是处理String）
- 单目运算符的问题（需要思考一下）

2020.6.3 一天就看这个了，自己弄明白，完全自己写算法。还做了一百道牛客的选择题。
P.s 为什么大学的时候从来不选计算器当课设……要不然这个肯定就会了呢φ(>ω<*) 


