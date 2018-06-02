---
layout:     post
title:      "go语言interface"
subtitle:   "interface使用介绍"
date:       2018-06-02 00:16:00
author:     "飞雪无情"
header-img: "img/1525971154898.jpg"
tags:
    - go
---
### 背景
接口是一种约定，它是一个抽象的类型，和我们见到的具体的类型如int、map、slice等不一样。具体的类型，我们可以知道它是什么，并且可以知道可以用它做什么；但是接口不一样，接口是抽象的，它只有一组接口方法，我们并不知道它的内部实现，所以我们不知道接口是什么，但是我们知道可以利用它提供的方法做什么。

抽象就是接口的优势，它不用和具体的实现细节绑定在一起，我们只需定义接口，告诉编码人员它可以做什么，这样我们可以把具体实现分开，这样编码就会更加灵活方面，适应能力也会非常强。

### 简单使用
我们可以定义很多类型，让它们实现一个接口，那么这些类型都可以赋值给这个接口，这时候接口方法的调用，其实就是对应*实体类型*对应方法的调用，这就是多态。

```
func main() {
	var a animal
	var c cat
	a=c
	a.printInfo()
	//使用另外一个类型赋值
	var d dog
	a=d
	a.printInfo()
}
type animal interface {
	printInfo()
}
type cat int
type dog int
func (c cat) printInfo(){
	fmt.Println("a cat")
}
func (d dog) printInfo(){
	fmt.Println("a dog")
}
```
以上例子演示了一个多态。我们定义了一个接口`animal`,然后定义了两种类型`cat`和`dog`实现了接口`animal`。在使用的时候，分别把类型`cat`的值`c`、类型`dog`的值`d`赋值给接口`animal`的值`a`,然后分别执行`a`的`printInfo`方法，可以看到不同的输出。
```
a cat
a dog
```

**我们看下接口的值被赋值后，接口值内部的布局。接口的值是一个两个字长度的数据结构，第一个字包含一个指向内部表结构的指针，这个内部表里存储的有实体类型的信息以及相关联的方法集；第二个字包含的是一个指向存储的实体类型值的指针。所以接口的值结构其实是两个指针，这也可以说明接口其实一个引用类型。**

### 接口赋值

我们都知道，如果要实现一个接口，必须实现这个接口提供的所有方法，但是实现方法的时候，我们可以使用指针接收者实现，也可以使用值接收者实现，这两者是有区别的，下面我们就好好分析下这两者的区别。

```
func main() {
	var c cat
	//值作为参数传递
	invoke(c)
}
//需要一个animal接口作为参数
func invoke(a animal){
	a.printInfo()
}
type animal interface {
	printInfo()
}
type cat int
//值接收者实现animal接口
func (c cat) printInfo(){
	fmt.Println("a cat")
}
```
还是原来的例子改改，增加一个`invoke`函数，该函数接收一个`animal`接口类型的参数，例子中传递参数的时候，也是以类型`cat`的值`c`传递的，运行程序可以正常执行。现在我们稍微改造一下，使用类型`cat`的指针`&c`作为参数传递。
```
func main() {
	var c cat
	//指针作为参数传递
	invoke(&c)
}
```

只修改这一处，其他保持不变，我们运行程序，发现也可以正常执行。通过这个例子我们可以得出结论：**实体类型以值接收者实现接口的时候，不管是实体类型的值，还是实体类型值的指针，都实现了该接口。**

下面我们把接收者改为指针试试。
```
func main() {
	var c cat
	//值作为参数传递
	invoke(c)
}
//需要一个animal接口作为参数
func invoke(a animal){
	a.printInfo()
}
type animal interface {
	printInfo()
}
type cat int
//指针接收者实现animal接口
func (c *cat) printInfo(){
	fmt.Println("a cat")
}
```
这个例子中把实现接口的接收者改为指针，但是传递参数的时候，我们还是按值进行传递，点击运行程序，会出现以下异常提示：
```
./main.go:10: cannot use c (type cat) as type animal in argument to invoke:
	cat does not implement animal (printInfo method has pointer receiver)
```

提示中已经很明显的告诉我们，说`cat`没有实现`animal`接口，因为`printInfo`方法有一个指针接收者，所以`cat`类型的值`c`不能作为接口类型`animal`传参使用。下面我们再稍微修改下，改为以指针作为参数传递。

```
func main() {
	var c cat
	//指针作为参数传递
	invoke(&c)
}
```

其他都不变，只是把以前使用值的参数，改为使用指针作为参数，我们再运行程序，就可以正常运行了。由此可见**实体类型以指针接收者实现接口的时候，只有指向这个类型的指针才被认为实现了该接口。**

现在我们总结下这两种规则，首先以方法接收者是值还是指针的角度看。

Methods Receivers| Values
-----|----
(t T)|T and *T
(t *T)|	*T

上面的表格可以解读为：**如果是值接收者，实体类型的值和指针都可以实现对应的接口；如果是指针接收者，那么只有类型的指针能够实现对应的接口。**

其次我们我们以实体类型是值还是指针的角度看。

Values|Methods Receivers
----|----
T|(t T)
*T|(t T) and (t *T)

上面的表格可以解读为：**类型的值只能实现值接收者的接口；指向类型的指针，既可以实现值接收者的接口，也可以实现指针接收者的接口。**

### 接口查询

在某些情况下我们需要判断一个对象实体是否实现了某一个接口，然后执行特定的代码。这种判断机制被称为接口查询，而且需要在程序运行的时候才能确定；与接口赋值不同，接口赋值只需要通过**静态类型**就可以判断赋值是否可以行。下面介绍一下简单的代码：
```
type animal interface {
	printInfo()
}

type cat int
type dog int

func (c cat) printInfo() {
	fmt.Println("a cat")
}
func (d dog) printInfo() {
	fmt.Println("a dog")
}

func main() {
	var c cat
	var i interface{} = c // 必须赋值给一个接口类型的变量
	if v, ok := i.(animal); ok {
		fmt.Println("animal")
		v.printInfo()
	}
	var k int
	i = k
	if v, ok := i.(animal); ok {
		fmt.Println("animal")
		v.printInfo()
	} else {
		fmt.Println("not animal")
	}
}
```

### 类型查询

上述的接口查询可以看做是**查询一个实例是否属于某一个群体**，而类型查询是用来**查询一个实例具体属于某一个类型**。相比于接口查询更加细致、准确一些。类型查询的例子如下：
```
var v1 interface{} = ...
switch v := v1.(type) {
    case int:
    case string:
    .....
}
```
同时需要注意的是通过反射的方式也可以实现类型查询。
