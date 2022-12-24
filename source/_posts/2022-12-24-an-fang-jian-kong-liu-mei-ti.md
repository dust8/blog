---
title: 安防监控流媒体综述
date: 2022-12-24 06:15:31
tags:
  - 流媒体
  - sip
---

## 需求
把摄像头集中管理, 多端可以预览视频, 可以进行喊话, 对讲功能.

碰到的问题:
- 摄像头官方的流量费太高; 并且提供的功能不全, 例如喊话, 对讲功能
- 第三方的搭建困难; 功能不全, 例如无喊话, 对讲功能; HLS延迟太大(基本40-60s)

## 调研结果
可以使用`wvp-GB28181-pro`(SIP服务器)+`ZLmMediaKit`(媒体服务器)来实现.

摄像头品牌多, 但基本都支持国标GB28181, GB28181是国家标准, 由公安部提出. 有2个版本, GB28181-2011是第一个版本, GB28181-2016是第二个版本, 进行了补充(例如支持TCP). 场景由内网的UDP, 到外网导致不稳定, 改为TCP.

GB28181 协议是在`SIP`协议上修改的, 通过TCP, UDP进行传输. 有控制和传输两方面, 控制是各种消息, 例如注册注销, 保活, invite(开始推流), bye(停止推流); 传输包括协议消息和媒体传输. 媒体传输是通过`RTP/RTCP`协议传输`PS`格式封装的视音频数据.

大体可以看作是4部分组成: 
- 媒体流接收者者(播放器)
- SIP服务器
- 媒体服务器
- 媒体流发布者(摄像机)

详细的流程可以看GB2818标准的文件, 这里简单描述下流程:    
- 摄像机注册到SIP服务器, 媒体服务器注册到SIP服务器
- SIP服务器发送`invite`消息给摄像机, 告诉推流地址为媒体服务器
- 摄像机推流到媒体服务器
- 媒体服务器接收流后, 转码为各种格式, 例如HLS, flv等
- 播放器开始播放视频

实际使用中会调整, SIP服务会提供restful接口来管理摄像机, 提供webhook接口来控制摄像机. 媒体服务器会把内部事件通过webhook请求发送到SIP服务器, 例如当播放器请求的流不存在事件, SIP服务器就会发送推流消息给摄像机, 让它推流到媒体服务器; 当无人观看流, 也会发送到SIP服务器, 就可以控制摄像机停止推流. 
这样按需推流可以节省流量, 不给服务器增加不必要的负担. 

下面是`ZLMediaKit`提供的按需推流的流程图:    
![使用ZLMediaKit的MediaServer进程可以实现按需推流](https://user-images.githubusercontent.com/11495632/67912185-1bcb9780-fbc4-11e9-9f99-02dbc7053ca1.png)


## SIP服务器
GB28181库比较少, 一些比较复杂的, 难扩展的(C++)就不列了.

| 平台 | 语言 | 优点 | 缺点 |
| --- | --- | --- | --- |
| 648540858/wvp-GB28181-pro | java | 比较全, 有ui | 未实现广播, 对讲 |
| panjjo/gosip | go |  | 支持功能比较少,未实现广播, 对讲. 无ui |


## 常见流媒体平台
| 平台 | 语言 | GB28181 | 优点 | 缺点 |
| --- | --- | --- | --- | --- |
|  ossrs/srs | c++ | 只支持部分SIP消息(注册, 邀请, 推流等), 无法停止推流(sip消息-bye). 4.0版本是分支, 5.0才合并到主分支. | 运营商级, 支持集群. 接收流端口可以复用 | gb28181功能不全. |
| ZLMediaKit/ZLMediaKit | c++ | 未支持sip信令, 只支持了接收推流功能 | 运营商级 | 未集成gui, 第三方都不更新. 接收流端口有多端口模式(兼容性好), 单端口模式(比较麻烦). |
| langhuihui/monibuca | go | 只支持部分SIP消息, 通过`panjjo/gosip`包来支持的 | 插件化, 好扩展, 入门容易, 支持完整sip信令 | 文档描述模糊, 核心开发人员少. gui闭源. v4版本发现GB28181推送的流,转化为HLS后无法播放. |


## 专业术语
- SIP 会话初始协议（Session Initiation - Protocol）：由 IETF - 制定的用于多方多媒体通信的框架协议。它是一个基于- 文本的应用层控制协议，独立于底层传输协议，用于建- 立、修改和终止 IP 网络上的双方或多方多媒体会话。
- IETF 互联网工程任务组（Internet Engineering Task - Force）
- SDP 会话描述协议（Session Description Protocol）
- PS 节目流（Program Stream）
- RTP 实时传输协议（Real-time Transport Protocol）
- RTCP 实时传输控制协议（Real-time Transport - Control Protocol）
- RTSP 实时流化协议（Real-Time Streaming Protocol）
- RTMP 实时信息传输协议（Real Time Message - Protocol）
- IPC 网络摄像机（InternetProtocolCamera）
- UA 用户代理（UserAgent）
- UAC 用户代理客户端（UserAgentClient）
- UAS 用户代理服务端（UserAgentServer）
- SSRC 同步源标识
- CNAME 规范性名字（Canonical Name）
- CSeq (Transaction Number)

## 使用 wvp-GB28181-pro
- 运行zlmediakit
`docker run -id -p 1935:1935 -p 8080:80 -p 8443:443 -p 8554:554 -p 10000:10000 -p 10000:10000/udp -p 8000:8000/udp -p 9000:9000/udp -p 30000-30005:30000-30005 zlmediakit/zlmediakit:master`
- 下载`wvp-GB28181-pro`代码, 按照官方文档编译, 配置, 运行. 使用到了`mysql`, `redis`. 按需推流还需要配置`user-settings/auto-apply-play` 为`true`.
```
# 运行redis. 因为免得和本地的redis端口冲突, docker运行的端口在默认端口+1. mysql同理
docker run -p 6380:6379 --name wvp-redis -d redis
# 运行mysql
docker run -itd --name wvp-mysql -p 3307:3306 -e MYSQL_ROOT_PASSWORD=123456 mysql

# 复制将要导入的sql文件到docker容器
docker cp mysql.sql wvp-mysql:/opt/mysql.sql
# 运行mysql客户端
docker exec -it wvp-mysql mysql -uroot -p
# 创建数据库
create database wvp;
# 切换当前使用数据库
use wvp;
# 导致数据
source /opt/mysql.sql;
```
- HLS地址是`zlmediakit`生成的, 详情查看wiki `https://github.com/zlmediakit/ZLMediaKit/wiki/%E6%92%AD%E6%94%BEurl%E8%A7%84%E5%88%99`. 格式是`http://somedomain.com/live/0/hls.m3u8`, 例如`http://127.0.0.1:8080/rtp/34020000001320000001_34020000001320000004/hls.m3u8`, `rtp`是固定的应用名, `34020000001320000001_34020000001320000004`是流ID, 由摄像机设备ID和通道ID组成.

如果需要查看docker容器日志
```
docker logs 容器id

# 实时日志
docker logs 容器id -f
```

## 公网部署
如果部署到公网需要开放端口和修改配置.
```
# 端口号, 备注
# 处理 http, 其他服务可以开放TCP和UDP
18080 wvp http
5060 wvp sip
8080 zlmediakit http
30000-30005 zlmediakit rtp
```
配置需要修改
```
sip:
    # [必须修改] 本机的IP. 必须是网卡的ip, 也就是内网ip. 云服务器有公网ip和内网ip.
    ip: 192.168.0.5
    
media:
    # [必须修改] zlm服务器的内网IP, sdp-ip与stream-ip使用默认值的情况下
    ip: 192.168.0.5
    # 需改成外网ip
    stream-ip: xxx.xxx.xxx.xxx
    sdp-ip: xxx.xxx.xxx.xxx
```



## 踩坑
### 问题1. 注意开放端口
为了简单运行, 是用docker来运行的, 官方文档的命令可能没映射需要的端口.
`srs` 的GB28181模块需要开放`5060`端口, 这个端口号是默认的SIP端口.
`
docker run --rm -it --env CANDIDATE="192.168.1.107" -p 1935:1935 -p 1985:1985 -p 5060:5060 -p 8080:8080 -p 9000:9000 registry.cn-hangzhou.aliyuncs.com/ossrs/srs:5 ./objs/srs -c conf/gb28181.conf
`

使用`wvp-GB28181-pro`时配置`ZLMediaKit`的接收流端口, 是多端口`port-range`, 而`ZLMediaKit`默认的docker运行命令是不带有的, 需要增加端口段映射`-p 30000-30005:30000-30005`, 因为如果端口太多, docker显示太长, 这里只映射了5个.
`docker run -id -p 1935:1935 -p 8080:80 -p 8443:443 -p 8554:554 -p 10000:10000 -p 10000:10000/udp -p 8000:8000/udp -p 9000:9000/udp -p 30000-30005:30000-30005 zlmediakit/zlmediakit:master`


### 问题2. wvp-GB28181-pro编译
`wvp-GB28181-pro`前端是后端集成的, 所以先编译完前端, 在编译后端. 前端编译完后文件保存在`src\main\resources\static`. 要使用文档的`cnpm`, 如果使用`npm`会报错. 运行后端后, 访问`localhost:18080`, 账户`admin`密码`admin`.

### 问题3. ZLMediaKit无法连接
`wvp-GB28181-pro`配置中需要指定`ZLMediaKit`的id, `ZLMediaKit`默认是`your_server_id`, 而`wvp-GB28181-pro`的配置默认是`FQ3TF8yT83wh5Wvz`, 需要改成一致即可.


### 问题4. 数据库找不到platform_gb_stream表
日志报该错误, 查看导入的sql文件, 发现先创建了该表, 后面又删掉了. 是在`https://github.com/648540858/wvp-GB28181-pro/commit/170b0a18004639a1855939bcdd3384ea546deed4`改版的时候弄错了. 修改sql文件, 把删除语句放到创建语句前面.


## 参考链接
 - [ossrs/srs](https://github.com/ossrs/srs)
 - [ZLMediaKit/ZLMediaKit](https://github.com/ZLMediaKit/ZLMediaKit)
 - [langhuihui/monibuca](https://github.com/langhuihui/monibuca)
 - [648540858/wvp-GB28181-pro](https://github.com/648540858/wvp-GB28181-pro)
 - [GB28181 Solution](https://github.com/orgs/GB28181/repositories)
 - [DLGCY_GB28181](https://gitee.com/DLGCY_GB28181)
 - [使用ZLMediaKit实现按需推流](https://github.com/ZLMediaKit/ZLMediaKit/wiki/%E4%BD%BF%E7%94%A8ZLMediaKit%E5%AE%9E%E7%8E%B0%E6%8C%89%E9%9C%80%E6%8E%A8%E6%B5%81)
 - [解决Cannot read properties of null (reading ‘pickAlgorithm‘)](https://blog.csdn.net/yyzx_yyzx/article/details/125716768)
 - [基于国标GB/T28181标准从海康摄像头获取PS流](https://blog.yasking.org/a/get-ps-stream-from-hik.html)
 - [海康摄像头PS流格式解析（RTP/PS/H264）](https://blog.yasking.org/a/hikvision-rtp-ps-stream-parser.html)
 - [将海康IPC摄像头PS流保存到文件并转换为MP4视频](https://blog.yasking.org/a/hikvision-rtp-ps-stream-save-to-file.html)
 - [MPEG-2 TS 容器封装格式概览](https://blog.yasking.org/a/learn-mpeg2-ts.html)
 - [基于GB28181从海康NVR获取目录/点播/回播信令备忘](https://blog.yasking.org/a/hikvision-nvr-playback-with-gb28181.html)
 - [基本流的第一次封装（PES）](https://blog.yasking.org/a/learn-pes-package.html)
 - [FFmpeg 推流与转码命令备忘（持续更新）](https://blog.yasking.org/a/ffmpeg-command-note.html)
 - [将IPC/NVR获取的PS流推送到srs分发为rtmp/http-flv/hls](https://blog.yasking.org/a/gb28181-ps-stream-to-rtmp.html)
 - [技术解码 | GB28181/SIP/SDP 协议](https://zhuanlan.zhihu.com/p/545703291)