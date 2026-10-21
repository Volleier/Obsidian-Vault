**FABRIK（Forward And Backward Reaching Inverse Kinematics，前后向到达反向运动学）是一种在位置空间里迭代求解 IK 的算法：每次迭代先把末端拉到目标、沿链往回拖动各关节点，再把根拉回原位、沿链往前拖回去，每一步只保持骨骼长度不变。它不涉及角度和矩阵，收敛快、姿势自然。UE 里对应 AnimGraph 的 FABRIK 节点（`FAnimNode_Fabrik`）和 Control Rig 的 FABRIK 节点。**

> 出处：Andreas Aristidou, Joan Lasenby, *FABRIK: A fast, iterative solver for the Inverse Kinematics problem*, Graphical Models 73(5), 2011, pp. 243–260；此前有 2009 年剑桥大学的技术报告。原笔记把作者写成 Lindenmayer 和 Habel，那是 L-system 的作者，与 FABRIK 无关。

## 思路

链上有关节点 $\mathbf{p}_0$（根）到 $\mathbf{p}_n$（末端），骨长 $d_i = \lVert \mathbf{p}_{i+1} - \mathbf{p}_i \rVert$，目标 $\mathbf{t}$。IK 要求末端到达目标，同时所有骨长不变、根不动。

雅可比法和 CCD 都在关节角空间里求解，每一步要算旋转。FABRIK 的观察是：如果只关心关节点的位置，“保持骨长”这个约束在几何上极其简单，给定一个点和一个方向，下一个点就在那个方向上距离 $d_i$ 的地方。于是可以不断地“拖”：拽着末端走到目标，每个关节被它的子关节拖着走，始终离子关节 $d_i$ 远；然后根被拽回原位，再反过来拖一遍。每遍都满足一端约束、破坏另一端，交替几次就同时满足两端。

## 算法

**可达性检查。** 若 $\lVert \mathbf{t} - \mathbf{p}_0 \rVert > \sum d_i$，目标够不到，直接把整条链朝目标拉直：

$$
\mathbf{p}_{i+1} = \mathbf{p}_i + d_i \frac{\mathbf{t} - \mathbf{p}_i}{\lVert \mathbf{t} - \mathbf{p}_i \rVert}
$$

**否则迭代，每次两个阶段：**

第一阶段（论文称 forward reaching，从末端往根）：记住根的原位置 $\mathbf{b} = \mathbf{p}_0$，令 $\mathbf{p}_n = \mathbf{t}$，对 $i = n-1, \dots, 0$：

$$
\mathbf{p}_i \leftarrow \mathbf{p}_{i+1} + d_i \frac{\mathbf{p}_i - \mathbf{p}_{i+1}}{\lVert \mathbf{p}_i - \mathbf{p}_{i+1} \rVert}
$$

第二阶段（backward reaching，从根往末端）：令 $\mathbf{p}_0 = \mathbf{b}$，对 $i = 0, \dots, n-1$：

$$
\mathbf{p}_{i+1} \leftarrow \mathbf{p}_i + d_i \frac{\mathbf{p}_{i+1} - \mathbf{p}_i}{\lVert \mathbf{p}_{i+1} - \mathbf{p}_i \rVert}
$$

当 $\lVert \mathbf{p}_n - \mathbf{t} \rVert$ 小于容差，或达到最大迭代次数时停止。注意“前向/后向”的叫法：论文里的 forward 指从末端出发的那一遍，和很多中文资料的直觉相反，原笔记的描述也把两者弄反了。叫法无所谓，顺序是固定的：先把末端贴到目标，再把根拉回来。

```cpp
// FABRIK 单链求解，只演示位置部分；UE 的 FABRIK 节点内部思路相同
// Joints[0] 为根，Joints.Num()-1 为末端；BoneLengths[i] 为 Joints[i] 到 Joints[i+1] 的长度
void SolveFABRIK(TArray<FVector>& Joints, const TArray<float>& BoneLengths,
                 const FVector& Target, float Tolerance, int32 MaxIterations)
{
	const int32 N = Joints.Num() - 1;
	const FVector Root = Joints[0];

	float TotalLength = 0.f;
	for (float L : BoneLengths) { TotalLength += L; }

	if (FVector::Dist(Root, Target) >= TotalLength)
	{
		// 够不到：朝目标拉直
		for (int32 i = 0; i < N; ++i)
		{
			const FVector Dir = (Target - Joints[i]).GetSafeNormal();
			Joints[i + 1] = Joints[i] + Dir * BoneLengths[i];
		}
		return;
	}

	for (int32 Iter = 0; Iter < MaxIterations; ++Iter)
	{
		if (FVector::Dist(Joints[N], Target) <= Tolerance) { break; }

		// 第一阶段：末端贴到目标，往根方向拖
		Joints[N] = Target;
		for (int32 i = N - 1; i >= 0; --i)
		{
			const FVector Dir = (Joints[i] - Joints[i + 1]).GetSafeNormal();
			Joints[i] = Joints[i + 1] + Dir * BoneLengths[i];
		}

		// 第二阶段：根拉回原位，往末端方向拖
		Joints[0] = Root;
		for (int32 i = 0; i < N; ++i)
		{
			const FVector Dir = (Joints[i + 1] - Joints[i]).GetSafeNormal();
			Joints[i + 1] = Joints[i] + Dir * BoneLengths[i];
		}
	}
}
```

这段只求出了关节点的新位置。要得到骨骼旋转，还要对每根骨骼算“旧方向 → 新方向”的最短旋转（`FQuat::FindBetweenNormals`），叠到原来的旋转上。这一步只确定了骨骼指向，绕骨骼自身轴的扭转保持原样。

## 为什么它好用

每次迭代对每个关节只做一次归一化和一次乘加，开销是 $O(n)$，没有矩阵求逆，也不会遇到奇异。论文给出的对比里，FABRIK 达到同样精度所需的迭代次数和耗时都明显少于 CCD 和雅可比类方法。它的结果也比 CCD 自然：CCD 从末端开始转，靠近末端的关节承担了大部分弯曲，链容易卷成钩状；FABRIK 的拖动把调整分摊到整条链上。

论文还给出了几个扩展：

- **关节约束。** 每次放置关节点后，把它投影到父骨骼允许的方向锥（或更一般的旋转限制区域）内，同时处理扭转限制。
- **多末端。** 骨骼树有分叉时（例如脊椎上挂两条手臂），先各分支分别做第一阶段，把各分支算出的分叉点位置取平均（质心）作为分叉点，再继续往根拖；第二阶段从根出发，到分叉点后分别向各分支展开。
- **闭环和运动目标。** 目标每帧移动时，从上一帧的解开始迭代，一两次就能跟上。

## 在 UE 里

AnimGraph 右键搜索 FABRIK 添加节点，它工作在组件空间。主要属性：

| 属性 | 作用 |
| --- | --- |
| `Effector Transform` | 目标变换，可以作为引脚接变量 |
| `Effector Transform Space` | 目标所在空间：World、Component、Parent Bone、Bone |
| `Effector Transform Bone` | 空间为 Bone 时参照的骨骼 |
| `Effector Rotation Source` | 末端骨骼的旋转：保持组件空间旋转、保持局部空间旋转，或复制目标的旋转 |
| `Tip Bone` / `Root Bone` | 链的末端和起点；链至少两段 |
| `Precision` | 末端与目标的距离容差 |
| `Max Iterations` | 最大迭代次数 |
| `Alpha` | 修正结果的权重 |

UE 的 FABRIK 节点不提供关节角度限制，需要限制时用 CCDIK 节点（Experimental），或者 Control Rig / IK Rig 的 Full Body IK。只有三关节的四肢，用 Two Bone IK 的解析解更便宜、更稳定。

## 容易踩的坑

**链太短或 Root/Tip 选反。** Root 必须是 Tip 的祖先，链至少包含两段骨骼。

**扭转丢失或错乱。** FABRIK 只决定骨骼指向，前臂、小腿这类带扭转骨骼的链，扭转仍来自原动画，目标旋转变化很大时要额外处理。

**肘、膝翻向。** 没有约束时，FABRIK 可能从初始姿势收敛到镜像解。关节点的初始位置就是动画姿势，动画本身弯曲方向正确通常就不会翻；对四肢更推荐 Two Bone IK 加 Joint Target。

**迭代次数开得很大。** 目标够不到或被约束卡住时，多出的迭代只是白算，用 `Precision` 和合理的 `Max Iterations` 控制。

## 相关

[[ECAB-IK和FK]] [[ECAH-循环星标下降算法]] [[ECAI-雅可比矩阵法]] [[EAABAA-IK]]
