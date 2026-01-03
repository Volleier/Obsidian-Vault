## 介绍
多边形裁剪（polygon clipping）是将一个输入多边形（subject polygon）与裁剪多边形（clip polygon）相交，求出输入多边形被裁剪后位于裁剪区域内的部分。常见应用：图形渲染、布尔运算、地图/GIS、几何预处理。

本文按笔记风格给出：算法原理、实现要点、伪代码、常见边界/数值问题及简要 Python 参考实现（非完整脚本，仅核心函数）。

## 基本概念与符号
- 输入多边形顶点按有向顺序记为 $$S=(s_0,s_1,\\dots,s_{n-1})$$，裁剪多边形顶点记为 $$C=(c_0,c_1,\\dots,c_{m-1})$$。
- 顶点顺序（顺时针/逆时针）影响“内部”定义，统一方向是实现前的必要步骤。
- 交点常用参数 $$t\\in[0,1]$$ 表示在边上的位置：若边为 $$P(t)=P_0+t(P_1-P_0)$$，交点对应某个 $$t$$。

---

## 算法一：Sutherland–Hodgman（逐边裁剪，适用于凸裁剪多边形）

### 核心思想
对裁剪多边形的每条边依次裁剪被裁剪多边形：把当前多边形作为输入，基于一条裁剪边（半平面）产生新多边形，重复处理所有裁剪边后得到最终结果。该方法在裁剪多边形为凸时正确且实现简单。

### 半平面判定
给裁剪边由点 $$A$$ 到 $$B$$（有向），定义点 $$P$$ 在边的“内部”当且仅当（取左侧为内部示例）：
$$
\\operatorname{orient}(A,B,P)=(B_x-A_x)(P_y-A_y)-(B_y-A_y)(P_x-A_x)\\ge 0
$$

### 单边处理规则
对当前多边形的每条边 $$(P_i,P_{i+1})$$ 采用四种情况处理：
- 内→内：输出 $$P_{i+1}$$。
- 内→外：输出交点 $$I$$。
- 外→外：不输出。
- 外→内：输出交点 $$I$$，再输出 $$P_{i+1}$$。

交点由两线段求交（参数化或线性方程）得到，需处理平行/共线情形并使用容差避免数值抖动。

### 伪代码
```
function sutherland_hodgman(subject, clipper):
  output = subject
  for each edge (A,B) in clipper:
    input = output
    output = []
    for each consecutive (P_i,P_j) in input:
      if inside(P_i) and inside(P_j): output.append(P_j)
      elif inside(P_i) and not inside(P_j): output.append(intersection(P_i,P_j,A,B))
      elif not inside(P_i) and inside(P_j): output.append(intersection(P_i,P_j,A,B)); output.append(P_j)
    # end for
  return output
```

### 复杂度与适用
- 时间复杂度：若被裁剪多边形顶点数为 $$n$$，裁剪多边形边数为 $$m$$，复杂度为 $$O(mn)$$。
- 适用：裁剪多边形凸（如矩形）时最优；凹裁剪多边形可能产生错误或需要改用更通用算法。

### 工程注意
- 统一顶点顺序；使用 $$\\varepsilon$$ 判定共线/边界；处理顶点重复与微小环。

---

## 算法二：Weiler–Atherton（通用裁剪，可处理凹、多段与孔）

### 核心思想
将被裁剪多边形与裁剪多边形的交点插入对应的顶点链表，标记交点的进入/离开性质，然后按链表在两多边形间切换遍历，提取封闭路径作为输出。适用于输出可能包含多个多边形或孔的情况。

### 主要步骤要点
1. 计算所有边对之间的交点，并按参数位置插入到所属边的顶点链表中。
2. 对每个交点判定 entry/exit（进入或离开裁剪区域）。
3. 从未访问的交点开始，按 entry/exit 规则在两表间切换遍历，形成一条输出回路；重复直到所有交点被访问。

### 数据结构建议
- 使用循环链表（或双向链表）存储顶点，交点以节点形式插入并互相链接（跨链表指针）。
- 每交点记录：位置、参数、已访问标志、entry/exit 标志、对端点引用。

### 工程要点
- 交点插入需按边上参数排序；进入/离开判定在共线或接触情形复杂，需要局部拓扑判断并加容差处理。

---

## 算法三：Greiner–Hormann / 通用布尔运算

与 Weiler–Atherton 思路相近，目标是支持任意两个简单多边形的交/并/差运算。实现包含交点插入、entry/exit 标记、交替遍历输出。重点在于重叠边/顶点相交的特殊处理与排序稳定性。

### 工程建议
- 对复杂生产环境推荐使用成熟库（Clipper、GEOS、CGAL、boost::geometry）。
- 若自行实现，建议：统一坐标精度（整数或固定点）、合并近似重复交点、写大量单元测试（凹、多段、共线、重叠、孔）。

---

## 数值稳定性与通用建议
- 统一顶点方向（外环/内环约定）。
- 使用小量 $$\\varepsilon$$ 判断相等或共线，或将坐标缩放为整数以避免浮点误差。
- 对重叠边做分割处理并合并近似点以去除微小环或退化多边形。

---

## 简要 Python 参考实现（核心函数片段，非完整脚本）

1) Sutherland–Hodgman 核心函数（半平面判断与交点）
```python
def inside(p, A, B):
    # 取 A->B 左侧为内部
    return (B[0]-A[0])*(p[1]-A[1]) - (B[1]-A[1])*(p[0]-A[0]) >= 0

def intersect(P, Q, A, B):
    # 返回 P-Q 与 A-B 的交点（浮点），处理平行情况返回 None
    Px, Py = P; Qx, Qy = Q
    Ax, Ay = A; Bx, By = B
    denom = (Qx-Px)*(By-Ay) - (Qy-Py)*(Bx-Ax)
    if abs(denom) < 1e-12:
        return None
    t = ((Ax-Px)*(By-Ay) - (Ay-Py)*(Bx-Ax)) / denom
    return (Px + t*(Qx-Px), Py + t*(Qy-Py))

def sutherland_hodgman(subject, clipper):
    output = subject[:]
    for i in range(len(clipper)):
        A = clipper[i]
        B = clipper[(i+1)%len(clipper)]
        input_list = output
        output = []
        if not input_list:
            break
        P_prev = input_list[-1]
        for P_curr in input_list:
            in_curr = inside(P_curr, A, B)
            in_prev = inside(P_prev, A, B)
            if in_prev and in_curr:
                output.append(P_curr)
            elif in_prev and not in_curr:
                I = intersect(P_prev, P_curr, A, B)
                if I: output.append(I)
            elif not in_prev and in_curr:
                I = intersect(P_prev, P_curr, A, B)
                if I: output.append(I)
                output.append(P_curr)
            P_prev = P_curr
    return output
```

2) 简化版 Weiler–Atherton 思路骨架（插入交点并按链表遍历的核心示例）
```python
# 伪实现：展示思想，生产环境需完善插入/排序/entry判定
class Node:
    def __init__(self, pt, is_inter=False):
        self.pt = pt
        self.is_inter = is_inter
        self.next = None
        self.other = None  # cross link to other polygon's intersection node
        self.entry = None
        self.visited = False

def build_polygon_linked_list(points):
    nodes = [Node(p) for p in points]
    for i in range(len(nodes)):
        nodes[i].next = nodes[(i+1)%len(nodes)]
    return nodes

# 后续需：
# - 计算并插入交点（在链表中按参数位置插入）
# - 标记 entry/exit（通过点位于裁剪内外判断）
# - 从未访问交点开始 traverse 切换链表输出路径
```

3) Greiner–Hormann/布尔运算提示：
- 实现路径与 Weiler–Atherton 类似，关键在于插入交点时的排序、entry/exit 标记规则（针对交/并/差不同）以及共线/重叠段的分割处理。

---

## 小结
- 若裁剪窗口凸（矩形），优先用 Sutherland–Hodgman；若需处理凹裁剪或产生洞，使用 Weiler–Atherton 或 Greiner–Hormann。工程上优先考虑成熟库以保证鲁棒性。

如需我把参考实现扩展为可运行脚本并附单元测试，我可以继续实现。
