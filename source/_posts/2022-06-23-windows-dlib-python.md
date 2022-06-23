---
title: windows编译dlib的python安装包
date: 2022-06-23 06:15:31
tags:
  - python
  - dlib
---

[dlib](https://github.com/davisking/dlib) 是一个机器学习算法的工具库.想在`python`中使用它的追踪算法.由于在最新的`pypi`上 `dlib` 已经没有上传已经编译好的 `windows` 版本, 最近的`windows`版本是 `python3.6`的.所以需要自己编译.

## 下载源码
[https://github.com/davisking/dlib](https://github.com/davisking/dlib)  
打开 `setup.py` 文件可以看到使用方式
```
# 编译并安装
python setup.py install
```
编译需要 `vs c++` , `cmake` 等依赖环境.本机还好, 如果在部署机上也安装依赖,就太麻烦了,所以可以编译成 `whl` 格式的安装包.


## 编译`whl`安装包
```
# 安装依赖
pip install twine wheel

# 编译. 这样在dist目录里面就可以看编译出的安装包
python setup.py bdist_wheel
```

有些`cpu`太老(如奔腾cpu)不支持`avx`指令, 编译的时候就需要指定参数
```
# 编译, 不使用avx指令, 并清除之前的编译文件
python setup.py bdist_wheel --no USE_AVX_INSTRUCTIONS  --clean
```

## 备注
- whl 安装包的命名是有规则的,不能乱命名.不然安装不了,报错为`ERROR: **.whl is not a supported wheel on this platform`

## 参考链接
- [Using dlib from Python](http://dlib.net/compile.html)