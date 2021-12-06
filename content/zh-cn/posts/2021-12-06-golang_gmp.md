+++
title= "Golang GMP调度"
description= "GMP for golang"
date = "2021-12-06"
tags = [
    "golang"
]
categories = [
  "技术"
]
series = [
  "概念"
]
+++

### 一、GMP “调度器” 的由来

##### 单进程时代不需要调度器
&emsp;早期的操作系统每个程序就是一个进程，直到一个程序运行完，才能进行下一个进程，就是 “单进程时代”.自动化控制中的PLC也是典型的单进程操作系统。

##### 多进程 / 线程时代有了调度器需求
&emsp;典型的操作系统unix/linux,对进程/线程进行了管理。

##### goroutine是比线程更轻量级的协程
&emsp;goroutine是golang独有的概念,是为了更少的内存和cpu开销，比线程更加轻量，可以看作是线程的"son".很明显操作系统管理不了，所以出现了GMP.

### 二、GMP 定义
&emsp; G=goroutine,M=mechine,P=processor.
  ![image](images/post/golang/GMP-define.webp)

### 三、GMP 模型
&emsp;在 Go 中，线程(M)是运行 goroutine 的实体，调度器的功能是把可运行的 goroutine 分配到工作线程上。
  ![image](images/post/golang/GMP-model.webp)
  - 全局队列（Global Queue）：存放等待运行的 G。
  - P 的本地队列：同全局队列类似，存放的也是等待运行的 G，存的数量有限，不超过 256 个。新建 G时，G优先加入到 P 的本地队列，如果队列满了，则会把本地队列中一半的 G 移动到全局队列。
  - P 列表：所有的 P 都在程序启动时创建，并保存在数组中，最多有 GOMAXPROCS(可配置) 个。
  - M：线程想运行任务就得获取 P，从 P 的本地队列获取 G，P 队列为空时，M 也会尝试从全局队列拿一批 G 放到 P 的本地队列，或从其他 P 的本地队列偷一半放到自己 P 的本地队列。M 运行 G，G 执行之后，M 会从 P 获取下一个 G，不断重复下去。
  

### 三、GMP 设计策略
- 复用线程：避免频繁的创建、销毁线程，而是对线程的复用。
  - work stealing 机制  
  &emsp; 当本线程无可运行的 G 时，尝试从其他线程绑定的 P 偷取 G，而不是销毁线程。
  - hand off 机制  
  &emsp;当本线程因为 G 进行系统调用阻塞时，线程释放绑定的 P，把 P 转移给其他空闲的线程执行。

- 利用并行：GOMAXPROCS 设置 P 的数量，最多有 GOMAXPROCS 个线程分布在多个 CPU 上同时运行。GOMAXPROCS 也限制了并发的程度，比如 GOMAXPROCS = 核数/2，则最多利用了一半的 CPU 核进行并行。golang 1.5+版本后，默认GOMAXPOCS=cpu总核数.

- 抢占：在 coroutine 中要等待一个协程主动让出 CPU 才执行下一个协程，在 Go 中，一个 goroutine 最多占用 CPU 10ms，防止其他 goroutine 被饿死，这就是 goroutine 不同于 coroutine 的一个地方。

- 全局 G 队列：在新的调度器中依然有全局 G 队列，但功能已经被弱化了，当 M 执行 work stealing 从其他 P 偷不到 G 时，它可以从全局 G 队列获取 G。

### 四、go func () 调度流程    
  ![image](images/post/golang/GMP-gofunc.webp)

从上图我们可以分析出几个结论：

  - 1、我们通过 go func () 来创建一个 goroutine；

  - 2、有两个存储 G 的队列，一个是局部调度器 P 的本地队列、一个是全局 G 队列。新创建的 G 会先保存在 P 的本地队列中，如果 P 的本地队列已经满了就会保存在全局的队列中；

  - 3、G 只能运行在 M 中，一个 M 必须持有一个 P，M 与 P 是 1：1 的关系。M 会从 P 的本地队列弹出一个可执行状态的 G 来执行，如果 P 的本地队列为空，就会想其他的 MP 组合偷取一个可执行的 G 来执行；

  - 4、一个 M 调度 G 执行的过程是一个循环机制；

  - 5、当 M 执行某一个 G 时候如果发生了 syscall 或则其余阻塞操作，M 会阻塞，如果当前有一些 G 在执行，runtime 会把这个线程 M 从 P 中摘除 (detach)，然后再创建一个新的操作系统的线程 (如果有空闲的线程可用就复用空闲线程) 来服务于这个 P；

  - 6、当 M 系统调用结束时候，这个 G 会尝试获取一个空闲的 P 执行，并放入到这个 P 的本地队列。如果获取不到 P，那么这个线程 M 变成休眠状态， 加入到空闲线程中，然后这个 G 会被放入全局队列中。

