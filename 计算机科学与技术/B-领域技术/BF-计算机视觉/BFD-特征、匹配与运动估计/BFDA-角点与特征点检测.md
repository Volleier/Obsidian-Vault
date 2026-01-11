# 角点与特征点检测（Corners & Keypoints）

角点（corner）是图像上在两个或多个方向上都有显著强度变化的点，具有**可定位性**（定位精确）、**区分性**（局部描述子可区分）和**重复性**（在不同视角/光照下可被重检测）的特点，因此是许多视觉任务的首选关键点来源：特征匹配、图像配准/拼接、光流/跟踪（KLT）、SLAM 与三维重建等。

---

## 1. Harris 角点检测器：数学推导与直观

### 1.1 基本思想

Harris 检测关注小窗口在平移时图像强度的变化量；若对任意小位移都能导致显著强度变化，则该窗口中心是角点。

设图像灰度为 \(I(x,y)\)，令窗口加权由权重函数 \(w(u,v)\)（如高斯）给出，考虑像素在位移 \((u,v)\) 下的平方强度差：

$$
E(u,v)=\sum_{x,y} w(x,y)\big[I(x+u,y+v)-I(x,y)\big]^2.
$$

对小位移用泰勒展开，忽略高阶项：

$$
I(x+u,y+v)\approx I(x,y) + I_x u + I_y v
$$

代入得：

$$
E(u,v) \approx \begin{bmatrix}u & v\end{bmatrix} M \begin{bmatrix}u\\ v\end{bmatrix},
$$

其中二阶矩（结构张量）为：

$$
M=\sum_{x,y} w(x,y) \begin{bmatrix} I_x^2 & I_x I_y\\ I_x I_y & I_y^2 \end{bmatrix}.
$$

### 1.2 特征值视角与响应函数

M 的两个特征值 \(\lambda_1,\lambda_2\) 描述了窗口在两个主方向上的强度变化：

- 若两个特征值都大 → 角点；
- 若一个大一个小 → 边缘；
- 若都小 → 平坦区域。

为了避免显式求特征值，Harris 提出如下响应函数：

$$
R = \det(M) - k\cdot\operatorname{trace}(M)^2 = \lambda_1\lambda_2 - k(\lambda_1+\lambda_2)^2,
$$

其中常用经验值 \(k\in[0.04,0.06]\)。当 \(R\) 足够大且为局部最大时，判为角点。

直观上：det 大且 trace 适中表示两个特征值都大；减去 \(k\cdot\)trace^2 抑制边缘响应。

---

## 2. Harris 的实现步骤（工程流程）

1. 计算图像梯度：用 Sobel / 高斯导数 得到 \(I_x, I_y\)。
2. 计算矩阵元素：\(I_x^2, I_y^2, I_x I_y\)。
3. 用窗口（常用高斯）对上述量进行平滑（求和）：得到 \(S*{xx}, S*{yy}, S\_{xy}\)。
4. 计算响应：\(R = S*{xx}S*{yy}-S*{xy}^2 - k(S*{xx}+S\_{yy})^2\)。
5. 阈值化：保留 \(R\) 大于阈值的点（阈值可为最大响应的比例或经验值）。
6. 非极大值抑制（NMS）：在局部邻域内取极大值点，去除邻近重复响应。
7. （可选）亚像素精修：使用灰度质心、拟合二次曲面或 `cv2.cornerSubPix` 进行精确定位。

参数选择提示：窗口（或高斯 σ）决定尺度；梯度算子与平滑核要配合；阈值与 k 对检测结果影响显著。

---

## 3. 角点响应的物理意义与分类准则

- **物理意义**：结构张量的两个特征值分别衡量窗口沿主方向的强度变化；角点对应两个方向皆大，意味着局部有可靠可区分信息。
- **分类准则（常用）**：
  - 角点：\(\lambda_1\) 与 \(\lambda_2\) 均大（或等价地 \(R\) 大且为局部极大）。
  - 边缘：一大一小（\(\det(M)\) 小但 \(\operatorname{trace}(M)\) 较大，Harris 的 R 值会较小或负）。
  - 平坦：两特征值都小（R 小）。

实用替代：Shi–Tomasi 提出直接用 \(\min(\lambda_1,\lambda_2)\) 作为评分（`goodFeaturesToTrack`），在跟踪任务中通常优于 Harris 的经验 R，因为它直接反映最小主方向能量。

---

## 4. 变体与工程技巧

- **Shi–Tomasi（Good Features to Track）**：评分函数为 \(\min(\lambda_1,\lambda_2)\)，阈值直观且在 KLT 跟踪中效果好。
- **尺度不变 Harris / 多尺度 Harris**：在不同尺度（不同高斯 σ）下计算响应并做尺度选择，以应对图像尺度变化。
- **FAST / AGAST / ORB 的角点**：基于像素强度比较（圆周邻域）实现极快检测，常用于实时系统；但这些方法与 Harris 在稳健性/重复性上有权衡。
- **亚像素精修**：`cornerSubPix` 用迭代最小化窗口内灰度差的方式优化角点位置，显著提高后续匹配精度。
- **预处理**：去噪（或适当平滑）与对比度归一化可提高角点稳定性；在低纹理区域可适当放宽阈值。

---

## 5. 角点检测在视觉任务中的应用

- **光流与跟踪（KLT）**：KLT 使用 Shi–Tomasi 选择角点并基于金字塔光流跟踪，广泛用于视频跟踪与视觉里程计。
- **特征匹配与拼接**：作为关键点检测器与 SIFT/SURF/ORB 描述子配合用于图像配准与全景拼接。
- **三维重建 / SfM / SLAM**：角点作为高质量特征帮助建立匹配与重投影约束。
- **目标定位与检测前处理**：在检测/识别算法中作为兴趣点抽取步骤。

---

## 6. 简短代码示例（Python + OpenCV）

```python
import cv2
import numpy as np

# 读取灰度图
img = cv2.imread('image.jpg', cv2.IMREAD_GRAYSCALE)

# Harris
dst = cv2.cornerHarris(img, blockSize=2, ksize=3, k=0.04)
# 扩展标记并阈值
dst_dilated = cv2.dilate(dst, None)
img_color = cv2.cvtColor(img, cv2.COLOR_GRAY2BGR)
img_color[dst > 0.01 * dst.max()] = [0,0,255]

# Shi-Tomasi（goodFeaturesToTrack）
corners = cv2.goodFeaturesToTrack(img, maxCorners=200, qualityLevel=0.01, minDistance=10)
corners = np.int0(corners)
for c in corners:
		x,y = c.ravel()
		cv2.circle(img_color, (x,y), 3, (0,255,0), -1)

# 亚像素精修示例
criteria = (cv2.TERM_CRITERIA_EPS + cv2.TERM_CRITERIA_MAX_ITER, 30, 0.01)
corners_float = np.float32(corners).reshape(-1,1,2)
cv2.cornerSubPix(img, corners_float, (5,5), (-1,-1), criteria)

cv2.imwrite('corners_out.png', img_color)
```

---

## 7. 小结与工程建议

- 优先使用 Shi–Tomasi（`goodFeaturesToTrack`）作为跟踪与匹配的角点选择器；在需要理论分析/可解释性时使用 Harris 的 R。
- 对尺度/光照变化敏感的应用采用多尺度策略与归一化预处理。
- 在实时系统中，可用 FAST/ORB 作为替代并结合描述子与筛选策略。

如需，我可以把此章节在 `BFDB-局部特征描述子.md`、`BFDC-特征匹配与最近邻搜索.md`、`BFDF-光流（传统方法）.md` 中添加简短导读链接，或把示例扩展为可运行的小 demo。
