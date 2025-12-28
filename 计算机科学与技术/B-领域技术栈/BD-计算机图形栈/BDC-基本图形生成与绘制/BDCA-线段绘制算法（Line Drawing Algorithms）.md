# 介绍

线段绘制算法用于在离散像素网格上逼近连续线段。它解决了从连续几何（两点确定一条线）到离散像素（像素中心采样、取整）之间的映射问题，是光栅化、字体渲染、2D 引擎与低级图形库的基础。

# 数学基础

- 参数方程：给定端点$(x_0,y_0),(x_1,y_1)$，一条连续线可写为$$x(t)=x_0+t(x_1-x_0), y(t)=y_0+t(y_1-y_0),t\in[0,1]$$
- 斜率：$m=(y_1-y_0)/(x_1-x_0)$。像素化核心问题是如何在每个像素行/列选择一个像素使得离散曲线视觉上接近连续直线。
- 像素定义：常用像素代表其中心坐标，但简化实现通常把像素映射到整点。

# 方法

## 直接法（参数采样/逐点法）

### 思路

按$t$均匀采样或按 x（或 y）递增计算对应的 y（或 x），随后四舍五入到像素网格。

### 优缺点

- 优点：
  - 实现简单
- 缺点：
  - 需要浮点运算，斜率极大极小会退化，积累误差明显，视觉效果差。

### 代码实现
```python
"""
直接法，返回像素列表 (x, y)。
"""
def direct_line(x0, y0, x1, y1):
    x0, y0, x1, y1 = float(x0), float(y0), float(x1), float(y1)
    dx = x1 - x0
    dy = y1 - y0
    steps = int(max(abs(dx), abs(dy)))
    if steps == 0:
        return [(int(round(x0)), int(round(y0)))]
    pts = []
    for i in range(steps + 1):
        t = i / steps
        x = x0 + t * dx
        y = y0 + t * dy
        pts.append((int(round(x)), int(round(y))))
    return pts
```

## DDA（数字差分算法）

### 思路

使用增量更新，按像素步进方向计算另一个坐标（浮点），避免每像素都做乘法。

### 优缺点

- 优点：
  - 比直接法更稳定
- 缺点：
  - 仍依赖浮点，性能与整数算法比略差

### 代码实现
```python
"""
DDA实现，返回像素列表 (x, y)。
"""
def dda_line(x0, y0, x1, y1):
    x0, y0, x1, y1 = float(x0), float(y0), float(x1), float(y1)
    dx = x1 - x0
    dy = y1 - y0
    steps = int(max(abs(dx), abs(dy)))
    if steps == 0:
        return [(int(round(x0)), int(round(y0)))]
    x_inc = dx / steps
    y_inc = dy / steps
    x, y = x0, y0
    pts = []
    for _ in range(steps + 1):
        pts.append((int(round(x)), int(round(y))))
        x += x_inc
        y += y_inc
    return pts
```


## Bresenham 算法（整数算法）

### 思想
利用误差项（决策变量）判断下一个像素应向哪一个方向移动，仅用加减与位移（或比较）。

### 优缺点
- 优点：
  - 高效、只用整数、视觉效果好。
  - 可扩展到宽线和圆的绘制（Bresenham 圆算法）
### 代码实现
```python
"""
Bresenham 整数算法（通用实现，处理所有斜率）。返回像素列表 (x, y)。
"""
def bresenham_line(x0, y0, x1, y1):
    x0, y0, x1, y1 = map(int, (x0, y0, x1, y1))
    pts = []
    steep = abs(y1 - y0) > abs(x1 - x0)
    if steep:
        x0, y0, x1, y1 = y0, x0, y1, x1
    swapped = False
    if x0 > x1:
        x0, x1 = x1, x0
        y0, y1 = y1, y0
        swapped = True
    dx = x1 - x0
    dy = abs(y1 - y0)
    error = dx // 2
    ystep = 1 if y0 < y1 else -1
    y = y0
    for x in range(x0, x1 + 1):
        pts.append((y, x) if steep else (x, y))
        error -= dy
        if error < 0:
            y += ystep
            error += dx
    if swapped:
        pts.reverse()
    return pts
```

## 中点法与变种

### 思路

中点法与 Bresenham 关系紧密，都是基于判断像素中心相对真实线的位置（误差）来决定像素选择。
变种包括处理对称性、端点策略（是否包含终点）和亚像素偏移

### 代码实现
```python
def midpoint_line(x0, y0, x1, y1):
    """
    中点法（使用中点测试，带轴交换与端点反转处理）。返回像素列表 (x, y)。
    注意：为简洁使用了浮点中点测试。
    """
    x0, y0, x1, y1 = map(int, (x0, y0, x1, y1))
    pts = []
    steep = abs(y1 - y0) > abs(x1 - x0)
    if steep:
        x0, y0, x1, y1 = y0, x0, y1, x1
    swapped = False
    if x0 > x1:
        x0, x1 = x1, x0
        y0, y1 = y1, y0
        swapped = True
    dx = x1 - x0
    dy = y1 - y0  # 保留符号以判断方向
    y = y0
    for x in range(x0, x1 + 1):
        pts.append((y, x) if steep else (x, y))
        # 中点坐标（x+1, y+0.5）相对于线的符号判断
        xm = x + 1
        ym = y + 0.5
        val = (y1 - y0) * (xm - x0) - (x1 - x0) * (ym - y0)
        if dy >= 0:
            if val > 0:
                y += 1
        else:
            if val < 0:
                y -= 1
    if swapped:
        pts.reverse()
    return pts
```

## 抗锯齿

### 思路

用像素的覆盖比例（或亮度权重）将连续线的强度分配到相邻像素，产生平滑效果。

### 优缺点
- 优点：
  - 视觉平滑
- 缺点：
  - 需要浮点、混合操作和更多像素更新（性能消耗更大）

### 代码实现
较多技术实现抗锯齿 