---
title: Mixins of Dart
date: 2023-09-06 10:59:33
tags:
---

# Dart里的Mixin

一开始以为只是简单的代码混入，感觉还蛮好用。

但是多个Mixin如果有相同的函数或变量混入了如何解决？

> Mixins in Dart work by creating a new class that layers the implementation of
> the mixin on top of a superclass to create a new class - it is not "on the side " but "on top" of the superclass, so there is no ambiguity in how to resolve lookups.
>
> \- Lasse R. H. Nielsen on StackOverflow.

以下的代码，实际上继承链为Disposable->A->B->AB。

```dart
abstract class Disposable {
  void dispose() {
     print('Disposable');
  }
}

mixin A on Disposable {
  @override
  void dispose() {
    super.dispose();
    print('A');
  }
}

mixin B on Disposable {
  @override
  void dispose() {
    super.dispose();
    print('B');
  }
}

final class AB extends Disposable with A,B {
  @override
  void dispose() {
    super.dispose();
    print('AB');
  }
}


void main() {
  AB().dispose();
}
```

