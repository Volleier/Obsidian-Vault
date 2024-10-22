卷积神经网络（Convolutional Neural Network，简称CNN）是一种深度学习模型，特别适用于处理具有网格结构的数据，如图像。CNN通过卷积层、池化层和全连接层的组合，能够自动提取图像的特征，并进行分类、检测等任务。
![[附件/人工智能/卷积神经网络-1.gif]]
# 主要组成部分
如图所示，卷积网络架构与常规人工神经网络架构非常相似。主要由五层组成：数据输入层，卷积计算层，ReLU激励层，池化层，全连接层
![[卷积神经网络-2.png]]

## 数据输入层 （Input Layer）

## 卷积计算层 （Convolutional Layer）

## ReLU激励层（ReLU Layer）

## 池化层 (Pooling Layer)

## 全连接层 （Full Connect Layer）





## 卷积层（Convolutional Layer）
卷积层是CNN的核心组件，通过卷积操作提取输入数据的局部特征。卷积操作使用滤波器（或称为卷积核）在输入数据上滑动，计算滤波器与输入数据的点积，从而生成特征图（Feature Map）。

- 滤波器（Filter）：通常是一个小矩阵，如3x3或5x5。
- 步幅（Stride）：滤波器在输入数据上滑动的步长。
- 填充（Padding）：在输入数据的边缘填充零，以保持特征图的尺寸。

## 激活函数（Activation Function）
激活函数引入非线性，使得神经网络能够学习复杂的模式。常用的激活函数包括ReLU（Rectified Linear Unit）、Sigmoid和Tanh。

- ReLU：f(x) = max(0, x)
- Sigmoid：f(x) = 1 / (1 + exp(-x))
- Tanh：f(x) = (exp(x) - exp(-x)) / (exp(x) + exp(-x))

## 池化层（Pooling Layer）
池化层用于减少特征图的尺寸，从而降低计算复杂度和防止过拟合。常见的池化操作有最大池化（Max Pooling）和平均池化（Average Pooling）。

- 最大池化：取池化窗口内的最大值。
- 平均池化：取池化窗口内的平均值。

## 全连接层（Fully Connected Layer）
全连接层将前一层的所有神经元与当前层的每个神经元相连，用于综合提取的特征进行分类或回归任务。

## Dropout层
Dropout层是一种正则化技术，通过在训练过程中随机丢弃一部分神经元，防止模型过拟合。

# 常见的CNN模型

## LeNet-5
LeNet-5是最早的卷积神经网络之一，由Yann LeCun等人提出，主要用于手写数字识别。

## AlexNet
AlexNet在2012年的ImageNet竞赛中取得了显著的成绩，推动了深度学习的发展。它引入了ReLU激活函数和Dropout层。

## VGGNet
VGGNet通过使用较小的卷积核（3x3）和较深的网络结构（16-19层），在多个图像分类任务中表现出色。

## ResNet
ResNet引入了残差连接（Residual Connection），解决了深层网络中的梯度消失问题，使得网络可以更深。

## Inception（GoogLeNet）
Inception网络通过引入Inception模块，在同一层中使用不同尺寸的卷积核，捕捉不同尺度的特征。

# 应用领域

- 图像分类：如手写数字识别、物体识别等。
- 目标检测：如人脸检测、行人检测等。
- 图像分割：如医学图像分割、语义分割等。
- 图像生成：如生成对抗网络（GAN）生成图像。