# 简介

YoloV5 是一种目标检测模型，属于 YOLO（You Only Look Once）系列模型的改进版本，专为实时检测任务设计。它通过一次性地在整个图像上预测目标的类别和位置，具有速度快、精度高的特点。
YoloV5 的核心优势在于其 **端到端的目标检测**。该模型可以在单次前向传播中同时预测目标类别和边界框位置，适用于实时应用场景，如视频监控、自动驾驶和无人机检测。

## YoloV5 关键技术

- CSPNet (Cross Stage Partial Networks)：这是 YoloV5 中用于优化神经网络层连接的技术，减少了冗余梯度信息的传播，增强了梯度流动，使得模型在保持较低计算量的同时，仍具备良好的学习能力和特征提取能力。
    
- FPN (Feature Pyramid Network) 和 PANet (Path Aggregation Network)：
    - FPN：通过多尺度特征融合，增强模型在不同尺寸下的检测能力，尤其是对小目标的检测。
    - PANet：进一步改进了特征融合机制，优化信息在网络中的传递路径，提升了检测性能，特别是在高分辨率图像中。
- Mosaic 数据增强：YoloV5 引入了 Mosaic 数据增强技术，它通过将四张图像随机拼接为一张来生成训练样本，从而增加训练数据的多样性和随机性。这有助于模型更好地泛化到不同场景，提升对复杂背景下目标的检测能力。
    
- AutoAnchor：自动根据数据集生成适合的 anchor box，省去了手动调整 anchor 的麻烦，确保 anchor 与数据集中目标的大小更加匹配，从而提升检测精度。
    
- CIoU 损失函数：相较于传统的 IoU 损失函数，CIoU（Complete IoU）综合考虑了目标框的重叠率、中心点距离以及宽高比，能够更准确地优化边界框预测，提升目标定位精度。
    
- Multi-scale prediction：YoloV5 通过多尺度预测实现不同大小目标的检测能力，确保从小物体到大物体都能在同一张图片上精确定位。
    
- Lightweight 模型结构：YoloV5 提供多种模型尺寸（如 YoloV5s、YoloV5m、YoloV5l、YoloV5x），适合不同计算资源需求的场景。轻量级版本如 YoloV5s 特别适合部署在移动设备或嵌入式系统中。
    
- PyTorch 框架：YoloV5 是基于 PyTorch 实现的，使用方便，训练和部署流程相对简单。PyTorch 的模块化设计使得 YoloV5 更容易进行自定义修改和调优


# 使用yolov5
## 安装yolov5
1. 在运行yolo前，需要配置好Anaconda环境。使用anaconda prompt创建虚拟环境
   ```cmd
   conda create -n '环境名' python=3.8 
	```
2. 激活当前环境
   ```cmd
   activate '环境名'
	```
3. 下载yolov5，VPN需要使用全局模式
   ```python
   git clone https: //github.com/ultralytics/yolov5
	```
	下载后的项目地址（默认）
	```
	C:\\users\user\yolov5
	```
4. 下载yolov5的预训练模型
   yolov5 共有四种模型：yolov5s、yolov5m、yolov5l、yolov5x。其中 yolov5s 目标检测速度最快，因为其网络参数最少，但相应的，检测效果相比是最差的；而 yolov5x 是检测效果最好的，参数最多，时间上最慢。
   将预训练模型放入项目文件夹中
   `通过 detect.py 对图像进行目标检测的实际操作时，detect.py 默认使用同目录下的 yolov5s.pt 模型，如果想用其他的模型，可以在命令里加入调用来指定`
5. 安装yolov5模型
   使用Anaconda Prompt进入项目地址
	```cmd
	cd C:\\users\user\yolov5
	```
   安装requirements.txt。
      ```python
	   pip install -r requirements.txt
	```
   使用清华镜像下载速度更快
	```python
	pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
	```
   
## gpu加速运行yolov5
由于 requirements.txt 依赖包里默认安装的 pytorch-cpu 版，安装 pytorch-gpu 版可以大幅度提升速度

## 训练yolov5
1. 使用[[Labelimg数据标注工具|Labelimg数据标注工具]]标注目标图片，文件目录格式如下
   ```
   -项目名
	   |- images
	   |- labels
	   |- info.yaml
	```
2. 记得创建info.yaml，名字可以随意。info.yaml文件中保存以下内容
   ```yaml
   # train and val data as 
   # 1) directory: path/images/, 
   # 2) file: path/images.txt, or 
   # 3) list: [path1/images/, path2/images/]
   train: ../yolo_A/images/ 
   val: ../yolo_A/images/
   # number of classes 
   nc: 1
   # class names 
   names: ['项目名']
	```
3. 使用Pycharm运行yolov5进行训练
## 检测yolov5
运行dectec.py文件
# 链接
- yolov5仓库地址 [github](https://github.com/ultralytics/yolov5)
- yolov5预训练模型[发布 ·Ultralytics/Yolov5 (github.com)](https://github.com/ultralytics/yolov5/releases)
- yolov5文档 [YOLOv5 -Ultralytics YOLO 文档](https://docs.ultralytics.com/zh/models/yolov5/)
- torch官网 https://pytorch.org/get-started/locally/