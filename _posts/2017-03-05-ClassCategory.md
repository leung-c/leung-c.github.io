---
layout: post
title: 由MVC框架下常见的代码问题想到的
date: 2017-03-05
categories: blog
tags: [objective-c,MVC,MVVM]
description: MVC和MVVM到底该如何使用？
---

在我们团队的开发工作中有这样的约定：尽可能保证view的纯展示、交互职能，以达到view最大限度的复用。可能你会问：难道不是本来就该这么做么？确实，理论应该是这样的。苹果CocoaTouch框架本身的设计就是很好的实现典范。然而我见过的项目中，有不少代码完全违背苹果MVC模式规范的。

![](https://ws4.sinaimg.cn/large/006tNc79gy1fnigwlo6tvj30xe0iaqmi.jpg)
上图是苹果官方的示意图，其中很重要却被有意或无意忽视的一点：View和Model彼此是不能通信相互依赖的。

这样的代码你应该也看过不少：
```objcv
UserInfoCell *cell = [UserInfoCell cellForTableview:tableview atIndex:indexPath];
[cell setUser:user];
```
model被view直接引用并持有，在view的内部将model的属性和view上的控件对应起来。这种情况非常多以至于成了一种“模式”了，多数人认为，大家都这么写那应该没什么问题，甚至认为这是封装性的一种体现（之前在面试时问到这种代码的问题时，不少人这样回答我）。在我个人看来宁可不要这种所谓的“封装”。这种写法可能还有一种目的：避免把MVC变成“Massive View Controller”，毕竟按照规范的写法上面的代码可能是这个样子：
```objc
User *user  = users[indexPath.row];
UserInfoCell *cell = [UserInfoCell cellForTableview:tableview atIndex:indexPath];
cell.imageView.image = user.icon;
cell.titleLabel.text = user.name;
cell.subTitle.text = user.desc;
cell.contentLabel.text  = user.detail;
...
```
这确实是令人头疼的问题，viewcontroller中包含各种这个样的代码：获取数据、视图组织、视图元素赋值、监听model属性的更改....但是我们有很多中解决方法，却偏偏需选择了最简单直接粗暴的方法：将model丢给view。



### MVVM

事实上MVVM就是是不错的选择，它引入了viewmodel层用于UI层的数据处理。在使用MVVM的时候通常会配合使用数据绑定，它使得model数据变更时viewmodel会接收到通知，viewmodel变更是view会收到通知，发过来也是一样（双向绑定），ios中可以通过Notification或KVO或ReactiveCocoa来实现，当然数据绑定并不是使用MVVM的必要条件，即使我们使用的是MVC我们同样可以借鉴MVVM的优点，形成 MVVM Without DataBinding。

MVVM（Model View ViewModel）从名字就可以看出淡化了Controller的作用，在Controller我们需要做的是：构造View、组织ViewModel数据、将ViewModel传递给View。在View中我们就可以像之前使用Model一样去使用ViewModel。ViewModel不应该是对Model的又一层封装，对ViewModel的设计应该考虑需求方的使用，这个需求方就是View。针对上面的代码它的MVVM Without DataBinding版本可能是这样的：
```objc
@implementation UserCellViewModel
 +(id) userCellViewModelWithUser:(User*)user{
     UserCellViewModel *vm = [UserCellViewModel alloc]init];
     vm.icon = user.icon;
     vm.title = user.name;
     vm.subTitle  = user.desc;
     vm.content = user.detail;
     return vm;
 }
 + (void) fetchUsers:(void ^(NSArray<UserCellViewModel*> *users,NSError *error))complete{
     NSMutableArray* users = [NSMutableArray array];
     //获取model数据，构造viewmodel
     ....
     [users addObject:[UserCellViewModel userCellViewModelWithUser:user]];
     complete(users,error);
 }
@end

@implementation UserListViewController
[UserCellViewModel fetchUsers:^(NSArray<UserCellViewModel*> *users,NSError *error){
    if(!error){
        self.users = users
        ...
    }
}];
...

UserCellViewModel *user  = users[indexPath.row];
UserInfoCell *cell = [UserInfoCell cellForTableview:tableview atIndex:indexPath];
cell.userViewModel = user;
@end
```
上面的示例中我讲网络请求放到了ViewModel中，返回给外面的数据就是组织好的ViewModel对象，你可以针对你们自己工程的特点来抽取这个过程，这里说一下我们的做法以供借鉴：我们对于接口的数据请求有单独的一个层DataSerivice去做，比如User相关的就会有一个UserDataService，它会封装好对User数据的所有网络请求，返回的是Model数据，这样我们的接口和Model也是单独可复用的。ViewModel会调用UserDataService拿到数据，并生成viewmodel对象，Controller中拿到处理好的ViewModel传递给view即可。

这里你可能会有疑问：UserInfoCell依赖现在改成了ViewModel，这和之前依赖Model本质上没有区别啊？反而多引入了一层数据。解答这个问题就需要回到我们上面提到的一句话：
>“ViewModel不应该是对Model的又一层封装，对ViewModel的设计应该考虑需求方的使用，这个需求方就是View。”

不知道你注意到没，我在定义UserCellViewModel的属性时命名是这样的：icon、title、subTitle、content，事实上和user没有任何关系了，反而是view有了某种“巧合”：这个view上要显示头像、一个主题文字、一个简述、一段详细内容。所以我们的ViewModel类名叫UserCellViewModel根本就不合适，同样的我们的View难道只能显示人员的信息吗？一本书、一张专辑等等都可以使用这个view类显示一个简要信息，虽然UserInfoCell并不限定要展示的数据类型，但是因为你的名字是“UserInfoCell”于是自我画地为牢了。修改一下类名:
```
UserInfoCell—>BaseInfoCell
UserCellViewModel—>BaseInfoViewModel
```

```objc
@implementation BaseInfoViewModel
 +(id) baseInfoViewModelWithUser:(User*)user{
     BaseInfoViewModel *vm = [BaseInfoViewModel alloc]init];
     vm.icon = user.icon;
     vm.title = user.name;
     vm.subTitle  = user.desc;
     vm.content = user.detail;
     return vm;
 }
 + (void) fetchUsers:(void ^(NSArray<BaseInfoViewModel*> *users,NSError *error))complete{
     NSMuatbleArray* users = [NSMutableArray array];
     //获取model数据，构造viewmodel
     ....
     [users addObject:[BaseInfoViewModel userCellViewModelWithUser:user]];
     complete(users,error);
 }
@end

@implementation UserListViewController
[BaseInfoViewModel fetchUsers:^(NSArray<BaseInfoViewModel*> *users,NSError *error){
    if(!error){
        self.users = users
        ...
    }
}];
...

BaseInfoViewModel *user  = users[indexPath.row];
BaseInfoCell *cell = [BaseInfoCell cellForTableview:tableview atIndex:indexPath];
cell.userViewModel = user;
@end
```
当然上面的代码还有一个问题，BaseInfoViewModel中获取了User数据，后续如果这个BaseInfoCell需要显示书籍信息，是否在BaseInfoViewModel中写一个获取Book数据的方法？避免BaseInfoViewModel越来越冗余可以单独抽取一个类来完成获取Model构造ViewModel的过程。是否需要这么做就要你自己根据实际需要判断了，毕竟在一个APP中各种类型的数据一般会有属自己的特定展示形态。


