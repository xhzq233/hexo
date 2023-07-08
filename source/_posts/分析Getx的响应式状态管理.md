---
title: 分析Getx的响应式状态管理
date: 2023-03-20 10:48:42
tags:
- Flutter
- Getx
- Reactive programing
---

# 分析Getx的响应式状态管理

> 你说的对，但是Getx是Flutter中的一个状态管理库，它可以帮助开发者更容易地管理Flutter应用程序中的状态，控制应用程序的生命周期和路由导航。同时，它还提供了很多实用工具，如国际化、依赖注入、路由管理、文件管理等功能。相较于其他状态管理库，Getx 开发起来更加简单，代码量更小，性能更高，在Flutter社区中也受到了广泛的关注和好评。

<!--more-->

一开始使用Getx看到如下的代码，这种针对性的响应式刷新，还以为是什么高级的黑魔法，就没有去了解它。

```dart
var name = 'Jonatas Borges'.obs;//使用.obs声明为State变量

Obx(() => Text("${controller.name.value}"));//name改变时会导致该Text rebuild，其他State变量改变则不会
```

今天尝试去阅读了源码，发现其实很简单。

先看**Obx**，继承自**ObxWidget**，一个StatefulWidget，有个build方法需要继承，Obx和ObxValue就是实现了build方法。

State\<ObxWidget>中一个**Observer**一个**Subscription**，分别用来监听State变量（即RxType变量）的变化和对应的储存下来的订阅（AnyCancelable），Observer监听到变化后会出发StatefulWidget里的setState标记Element为dirty，在下一frame中rebuild。

```dart
class _ObxState extends State<ObxWidget> {
  final _observer = RxNotifier();
  late StreamSubscription subs;
  
  void initState() {
    super.initState();
    subs = _observer.listen(_updateTree, cancelOnError: false);
  }
  
  void _updateTree(_) {
    setState(() {});
  }

  void dispose() {
    subs.cancel();
    _observer.close();
    super.dispose();
  }

  Widget build(BuildContext context) => RxInterface.notifyChildren(_observer, widget.build);
}
```

所以Observer如何监听到内部使用到的State变量的变化？

注意到**RxInterface.notifyChildren**，如下是处理过的源码，注意到**canUpdate**表示内部有没有使用到State变量、RxInterface.proxy被替换并且build完后被还原了回来，说明跟这俩肯定脱不开干系。

```dart
static Widget notifyChildren(RxNotifier observer, builder) {
  final _observer = RxInterface.proxy;
  RxInterface.proxy = observer;//替换
  final result = builder();//构造Widget
  if (!observer.canUpdate) {
    RxInterface.proxy = _observer;
    throw """
    [Get] th...
    """;//表示内部没有使用到State变量
  }
  RxInterface.proxy = _observer;//还原
  return result;
}
```

再看到**RxType**的value和canUpdate的**get**方法，RxType在get过程中会使用下面的**addListener**方法，把自己的通知/发布者（**Subject**）添加进**RxInterface.proxy**，也就是让RxInterface.proxy接受自己的通知（比如状态改变了），canUpdate就是判断这个订阅存不存在。

```dart
//Rx<T>内部
T get value {
  RxInterface.proxy?.addListener(subject);
  return _value;
}

//Observer内部
bool get canUpdate => _subscriptions.isNotEmpty;

void addListener(GetStream<T> rxGetx) {
  if (!_subscriptions.containsKey(rxGetx)) {//去重，因为value会get多次
    final subs = rxGetx.listen((data) {//监听、获取订阅
      if (!subject.isClosed) subject.add(data);
    });
    final listSubscriptions =
      _subscriptions[rxGetx] ??= <StreamSubscription>[];
    listSubscriptions.add(subs);//添加订阅
  }
}
```

那么已经很明确了，Obx的build过程分为一下几步

1. Obx.Observer替换RxInterface.proxy

2. 调用child.build()方法，方法中如果有使用到Rx变量的get方法，就会将Rx发布者注册进RxInterface.proxy

   > 这便是为什么Obx的构造方法要用函数当参数

3. 还原RxInterface.proxy

这样子当Rx改变时就会发布通知到对应Obx.Observer引起rebuild。
