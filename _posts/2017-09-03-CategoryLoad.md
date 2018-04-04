---
layout: post
title: OC中分类Category的探究
date: 2017-09-03
categories: blog
tags: [Category,Objective-c]
description: OC中分类Category方法的加载以及对Load方法的处理
---
### Category初探
category是Objective-C 2.0之后添加的语言特性，Category可以为已存在的类添加方法，达到在不修改原有类的设计和实现的情况下添加新的功能，即使这个类的源码对你是不公开的。Cateory的使用场景包括但不限于：
- 把类的实现分开在几个不同的文件里面。这样做有几个显而易见的好处：
   
    a)可以减少单个文件的体积 

    b)可以把不同的功能组织到不同的category里 

    c)可以由多个开发者共同完成一个类 

    d)可以按需加载想要的category 等等。

- 声明私有方法。
- 公开framework中的私有方法。

category这种无侵入型或者侵入型极轻的特性被跟多第三方库大量使用。

### Extension是匿名的Category吗？
Extension的使用和Category很相似，很像匿名的Category，但实际上Extension和Category其实是两种概念，他们有本质的区别，现在网上很多文章在写Category的时候都将Extension作为Cateory的特例，这种说法其实是不对的。
Extension其实就是类的一部分，我们只能给一个已存在的，有源代码文件的类添加Extension，我们在Extension中添加的属性和方法，他们的实现在原始类的@implementation代码块中。我们通常会在.h文件中定义类的属性，这些属性是可以被外部直接访问到的，然而很多时候我们并不希望外部知道这些属性，我们可以利用类扩展来隐藏一些私有方法和属性。可以说Extension只是起到将一些方法和属性的声明隔离的作用，它本质上和你在.h文件定义的属性和方法没区别。

Category是和类一样是有自己的数据结构表示的，和Extension在编译时决议不同，Category的方法是在运行时编译期决议的，所以Category不支持添加属性，因此如果支持意味着需要在运行时更改对象的内存布局，每个属性的内存便宜可能都需要更新，开销比较大。（其实从苹果源码的注释中可以看到苹果本来打算让Category支持属性的）。

### Category的内存结构
前面提到Category有自己的数据结构表示，Class在OC中使用struct表示，Category也不例外。
```c
struct category_t {
    const char *name;
    classref_t cls;
    struct method_list_t *instanceMethods;
    struct method_list_t *classMethods;
    struct protocol_list_t *protocols;
    struct property_list_t *instanceProperties;
    // Fields below this point are not always present on disk.
    struct property_list_t *_classProperties;

    method_list_t *methodsForMeta(bool isMeta) {
        if (isMeta) return classMethods;
        else return instanceMethods;
    }

    property_list_t *propertiesForMeta(bool isMeta, struct header_info *hi);
};
```
一个Category结构实例保存一个分类的信息：分类名、原类名、分类中的实例方法列表、分类的类方法列表、遵守的协议列表。可以看到Category中没有变量的信息，所以这也是为什么Category不支持属性的原因。当然我们可以通过其他的方式达到在分类中添加属性的目的。这些结构体和结构体成员的值在编译期就有编译器自动创建并维护了。

method_list_t是一个方法列表的容器，其继承自：entsize_list_tt，这是一个泛型容器类，method_list_t容器内的元素类型是method_t,其定义如下：

```c
struct method_t {
    SEL name;
    const char *types;
    IMP imp;
};
```
还有一个结构体 category_list 用来存储所有的 category，它的定义如下:
```c
struct locstamped_category_t {
    category_t *cat;
    struct header_info *hi;
};

struct locstamped_category_list_t {
    uint32_t count;
#if __LP64__
    uint32_t reserved;
#endif
    locstamped_category_t list[0];
};
```
除了标记存储的 category 的数量外，locstamped_category_list_t 结构体还声明了一个长度为零的数组，这是C99中的一种写法，允许在运行期动态的申请内存。
上面列出数据结构基本就是分类所涉及到的，从这里也可以看出Category和Extension有很大区别。

### Category方法的加载
前面提到Category的方法是在运行时添加的，我们根据源码的调用过程来看一下Category方法的加载过程。runtime依赖于dyld动态加载，在 objc-os.mm 文件中可以找到入口，它的调用栈简单整理如下:
```c
void _objc_init(void)
└──const char *map_2_images(...)
    └──const char *map_images_nolock(...)
        └──void _read_images(header_info **hList, uint32_t hCount)
```
在_read_images的最后是处理分类方法的加载，在期方法体内有这样一段代码：
```c
    // Discover categories. 

    for (EACH_HEADER) {
        category_t **catlist = 
            _getObjc2CategoryList(hi, &count);
        bool hasClassProperties = hi->info()->hasCategoryClassProperties();

        for (i = 0; i < count; i++) {
            category_t *cat = catlist[i];
            Class cls = remapClass(cat->cls);
            ...

            if (cat->instanceMethods ||  cat->protocols  
                ||  cat->instanceProperties) 
            {
                addUnattachedCategoryForClass(cat, cls, hi);
                if (cls->isRealized()) {
                    remethodizeClass(cls);
                    classExists = YES;
                }
                ...
            }

            if (cat->classMethods  ||  cat->protocols  
                ||  (hasClassProperties && cat->_classProperties)) 
            {
                addUnattachedCategoryForClass(cat, cls->ISA(), hi);
                if (cls->ISA()->isRealized()) {
                    remethodizeClass(cls->ISA());
                }
            }
        }
    }
```
上面代码块的就是遍历类的分类，添加分类中的实例方法和类方法，从两次调用remethodizeClass方法的区别上可以看出，实例方法添加在实例的类上，类方法则是添加到元类上，这和类本身方法的存储行为是一致的。addUnattachedCategoryForClass作用是将类和当前的分类进行一次关联记录，真正添加方法的操作是有remethodizeClass进行的。remethodizeClass中会对类的分类再进行一次检查，然后调用attachCategories来进行添加。

```c
static void 
attachCategories(Class cls, category_list *cats, bool flush_caches)
{
    if (!cats) return;
    if (PrintReplacedMethods) printReplacements(cls, cats);

    bool isMeta = cls->isMetaClass();

    // fixme rearrange to remove these intermediate allocations

    method_list_t **mlists = (method_list_t **)
        malloc(cats->count * sizeof(*mlists));
    property_list_t **proplists = (property_list_t **)
        malloc(cats->count * sizeof(*proplists));
    protocol_list_t **protolists = (protocol_list_t **)
        malloc(cats->count * sizeof(*protolists));

    // Count backwards through cats to get newest categories first

    int mcount = 0;
    int propcount = 0;
    int protocount = 0;
    int i = cats->count;
    bool fromBundle = NO;
    while (i--) {
        auto& entry = cats->list[i];

        method_list_t *mlist = entry.cat->methodsForMeta(isMeta);
        if (mlist) {
            mlists[mcount++] = mlist;
            fromBundle |= entry.hi->isBundle();
        }

        property_list_t *proplist = 
            entry.cat->propertiesForMeta(isMeta, entry.hi);
        if (proplist) {
            proplists[propcount++] = proplist;
        }

        protocol_list_t *protolist = entry.cat->protocols;
        if (protolist) {
            protolists[protocount++] = protolist;
        }
    }

    auto rw = cls->data();

    prepareMethodLists(cls, mlists, mcount, NO, fromBundle);
    rw->methods.attachLists(mlists, mcount);
    free(mlists);
    if (flush_caches  &&  mcount > 0) flushCaches(cls);

    rw->properties.attachLists(proplists, propcount);
    free(proplists);

    rw->protocols.attachLists(protolists, protocount);
    free(protolists);
}
```
在while循环遍历所有的分类，然后获取每个分类的方法列表mlist，将其加载mlists中，mlists其实是一个二维数组了，它里面的每一个元素就是一个分类的方法列表，这里注意while循环是逆序的，实际上分类列表后面的分类的方法在mlists的前面。

最后再调用类对象中method成员变量的attchList方法将分类的方法列表和原类合并
```c
    void attachLists(List* const * addedLists, uint32_t addedCount) {
        if (addedCount == 0) return;

        if (hasArray()) {
            // many lists -> many lists

            uint32_t oldCount = array()->count;
            uint32_t newCount = oldCount + addedCount;
            setArray((array_t *)realloc(array(), array_t::byteSize(newCount)));
            array()->count = newCount;
            memmove(array()->lists + addedCount, array()->lists, 
                    oldCount * sizeof(array()->lists[0]));
            memcpy(array()->lists, addedLists, 
                   addedCount * sizeof(array()->lists[0]));
        }
        else if (!list  &&  addedCount == 1) {
            // 0 lists -> 1 list

            list = addedLists[0];
        } 
        else {
            // 1 list -> many lists

            List* oldList = list;
            uint32_t oldCount = oldList ? 1 : 0;
            uint32_t newCount = oldCount + addedCount;
            setArray((array_t *)malloc(array_t::byteSize(newCount)));
            array()->count = newCount;
            if (oldList) array()->lists[addedCount] = oldList;
            memcpy(array()->lists, addedLists, 
                   addedCount * sizeof(array()->lists[0]));
        }
    }
```
这段代码是先调用 realloc() 函数将原来的空间拓展，然后把原来的数组往后移，最后再把新数组放到前面。
这里的处理，苹果分了三种情况：

1. 原本的列表有多个元素，要添加的列表有多个元素，即多对多
1. 原本的列表有0个元素，要添加的列表有列表有多个元素，即0对多
1. 原本的列表有1个元素，要添加的列表有列表有多个元素，即1对多

之所以这样处理我想应该是减少对malloc的调用，特别是第二种情况，不需要原地扩容直接在原列表重新赋值即可。但是无论哪种情况分类的方法列表最终是放在原类方式列表前面的。所以我们平时所说的分类的方法如果和原来的类重名了，会“覆盖”原类的方法，这种说法是不成立的，因为实际上由于分类的方法在列表的前面，当我们调用对象的方法时，运行时库检查这个类和其超类的方法列表，找到一个匹配这条消息的方法，然后基于那个方法调用函数（IMP)，所以分类方法和原类方法同名时，先找到了分类的方法，执行了分类的方法，造成了原类被覆盖的假象。所以我们只要顺着方法列表找到最后一个对应名字的方法，也就可以调用原来类的方法了。

根据前面分类方法列表的添加顺序和调用方法时方法的查找顺序，可以分析得出：

1. 分类方法和原类同名时，调用对象的该方法将执行分类中的方法实现。
2. 有多个分类都有同名的方法是，在分类列表中的最后一个分类的方法将被执行。

我们写一个demo来分析下分类原类方法同名时方法的调用情况

```objc
@implementation MyClass
- (void) name{
    NSLog(@"call method in MyClass");
}
@end
@implementation MyClass (Category1)
- (void) name{
    NSLog(@"call method in MyClassCategory1");
}
@end
@implementation MyClass (Category2)
- (void) name{
    NSLog(@"call method in category2");
}
int main(int argc, char * argv[]) {
    @autoreleasepool {
        MyClass *obj = [[MyClass alloc]init];
        [obj name];
        return 1;
    }
}
```
首页新建一个MyClass类，里面实现了一个名为name的方法，然后新建了两个分类：Category1和Category2，两个分类也实现名为name的方法，然后在程序入口main函数中创建一个Myclass的实例，调用它的name方法。
在编译设置中文件的变异顺序如下：

1. MyClass
1. MyClass+Category1
1. MyClass+Catgory2

执行程序的打印结果：
```
2018-03-08 13:50:09.516736+0800 CategoryMethodsLoad[20902:6340460] call method in category2
```
其调用了MyClass+Catgory2中的方法，那么将顺序如果调整为：Category2、Category1、MyClass，会是什么结果呢？
```
2018-03-08 14:06:30.509841+0800 CategoryMethodsLoad[21185:6379393] call method in MyClassCategory1
```
上面的打印结果页印证了前述的两个结论。
那么load方法也是这样的吗？相当多的第三方库中在分类中实现了load方法，load方法的执行也会遵循上面两个结论吗？

### Load方法揭秘
我们在上面Demo中的类和分类中实现Load方法，看看Load的执行是何种情况，此时的编译顺序为：Category2、Category1、MyClass
```objc
@implementation MyClass
+ (void) load{
    NSLog(@"call load in MyClass");
}
@end
@implementation MyClass (Category1)
+ (void) load{
    NSLog(@"call load in MyClassCategory1");
}
@end
@implementation MyClass (Category2)
+ (void) load{
    NSLog(@"call load in MyClassCategory2");
}
@end
```
打印结果如下：
```
2018-03-08 14:15:10.280068+0800 CategoryMethodsLoad[21288:6395187] call load in MyClass
2018-03-08 14:15:10.280815+0800 CategoryMethodsLoad[21288:6395187] call load in MyClassCategory2
2018-03-08 14:15:10.281132+0800 CategoryMethodsLoad[21288:6395187] call load in MyClassCategory1
```
我们可以调整文件的变异顺序，看看不同情况下的打印结果，这里我就不做过多展示，直接给出结论：
* 分类和所有分类的load方法都会执行；
* 原类的load方法先于分类执行
* 分类load执行顺序取决于编译顺序，编译顺序在前，则load方法先执行。
另外我们可以给MyClass添加一个父类，父类中也实现load方法，再看看父类load方法的执行时机，测试之后的结论——父类的load方法会先于子类执行。综合前面的结论最终得到一下结果：
* 父类的load方法会先于子类执行
* 类和所有分类的load方法都会执行；
* 原类的load方法先于分类执行
* 分类load执行顺序取决于编译顺序，编译顺序在前，则load方法先执行。

load方法和普通的方法有如此大的普通，看来在分类加载时应该是苹果对load方法进行了特殊处理，我们结合源码来看看load到底是如何让处理的，还是在objc-os.mm文件中，在入口函数init方法体最后调用了_dyld_objc_notify_register来出发load_images的执行，在load_images内部中执行了call_load_methods()，从名称就可以堪称这就是load方法的触发点。
```c
void _objc_init(void)
{
    static bool initialized = false;
    if (initialized) return;
    initialized = true;
    environ_init();
    tls_init();
    static_init();
    lock_init();
    exception_init();
    _dyld_objc_notify_register(&map_images, load_images, unmap_image);
}
load_images(const char *path __unused, const struct mach_header *mh)
{
    if (!hasLoadMethods((const headerType *)mh)) return;

    recursive_mutex_locker_t lock(loadMethodLock);
    {
        rwlock_writer_t lock2(runtimeLock);
        prepare_load_methods((const headerType *)mh);
    }
    // Call +load methods (without runtimeLock - re-entrant)

    call_load_methods();
}
void call_load_methods(void)
{
    static bool loading = NO;
    bool more_categories;
    loadMethodLock.assertLocked();
    // Re-entrant calls do nothing; the outermost call will finish the job.

    if (loading) return;
    loading = YES;
    void *pool = objc_autoreleasePoolPush();
    do {
        // 1. Repeatedly call class +loads until there aren't any more

        while (loadable_classes_used > 0) {
            call_class_loads();
        }
        // 2. Call category +loads ONCE

        more_categories = call_category_loads();

        // 3. Run more +loads if there are classes OR more untried categories

    } while (loadable_classes_used > 0  ||  more_categories);
    objc_autoreleasePoolPop(pool);
    loading = NO;
}
```
call_load_methods中的先调用call_class_loads处理类的load方法，再调用call_category_loads处理分类的load方法，这印证了前面类的load先于分类的结论和打印结果。在call_category_loads中则会遍历分类列表每次变异都去执行当前分类的load方法，着也印证了分类load执行顺序取决于编译顺序的结论。

至此我们已经对Category的加载和其与原类方法同名是需系统的处理有了一定了解，顺便揭秘了分类load方法执行的机制，结合demo和源码我们可以看到很多细节上的东西，了解一些oc特性背后的故事。

