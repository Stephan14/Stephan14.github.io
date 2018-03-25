---
layout:     post
title:      "Linux中使用正则表达式"
subtitle:   "正则表达式"
date:       2018-03-25 10:00:00
author:     "邹盛富"
header-img: "img/reg.png"
tags:
    - Linux
    - 正则表达式
---

### 介绍

正则表达式（regular expression）是一种指定字符串模式的简洁方式，通常简写为regex或re，最常见的应用就是搜索字符串。


### 使用方法

通过一个例子来学习正则表达式。现在有如下的一个data.txt文件：
```
Harley is smart

Harley

I like Harley

the dog likes the cat

```

目的                 | 使用方法
--------------------|--------------------
搜索以”Harley“开头的行 | grep ‘^Harley’ data
搜索以”Harley“结尾的行 |  grep ‘Harley$’ data
搜索整行都是“Harley”的行|grep ‘^Harley$’ data
统计空行数| grep ‘^$’ data  wc -l
查找包含字符串“Har”的行，并且“Har”出现在单词的开头|grep '\<Har' data
查找包含字符串“Har”的行，并且“Har”出现在单词的结尾|grep 'Har\>' data
查找包含字符串“Harley”的行，并且“Harley”为单词|grep '\<Harley\>' data
查找包含字符串”Har“后面跟两个任意字符，再跟一个字母”y“的行|grep 'Har..y' data
查找包含字符串”Har“后面跟A或者a的字符串的行|grep 'H[aA]' data
使用预定义字符类查找包含包含大写字母后跟小写字母的所有行|grep '[[:upper:]][[:lower:]]' data
使用范围查找包含字符串”3-9“字符串中任意一个的所有行|grep '[3-9]' date
查找包含字符串”X“，同时后面不跟有“a”和“o”的所有行|grep ‘X[^ao]’ data
使用范围和预定义字符类查找一行中某一个字符是非字母字符的行|grep ‘[^A-Za-z]’ data 或者 grep '[^[:alpha:]]' data
查找包含大写字母的“H”后面包含跟0个或者多个小写字母的行|grep 'H[[:lower:]]*' data


### 符号含义

**理解正则表达式的关键就是记住每个字符类只表示一个单独的字符**

符号| 含义
----|----
* |表示匹配前面的字符0次或者多次出现,最常用的是.*可以匹配任何字符的0次或者多次出现。
+ | 表示匹配前面的字符1次或者多次出现
？| 表示匹配前面的字符1次或者0次出现
{n}|正好匹配n次
{n，}|至少匹配n次
{,m}|最多匹配m次（在一些程序中无法使用）
{n,m}|至少匹配n次，最多匹配m次


#### 例子

* 查找所有包含2个数字或者3个数字的行（使用\<.\>匹配整个数字）

  grep ‘\<[0-9]{2,3}\>’ data


* 查找cat dog bird hamster中包含上述任意单词的一行

  grep '\<(cat\|dog\|bird\|hamster)\>' data

如果希望查找真是的*,.或者\|，可以使用\引用这些字符。
