---
title: ReactiveCocoa踩坑实战
date: 2016-03-16 21:41:35
tags:
---

> 先吐个槽，刚才ssh连接github的时候遇到了 `ssh_exchange_identification: read: Connection reset by peer` 的错误，查了半天资料也没找到问题，最后是因为开着代理VPS限制了TCP最大连接数所以失败的，关闭代理就可以使用了。

### 前言

---

几个月前看了RxSwift看的是云里雾里，还去作死翻译了一下官方的文档结果是翻译的一塌糊涂，最近又开始学习OC还是对响应式函数编程念念不忘，于是去踩坑ReactiveCocoa，看了一天加上之前的基础没那么难懂了。

<!--more-->

### 个人理解

响应式函数编程的编程思想跟之前的有很大的不同，我们之前理解的编程都是基于数据的，数据是不变的，我们通过编程来操作数据，我们要随时记录这数据的状态。而响应式函数编程的核心思路是面向流的，数据是变化的，形象的来说我们通过来构建一个**管道**来让数据流通过，我们不需要记录数据的状态。



RACSignal有两种状态，cold跟hot状态，通俗的说就是休眠跟激活状态。一般情况下一个RACSignal创建出来都是休眠状态，有人去订阅（subscribe）才会被激活。

RACSignal能且只能产生三种事件， next， complete， error

- next表示这个Signal产生了一个值
- completed表示Signal结束，结束信号只标志成功结束，不带值
- error表示Signal中出现错误，立刻结束

有很多Operation，可以参考https://github.com/ReactiveCocoa/ReactiveCocoa/blob/master/Documentation/BasicOperators.md#filtering



Reactive cocoa使用信号来代表一系列异步事件，提供了一种统一的方式来处理所有异步的行为，包括代理方法、`block` 回调、`target-action` 机制、通知、`KVO` 等。

``` objective-c
// 代理方法
[[self
    rac_signalForSelector:@selector(webViewDidStartLoad:)
    fromProtocol:@protocol(UIWebViewDelegate)]
    subscribeNext:^(id x) {
        // 实现 webViewDidStartLoad: 代理方法
    }];

// target-action
[[self.avatarButton
    rac_signalForControlEvents:UIControlEventTouchUpInside]
    subscribeNext:^(UIButton *avatarButton) {
        // avatarButton 被点击了
    }];

// 通知
[[[NSNotificationCenter defaultCenter]
    rac_addObserverForName:kReachabilityChangedNotification object:nil]
    subscribeNext:^(NSNotification *notification) {
        // 收到 kReachabilityChangedNotification 通知
    }];

// KVO
[RACObserve(self, username) subscribeNext:^(NSString *username) {
    // 用户名发生了变化
}];

```







明天在学一学，多看一些资料再来写