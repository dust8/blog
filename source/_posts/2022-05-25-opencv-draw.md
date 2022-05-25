---
title: OpenCV 中的绘图函数
date: 2022-05-25 06:15:31
tags:
  - OpenCV
---

## 画线

[cv.line()](https://docs.opencv.org/4.x/d6/d6e/group__imgproc__draw.html#ga7078a9fae8c7e7d13d24dac2520ae4a2)  
pt1: 第一个点  
pt2:是第二个点

```python
cv.line(img, pt1, pt2, color[, thickness[, lineType[, shift]]]) ->img

import numpy as np
import cv2 as cv


# Draw a diagonal blue line with thickness of 5 px
cv.line(img,(0,0),(511,511),(255,0,0),5)

cv.imshow('frame',img)
k = cv.waitKey(0)
```

## 画矩形

[cv.rectangle()](https://docs.opencv.org/4.x/d6/d6e/group__imgproc__draw.html#ga07d2f74cadcf8e305e810ce8eed13bc9)  
pt1: 矩形的顶点, 左上角的点  
pt2: 与 pt1 相对的矩形的顶点, 右下角的点

```python
cv.rectangle(img, pt1, pt2, color[, thickness[, lineType[, shift]]]) ->img
cv.rectangle(img, rec, color[, thickness[, lineType[, shift]]]) ->img

cv.rectangle(img,(384,0),(510,128),(0,255,0),3)
```

## 画圆

[cv.circle()](https://docs.opencv.org/4.x/d6/d6e/group__imgproc__draw.html#gaf10604b069374903dbd0f0488cb43670)
center: 中心点坐标  
radius: 圆半径

```python
cv.circle(img, center, radius, color[, thickness[, lineType[, shift]]]) ->img

cv.circle(img,(447,63), 63, (0,0,255), -1)
```

## 画椭圆

[cv.ellipse()](https://docs.opencv.org/4.x/d6/d6e/group__imgproc__draw.html#ga28b2267d35786f5f890ca167236cbc69)

```python
cv.ellipse(img, center, axes, angle, startAngle, endAngle, color[, thickness[, lineType[, shift]]]	) ->img
cv.ellipse(img, box, color[, thickness[, lineType]]	) ->img

cv.ellipse(img,(256,256),(100,50),0,0,180,255,-1)
```

## 画多边形

[cv.polylines()](https://docs.opencv.org/4.x/d6/d6e/group__imgproc__draw.html#gaa3c25f9fb764b6bef791bf034f6e26f5)
pts: 多边形曲线数组  
isClosed: 指示绘制的多段线是否闭合的标志。 如果它们是闭合的，该函数会从每条曲线的最后一个顶点到它的第一个顶点绘制一条线。

在画不规则区域时用到过. 例如在统计人流时,因为统计的区域是不规则的,也许就是个凹凸这样的形状,需要在图像上画出统计的区域.

```python
cv.polylines(img, pts, isClosed, color[, thickness[, lineType[, shift]]]	) ->img

# 要绘制多边形，首先需要顶点坐标。
# 将这些点放入一个形状为 ROWSx1x2 的数组中，其中 ROWS 是顶点数，它应该是 int32 类型。
# 在这里，我们用黄色绘制一个带有四个顶点的小多边形。
pts = np.array([[10,5],[20,30],[70,20],[50,10]], np.int32)
pts = pts.reshape((-1,1,2))
cv.polylines(img,[pts],True,(0,255,255))

print(pts)
[[[10  5]]

 [[20 30]]

 [[70 20]]

 [[50 10]]]
```

## 画字

[cv.putText()](https://docs.opencv.org/4.x/d6/d6e/group__imgproc__draw.html#ga5126f47f883d730f633d74f07456c576)
org: 图像中文本字符串的左下角

```python
cv.putText(	img, text, org, fontFace, fontScale, color[, thickness[, lineType[, bottomLeftOrigin]]]	) ->	img

font = cv.FONT_HERSHEY_SIMPLEX
cv.putText(img,'OpenCV',(10,500), font, 4,(255,255,255),2,cv.LINE_AA)
```

## 参考链接

- [Drawing Functions in OpenCV](https://docs.opencv.org/4.x/dc/da5/tutorial_py_drawing_functions.html)
