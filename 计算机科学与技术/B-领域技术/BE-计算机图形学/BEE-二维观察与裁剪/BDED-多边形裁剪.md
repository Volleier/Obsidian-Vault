# 介绍
多边形裁剪（polygon clipping）是将一个输入多边形（subject polygon）与裁剪多边形（clip polygon）相交，求出输入多边形被裁剪后位于裁剪区域内的部分。常应用于图形渲染、布尔运算、地图/GIS、几何预处理。

## 基本概念与符号
- 输入多边形顶点按有向顺序记为 $S=(s_0,s_1,\dots,s_{n-1})$，裁剪多边形顶点记为 $C=(c_0,c_1,\dots,c_{m-1})$。
- 顶点顺序（顺时针/逆时针）影响“内部”定义，统一方向是实现前的必要步骤。
- 交点常用参数 $t\in[0,1]$ 表示在边上的位置：若边为 $P(t)=P_0+t(P_1-P_0)$，交点对应某个 $t$

# 多边形裁剪方法
## Sutherland–Hodgman（逐边裁剪，适用于凸裁剪多边形）

### 核心思想
对裁剪多边形的每条边依次裁剪被裁剪多边形：把当前多边形作为输入，基于一条裁剪边（半平面）产生新多边形，重复处理所有裁剪边后得到最终结果。该方法在裁剪多边形为凸时正确且实现简单。

#### 半平面判定
给裁剪边由点 $A$ 到 $B$（有向），定义点 $P$ 在边的“内部”当且仅当（取左侧为内部示例）：
$$
\operatorname{orient}(A,B,P)=(B_x-A_x)(P_y-A_y)-(B_y-A_y)(P_x-A_x)\ge 0
$$

#### 单边处理规则
对当前多边形的每条边 $(P_i,P_{i+1})$ 采用四种情况处理：
- 内→内：输出 $P_{i+1}$。
- 内→外：输出交点 $I$。
- 外→外：不输出。
- 外→内：输出交点 $I$，再输出 $P_{i+1}$。
交点由两线段求交（参数化或线性方程）得到，需处理平行/共线情形并使用容差避免数值抖动。

### 实现
#### 算法实现
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
- 算法要点：
	- 时间复杂度：若被裁剪多边形顶点数为 $n$，裁剪多边形边数为 $m$，复杂度为 $O(mn)$。
	- 适用：裁剪多边形凸（如矩形）时最优；凹裁剪多边形可能产生错误或需要改用更通用算法。

## Weiler–Atherton（通用裁剪，可处理凹、多段与孔）

### 核心思想
将被裁剪多边形与裁剪多边形的交点插入对应的顶点链表，标记交点的进入/离开性质，然后按链表在两多边形间切换遍历，提取封闭路径作为输出。适用于输出可能包含多个多边形或孔的情况。

### 实现
#### 实现要点
1. 计算所有边对之间的交点，并按参数位置插入到所属边的顶点链表中。
2. 对每个交点判定 entry/exit（进入或离开裁剪区域）。
3. 从未访问的交点开始，按 entry/exit 规则在两表间切换遍历，形成一条输出回路；重复直到所有交点被访问。

#### 算法实现
``` python
class Node:
    def __init__(self, pt, is_inter=False, alpha=0.0):
        self.pt = pt            # (x,y)
        self.is_inter = is_inter
        self.alpha = alpha      # 在所属边上的参数位置
        self.next = None
        self.prev = None
        self.other = None       # 跨多边形的交点配对
        self.entry = None       # True: entry, False: exit
        self.visited = False

def make_ring(points):
    nodes = [Node(p) for p in points]
    n = len(nodes)
    for i in range(n):
        nodes[i].next = nodes[(i+1) % n]
        nodes[(i+1) % n].prev = nodes[i]
    return nodes

def seg_intersection(p0, p1, q0, q1, eps=1e-12):
    # 返回 (t,u,(ix,iy)) 或 None，其中 t 在 p 段上，u 在 q 段上
    (x1,y1),(x2,y2) = p0,p1
    (x3,y3),(x4,y4) = q0,q1
    denom = (x2-x1)*(y4-y3) - (y2-y1)*(x4-x3)
    if abs(denom) < eps:
        return None
    t = ((x3-x1)*(y4-y3) - (y3-y1)*(x4-x3)) / denom
    u = ((x3-x1)*(y2-y1) - (y3-y1)*(x2-x1)) / denom
    if 0.0 <= t <= 1.0 and 0.0 <= u <= 1.0:
        ix = x1 + t*(x2-x1)
        iy = y1 + t*(y2-y1)
        return (t, u, (ix, iy))
    return None

def insert_intersections(subject_nodes, clip_nodes):
    # 对每条边对做相交测试，插入交点节点（按 alpha 排序）并建立 cross-link
    for snode in subject_nodes:
        s0 = snode.pt
        s1 = snode.next.pt
        # 遍历裁剪边
        for cnode in clip_nodes:
            c0 = cnode.pt
            c1 = cnode.next.pt
            res = seg_intersection(s0, s1, c0, c1)
            if res:
                t, u, (ix,iy) = res
                # 在 subject 边上插入节点
                s_inter = Node((ix,iy), is_inter=True, alpha=t)
                # 在链表中插入 s_inter 在 snode -> snode.next 之间（此处为简化插入）
                s_inter.next = snode.next
                snode.next.prev = s_inter
                snode.next = s_inter
                s_inter.prev = snode

                # 在 clip 边上插入节点
                c_inter = Node((ix,iy), is_inter=True, alpha=u)
                c_inter.next = cnode.next
                cnode.next.prev = c_inter
                cnode.next = c_inter
                c_inter.prev = cnode

                # cross-link
                s_inter.other = c_inter
                c_inter.other = s_inter

    # 返回更新后的节点表（可由原始点链表扫描得到）
    return

def point_in_poly(pt, poly):
    # 简单射线法判断点内（用于 entry/exit 判定）
    x,y = pt
    inside = False
    n = len(poly)
    j = n-1
    for i in range(n):
        xi, yi = poly[i]; xj,yj = poly[j]
        if ((yi>y) != (yj>y)) and (x < (xj-xi)*(y-yi)/(yj-yi+1e-16) + xi):
            inside = not inside
        j = i
    return inside

def mark_entry_exit(subject_nodes, clip_polygon):
    for node in subject_nodes:
        if node.is_inter:
            # 取交点前的一个微小采样点，判定是否在剪切多边形内
            prev = node.prev
            sample = ((node.pt[0]*0.999 + prev.pt[0]*0.001), (node.pt[1]*0.999 + prev.pt[1]*0.001))
            node.entry = not point_in_poly(sample, clip_polygon)

def traverse(subject_nodes):
    results = []
    for node in subject_nodes:
        if not node.is_inter or node.visited:
            continue
        poly = []
        cur = node
        while True:
            if cur.visited:
                break
            cur.visited = True
            poly.append(cur.pt)
            # 当遇到交点时，跳到对应多边形并继续
            if cur.is_inter:
                cur = cur.other
                cur.visited = True
                cur = cur.next
            else:
                cur = cur.next
            if cur == node:
                break
        if poly:
            results.append(poly)
    return results
```
算法核心要点
- 使用循环链表（或双向链表）存储顶点，交点以节点形式插入并互相链接（跨链表指针）。
- 每交点记录：位置、参数、已访问标志、entry/exit 标志、对端点引用。

## Greiner–Hormann / 通用布尔运算
与 Weiler–Atherton 思路相近，目标是支持任意两个简单多边形的交/并/差运算。实现包含交点插入、entry/exit 标记、交替遍历输出。重点在于重叠边/顶点相交的特殊处理与排序稳定性。

### 实现
#### 实现要点
1. 规范输入：统一顶点方向，去掉重复或几乎重合的顶点，必要时把浮点坐标 scale→整数
2. 建环：把 subject 与 clipper 各自转为双向循环链表，节点含：坐标、is_inter、alpha、neighbor、visited、type（entry/exit）。
3. 计算所有边对的相交点：遍历 subject 的每条边与 clipper 的每条边，使用线段相交公式求参数 t（在 subject 边上）和 u（在 clip 边上）；若 0≤t≤1 且 0≤u≤1，则计算交点坐标。
4. 在链表中插入交点节点并建立配对：在对应边的链表上按 alpha插入交点节点；为两个交点互设 neighbor/other 指针。
5. 合并与处理重叠/共线：合并数值上接近的交点；若检测到边重叠，把重叠段分割为端点与交点以便后续决策。
6. 标记 entry / exit：对每个交点，在其所属边上取交点前的一个微小采样点，判断该采样点是否在另一个多边形内部，从而确定交点是“进入”还是“离开”。不同的布尔操作会改变入/出判定规则。
7. 遍历输出多边形环：从任一未访问的交点开始：沿当前多边形向前遍历并记录顶点，遇到交点则切换到 neighbor 节点所在的另一个多边形并继续；直到回到起点，得一输出环。重复直到所有交点被访问。
8. 后处理与校验：删除面积或顶点数过小的退化环；合并共线顶点，规范环的朝向（外环/内环约定）；校验无自交、连通性正确
#### 算法实现
``` python 
class GHNode(Node):
    def __init__(self, pt, is_inter=False, alpha=0.0):
        super().__init__(pt, is_inter, alpha)
        self.neighbor = None  # 对应交点
        self.type = None      # 'entry' 或 'exit'

def build_intersections(subject, clipper):
    # 查找交点并插入（与前面类似），返回所有交点节点列表
    intersections = []
    # ... 使用 seg_intersection 插入交点并设置 neighbor 链接
    return intersections

def classify_intersections(intersections, subject, clipper):
    # 根据操作类型（例如交集），为每个交点标记 entry/exit
    for node in intersections:
        sample = ((node.pt[0]*0.999 + node.prev.pt[0]*0.001), (node.pt[1]*0.999 + node.prev.pt[1]*0.001))
        node.type = 'entry' if not point_in_poly(sample, clipper) else 'exit'

def gh_traverse(intersections):
    results = []
    for inter in intersections:
        if inter.visited:
            continue
        poly = []
        cur = inter
        while True:
            cur.visited = True
            # walk along current polygon until next intersection
            node = cur
            while not node.is_inter:
                poly.append(node.pt)
                node = node.next
            # at intersection, follow neighbor to other polygon
            cur = node.neighbor
            if cur.visited:
                break
        results.append(poly)
    return results

```
算法核心要点
- 统一顶点方向（外环/内环约定）。
- 使用小量 $\varepsilon$ 判断相等或共线，或将坐标缩放为整数以避免浮点误差。
- 对重叠边做分割处理并合并近似点以去除微小环或退化多边形。