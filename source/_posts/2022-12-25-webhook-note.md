---
title: webhook笔记
date: 2022-12-25 06:15:31
tags:
  - webhook
---


webhook 是用来和其他系统通信的, 把内部系统的事件发送到外部系统.

可以看作是发布/订阅模式, 或者观察者模式, 虽然有些不同, 但大致是一样的. 
好处是外部系统如果想知道内部系统的事件, 不需要轮询, 有变化就会被通知到.

当内部系统(发布者)有事件发生, 就会发送api请求(通常是post)到外部系统(订阅者), 请求载荷就是事件数据, 外部系统的接口收到请求, 做出响应.

## 常见场景
### github
例如提交代码到github, 就会运行ci/cd. 这里就是利用的webhook, github系统和ci/cd系统可以是2个不同的公司产品. github是发布者, ci/cd是订阅者, 当ci/cd订阅了代码提交事件, 一旦代码被提交, github就会发送请求到ci/cd, 告诉ci/cd发生了代码提交事件, ci/cd就知道需要拉取新的代码进行测试部署.
```
+---------------+                 +---------------+
|               |     webhook     |               |
|    github     |  -------------> |    CI/CD      |
|               |                 |               |
+---------------+                 +---------------+
```

### 支付宝/微信支付
做支付的时候, 支付宝和微信支付都需要配置支付回调, 支付回调地址就是我们的业务系统. 支付宝/微信支付系统就是发布者, 业务系统就是订阅者. 当发生支付成功事件时, 支付宝/微信支付就会发送请求到我们业务系统, 请求的数据包含订单号,状态等, 我们业务系统收到后就知道哪个订单支付成功了.

### 飞书机器人/钉钉机器人
飞书机器人/钉钉机器人都提供一个webhook地址, 我们可以向这个地址发送请求. 例如我们开发一个报警系统, 一旦监控的网站没有响应, 就向webhook地址发送请求, 飞书/钉钉接收到请求后, 就会在群里发送消息了. 这里报警系统就是发布者, 飞书机器人/钉钉机器人就是订阅者.


### 流媒体服务器
常见的流媒体服务器中(srs, ZLMediaKit,monibuca等 ), 都提供webhook配置, 把内部的事件发送出去. 例如在监控领域, 当有人向流媒体服务器请求某个摄像头视频流时, 流媒体服务器就会把这个事件发送到业务系统, 业务系统收到后就可以控制摄像头推流到流媒体服务器; 当某个视频流无人观看时, 流媒体也会把这个事件发送到业务系统, 业务系统就可以控制摄像头停止推流. 这样就避免了无人观看视频时, 摄像头还一直推流, 节约了流量. 

## 如何开发webhook接收者
其实就是服务端的一个普通web http 接口.

## 如何开发webhook发布者
其实就是一个能发送web http请求的客服端.
最基本就是: 一个是url地址, 确定了向谁发送.
更完整的可以包括:
- 事件类型. 事件很多, 需要响应的处理. 可以使用`namespace.noun.verb` 格式命名, 例如`github.issue.created`.
- 事件数据. 每个事件可能有不同的内容, 例如`issue`地址
- 订阅事件类型集合. 由于事件类型很多, 可以只订阅指定类型的事件. 例如["*"]订阅全部事件, ["github.issue.created"]只订阅`issue`创建的事件.
- 名称. 用来描述这个webhook

更加健壮的可以包括:
- 发送请求重试功能, 每次重试时间间隔递增. 例如1s, 4s, 16s, 256s.
- 发送请求队列


## 参考链接
- [观察者模式](https://refactoringguru.cn/design-patterns/observer)
- [Webhook 服务的设计与实现](https://blog.whichxjy.com/webhook-service-design/)
- [How to Build a Webhook System in Rails Using Sidekiq](https://keygen.sh/blog/how-to-build-a-webhook-system-in-rails-using-sidekiq/)
- [github 关于web挂钩](https://docs.github.com/cn/developers/webhooks-and-events/webhooks/about-webhooks)
- [atlassian Webhooks](https://developer.atlassian.com/server/jira/platform/webhooks/)
- [Build a Shipment Notification Service with Python, Flask, Twilio and EasyPost](https://www.twilio.com/blog/build-shipment-notification-service-python-flask-twilio-easypost)
- [Building Webhooks Into Your Application: Guidelines and Best Practices](https://workos.com/blog/building-webhooks-into-your-application-guidelines-and-best-practices)
- [Should You Build a Webhooks API?](https://brandur.org/webhooks)
- [asciiflow](https://asciiflow.com/)
- [zulip-Incoming webhook integrations](https://zulip.com/api/incoming-webhooks-overview)