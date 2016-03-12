---
title: GCD总结
date: 2016-03-12 17:44:14
tags:
---


## GCD的API以及基本概念

---

- 串行队列（Serial Dispatch Queue）
一次只执行一个线程，按照添加到队列的顺序依次执行
- 并发队列(Concurrent Dispatch Queue）
一次可以执行多个线程，线程的执行没有先后顺序

### dispatch_queue_create

那么如何创建Dispatch Queue呢，一共有两种方式。

- 第一种方式：通过`dispatch_queue_create`来创建

```
dispatch_queue_t mySerialDispatchQueue = dispatch_queue_create("com.example.gcd.MySerialDispatchQueue", NULL);

```
`dispatch_queue_create`这个函数第一个参数指定队列的名称，该名称在Xcode跟Instruments的调试器中作为Dispatch Queue的名称表示，另外该名称也出现在应用程序崩溃时所声称的CrashLog中。

第二个参数表示队列的类型，DISPATCH_QUEUE_SERIAL就是NULL，而如果想要创建并行队列则要写成`DISPATCH_QUEUE_CONCURRENT`

在6.0SDK之前还需要通过`地下patch_retain`跟`dispatch_release`来手动管理队列，在6.0之后就不需要了。

- 第二种方式：获取系统标准提供的Dispatch Queue

如果不用特地生成Dispatch Queue， 系统也会给我们提供几个，那就是`Main Dispatch Queue`和`Global Dispatch Queue`

各种Dispatch Queue的获取方法如下：

```
/*
* Main Dsspatch Queue的获取方法
*/
dispatch_queue_t mainDispatchQueue = dispatch_get_main_queue();

/*
* Global Dsspatch Queue（高优先级）的获取方法
*/
dispatch_queue_t globalDispatchQueueHigh = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0);

/*
* Global Dsspatch Queue（默认优先级）的获取方法
*/
dispatch_queue_t globalDispatchQueueHigh = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

/*
* Global Dsspatch Queue（低优先级）的获取方法
*/
dispatch_queue_t globalDispatchQueueHigh = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_LOW, 0);

/*
* Global Dsspatch Queue（后台优先级）的获取方法
*/
dispatch_queue_t globalDispatchQueueHigh = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);

```

### dispatch_set_target_queue

通过`dispatch_queue_create`创建的队列不管是串行的还是并行的都是默认优先级的，如果想要改变队列的优先级就需要使用`dispatch_set_target_queue`函数。

举个栗子

```
dispatch_queue_t mySerialDispatchQueue = dispatch_queue_create("com.example.gcd.MySerialDispatchQueue", NULL);

dispatch_queue_t globalDispatchQueueBackground = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_BACKGROUND, 0);

dispatch_set_target_queue(mySerialDispatchQueue, globalDispatchQueueBackground);
```

在必须将不可并行执行的的处理追加到多个Serial Dispatch Queue中时，如果使用dispatch_set_target_queue函数将目标指定为某一个Serial Dispatch Queue，即可方式处理并行执行。



