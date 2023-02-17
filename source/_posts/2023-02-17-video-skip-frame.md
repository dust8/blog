---
title: 视频流跳帧处理
date: 2023-02-17 06:15:31
tags:
  - 视频
---

跳帧是视频流处理中, 跳过一些帧不处理.

## 为什么跳帧

一个是设备性能低, 无法实时处理视频流, 导致任务堆积, 造成程序报错无法运行. 例如在分析RTSP视频流的时候, AI推理太慢, 导致视频流接收堆积, 程序报错.
一个是减少设备运行负载, 可以运行更多的任务. 例如某些任务不需要每帧图片进行分析, 通过跳帧既符合需求, 也不浪费设备资源.

## 跳帧原理

如果是`N`帧中取一帧, 可以通过总帧数对`N`取余, 判断余数是否等于零, 如果等于则取该帧.
如果是每隔`N`帧取一帧, 也是同理, 通过总帧数对`N+1`取余, 判断余数.
如果是`M`分钟取`N`帧, 也是同理, 需要先获取`FPS`也就是每分钟的帧数, 公式为`总帧数 % (M * FPS / N)`.

## 实现

```python
import cv2

uri = "xxx"
cap = cv2.VideoCapture(uri)
# 查询视频帧率. 后面可以用来计算skip_n参数, 例如1分钟取3帧, 公式为int(1*fps/3), 假设fps是24, 则skip_n为8.
fps = cap.get(cv2.CAP_PROP_FPS)
skip_n = 8  # 跳帧, 每隔N帧取一帧
frame_count = 0
while True:
    ret, frame = cap.read()
    frame_count += 1

    # 判断是否需要跳帧
    if frame_count % skip_n != 0:
        continue

    cv2.imshow("frame", frame)
    if cv2.waitKey(1) & 0xFF == ord("q"):
        break

cv2.destroyAllWindows()
cap.release()
```

