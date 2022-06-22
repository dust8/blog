---
title: Dockerfile修改ubuntu源
date: 2022-06-22 06:15:31
tags:
  - docker
  - ubuntu
---

`Dockerfile` 里面需要安装软件,有时国外源太慢,导致安装不了.可以通过替换 apt 源来解决

## 替换源
阿里源: [https://developer.aliyun.com/mirror/ubuntu](https://developer.aliyun.com/mirror/ubuntu)


## 修改方法
### 通过命令直接替换字符
```
RUN sed -i s@/archive.ubuntu.com/@/mirrors.aliyun.com/@g /etc/apt/sources.list
RUN apt clean
```

### 通过替换源文件
1.在该文件夹中创建`sources.list`文件
2.在`sources.list`中添加国内镜像源
3.在dockerfile中添加命令,把`sources.list`映射到容器内
```
ADD sources.list /etc/apt
```