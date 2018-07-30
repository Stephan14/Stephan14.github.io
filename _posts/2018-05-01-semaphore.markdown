---
layout:     post
title:      "信号量"
subtitle:   "互斥访问与条件同步"
date:       2018-05-01 21:34:00
author:     "邹盛富"
header-img: "img/semaphore.jpeg"
tags:
    - 操作系统
---

### 背景

在多个进程或者线程读写数据的时候，最终结果依赖于多个进程指令执行的顺序。为了解决这种问题，科学家们提出了几种并发机制，例如：信号量，管程，自旋锁，消息传递等机制。这里先来介绍一下信号量。

### 原理
信号量是用于进程之间传递信号的一个整数值，这里由sem表示。在信号量只有三种操作可以进行：初始化，P操作，V操作，这三种操作都是原子操作。

P操作：

    sem = sem - 1；

如果sem  < 0,则该进程进入阻塞状态，否则继续运行。

V操作：

    sem = sem + 1；

如果sem *<=* 0,则唤醒一个阻塞状态的进程。

### 特征
- 初始化完成之后，只能通过P操作和V操作来改变信号量。
- P操作和V操作是原子操作，其优先级高于用户代码。

### 实现
```
struct semaphore{
    int count;
    waitqueue queue;
};

void P(semaphore s){//P操作
    s.count--;
    if (s.count < 0) {
        /*把当前进程插入到队列中*/
        /*阻塞当前进程*/
    }
}
void V(semaphore s){//V操作
    s.count++;
    if (s.count <= 0) {
        /*把进程P从当前队列中移除*/
        /*把进程P插入到就绪队列中*/
    }
}
```

### 分类

- 二进制信号量：资源数为0或1,通常用于进程互斥
- 资源信号量：资源数目为任何非负值,通常用于进程同步

### 使用场景
- 互斥访问：临界区的互斥访问控制
- 条件同步：线程之间的事件等待

### 信号量实现临界区的互斥访问

每类资源设置一个信号量，其初值为1

```
semaphore* mutex = new semaphore(1);  
P(*mutex);  
critical section;  
V(*mutex);  
```
#### 注意

1. 必须成对使用P操作和V操作
2. P操作保证互斥访问的临界资源
3. V操作在使用之后释放资源
4. PV操作次序不能错误或者遗漏或者重复

### 信号量实现线程间的条件同步

设置一个信号量，期初值为0。要求线程B执行完X后，线程A才能执行N。

```
semaphore* condition = new semaphore(0);

       /*线程A*/                          /*线程B*/
   ................                  ..................
   ................                  ..................
   .......M........                  ..................
   P(*condition);                    ..................
   .......N........                  .........X........
   ................                  V(*condition);
   ................                  .........Y........
   ................                  ..................
   ................                  ..................
   ................                  ..................

```
### 用信号量解决生产者-消费者问题

#### 问题分析
- 任何时候只能有一个线程操作缓冲区---------互斥访问
- 缓冲区空时，消费者必须等待生产者---------条件同步
- 缓冲区满时，生产者必须等待消费者---------条件同步

#### 伪代码
用信号量描述每个约束：
- 互斥访问：二进制信号量mutex
- 条件同步：资源信号量empty
- 条件同步：资源信号量full
注：emputy与full的和为缓冲区的大小

```
#define N 100  
typedef int semaphore  
semaphore mutex = 1;  
semaphore empty = N;  
semaphore full = 0;  

void producer ( void ){  

    int item;  

    while ( TRUE ) {  
        item = produce_item( );      //产生放在缓冲区的数据  
        P( *empty );                 //将空槽数减1  
        P( *mutex );                 //进入临界区  
        insert_item( item );         //将新数据放入缓冲区  
        V( *mutex );                 //离开临界区  
        V( *full );                  //将满槽数加1  
    }  
}  


void consumer ( void ){  

    int item;  

    while ( TRUE ) {  
        P( *full );                  //将满槽数减1  
        P( *mutex );                 //进入临界区  
        item = remove_item( item );  //从行冲区中取出数据项  
        V( *mutex );                 //离开临界区  
        V( *empty );                 //将空槽数加1  
        consumer_item( item );       // 处理数据项  
    }  
}

```
#### C代码
```
#define _GNU_SOURCE
#include "sched.h"
#include "pthread.h"
#include "stdio.h"
#include "stdlib.h"
#include "semaphore.h"
#include "unistd.h"
int producer(void * args);
int consumer(void *args);
pthread_mutex_t mutex;
sem_t product;
sem_t warehouse;
char buffer[8][4];
int bp=0;
int main(int argc,char** argv)
{
    pthread_mutex_init(&mutex,NULL);
    sem_init(&product,0,0);
    sem_init(&warehouse,0,8);
    int clone_flag,arg,retval;
    char *stack;
    clone_flag = CLONE_VM | CLONE_SIGHAND | CLONE_FS |CLONE_FILES;
    int i;
    for(i=0;i<2;i++)
    {  //创建四个线程
        arg = i;
        stack =(char*)malloc(4096);
        retval=clone((void*)producer,&(stack[4095]),clone_flag,  (void*)&arg);
        stack =(char*)malloc(4096);
        retval=clone((void*)consumer,&(stack[4095]),clone_flag,   (void*)&arg);
    }
    exit(1);
}
int producer(void* args)
{
    int id = *((int*)args);
    int i;
    for(i=0;i<10;i++)
    {
        sleep(i+1);  //表现线程速度差别
        sem_wait(&warehouse);
        pthread_mutex_lock(&mutex);
        if(id==0)
            strcpy(buffer[bp],"aaa\0");
        else
            strcpy(buffer[bp],"bbb\0");
        bp++;
        printf("producer%d produce %s in %d\n",id,buffer[bp],bp-1);
        pthread_mutex_unlock(&mutex);
        sem_post(&product);
    }
    printf("producer%d is over!\n",id);
}
int consumer(void *args)
{
    int id = *((int*)args);
    int i;
    for(i=0;i<10;i++)
    {
        sleep(10-i);  //表现线程速度差别
        sem_wait(&product);
        pthread_mutex_lock(&mutex);
        bp--;
        printf("consumer%d get %s in%d\n",id,buffer[bp],bp+1);
        strcpy(buffer[bp],"zzz\0");
        pthread_mutex_unlock(&mutex);
        sem_post(&warehouse);
    }
    printf("consumer%d is over!\n",id);
}
```
