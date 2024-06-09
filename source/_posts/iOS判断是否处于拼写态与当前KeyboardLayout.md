---
title: iOS判断是否处于拼写态与当前KeyboardLayout
date: 2024-05-30 10:37:56
tags:
  - iOS
---

最近在做一个相关的需求，需要判断当前的输入法是否处于拼写态，以及当前的输入法的布局。

拼写态即用户正在拼写还未形成最终的字、词的状态，例如在中文全键上 typing 拼音，输入法上还待选中文但输入框显示字母；又比如英文全键输入字母还未成单词时，候选框待选。 这个词感觉很难用中文解释，用中文关键字 google 几乎不会得到结果。

键盘布局（KeyboardLayout），例中文全键、中文九宫格、英文全键、emoji等。

<!--more-->

拼写态可以通过 Apple 文档找到相关说明，主要相关方法是 `UITextField.markedTextRange`。

标记文本也算做文本属性，所以 textDidChangeNotification 在拼写态完成时（即点击候选框，完成拼写）会触发两次，一次是改变 markedTextRange，一次是改变 text，看似是两次一样的 text 回调。

> Marked text is provisionally inserted text that requires user confirmation; it occurs in multistage text input. The current selection, which can be a caret or an extended range, always occurs within the marked text.
> 标记文本是临时插入的文本，需要用户确认；它发生在多阶段文本输入中。标记文本总是会被加入或扩展。

利用该方法可以得到是否拼写态的表达式

```swift
let isSpelling = (textField.markedTextRange != nil) && (!textField.markedTextRange!.isEmpty)
```

KeyboardLayout 在看如何分辨 qwerty 键盘与其他 layout （例如 azerty）区别的时候解决

> https://stackoverflow.com/questions/61249593/ios-detect-keyboard-layout-e-g-qwerty-azerty

总结下来，得到当前 textfield 的键盘布局的方法是

```swift
let layout = textfield.textInputMode?.value(forKey: "identifierWithLayouts") as? String
```

得到所有系统支持的键盘的方法是

```swift
let avaliableKeyboards = UserDefaults.standard.object(forKey: "AppleKeyboards")
```

例

![WeCom截圖_f5ee37e7-cc97-4e1a-af9d-96eb16c9aa45](https://raw.githubusercontent.com/xhzq233/images/master/2024-06-03/WeCom%E6%88%AA%E5%9C%96_f5ee37e7-cc97-4e1a-af9d-96eb16c9aa45.png)