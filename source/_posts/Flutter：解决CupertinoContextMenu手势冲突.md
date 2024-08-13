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

### 理解CupertinoContextMenu部分原理

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

## 冲突原因

如果我们在“CupertinoContextMenu”的子树上放置一个“TapGestureRecognizer”。

为了简单起见，我们将“CupertinoContextMenu”中的“TapGestureRecognizer”命名为**tg2**，将“CupertinoContextMenu”子树中的“TapGestureRecognizer”命名为**tg1**，按压持续时间命名为**t1**。

当 `kPressTimeout+_previewLongPressTimeout/2` (500ms) < **t1** < `kPressTimeout+_previewLongPressTimeout `(900ms)时，此时

1.虽然**tg2**被拒绝了，但`_ContextMenuRoute`仍然被打开。

2.**tg1**的onTap方法被触发

问题是，此ContextMenu中的GestureRecognizer是TapGestureRecognizer，而不是LongPressGestureRecognizer，Tap不声称获胜，但GestureArena选择最深的Tap来获胜。

1.当**t1**大于500ms时，这意味着它已通过“PrimaryPointerGestureRecognizer.deadline”加上“openController”持续时间的一半，意味着“_ContextMenuRoute”即将打开。在“GestureArena”收到指针活动后，**tg1**与**tg2**竞争。默认的“子胜利”起作用了，并调用了tg1的onTap，因此两者都被触发了。

2.当**t1**小于900ms时，因为路线此时已打开，并且两者都将被竞技场拒绝。

## 解决方法

原因是“TapGestureRecognizer”不会自行宣布胜利。添加以下代码。

```dart
// call this when animation's value first reaches [_midpoint]

_tapGestureRecognizer.resolve(GestureDisposition.accepted);
```

## Related issues

https://github.com/flutter/flutter/issues/70716

https://github.com/flutter/flutter/issues/52226

https://github.com/flutter/flutter/issues/81057

## Related PR

https://github.com/flutter/flutter/pull/131030

## 2024/8/12 Updated

以上只声明了手势胜利，如果没有主动声明失败，也会导致手势冲突。

### 理想情况和现状

比如一个 tg1 套在 contextMenu(tg2) 外层。理想情况是，点击后只有 tg1 响应（contextMenu 不应该点一下就打开），然而现状是点击时只会响应 tg2 。

因为 tg2 在子树，且他没有主动声明失败，所以默认 tg1失败了。

### 解决方法

为了主动声明失败，我们需要列一下失败场景：

1. 非常迅速的点击，未超过 100ms。此时两个 tg 的 ontapdown 方法还未触发，onPointerUp 便已经触发，竞技场准备好决胜了。此时 tg2 准备获胜，tg2 的 ontapdown、ontapup 被依次调用，decoyChild的动画也被短暂执行。
2. 超过 100ms 但未超过 midpoint。此时 tg1 和 tg2 都作为待选者，二者 ontapdown 都被触发。在竞技场决胜时（onPointerUp），最终 tg2 还是获胜，tg 的 ontapup 被调用。
3. 超过 midpoint 的**持续按压**。此时 tg2 主动声明胜利。

可见场景 1 和 2 是需要调整的。调整为：

1. 在竞技场决胜时，tg2 直接声明失败，tg2 ontapdown、ontapup、ontapcancel 均不会调用，tg1 胜利。
2. tg2 声明失败，tg2 ontapdown虽已调用，但不会调用 up ，会调用 ontapcancel 清理decoyChild动画逻辑，tg1 胜利。

综上所述，解决方法应该只有一个，Listener中的onPointerUp事件中添加以下代码：

```dart
Listener(
onPointerUp: (PointerUpEvent event) {
  if (_openController.value < _midpoint) {
    _tapGestureRecognizer.resolve(GestureDisposition.rejected);
  }
});
```

曾想过用 GestureArenaManager.hold 方式延缓决胜，这样也许可以让场景 1 和 2 统一，但非常迅速的点击出现decoyChild的动画不合理，遂放弃。