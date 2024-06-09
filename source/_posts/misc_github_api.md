---
title: "misc: github api"
date: 2024-06-09 12:23:12
tags:
  - github api
  - flutter
---

需求：

1. 客户端表现，写完即销毁的 Notes
2. 发送到特定的另一方查阅

大致实现：

1. 用户名称双方绑定，储存本地
2. 笔记本地暂存，提交后销毁
3. 仅拉取另一方的信息

一开始打算使用iOS原生，cloudkit+sharing提供后端，双方共享储存。但由于 apple 系原生软件不允许私人使用（自签要给每个人手机开开发者，还只有7天），除了企业认证、tf 内测等歪门邪道，以上还需持有每年688的 appledev 会员。

于是考虑使用 flutter + web 代替，尽量完整 apple 系原生体验，后端使用 github issues api，参考评论区系统 https://jw1.dev/2022/10/23/a01/

