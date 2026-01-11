# Bag-of-Visual-Words（BoVW）模型概述

- 思路借鉴自然语言的 Bag-of-Words：把图像视为“视觉词”的集合，忽略严格空间顺序但统计词频以表征图像。
- 经典流水线：
  1.  关键点检测（或稠密采样）→ 提取局部描述子（SIFT、SURF、ORB 等）。
  2.  码本构建：对训练描述子做聚类（常用 k-means），每个簇中心为一个视觉词（visual word）。
  3.  量化编码：将描述子映射到最近的视觉词（硬分配）或做软分配/概率编码。
  4.  池化/统计：对图像或区域统计词频直方图并归一化（L1/L2）。
  5.  学习与分类：用直方图作为特征训练分类器（常用 SVM）。

## BoW 的变体与注意事项

- 硬量化 vs 软量化：硬量化（硬分配）简单但引入量化误差；软分配、VLAD、Fisher Vector 能捕获更丰富信息并提升表现。
- 码本规模：k 的大小影响表征能力与计算成本，常在数百到数千之间调整。
- 不变性：BoVW 本身对轻微平移/遮挡有鲁棒性，但对空间布局敏感，故有空间扩展（见下）。
- 降维/编码：使用 PCA、PCA-Whitening、Fisher/VLAD 进一步压缩并增强判别性。

## 空间金字塔表示（Spatial Pyramid Matching，SPM）

- 目的：在保留局部统计信息的同时引入弱空间布局，从而提升分类性能。
- 方法：对图像按金字塔分层（level 0: 整图；level 1: 2x2；level 2: 4x4 等），在每个单元格上计算 BoW 直方图并按层次拼接这些直方图成为最终特征。通常对不同层给予加权（低层更高权重）。
- 优点：在大多数场景中显著提升对背景干扰与局部排列不变性的区分能力。

## 简短实现示例（构建 BoW + SVM）

```python
import cv2
import numpy as np
from sklearn.cluster import KMeans
from sklearn.svm import LinearSVC

# 1) 提取 SIFT 描述子（示例）
sift = cv2.SIFT_create()
descriptors = []
for img in train_images:
		kps, desc = sift.detectAndCompute(img, None)
		if desc is not None:
				descriptors.append(desc)
all_desc = np.vstack(descriptors)

# 2) 构建码本
k = 1000
kmeans = KMeans(n_clusters=k).fit(all_desc)

# 3) 词频直方图编码
def bow_hist(desc, kmeans):
		words = kmeans.predict(desc)
		hist, _ = np.histogram(words, bins=np.arange(k+1))
		return hist.astype(float) / (np.linalg.norm(hist) + 1e-8)

# 4) 对训练集编码并训练 SVM
X = [bow_hist(d, kmeans) for d in descriptors]
y = train_labels
clf = LinearSVC().fit(X, y)
```
