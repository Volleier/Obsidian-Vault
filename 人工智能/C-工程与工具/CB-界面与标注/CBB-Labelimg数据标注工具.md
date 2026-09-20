# Labelimg简介
LabelImg 是一个开源的图像标注工具，使用 Python 编写，支持图形用户界面。它用于手动标注图像中的对象，以生成训练计算机视觉模型所需的标注文件。它主要支持 Pascal VOC 和 YOLO 格式的输出。

# Labelimg使用
## Labelimg安装
1. 在仓库下载完Labelimg项目后，先删除labelImg-master\data\predefined_classes.txt（避免标记的时候出现奇怪的类别）
2. 在yolo环境中创建Labelimg需要的环境
      ```
	conda install pyqt=5
	conda install -c anaconda lxml
	pyrcc5 -o libs/resources.py resources.qrc
	```
3. 使用Lebelimg命令进入工具
   ```
   python labelimg.py
	```
4. 修改设置方便使用
   ![[Labelimg数据标注工具-1.png]]
5. **大多数情况下记得把yolo格式点上**

## Labelimg使用
点击Open dir选择我们图片所在的images文件夹，选择之后会弹窗让你选择labels所在的文件夹。
当然如果选错了，也可以点change save dir进行修改。
左上角可以选择标注的标签

| 快捷键              | 功能                                        |
| ---------------- | ----------------------------------------- |
| Ctrl + u         | Load all of the images from a directory   |
| Ctrl + r         | Change the default annotation target dir  |
| Ctrl + s         | Save                                      |
| Ctrl + d         | Copy the current label and rect box       |
| Ctrl + Shift + d | Delete the current image  <br>            |
| Space            | Flag the current image as verified        |
| w                | Create a rect box                         |
| d                | Next image                                |
| a                | Previous image                            |
| del              | Delete the selected rect box              |
| Ctrl++           | Zoom in                                   |
| Ctrl--           | Zoom out                                  |
| ↑→↓←             | Keyboard arrows to move selected rect box |
# Labelimg下载
- Github仓库 https://github.com/tzutalin/labelImg