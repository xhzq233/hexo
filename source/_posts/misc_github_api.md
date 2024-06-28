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

私以为这样一个仅双方交流的笔记软件用不着专门提供后端（其实是因为没有现有服务器），自行解决办法可以跨越物理距离面对面快传，甚至走 p2p 形式。最终选择使用某些公有服务。

客户端方面一开始打算使用iOS原生，cloudkit+sharing提供后端，双方共享储存。但由于 apple 系原生软件不允许私人使用（自签要给每个人手机开开发者，还只有7天），除了企业认证、tf 内测等歪门邪道，以上还需持有每年688的 appledev 会员。

于是考虑使用 flutter + web 代替，尽量完整 apple 系原生体验，后端使用 github issues api，参考评论区系统 https://jw1.dev/2022/10/23/a01/

----

Update

以上大部分作废，github oauth app还是需要回传 token，静态 pages 无法实现。

于是借用了一位爹的服务器打算自己造一下储存，典中典之 gin+gorm 后端。

之后又想了下用狗狗公共服务懒得自己搭数据库，使用 firebase database + messaging + google oauth + 爹的服务器上的pushNotification服务作为后端，flutter 作 web 前端。

