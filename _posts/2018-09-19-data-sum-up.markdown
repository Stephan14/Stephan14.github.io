---
layout:     post
title:      "go 数据类型使用总结"
subtitle:   "《Go语言学习笔记》第五章总结"
date:       2018-09-19 20:05:30
author:     "邹盛富"
header-img: "img/sunset-clouds-1149792_1920.jpg"
tags:
    - go
---

## 字符串

字符串为**不可变字节序列**,其本身为复合结构:
```
type stringStruct struct {
    str unsafe.Pointer
    len int
}
```
其中头部指针指向字节数组，没有NULL结尾，默认以UTF-8编码存储Unicode字符。
- 字符串的默认值不是nil，而是""
- 支持 !=、 ==、 <、 >、+、 += 操作符
- 允许以索引访问**字节数组**，但是不能获取元素地址
```
func main() {
    s := "abc"
    println(s[1])
    println(&s[1]) //错误：不能取地址
}
```
### 遍历

```
func main() {
    s := "雨痕"
    //byte 遍历
    for i:=0; i <  len(s); i++ {
        fmt.Printf("%d : [%c]\n", i, s[i])
    }
    //rune 遍历
    for i, c := range s {
        fmt.Printf("%d : [%c]\n", i ,c)
    }
}
```
### 转换
如果要修改字符串，需要将其转化为可变类型([]byte或者[]rune),然后再转换回来，这需要重新分配内存。

```
func main() {
    s := "hello world"

    bs := []byte(s)
    s2 := string(bs)

    rs := []rune(s)
    s3 := string(rs)
}
```
在某些情况下，为了提高性能会对其进行优化，避免额外分配内存和赋值操作
- 在类型为`map[string]`中查询时，默认将`[]byte`转换为string key
- `for`循环迭代时，将`string`转换为`[]byte`,直接取字节赋值给局部变量

### 性能
构建字符串容易造成性能问题，用加法操作拼接字符串每次都需要重新分配内存。

通常的解决方式是通过预先分配足够的内存空间，然后使用strings.Join函数拼接成字符串

### Unicode

类型`rune`专门用来存储Unicode码，它是int32的别名。
