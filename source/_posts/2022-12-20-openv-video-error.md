---
title: opencv之打开视频出现旋转错误
date: 2022-12-20 06:15:31
tags:
  - opencv
---

## 起因
做人脸动作活体检测的时候使用手机拍摄视频, 视频用`opencv`打开出现了上下颠倒的情况.
当前`opencv`版本是`4.6.0.66`, 使用版本`4.5.5.64`版本正常.

## 分析
### 使用ffmpeg查看信息
```
ffprobe -print_format json -select_streams v -show_streams -i video_20221124_135508.mp4
```
`-print_format json` 是输出的格式为json. `-select_streams v`是选择视频流,`v`是视频, `-show_streams` 是显示流信息. `-i video_20221124_135508.mp4`是输入文件地址.
输出结果:
```json
"streams": [
        {
            "index": 0,
            "width": 1920,
            "height": 1080,
            "coded_width": 1920,
            "coded_height": 1080,
            "side_data_list": [
                {
                    "side_data_type": "Display Matrix",
                    "rotation": 90
                }
            ]
        }
    ]
}
```
`rotation`是一个十进制数, 指定视频在显示之前应逆时针旋转的度数.
 这个解释来源于`ffmpeg`的`-display_rotation`参数. 
 还有一个叫`rotate`也是度数, 官方解释还未找到, 它们的关系在`opencv`源代码里面可以看出.
 `rotate`等于负的`rotation`, 如果`rotate`小于0, 则`rotate`等于`rotate`加360度.
 ```c++
 // https://github.com/opencv/opencv/blob/da4ac6b7eff2e8869567e4faaff73312f9e1ef57/modules/videoio/src/cap_ffmpeg_impl.hpp#L1777
 
 void CvCapture_FFMPEG::get_rotation_angle()
{
    rotation_angle = 0;
#if LIBAVFORMAT_BUILD >= CALC_FFMPEG_VERSION(57, 68, 100)
    const uint8_t *data = 0;
    data = av_stream_get_side_data(video_st, AV_PKT_DATA_DISPLAYMATRIX, NULL);
    if (data)
    {
        rotation_angle = -cvRound(av_display_rotation_get((const int32_t*)data));
        if (rotation_angle < 0)
            rotation_angle += 360;
    }
#elif LIBAVUTIL_BUILD >= CALC_FFMPEG_VERSION(52, 94, 100)
    AVDictionaryEntry *rotate_tag = av_dict_get(video_st->metadata, "rotate", NULL, 0);
    if (rotate_tag != NULL)
        rotation_angle = atoi(rotate_tag->value);
#endif
}
 ```

### opencv自带的旋转
搜索`opencv`的`Issues`可以知道, 大概是4.5版才加入的默认自动旋转视频, 使用的是`ffmpeg`的后端.
```python
# 默认是自动旋转, 可以设置为0来取消
cap.set(cv2.CAP_PROP_ORIENTATION_AUTO, 0)

# 通过CAP_PROP_ORIENTATION_META可以获取到rotate
orientation = int(cap.get(cv2.CAP_PROP_ORIENTATION_META))
```
自动旋转的源代码
```c++
// https://github.com/opencv/opencv/blob/97c6ec6d49cb78321eafe6fa220ff80ebdc5e2f4/modules/videoio/src/cap_ffmpeg.cpp#L144
void rotateFrame(cv::Mat &mat) const
    {
        bool rotation_auto = 0 != getProperty(CAP_PROP_ORIENTATION_AUTO);
        int rotation_angle = static_cast<int>(getProperty(CAP_PROP_ORIENTATION_META));

        if(!rotation_auto || rotation_angle%360 == 0)
        {
            return;
        }

        cv::RotateFlags flag;
        if(rotation_angle == 90 || rotation_angle == -270) { // Rotate clockwise 90 degrees
            flag = cv::ROTATE_90_CLOCKWISE;
        } else if(rotation_angle == 270 || rotation_angle == -90) { // Rotate clockwise 270 degrees
            flag = cv::ROTATE_90_COUNTERCLOCKWISE;
        } else if(rotation_angle == 180 || rotation_angle == -180) { // Rotate clockwise 180 degrees
            flag = cv::ROTATE_180;
        } else { // Unsupported rotation
            return;
        }

        cv::rotate(mat, mat, flag);
    }
```

## 自定义旋转
如果是其他的`opencv`不支持旋转的后端, 可以自己实现.
```python
# 方式1
cv2.rotate()

# 方式2
cv2.flip()

# 方式3
def rotate_bound(image, angle):
    # grab the dimensions of the image and then determine the
    # center
    (h, w) = image.shape[:2]
    (cX, cY) = (w // 2, h // 2)
    # grab the rotation matrix (applying the negative of the
    # angle to rotate clockwise), then grab the sine and cosine
    # (i.e., the rotation components of the matrix)
    # getRotationMatrix2D, angle正值表示逆时针旋转（坐标原点假定为左上角）
    M = cv2.getRotationMatrix2D((cX, cY), -angle, 1.0)
    cos = np.abs(M[0, 0])
    sin = np.abs(M[0, 1])
    # compute the new bounding dimensions of the image
    nW = int((h * sin) + (w * cos))
    nH = int((h * cos) + (w * sin))
    # adjust the rotation matrix to take into account translation
    M[0, 2] += (nW / 2) - cX
    M[1, 2] += (nH / 2) - cY
    # perform the actual rotation and return the image
    return cv2.warpAffine(image, M, (nW, nH))
```

## 参考链接
- [Vertical video detection flipped upside-down](https://hub.nuaa.cf/ultralytics/yolov5/issues/8493)
- [cv::VideoCapture ignores orientation metadata](https://github.com/opencv/opencv/issues/15499)
- [Rotate images (correctly) with OpenCV and Python](https://pyimagesearch.com/2017/01/02/rotate-images-correctly-with-opencv-and-python/)