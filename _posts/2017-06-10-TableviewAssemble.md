---
layout: post
title: 一种灵活简单的tablview组装模式
date: 2017-06-10
categories: blog
tags: [objective-c,tableview，MVVM]
description: MVVM模式下tableview数据和cell的组装
---

UITableView可以说是ios开发过程中用到频率最高的控件了，随便打开一个的APP，随便进入一个页面百分之八九十就是利用tableView实现的，下图就是京东和天猫最重要的商品详情页面和某APP简单的个人信息页。

![](https://ws1.sinaimg.cn/large/006tNc79gy1flxs1xt2drj30b40jr1kx.jpg)![](https://ws1.sinaimg.cn/large/006tNc79gy1flxs1xfm3nj30b40jrgrt.jpg)![]
![](https://ws2.sinaimg.cn/large/006tNc79gy1flxs36skv1j30b40jrqk4.jpg)

复杂多变的页面体现了tableview极高的灵活性，甚至天猫、京东的首页都是可以用tableview去实现的。当然他们的首页应该是基于ScrollView，仿tableview复用机制，自己实现了一个组件，具体实现这里不作叙述，后面有机会再探讨。
回想一下TableView的使用，核心就是下面几个代理方法：
* cell的行数

```objc
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section 
```
* 指定位置cell的高度

```objc
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath
```
* 返回指定位置要展示的cell

```objc
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
```
* 选中指定位置的cell

```objc
- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath
```
上面的方法决定了cell展现效果、交互操作。相信大家对tablview代理方法的执行顺序，复用机制应该是是烂熟于心了。回到上面天猫或京东的商品详情页面，使用tableview的话我们需要做哪些事呢？
在回答这个问题前我们先分析一下问题：
1. cell的个数固定吗？
1. 有几种类型的cell？
1. 同类型的cell点击后的交互操作不一样如何处理

显然cell个数不固定：如果商品不支持白条，不会有白条cell；如果商品可以领优惠券，需要显示优惠券cell；如果商品不参加促销活动，促狭互动cell也不显示... 具体有多少个cell是根据数据确定的。

有几种cell这里比较直观，名称、价格、白条/已选规格/促销、地址、评价 5种cell，当然地址和白条cell可以作为一种，但是这样会造成单个cell内部元素控制复杂化。

cell点击的其实应该是最后需要处理的，如果解决了cell的呈现，如何区分cell点击进行对应业务的处理也就迎刃而解了。

最简单粗暴的方案是依据jindexPath确定对应位置的cell，但这方案毫无可读性、维护性可言。我们可以模拟一下，如果是这个方案那上面四个代理方法将会是什么样子
```objc
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    if(section == 0){
        int rowCount = 2;//名称＋价格
        //支持白条
        if(canBlanknote){
            rowCount ++;
        }
        //有促销
        if(hasPromotion){
            rowCount ++;
        }
        //有优惠券
        if(hasCounpon){
            rowCount ++;
        }
    }elseif (section == 1){
        ......
    }
    .......
}
```
```objc
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    if(indexPath.section == 0){
        if(indexPath.row == 0){
            return 100;
        }else if(indexPath == 1){
            return 50;
        }else{
            ...
        }
        ....
    }
    ....
}
```
模拟代码写到这里我就不想也不必写下去了，我只是简单的模拟了一下，真个过程是非常枯燥繁琐的，每添加一行if else都需要对前面的判断分支进行逻辑梳理，看看indexpath数的是否正确，，而且整个过程需要重复四遍，因为你需要实现四个代理方法。我相信很多人在从业经历中或多或少看到这养糟糕的代码，可能一个简单的页面最终实现后的代码行数奔1000＋了。最痛苦的是好不容易显示正确功能完成，出现bug后调试过程绝对让人崩溃，更别说日后产品要求调整相关信息的显示顺序了，这又会让人脱一层皮。

那么有没有一种比较轻松明了、可维护、可扩展的的方式呢？你应该也有自己的方案，这里说一下我平时采用的方式，并且将这种方式可以扩展到CollectionView，最终我们初步形成了一套简单的框架，让开发人员只需要关注业务数据和业务处理，整个个cell装载的过程是透明的。在这个方式下，开发只需要做两件事：根据业务维护数据、写cell。下面我们看一下具体是什么样的方式。

页面的呈现是依赖于数据的，也就是说有A数据那就有需要显示A的cell，没有A数据也就无需A cell了，我们先不去关注这个A到底需要在第几个section的第几个位置，假如我们先把cell写好了，后端同学的接口也提供了，现在我们要把数据通过cell呈现到界面上，仔细想一下我们还缺什么呢？

我们还缺少数据和界面的描述。以商品标题为例：“Apple／苹果 iPhone8...” 这个是数据，这个数据展现时要黑色18号字体、50的行高等等，现在我们需要建立对数据要如何呈现的描述，创建一个类完成这个工作：
```objc
@interface InfoCellViewModel : NSObject
//标题
@property (copy , nonatomic) NSString *title;
//子标题
@property (copy , nonatomic) NSString *description;
//字体 颜色
@property (strong,nonatomic) UIFont *titleFont;
@property (strong,nonatomic) UIColor *titleColor;
@end
```
从类名可以看到这里实际是使用MVVM模式。对商品标题来说我们创建一个InfoCellViewModel实例维护好对应属性就行了，可是使用什么cell去显示？显示多高？显然我们加一个属性表示要用的cell就行了：
```objc
//cell的复用id
@property (copy , nonatomic) NSString *cellReuseId;
@property (assign , nonatomic) CGFloat cellHeight;
```
现在数据显示所需要的信息基本都具备了，对于要显示商品标题我们可以这样描述它了：
```objc
InfoCellViewModel *title = [InfoCellViewModel new];
title.title = @"iphone8";
title.description = @"不要钱";
title.cellReuseId = @"TitleInfoTableViewCell";
title.cellHeight = 100;//这里也可以换成动态计算的高度
```
按照这种方式，其它的数据我们也可以描述出来了，当然可能有的数据InfoCellViewModel是描述不了的，比如：购买数量（可编辑的），我们再创建一个可以描述这些数据的类就行了，创建ViewMode类时一定要注意：它的作用是用来描述数据显示的，不要牵扯具体的业务数据和业务逻辑。另外cell的高度、cell复用Id这些是基础信息是必有的，所以我们可以建一个基类BaseCellViewModel来定义这些信息，其它类继承即可。
```objc
//商品标题cell
InfoCellViewModel *title = [InfoCellViewModel new];
    ....
title.cellReuseId = @"TitleInfoTableViewCell";

//商品价格cell
InfoCellViewModel *price = [InfoCellViewModel new];
    ....
price.cellReuseId = @"PriceInfoTableViewCell";

//数量输入cell
InputTextCellViewModel *price = [InputTextCellViewModel new];
price.placeHolder = @"请在这里输入您要购买的数量";
//只能输入数字
price.textKeyboardtype = UIKeyboardTypeNum;
    ....
price.cellReuseId = @"TextInputTableViewCell";
```
到这里我们已经完成了对需要显示数据的描述，现在我们要做的就是将这个描述数据和tableView的代理方法联系起来，并通过tableView的代理方法帮我们决定什么位置显示什么cell。那么，我们需要做的就是将这些描述对象维护到数组就行了，
前面我们说过：有数据A就显示A的cell，我们维护描述对象时就可以做到这一点了：
```objc
NSMutableArray  *allSections = [NSMutableArray array];

NSMutableArray *section1Cells = [NSMutableArray array];
//商品标题cell
InfoCellViewModel *title = [InfoCellViewModel new];
    ...
[sectionCells addObject:title];
//商品价格cell
InfoCellViewModel *price = [InfoCellViewModel new];
    ...
[sectionCells addObject:price];
if(hasCounpon){
    //优惠信息cell
    InfoCellViewModel *coupon = [InfoCellViewModel new];
        ...
    [sectionCells addObject:coupon];
}
if(hasPromotion){
    //促销信息cell
    InfoCellViewModel *promotioin = [InfoCellViewModel new];
        ...
    [sectionCells addObject:promotioin];
}
//第二个section的数据
NSMutableArray *section2Cells = [NSMutableArray array];
//数量输入cell
InputTextCellViewModel *price = [InputTextCellViewModel new];
    ....
[section2Cells addObject:promotioin];

allSections = @[section1Cells,section2Cells,.....];
```
上面的代码就是整个tableview要现实的数据的描述了，对于tableView的代理方法里面的实现就非常简单了，不需要任何硬编码：
```objc
- (NSInteger)tableView:(UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    NSMutableArray ＊cells = self.allSections[section]
    return cells.count
}
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath {
    BaseCellViewModel *cellVM = self.allSections[indexPath.section][indexPath.row];
    return cellVm.cellHeight;
}
- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    BaseCellViewModel *cellVM = self.allSections[indexPath.section][indexPath.row];
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:cellVM.reuseId];
    if (!cell) {
        Class cellClass = cellVM.viewClass?: NSClassFromString(cellVM.reuseId);
        cell = [[cellClass alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:cellVM.reuseId];
    }
    return cell;
}
```
那么cell的点击我们怎么去处理呢？如同处理使用什么cell来现实的问题一样，我们在描述对象上创建block回调来解决：
```objc
typedef id(^CallBackBlock)();
@property (copy, nonatomic) CallBackBlock callBack;
```
于是，当某些数据需要处理点击时，我们在维护它的描述对象时写好回调就行了：
```objc
InfoCellViewModel *coupon = [InfoCellViewModel new];
        ...
[counpon setCallBack:id(^){
    //弹出优惠券信息选择框
}];

- (void)tableView:(UITableView *)tableView didSelectRowAtIndexPath:(NSIndexPath *)indexPath {
    [tableView deselectRowAtIndexPath:indexPath animated:YES];
    BaseCellViewModel *cellVM = self.allSection[indexPath.section][indexPath.row];
    if (cellVM.callBack) {
        cellVM.callBack();
    }
}
```

至此，我们基本上了解上述方案的整个实现过程，其主要思路是将数据和界面元素通过一个个描述对象进行关联，描述对象相当于一条纽带，上面携带数据和其要现实的信息，cell的操作通过这条纽带作用于数据，触发业务逻辑。数组的组装是在VC中进行的，因为当前页面要显示成什么样VC是最清楚的，也应该由它来决定。这方式也达到了前面所说的开发人员只需要组装好数据、写好cell就可以，cell的装载不需要介入。

采用这种方式后续的维护和扩展都极其方便，无非就是新增cell、有可能需要新增ViewModel类、VC中组装对应ViewModel对象就行了，对于修改和移除也只需对应项的ViewModel，而且非常适合多人开发合作，有人写cell有人写ViewModel有人写VC，当然这本身就是MVVM的一个有点。

回到前面最后的那张图，采用这种方式很容易就可以实现这个功能，以后如果要添加新的信息，只需要在数据源中添加就行了。但是，仔细想一下是这里是否和上面商品详情的实现有重复的代码呢？整个Cell的构造和点击触发其实是一样的；另外，如果需要显示SectionHeaderView或者SectionFooterView呢？CollectionView的数据组装是不是也可以这么做？基于这些问题我门对这种模式进行了抽象和扩展，形成一套简单的类库，基于这套模式，新的页面几乎不需要处理TableView／CollectionView的代理方法，而专注于页面数据的组装和业务逻辑。github地址:https://github.com/leung-c/LCListComponent.git

