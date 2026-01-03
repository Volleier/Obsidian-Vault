# 介绍
线段裁剪的目标是计算给定线段在裁剪窗口（通常为轴对齐矩形）内可见的那一段。典型应用包括窗口系统、图形渲染管线与几何预剪裁。


# 线段裁剪算法
## Cohen-Sutherland 算法（区域编码法）
### 基本思想
将平面按窗口划分为9个区域。对每个点用4位二进制码编码位置（Top, Bottom, Right, Left），然后利用位运算快速判定完全可见、完全不可见或需要裁剪。
区域编码规则（每位表示相对于窗口的关系）：
    - L (Left)：点在窗口左侧，即 $x < x_{min}$
    - R (Right)：点在窗口右侧，即 $x > x_{max}$
    - B (Bottom)：点在窗口下侧，即 $y < y_{min}$
    - T (Top)：点在窗口上侧，即 $y > y_{max}$

### 实现
#### 实现思想
 1. 对端点 $P_0(x_0,y_0)$ 和 $P_1(x_1,y_1)$ 计算编码 $code_0$ 与 $code_1$。
 2. 若 $(code_0\,|\,code_1) = 0$，两个端点均在窗口内：接受整段线段。
 3. 若 $(code_0\,\&\,code_1) \ne 0$，两端点在同一窗口外区域的同一侧：拒绝整段线段（不可见）。
 4. 否则，选择编码非零的端点（位于窗口外），按其编码的某一位（例如 L、R、T、B）计算与对应窗口边界的交点：
	 - 若选择左边界：与 $x = x_{min}$ 求交，可由参数方程或直线方程求得交点坐标。
	 - 同理处理右/上/下边界。
 5. 用交点替换原来的外部端点，重新计算其编码，回到第2步循环，直到接受或拒绝。

#### 算法实现
``` python
LEFT, RIGHT, BOTTOM, TOP = 1, 2, 4, 8

def compute_code(x, y, xmin, xmax, ymin, ymax):
    code = 0
    if x < xmin: code |= LEFT
    if x > xmax: code |= RIGHT
    if y < ymin: code |= BOTTOM
    if y > ymax: code |= TOP
    return code

def cohen_sutherland_clip(x0, y0, x1, y1, xmin, xmax, ymin, ymax):
    code0 = compute_code(x0, y0, xmin, xmax, ymin, ymax)
    code1 = compute_code(x1, y1, xmin, xmax, ymin, ymax)

    while True:
        if (code0 | code1) == 0:
            # 完全可见
            return (x0, y0, x1, y1)
        if (code0 & code1) != 0:
            # 完全不可见
            return None

        # 选择一个在窗口外的端点
        out_code = code0 if code0 != 0 else code1
        if out_code & TOP:
            # 与 y = ymax 求交
            if y1 != y0:
                x = x0 + (x1 - x0) * (ymax - y0) / (y1 - y0)
            else:
                x = x0
            y = ymax
        elif out_code & BOTTOM:
            if y1 != y0:
                x = x0 + (x1 - x0) * (ymin - y0) / (y1 - y0)
            else:
                x = x0
            y = ymin
        elif out_code & RIGHT:
            if x1 != x0:
                y = y0 + (y1 - y0) * (xmax - x0) / (x1 - x0)
            else:
                y = y0
            x = xmax
        elif out_code & LEFT:
            if x1 != x0:
                y = y0 + (y1 - y0) * (xmin - x0) / (x1 - x0)
            else:
                y = y0
            x = xmin

        # 用交点替换外部端点并更新编码
        if out_code == code0:
            x0, y0 = x, y
            code0 = compute_code(x0, y0, xmin, xmax, ymin, ymax)
        else:
            x1, y1 = x, y
            code1 = compute_code(x1, y1, xmin, xmax, ymin, ymax)
```
算法关键要点
- 采用整数位运算实现编码测试，速度快且分支少。
- 选择计算交点时，最好使用浮点运算得出精确交点坐标，然后将其赋回端点（保持坐标为浮点以便后续计算）。
- 对垂直或水平线求交时注意避免除以零（可用不同的求交公式或分支处理）。
- 对多段折线裁剪时注意端点重复：通常对每段都包含起点但在连接处跳过重复终点。
## Liang–Barsky 算法（参数化、不等式法）
### 基本思想
把线段用参数 $u\in[0,1]$ 表示，利用裁剪窗口的四个不等式直接求出参数区间 $[u_1,u_2]$，无需循环地逐步裁剪端点，边界交点参数只需计算一次。
- 参数化表示：
$$
x(u)=x_0+u\Delta x,
\qquad y(u)=y_0+u\Delta y,
\qquad \Delta x=x_1-x_0,
\ \Delta y=y_1-y_0
$$

- 窗口内条件等价为四个不等式：
$$
x_{min}\le x_0+u\Delta x\le x_{max},
\qquad y_{min}\le y_0+u\Delta y\le y_{max}
$$

把每个不等式整理为统一形式：$u\,p_k\le q_k,\quad k=1..4$，其中通常取
$$ \begin{align}
p_1 &= -\Delta x, & q_1 &= x_0 - x_{\min} \\
p_2 &= \Delta x,  & q_2 &= x_{\max} - x_0 \\
p_3 &= -\Delta y, & q_3 &= y_0 - y_{\min} \\
p_4 &= \Delta y,  & q_4 &= y_{\max} - y_0
\end{align} $$

### 实现
#### 实现思想
 1. 初始化：$u_1=0,\ u_2=1$。
 2. 对每个边界 $k=1..4$：
	 - 若 $p_k==0$：线段与该边界平行。若 $q_k<0$，则线段在该边界外（不可见），直接拒绝。否则不影响 $[u_1,u_2]$。
	 - 否则计算交点参数 $u = q_k / p_k$。
		 - 若 $p_k < 0$（“进入”边），更新 $u_1 = \max(u_1, u)$。
		 - 若 $p_k > 0$（“离开”边），更新 $u_2 = \min(u_2, u)$。
 3. 若在任一步出现 $u_1>u_2$，则线段完全不可见；否则可见子段为 $u\in[u_1,u_2]$，对应端点为 $P(u_1),\ P(u_2)$。

#### 算法实现
``` python
def liang_barsky_clip(x0, y0, x1, y1, xmin, xmax, ymin, ymax, eps=1e-12):
    dx = x1 - x0
    dy = y1 - y0
    p = [-dx, dx, -dy, dy]
    q = [x0 - xmin, xmax - x0, y0 - ymin, ymax - y0]

    u1, u2 = 0.0, 1.0
    for pk, qk in zip(p, q):
        if abs(pk) < eps:
            # 平行于该边界
            if qk < 0:
                return None
            else:
                continue
        u = qk / pk
        if pk < 0:
            u1 = max(u1, u)
        else:
            u2 = min(u2, u)
        if u1 > u2:
            return None

    cx0 = x0 + u1 * dx
    cy0 = y0 + u1 * dy
    cx1 = x0 + u2 * dx
    cy1 = y0 + u2 * dy
    return (cx0, cy0, cx1, cy1)```
算法关键要点
- 使用浮点计算 $u$，但对精度敏感时可考虑相对容差（如比较时加入小量 eps）。
- 若仅需判断可见性，可以在计算出任一边界的拒绝条件时返回，避免额外计算。
- Liang–Barsky 在每个边界仅做一次算术运算，理论上比 Cohen-Sutherland 少重复求交，效率更高。

## Nicholl–Lee–Nicholl (NLN) 算法（区域细分，最小求交次数）
### 基本思想
以第一个端点 $P_0$ 为参考，将平面按扇区细分（通常分为 8 个扇形区域），根据第二个端点 $P_1$ 所在扇区直接决定需要与哪些边界求交，从而在多数情况下只需最多两次求交就能完成裁剪。
相比 Cohen-Sutherland 可能多次剪掉外部小段，NLN 通过预先分类避免冗余的重复求交，特别在许多常见几何配置下能减少运算量。

### 实现
#### 实现思想
1. 首先对 $P_0$ 做快速编码判断（是否在窗口内）；若 $P_0$ 在外部，可先把问题转换为其在内部的等价情况或直接处理对称情形。
2. 以 $P_0$ 为中心，将窗口用相对角线与边界划分为若干区域（例如左上、上、右上、右、右下、下、左下、左）；根据 $P_1$ 的所属区域选择对应的裁剪路径。
3. 每种扇区对应一棵小的决策树（流程）——直接给出至多两次需要求交的边界，求出交点，组合为裁剪后线段。

#### 算法实现
``` python
def inside(x, y, xmin, xmax, ymin, ymax):
    return xmin <= x <= xmax and ymin <= y <= ymax

def intersect_with_boundary(x0, y0, dx, dy, boundary, xmin, xmax, ymin, ymax):
    # 返回 (u, x, y) 或 None
    eps = 1e-12
    if boundary == 'left':
        if abs(dx) < eps: return None
        u = (xmin - x0) / dx
        y = y0 + u * dy
        if ymin - eps <= y <= ymax + eps: return (u, xmin, y)
    elif boundary == 'right':
        if abs(dx) < eps: return None
        u = (xmax - x0) / dx
        y = y0 + u * dy
        if ymin - eps <= y <= ymax + eps: return (u, xmax, y)
    elif boundary == 'bottom':
        if abs(dy) < eps: return None
        u = (ymin - y0) / dy
        x = x0 + u * dx
        if xmin - eps <= x <= xmax + eps: return (u, x, ymin)
    elif boundary == 'top':
        if abs(dy) < eps: return None
        u = (ymax - y0) / dy
        x = x0 + u * dx
        if xmin - eps <= x <= xmax + eps: return (u, x, ymax)
    return None

def nln_clip(x0, y0, x1, y1, xmin, xmax, ymin, ymax):
    # 若任一端点在窗口内，将其作为 P0
    if not inside(x0, y0, xmin, xmax, ymin, ymax):
        if inside(x1, y1, xmin, xmax, ymin, ymax):
            # 交换，使 P0 在内
            x0, y0, x1, y1 = x1, y1, x0, y0
        else:
            # 两端都在外部：回退到 Liang–Barsky 保证正确性
            return liang_barsky_clip(x0, y0, x1, y1, xmin, xmax, ymin, ymax)

    # 现在保证 P0 在内或者两端都外但已回退
    dx = x1 - x0
    dy = y1 - y0

    # 确定 P1 的扇区（相对 P0）
    region_x = 'center'
    if x1 < xmin: region_x = 'left'
    elif x1 > xmax: region_x = 'right'
    region_y = 'center'
    if y1 < ymin: region_y = 'bottom'
    elif y1 > ymax: region_y = 'top'

    # 根据扇区选择候选边界（NLN 的核心：只检查这些边）
    candidates = []
    if region_x == 'left' and region_y == 'top':
        candidates = ['top', 'left']
    elif region_x == 'center' and region_y == 'top':
        candidates = ['top']
    elif region_x == 'right' and region_y == 'top':
        candidates = ['top', 'right']
    elif region_x == 'right' and region_y == 'center':
        candidates = ['right']
    elif region_x == 'right' and region_y == 'bottom':
        candidates = ['right', 'bottom']
    elif region_x == 'center' and region_y == 'bottom':
        candidates = ['bottom']
    elif region_x == 'left' and region_y == 'bottom':
        candidates = ['bottom', 'left']
    elif region_x == 'left' and region_y == 'center':
        candidates = ['left']
    else:
        # P1 在窗口内（center,center）——整段可见
        return (x0, y0, x1, y1)

    intersections = []
    for b in candidates:
        res = intersect_with_boundary(x0, y0, dx, dy, b, xmin, xmax, ymin, ymax)
        if res:
            u, ix, iy = res
            if 0.0 <= u <= 1.0:
                intersections.append((u, ix, iy))

    # NLN 期望最多两次交点
    if not intersections:
        # 没有交点（可能从内向外直接穿出另一边），把候选扩展为所有边再试一次
        all_bounds = ['left', 'right', 'bottom', 'top']
        for b in all_bounds:
            res = intersect_with_boundary(x0, y0, dx, dy, b, xmin, xmax, ymin, ymax)
            if res:
                u, ix, iy = res
                if 0.0 <= u <= 1.0:
                    intersections.append((u, ix, iy))

    if not intersections:
        return None

    intersections.sort(key=lambda t: t[0])
    if len(intersections) == 1:
        u, ix, iy = intersections[0]
        return (x0, y0, ix, iy)
    else:
        (u0, ix0, iy0), (u1, ix1, iy1) = intersections[0], intersections[1]
        return (ix0, iy0, ix1, iy1)
```
算法关键要点
- NLN 的分支较多，实现要求严格且分支逻辑繁复；因此在工程上常作为研究或在特定高性能场景下采用。
- 在现代 CPU 上复杂分支可能抵消其理论上的低算术量优势；因此仅在对分支开销有可控优化时才优先选用。