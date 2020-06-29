---
title: shell初学者编程
date: 2020-06-27 00:40:25
tags:
	- others
---

shell 编程初学

shell中用的最多的竟然是字符串，为难我┑(￣Д ￣)┍

<!--more-->

## 1.关于$1作为函数后缀的困惑
在sh中写一个函数:
```sh
#!/bin/bash

func_readfile()
{
    echo "echo"    
}

func_$1
```
之前一直没看懂func_\$1是什么意思
以为是这个shell中以func_为前缀的所有函数
原来\$1指的是执行shell时的第一个参数

这个代码这样执行:
![](https://upload-images.jianshu.io/upload_images/19092361-08ade47bcdcd8110.png)
## 2.把文件路径作为命令行参数

```sh
#!/bin/bash

LOG_FILE=$1
func_readfile()
{
    cat ${LOG_FILE} | while read line
    do
	    echo $line
    done    
}

func_$2

```

**千万千万不要忘记变量前边加 \$(🙃查了半个小时,结果是这么个愚蠢错误)**
## 3. 嵌套循环
```sh
#!/bin/bash

LOG_FILE=$1
CONFIG_FILE=$2

func_capbility()
{
    cat ${LOG_FILE} | while read line
    do
	    cat ${CONFIG_FILE} | while read configline
	        do 
		        echo $line
		        echo $configline
	         done
    done    
}

func_$3
```
死循环了嘤（；´д｀）ゞ

## 4. 切割字符串
```sh
#!/bin/bash
str="1                     root               root              init               []"
arr=($str)
echo ${arr[2]}
```
还有很多方法

## 5.读取某个目录下所有文件的名字并返回到数组中(不递归遍历子目录)
```sh
#!/bin/bash

LOG_FILE=$1
#CONFIG_FILE=$2
CONFIG_DIR=$2

func_getconfig_dirpath()
{
    files=$(ls $CONFIG_DIR 
)
    echo ${files[@]}
}
//调用
func_capbility()
{
    config_files=$(func_getconfig_dirpath)
}
```
`files=$(ls $CONFIG_DIR)`这一句应该是重定向吧

## 6.shell读取json文件
本着尽量减少不下载包的原则,就用系统已有的命令做

```sh
#!/bin/bash
CONFIG_FILE=$1

func_parsejson(){
    local value=$(cat $CONFIG_FILE | tr '\n' ' ' )#ls竟然是有换行符的，命令行坑我
    echo $value
}
```
这样整个json文件就变成了一行

然后使用awk跟sed获取到想要的字段信息

```sh
解析json
func_parsejson(){
    local value=$(cat $1)
    echo $value
    capbility=`echo $value | awk -F'],' '{print $1}'|awk -F'[:[]+' '{print$2}'|sed 's/\"//g'`
    cap_arr=(${capbility//,/})
    echo $cap_arr
}
```
```json
monitor.json
{ 
    "capbility":[
        "cap_setgid",
        "cap_setuid",
        "cap_sys_boot+ep"
    ],
    "DAC":[
        {"path1":777},
        {"path2":750},
        {"path3":640}
    ]
}
```
## 7 debug脚本
sh -x example.sh arg1 arg2....
一直报我有语法错误
经查
改为
bash -x  example.sh arg1 arg2....
>┑(￣Д ￣)┍ 脚本的debug真的友好，java的命令行debug是不是也很友好

## 8 判断shell中的两个数组是否相等（数组中都没有重复元素，无序）
```sh
isEqual(){
    arr1=($1)
    arr2=($2)
    local flag=0
    intersection=($(comm -12 <(printf "%s\n" ${arr1[@]} | LC_ALL=C sort) <(printf "%s\n" ${arr2[@]} | LC_ALL=C sort)))
    
    #echo ${intersection[@]}
    if [ "${#intersection[@]}" == "${#arr2[@]}" ] && [ "${#intersection[@]}" == "${#arr1[@]}" ];then
    	flag=1
    else
	    flag=0
    fi
    echo $flag
    return $?
}
```
通过comm命令来比较两个数组，printf是把两个数组打印到文件中(应该是个临时文件)，sort是先把数组排序
comm -12 提取出二者的交集
交集的大小跟两个数组大小都相同，那么这两个数组相同。


用法：comm [选项]... 文件1 文件2
逐行比较已排序的文件文件1 和文件2。

When FILE1 or FILE2 (not both) is -, read standard input.

如果不附带选项，程序会生成三列输出。第一列包含文件1 特有的行，第二列包含 文件2 特有的行，而第三列包含两个文件共有的行。

  -1		不输出文件1 特有的行
  -2		不输出文件2 特有的行
  -3		不输出两个文件共有的行

  --check-order			检查输入是否被正确排序，即使所有输入行均成对
  --nocheck-order		不检查输入是否被正确排序
  --output-delimiter=STR	依照STR 分列
  -z, --zero-terminated    line delimiter is NUL, not newline
      --help		显示此帮助信息并退出
      --version		显示版本信息并退出

Note, comparisons honor the rules specified by 'LC_COLLATE'.

示例：
  comm -12 文件1 文件2  只打印在文件1 和文件2 中都有的行
  comm -3  文件1 文件2  打印在文件1 中有，而文件2 中没有的行。反之亦然。
## 总代码
```sh
#!/bin/bash

LOG_FILE=$2
CONFIG_DIR=$3

getconfig()
{
    files=$(ls $CONFIG_DIR | tr '\n' ' ')
    echo ${files[@]}
}

parsejson(){
    value=$(cat $1)
    capbility=`echo $value | awk -F'],' '{print $1}'|awk -F'[:[]+' '{print$2}'|sed 's/\"//g'`
    cap_arr=${capbility//,/}
    echo ${cap_arr[@]}
}

isEqual(){
    arr1=($1)
    arr2=($2)
    local flag=0
    intersection=($(comm -12 <(printf "%s\n" ${arr1[@]} | LC_ALL=C sort) <(printf "%s\n" ${arr2[@]} | LC_ALL=C sort)))
    
    #echo ${intersection[@]}
    if [ "${#intersection[@]}" == "${#arr2[@]}" ] && [ "${#intersection[@]}" == "${#arr1[@]}" ];then
    	flag=1
    else
	    flag=0
    fi
    echo $flag
    return $?
}

func_capbility(){
    printf "%-20s% -100s %-100s\n" "process" "log" "config" > logconfigcompare.log
    config_files=( $(getconfig) )
    cat $LOG_FILE | while read logline
    do {
    	for filename in ${config_files[@]}
    	do {
	    log_arr=($(echo $logline | sed 's/\s\+/ /g'))
    	    realname=$(echo $filename | cut -d . -f1)
	    cmdline=$(echo ${log_arr[3]})
	    if [[ $cmdline == *$realname* ]];then
		cap_config=$(parsejson ${CONFIG_DIR}${filename})
		cap_log_temp=`echo ${log_arr[4]}|sed 's/\[//;s/\]$//'`
		cap_log=(${cap_log_temp//,/ })
		
		diff=($(isEqual "${cap_config[*]}" "${cap_log[*]}"))
                if [ "$diff" -eq 0 ];then
		    printf "%-20s %-100s %-100s\n" $realname "$cap_log_temp" "$cap_config" >> logconfigcompare.log 
		fi
	    fi
	}
    	done
    }
    done
}

func_$1 $*

```

[https://wiki2.xbits.net:4430/start](https://wiki2.xbits.net:4430/start)

两个数组是否相等
换个思路：
求两个数组的交集，看这个交集的长度是不是跟这两个数组的长度都相等！！
bingo！
```sh
#!/bin/bash

arr1=(cap_a cap_b cap_c)
arr2=(cap_b)
intersection=($(comm -12 <(printf '%s\n' "${arr1[@]}" | LC_ALL=C sort) <(printf '%s\n' "${arr2[@]}" | LC_ALL=C sort)))
echo ${intersection[@]}
```


## 9 数组中*跟@的比较
>题外话，写脚本的时候确实感觉引号，括号，花括号小括号很难分清楚怎么用，要debug才能知道写的是不是自己想要的。还有 `` 跟（）的区别等等

转载
[Shell脚本数组中@跟*的区别与如何将数组作为函数参数的方法]([https://blog.csdn.net/weixin_44901564/article/details/101017738?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase](https://blog.csdn.net/weixin_44901564/article/details/101017738?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase)
)



一、shell脚本数组中@跟*的区别
`“${数组变量[*]}”`：加上双引号，bash会当成是一串字符串处理
`${数组变量[*]}`：不加上双引号，bash会当成是数组处理
`${数组变量[@]`：加不加双引号bash都是当成数组处理

二、将数组作为函数参数的方法
假设一个数组：
a=(1 2 3)
下面用脚本来说一下：
```sh
#!/bin/bash
function ceshi //这里定义一个函数
{
z=($1)
echo ${z[@]}
}

a=(1 2 3) //定义数组a
ceshi “${a[*]}” //运行函数，并且传入数组参数
```
解释：




## 10 全部改成字符串操作
>设备上不支持数组也是醉了

```sh
#!/bin/bash


string_tofile(){
    string="$1"
    file="$2.txt"
    >$file
    i=1
    while ((1==1))
    do
        splitchar=$(echo $string|cut -d "," -f$i)
	if [ "$splitchar" != "" ]
	then
		((i++))
		echo $splitchar>>$file
	else
		break
	fi
    done
}

isEqual(){
    string1=$(sort config.txt)
    string2=$(sort log.txt)
    local flag=0
    if [ "$string1" == "$string2" ]
    then
	flag=1
    else
	flag=0
    fi
    echo $flag
    return $?
}

func_test(){
    str1="cap_high,cap_ability,cap_tap,cap_override"
    str2="cap_override,cap_high,cap_ability,cap_tap"
    string_sort_tofile "$str1" "config"
    string_sort_tofile "$str2" "log"
    diff=$(isEqual)
    if [ "$diff" -eq 1 ]
    then
	    echo "equal"
    else
	    echo "no"
    fi
}

func_test

```

