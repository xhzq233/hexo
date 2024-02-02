---
title: 实现ModalBottomSheet
date: 2023-08-07 10:02:55
tags:
---

# 实现`ModalBottomSheet`

定义`ModalBottomSheet`是**可滑动关闭**的**底部弹出**Route，并且可以定义上一Route的Transition动画，这个动画是跟手的。

iOS13中新出现了一种弹出Route，是堆叠形式的**底部弹出**Route，暂且将其命名为`StackModalPopup`。

所以`StackModalPopup`是`ModalBottomSheet`的子集，只是自定义了动画。

<!--more-->

## 如何在route中获取上一个和下一个route

根据Route中的`TickerFuture didPush()`函数的文档：

> The [didChangeNext] and [didChangePrevious] methods are typically called immediately after this method is called.

## 关于`ModalBottomSheet`与其他Route联动

在`ModalBottomSheet`中如果再push其他路由（包括自己）

```dart
Other 					 route2 	<- top
ModalBottomSheet route1
```

此时route1在secondaryAnimation下需要不同的transition，比如在push `ModalBottomSheet`应该是继续堆叠stack动画，其他的时候应该是向左滑出。

在transition里写判断我觉得不可取，而且如果push进来的不是`ModalBottomSheet`，此时向下滑动应该能直接退出该`ModalBottomSheet`里的所有路由才符合直觉，如下。

```dart
Other 					 route3 	<- top
Other 					 route2
ModalBottomSheet route1   // 理想情况：向下滑动route1将退出route1,2,3
```

所以应该再嵌套一个Navigator，如下。

`route2_1(route1_1)`代表的意思是此时它同属两种route，因为两个Navigator有各自的routes。

此时`navigator1.push(route1_2)`执行的是route1_1的transition，`navigator2.push(route2_1)`执行的是route2_1的transition，这样两个就互不影响了。

```dart
ModalBottomSheet 	 route2_2   <- top of navigator2
	Other 					 route1_3 	<- top of inside navigator1
	Other 					 route1_2
ModalBottomSheet   route2_1(route1_1)
```

但是直接嵌套会出现问题，存在于有实体（虚拟）返回键的设备上。目前有两个Navigator，而返回键默认触发的是WidgetsApp中的`Navigator.maybePop()`。

如果此时位于route1_3，因为backbutton触发的是`navigator2.pop()`，这将导致route1_*都被推出，显然这是不符合预期的。解决方案就在于如何hack来改变返回键的默认触发。

### 如果使用Router

需要在特定地点加上`Router(backButtonDispatcher: ChildBackButtonDispatcher())`。

### 使用Navigator

Navigator是如何与平台的back button绑定上的？答案在`WidgetsBindingObserver.didPopRoute()`，根据其文档

> Observers are notified in **registration order** until one returns true. If none return true, the application quits.

所以只需要抢先在`WidgetsApp`前注册就行。

简化的代码：

```dart
_navigatorKey ??= GlobalKey<NavigatorState>();
fakeBackButtonDispatcher.onWillPop = () {
  return _navigatorKey?.currentState?.maybePop() ?? Future.value(false);
};
child = Navigator(
  key: _navigatorKey,
  onGenerateRoute: (_) => SomePageRoute(
    builder: (context) => widget.child,
  ),
);
```
