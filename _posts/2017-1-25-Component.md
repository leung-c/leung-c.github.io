---
layout: post
title: 基于tableview的页面组件化方案
date: 2017-01-25
categories: blog
tags: [objective-c,首页,天猫,京东,CMS,组件]
description: 
---

 什么是页面动态化？
> 本文所说的页面动态化是指：在不修改APP代码的情况，页面的展示形态和数据根据服务端的配置是可变化的。
本文说的组件是指：APP内写好的功能完备的，且支持一定程度的定制化的自定义View。

对于大多数APP来说，对页面的动态化需求并不高，它们各个页面包括最重要的首页形式上基本不会有很大的改变。如果需要一定的改动，修改代码发新版本，现在的审核速度基本能够满足需求。但是，对于综合电商类APP来说的，某些页面尤其是推广页，根据运营或时效性（各种节日、各种活动）的需要，要求能快速的改头换面营造气氛。

#### 背景
我们APP的首版上线时，首页的形态是根据UI和PM的设计固定好的。随着业务的发展，对于运营的灵活性要求越来越高，这种灵活性首先就体现APP端：能支持运营人员通过配置控制内容的展示形态，而且控制粒度在“一个独立功能”组件。比如：一个轮播块，只要我给定所需的图片资源，他就可以工作，不依赖于其他的功能组件。

#### 组件化
运营对于页面动态控制实际是通过对页面配置组件实现的，所以问题的核心就是：
1. APP端需要提供一套组件库，每种组件对外有一个组件标识，运营将所需的组件配置到页面上，并配置好数据源。
2. 进入APP页面时，获取该页面的配置，加载对应组件，填充数据。

第一个问题比较好解决，我们根据UI和PM的设计可以确定好有哪几种组件，每种对应封装好一个View即可，当然这里实现时要考虑有很多细节。难点就在与第二个问题：组件以何种方式加载？一种类型的组件可以有多个，如何支持复用？组件的位置布局如何确定？

##### UIScrollView
最初的方案是使用UIScrollView来承载组件，将组件作为子View一个个添加到UIScrollView中，但是考虑之后放弃了，因为需要自己维护ContentSize，并且需要实现类似TableView的复用机制，这两项都可以做到但是比较耗费成本，可能实现高效复用机制的时间要比实现所有组件还多，当然使用UIScrollView来做可以完全自己掌控组件的加载，就看我们怎么取舍。那么，我们能否利用TableView现有的复用机制呢？

##### 没有Cell的UITableView
使用TableView的话ContentSize不需要我们维护了，我们要在heightForRow等代理方法中返回该位置组件的高度即可，它会自动计算好。那么，tableView的复用机制我们怎么利用呢？TableView的dequeueReusableCellWithIdentifier根据复用ID返回的是一个Cell，是否用Cell实现组件或者用Cell再把组件封装一层？因为使用TableView的大多数情况下我们是在和Cell打交道，所以会形成思维定势。其实SectionHeaderView也是可以支持复用的，我们完全可以把组件作为TableView的SectionHeaderView，后台配置了多少个组件就有多少个Section，只是这个Section中没有Cell。
#### UITableViewHeaderFooterView
这里有一个小限制，复用的SectionHeaderView必须是UITableViewHeaderFooterView类型，意味着我们组件的基类必须是UITableViewHeaderFooterView，为了避免每个组件直接继承自UITableViewHeaderFooterView，我们新建了一个ComponentBaseView，作为组件的基类，并且定义了所有组件通用的属性和方法（方法具体由各个子类自己实现），起到一个抽象类的作用。其大致形式如下：
```objc
@interface ComponentBaseView
/**
 设置组件数据数据
 */
- (void) setData:(nullable id)data;
/**
 根据复用标识id实例化组件
 */
- (nonnull instancetype)initWithReuseIdentifier:(nullable NSString *)reuseIdentifier;

/**
 根据数据返回该组件的高度
 */
+ (CGFloat) heightForData:(nullable id)data;

/**
 组件的复用ID
 */
+ (nonnull NSString*) reuseId;

@end
```
通过基类我们可以对组件统一的行为进行管理，比如每种组件都支持背景图、背景色、阴影等等。组件的高度是多少组件自己是最清楚的，不管固定的高度或者动态计算的高度我们希望都有组件自己决定，而heightForRowAtIndexPath这个代理方法调用非常频繁，不可能创建好组件填充好数据后再返回组件对象的高度，所以我们将heightForData定义成类方法。

每种组件的实现就很简单了，继承自ComponentBaseView，写好组件的功能即可，接下来就剩组件的加载过程了。

#### ComponentContainerScrollView
理论上我们的组件可以用与普通的TableView，但是我们的组件加载有其特殊性：我们需要根据组件类型去实例化并加载组件。为了方便管理组件的加载过程，我们创建了一个ComponentContainerScrollView类来作为专门承载组件的容器，它继承自UITableView，并且其代理和数据源代理是其本身。
```objc
@interface ComponentContainerScrollView : UITableView
//组件数据源
@property(strong,nonatomic) NSArray<ComponentViewModel*>* components;
@end

@interface ComponentViewModel : NSObject

//组件的唯一标示，复用标识
@property(copy,nonatomic) NSString* identifier;

//用于组件展示的数据
@property(strong,nonatomic) id data;
@end

@implementation ComponentContainerScrollView

- (instancetype) initWithFrame:(CGRect)frame{
    self = [super initWithFrame:frame style:(UITableViewStyleGrouped)];
    self.delegate = self;
    self.dataSource = self;
    //默认是没有cell的
    self.rowHeight = 0;
    self.separatorStyle = UITableViewCellSeparatorStyleNone;
    return self;
}
@end
```
我们实现TableView的代理和数据源方法
```objc
//组件的个数
- (NSInteger) numberOfSectionsInTableView:(UITableView *)tableView{
    return self.components.count;
}

//组件高度
- (CGFloat) tableView:(UITableView *)tableView heightForHeaderInSection:(NSInteger)section{
    ComponentViewModel *model = [self.components safeObjectAtIndex:section];
    Class class = [[ComponentBuilder sharedInstance] componentTypeForIdentifier:model.identifier];
    return [class heightForData:model];    
}

//创建组件
- (UIView*)tableView:(UITableView *)tableView viewForHeaderInSection:(NSInteger)section{
    ComponentViewModel *model = [self.components safeObjectAtIndex:section];
    ComponentContainerScrollView *view = (ComponentContainerScrollView*)[tableView dequeueReusableHeaderFooterViewWithIdentifier:model.identifier];
    if(view==nil){
        view = [[ComponentBuilder sharedInstance] componentWithIdentifier:model.identifier];
    }
    [view setData:model];
    return view;
}
```
这里面我们用到了一个重要但是实际很简单的一个类：ComponentBuilder，根据类名可以看出，它负责组件的创建，外部给定一个组件标识它创建并返回一个组件，同时也提供一个方法根据标识返回组件的类。

ComponentBuilder的内部实际维护了一个组件标识符和组件类的一个映射，期内部的实现大致如下：
```objc
@implementation ComponentBuilder
{
    NSMutableDictionary* _components;
}

- (ComponentBaseView*) componentWithIdentifier:(NSString*)componentId{
    Class viewType = _components[componentId];
    
    if(viewType){
        return  [[viewType alloc] initWithReuseIdentifier:componentId];
    }
    return nil;
}

- (void) registerComponentType:(Class)componentClass forIdentifier:(NSString*)componentId{
    if(!_components){
        _components = [NSMutableDictionary new];
    }
    _components[componentId] = componentClass;
}

- (Class) componentTypeForIdentifier:(NSString*) componnentId{
    return _components[componnentId];
}
```
registerComponentType用于注册一个组件类型到ComponentBuilder，这个方法由组件调用，在每个组件load方法中调用这个方法，每个类的load方法由系统在程序启动时执行，这样可以保证我们程序启动后所有的组件都是已注册状态。

接下来，我们只需要创建一个VC来作为动态化页面的入口，完成网络请求并设置组件容的数据源，这个基于TableView的页面组件化加载差不多就完成了，当然了，这还只是一个非常简单仅仅可用的状态。
```objc
@interface ComponentViewController : UIViewController
/**
 页面ID
 */
@property(copy,nonatomic) NSString* pageId;
@end
- (void)viewDidLoad{
    self.componentView = [[ComponentContainerScrollView alloc]initWithFrame:CGRectMake(0, 0, kScreenWidth, kScreenHeight)];
    [self.view addSubview:self.componentView];
    [self.componentView mas_makeConstraints:^(MASConstraintMaker *make) {
        make.left.right.top.bottom.offset(0);
    }];
    [self loadData];
}

- (void)loadData{
    WEAKSELF;
    //
    [_manger requestForPage:self.pageId components:^(NSError *error,NSArray *array){
        weakSelf.components = array;
        weakSelf.componentView.components = weakSelf.components;
    }];
}
```

##### 支持Cell的组件容器
前面我们提到了，我们把组件作为tableView的sectionHeaderView来加载，所以实际是不需要用到cell的。但是，如果我们的组件形式是类似于下面这样的呢：
![](https://ws2.sinaimg.cn/large/006tNc79gy1fnp4vxowf8j30ku0qytgo.jpg)

按照之前的设计，这是一个组件作为一个HeaderView，意味着上面的“附近店铺”和下面的店铺列表是一个View里面的，它的复用级别是这个View整体来复用的。我门创建这个HeaderView时根据数据创建对应数量的“店铺信息”子view，这个HeaderView复用时还需要根据此时的数据来“多退少补”-多的view要隐藏，少的view要补充创建。如果同样的组件有多个，而且数据个数不一样时，快速的来回滚动，会有瞬间的一丝卡顿，虽然不明显的影响用户体验。

理想中的做法是店铺信息能用Cell来展示，这样复用的话不仅粒度较小节省内存和解决滑动的问题，而起维护非常简单，不存在“多退少补”的问题。

那么，组件能否用cell呢？回到我们的组件容器ComponentContainerScrollView中，它实现了TableView的代理方法，但是实际每个位置的view是什么，高度是什么等等这些全部是交给组件类决定的，具体的实现是通过调用的组件的类方法。所以这个组件需不需要cell，用什么cell，cell有多高是不是也可以由组件自身决定呢？根据SectionHeader的加载过程，我们可以用同样的方式实现组件容器Cell的加载。

ComponentBaseView修改如下:
```objc
/**
 设置组件数据数据
 */
- (void) setData:(nullable id)data;
/**
 根据复用标识id实例化组件
 */
- (nonnull instancetype)initWithReuseIdentifier:(nullable NSString *)reuseIdentifier;

/**
 根据数据返回该组件的高度
 */
+ (CGFloat) heightForData:(nullable id)data;

/**
 如果组件的数据需要cell的显示，需要实现以下方法
 */

//cell的个数
+ (NSInteger) cellRowsAtIndexPath:(nullable NSIndexPath*)path forData:(nullable ComponentViewModel*)model;

//cell的高度
+ (CGFloat) cellHeightAtIndexPath:(nullable NSIndexPath*)path forData:(nullable ComponentViewModel*)model;

//返回一个cell
+ (nullable UITableViewCell*)cellForTableView:(nullable UITableView*)table atIndexPath:(nullable NSIndexPath *)path withData:(nullable ComponentViewModel *)model;

//点击了一个cell
+ (void) tableView:(nullable UITableView*)table didSelectRowAtIndexPath:(nullable NSIndexPath *)path withData:(nullable oComponentViewModel *)model;

/**
 组件的复用ID
 */
+ (nonnull NSString*) reuseId;

@end

```

ComponentViewController修改如下：
```objc
@implementation ComponentContainerScrollView
- (NSInteger) tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section
{
    ComponentViewModel *model = [self.components safeObjectAtIndex:section];
    Class class = [[CMSComponentBuilder sharedInstance] componentTypeForIdentifier:model.identifier];
    return  [class cellRowsAtIndexPath:[NSIndexPath indexPathForRow:0 inSection:section] forData:model];
}
//cell高度
- (CGFloat) tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath
{
    ComponentViewModel *model = [self.components safeObjectAtIndex:indexPath.section];
    Class class = [[CMSComponentBuilder sharedInstance] componentTypeForIdentifier:model.identifier];
    return  [class cellHeightAtIndexPath:indexPath forData:model];
}
//创建cell
- (UITableViewCell*)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    ComponentViewModel *model = [self.components safeObjectAtIndex:indexPath.section];
    Class class = [[CMSComponentBuilder sharedInstance] componentTypeForIdentifier:model.identifier];
    if (!class) {
        return  [[UITableViewCell alloc]initWithStyle:(UITableViewCellStyleDefault) reuseIdentifier:EmptyString];
    }
    return  [class cellForTableView:tableView atIndexPath:indexPath withData:model];
}
//组件的加载
- (NSInteger) numberOfSectionsInTableView:(UITableView *)tableView
{
    return self.components.count;
}
//组件高度
- (CGFloat) tableView:(UITableView *)tableView heightForHeaderInSection:(NSInteger)section{
    ComponentViewModel *model = [self.components safeObjectAtIndex:section];
    Class class = [[ComponentBuilder sharedInstance] componentTypeForIdentifier:model.identifier];
    return [class heightForData:model];    
}
//组件创建
- (UIView*)tableView:(UITableView *)tableView viewForHeaderInSection:(NSInteger)section
{
    ComponentViewModel *model = [self.components safeObjectAtIndex:section];
    ComponentContainerScrollView *view = (ComponentContainerScrollView*)[tableView dequeueReusableHeaderFooterViewWithIdentifier:model.identifier];
    if(view==nil){
        view = [[ComponentBuilder sharedInstance] componentWithIdentifier:model.identifier];
    }
    [view setData:model];
    return view;
}
-(void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath
{
    ComponentViewModel *model = [self.components safeObjectAtIndex:indexPath.section];
    Class class = [[CMSComponentBuilder sharedInstance] componentTypeForIdentifier:model.identifier];
    
    [class cellForTableView:tableView didSelectRowAtIndexPath:indexPath withData:model];
}
@end
```

对于某一个需要使用cell的组件来说只需要实现基类定义的cell相关方法就可以了，还是以上面的“附件的店铺”为例：
```objc
@interface NearbyStoreView:ComponentBaseView
@end
@implementation NearbyStoreView
/**
 这里只需要设置头部的数据
 */
- (void) setData:(nullable id)data{
    slef.titleLabel = @"—————— 附近的店铺 ————————";
}
/**
 根据复用标识id实例化组件
 */
- (nonnull instancetype)initWithReuseIdentifier:(nullable NSString *)reuseIdentifier{
    self = [supper initWithReuseIdentifier:[self className]];
    if(self){
        UILabel *label = [[UILabel alloc]init];
        [self addSubview:label];
    }
    return self;
}

/**
 这里只需要返回头部的高度即可，cell的高度由cell相关方法返回。
 */
+ (CGFloat) heightForData:(nullable id)data{
    return 50;
}

//cell的个数
+ (NSInteger) cellRowsAtIndexPath:(nullable NSIndexPath*)path forData:(nullable ComponentViewModel*)model{
    return [model.data[@"stores"] count];
}

//cell的高度
+ (CGFloat) cellHeightAtIndexPath:(nullable NSIndexPath*)path forData:(nullable ComponentViewModel*)model{
    reurn 150;
}

//返回一个cell
+ (nullable UITableViewCell*)cellForTableView:(nullable UITableView*)table atIndexPath:(nullable NSIndexPath *)path withData:(nullable ComponentViewModel *)model{
    StroreInfoCell *cell =  [StroreInfoCell cellForTableView:table];
    NSArray *stores = model.data[@"stores"];
    cell.store = stores[indexPath.row];
    return cell;
}

//点击了一个cell
+ (void) tableView:(nullable UITableView*)table didSelectRowAtIndexPath:(nullable NSIndexPath *)path withData:(nullable oComponentViewModel *)model{
    StoreVC *vc = [[StoreVC alloc] init];
    [self.VC.naviController pushViewController:vc animated:YES];
}

/**
 组件的复用ID
 */
+ (nonnull NSString*) reuseId{
    return [self className];
}
@end
```
现在，组件的形式可以很灵活了，它也可以全部只有cell，真个装载过程我们将组件的形式、行为完全交给它自身管理，外部在相关处理点调用对应的方法即可，相关类的交互过程如下：
```sequence
ComponentBaseView->ComponentBuilder: +Regist
Note left of ComponentBaseView: SubClassImplement
Note right of ComponentContainer: TableView Delegate&DataSource \nMethods

ComponentBuilder->ComponentContainer:GetComponent
ComponentBaseView-->ComponentContainer: +numberOfRows
ComponentBaseView-->ComponentContainer: +heightForHeaderView\n+heightForCell
ComponentBaseView-->ComponentContainer: +ViewtForSection\n+CellForRow
```

