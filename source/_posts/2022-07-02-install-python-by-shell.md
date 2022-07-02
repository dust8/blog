---
title: 一键安装Python的shell脚本
date: 2022-07-02 06:15:31
tags:
  - python
  - shell
---
```bash
#!/bin/bash

# 一键安装Python

py_version=3.8.8
install_path=/usr/local

install_dependencies () {
    # update the packages index
    sudo apt-get update

    # build dependencies
    sudo apt-get build-dep python3 -y
    sudo apt-get install pkg-config -y

    # build all optional modules
    # https://devguide.python.org/setup/#install-dependencies
    sudo apt-get install -y build-essential gdb lcov pkg-config \
        libbz2-dev libffi-dev libgdbm-dev libgdbm-compat-dev liblzma-dev \
        libncurses5-dev libreadline6-dev libsqlite3-dev libssl-dev \
        lzma lzma-dev tk-dev uuid-dev zlib1g-dev
}

install_python () {
    # 国外源太慢, 可以用阿里源 https://registry.npmmirror.com/
    # wget -c https://www.python.org/ftp/python/$py_version/Python-$py_version.tgz 
    wget -c https://registry.npmmirror.com/-/binary/python/$py_version/Python-$py_version.tgz

    tar -xzf Python-$py_version.tgz -C /tmp

    cd /tmp/Python-$py_version/

    ./configure --prefix=$install_path --enable-optimizations --with-ssl

    make -j2

    # 避免覆盖默认的python
    sudo make altinstall
}

# http://c.biancheng.net/view/2991.html
read -p "请输入安装的python版本(例如: 3.8.8): " py_version
echo "python版本: $py_version"

install_dependencies;
install_python;
```

## 参考链接
- [Python Developer's Guide](https://devguide.python.org/setup/#install-dependencies)
- [Shell read命令：读取从键盘输入的数据](http://c.biancheng.net/view/2991.html)