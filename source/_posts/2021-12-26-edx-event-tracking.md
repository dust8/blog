---
title: edx源码分析之event-tracking
date: 2021-12-26 05:15:31
tags:
  - edx
  - python
---

event-tracking 是一个事件跟踪库, 为后续的分析提供原始数据.位置如下图所示.    
![](https://edx.readthedocs.io/projects/edx-installing-configuring-and-running/en/open-release-lilac.master/_images/Analytics_Pipeline.png)


怎么使用不多说,可以看示例; 也可以去edx的其他代码库搜索它们怎么用的. 额外说下事件的命名,是由点号分割来避免冲突,规则为`namespace.object.action`(https://event-tracking.readthedocs.io/en/latest/user_guide/design.html#best-practices, https://edx.readthedocs.io/projects/edx-developer-guide/en/latest/analytics.html#naming-events).

![](/assert/2021-12-26-edx-event-tracking.svg)




# event-tracking
 - https://github.com/edx/event-tracking
 - https://event-tracking.readthedocs.io/en/latest/overview.html
 - https://edx.readthedocs.io/projects/edx-developer-guide/en/latest/analytics.html
## tracker
  - Tracker
    - 追踪器
    - emit
      - 提交事件.emit(name=None, data=None)
      - name为事件名,像'edx.course.enrollment.activated',
        可以用'namespace.object.action'结构;
        data为事件,类型为可以序列化的字典
  - DjangoTracker
    - 从django配置初始化的追踪器,继承Tracker
  - register_tracker
  - get_tracker
  - emit
## locator
  - DefaultContextLocator
  - ThreadLocalContextLocator
## backend
  - 事件追踪后端,必须提供一个send(event)方法,用来处理接收到的事件
  - LoggerBackend
    - 使用python的logger的后端,事件使用logger处理
  - MongoBackend
    - 使用mongodb数据库的后端,事件插入数据库,默认是eventtracking库的events表
  - RoutingBackend
    - 将事件路由到适当的后端
    - Processors
      - 处理器集合.管道处理,类似责任链模式
    - backends
      - 处理后端集合,可以是所有的处理后端,包括RoutingBackend自身
  - AsyncRoutingBackend
    - 异步路由后端继承RoutingBackend,使用了celery来处理
  - SegmentBackend
    - 将事件发送到segment.com的后端

## processor
  - 处理器,用来过滤事件
  - RegexFilter
    - 正则处理器
  - NameWhitelistProcessor:
    - 白名单处理器

## 参考链接
- [edx/event-tracking](https://github.com/edx/event-tracking)
- [edx/event-tracking文档](https://event-tracking.readthedocs.io/en/latest/)
- [Analytics](https://edx.readthedocs.io/projects/edx-developer-guide/en/latest/architecture.html#analytics)
- [安装 edX Insights 的选项](https://edx.readthedocs.io/projects/edx-installing-configuring-and-running/en/open-release-lilac.master/insights/install_insights.html)