---
layout:     post
title:      RunLoop 学习笔记
subtitle:   Category
date:       2018-1-22
author:     Mutong
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - iOS
    - 基础
    - RunLoop
---

 
## RunLoop 学习笔记
#### 	1. `Runloop` 基本概念
一个RunLoop是一个时间处理环, 系统利用这个事件处理环来安排事物, 协调输入的各种事件
RunLoop 的目的是让你的线程在有工作的时候忙碌, 没有工作的时候休眠, 节省 CPU 时间
线程和RunLoop 之间是以键值对的形式一一对应的, 其中key 是 thread,value 是 RunLoop, 一句话, RunLoop 是管理线程的一种机制


RunLoop 实际上就是一个对象, 这个对象管理了其需要处理的事件和消息, 并提供了一个入口函数来执行 Event Loop 的逻辑, 线程执行了这个函数后, 就会一直处于这个函数内部"接受消息->等待->处理"的循环中, 知道这个循环结束(比如传去 quit 的消息), 函数返回
OSX/iOS 系统中,提供了两个这样的对象: NSRunLoop 和 CFRunLoopRef.
CFRunLoopRef 是在 CoreFoundattion 框架内的, 他提供了纯 C 函数的 API, 所有这些 API 都是线程安全的
NSRunLoop 是基于 CFRunLoopRef 的封装, 提供了面向对象的 API, 但是这些 API 不是线程安全的


CADisplay CAAniamtion CAAnimation 
AFNetworking 都和 RunLoop 有关系
RunLoop 和 Thread 的关系
NSTimer 和 performSelector..after..  CADisplay 都是 RunLoopTimer 的封装

1. 线程与 runLoop 的关系
    runLoop 是为了线程而生, 没有线程, 他就没有存在的必要. RunLoop 是线程的基础框架部分, Cocoa 和 coreFundation 都提供了 runloop 对象方便配置和管理线程的 runLoop, 每个线程, 包括程序的主线程( main thread) 都有能与之对应的 runLoop 对象.
    苹果不允许直接创建 RunLoop, 他只提供了两个自动获取的函数: CFRunLoopGetMain() 和 CFRunLoopGetCurrent().
    线程和 RunLoop 之间是一一对应的, 其关系保存在一个全局的 Dictionary 里. 线程刚创建时并没有 RunLoop, 如果你不主动获取, 那他一直都不会有. RunLoop 的创建是发生在第一个获取时, RunLoop 的销毁时发生在线程结束时, 你只能在一个线程的内部获取其 RunLoop (主线程除外)
2. 主线程的 默认是启动的
    ios的应用程序里, 程序启动后会有一个如下的 main()函数:
```    
int main(int argc, char *argv[]){
    @autoreleasepool{
    return UIApplicationMain(argc,argv,nil,NSStringFromClass([appDelegate class]));
}
}      
```
    重点是 UIApplicationMain()函数, 这个方法会为 mainThread 设置一个 NSRunLoop 对象, 我们的应用可以再无人操作的时候休息, 需要让他干活的时候又能马上响应
3. 对于其他线程来说,  runLoop 默认是没有启动的, 如果你需要更多的线程交互则可以手动配置和启动, 如果线程只是去执行一个长时间已确定的任务则不需要
4. 在任何一个Cocoa 程序的线程中, 都可以通过:
    ```
    NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
    ```  
    来获取到当前线程的 run Loop
5. RunLoop 同时也负责 autoRelase Pool 的创建和释放
    在使用手动的内存管理的项目中, 会经常用到很多内存是否的对象, 如果这些对象不能够被及时释放掉, 会造成内存占用量急剧增大. RunLoop 就为我们做了这样的工作, 每当一个运行循环结束的时候, 他都会释放一次 autoRelease Pool, 同时 pool 中的所有动态释放类型变量都会被释放掉
6. RunLoop 的优点
    一个 runLoop 就是一个时间处理循环, 用来不停的监听和处理输入事件并将其分配到对应的目标上进行处理
    首先, 它并不是一个简单的 while 循环, 他是一种更加高明的消息处理模式, 他就高明在对消息处理进行了更好的抽象和封装,这样你就不用处理一些很琐碎很低层次的具体消息的处理
    其实, 也是最重要的一点, 使用 runLoop 可以使你的线程在有工作的时候工作, 没有工作的时候休眠, 这可以大大节省系统资源


孙源的意思
AutoReleasePool 什么时候释放 
    在没有手加 AutoRelease Pool 的情况下, AutoRelease 对象是在当前的 RunLoop 迭代结束时释放的, 而它能够释放的原因是系统在每个 RunLoop 迭代中都加入了自动释放池 Push 和 Pop
App 启动后, 系统在主线程 RunLoop 里注册两个 Observer, 其回调都是 _wrapRunLoopAutoReleasePoolHandler()
1. 第一个 Observer 监听事件
    是 Entry( 即将进入 Loop), 其回调会调用 _objc_autoreleasePoolPush() 创建自动释放池. 其优先级最高, 保证创建释放池发生在其他所有回调之前
2. 第二个 Observer 监听了两个事件
    1. _BeforeWaiting( 准备进入休眠)时
        调用 _objc_autoReleasePoolPop() 和 _objc_autoReleasePoolPush() 释放旧的池并创建新池
    2. _Exit( 即将退出 Loop) 时
        调用 _objc_autoreleasePoolPop() 来释放自动释放池, 这个 Observer 优先级最低, 保证其释放自动释放池发生在其他所有回调之后

AutoReleasePool 是在RunLoop 即将进入 Runloop 和准备进入休眠这两种状态的时候被创建和销毁的

https://www.jianshu.com/p/405b8dfc3829

使用容器的 block 版本的枚举器时, 内部会自动添加一个 AutoReleasePool:
```
[array enumerateObjctsUsingBlock:^(id obj, NSUInteger idx, BOOL *stop){
        //这里被一个局部@ autoreleasepool 包围着
    }]
```






























 



