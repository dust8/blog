---
title: ubuntu设置虚拟内存
date: 2023-03-23 06:15:31
tags:
  - ubuntu
---
## 查看环境
### 查看内存情况
`free -h` 命令查看内存情况和swap分区大小
```
              total        used        free      shared  buff/cache   available
Mem:          1.9Gi       259Mi       1.4Gi       1.0Mi       207Mi       1.5Gi
Swap:         0B          0B       0B
```
可以看到swap目前是0, 也就是没有设置虚拟内存.

### 查看磁盘空间大小
`df -h` 命令查看磁盘空间情况. 因为下面需要在磁盘创建一个文件当作虚拟内存, 所以看下在哪个目录下创建比较好.
```
Filesystem      Size  Used Avail Use% Mounted on
overlay         5.8G  589M  5.0G  11% /
devtmpfs        887M     0  887M   0% /dev
tmpfs           954M     0  954M   0% /dev/shm
tmpfs           191M  1.8M  189M   1% /run
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           954M     0  954M   0% /sys/fs/cgroup
/dev/mmcblk0p1  128M   63M   66M  49% /boot
/dev/mmcblk0p7   15G  4.7G  9.9G  33% /data
/dev/mmcblk0p4  2.4G  2.2G  157M  94% /media/root-ro
/dev/mmcblk0p5  5.8G  589M  5.0G  11% /media/root-rw
/dev/mmcblk0p6  2.0G  226M  1.6G  13% /opt
/dev/mmcblk0p2  2.9G   52M  2.8G   2% /recovery
tmpfs           191M     0  191M   0% /run/user/1000
```
可以看到`/data`目录空间比较大, 可以在此目录创建.

## 创建虚拟内存
1.创建swap文件
```
sudo dd if=/dev/zero of=/data/swapfile bs=1M count=4096
```
`/data/swapfile` 就是创建的虚拟内存文件路径, `bs=1M` 代表文件大小单位是1M, `count=4096` 代表创建多少个单位, 合起来就是4个G.

2. 设置swap文件
```
sudo mkswap /data/swapfile
```

3. 激活swap文件
```
sudo swapon /data/swapfile
```

4. 校验是否生效
运行 `free -h`, 查看`swap`部分`total`是否为0, 不是0则生效.
```
              total        used        free      shared  buff/cache   available
Mem:          1.9Gi       259Mi       1.4Gi       1.0Mi       207Mi       1.5Gi
Swap:         4.0Gi          0B       4.0Gi

```

## 设置开机启动
上面的设置是临时生效的, 为了重启仍然生效, 需要修改 `/etc/fstab` 文件, 设置开机自动挂载.
在 `/etc/fstab` 文件后面添加一行.
```
/data/swapfile  swap    swap   defaults  0   0
```

上面是一般情况就设置成功了, 有时碰到特殊情况, 例如系统是定制的, `/etc/fstab` 文件重启后会被还原为默认配置. 这时可以通过 `systemd` 来管理开机启动, 也就是设置一个开机启动的服务, 该服务启动后执行激活swap文件命令.
在 `/usr/lib/systemd/system/` 目录创建一个 `my-swap.service` 文件, 文件内容为
```
[Unit]
Description=my swap

[Service]
ExecStart=sudo swapon /data/swapfile

[Install]
WantedBy=multi-user.target
```
完成后可以通过`systemctl cat my-swap` 产看该配置, 目测下是否正确.
运行`systemctl start my-swap` 执行该服务, 然后运行`free -h` 检查是否激活swap成功, 如果没有就在检查下配置.
运行`systemctl enable my-swap` 则正式启用该服务, 重启后会自动运行该服务.


## 取消虚拟内存
1. 运行 `sudo swapoff /data/swapfile` 关闭系统交换区
2. 运行 `sudo rm /data/swapfile` 删除系统交换区文件, 释放占用的磁盘空间.
3. 取消开机启动. 删除添加到 `/etc/fstab` 里的`/data/swapfile` 那一行. 如果使用的是`systemd`, 则运行`systemctl disable my-swap`, 在删除`my-swap.service`文件.

## 参考链接
- [Systemd 入门教程：命令篇](http://ruanyifeng.com/blog/2016/03/systemd-tutorial-commands.html)