# 简介
Nanite是虚幻引擎（Unreal Engine）中的一项革命性技术，专为处理虚拟几何体而设计。它允许开发者在实时渲染中使用极其详细和复杂的几何模型，而无需担心传统的多边形限制（可处理高达上亿面的模型）。通过Nanite，开发者可以在游戏和虚拟现实体验中实现高水平的细节和视觉保真度，同时保持高效的性能和流畅的用户体验。

Nanite利用一种被称为“虚拟化几何体”的技术，它将几何数据存储在一种新的数据格式中，并在运行时动态加载和卸载这些数据。Nanite能够根据视角、距离和屏幕分辨率等因素实时调整几何体的细节层次（Level of Detail, LOD），确保在任何场景下都能呈现最适合的细节水平。这种技术使得开发者可以在虚幻引擎中使用电影级别的资产，而不必担心性能瓶颈。

Nanite的核心特点之一是其自动化的LOD管理。传统上，开发者需要手动创建和管理不同的LOD模型，以优化性能并减少资源消耗。然而，Nanite通过自动生成和管理这些LOD，通过细粒度的几何细节裁剪和动态细节调整，实现了前所未有的渲染效率，大大简化了工作流程，使开发者能够专注于创作而不是技术细节。

Nanite的另一大优势是其对现代硬件的优化支持。利用现代GPU的强大计算能力，Nanite能够实现极高的渲染性能，即使在使用超高分辨率纹理和复杂几何体的情况下，依然能够保持流畅的帧率。这种性能优化使得Nanite特别适用于高端游戏和虚拟现实应用，能够充分发挥现代硬件的潜力。

# 包含技术
[[Nanite/GPU Driven Render Pipeline|GPU Driven Render Pipeline]]
[[Nanite/Virtualized Geometry Nanite|Virtualized Geometry Nanite]]
[[Nanite/Visibility Buffer|Visibility Buffer]]

# [**Nanite文档**](https://dev.epicgames.com/documentation/en-us/unreal-engine/nanite-virtualized-geometry-in-unreal-engine?application_version=5.4)