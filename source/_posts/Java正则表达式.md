---
title: Java正则表达式
date: 2020-04-20 10:05:41
tags:
	- Java
---

部分字符总结：

| 元字符 | 匹配对象 | 匹配单个字符的元字符 |
| ------ | -------- | -------------------- |
|        |          |                      |



| .      | 点号         | 匹配单个任意字符                                             |
| :----- | :----------- | :----------------------------------------------------------- |
| [···]  | 字符组       | 匹配单个列出的字符                                           |
| [^···] | 排除型字符组 | 匹配单个未列出的字符                                         |
| \char  | 转义字符     | 若 char 是元字符，或转义序列无特殊含义时，匹配 char 对应的普通字符 |

<!--more-->

提供计数功能的字符

| ？        | 问号     | 容许匹配一次，但非必须             | <=1              |
| :-------- | :------- | :--------------------------------- | :--------------- |
| *         | 星号     | 可以匹配任意多次                   | >=0              |
| +         | 加号     | 至少需要匹配一次，至多可能任意多次 | 1<= 次数<= ∞     |
| {min,max} | 区间量词 | 至少需要 min 次，至多容许 max 次   | min<= 次数<= max |

匹配位置的元字符

| ^    | 脱字符     | 匹配一行的开头位置 |
| :--- | :--------- | :----------------- |
| $    | 美元符     | 匹配一行的结束位置 |
| \<   | 单词分界符 | 匹配单词的开始位置 |
| \>   | 单词分界符 | 匹配单词的结束位置 |

其他元字符

| \|        | alternation | 匹配任意分隔的表达式                                         |
| :-------- | :---------- | :----------------------------------------------------------- |
| (···)     | 括号        | 限定多选结构的范围，标注量词作用的元素，为反向引用“捕获”文本 |
| \1,\2,... | 反向引用    | 匹配之前的第一、第二括号内的字表达式匹配的文本               |

用 Java 语言为例：

例 1 匹配数字：

```java
"+9.726".matches("^(\\+|\\-)?\\d\\.\\d*$")
//true
```

+9.726 是目标字符串
后面的正则表达式可以分解为以下几部分：

* ^ 表示行首，$表示行尾
* （\\+|\\-)?  这一部分是为了匹配正负号，\\是转义，为了匹配 +-，中间一个 | 表示或者，而？表示正负号可以出现一次，也可以不出现。
* \\.匹配小数点：输入的数字必须包含小数点
* \\d* ：小数点后的数字匹配，可以匹配任意长度的小数。

例 2：

匹配空格

2-1：匹配空格

```java
"\\s"
```

在\\s 后面加入表示匹配次数的元符号就可以表示匹配空格的数量
比如，\\s+ 表示至少匹配一个空格

           \\s* 可以不匹配空格  到匹配任意多个空格
    
           ？和{min，max}同理

2-1：匹配限定数量的空格

```java
System.out.println("  ".matches("^\\s{1,4}$"));
//true
System.out.println(" ".matches("^\\s{2}$"));
" "中包含了一个空格，\\s{2}表示匹配两个空格 
//false
```

**一些常用的匹配：**

| \b   | 匹配一个字边界，即字与空格间的位置。例如，"er\b"匹配"never "中的"er"，但不匹配"verb"中的"er"。 |
| :--- | :----------------------------------------------------------- |
| \B   | 非字边界匹配。"er\B"匹配"verb"中的"er"，但不匹配"never "中的"er"。 |
| \d   | 数字字符匹配。等效于 [0-9]。                                 |
| \D   | 非数字字符匹配。等效于 [^0-9]。                              |
| \s   | 匹配任何空白字符，包括空格、制表符、换页符等。与 [ \f\n\r\t\v] 等效。 |
| \S   | 匹配任何非空白字符。与 [^ \f\n\r\t\v] 等效。                 |
| \w   | 匹配任何字类字符，包括下划线。与"[A-Za-z0-9_]"等效。         |
| \W   | 与任何非单词字符匹配。与"[^A-Za-z0-9_]"等效。                |

例 3：获取匹配的内容

Java 中 有两种子模式 Capturing 和 Non-Capturing。

Capturing 指获取匹配,是指系统会在幕后将所有子模式匹配结果保存起来，供查找和替换

Non-Capturing 指非获取匹配，这时系统不会保存子模式的匹配结果。

3-1：初识 group

```java
        Pattern pattern = Pattern.compile("\\d{3,5}[a-z]{2}");
        //此正则表达式代表 3-5 个数字后面有两个字母
		String s = "123aa-5423zx-642oi-00";
		Matcher matcher = pattern.matcher(s);
		while(matcher.find()) {
			System.out.println(matcher.group());
		}
        //结果：
        123aa
        5423zx
        642oi
        //输出 group——size
        
```

上面的 "\\d{3,5}[a-z]{2}" 是一组，因此 matcher.group()就是遍历找到的组
而正则表达式也支持 表达式分组

```java
Pattern pattern = Pattern.compile("(\\d{3,5})([a-z]{2})");
String s = "123aa-5423zx-642oi-00";
Matcher matcher = pattern.matcher(s);
while(matcher.find()) {
	System.out.println(matcher.group(1));
	System.out.println(matcher.group(2));
}
//结果：
123
aa
5423
zx
642
oi
//可以把 group 理解为一个嵌套，matcher 的 group（）是匹配的整个表达式的结果
//而 group 中的 0,1,2 表示的是 自身，第一个分组，第二个分组 的匹配内容。
//python 中好像也是这样的形式。
```

如果并不想获取 group 里的某些内容，做法如下：

```java
Pattern pattern = Pattern.compile("(？：\\d{3,5})([a-z]{2})");
String s = "123aa-5423zx-642oi-00";
Matcher matcher = pattern.matcher(s);
System.out.println("GroupCount:");
System.out.println(matcher.groupCount());
while(matcher.find()) {
	System.out.println(matcher.group(1));
}
//结果如下：
GroupCount:
1
aa
zx
oi
```

3-2 获取匹配

* 反向引用

捕获会返回一个捕获组，这个分组保存在内存中，不仅可以在正则表达式外部通过程序进行引用，也可以在正则表达式内部进行引用，这种引用方式就是反向引用

例 4：查找一串字母“aabbbbgbddesddfiid”里成对的字母

如果按照写代码的思路：

（1）匹配到第一个字母

（2）向右扫描下一个字母，检查是否与上一个字母相同

（3）如果一样，匹配成功，否则失败。

问题在于：正则表达式如何记住上一个匹配到的字母？

```java
String test = "aabbbbgbddesddfiid";
Pattern pattern = Pattern.compile("(\\w)\\1");
Matcher mc= pattern.matcher(test);
while(mc.find()){
    System.out.println(mc.group());
}
//结果如下：
aa
bb
bb
dd
dd
ii
```

在正则表达式"(\\w)\\1"中，“\\1”就代表第一个捕获组获取到的内容
例 5：替换指定字符串

```java
"abc def aaa bbb".replaceAll("(\\w+)\\s+(\\w+)", "$2 $1"); //结果是 def abc bbb aaa
```

分析：

首先这个正则表达式会匹配到两个 大group，分别是

abc def

aaa bbb

然后，replaceAll操作把每个组的 第一个小组 的结果与 第二个小组的 内容交换。

3-3 非获取匹配

非捕获匹配时，系统不会保存子模式的匹配结果，更多的只作为一种限制条件使用。

* 正向预查

正向预测(?=pattern),即查找一个字符串，该字符串的后边接有符合pattern条件的子字符串，但此pattern为非匹配捕获，即不需要获取以供以后使用。

例6 查找元字符中符合“win ”格式的子字符串，而且该字符串后面一定跟着一个“7”字符串。

```java
Pattern pattern = Pattern.compile("win\\s(?=7)",Pattern.UNICODE_CASE);
Matcher matcher = pattern.matcher("win 8");
while(matcher.find()) {
	System.out.println(matcher.group());
}
```

* 反向预查

查找元字符中符合“win ”格式的子字符串，而且该字符串后面没有跟一个“7”字符串。

```java
Pattern pattern = Pattern.compile("win\\s(?!7)",Pattern.UNICODE_CASE);
Matcher matcher = pattern.matcher("win 8");
while(matcher.find()) {
	System.out.println(matcher.group());
}
```

此正则表达式要匹配的是 win后边带一个空格，而且后面没有跟着7
如果 string是 “win 7”则匹配不到。

* 负正向预查

也就是前边一定要有个 xxx字符

盲猜一下，格式是 **（?=）rex !!!!!!!!!!!!!!!!!!!!!!! wrong answer**

正确格式：(?<=7) rex


* 负反向预查

要匹配的字符前边一定没有什么字符

(?<！7) rex

- [ ] 贪婪与非贪婪



