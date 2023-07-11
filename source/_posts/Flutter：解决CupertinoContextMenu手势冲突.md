---
title: Flutter：解决CupertinoContextMenu手势冲突
date: 2023-07-09 12:13:12
tags:
  - Flutter
  - Cupertino
  - 手势冲突
---

# Flutter：解决CupertinoContextMenu手势冲突

<!--more-->

## 前置知识

### 两个TapGestureRecognizer的冲突

两个Tap gesture Recognizer，一个套在另一个上层

1. 时间过短的tap，arena刚开启，kPressTimeout还未完成就将关闭，两个tap不会自己承认成功。arena默认选择子child的tap作winner，顺序

   onPointerDown
   inside onTapDown
   inside onTapUp
   inside onTap

2. 一旦超过100ms(kPressTimeout)，两个tap的onTapDown都会触发

3. 接下来无论持续按多久

    1. 只要平移距离在preAcceptSlopTolerance（flutter里这个值限定死了是kTouchSlop，令人感叹）内

    2. arena里没有其他手势宣布胜利（在本情形里没有其他主动胜利的gesture）

   那么两个tap都准备成功，最后选出子child的tap作winner，reject 外层的tap，最终触发顺序：

    1. onPointerDown

    2. inside onTapDown and onTapDown

    3. inside onTapUp

    4. inside onTap（与onTapUp一般来说是同时触发，但这里标明顺序是因为flutter里这么写的=w=）and onTapCancel（一个手势win同时会触发另一个的lose）

### 理解Flutter CupertinoContextMenu部分原理

```dart
// The duration of the transition used when a modal popup is shown. Eyeballed
// from a physical device running iOS 13.1.2.
const Duration _kModalPopupTransitionDuration = Duration(milliseconds: 335);

// The duration it takes for the CupertinoContextMenu to open.
// This value was eyeballed from the XCode simulator running iOS 16.0.
const Duration _previewLongPressTimeout = Duration(milliseconds: 800);

// The total length of the combined animations until the menu is fully open.
final int _animationDuration = _previewLongPressTimeout.inMilliseconds + _kModalPopupTransitionDuration.inMilliseconds;// 1,135

/// The point at which the CupertinoContextMenu begins to animate into the open position.
final double animationOpensAt = _previewLongPressTimeout.inMilliseconds / _animationDuration;// 0.704845815

final double _midpoint = animationOpensAt / 2;//0.3524229075
```


打开_ContextMenuRoute之前的三个阶段：

1. onTapDown：叠加诱饵子(Decoy child)，并开始[_openController]动画。

   > triggerred after a period of time after Listener sent onPointerDown. Typically, the time elapsed is the [kPressTimeout] in flutter.

2. holding: 只要手指一直按住，openController就会保持动画

3. 一旦指针抬起屏幕，就会调用onTapUp（如果对方获胜，则调用onTapCancel）

   1.如果动画进度大于中点：继续完成。动画完成后，_ContextMenuRoute将被推进，诱饵子将被删除。
   2.其他：反向执行动画直到完毕。一旦完毕，就删除诱饵子。

总之，ContextMenu的成功触发并不取决于此点击手势的获胜。

## 什么冲突了？

如果置一个TapGestureRecognizer（为了简便我们将其命名为**tg1**）在ContextMenu（ContextMenu里面的TapGestureRecognizer我们命名为**tg2**）子树上，当Tap持续的时间（命名为**t1**）在

`kPressTimeout+_previewLongPressTimeout/2` (500ms)

到

`kPressTimeout+_previewLongPressTimeout `(900ms)

之间

此时必定会触发1. _ContextMenuRoute被打开 2. **tg1**的onTap方法，这意味着什么？

如果这是一个展示图片的widget，长按打开ContextMenu，而点击打开图片详情页。这时，两者都触发的结果是灾难性的。

## 为什么冲突？

问题就出在与这个ContextMenu里面的GestureRecognizer是Tap而不是LongPress，Tap并不会宣称自己胜利，而是由Arena选出「子Tap」胜利。

当**t1**大于kPressTimeout+\_previewLongPressTimeout/2，代表其经过了PrimaryPointerGestureRecognizer.deadline和\_openController的一半，意味着_ContextMenuRoute即将被打开。而**tg1**此时还未结束识别，Arena识别到pointerUp后，便将其与**tg2**竞争，默认的「子胜利」起了作用，tg1的onTap被调用，于是两者都触发了。

而**t1**小于kPressTimeout+\_previewLongPressTimeout是因为此时Route被打开，两者都会被Arena拒绝。

## 如何解决？

1. 如上所述，换成LongPressGestureRecognizer，LongPress胜出会直接取消tg1的胜出。

2. 更简单的方法，根本原因是TapGestureRecognizer不会自己宣布胜利，下面一行代码即可解决

   ```dart
   // call this when animation's value first reaches [_midpoint]
   _tapGestureRecognizer.resolve(GestureDisposition.accepted);
   ```

## 后记

真尼玛逆天啊flutter，在写如上issue测试的时候发现：

在flutter test里模拟手势点击是有问题的，这么基础怎么没人发现啊。

于是又提了个issue给flutter。。。https://github.com/flutter/flutter/issues/129984

