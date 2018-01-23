---
layout: post
title: 利用分类让你的view强大起来
date: 2016-12-23
categories: blog
tags: [objective-c,category]
description: 让View既能保持纯洁性，又能处理相关业务逻辑的方式。
---

> “保证UI与逻辑分离，不要在view中写页面跳转，不要在view中写网络请求、不要在view中处理业务计算进行业务验证。”

这应该是很多开发的一条必须遵循的准则了。但是开发过程中经常存在这样苦恼：
- 我们想让view能够承担更多的功能，但是又不能牵扯到业务逻辑。于是我们经常需要将view触发上的操作通过代理模式或者Block回调到控制器中，由控制器去完成工作，如果view的层级很深就有可能形成“Callback hell”，一级级的向上回调造成回调深渊。
- 如果不同的模块(控制器)，有部分相同功能的view和操作就得在不同的页面中实现一样的逻辑。同样功能的代码散落在工程的各个角落。

比如加入购物车控件：
![](https://ws2.sinaimg.cn/large/006tNc79gy1fni819kn18j30ga05m41x.jpg)
> 初始状态

![](https://ws4.sinaimg.cn/large/006tNc79gy1fni81bw11bj30gg05owi2.jpg)
> 点击加号按钮

它的职责：

- 点击将商品加入购物车一次，界面变成可加减的模式，并显示该商品在购物车的总数量
- 点击加或者减增加或减少商品在购物车的数量
- 输入数量后将商品在购物车的数量修改为输入值

这个加车控件在许多界面可能都会用到：商品搜索结果页、分类商品页、店铺首页等等如果通过回调或者代理将上面的操作交给外部操作，意味着好几个页面都会有重复的逻辑和代码。

> don't repeat yourself

如果代码有重复，我们应该想办法封装这段代码以复用。我们的目的是不破坏View单纯的UI职责，不给它添加业务逻辑，再此基础上需要给它放权，让它能实现自己应该做的功能。

一种做法：我们可以把这个过程三个操作抽取出来，然后在View的回调或代理中调用对应的操作，将子逻辑独立，然后由VC来组织调度这写子逻辑。这种方法基本能满足我们大部分的要求了，虽然真对上面加车的例子各个VC中还是有重复的代码逻辑,但是比起每个VC都去实现一边相同的子操作逻辑还是省了不少代码。那么有没有一种方式让View自己做好自己的功能，同时不破坏其纯洁性呢？

OC的Category能让我们为原始类添加新的功能，这个特性很符合我们的需求。例如日我们可以写一个很单纯的NumberEditView，并将一些控件作为只读属性暴露出来。
```objc
typedef void(^ButtonDidPressed)(void);
typedef void(^InputCompelete)(NSString *text);
@interface NumberEditView : UIView
//数量加回调

@property (copy,nonatomic) ButtonDidPressed addAction;
//数量减回调

@property (copy,nonatomic) ButtonDidPressed reduceAction;
// 输入完成回调

@property (copy,nonatomic) InputCompelete inptutCompelete;

@property (strong,nonatomic,readonly) UIButton *addButton;
@property (strong,nonatomic,readonly) UIButton *reduceButton;
@property (strong,nonatomic,readonly) UITextField *inputText;
@end
```
这个view本身只包含很简单纯展示和交互功能，现在我们可以给它加一个分类来处理添加，减少和输入的逻辑。

```objc
@implementation NumberEditView(addToShopCart)
//为添加购物车功能做好准备工作

- (void) prepareForCartOperation{
    __weak typeof(self) weakView = self;
    [self setAddAction:^{
        [weakView add];
    }];
    [self setReduceAction:^{
        [weakView reduce];
    }];
    [self setInptutCompelete:^(NSString *text) {
        [weakView updateToNumber:text];
    }];
}
- (void) add{
    //调用添加接口
    
    __weak typeof(self) weakView = self;
    [net request:^(NSError *error,id response){
        if(!error){
            ...
            //显示在购车的最新数据
            
            weakView.inputText.text = response;
            ..
        }
    }];
}
- (void) reduce{
    //调用减少接口
    
    __weak typeof(self) weakView = self;
    [net request:^(NSError *error,id response){
        if(!error){
            ...
            //显示在购车的最新数据
            
            weakView.inputText.text = response;
            ...
        }
    }];
}
- (void) updateToNumber:(NSString*) numString{
    //调用更新接口
    
    __weak typeof(self) weakView = self;
    [net request:^(NSError *error,id response){
        if(!error){
            ...
            //显示在购车的最新数据
            
            weakView.inputText.text = response;
            ...
        }
    }];
}
@end
```
在使用这个view并且需要添加购物车功能的地方引入这个分类，创建好view之后调用prepareForCartOperation就可以为这个view加载上购物车相关功能了。如果其他地方想使用这个view单纯进行数量编辑，并不是加入购物车，直接使用主类就好了。

这种给view添加分类的方式去封装业务逻辑，即保证了view的UI职责的纯粹性，又能让view得到更多权利去实现和自身相关的功能，而且这个功能可插拔的。像插件一样让原始类更强大又不影响原始类本身的功能。


