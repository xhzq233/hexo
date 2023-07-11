---
title: Dart里坑人的extension
date: 2023-07-11 11:05:54
tags:
---

# Dart里坑人的extension

先随便去哪里run下面的代码（图简便直接跑[DartPad](https://dartpad.cn/)）

```dart
void main() {
  final x = _111();
  print('methods inside class');
  for (int i = 0; i < 5; i++) {
    print(x.helloworld.hashCode);
  }
  print('methods inside extension of class');
  for (int i = 0; i < 5; i++) {
    print(x.helloworldOnExtension.hashCode);
  }
}

class _111 {
  void helloworld() {}
}

extension on _111 {
  void helloworldOnExtension() {}
}
```

看输出你发现了什么？

1. methods inside class的地址（hashcode）是不会变的
2. methods inside extension of class的地址是会变的，这说明dart里的extension中方法的实现机制可能是：在引用时创建一个匿名函数，临时分配地址

## 为什么坑人

如果想用extension做代码拆分的任务（虽然也有可能是这个初衷就存在问题，本来就不应该用extension做代码拆分），往extension里写入了一些你本来认为是「静态函数」的函数，认为引用的地址不会变。

比如

```dart
somePublisher.addListener(helloworldOnExtension)

// then
somePublisher.removeListener(helloworldOnExtension) // doesn't work
```

此时removeListener不会起作用，因为此时的helloworldOnExtension又是一个新的地址。

## 结论

少用extension（

## 关于dart里面的方法调度

如果将最开始的代码第二行改为

```dart
  final dynamic x = _111();
```

运行时会抛出error：

```dart
Uncaught TypeError: x.get$helloworldOnExtension is not a functionError: TypeError: x.get$helloworldOnExtension is not a function
```

这也侧面印证了dart里extension函数是动态生成的，并不存在于函数表里。

来到dart类里面的函数、变量查找机制。

即使是`dynamic`类型只要调用的方法名存在与该类，dart就能找到并调用对应的方法。

这说明dart可能用的是和OC一样的消息传递调用方法的机制，如上也可以看出来，通过将`helloworldOnExtension`发送给`x.get`来完成方法调用。
