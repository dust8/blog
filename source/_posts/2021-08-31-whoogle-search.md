---
title: whoogle-search 元搜索引擎
date: 2021-08-31 05:15:31
tags:
  - whoogle-search
  - 搜索
---

今天在 `github` 的 `python Trending` 看到一个叫 [benbusby/whoogle-search](https://github.com/benbusby/whoogle-search) 的项目. 它是一个自部署,无广告,隐私保护的元搜索引擎.    
点击查看demo [http://search.jyt321.com](http://search.jyt321.com)


## 安装
最简单的就是用 `docker` 来安装了
```bash
docker pull benbusby/whoogle-search
docker run --publish 5000:5000 --detach --name whoogle-search benbusby/whoogle-search:latest
```

在配置 `nginx`, 绑定域名, 就有一个属于你自己网址的搜索引擎了.