---
layout:     post
title:      iOS 之 Category
subtitle:   Category
date:       2017-12-17
author:     Mutong
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - iOS
    - 基础
    - Category
---



## Category 学习笔记
#### 	1. `Category` 简介

`Category` 是 Object-C 2.0 以后添加的语言特性, Category的主要作用是为已经存在的类添加方法, 除此之外, 还有另外使用场景

1. 可以把类的实现分开在几个不同的文件里面. 这样做有几个显而易见的好处

		1. 可以减少单个文件的体积
		2. 可以把不同的功能组织到不同的 `Category` 里
		3. 可以由多个开发者共同完成一个类
		4. 可以按需加载想要的 `Category`

1. 声明私有方法
1. 模拟多继承
1. 把 framework 的私有方法公开

#### 2. `Category` 和 `extension` 的区别
1. `extension`看起来很像一个匿名的`Category`
但是`extension`和 有名字的 `Category` 几乎完全是两个东西, `extension`在编译期决议, 他是类的一部分, 在编译器和头文件里的`@interface `以及实现文件里`@implement` 一起形成一个完整的类, 它伴随类的产生而产生, 亦随之一起消亡. `extension` 一般用来隐藏类的私有信息, 你必须有一个类的源码才能为一个类添加 `extension`, 所以你无法为系统的类添加 `extension`
2. `Category` 是在运行期决议的
因为 `Category` 是在运行期决议的, 所以无法添加实例变量, 因为在运行期, 对象的内存布局已经确定, 如果添加实例变量就会破坏类的内部布局, 而且 看 `category_t` 的结构体, 没找到添加实例变量的接口
```
typedef struct category_t{
	const char *name; 类的名字
	calssref_t cls; 类
	struct method_list_t *instanceMethods; 给类添加的实例方法列表
	struct method_list_t *classMethods; 添加的类方法的列表
	struct protocol_list_t *protocols; 实现的所有协议的列表
	stuct property_list_t *instanceProperties; 添加的所有属性
} category_t
```
在 App 启动加载镜像文件时, 会在 `_read_images` 函数间接调用到 `attachCategories` 函数, 完成向类中添加 `Category` 的工作. 原理就是向 `class_rw_t` 中的 `method_array_t`, `property_Array_t` `protocol_Array_t` 数组中分别添加 `method_list_t`, `property_list_t`,`protocol_list_t` 指针
更具体来说 `attachCategories` 做的就是将 `locstamped_category_list_t.list` 列表中每个 `locstamped_category_t.cat` 中的方法, 协议和属性分别添加到类的 `class_rw_t` 对应列表中
 但是 Extension 可以添加实例变量
		
#### 3. `category` 如何加载

1. 把 `Category` 的实例方法, 协议, 以及属性添加到类上
1. 把 `Category` 的类方法和协议添加到类的 metaclass 上
1. 注意点
	1. `Category` 的方法没有 "完全替换掉" 原来类已经有的方法, 也就是说如果 `Category`和原来类都有 `methodA`, 那么 `Category` 附加完成之后, 类的方法列表里会有两个 `MethodA`
	2. `Category` 的方法被放到了新方法列表的前面, 而原来类的方法被放到了新方法列表的后面, 这也就是我们平常所说的 `category` 的方法会 "覆盖"掉原来类的同名方法, 这是因为运行时在查找方法的时候是顺着方法列表的顺序查找的, 一直到对应名字的方法, 就会停止


#### 4. 方法冲突

1. 在类和 `Category` 中都可以有 `+load` 方法, 调用顺序是咋样的

	答: 先类后 `Category`, 而 `Category` 的 `+load` 执行顺序是根据编译顺序决定的, 谁先编译, 先执行谁, 可以在 XCode Compile Sources 中修改
1. 如果多个 `Category` 和 类 有多个相同的方法, 调用顺序是咋样的

	答: 会找到最后一个编译的 `Category` 里对应的方法
1. 在 类的 +load 方法调用的时候, 我们能否调用 `Category` 中声明的方法吗?

	答: 可以的, 因为附加 `Category` 到类的工作会先于 `+ load` 方法的执行
1. `Category` 的方法和 类的方法冲突了, 我想调用类的方法咋办

	答: 在运行时的时候获取类和 `Category` 结合的方法列表, 然后遍历, 获取最后一个方法

#### 		5. 关联对象
1. `Category` 无法添加实例变量, 但是我们可以添加和对象关联的值, 使用 runtime
		  
```objc
ClassA.m
-(void)setName:(NSString *)name{
	objc_setAssocicatedObject(self,"name",name,OBJC_ASSOCIATION_COPY);
}
-(NSString *)name{
	NSString *nameObjct =objc_getAssociatedObject(self, "name");
	return name nameObjct;
}
```
2. 对象销毁的时候如何处理关联对象
	所有的关联对象都是由 `AssociationsManager` 管理, 里面有一个静态 `AssociationsHashMap` 来存储所有的关联对象的, `runtime` 的销毁对象 `objc_destructInstance` 里面会判断这个对象有没有关联对象, 如果有, 会调用 `_object_remove_assocations` 做关联对象的清理方法


#### 6. `Category` 添加属性
1. `Category` 可以添加属性但是不能添加实例变量 (实例变量是成员变量的一种特殊情况)
	1. 其实这句话可以这么理解, 添加属性就是添加`set` `get` 方法, `Category` 可以添加方法, 但是不能添加 实例变量 也就是 _XXX
	2. 我们经常在 iOS 代码中看到在类别中添加属性, 在这种情况下, 是不会自动生成实例变量的
	3. 匿名类别是可以添加实例变量的(我擦, 这个匿名类别就是扩展, 还取名叫匿名类别, 让人糊涂) . 非匿名类别是不能添加实例变量的, 只能添加方法或者属性(非匿名类别就是正常的 `Catrgory`, 去这么多名字, 让人疑惑)
			
1. 为什么不能添加实例变量
	1. 要从 Class 的结构说起, 下面是 Class 的结构体
				
```objc
struct objc_class{
	  Class isa;  指向对象类型
	  Class super_class; 指向父类
	  const char *name; 类名
	  long version; 类的版本信息
	  long info; 供运行期使用的一些位标识
	  long instance_size; 类的实例大小
	  struct objc_ivar_list *ivars; 成员变量的数组
	  struct objc_method_list **methodList; 方法定义的数组, 注意这里是 **
	  objc_cache; 最近使用的方法, 用于方法调用的优化
	  struct objc_protocol_list *protocols; 协议的数组
}

`methodList`: 这是方法的定义列表, 是指针的指针, 所以可以通过修改过该指针指向的指针的地址, 来动态增加方法, 这也就是 `Category` 的实现原理
简单说 `Category` 设计的目的就是用来扩展类功能, 而非封装数据,所有的属性和成员变量应该放在主接口 `(main interface)`, 才能使类的设计更加清晰
`ivars`: 存储对象成员变量的指针, 所以无法动态增加成员变量

```

来个例子讲解下如果在 `Category` 中添加属性
	先了解下一会要用到的一个方法

```objc
OBJC_EXPORT void objc_setAssociatedObject(id object, const void *key, id value, objc_AssociationPolicy policy) __OSX_AVAILABLE_STARTING(__MAC_10_6, __IPHONE_3_1);

object 源对象
key 关联的键, objc_getAssociatedObject 通过这个 key 取出被关联的对象
value 被关联的值
policy 一个枚举值, 表示关联对象的行为, 下面是全部的枚举
```

```objc
typedef OBJC_ENUM(uintptr_t, objc_AssociationPolicy) {
    OBJC_ASSOCIATION_ASSIGN = 0,           /**< Specifies a weak reference to the associated object. */
    OBJC_ASSOCIATION_RETAIN_NONATOMIC = 1, /**< Specifies a strong reference to the associated object. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_COPY_NONATOMIC = 3,   /**< Specifies that the associated object is copied. 
                                            *   The association is not made atomically. */
    OBJC_ASSOCIATION_RETAIN = 01401,       /**< Specifies a strong reference to the associated object.
                                            *   The association is made atomically. */
    OBJC_ASSOCIATION_COPY = 01403          /**< Specifies that the associated object is copied.
                                            *   The association is made atomically. */
};
```

首先在 .h 文件中添加一个属性 
		
```objc
@property (nonatomic, copy) NSString *name;
``` 

在 .m 文件中添加 set 和 get 方法, 首先声明一个静态变量

```objc
static const char associatedKey
```

实现 get 和 set 方法
```objc
//Category中的属性，只会生成setter和getter方法，不会生成成员变量

- (void)setName:(NSString *)name {
    objc_setAssociatedObject(self, &associatedKey, name, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSString *)name {
    return objc_getAssociatedObject(self, &associatedKey);
}
```

有的时候根据需求不同,可能要移除被关联的对象, 如果移除呢? 有两种方式

1. 移除源对象中所有的关联对象

	 `objc_removeAssociatedObjects(self);`

2. 移除定义的关联对象, 只需要把value 置为 nil 就可以了

	`objc_setAssociatedObject(self, &associatedKey, nil, OBJC_ASSOCIATION_COPY_NONATOMIC);`


#### 6. 换肤功能
我司在 App 中增加了一个换肤的功能, 当初就是用的 runtime + RAC 实现的,说下大概实现的方法, 重写所有和 UI 有关的系统方法,  比如 UIview 的 setBackgroundColor 的方法,  UILabel 的 setTitltColor 的方法, 在皮肤发生变化的时候, 重新去颜色表中读取颜色, UI 重新布局
		
下面以 UIView 的 backgroundColor 为例子, 其中涉及到了  RAC 相关的知识, 就不在这里介绍了
	下面开始:

		1. 创建 UIView 的类别  UIView+Theme 文件
		2. 在 .h 文件中声明设置背景色的方法

```
	@param scheme 色系
	- (void)setBackgroundSchemeNew:(NSString *)scheme;
```

		3.  在 .m 文件中声明静态变量

```
	static const char backgroundScheme;
```

		4. 实现方法
	
```objc
	- (void)setBackgroundScheme:(NSString *)scheme{
	    //去掉重复添加
	    NSString *background = objc_getAssociatedObject(self, &backgroundScheme);
	    if ([background isEqualToString:scheme]) {
	        return;
	    }
	    
	    objc_setAssociatedObject(self, &backgroundScheme, scheme, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
	    @weakify(self);
	    //下面是我司用 RAC 实现监听皮肤改版的信号, 等皮肤改变了会触发这个信号
	    [[[SignalCenter sharedInstance] themeStatusSignal] subscribeNext:^(id x) {
	        @strongify(self);
	        [self changeThemeBackground];
	    }];
	    [self changeThemeBackground];
	}
	//接受到换肤信号之后会执行这个方法
	- (void)changeThemeBackground{
	    NSString *background = objc_getAssociatedObject(self, &backgroundScheme);
	    //ResourceManager 是我们存储颜色的单例, 暴露各种颜色的接口
	    ResourceManager *manager = [ResourceManager sharedInstance];
	    SEL seletor = NSSelectorFromString(background);
	    if ([manager respondsToSelector:seletor]) {
	        self.backgroundColor = [manager performSelector:seletor];
	    }
	}
```
## 补充
##### 实例变量和属性, 成员变量

1. 属性就是我们正常声明的 `@property (nonatomic, strong) UIButton *btn`
1. 实例变量 `_btn`
1. 成员变量是定义在 `@interface XX {}` 中的变量, 如果变量的数据类型是第一个类, 则称这个变量为实例变量, 可以这么理解, 实例变量本质上就是成员变量, 只是实例是针对类而言, 实例是指类的声明
1. 实例变量 + 基本数据类型变量 = 成员变量


## 参考博客

* [https://tech.meituan.com/DiveIntoCategory.html](https://tech.meituan.com/DiveIntoCategory.html) 
* [http://www.cnblogs.com/crazypebble/p/3439261.html](http://www.cnblogs.com/crazypebble/p/3439261.html) 
* [https://www.cnblogs.com/Jenaral/p/5970393.html](https://www.cnblogs.com/Jenaral/p/5970393.html)

