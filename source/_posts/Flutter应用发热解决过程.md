---
title: Flutter应用发热解决过程
date: 2023-09-26 13:54:38
tags:
---

# 某Flutter应用发热解决过程

某版本改动过后，发热量徒增（尤其是iOS的弱散热平台），遂开始溯源。

先CPU Profiler开profile看实时cpu消耗，注意到键盘升起时报Low Memory Warning，同理键盘收取也是。并且CPU Usage飙升，并且只要键盘存在CPU Usage就维持在较高水平。

在活跃约1分钟后Thermal State变为Fair，一分半变为Serious。发热确实很严重。

初步判断flutter Textfield在iOS平台有问题，去flutter issue寻找，果真找到了[#128197](https://github.com/flutter/flutter/issues/128197)。

于是将flutter Textfield替换成iOS原生UITextView的实现，发热解决了。

## 但是CPU高占用和Low Memory Warning还是未解决。

Low Memory Warning可能是大量新创建的对象Object，即可能是大量的Widget被重新创建导致的build。

rebuild范围大主要原因就是监听了不必要的dependency，比如MediaQuery.of会监听所有MediaQueryData，context.watch会监听所有来自ChangeNotifier的变化。所以应该改成如MediaQuery.paddingOf和context.select的形式，并且以抽Widget代替function的方式，可以有效减小rebuild范围。

