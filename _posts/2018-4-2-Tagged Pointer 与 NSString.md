---
layout:     post
title:      
subtitle:   
date:       2018-3-19
author:     Mutong
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - iOS
    - 基础
    - NSString
---

## Tagged Pointer 与 NSString

### 前言

在2013年9月, 苹果推出了iPhone5s, 而且他也是首款采用了64位架构的A7双核处理器, 为了节省内存和提高执行效率, 苹果提出了Tagged Pointer的概念. 对于64位程序, 引入Tagged Pointer后, 相关逻辑减少一半的内存占用, 以及3倍的访问速度提升, 100倍的创建, 销毁速度提升

### 为啥推出Tagged Pointer

假设我们要存储一个NSNumber对象, 其值是一个整数. 正常情况下, 如果这个整数只是一个NSInteger的普通变量, 那么它所占用的内存是与CPU的位数有关的, 在32位CPU下占4个字节, 在64位CPU下是占8个字节的. 而指针类型的大小通常也是与CPU位数相关, 一个指针所占用的内存在32位CPU下为4个字节, 在64位CPU下就是8个字节

所以一个普通的iOS程序, 如果没有Tagged Pointer 对象, 从32位机器迁移到64位机器后, 虽然逻辑没有任何变化, 但是NSNumber, NSDate一类的对象所占的内存会翻倍, 如下图

![别人的图](https://res.infoq.com/articles/deep-understanding-of-tagged-pointer/zh/resources/0519060.jpg)

为了改进上面提到的内存占用和效率问题, 苹果提出了Tagged Pointer. 由于NSNumber, NSDate一类的变量本身的值需要占用的内存大小通常不需要8个字节, 拿整数来说, 四个字节所能表示的有符号证书可以达到20多亿(2^31=2147483648, 有一位作为符号位), 对于绝大数情况都是可以处理的

所以我们可以将一个对象的指针拆为两部分, 一部分直接保存数据, 另一部分作为特殊标记, 表示这是一个特别的指针, 不指向任何一个地方. 所以, 引入了Tagged Pointer之后, 64位CPU下NSNumber的内存图变成了一下这样
![](https://res.infoq.com/articles/deep-understanding-of-tagged-pointer/zh/resources/0519061.jpg)

### Tagged Pointer的特点

* Tagged Pointer 专门用来存储小的对象, 比如 NSNumber和NSDate, 个别NSString
* Tagged Pointer 指针的值不再是地址了, 而是真正的值, 所以, 实际上他不在是一个对象了, 他只是一个披着对象皮的普通变量而已, 所以, 他的内存并不存储在堆上, 也就不需要malloc和free, 也不受引用计数的控制
* 在内存读取上有着3倍的效率, 创建时比以前快了106倍

由此可见, 苹果引入Tagged Pointer, 不但减少了64位机器下程序的内存占用, 还提高了运行效率. 完美的解决了小内存对象在存储和访问效率上的问题


### NSString内存管理

这里是笔记的重点, 当初也是疑惑了很久

OC中的NSString不论是在编译时还是在运行时都做了很多的优化, 并不同于普通的对象, 他是一个非常复杂的存在.

首先定义几个宏定义方便打印观察结果:

```object-c
#if __has_feature(objc_arc)
#define Obj_RetainCount(obj) \
CFGetRetainCount((__bridge CFTypeRef)(obj))
#else
#define Obj_RetainCount(obj) \
DebugLog(@"%lu",[obj retainCount]);
#endif

#define XFLog(_var) NSLog(@"%@ : class = %@ p = %p retainCount = %d",@#_var,NSStringFromClass([_var class]),_var,Obj_RetainCount(_var));

```

测试代码如下:

```object-c
NSString *a = @"str";
NSString *b = [[NSString alloc]init];
NSString *c = [[NSString alloc]initWithString:@"str"];
NSString *d = [[NSString alloc]initWithFormat:@"str"];
NSString *e = [NSString stringWithFormat:@"str"];
NSString *f = [NSString stringWithFormat:@"123456789"];
NSString *g = [NSString stringWithFormat:@"1234567890"];
NSString *h = [NSString stringWithFormat:@"我"];

XFLog(a);
XFLog(b);
XFLog(c);
XFLog(d);
XFLog(e);
XFLog(f);
XFLog(g);
XFLog(h);
```

打印结果

```object-c
a : class = __NSCFConstantString p = 0x10eb5d090 retainCount = -1
b : class = __NSCFConstantString p = 0x10ee90470 retainCount = -1
c : class = __NSCFConstantString p = 0x10eb5d090 retainCount = -1
d : class = NSTaggedPointerString p = 0xa000000007274733 retainCount = -1
e : class = NSTaggedPointerString p = 0xa000000007274733 retainCount = -1
f : class = NSTaggedPointerString p = 0xa1ea1f72bb30ab19 retainCount = -1
g : class = __NSCFString p = 0x7ff449c2a5c0 retainCount = 2
h : class = __NSCFString p = 0x60000002a460 retainCount = 2
```

可以看到, 不同方式创建的字符串类型不同, 引用计数也有所区分, 并不是我们常规理解的对象初始化后引用计数为1, 创建的字符串由三种类型

* __NSCFConstantString
* NSTaggedPointerString
* __NSCFString

之所以有三种类型是由于OC对字符串做的内存优化

#### __NSCFConstantString

对变量类型名上就可以看出, 这种类型的字符串是常量字符串. 该类型的字符串以字面量的方式创建, 保存在字符串常量区(内存的五大区域之一), 是在编译时创建的, 例如:

```oc
NSString *a = @"good afternoon!";
NSString *b = [[NSString alloc]initWithString:@"good afternoon!"];
```

对于`initWithString`实例方法以及`stringWithString`类方法, 编译器会给出`redundant`警告, 原因是该方法创建字符串等同于直接复制字符串字面量

* 当创建的字符串变量值在常量区已经存在时, 会指向那个字符串,这是编译器做的优化
* 由于是常量, 因此其内存管理并不同于对象的内存管理, 引用计数用整形格式打出始终为-1

#### __NSCFString

`__NSCFString`表示对象类型的字符串, 在运行时创建, 保存在堆区, 初始引用计数为1,其内存管理方式就是对象的内存管理方法. 该类型字符串通过format方式创建, 并且字符串内容仅为数字, 字母和常规ASCII字符构成, 且其长度不能太小, 否则创建的是NSTaggedPointerString类型

通过`alloc`才能创建`__NSCFString`

这个小咋定义呢?

* 如果里面是汉字, 那么创建出来的就是`__NSCFString`
* 如果为纯数字需要大于11位, 纯数字也要符合编码方式, 下面会说到八位编码, 六位编码
* 如果为纯字母, 这里需要区分字母, 一会详解 Tagged Pointer的时候会说道

```oc
NSString *d = [[NSString alloc]initWithFormat:@"我是对象"];
NSString *e = [NSString stringWithFormat:@"1234567890"]; //__NSCFString
```

#### NSTaggedPointerString

`NSTaggedPointerString`类型字符串是对`__NSCFString`类型的一种优化, 在运行时创建字符串时, 会对字符串内容及长度做判断, 下面详细解释下


### 采用Tagged Pointer的NSString

看段代码
```oc
    NSMutableString *muStr2 = [NSMutableString stringWithString:@"1"];
    for(int i=0; i<14; i+=1){
        NSString *strFor = [[muStr2 mutableCopy] copy];
        NSLog(@"%@-->%@-->%p",strFor, [strFor class], strFor);
        [muStr2 appendString:@"1"];
    }
```

看输出

```oc
1-->NSTaggedPointerString-->0xa000000000000311
11-->NSTaggedPointerString-->0xa000000000031312
111-->NSTaggedPointerString-->0xa000000003131313
1111-->NSTaggedPointerString-->0xa000000313131314
11111-->NSTaggedPointerString-->0xa000031313131315
111111-->NSTaggedPointerString-->0xa003131313131316
1111111-->NSTaggedPointerString-->0xa313131313131317

11111111-->NSTaggedPointerString-->0xa0079e79e79e79e8
111111111-->NSTaggedPointerString-->0xa1e79e79e79e79e9

1111111111-->NSTaggedPointerString-->0xa03def7bdef7bdea
11111111111-->NSTaggedPointerString-->0xa7bdef7bdef7bdeb

111111111111-->__NSCFString-->0x6000004382e0
1111111111111-->__NSCFString-->0x600000438340
11111111111111-->__NSCFString-->0x600000437740
```

最高位`a`表示类型, 最低位表示字符串长度, 然后字符串内容转为ASCII码存储, (`1`的ASCII为49, 转为16进制为31)

剩余的56位用来存储数组, 这里需要注意的是, 当字符串长度超过56位的时候, Tagged Pointer 并没有立即用指针转向, 而是采用了一种算法编码, 将字符串长度进行压缩存储

采用Tagged Pointer的字符串的结构是

* 如果长度介于0到7, 直接用八位编码存储字符串
* 如果长度是8或9, 用六位编码存储字符串, 使用编码表`eilotrm.apdnsIc ufkMShjTRxgC4013bDNvwyUL2O856P-B79AFKEWV_zGJ/HYX`
* 如果长度是10或11, 使用五位编码存储字符串, 使用编码表`eilotrm.apdnsIc ufkMShjTRxgC4013`

上面的三种情况时要符合编码方式的, 我举个例子, `b`在六位编码和五位编码都是不存在的,一个字符串有`b`和没有`b`是不一样的

看下面这段代码
```oc
    NSMutableString *mutable = [NSMutableString string];
    NSString *immutable;
    char c = 'a';
    for (int i =0; i<14; i++) {
        [mutable appendFormat: @"%c", c++];
        immutable = [mutable copy];
        NSLog(@"%p %@ %@", immutable,immutable, object_getClass(immutable));
    }
```

看输出
```oc
0xa000000000000611 a NSTaggedPointerString
0xa000000000062612 ab NSTaggedPointerString
0xa000000006362613 abc NSTaggedPointerString
0xa000000646362614 abcd NSTaggedPointerString
0xa000065646362615 abcde NSTaggedPointerString
0xa006665646362616 abcdef NSTaggedPointerString
0xa676665646362617 abcdefg NSTaggedPointerString
0xa0022038a0116958 abcdefgh NSTaggedPointerString
0xa0880e28045a5419 abcdefghi NSTaggedPointerString
0x604000439ce0 abcdefghij __NSCFString
0x60000022eac0 abcdefghijk __NSCFString
0x600000027320 abcdefghijkl __NSCFString
0x604000439120 abcdefghijklm __NSCFString
0x604000439c60 abcdefghijklmn __NSCFString
```
10位的时候字符串就变成了`__NSCFString`, 而不是`NSTaggedPointerString`, 因为长度是10或11位的时候, 采用的是五位编码存储字符串

如果咱们把`b`干掉呢,看下面


```oc
 for (int i =0; i<14; i++) {
        if (c != 'b') {
            [mutable appendFormat: @"%c", c++];
            immutable = [mutable copy];
            NSLog(@"%p %@ %@", immutable,immutable, object_getClass(immutable));
         }
        else{
            c++;
        }

    }
```

输出

```oc
0xa000000000000611 a NSTaggedPointerString
0xa000000000063612 ac NSTaggedPointerString
0xa000000006463613 acd NSTaggedPointerString
0xa000000656463614 acde NSTaggedPointerString
0xa000066656463615 acdef NSTaggedPointerString
0xa006766656463616 acdefg NSTaggedPointerString
0xa686766656463617 acdefgh NSTaggedPointerString
0xa0020e28045a5418 acdefghi NSTaggedPointerString
0xa0838a0116950569 acdefghij NSTaggedPointerString
0xa010e5023aa86d2a acdefghijk NSTaggedPointerString
0xa21ca047550da42b acdefghijkl NSTaggedPointerString
0x604000434080 acdefghijklm __NSCFString
0x604000434180 acdefghijklmn __NSCFString
```

看看到了12位才是用的`__NSCFString`, 完美证明

总结: 根据字符串长度已经编码方式确认字符串类型

终于总结明白了, 哭哭哭

### 字符串常量

字符串常量从不存储为Tagged Pointer, 字符串常量必须在不同的操作系统下版本保持二进制兼容, 而Tagged Pointer 的内部细节是没有保证的

```oc
    NSString *a = @"str";
    NSString *b = @"从上面的指针输出可以看出，最低位表示字符串的长度，而其余的56位也是用来存储数组，这里需要注意的是";
    NSString *c = @"str";
    NSString *d = @"str";
    NSString *e = @"a";
    NSString *f = @"ab";
    NSString *g = @"abc";
    NSString *h = @"abcdabcdabcdabcdabcd";
```

上面的输出的类型都是`__NSCFConstantString`

### 参考博客

* [https://blog.csdn.net/Lotheve/article/details/52035477](https://blog.csdn.net/Lotheve/article/details/52035477)
* [http://www.cocoachina.com/ios/20150918/13449.html](http://www.cocoachina.com/ios/20150918/13449.html)
* [http://www.infoq.com/cn/articles/deep-understanding-of-tagged-pointer/](http://www.infoq.com/cn/articles/deep-understanding-of-tagged-pointer/)

 