---
title: dispatch_method_in_swift
date: 2025-01-06 10:54:32
tags:
- swift
- dispatch method
---

# Swift 函数的派发方法

static func 会变成直接派发 `direct dispatch`

重点探讨 `dynamic` & `@objc`

1.swift

```swift
class PigeonError {
    dynamic func wrapResult() {}
}
```

生成对应 sil 中间代码并且替换 shadow name 

```
$ swiftc 1.swift -emit-silgen | xcrun swift-demangle > 1.out
```

### `dynamic func` compare with `@objc func`

几个关键区别，dynamic 多了 dynamically_replacable，说明其动态可替换

```swift
class PigeonError {
    dynamic func wrapResult() {}
}
// PigeonError.wrapResult()
+++ sil hidden [dynamically_replacable] [ossa] @main.PigeonError.wrapResult() -> () : $@convention(method) (@guaranteed PigeonError) -> () {
  ...
}
```

@objc 多了 objc_method，一个暴露给 objc runtime 的壳方法，copy_value & borrow PigeonError实例，再 call 源函数，再 return。

```swift
class PigeonError {
  @objc func wrapResult()
  @objc deinit
  init()
}

// @objc PigeonError.wrapResult()
sil private [thunk] [ossa] @@objc main.PigeonError.wrapResult() -> () : $@convention(objc_method) (PigeonError) -> () {
// %0                                             // user: %1
bb0(%0 : @unowned $PigeonError):
  %1 = copy_value %0 : $PigeonError               // users: %6, %2
  %2 = begin_borrow %1 : $PigeonError             // users: %5, %4
  // function_ref PigeonError.wrapResult()
  %3 = function_ref @main.PigeonError.wrapResult() -> () : $@convention(method) (@guaranteed PigeonError) -> () // user: %4
  %4 = apply %3(%2) : $@convention(method) (@guaranteed PigeonError) -> () // user: %7
  end_borrow %2 : $PigeonError                    // id: %5
  destroy_value %1 : $PigeonError                 // id: %6
  return %4 : $()                                 // id: %7
} // end sil function '@objc main.PigeonError.wrapResult() -> ()'
```

两者皆是 vtable dispatch 方法，@objc 并未使用消息派发机制

```swift
sil_vtable PigeonError {
  #PigeonError.wrapResult: (PigeonError) -> () -> () : @main.PigeonError.wrapResult() -> ()	// PigeonError.wrapResult()
  #PigeonError.init!allocator: (PigeonError.Type) -> () -> PigeonError : @main.PigeonError.__allocating_init() -> main.PigeonError	// PigeonError.__allocating_init()
  #PigeonError.deinit!deallocator: @main.PigeonError.__deallocating_deinit	// PigeonError.__deallocating_deinit
}
```

### `@objc dynamic func`

与 objc 唯一区别就是，func wrapResult 从 vtable中去掉了，说明转为objc 消息发送了

```swift
sil_vtable PigeonError {
  #PigeonError.init!allocator: (PigeonError.Type) -> () -> PigeonError : @main.PigeonError.__allocating_init() -> main.PigeonError	// PigeonError.__allocating_init()
  #PigeonError.deinit!deallocator: @main.PigeonError.__deallocating_deinit	// PigeonError.__deallocating_deinit
}
```

[objc.out](/files/objc.out)

[dynamic.out](/files/dynamic.out)

[objc_dynamic.out](/files/objc_dynamic.out)

[static.out](/files/static.out)
