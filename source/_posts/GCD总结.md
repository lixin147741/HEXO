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

<!--more-->

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


### dispatch_after

`dispatch_after`顾名思义，可以在指定时间结束后执行处理的情况。

举个栗子

```
dispatch_time_t time = dispatch_time(DISPATCH_TIME_NOW, delay * NSEC_PER_SEC);

dispatch_after(time, dispatch_get_main_queue(), ^{
    NSLog(@"waited at least three seconds.");
})
```

不过需要注意的是，`dispatch_after`并不是在指定时间后执行处理，而只是在指定的时间追加处理到Dispatch Queue。上述代码在3秒后用`dispatch_async`函数追加Block到Main Dispatch Queue

因为Main Dispatch Queue在主线程的RunLoop中执行，所以在比如每隔1/60秒执行的RunLoop中，Block最快在3秒后执行，最慢在3秒+1/60秒后执行，并且在主线程有大量追加或主线程本身有延迟时这个时间会更长。


### Dispatch Group

在追加到Dispatch Queue的多个处理全部结束后才执行某个处理，这种情况会经常出现。

当只使用一个串行队列时这种情况比较简单，当使用并行队列或者多个队列的时候就会变得比较复杂。

这种情况下就应该使用Dispatch Group

举个栗子

```
dispatch_queue_t queue = diapatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();

dispatch_group_async(group, queue, ^{NSLog(@"blk0");});
dispatch_group_async(group, queue, ^{NSLog(@"blk1");});
dispatch_group_async(group, queue, ^{NSLog(@"blk2");});

dispatch_group_notify(group, dispatch_get_main_queue(), ^{NSLog(@"done");});

```

执行结果如下：

```
blk1
blk2
blk0 
done
```

无论象什么样的Dispatch Queue中追加处理，使用Dispatch Group都可以见监视这些处理执行的结束。

另外，在Dispatch Group中也可以使用`dispatch_group_wait`函数来等待全部处理执行结束。

```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
dispatch_group_t group = dispatch_group_create();

dispatch_group_async(group, queue, ^{NSLog(@"blk0");});
dispatch_group_async(group, queue, ^{NSLog(@"blk1");});
dispatch_group_async(group, queue, ^{NSLog(@"blk2");});

dispatch_group_wait(group, DISPATCH_TIME_FOREVER);

```

`dispatch_group_wait`的第二个蚕食为超时时间，`DISPATCH_TIME_FOREVER`意味着永久等待。

只要属于Group中的处理尚未结束，就会一直等待，不能取消。

### diapatch_barrier_async

当我们访问数据库或者读写文件的时候难免会遇到数据竞争的问题，这时候用`diapatch_barrier_async`就可以很好的解决这些问题。

举个栗子当我们在一堆读操作中间加一个写操作，我们希望后面的读操作能够正确的读出写入的信息。

```
dispatch_queue_t queue = dispatch_queue_create("com.example.gcd.ForBarrier", DISPATCH_QUEUE_CONCURRENT);

dispatch_async(queue, blk0_for_reading);
dispatch_async(queue, blk1_for_reading);
dispatch_async(queue, blk2_for_reading);
dispatch_async(queue, blk3_for_reading);
dispatch_async(queue, blk4_for_reading);
dispatch_barrier_async(queue, blk_for_writing);
dispatch_async(queue, blk5_for_reading);
dispatch_async(queue, blk6_for_reading);
dispatch_async(queue, blk7_for_reading);
dispatch_async(queue, blk8_for_reading);

```

使用方法非常简单，`dispatch_barrier_async`函数会等待追加到并行队列上的处理全部结束之后再将指定的处理追加到并行队列上，然后等待指定的处理执行完后，队列才恢复一般的操作。

简单的来说，使用了`dispatch_barrier_async`之后，在他之前的处理会先执行完，然后再执行他自己的处理，等他执行完了在执行最后的剩下的处理。

总之使用它可以进行高效率的数据库跟文件访问。（书上说的高效，没测过→_→）


### dispatch_apply

`dispatch_apply`函数将指定的次数的Block追加到DIspatch Queue中，并等待全部处理执行结束。

一般的使用场景就是对一个数组里面的每一个元素都执行一个Block。

### Dispatch Semaphore

当并行执行的处理更新数据的时候，会产生数据不一致的情况。有时应用程序还会崩溃，虽然使用Serial Dispatch Queue跟dispatch_barrier_asyn函数可以避免这类问题，但是有必要进行更细粒度的排他控制。

这个更细的粒度有点类似于操作系统的信号量的概念

举个栗子就明白啦

```
dispatch_queue_t queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);

/*
* 生成Dispatch Semaphore
* 
* Dispatch Semaphore的计数初始值设置为1
* 
* 保证可以访问的NSMutableArray的线程只能有一个
* 
* /

dispatch_semaphore_t semophore = dispatch_semaphore_create(1);

NSMutableArray *array = [[NSMutableArray alloc] init];

for(int i = 0; i < 10000; ++i) {
    
    dispatch_async(queue, ^{
        /*
        * 等待Dispatch Semophore
        * 
        * 一直等待，直到Dispatch Semophore的计数大于等于1
        * /
        
        dispatch_semophore_wait(semophore, DISPATCH_TIME_FOREVER);
        
        
        [array addObject: [NSNumber numberWithInt:i]];
        
        
        dispatch_semophore_signal(semophore);
    });

}

```
解释一下当Dispatch semophore的计数等于0时会一直等待，直到计数大于等于1的时候才会继续执行，继续执行的同时又会把计数减一。最后通过`dispatch_semophore_signal`把计数加一。

### diapatch once

这个函数是保证在应用程序中只执行一次指定处理的API。经常用于单例模式。

```
static dispatch_once_t onceToken;

dispatch_once(&onceToken, ^{

    
});
```



