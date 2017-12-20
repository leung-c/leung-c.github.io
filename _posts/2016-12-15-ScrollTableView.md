---
layout: post
title: ScrollView内嵌tableView无缝滚动的实现
date: 2016-12-15
categories: blog
tags: [scrollview,tableview]
description: 
---


当UIScrollView内部嵌套了另一个ScrollView(TableView、CollectionView、WebView等)时，如何处理外部ScrollView和内部ScrollView的无缝平滑滚动，本文提供了一种简单的方式来处理这种情况。

## ScrollView内嵌tableView无缝滚动的实现

> 本文从以前简书博文迁移过来，更新了部分内容


### 功能业务场景

大家在开发过程中应该遇到过这样的页面交互设计：在一个纵向滚动的页面中，上面的是一块信息的v展示区域，下面是一个列表，用户在滚动页面时，先是整体页面滚动，当动到一个阈值临界点时，上面部分固定，下面的列表继续滚动。这里只是做一个大概的描述，实际的页面有更多复杂的细节和交互，在我们常用的APP中类似这种的典型页面如下图：
![](https://ws2.sinaimg.cn/large/006tKfTcgy1fmmzsujel3j30ku1127wh.jpg)
![](https://ws3.sinaimg.cn/large/006tKfTcgy1fmmzsj72ssj30ku112x1a.jpg)
> 饿了么商家主页

这里对视图结构的具体实现不做分析，着重于对上面描述的滚动交互实现的讨论，分析一下问题：

1. 外层的ScrollView存在一个阈值，当其ContentOffset未达到这个阈值时，内部ScrollView不滚动，外层ScrollView继续滚动，达到／超过阈值后外部ScrollView不可滚动，内部ScrollView可滚动。
1. 当内部的ScrollView的ContentOffset小于或等于0时，内部ScrollView不滚动，外部ScrollView可滚动。
1. 页面跟随手指滑动而滚动，当外部Scroll达到阈值，手指继续滑动，此时外部ScrollView不再滚动而是下面列表进行滚动，注意这里有一个手势响应对象的转移。

### scrollViewDidScroll

需要对滚动进行实时的处理，显然我们要用刀ScrollViewDidScroll这个代理方法了。假设：外部ScrollView滚到100后固定；外部ScrollView和内部ScrollView的代理对象为同一个。里面的代码大致如下：
```objc
- (void) scrollViewDidScroll:(UIScrollView*)scrollView{
    static CGFloat fixedContentOffsetY = 100.0;
    if(scrollView == _superScrollView){
        BOOL scrollDown = (scrollView.contentOffset.y - _superScrollLastOffsetY) > 0;
        if(scrollView.contentOffset.y >= fixedContentOffsetY && scrollDown){
            scrollView.contentOffset = CGPointMake(0,fixedContentOffsetY);
            scrollView.scrollEnabled  = NO;
            _childScrollView.scrollEnabled = YES;
        }else if (!scrollDown){
            scrollView.scrollEnabled  = YES;
        }
        _superScrollLastOffsetY = scrollView.contentOffset.y;
    }
    else{
        BOOL scrollDown = (scrollView.contentOffset.y - _childScrollLastOffsetY) > 0;
        if(_childScrollView.contentOffset.y <= 0 && !scrollDown){
            _childScrollView.contentOffset = CGPointMake(0,0);
            _childScrollView.scrollEnabled = NO;
            _superScrollView.scrollEnabled = YES;
        }else if (scrollDown){
            _childScrollView.scrollEnabled = YES;
        }
        
        _childScrollLastOffsetY = _childScrollView.contentOffset.y;
    }
}
```
以上经过真机滑动调试，可能需要做调整，例如：滑到临界点后往回滑动等；但整体的实现应该差不多。这种方法可以解决前面两个问题但是对于第三个问题没办法解决，每次滑动到临界点，继续滑动时superScrollView的确是不滚了，但是childScrollView也没有滚，这是因为每次对手势的识别，是superScrollView、childScrollView两个的其中一个，只有当手指离开屏幕，再次触摸开始滑动时，才有可能(取决于手势的位置)切换到另一个ScrollView识别新的手势。
问题的核心就在于：每次的滑动手势需要两个ScrollView都能响应！在这个前提下，我们再通过判断是否达到临界点条件来决定哪个ScrollView的手势响应不处理。
能否拿到UIScrollView的手势对象呢？如果能拿到，我们可以在手势对象的代理方法中来明确设定某个ScrollView的ContentOffset，这似乎是一个可行办法，UIScrollView上的确有个panGestureRecognizer属性，然而看一下这个属性的定义：
```objc
/ Use these accessors to configure the scroll view's built-in gesture recognizers.
// Do not change the gestures' delegates or override the getters for these properties.

// Change `panGestureRecognizer.allowedTouchTypes` to limit scrolling to a particular set of touch types.
@property(nonatomic, readonly) UIPanGestureRecognizer *panGestureRecognizer NS_AVAILABLE_IOS(5_0);

```
苹果的注释中强调：不要改变这个手势的delegate对象或者重写这个属性的get方法。苹果关了一扇门，却给我们留一扇窗。

### UIGestureRecognizerDelegate

UIGestureRecognizer(手势识别器)的delegate需要遵守UIGestureRecognizerDelegate协议，在这个协议中有这样的一个方法声明：
```objc
- (BOOL)gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRecognizeSimultaneouslyWithGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer
```
从上面Scroll的的手势识别器的定义中得知，Scroll中内置的手势识别器的delegate就是ScrollView本身。

我们看一下苹果官方对这个方法的说明：
> Asks the delegate if two gesture recognizers should be allowed to recognize gestures simultaneously.
This method is called when recognition of a gesture by either gestureRecognizer or otherGestureRecognizer would block the other gesture recognizer from recognizing its gesture. Note that returning YES is guaranteed to allow simultaneous recognition; returning NO, on the other hand, is not guaranteed to prevent simultaneous recognition because the other gesture recognizer's delegate may return YES.
简单翻译一下：
询问代理对象是否允许两个手势识别器同时识别手势。这个方法在gestureRecognizer(识别器1)或者otherGestureRecognizer(识别器2)进行识别会被调用。如果返回YES表示可以进行同时识别；返回NO则保证不阻止同时识别，因为其它的识别器代理有可能返回YES。

因此，我们需要帮UIScrollView实现这个代理方法，并返回YES，使得外部的ScrollView和内部的Scrollview都能响应在其视图内的手势，未了不必新建一个UIScrollView的子类来实现这个方法，我们通过分类的方式来达到目的。并不是所有的UIScrollView都需要支持多识别器同时识别，所以我们在分类中新增一个属性：shouldRecognizeSimultaneouslyGesture，然后在gestureRecognizer:shouldRecognizeSimultaneouslyWithGestureRecognizer方法中直接返回这个属性。
```objc
/**
 是否需要处理同时发生的手势，内嵌scrollview如需无缝滚动，请设置为YES；
 */
@property (nonatomic,assign) BOOL shouldRecognizeSimultaneouslyGesture;

- (BOOL) gestureRecognizer:(UIGestureRecognizer *)gestureRecognizer shouldRecognizeSimultaneouslyWithGestureRecognizer:(UIGestureRecognizer *)otherGestureRecognizer{
    return self.shouldRecognizeSimultaneouslyGesture;
}
```
我们把外部的ScrollView和内部的ScrollView的shouldRecognizeSimultaneouslyGesture均设置为YES就可以了。现在我们手指滑动时内外的ScrollView都会滚动，接下来我们只需要控制好达到临界点时的内外ScrollView的滚动就行了，依然再scrollViewDidScroll中实现这个控制：
```objc
- (void) scrollViewDidScroll:(UIScrollView*)scrollView{
    static CGFloat fixedContentOffsetY = 100.0;
    if(scrollView == _superScrollView){
        if (_superScrollView.contentOffset.y >= fixedContentOffsetY) {
            _superScrollView.contentOffset = CGPointMake(0, fixedContentOffsetY);
            _fixOnTop = YES;//是否要固定到顶了
        }
        if(_fixOnTop){
            _superScrollView.contentOffset = CGPointMake(0, fixedContentOffsetY);
        }
    }
    else{
        if (_childScrollView.contentOffset.y <= 0) {
            _childScrollView.contentOffset = CGPointMake(_childScrollView.contentOffset.x, 0);
            _fixOnTop = NO;
        }
        if(!_fixOnTop){
            _childScrollView.contentOffset = CGPointMake(_childScrollView.contentOffset.x, 0);
        }

    }
}
```
比之前的处理要简单不少，基本上如果需要支持这种无缝滚动的地方，我们设置好属性后在UIScrollView的代理didScrollView方法中写好控制就可以了。更进一步，didScroll中大部分的代码都是想相同，只是每个地方的设置的固定高度不同，所以我们可以将这些控制处理封装起来，外面调用时传入要控制的高度即可。
```objc
/**
 @param height 外部ScrollView滑动后需固定的高度
*/
- (void) superScrollViewDidScrollWithFixedHeight:(CGFloat)height;

/**
 @param superScrollView 外部ScrollView
 */
- (void) childScrollViewDidScrollInSuperScrollView:(UIScrollView*)superScrollView;
```
```objc
- (void) superScrollViewDidScrollWithFixedHeight:(CGFloat)height{
    if (self.contentOffset.y >= height) {
        self.contentOffset = CGPointMake(0, height);
        self.fixedOnTop = YES;
    }
    if(self.fixedOnTop){
        self.contentOffset = CGPointMake(0, height);
    }
}
- (void) childScrollViewDidScrollInSuperScrollView:(UIScrollView*)superScrollView{
    if (self.contentOffset.y <= 0) {
        self.contentOffset = CGPointMake(self.contentOffset.x, 0);
        superScrollView.fixedOnTop = NO;
    }
    if(!superScrollView.fixedOnTop){
        self.contentOffset = CGPointMake(self.contentOffset.x, 0);
    }
}
```

### Demo

基于这个方案，我将这些处理封装到一个UIScrollView的分类里，并写了一个Demo来演示如何使用。具体代码可见github地址：<https://github.com/leung-c/ScrollTableView>












