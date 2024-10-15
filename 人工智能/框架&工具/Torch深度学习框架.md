# Torch简介
PyTorch 是一个开源的深度学习框架，由 Facebook 的人工智能研究团队（FAIR）开发，主要用于构建和训练神经网络。它以易用性、灵活性和强大的动态计算图（Dynamic Computational Graph）功能著称，广泛应用于研究和工业中的人工智能模型开发。
## PyTorch 的特点
- 动态计算图：PyTorch 提供了灵活的动态计算图，使得在模型训练过程中可以根据需要动态修改计算图。这一特性使得调试和开发变得更加容易，尤其是与 TensorFlow 1.x 相比，后者采用的是静态计算图。
- 自动微分 (Autograd)：PyTorch 的 Autograd 模块可以自动计算梯度，从而简化了反向传播算法的实现。通过跟踪所有操作，PyTorch 可以自动计算任意复杂模型中参数的梯度，这大大简化了模型的训练过程。
- 模块化和易用性：PyTorch 使用面向对象的模块化设计，允许开发者使用现有模块或自定义模块来构建复杂的神经网络。其 API 设计直观，易于使用，开发者可以轻松实现复杂的深度学习模型。
- 广泛的模型支持：PyTorch 支持多种类型的深度学习模型，包括卷积神经网络（CNN）、循环神经网络（RNN）、生成对抗网络（GAN）等。它提供了许多预训练模型，可以帮助开发者快速搭建模型。
- 强大的 GPU 支持：PyTorch 提供对 GPU 和多 GPU 训练的原生支持，能够有效加速模型的训练和推理。通过简单的代码调用，开发者可以轻松将模型从 CPU 切换到 GPU。
- 社区支持与生态系统：PyTorch 拥有活跃的开发者社区，许多开源项目、预训练模型和研究论文都基于 PyTorch 实现。同时，PyTorch 还拥有诸如 `Torchvision`、`TorchText` 等库，提供图像、文本等数据处理和模型训练的工具，丰富了其生态系统。
- 与 Python 无缝集成：PyTorch 完全依赖 Python 语言，不仅继承了 Python 的简洁性和易读性，还与 Python 的各种库（如 NumPy）高度兼容，支持直接转换与操作。

## PyTorch 的核心模块

- `torch` 模块：这是 PyTorch 的核心模块，提供了张量操作、线性代数、随机数生成等功能，类似于 NumPy，但增加了 GPU 加速支持。
    
- `torch.nn` 模块：用于构建神经网络，提供了大量的神经网络层和损失函数，可以方便地定义和构建复杂的深度学习模型。
    
- `torch.autograd` 模块：自动微分模块，跟踪张量上的所有操作，并根据操作生成相应的反向传播路径，自动计算梯度。
    
- `torch.optim` 模块：提供了多种优化算法，如 SGD、Adam、RMSprop 等，帮助用户更方便地训练神经网络。
    
- `torch.utils.data` 模块：用于加载和处理数据集，支持批量加载、随机打乱数据等功能，方便用户处理大型数据集。

# Torch安装
Torch安装默认是cpu版本的，gpu版本需要额外下载
在电脑显卡被正确安装并且打上驱动的情况下，在电脑CMD中查看显卡支持的Cuda版本
```
   nvidia-smi.exe
```
选择Cuda的对应版本，在Conda环境中运行Command（Conda命令）
![[../../附件/yolov5运行流程-1.png]]
检查torch是否被正确安装，在对应环境中输入下列cmd命令
```cmd
pip list
```
输入python进入python模式
```cmd
python
```
导入torch查看能否正确使用
```python
import torch
torch.cuda.is_available()
```
没报错，并且返回True就是正确安装
## Cuda安装
CUDA（Compute Unified Device Architecture）是由 NVIDIA 开发的并行计算平台和编程模型，允许开发者利用 NVIDIA GPU 的强大计算能力来加速计算密集型任务。通过 CUDA，开发者可以使用类似 C/C++ 的语言编写程序，在 GPU 上执行复杂的并行计算。
英伟达开发网站下载Cuda Toolkits
按照步骤安装即可（确保英伟达显卡驱动已经被正确安装）

# 链接
- Torch下载地址 https://download.pytorch.org/whl/torch_stable.html
- CUDA Toolkit下载地址 [CUDA Toolkit | NVIDIA Developer](https://developer.nvidia.com/cuda-downloads)
- pytorch网站 [PyTorch](https://pytorch.org/get-started/locally/)