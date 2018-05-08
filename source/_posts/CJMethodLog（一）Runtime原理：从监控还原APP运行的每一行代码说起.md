---
title: CJMethodLog（一）Runtime原理：从监控还原APP运行的每一行代码说起
date: 2018-03-02
updated: 2018-03-02
---


### 导语：
CJMethodLog 对于Objective-C中的任意类、任意方法，均可实时根据用户的操作行为，监控还原对应的函数调用日志，而且能够自定义记录当前函数的参数类型、返回类型、执行时间……

### 背景说明

是否遇到过如此场景：对于项目中一些不是Crash的问题，由于缺乏log日志，排查起来很是麻烦；又或者对于一些特定设备、特定场景的问题，由于缺乏条件没法重现，最后只能不了了之。比如下面的例子：

你负责开发了一个考勤打卡页面。某天你正当心情愉悦哼着小曲开始一天工作时，产品经理突然闯入：又有用户反馈明明打了卡却提示旷工！并且那用户未能提供当时的操作记录，而且也无法再重现问题。无语的你只好再次默默过了一遍打卡代码，没有发现问题，然后求助后端同事查看接口调用日志，却发现该用户反馈问题的那一天根本就没有相关的接口调用记录……
——最后结论是用户当时忘打卡了（但其实你却不能十足确定打卡代码没有问题）

<!-- more -->

&emsp;&emsp;那是否有这样一种系统：它能够实时记录用户对APP的操作行为，并还原当前操作对应的运行代码，然后将操作记录写入日志，生成一份详细的函数调用堆栈日志。<br>
&emsp;&emsp;事实上，如果是只针对Crash错误，那么使用`NSSetUncaughtExceptionHandler`就完全可以收集到APP崩溃时刻的函数调用堆栈信息了（这点有时间我再起一篇文章说明）。<br>
&emsp;&emsp; [CJMethodLog](https://github.com/lele8446/CJMethodLog)就是这样一个类库：对于Objective-C中的任意类、任意方法，均可实时根据用户的操作行为，监控还原对应的函数调用日志，而且能够自定义记录当前函数的参数类型、返回类型、执行时间等等。
***
### 实现原理
flag立的有点大😅，但是如果你了解Objective-C运行时（Runtime）的语言特性，就知道理论上这是完全可以实现的。

##### 一些基本概念
在xcode中用快捷键shift＋cmd＋O 分别搜索`objc.h` `runtime.h` `NSObject.h`
![isa.jpg](http://upload-images.jianshu.io/upload_images/1429982-ff7f4414d2720f8a.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/350)


可以看出:

* **`NSObject`**是大部分Objective-C类继承体系的基类，有一个私有成员变量isa
* **`id`**实际上是一个`objc_object`结构体指针（这也是为什么使用`id`时不用再在前面加*的原因），id可以指向任何对象
* **`objc_object`**是Objective-C对对象的定义, 其本质上是结构体对象，其中 isa是它唯一的私有成员变量，所有对象都有isa指针
* **`Class`**是一个`objc_class`结构类型的指针
* **`SEL`**(方法选择器)是一个`objc_selector`结构类型的指针，其和C的函数指针有所不同，函数指针直接保存了方法的地址，而SEL只是方法编号，它是Objective-C在编译时，根据方法名字生成的用来区分这个方法的唯一ID。两个类之间，不管是父子类，还是没有关系，只要方法名相同，那么它们的SEL就是一样的。不同类的实例对象执行相同的SEL时，会在各自的方法列表中根据SEL去寻找自己对应的IMP。
* **`IMP`**是一个函数指针，这个被指向的函数包含一个接收消息的对象id(self  指针)，调用方法SEL (方法选择器)，以及不定个数的方法参数，并返回一个id。也就是说 IMP 是消息最终调用的执行代码，我们可以像在Ｃ语言里面一样使用这个函数指针。

再看一下 **`objc_class`** 的定义
![runtime.png](http://upload-images.jianshu.io/upload_images/1429982-136e803ecffe8637.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/400)

* `isa`每个对象结构体的首个成员是Class类的变量，该变量定义了对象所属的类，通常称为isa指针
* `super_class`父类，如果该类已经是最顶层的根类,那么它为nil
* `name`类名
* `version`类的版本信息，默认为0
* `info`供运行期使用的一些位标识
* `instance_size`该类的实例变量大小
* `ivars`用来存储成员变量的数组
* `methodLists`用来存储当前类的方法链表
* `cache`用来缓存用过的方法，提高查找性能
* `protocols`协议链表

看到这里你可能有点晕了。id代表一个objc_object结构体指针，该结构体包含一个isa指针指向所属的类；但是类里面又还有一个isa，那这个isa指向的又是谁呢？这里就引入了Meta Class（元类），Meta Class本身也是一个类，它跟其他类一样也有自己的 isa 和 super_class 指针。

比如对于下面例子，类**Son**继承自**Father**，**Father**继承自**NSObject**

	@interface Father : NSObject
	+ (void)sendMessage;
	@end
	@implementation Father
	+ (void)sendMessage {
	    NSLog(@"sendMessage");
	}
	@end

	@interface Son : Father
	- (void)doSomething:(id)parameter;
	- (void)somethingWrong;
	@end
	@implementation Son
	- (void)doSomething:(id)parameter {
	    NSLog(@"doSomething:");
	}
	@end

那么其结构如下：
![MetaClass.png](http://upload-images.jianshu.io/upload_images/1429982-935b01990aa419c7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/650)

**Meta Class** 结论：

* 每个Class都有一个isa指针指向一个唯一的Meta Class
* 每一个Meta Class的isa指针都指向最上层的Meta Class（图中为NSObject的Meta Class）
* 最上层的Meta Class的isa指针指向自己，从而形成一个回路
* 每一个Meta Class的super class指针指向它原本Class的 Super Class对应的Meta Class。但是最上层的Meta Class的 Super Class指向NSObject Class本身
* 最上层的NSObject Class的super class指向 nil

执行代码，验证一下：

![SonClass.png](http://upload-images.jianshu.io/upload_images/1429982-08cc668a4c5bbcaa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)


* 元类：
 1. `SonClassIsa`和`SonClassMetaClass`是Son类的MetaClass（这里用了两种不同的方式获取MetaClass）
 2. `FatherClassIsa`是Father类的MetaClass
 3. `NSObjectClassIsa`是NSObject类的MetaClass
 4. `MetaClass0`是`SonClassIsa`的MetaClass，`MetaClass1`是`FatherClassIsa `的MetaClass，`MetaClass2`是`NSObjectClassIsa `的MetaClass；`MetaClass0` `MetaClass1` `MetaClass2`都指向`NSObjectClassIsa`本身（四者代表同一个对象）

* 继承关系：
 1. 类：Son -> Father -> NSObject -> nil
 2. 元类: SonClassIsa -> FatherClassIsa -> NSObjectClassIsa -> NSObject -> nil

* **class** 方法：
再来看看 `[self class];` 的低层实现，下载[Rumtime](https://opensource.apple.com/tarballs/objc4/)源码，搜索`NSObject.mm`可以看到class方法的实现如下：
![class.png](http://upload-images.jianshu.io/upload_images/1429982-95c4df53298f0636.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/400)
如果是类方法执行 `[self class];`返回的是当前类本身；如果是实例方法执行 `[self class];`，底层其实执行的是`object_getClass(id obj)`，最终返回的是实例对象的isa指针，这也验证了前面的说法 —— 实例对象的isa指针指向它所在的类。

* **self** 与 **super** ：
或许你已经注意到了，前面获取super class不是调用方法`[super class];`那为什么不呢？事实上，如果你真的那样调用，那么恭喜你！—— 准确掉坑里去了！！
来看一下下面代码，猜猜输出啥：

		@implementation Son: Father
		- (instancetype)init {
			if ([super init]) {
			    NSLog(@"self calss = %@",[self class]);
			    NSLog(@"super calss = %@",[super class]);
			}
			return self;
		}
		@end
你以为是Son和Father，但其实输出的都是**Son** ！！！
要知道，在Objective-C中，self是一个隐藏参数，它指向当前调用方法的对象（receiver）；而super却不是，它只是一个预编译指令。调用super方法，底层执行的是`objc_msgSendSuper`（下一节有详细说明）
  
    	objc_msgSendSuper(struct objc_super * _Nonnull super, SEL _Nonnull op, ...)
    
其中`objc_super `结构体的构成如下：
![objc_super.png](http://upload-images.jianshu.io/upload_images/1429982-09d0bb1ed3ab631f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/400)
receiver表示当前调用方法的对象，super_class表示其父类，`objc_msgSendSuper` 的作用是先构建objc_super结构体，然后从结构体指向的父类开始寻找方法实现，找不到再往上一级父类寻找。找到后最终会转变为以下方法：

    objc_msgSend(objc_super->receiver, @selector(class))
这时候 self 和 objc_super->receiver 表示的是同一个接收者（receiver），同一个接收者执行相同的方法@selector(class)，那结果肯定是相同的。

##### 消息传递
Objective-C中对于方法的调用，其实都是以消息的方式在传递。

    Son *mySon = [Son new];
    //调用方法
    [mySon doSomething:@"parameter"];
    [mySon performSelector:@selector(somethingWrong)];   
    [Son sendMessage];
以上三个方法的调用，编译器最终会将其转换为C函数：

    objc_msgSend(mySon,@selector(doSomething:),@"parameter")
    objc_msgSend(mySon,@selector(somethingWrong))
    objc_msgSend(Son,@selector(sendMessage))

其中：

* mySon叫做接收者(receiver)

* @selector(doSomething:)叫做选择器(selector)，类型为SEL；选择器和参数合起来成为消息(message)
* 当对一个实例对象发送消息时，会在该实例对应的类里查找
* 当对一个类发送消息时，会在该类的 Meta Class 里查找
* **`objc_msgSend `** 是一个参数个数不定的函数，它的作用是向一个实例类发送一个带有简单返回值的消息:

      	objc_msgSend(id _Nullable self, SEL _Nonnull op, ...)
      
  另外还有：`objc_msgSend_stret` 、`objc_msgSendSuper`和`objc_msgSendSuper_stret`。发送给父类的消息会使用`objc_msgSendSuper`，如果方法的返回值是一个结构体(structures)，那么就会使用`objc_msgSendSuper_stret`或者`objc_msgSend_stret`，其他的消息会使用`objc_msgSend`。

   消息传递的时候会先在接收者所属类中的`cache` 中查找方法，如果找不到就在`methodLists ` 中查找，如果找到和选择器名称相符的方法就跳转其实现代码；如果都找不到，就在其父类找，如果该消息无法被该类、其父类、父类的父类……解读，那就会进入消息转发阶段。

下面是以上三个方法调用时的流程图：
![消息传递.png](http://upload-images.jianshu.io/upload_images/1429982-7c2705e1c32c386d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

##### 消息转发

消息转发是Objective-C运行时的一个重要特性，具体表现是当调用一个不存在的方法时，并不会立马Crash，Runtime会有三次挽救的机会（准确的说是1次动态方法解析 + 1次快速消息转发 + 1次完整消息转发；）

调用阶段 | 调用方法 | 备注
:-: | :-: |:-:
动态方法解析 | `+resolveInstanceMethod:` (实例方法)<br><br> `+resolveClassMethod:`(类方法) |这里可以动态添加方法
快速消息转发<br>（也叫备援接收者） | `-forwardingTargetForSelector:` |可以在此将消息转发到指定对象，触发新的消息传递 
完整消息转发 |`-methodSignatureForSelector:`<br><br>`-forwardInvocation:` | 获取签名，并根据方法签名包装成的Invocation，对方法进行处理
消息处理失败|`-doesNotRecognizeSelector:`|抛出异常

接上面，对于调用方法：

    [mySon performSelector:@selector(somethingWrong)];
这是一个没有实现的方法，最终会进入消息转发阶段，那么其完整的执行流程如下：
![objc_msgSend.png](http://upload-images.jianshu.io/upload_images/1429982-90ef042246750ade.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000)

到此为止已经梳理完了Objective-C中完整的方法调用过程。细心的你可能会发现

	- (id)forwardingTargetForSelector:(SEL)aSelector;
	- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector;
	- (void)forwardInvocation:(NSInvocation *)anInvocation;
	- (void)doesNotRecognizeSelector:(SEL)aSelector;
这几个方法都是实例方法，那是否意味着消息转发只针对实例方法有效呢？答案是否定的！前面基本概念的介绍中提到，Classs的isa指针指向的是它的Meta Class，那么意味着一个Class的类方法会加入到它的Meta Class对应的methodLists中，所以你只需要在类中重写下面的类方法，同样可以实现对未知类方法的消息转发。

	+ (id)forwardingTargetForSelector:(SEL)aSelector;
	+ (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector;
	+ (void)forwardInvocation:(NSInvocation *)anInvocation;
	+ (void)doesNotRecognizeSelector:(SEL)aSelector
	
***

### 未完待续
上面已经完整介绍了Objective-C中的消息传递以及消息转发原理，回顾开篇提到的需求，聪明的你是否已经get到了 [CJMethodLog](https://github.com/lele8446/CJMethodLog) 的实现要点，这里先卖个关子。

本来是想只用一篇文章说完所有的，但是写着写着发现篇幅已经不小了，所以另起一篇文章
[CJMethodLog 二：从监控还原APP运行的每一行代码说起](http://lele8446.top/2018/03/22/CJMethodLog%20二：从监控还原APP运行的每一行代码说起/)
来讲解 [CJMethodLog](https://github.com/lele8446/CJMethodLog) 的具体实现。