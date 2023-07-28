---
title: 不要靠近Flutter的CupertinoContextMenu
date: 2023-07-28 18:01:51
tags:
---

# 不要靠近Flutter的CupertinoContextMenu

## 已存在的issue

跟原生完全没得比，child大小适配没有做https://github.com/flutter/flutter/issues/58880

child大小总是占屏幕的一半

actions弹出位置总是固定，一旦action多了就会超出屏幕，没有办法点击，也没有缩小或者滑动的策略让用户点击https://github.com/flutter/flutter/issues/55025

## 我使用过程发现的issue

### #1

flutter在弹出ctxMenu的策略是先放置overlay，在推入路由。

一旦这个overlay里面有跟其他组件交互的方法（在Widgets树中向上查找其他组件），而WidgetApp中Overlay处于较上层的位置，所以他所能找到的组件是有限的。

举个例子，Overlay在WidgetApp中是位于Navigator上层的，也就是你用Overlay下的context不可能调用Navigator.push等方法，因为在Overlay下context下找不到Navigator，此时会抛出

`Navigator operation requested with a context that does not include a Navigator.
The context used to push or pop routes from the Navigator must be that of a widget that is a descendant of a Navigator widget.`

所以如果想制作某个依赖上层dependency的overlay，最好做法是传context进去。

### #2 CupertinoContextMenu中可触发多手势

如果在长按触发CupertinoContextMenu过程中点击其他可响应事件的地方，这时候两个手势会在分别的GestureArena中不互相竞争，导致两个都可能触发。

比如在长按一张照片1时点击另一张照片2，这时候不仅会出现照片1的ContextMenuRoute，还会出现照片2的的照片详情页。但是在iOS中原生里，点击其他地方会取消打开这个照片1的ContextMenuRoute，而不是去响应这个点击手势。

### #3 [CupertinoContextMenu] Potential unremoved overlay entry

### Details
According to  https://github.com/flutter/flutter/blob/0ff68b8c610d54dd88585d0f97531208988f80b3/packages/flutter/lib/src/cupertino/context_menu.dart#L560C1-L583C1

`_lastOverlayEntry` is only removed when animation is dismissed or completed. 

What if  the overlayEntry is inserted, and then the `CupertinoContextMenu` is disposed and removed from widgets tree? So the entry will never be removed.

Temporary solution is add `_lastOverlayEntry?.remove()` to

https://github.com/flutter/flutter/blob/0ff68b8c610d54dd88585d0f97531208988f80b3/packages/flutter/lib/src/cupertino/context_menu.dart#L683C1-L687C4

But this is not enough, because there will be no animation, and it will be abrupt for it to suddenly disappear.

### Steps to reproduce

1. Add a `CupertinoContextMenu` to any visible widget.
2. Press it to insert the overlayEntry (the `_DecoyChild`) , while remove the `CupertinoContextMenu` from the widgets tree.
3. The entry never dismissed.

issue在https://github.com/flutter/flutter/issues/131471
