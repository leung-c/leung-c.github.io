---
layout: post
title: objective-c中的对象、类和元类
date: 2017-07-02
categories: blog
tags: [objective-c,runtime]
description: 什么是OC中的对象、类、元类，它们被如何表示，它们之间的关系。
---

> 本问粗浅探讨了OC中对象、类、元类的数据定义和他们之间的关系，梳理了相关概念

###什么是OC中的对象、类、元类

本文开始之前先看一下下面四个习题：

######1.下面代码的输出是什么？
```C
@interface BlackCat : Cat
@end
@implementation BlackCat
 - (instancetype) init{
    self = [super init];
    if(self){
        NSLog(@"%@",NSStringFromClass([self class]));
        NSLog(@"%@",NSStringFromClass([super class]));
    }
    return self;
}
```
######2.下面代码的输出是什么？ 
```C
 @interface Animal : NSObject
 @property(copy,nonatomic) NSString* name;
 @end
 @implementation Animal
 @end
  BOOL res1 = [(id)[NSObject class] isKindOfClass:[NSObject class]];
        BOOL res2 = [(id)[NSObject class] isMemberOfClass:[NSObject class]];
        BOOL res3 = [(id)[Animal class] isKindOfClass:[Animal class]];
        BOOL res4 = [(id)[Animal class] isMemberOfClass:[Animal class]];
        NSLog(@"%d %d %d %d", res1, res2, res3, res4);

```

######3.下面代码会编译能否通过？运行会闪退？输出是什么？ 
```C
@interface NSObject (Animal)
+ (void) eat;
@end
@implementation NSObject (Animal)
- (void)eat
{
    NSLog(@"IMP: -[NSObject(Animal) eat]");
}
[NSObject eat];
 [[NSObject new] eat];
 ```
######4.下面代码会编译能否通过？运行会闪退？输出是什么？ 
```C
@interface WhiteCat : Cat
- (void)speak;
@end
@implementation WhiteCat
- (instancetype) init{
    self = [super init];
    if (self) {
        id cls = [WhiteCat class];
        void *obj = &cls;
        [(__bridge id)obj speak];
    }
    return self;
}
- (void) speak{
    NSLog(@"I'm a white cat");
}
```

#####什么数据结构才能称之为对象？
在面向对象的语言体系中对象指的是类的实例，每个对象都有类。这是面向对象的基本概念，那么对象和类采用什么样的数据结构进行表示的呢？在Objective-C中，含有一个指针且该指针可以正确指向类的数据结构，都可以被视作为对象。
在Objective-C中，对象最基本的定义是这样的：
```C
/// Represents an instance of a class.
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
/// A pointer to an instance of a class.
typedef struct objc_object *id;
```
objc_object结构体表示某个类的一个实例(对象)，id表示objc_object类型的结构体指针。对象的类型是 isa指针决定的，它指向对象所属的类。其实字面意思就是“is a”。表示：“是一个什么什么”
这意味着：任何带有以指针开始并指向其所属类的结构都可以被视作objc_object。可以看到，对象的表示非常简单，它本身只携带所属类的信息，这是因为对象有哪些属性、成员变量、哪些方法都可以通过isa指针从它的父类中找到，对象本身对应一块内存区域，通过alloc init创建的对象会在丢中开辟一块内存区域，里面保存着对象的属性值或值的引用，一个对象的内存布局大致为下图： 

![](https://ws4.sinaimg.cn/large/006tKfTcgy1flrucylj2lj302104udft.jpg)

即：

* ISA指针
* 根类的实例变量
* 倒数第二层父类的实例变量
* ...
* 父类的实例变量
* 类的实例变量

一个对象在创建的时候分配内存时，会根据他的类型会计算出它需要的内存大小，这正是对象的alloc方法做的事情。根据前面的定义对象实际是一个结构体实例，因此该结构体的大小并不能动态变化。这也是为什么对象创建后增加成员变量。当然通过objc_setAssociatedObject 和 objc_getAssociatedObject方法可以变相地给对象增加成员变量，但由于实现机制不一样，所以并不是真正改变了对象的内存结构。
> 简单了解一下objc_setAssociatedObject的实现机制：每一个对象地址对应一个 ObjectAssociationMap 对象，而一个 ObjectAssociationMap 对象保存着这个对象的若干个关联记录。一条关联记录保存着一个被关联对象、内存管理策略等信息。

当我们调用对象的方法时，实际是使用objc_msgSend函数进行消息发送，运行时库会追寻着对象的isa指针得到了对象所属的类。这个类包含了能应用于这个类的所有实例方法和指向超类的指针以便可以找到父类的实例方法。运行时库检查这个类和其超类的方法列表，找到一个匹配这条消息的方法。运行时库基于那个方法调用函数（IMP）。如果最终没有在其继承链中找到对应的方法就会进入消息转发流程。
self和supper的决定了是编译时方法调用是使用objc_msgSend还是的objc_msgSendSuper，objc_msgSendSuper将直接从superClass中查找对应方法。self 是类的隐藏参数，指向当前调用方法的实例（对象或类对象）。而 super 是一个 Magic Keyword， 它本质是一个编译器标示符，和 self 是指向的同一个消息接受者。上面的例子不管调用[self class]还是[super class]，接受消息的对象都是当前 Animal ＊xxx 这个对象。而不同的是，super是告诉编译器，调用 class 这个方法时，要去父类的方法，而不是本类里的。当使用 self 调用方法时，会从当前类的方法列表中开始找，如果没有，就从父类中再找；而当使用 super 时，则从父类的方法列表中开始找。然后调用父类的这个方法。

类的定义 
```C
struct objc_class {
    Class isa  OBJC_ISA_AVAILABILITY;

    Class super_class   OBJC2_UNAVAILABLE;
    const char *name    OBJC2_UNAVAILABLE;
    long version        OBJC2_UNAVAILABLE;
    long info           OBJC2_UNAVAILABLE;
    long instance_size  OBJC2_UNAVAILABLE;
    struct objc_ivar_list *ivars OBJC2_UNAVAILABLE;
    struct objc_method_list **methodLists OBJC2_UNAVAILABLE;
    struct objc_cache *cache  OBJC2_UNAVAILABLE;
    struct objc_protocol_list *protocols OBJC2_UNAVAILABLE;

} OBJC2_UNAVAILABLE;

```
从定义可以看出：它实际上是一个指向objc_class结构体的指针。这个结构体的成员中我们看一下下面几个的含义：

1. isa：其实字面意思就是“is a”，这个字断类型是Class，表示是当前类一个什么类，需要注意的是在Objective-C中，所有的类自身也是一个对象，这个对象的Class里面也有一个isa指针，它指向metaClass(元类)，我们会在后面介绍它。
2. super_class：指向该类的父类
3. cache：缓存最近使用的方法。一个接收者对象接收到一个消息时，它会根据isa指针去查找能够响应这个消息的对象。在实际使用中，这个对象只有一部分方法是常用的，很多方法其实很少用或者根本用不上。这种情况下，如果每次消息来时，我们都是methodLists中遍历一遍，性能势必很差。这时，cache就派上用场了。在我们每次调用过一个方法后，这个方法就会被缓存到cache列表中，下次调用的时候runtime就会优先去cache中查找，如果cache没有，才去methodLists中查找方法。这样，对于那些经常用到的方法的调用，但提高了调用的效率。
4. ivars：用来表示instance variable(实例变量，跟某个对象关联，不能被静态方法使用，与之想对应的是class variable)，其声明如下，类中定义的属性其实本质就是成员变量，只是编译器会自动帮我们加上对应的getter和setter
```C
 struct objc_ivar {
     char *ivar_name   OBJC2_UNAVAILABLE;
     char * ivar_type  OBJC2_UNAVAILABLE;
     int ivar_offset   OBJC2_UNAVAILABLE;
    #ifdef __LP64__
     int space  OBJC2_UNAVAILABLE;
    #endif OBJC2_UNAVAILABLE;
    }
```
> objc_ivar保存了对象的成员变量的名称、类型、地址偏移量的关键信息，
5. methodLists：用来表示instance methods(实例方法，跟某个对象关联，不能被静态方法使用，与之想对应的是class method)，其声明如下：
```C
struct objc_method {
    SEL method_name    OBJC2_UNAVAILABLE;
    char *method_types                                       OBJC2_UNAVAILABLE;
    IMP method_imp                                           OBJC2_UNAVAILABLE;
}
```
这里实际维护的是方法名(SEL)和方法实现(IMP)的映射关系，也是为什么我们可以做到方法替换关键。
从以上的定义可以看出对象的属性、方法的核心其实在于类，它决定对象具备哪些行为和能力。那么类方法和类静态变量是在哪里维护的呢？这涉及我们平时编码中几乎不会接触的一个概念：元类(MetaClass)，元(Meta)表示用于描述数据的数据，意味着元类是用于描述类的，类描述了对象有哪些属性特征和行为，同样元类描述了类有哪些属性特征和行为。
类其实也是一种对象，我们称之为：类对象，所以和对象一样它有isa指针指向它所属的类（元类）。isa指针的类型是Class，说明元类和类用的是同一种数据结构，Objective-C的设计哲学中，一切都是对象，那么元类也可以看做为对象，这和万物皆对象的思想是一致的。这里我们可以思考一个问题：对象和类的isa指向它们所属的类，那元类的isa指针呢？按照Class的定义isa指向所属类，而元类本身也是Class，isa的指针不可能无限指向新的类。
我们可以写代码验证一下： 
```C
   NSObject* obj = [[NSObject alloc]init];
        //类
        Class instanceClass = [obj class];
        //元类（类的isa指向）
        Class metaOfInstanceClass = object_getClass(instanceClass);
        //元类的类（元类的isa指向）
        Class rootMetaCls = object_getClass(metaOfInstanceClass);
```
在控制台调制可以看到以下信息： 
![](https://ws4.sinaimg.cn/large/006tKfTcgy1flrysed41tj30dg03j751.jpg)

NSObject元类的isa指向了自己，这里我们需要怀疑的是NSObject是所有类的根类，它的元类是否具有特殊性，就像它没有父类一样，我们新建一个类试一试： 
```c
Animal* animal = [[Animal alloc]init];
        //类
Class instanceClass = [animal class];
        //元类（类的isa指向）
Class metaOfInstanceClass = object_getClass(instanceClass);
        //元类的类（元类的isa指向）
Class rootMetaCls = object_getClass(metaOfInstanceClass);
        //NSObject类
Class rootObjClass = [NSObject class];
        //NSObject元类
Class rootObjcMetaCls = object_getClass(rootObjClass);
```
控制台信息：
![](https://ws2.sinaimg.cn/large/006tKfTcgy1flrytt8tauj30d903twfk.jpg)

从上图可以看出Animal元类的isa指向了NSObject的元类，就像Animal的supperClass指向NSObject一样。这也正是类方法可继承的原因，我们在上面解释了对象方法的查找过程，同样的类方法也是根据isa指针查找的。由此我们可以得出以下结论：

* 每个Class都有一个isa指针指向一个唯一的Meta Class
* 每一个Meta Class的isa指针都指向最上层的Meta Class（图中的NSObject的Meta Class）
* 最上层的Meta Class的isa指针指向自己，形成一个回路

在上面的class的定义中，有一个super_class成员，它指向当前类的父类，那么MetaClass的super_class指向哪里呢？我新建一个子类看看：
```c
 // Cat继承自Animal类
        Cat* animal = [[Cat alloc]init];
        //类
        Class class = [animal class];
        //父类
        Class supperOfClass = [class superclass];
        //元类（类的isa指向）
        Class metaOfClass = object_getClass(class);
        //元类的元类（元类的isa指向）
        Class metaOfMetaCls = object_getClass(metaOfClass);
        //元类的父类
        Class supperOfMetaCls = [metaOfClass superclass];
        //NSObject类
        Class rootObjClass = [NSObject class];
        //NSObject的父类
        Class supperOfRootCls = [rootObjClass superclass];
        //NSObject元类
        Class metaOfRootCls = object_getClass(rootObjClass);
        //NSObject元类的父类
        Class superOfRootObjMetaCls = [metaOfRootCls superclass];
```
我们来看看各个变量的值：

![](https://ws1.sinaimg.cn/large/006tKfTcgy1flryza7kikj309904ogms.jpg)
 
可以看到：
1.元类的父类指向了，父类的元类；
2.NSObject的supperClass为nil
3.NSobject的元类的父类指向了NSObject；
 
结合继承关系和isa关系我们可以得出以下这张图：
![](https://ws3.sinaimg.cn/large/006tKfTcgy1flrz2rlxunj309z0ag0tv.jpg)

总结一下我们前面得出的结论：

* 每个Class都有一个isa指针指向一个唯一的Meta Class
* 每一个Meta Class的isa指针都指向最上层的Meta Class（图中的NSObject的Meta Class）
* 最上层的Meta Class的isa指针指向自己，形成一个回路
* 每一个Meta Class的super class指针指向它原本Class的 Super Class的Meta Class。但是最上层的Meta Class的 Super Class指向NSObject Class本身
* 最上层的NSObject Class的super class指向 nil
看到这里文章开头的四个问题你能分析出来吗？
