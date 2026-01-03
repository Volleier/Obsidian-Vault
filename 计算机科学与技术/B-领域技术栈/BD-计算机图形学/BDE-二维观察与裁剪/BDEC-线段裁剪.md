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

### 算法流程：
 1. 对端点 $P_0(x_0,y_0)$ 和 $P_1(x_1,y_1)$ 计算编码 $code_0$ 与 $code_1$。
 2. 若 $(code_0\,|\,code_1) = 0$，两个端点均在窗口内：接受整段线段。
 3. 若 $(code_0\,\&\,code_1) \ne 0$，两端点在同一窗口外区域的同一侧：拒绝整段线段（不可见）。
 4. 否则，选择编码非零的端点（位于窗口外），按其编码的某一位（例如 L、R、T、B）计算与对应窗口边界的交点：
	 - 若选择左边界：与 $x = x_{min}$ 求交，可由参数方程或直线方程求得交点坐标。
	 - 同理处理右/上/下边界。
 5. 用交点替换原来的外部端点，重新计算其编码，回到第2步循环，直到接受或拒绝。

- 关键实现
    - 采用整数位运算实现编码测试，速度快且分支少。
    - 选择计算交点时，最好使用浮点运算得出精确交点坐标，然后将其赋回端点（保持坐标为浮点以便后续计算）。
    - 对垂直或水平线求交时注意避免除以零（可用不同的求交公式或分支处理）。
    - 对多段折线裁剪时注意端点重复：通常对每段都包含起点但在连接处跳过重复终点。

- 伪代码（高层）：

```
function cohen_sutherland_clip(P0, P1, rect):
    code0 = compute_code(P0, rect)
    code1 = compute_code(P1, rect)
    while True:
        if (code0 | code1) == 0:
            return accept entire segment
        elif (code0 & code1) != 0:
            return reject
        else:
            out_code = code0 != 0 ? code0 : code1
            compute intersection of segment with boundary indicated by out_code
            replace the outside endpoint with intersection and recompute its code
```

- 复杂度：最坏情况下需要多次迭代裁剪（但通常很快），平均开销小，容易实现。

## Liang–Barsky 算法（参数化、不等式法）

- 基本思想：把线段用参数 $u\in[0,1]$ 表示，利用裁剪窗口的四个不等式直接求出参数区间 $[u_1,u_2]$，无需循环地逐步裁剪端点，边界交点参数只需计算一次。

- 参数化表示：

$
x(u)=x_0+u\Delta x,\qquad y(u)=y_0+u\Delta y,\qquad \Delta x=x_1-x_0,\ \Delta y=y_1-y_0
$

- 窗口内条件等价为四个不等式：

$
x_{min}\le x_0+u\Delta x\le x_{max},\qquad y_{min}\le y_0+u\Delta y\le y_{max}
$

把每个不等式整理为统一形式：$u\,p_k\le q_k,\quad k=1..4$，其中通常取

$
p_1=-\Delta x,\ q_1=x_0-x_{min}\\
p_2=\Delta x,\ q_2=x_{max}-x_0\\
p_3=-\Delta y,\ q_3=y_0-y_{min}\\
p_4=\Delta y,\ q_4=y_{max}-y_0
$

- 算法核心（实现细节）：

 1. 初始化：$u_1=0,\ u_2=1$。
 2. 对每个边界 $k=1..4$：

        - 若 $p_k==0$：线段与该边界平行。若 $q_k<0$，则线段在该边界外（不可见），直接拒绝。否则不影响 $[u_1,u_2]$。
        - 否则计算交点参数 $u = q_k / p_k$。
            - 若 $p_k < 0$（“进入”边），更新 $u_1 = \max(u_1, u)$。
            - 若 $p_k > 0$（“离开”边），更新 $u_2 = \min(u_2, u)$。

 3. 若在任一步出现 $u_1>u_2$，则线段完全不可见；否则可见子段为 $u\in[u_1,u_2]$，对应端点为 $P(u_1),\ P(u_2)$。

- 数值注意事项与实现提示：

    - 使用浮点计算 $u$，但对精度敏感时可考虑相对容差（如比较时加入小量 eps）。
    - 若仅需判断可见性，可以在计算出任一边界的拒绝条件时返回，避免额外计算。
    - Liang–Barsky 在每个边界仅做一次算术运算，理论上比 Cohen-Sutherland 少重复求交，效率更高。

- 伪代码（高层）：

```
function liang_barsky_clip(P0, P1, rect):
    dx = x1 - x0; dy = y1 - y0
    u1 = 0; u2 = 1
    for k in 1..4:
        compute p_k, q_k
        if p_k == 0 and q_k < 0: return reject
        if p_k != 0:
            u = q_k / p_k
            if p_k < 0: u1 = max(u1, u)
            else:       u2 = min(u2, u)
        if u1 > u2: return reject
    return P(u1), P(u2)
```

## Nicholl–Lee–Nicholl (NLN) 算法（区域细分，最小求交次数）

- 基本思想：以第一个端点 $P_0$ 为参考，将平面按扇区细分（通常分为 8 个扇形区域），根据第二个端点 $P_1$ 所在扇区直接决定需要与哪些边界求交，从而在多数情况下只需最多两次求交就能完成裁剪。

- 为什么有效：相比 Cohen-Sutherland 可能多次剪掉外部小段，NLN 通过预先分类避免冗余的重复求交，特别在许多常见几何配置下能减少运算量。

- 实现重点与流程提示：

    - 首先对 $P_0$ 做快速编码判断（是否在窗口内）；若 $P_0$ 在外部，可先把问题转换为其在内部的等价情况或直接处理对称情形。
    - 以 $P_0$ 为中心，将窗口用相对角线与边界划分为若干区域（例如左上、上、右上、右、右下、下、左下、左）；根据 $P_1$ 的所属区域选择对应的裁剪路径。
    - 每种扇区对应一棵小的决策树（流程）——直接给出至多两次需要求交的边界，求出交点，组合为裁剪后线段。

- 实现复杂度：

    - NLN 的分支较多，实现要求严格且分支逻辑繁复；因此在工程上常作为研究或在特定高性能场景下采用。
    - 在现代 CPU 上复杂分支可能抵消其理论上的低算术量优势；因此仅在对分支开销有可控优化时才优先选用。

- 伪代码（说明性）：

```
function nln_clip(P0, P1, rect):
    if P0 inside rect:
        determine sector of P1 relative to P0
        follow sector-specific case to compute up to two intersections
        return clipped segment (if any)
    else:
        // 可以先把 P0,P1 交换，使 P0 在内部，或按对称处理
        handle symmetric cases
```

## 总结与工程建议

- 选择算法依据：

    - 若实现简单、可读性优先：使用 Cohen–Sutherland，容易调试。
    - 若追求效率且窗口为轴对齐矩形：优先使用 Liang–Barsky（参数化，计算量少）。
    - 若在特定二维场景下需要最少的交点计算并能接受复杂实现：可考虑 NLN。

- 额外注意：处理浮点精度、端点包含策略（是否包含终点以避免多段连线重复绘制）以及垂直/水平退化情形都是工程中必须统一的细节。

如果你希望，我可以：

- 1) 在每个算法后附上精确的可直接复制粘贴的 C/Python 参考实现；
- 2) 为 `Liang–Barsky` 添加数值稳定性（EPS）处理示例；
- 3) 为 `Cohen–Sutherland` 添加位编码的示例实现。

告诉我你希望我继续哪一项，我会继续补上代码示例。
