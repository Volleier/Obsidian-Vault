# 介绍

### 1.1 相机的基本功能
- **观察变换**：将3D世界坐标系中的物体转换到相机坐标系
- **投影变换**：将3D相机坐标系中的点投影到2D成像平面
- **裁剪**：移除视锥体外的几何体
- **视口变换**：将投影后的2D坐标映射到屏幕像素坐标

### 1.2 相机的基本参数
- **位置（Eye Point）**：相机在世界坐标系中的位置
- **观察方向（Look-at Direction）**：相机指向的方向
- **向上向量（Up Vector）**：定义相机的"向上"方向
- **视场角（Field of View, FOV）**：决定可见范围的角度
- **宽高比（Aspect Ratio）**：投影平面的宽度与高度之比
- **近/远剪裁平面（Near/Far Clipping Planes）**：定义可见深度范围

## 二、视图变换（View Transformation）

### 2.1 目标
将世界坐标系转换为相机坐标系（观察坐标系）

### 2.2 相机坐标系定义
- **原点**：相机位置
- **z轴**：观察方向的反方向（通常指向屏幕内）
- **x轴**：与z轴和向上向量垂直，指向右侧
- **y轴**：与x轴和z轴垂直，指向上方

### 2.3 视图矩阵推导
设：
- **e** = 相机位置（世界坐标）
- **g** = 观察方向向量
- **t** = 向上向量（世界坐标）

计算：
1. 相机z轴：**w** = -g/‖g‖（指向观察方向反方向）
2. 相机x轴：**u** = (t × w)/‖t × w‖
3. 相机y轴：**v** = w × u

视图矩阵 M_view = R · T
- T = 平移矩阵，将相机移到原点
- R = 旋转矩阵，将相机坐标系对齐到世界坐标系

$$
M_{view} = \begin{bmatrix}
u_x & u_y & u_z & -\mathbf{u}·\mathbf{e} \\
v_x & v_y & v_z & -\mathbf{v}·\mathbf{e} \\
w_x & w_y & w_z & -\mathbf{w}·\mathbf{e} \\
0 & 0 & 0 & 1
\end{bmatrix}
$$

## 三、投影变换（Projection Transformation）

### 3.1 投影类型
#### 3.1.1 透视投影（Perspective Projection）
- 模拟人眼视觉，有"近大远小"效果
- 投影线汇聚于一点（视点）

#### 3.1.2 正交投影（Orthographic Projection）
- 平行投影线，保持物体大小不变
- 无透视变形

### 3.2 透视投影详解

#### 3.2.1 视锥体（Frustum）
- **平截头体**：由近剪裁平面和远剪裁平面定义
- 参数：
  - fov：垂直视场角
  - aspect：宽高比（width/height）
  - n：近剪裁平面距离
  - f：远剪裁平面距离

#### 3.2.2 透视投影矩阵推导
假设：
- 视锥体对称：left = -right, bottom = -top
- 近剪裁平面：z = -n（相机坐标系下）

计算投影矩阵：
1. 将视锥体变换为规范视锥体（Canonical View Volume）
2. 规范视锥体：x,y,z ∈ [-1, 1]

$$
M_{persp} = \begin{bmatrix}
\frac{n}{r} & 0 & 0 & 0 \\
0 & \frac{n}{t} & 0 & 0 \\
0 & 0 & \frac{f+n}{n-f} & \frac{2fn}{f-n} \\
0 & 0 & -1 & 0
\end{bmatrix}
$$

其中：
- r = n·tan(θ/2)·aspect
- t = n·tan(θ/2)
- θ = 垂直视场角（fov）

常用简化形式（基于fov和aspect）：
$$
M_{persp} = \begin{bmatrix}
\frac{1}{\text{aspect}·\tan(\frac{fov}{2})} & 0 & 0 & 0 \\
0 & \frac{1}{\tan(\frac{fov}{2})} & 0 & 0 \\
0 & 0 & \frac{f+n}{n-f} & \frac{2fn}{f-n} \\
0 & 0 & -1 & 0
\end{bmatrix}
$$

#### 3.2.3 透视除法（Perspective Divide）
- 投影变换后需要进行齐次坐标的w分量除法
- (x, y, z, w) → (x/w, y/w, z/w, 1)

### 3.3 正交投影详解
$$
M_{ortho} = \begin{bmatrix}
\frac{2}{r-l} & 0 & 0 & -\frac{r+l}{r-l} \\
0 & \frac{2}{t-b} & 0 & -\frac{t+b}{t-b} \\
0 & 0 & \frac{2}{n-f} & -\frac{n+f}{n-f} \\
0 & 0 & 0 & 1
\end{bmatrix}
$$

对称情况（l = -r, b = -t）：
$$
M_{ortho} = \begin{bmatrix}
\frac{1}{r} & 0 & 0 & 0 \\
0 & \frac{1}{t} & 0 & 0 \\
0 & 0 & \frac{2}{f-n} & -\frac{f+n}{f-n} \\
0 & 0 & 0 & 1
\end{bmatrix}
$$

## 四、完整的图形管线变换流程

### 4.1 变换序列
模型坐标 → 世界坐标 → 相机坐标 → 裁剪坐标 → 标准化设备坐标 → 屏幕坐标

### 4.2 矩阵表示
1. **模型矩阵（Model Matrix）**：M_model，物体局部坐标→世界坐标
2. **视图矩阵（View Matrix）**：M_view，世界坐标→相机坐标
3. **投影矩阵（Projection Matrix）**：M_proj，相机坐标→裁剪坐标
4. **透视除法**：裁剪坐标→标准化设备坐标（NDC）
5. **视口矩阵（Viewport Matrix）**：NDC→屏幕坐标

## 五、深度缓存与深度值

### 5.1 深度值计算
- 投影变换后的z值需要映射到深度缓存范围（通常[0,1]）
- 透视投影的深度值非线性分布：靠近近平面的精度高，远平面的精度低

### 5.2 深度冲突（Z-fighting）
- 原因：深度缓冲精度不足导致两个表面交替显示
- 解决方法：
  - 增大近剪裁平面距离
  - 减小远剪裁平面距离
  - 使用更高精度的深度缓冲

## 六、高级相机模型

### 6.1 相机移动控制
- **轨道相机（Orbit Camera）**：围绕目标点旋转
- **第一人称相机（FPS Camera）**：自由移动和视角控制
- **跟随相机（Follow Camera）**：跟随目标物体移动

### 6.2 透视投影变体
- **非对称视锥体**：用于立体渲染、阴影映射等
- **斜投影（Oblique Projection）**：投影方向不垂直于投影平面

### 6.3 相机镜头效果
- **景深（Depth of Field）**：模拟真实相机焦点效果
- **镜头畸变**：桶形畸变、枕形畸变
- **运动模糊**：模拟相机或物体运动的效果

## 七、常见问题与注意事项

### 7.1 坐标系约定
- 左手系 vs 右手系
- Y轴向上 vs Z轴向上
- 需要保持所有变换使用同一坐标系

### 7.2 数值稳定性
- 避免近剪裁平面距离过小（接近0）
- 避免远剪裁平面距离过大
- 保持宽高比与实际显示窗口一致

### 7.3 性能优化
- 尽早进行视锥体裁剪
- 使用层次结构加速可见性判断
- 根据场景需要选择合适的投影类型

## 八、数学公式总结

### 8.1 视图矩阵
$$
\mathbf{w} = -\frac{\mathbf{g}}{\|\mathbf{g}\|}, \quad
\mathbf{u} = \frac{\mathbf{t} \times \mathbf{w}}{\|\mathbf{t} \times \mathbf{w}\|}, \quad
\mathbf{v} = \mathbf{w} \times \mathbf{u}
$$

$$
M_{view} = \begin{bmatrix}
u_x & u_y & u_z & -\mathbf{u}·\mathbf{e} \\
v_x & v_y & v_z & -\mathbf{v}·\mathbf{e} \\
w_x & w_y & w_z & -\mathbf{w}·\mathbf{e} \\
0 & 0 & 0 & 1
\end{bmatrix}
$$

### 8.2 透视投影矩阵
$$
M_{persp} = \begin{bmatrix}
\frac{\cot(\frac{fov}{2})}{aspect} & 0 & 0 & 0 \\
0 & \cot(\frac{fov}{2}) & 0 & 0 \\
0 & 0 & \frac{f+n}{n-f} & \frac{2fn}{f-n} \\
0 & 0 & -1 & 0
\end{bmatrix}
$$

### 8.3 变换链
$$
\mathbf{P}_{screen} = M_{viewport} \cdot (M_{proj} \cdot M_{view} \cdot M_{model} \cdot \mathbf{P}_{local})
$$