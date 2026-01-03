# 介绍

线段绘制算法用于在离散像素网格上逼近连续线段。它解决了从连续几何（两点确定一条线）到离散像素（像素中心采样、取整）之间的映射问题，是光栅化、字体渲染、2D 引擎与低级图形库的基础。

# 数学基础

- 参数方程：给定端点$(x_0,y_0),(x_1,y_1)$，一条连续线可写为$$x(t)=x_0+t(x_1-x_0), y(t)=y_0+t(y_1-y_0),t\in[0,1]$$
- 斜率：$m=(y_1-y_0)/(x_1-x_0)$。像素化核心问题是如何在每个像素行/列选择一个像素使得离散曲线视觉上接近连续直线。
- 像素定义：常用像素代表其中心坐标，但简化实现通常把像素映射到整点。

# 方法

## 直接法（参数采样/逐点法）

### 思路
#### 核心想法
以参数 $t\in[0,1]$ 对线段的参数方程
$$x(t)=x_0+t(x_1-x_0),\qquad y(t)=y_0+t(y_1-y_0)$$
做均匀采样，计算浮点坐标后映射到最接近的像素中心（通常用四舍五入，即 $(\operatorname{round}(x),\operatorname{round}(y))$，也可按需使用其他取整规则）。

#### 采样域选择（常用两种等价策略）
- 按最大轴采样：令 $\Delta x=x_1-x_0,\ \Delta y=y_1-y_0$，取
  $$\text{steps}=\max(|\Delta x|,|\Delta y|).$$
  对 $i=0,\dots,\text{steps}$ 计算
  $$t=\frac{i}{\text{steps}},\quad x=x(t),\ y=y(t)$$
  并映射到像素网格，保证沿主轴每像素至少检查一次。
- 按主轴逐增采样
- 当 $|\Delta x|>|\Delta y|$ 时以 $x$ 为主轴，令 $x$ 以单位步进从 $x_0$ 到 $x_1$（含或不含端点视策略而定），按 $x$ 的等距值反算 $t$ 或直接用线性关系计算 $y$；否则以 $y$ 为主轴同理。主轴采样避免在极端斜率下采样过密或过稀。

取整与端点策略：
- 建议使用 $\operatorname{round}(\cdot)$ 将浮点坐标映射到像素中心，并在实现中明确端点包含规则（是否包含 $(x_1,y_1)$）。零长度线（steps = 0）应直接绘制端点像素。

数值与误差：
- 参数采样每点独立计算，避免增量法的累积误差，但存在浮点计算和四舍五入引入的独立误差。
- 当斜率极大或接近 $0$ 时，按均匀 $t$ 采样会导致主轴方向像素跳跃不均，主轴采样可缓解该问题。
- 在按非主轴采样时要防止除零（例如垂直线按 $x$ 递增会产生零步长），需特殊处理垂直/水平线。
- 实现时明确边界条件（含端点、取整规则）以保证可复现的像素覆盖结果。

### 优缺点

- 优点：
  - 实现简单
- 缺点：
  - 需要浮点运算，斜率极大极小会退化，积累误差明显，视觉效果差。

### 实现
#### 实现要点
1. 计算 dx, dy, steps = max(|dx|,|dy|)。
2. 如果 steps\=\=0，绘制端点并返回。
3. 对 i 从 0 到 steps：t = i/steps；计算 x,y；plot(round(x), round(y))。
4. 明确端点策略并退出。

#### 算法实现
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
核心想法：选择主轴（使主轴每步为 1 像素），用增量值累计推进坐标，每步取整绘制。设 $Δx = x_1 - x_0$，$Δy = y_1 - y_0$，令
$$\text{steps}=\max(|\Delta x|,|\Delta y|)$$
或在已知主轴时取
$$\text{steps}=\begin{cases}|\Delta x|,&|\Delta x|>|\Delta y|\\[4pt]|\Delta y|,&\text{否则}\end{cases}.$$
增量计算：
$$x_{\text{inc}}=\frac{\Delta x}{\text{steps}},\qquad y_{\text{inc}}=\frac{\Delta y}{\text{steps}}.$$
起始坐标使用浮点：
$$x=x_0,\ y=y_0$$
每步做
$$x\gets x+x_{\text{inc}},\quad y\gets y+y_{\text{inc}},\quad \text{plot}(\operatorname{round}(x),\operatorname{round}(y)).$$

数值与漂移：DDA 通过累加代替重复乘法，降低开销但会产生浮点累积误差（漂移）。可用以下办法减轻：
- 周期性重算：每 N 步用参数 $t=i/\text{steps}$ 重新计算 $x=x_0+t\Delta x,\ y=y_0+t\Delta y$。
- 固定点整数：选定缩放因子 $S$，用整数表示 $X=\lfloor x\cdot S\rfloor$，增量为 $X_{\text{inc}}=\lfloor x_{\text{inc}}\cdot S\rfloor$，全部用整数累加以避免浮点误差。
处理斜率与方向：通过步长符号确定方向，若 $x_0>x_1$ 或 $y_0>y_1$ 应交换端点或使 $x_{\text{inc}},y_{\text{inc}}$ 带符号（例如 $x_{\text{inc}}=\pm1$ 当以 x 为主轴时）。对负斜率、垂直/水平线进行专门判断以防除零或无步进。端点策略：一般包含起点与终点；若用于多段折线可选择不绘制每段终点以避免重复像素。

### 优缺点
- 优点：
  - 比直接法更稳定
- 缺点：
  - 仍依赖浮点，性能与整数算法比略差

### 算法实现
算法步骤：
1. 输入起点 $(x_1, y_1)$ 和终点 $(x_2, y_2)$
2. 计算 $Δx = x_2 - x_1$，$Δy = y_2 - y_2$
3. 计算 $m = Δy / Δx$
4. 根据 $m$ 选择上述三种情况之一计算下一点
5. 重复直到到达终点
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
### 算法实现
算法步骤：
1. 输入起点 $(x_1, y_1)$ 和终点 $(x_2,y_2)$
2. 计算 $Δx = x_2 - x_1$，$Δy = y_2 - y_1$
3. 初始决策参数 $p_0 = 2Δy - Δx$
4. 对每个点 $(x_k, y_k)$：
    - 如果 $p_k < 0$：
        - $p_{k+1} = p_k + 2Δy$
        - $(x_{k+1}, y_{k+1}) = (x_k + 1, y_k)$
    - 如果 $p_k ≥ 0$：
        - $p_{k+1} = p_k + 2Δy - 2Δx$
        - $(x_{k+1}, y_{k+1}) = (x_k + 1, y_k + 1)$
5. 重复直到 $x_k = x_2$
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

### 算法实现
算法步骤：
1. 输入起点 $(x_1, y_1)$ 和终点 $(x_2, y_2)$
2. 计算 $Δx$、$Δy$
3. 计算 $Δd = 2(Δy - Δx)$
4. 初始决策参数 $d_0 = 2Δy - Δx$
5. 对每个点：
    - 如果 $d_i < 0$：
        - $(x_{k+1}, y_{k+1}) = (x_k + 1, y_k)$
        - $d_n = d_i + 2Δy$
    - 如果 $d_i ≥ 0$：
        - $(x_{k+1}, y_{k+1}) = (x_k + 1, y_k + 1)$
        - $d_n = d_i + Δd$
6. 重复直到终点
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