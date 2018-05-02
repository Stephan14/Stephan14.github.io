---
layout:     post
title:      "处理机调度"
subtitle:   "实时系统中的调度"
date:       2018-05-02 23:34:00
author:     "邹盛富"
header-img: "img/processer.jpg"
tags:
    - 操作系统
---

### 实时调度算法

目前业界比较认可并且已经成为标准的实时调度算法有两个：
- 最优动态优先级调度算法：最早截止优先级调度算法（EDF）
- 最优固定优先级调度算法：单速率调度算法（RMS）

### 调度算法分类

- 抢占和非抢占调度

    根据任务运行的过程中能否被中断的情况，把调度算法分为抢占和非抢占两种。在抢占式调度算法中，正在运行的任务可以被其他任务打断。在非抢占式调度算法中，一旦任务开始运行，该任务只有在运行完后而主动放弃CPU资源或者是因为等待其他资源而被阻塞的情况下才可能停止。

- 静态和动态调度

    根据任务优先级确定的时机，把调度算法分为静态调度和动态调度两种。在静态调度算法中，所有任务的优先级在运行之前已经确定下来，这就要求能够完全把握系统中的所有任务及其时间约束（如截止时间，运行时间，优先顺序等）。在动态调度算法中，任务的优先级在在运行时确定，并且可能不断的发生变化。

### EDF算法

EDF算法是一种采用动态调度的优先级调度的算法，任务的优先级根据任务的截止时间来确定。任务的截止时间越近，任务的优先级越高，当有新的任务处于就绪状态的时候，任务的优先级就有可能进行调整。

#### 实例

下面举个例子来说明一下，现在有如下的一些任务：

|任务名称 |  到达时刻 | 执行时间 | 截止期限|
| - | - | - | -|
| T1 | 0 | 10 | 30|
| T2 | 4 | 3 | 10|
| T3 | 5 | 10 | 25|

任务的执行过程

当T1到达的时候，它是唯一处于就绪状态的任务，因此立刻开始执行。T2在时刻4到达，因为T2的截止期限比T1的截止期限小，所以它的优先级高于T1，因此打断T1开始执行。T3在时刻5到达，因为T3的截止期限比T2的大，所以T2的优先级高于T3的优先级，因此必须等待T2执行完毕。当T2在时刻7执行完毕之后，由于T3的截止期限小于T1的截止期限，所以T3的优先级高，因此开始执行T3，一直执行到时刻17完毕，此时继续执行T1。


#### EDF算法的假设

- 可以在任何时间抢占任何任务，并且还原它，这个过程没有损失并且任务的抢占次数不改变处理器的工作负荷

- 保证有充分的容量，以在截止期限的要求下执行任务，没有其他的内存等约束条件导致问题复杂

- 任务的释放时间不依赖于其他的任务的结束时间

#### EDF算法可调度的测试

如果所有任务的都是周期性的，例如：有m个周期事件，事件i以周期Pi发生，要Ci秒CPU时间处理一个事件，那么可负载处理的条件是
```
∑（1<=i<=m）(Ci/Pi)   <= 1
```
满足这个条件，则称这些任务集是可以调度的。

例如现在考虑现在考虑一个有三个周期性事件的软实时系统，其周期分别为100ms，200ms和500ms。如果这些事件分别需要50ms，30ms和100ms的CPU时间，由于50/100+30/200+100/500<1，所以这个实时系统是可以调度的。

EDF算法不仅可以调度硬实时周期任务，还可以调度硬实时非周期任务，且调度硬实时周期任务集时，周期任务集最大的总负载可达到100%。

### RMS算法

RMS算法是根据任务的优先级按任务周期T来分配，它根据任务的执行周期的长短来决定调度的优先级，哪些具有小的周期的任务较高的优先级，周期长的任务具有较低的优先级。

#### RMS算法的假设
- 各个任务之间没有资源共享，没有忙等，没有mutex, 也没有semaphore.
- 每个任务的最后期限是周期性的。
- 基于优先级抢占的，即高优先级任务一旦就绪的话，会立马抢占低优先级任务。
- 任务优先级的分配原则是，周期越短的任务，优先级越高。
- 任务切换以及纯内核任务的消耗忽略不计对于这个理论模型。

#### RMS算法可调度的测试

如果所有任务的都是周期性的，例如：有m个周期事件，事件i以周期Pi发生，要Ci秒CPU时间处理一个事件，那么可负载处理的条件是
```
∑（1<=i<=m）(Ci/Pi) < n(2^(1/n)-1)
```
满足这个条件，则称这些任务集是可以调度的。

### 代码实现
```

//
//  main.c
//  os_lab_3
//
//  Created by ZouSF on 15/6/3.
//  Copyright (c) 2015年 ZouSF. All rights reserved.
//

#include "math.h"
#include "sched.h"
#include "pthread.h"
#include "stdio.h"
#include "stdlib.h"
#include "semaphore.h"
#include "unistd.h"

typedef struct{                //实时任务描述
    char task_id;
    int call_num;                  //任务发生次数
    int ci;                         //任务处理时间
    int ti;                         //任务发生周期
    int ci_left;
    int ti_left; //record the reduction of ti \ci
    int flag;                      //任务是否活跃,0否，2是
    int arg;                       //参数
    pthread_t th;                       //任务对应线程
}task;

void proc(int *args);
void *idle();
int select_proc(int alg);

int task_num=0;
int idle_num=0;

int alg;                        //所选算法，1 for EDF，2 for RMS
int curr_proc=-1;
int demo_time=100;              //演示时间

task *tasks;
pthread_mutex_t proc_wait[10];    //the biggest number of tasks
pthread_mutex_t main_wait,idle_wait;
float sum=0;
pthread_t idle_proc;

int main(int argc,char **argv)
{
    pthread_mutex_init(& main_wait , NULL);
    pthread_mutex_lock(& main_wait);      //下次执行lock等待
    pthread_mutex_init(& idle_wait , NULL);
    pthread_mutex_lock(& idle_wait);     //下次执行lock等待
    printf("Please input number of real time task:\n");
    int c;
    scanf("%d",& task_num);                 //任务数
    tasks=(task *)malloc(task_num *sizeof(task));
    while((c=getchar())!='\n' && c!=EOF);          //清屏
    int i;
    for(i=0 ; i<task_num ; i++)
    {
        pthread_mutex_init( & proc_wait[i] , NULL);
        pthread_mutex_lock( & proc_wait[i]);
    }

    for(i=0;i<task_num;i++)
    {
        printf("Pleased input task id,followed by Ci and Ti:\n");
        scanf("%c,%d,%d,",&tasks[i].task_id, &tasks[i].ci, &tasks[i].ti);
        tasks[i].ci_left=tasks[i].ci;
        tasks[i].ti_left=tasks[i].ti;
        tasks[i].flag=2;
        tasks[i].arg=i;
        tasks[i].call_num=1;
        sum=sum+(float)tasks[i].ci / (float)tasks[i].ti;
        while((c=getchar())!='\n'&&c!=EOF);  //清屏
    }

    printf("Please input algorithm,1 for EDF,2 for RMS:");
    scanf("%d",&alg);
    printf("Please input demo time:");
    scanf("%d", & demo_time);
    double r = 1;             //EDF算法，最早截止期优先调度

    if(alg == 2)
    {        //RMS算法，速率单调调度
        r=((double)task_num)*(exp(log(2)/(double)task_num)-1);
        printf("r is %lf\n",r);
    }
    if(sum>r) // 综合EDF和RMS算法任务不可可调度的情况
    {           //不可调度
        printf("(sum=%lf>r=%lf),not schedulable!\n",sum,r);
        exit(2);
    }
    //创建闲逛线程
    pthread_create(& idle_proc , NULL , (void*)idle , NULL);

    for(i=0 ;i<task_num ;i++)    //创建实时任务线程
        pthread_create(&tasks[i].th, NULL, (void*)proc, &tasks[i].arg);

    for(i=0;i<demo_time;i++)
    {
        int j;
        if((curr_proc=select_proc(alg))!=-1)
        {   //按调度算法选择线程
            pthread_mutex_unlock(&proc_wait[curr_proc]);    //唤醒
            pthread_mutex_lock(&main_wait);                 //主线程等待
        }
        else
        {       //无可运行任务，选择闲逛线程
            pthread_mutex_unlock(&idle_wait);
            pthread_mutex_lock(&main_wait);
        }

        for(j=0;j<task_num;j++)
        {    //Ti--,直至为0时开始下一周期
            if(--tasks[j].ti_left==0)
            {
                tasks[j].ti_left=tasks[j].ti;
                tasks[j].ci_left=tasks[j].ci;
                pthread_create(&tasks[j].th,NULL,(void*)proc,&tasks[j].arg);
                tasks[j].flag=2;
            }
        }
    }
    printf("\n");
    sleep(10);
};

void proc(int *args)
{
    while(tasks[*args].ci_left>0)
    {
        pthread_mutex_lock(&proc_wait[*args]); //等待被调度
        if(idle_num!=0)
        {
            printf("idle(%d)",idle_num);
            idle_num=0;
        }
        printf("%c%d",tasks[*args].task_id,tasks[*args].call_num);
        tasks[*args].ci_left--;    //执行一个时间单位
        if(tasks[*args].ci_left==0)
        {
            printf("(%d)",tasks[*args].ci);
            tasks[*args].flag=0;
            tasks[*args].call_num++; //
        }
        pthread_mutex_unlock(&main_wait);    //唤醒主线程
    }
};


void *idle()
{
    while(1)
    {
        pthread_mutex_lock(&idle_wait);  //等待被调度
        printf("->");                   //空耗一个时间单位
        idle_num++;
        pthread_mutex_unlock(&main_wait);          //唤醒主线程
    }
};

int select_proc(int alg)
{
    int j;
    int temp1,temp2;
    temp1=10000;
    temp2=-1;
    if((alg==2)&&(curr_proc!=-1)&&(tasks[curr_proc].flag!=0))
        return curr_proc;
    for(j=0;j<task_num;j++)
    {
        if(tasks[j].flag==2)
        {
            switch(alg)
            {
                case 1:             //EDF算法
                    if(temp1>tasks[j].ci_left)
                    {
                        temp1=tasks[j].ci_left;
                        temp2=j;
                    }
                case 2:          //RMS算法
                    if(temp1>tasks[j].ti)
                    {
                        temp1=tasks[j].ti;
                        temp2=j;
                    }
            }
        }
    }
    return temp2; //return the selected thread or task number
};

```
