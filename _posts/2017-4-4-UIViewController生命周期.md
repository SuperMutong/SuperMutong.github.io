---
layout:     post
title:      
subtitle:   
date:       2017-4-4
author:     Mutong
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - iOS
    - 基础
    - NSString
---

## App 相关生命周期总结

居然从印象笔记中翻出来了当年写的笔记了,[23333333333]

#### awakeFromNib
&emsp;&emsp;当 view 被从 StoryBoard 或者 Nib 文件中加载出来时会调用这个方法, 只会在Nib 和 StroyBoard 中所有的对象被创建后调用.

&emsp;&emsp;在这里可以做一些额外的 SetUP 的工作

&emsp;&emsp;当这个方法被调用时代表所有 nib 文件中的对象都已经被创建出来了, 所有的 IBOutletes 和 Action 已经被建立

&emsp;&emsp;此方法覆盖是需要调用 super 方法, 默认的 super 方法里没有实现

&emsp;&emsp;另外, 由于是 Archive 并实例化对象, 所以 ViewController 在初始化时调用的是 initWithCoder:, 如果手动调用 init 则不会从 nib 文件中加载

&emsp;&emsp;这个方法在执行 loadNibNamed: 一类的方法是就会被调用

## loadView()
&emsp;&emsp; ViewConroller 创建后需要加载 self.view 时会调用这个方法, 此方法不应该被直接调用

&emsp;&emsp;如果我们的界面是在 Storyboard 中创建的, 那我们也不应该覆盖这个方法

&emsp;&emsp;当 ViewController 有以下情况时都会在此方法中从 nib 文件加载 View

&emsp;&emsp;&emsp;&emsp;1. ViewController 是从 storyBoard 中实例化

&emsp;&emsp;&emsp;&emsp;2. 通过 initWithNibName:bundle: 初始化

&emsp;&emsp;&emsp;&emsp;3.在 APP Bundle 中有一个 nib 文件名称和本类名相同

&emsp;&emsp;当我们调用 self.view 时, 如果 self.view 非 nil, 那么会直接调用该对象, 如果为 nil, 那么会调用 self.loadView()创建一个 UIVIew 并讲这个对象赋值给 self.view

&emsp;&emsp;在这个函数里调用 self.view 属性会造成死循环, 因为访问 self.view 时发现该属性为空, 回去调用 loadView() 方法, 此时会造成死循环

&emsp;&emsp;此方法覆盖时不该调用 super 方法
## viewDidLoad()
&emsp;&emsp;当 ViewController 的 View 被加载进入内存后会调用这个方法, 因此正常情况下只会调用一次

&emsp;&emsp;在 ViewDidLoad()中可以对 View 进行设置, 例如设定 UIButton 的内容等等, 当 view 是从 nib 中加载出来时, 可以在此设置

&emsp;&emsp;此时 View 还没有被加入 View Hierarchy 中, 只是在加载入内存中, 在这里如果执行 presentViewController 的操作, 会出现出错, 但是如果是 PushViewController 没有问题
此方法覆盖时需要调用 super 方法

## viewWillAppear
&emsp;&emsp;当 View 将要被添加到 View Hierarchy 中时会调用这个方法, 每一次 View 将要显示时都会调用. 在这个方法被调用时, 也是在显示 View 所需要的动画被配置前.

&emsp;&emsp;这个时候在做一些和 frame 相关的操作时仍会出错, 在这里 View 将要被加入 View Hierarchy, 但是依旧没有添加进去.

&emsp;&emsp;此方法覆盖时需要调用 Super 方法

## viewWillLAyoutSubViews
&emsp;&emsp;在 ViewController.view 将要布局 subViews 时调用, 当每一次界面的布局发生变化时都会被调用, 例如旋转

&emsp;&emsp;在这之后, AutoLayout 会改变布局

## viewDidLayoutSubviews
&emsp;&emsp;已经布局完成

&emsp;&emsp;已通过 AutoLayout 布局

## viewDidAppear
&emsp;&emsp;此时页面已经被显示出来了, 在做一些操作时可能会让页面的变化可见

## 消失过程
## viewWillDisappear
&emsp;&emsp; view即将从 superView 中移除

## viewDidDisappear
&emsp;&emsp; view 已经从 superView 中移除
## viewWillUnLoad
&emsp;&emsp; Swift 3.0中没找到这个方法, OC 中还是有的

&emsp;&emsp; 在 VC 中的 view 对象从内存中释放之前调用
## viewDidUnload 
&emsp;&emsp; Swift 3.0中没找到这个方法, OC 中还是有的

&emsp;&emsp; 在 VC 中的 view 对象从内存中释放之后调用
## deinit
&emsp;&emsp;Swift 中的方法, OC 是 dealloc 方法

&emsp;&emsp; 检测内存是否已经释放



# UIView

### 如果 View 是 Xib 创建的, 什么时候才能获取到View 上的子 View 正确的大小?
![](https://ws1.sinaimg.cn/large/006tNc79ly1fq0ocwnftcj30qf0mn74y.jpg)

&emsp;&emsp;1.创建了一个 xib 文件, 里面放了一个 viewA, 加了上下左右的约束分别是0,59,0,83

&emsp;&emsp;2.在 Controller 中实例化这个 view, 使用 [NSBundle MainBundle] 的方法, 并赋予size = 100,100, 理论上 ViewA 的size 应该是{117,141}

&emsp;&emsp;2.在 .h 中声明一个方法 - setUpView, 进行一些数据赋值的操作

&emsp;&emsp;3.看下方红框的输出,

&emsp;&emsp;先调用 -awakeFormNib 在调用 -setUpView 赋值, 在调用 -drawRect 方法

&emsp;&emsp; awakeFromNib 和 setUpView的时候 size 的大小还是 xib 中的大小, 还没有根据父视图的大小进行适配. 只有在 drawRect 的时候才能获取到正确的大小, 如果需要对 layer 的 CornerRadius 进行设置的时候, 最好放在这里

&emsp;&emsp;如果想在 setUpView 进行赋值的时候想要拿到正确的大小的时候, 需要调用 [Self layoutIfNeeded] 方法, 调用完这个方法, 就可以获取到正确的大小

### UIView 的 layoutSubViews 和 drawRect 方法何时调用

####  layoutSubViews 在以下情况下被调用
1. init 初始化不会触发 layoutSubViews
2. addSubView 会触发 layoutSubViews
3. 设置 view 的 frame 会触发 layoutSubViews, 当然前提是 frame 的值设置前后发生了变化
4. 滚动一个 UIScrollView 会触发 layoutSubViews
5. 改变一个 UIView 大小的时候也会触发 UIView 上的 layoutSubViews 事件
6. 直接调用 setLayoutSubViews

####  drawRect 在以下情况调
1. 如果 view 是纯代码创建的, drawRect 方法是在设置大小之后调用, ,如果通过 Xib 创建的无论是否在 VC 中设置大小, 都会走这个方法
2. 该方法在调用sizeTofit后被调用
3. 直接调用setNeedsDisPlay 或者setNeedsDisplayInrect:rect  但是前提是rect不能为0

### App的生命周期

#### 第一次安装

`didFinishLaunchingWithOptions`

`applicationDidBecomeActive`

#### 双击Home进入到任务管理

`applicationWillResignActive`

#### 进入后台

`applicationWillResignActive`

`applicationDidEnterBackground`

#### 从后台进去前台

`applicationWillEnterForeground`

`applicationDidBecomeActive`

#### 杀死App

`applicationDidEnterBackground`

`applicationWillTerminate`

#### 当前App还在存活,但是进入到了后台, 别的App打开当前App

`applicationWillEnterForeground`

`continueUserActivity`

`applicationDidBecomeActive`



 