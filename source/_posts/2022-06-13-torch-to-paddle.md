---
title: torch模型转换为paddle模型
date: 2022-06-13 06:15:31
tags:
  - paddle
  - torch
---

因为训练好的模型框架来源比较多, 每个框架部署的方式不一样,造成不便, 可以通过模型转换统一到同一格式. 国内的框架是飞桨`paddle`, 所以可以统一到`paddle`模型.

比如把`torch` 框架 [ultralytics/yolov5](https://github.com/ultralytics/yolov5) 的模型转换为 `paddle` 的模型.


## 技术选型
转换路径: torch -> onnx -> paddle


百度提供[PaddlePaddle/X2Paddle](https://github.com/PaddlePaddle/X2Paddle) 工具来转换.由于版本的差异按照官方文档的代码转换方式全都报错,经过不同的尝试,最后导出成功.如果按官方文档的方式报错,可以尝试切换版本到已成功的环境.    
如下所示:   
```
onnx                    1.11.0
paddlepaddle            2.3.0
torch                   1.10.1
torchvision             0.11.2
x2paddle                1.3.6
```
`torchvision` 和 `torch` 的版本是需要相对应的,可以查看 [pypi](https://pypi.org/project/torchvision/) 上面的适配表.

目录结构:    
```
$ tree -L 1
.
|-- 1.jpg
|-- pd_model
|-- t1.py
|-- t2p_utils.py
|-- test.py
|-- venv
|-- yolov5
|-- yolov5s.onnx
`-- yolov5s.pt
```
## torch 转换为 onnx
已经成功的是v6.1版本,如果最新代码失败,可以切换到该版本尝试

```
# 下载代码,并安装依赖, 
git clone https://github.com/ultralytics/yolov5
cd yolov5
pip install -r requirements.txt  # install

python yolov5/export.py --weights yolov5s.pt --include onnx
```

## onnx 转换为 paddle
转换成功就保存在pd_model目录中
```
pip install x2paddle

x2paddle --framework=onnx --model=yolov5s.onnx --save_dir=pd_model
```
转换成功的目录
```
|-- inference_model
|   |-- model.pdiparams
|   |-- model.pdiparams.info
|   `-- model.pdmodel
|-- model.pdparams
`-- x2paddle_code.py
```


## 参考链接
- [TFLite, ONNX, CoreML, TensorRT Export ](https://github.com/ultralytics/yolov5/issues/251)
- [PaddlePaddle/X2Paddle](https://github.com/PaddlePaddle/X2Paddle)
- [torchvision](https://pypi.org/project/torchvision/)