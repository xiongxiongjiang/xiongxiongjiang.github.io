---
layout:     post
title:      "Notification"
subtitle:   ""
date:       2019-05-23
author:     "xurong"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    
---



## 同步还是异步发送

同步，在哪个线程发，就在哪个线程收。



## addObserver和removeObserver调用多次

addObserver：调用几次，post后就会收到几次。

removeObserver：多次调用不会有影响

在iOS9之前，如果不手动调用removeObserver会造成崩溃，在iOS9之后，可以不调用。

> If your app targets iOS 9.0 and later or OS X v10.11 and later, you don't need to unregister an observer in its deallocation method。



## 在子线程发送的通知，怎么让其在主线程接收到

1. NSNotificationCenter的方法

    ```objective-c
    [[NSNotificationCenter defaultCenter] addObserverForName:@"test" object:nil queue:[NSOperationQueue mainQueue] usingBlock:^(NSNotification * _Nonnull note) {
        //your code
    }];
    ```

2. 在主线程注册一个`machPort`，它是用来做线程通信的，当在异步线程收到通知，然后给`machPort`发送消息

