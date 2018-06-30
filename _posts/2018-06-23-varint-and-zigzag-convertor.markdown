---
layout:     post
title:      "实现varint和zigzag转换器"
subtitle:   "RPC服务中流量的极致优化"
date:       2018-06-23 11:53:00
author:     "邹盛富"
header-img: "img/salt-3060093_1920.jpg"
tags:
    - 算法
---

### 背景
在RPC服务中，消息传递的大部分中使用的整数都是很小的非负整数，但是一个整数一般都占用4个字节，这样会导致很占用资源。所以就发明了一个变长整型类型`varint`，这样当时数值非常小的时候，就可以只使用1个字节，稍微大一点的时候就使用2个字节，再大一点就使用3个字节，还可以使用超过4个字节来表示整型数字。

### 原理

其原理就是将数字的二进制形式以7个比特位为一组进行拆分，每7个比特位和一个标志位作为最高位组成一个新的`字节`，这个最高位用来标识否后面还有字节，1表示还有字节需要继续读，0表示到读到当前字节就结束。其结构如图所示：
![](http://res.cloudinary.com/bytedance14/image/upload/v1529732203/blog/%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7_2018-06-23_%E4%B8%8B%E5%8D%8812.14.05.png)
因此，对于一个小于128的数字可以使用byte来表示；对于一个大于128的数字，比如300，就会使用两个字节来表示：`1010 1100 0000 0010`,如下图所示：

![](http://res.cloudinary.com/bytedance14/image/upload/v1530335754/blog/20120503100854213.jpg)

### 问题
但是对于上面的编码方式，还是存在一个问题那就是对于负数怎么办，如过是-1，则其对应的16进制为`0xFFFFFFFF`,如果要按照这个编码方式岂不是需要6个字节才能够存下来。于是`zigzag`编码方式应运而生，`zigzag`编码方式将整数范围--映射到自然数范围之内，然后在进行编码，映射关系如下：
```
0 => 0
-1 => 1
1 => 2
-2 => 3
2 => 4
-3 => 5
3 => 6
```

zigzag 将负数编码成正奇数，正数编码成偶数。解码的时候遇到偶数直接除 2 就是原值，遇到奇数就加 1 除 2 再取负就是原值。

### 代码实现

```
#include <iostream>
#include <vector>

class Convertor {
    public:
        void SetNum(int64_t num);
        int64_t GetNum();
        Convertor();
        ~Convertor();

    private:
        std::vector<unsigned char> data_;
    private:
        u_int64_t signed2unsigned(int64_t num);

        int64_t unsigned2signed(u_int64_t num) {
            return (-(num & 0x01)) ^ ((num >> 1) & ~(1 << 31));
        }
};


Convertor::Convertor() {
}

Convertor::~Convertor() {
}

//zigzag 将负数编码成正奇数，正数编码成偶数
//解码的时候遇到偶数直接除 2 就是原值，遇到奇数就加 1 除 2 再取负就是原值。
u_int64_t Convertor::signed2unsigned(int64_t num) {
    return (u_int64_t)((num << 1) ^ (num >> 31)); //符号位向右移动后,正数的话补0,负数补1
}

void Convertor::SetNum(int64_t num) {

    u_int64_t unsigned_num = signed2unsigned(num);
    do{
        unsigned char tmp = (unsigned char)(unsigned_num & 0x7F);
        unsigned_num >>= 7;
        if (num != 0) {
            tmp |= 0x80;
        } else {
            tmp &= 0x7F;
        }
        data_.push_back(tmp);
    }while (unsigned_num != 0);
}

int64_t Convertor::GetNum() {
    u_int64_t num = 0;

    for(auto index = 0; index < data_.size(); ++index) {
        num |= (data_[index] & 0x7F) << (7 * index);
        if ((data_[index] & 0x80) == 0) {
            return unsigned2signed(num);
        }
    }
    return unsigned2signed(num);
}

int main() {
    Convertor c;
    c.SetNum(-123124);
    std::cout << c.GetNum() << std::endl;
    return 0;
}
```
