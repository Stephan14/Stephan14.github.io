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

## 数组

定义数组类型的时候，长度必须是非负整型常量表达式，长度是类型的组成部分；也可以使用`...`来代替数组长度，此时编译器按照初始化值的数量确定数组长度,并且定义多维数组的时候仍然可以使用。
```
a := [4]int{2, 5}
b := [...]int{10, 3:100}
```

- `len`和`cap`都返回多维数组的第一维长度
- 如果元素类型支持 != == 操作符，那么数组也支持  
- 可以获取任意的数组元素
- 数组指针可以直接用来操作元素
- 数组是值类型，赋值和传参操作都会复制整个数组的数据

## 切片

其复合结构如下：
```
type slice struct {
    array unsafe.Pointer
    len int
    cap int
}
```
- 可以基于数组和数组指针创建切片
- `cap`表示切片所引用数组片段的真实长度，`len`用于限定可读的写元素数量
- 通过`make`指定`cap`和`len`的指定切片的长度和容量
- 初始化的区别:
    - 定义`int[]`类型变量，并未执行初始化操作
    - 完成定义以及初始化全部操作
```
var a int[]
b := []int{}
```
- 不支持比较，及时元素类型支持
- `*int[]`类型不支持索引操作，必须返回`[]int`类型对象
- 可以在切片的基础上创建切片，但是**不能超出去原来的`cap`**,并且不受`len`的限制

### append

超出切片的`cap`限制，新分配的数组长度是原来`cap`的2倍，而非原数组的二倍，对于较大的切片，会尝试扩容1/4

### copy
允许两个切片指向同一个数组，允许目标区域重叠，最终所复制的长度以较短的长度为准

## 字典

- 要求key必须支持相等运算
- 访问不存在的key值，返回value的默认值
- 对字典进行迭代，每次迭代的循序都不一样
- 字典被设计成“not addressable”，不能直接修改value成员

    ```
    func main() {
        type user struct {
            name string
            age byte
        }

        m := map[int]user {
            1: {"Tom", 19},
        }

        m[1].age += 1   //错误，不可寻址

        u := m[1]
        u.age += 1
        m[1] = u        //正确

        m2 := map[int]*user {
            1: &user{"jack", 20},
        }
        m2[1].age++
    }
    ```
- 不能对nil字典进行写操作，但是可以读
    ```
    func main() {
        var m map[string]int
        println(m["a"])
        m["a"] = 1

        var m2 map[string]int
    }
    ```
- 在迭代期间，新增或者删除key是安全的
- 字典运行时会进行并发操作检测，如果某个任务正在进行写操作，那个其他任务就不能对字典进行并发操作（读、写、删除）

### 性能
- 字典对象本身就是指针的封装，传参时无需再次取地址
- 创建时预先准备足够空间有助于提升性能，减少扩张时内存分配和重哈希操作
