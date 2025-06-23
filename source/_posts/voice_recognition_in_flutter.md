---
title: voice_recognition_in_flutter
date: 2023-06-21 00:11:00
tags: 
- Flutter
---

# Flutter 仿微信语音识别

1. 识别手势上滑，界面模糊。
2. ffi分发识别请求给安卓/iOS。
3. iOS 使用 Apple SpeechKit 内置语音识别；安卓使用 Azure。
4. ffi不断回调给 flutter 语音识别结果。
5. C++ 处理 iOS 和安卓回传的 audio samples。
6. 使用傅里叶变换将音频样本转换为频谱数据，将振幅传给 flutter 可视化。
