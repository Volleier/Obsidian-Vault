**“基于流的消除高阶的时间同步”指的是 Zachary Booth Simpson 在 2000 年提出的一种网络游戏时钟同步方法：客户端向服务器发多次时间请求，按往返延迟排序取中位数，丢掉延迟明显偏高的样本（主要是 TCP 重传造成的离群值），对剩余样本的时钟偏差取平均。“基于流”指它可以跑在 TCP 这类流式协议上，“消除高阶”来自 GAMES104 课件的标题 *Stream-Based Time Synchronization with Elimination of Higher Order Modes*，意思是剔除延迟直方图里高延迟一侧的“杂散众数”，而不是消除什么高阶误差项。**

> 出处：Zachary Booth Simpson, *A Stream-based Time Synchronization Technique For Networked Computer Games*, 2000-03-01（原载 mine-control.com，现可在 Internet Archive 查到）；GAMES104 第 18 讲（网络游戏的架构基础）。UE 部分对照 Epic 5.8 API 文档 `AGameStateBase`。
>
> 原笔记把这个名字理解成“对网络延迟的二阶、三阶变化率建模并消除”，用多项式拟合、卡尔曼滤波校正高阶误差项。原论文和课件里都没有这些内容，这部分是对译名的误读，已删去。

## 要解决的问题

网络游戏里大量逻辑需要一个各端认可的“当前时间”：航位推测（dead reckoning）要知道一个位置包是多久以前发出的，才能正确外推；技能冷却、倒计时、回合开始时间要在各端对齐；延迟补偿要把客户端的操作映射回服务器时间线。如果不同步时钟，客户端只能假设每个包都是零延迟到达的，外推误差至少等于那一包的延迟。Simpson 在文中给出的量级是 100～3000 ms，而人对超过约 150 ms 的操作延迟就很敏感。

通用方案是 NTP，但作者认为它不适合游戏：收敛太慢，玩家不愿意在开局等时钟同步；NTP 和 SNTP 都基于 UDP，当时很多 ISP 和企业网络会拦截 UDP。改用 TCP 又带来新问题：SNTP 这类协议把往返时间除以二当作单程延迟，前提是来回对称，而 TCP 会在底层重传丢失或乱序的包，这次测量就出现了异常且不对称的延迟，上层代码还无从得知重传发生过。

## 算法

1. 客户端在“时间请求”包里写上本地时间 $t_c$，发给服务器；
2. 服务器收到后写上服务器时间 $t_s$，发回；
3. 客户端收到时本地时间为 $t_c'$，算出延迟和时钟偏差：

$$
\text{latency} = \frac{t_c' - t_c}{2},\qquad
\Delta = t_s - t_c' + \text{latency}
$$

  到这一步和 SNTP 基本一样；
4. 第一个结果立刻用来校正本地时钟，先让时间“大致对”；
5. 重复 1～3 五次或更多，每次间隔几秒，期间尽量减少其他流量；
6. 把样本按延迟从低到高排序，取中间那个作为中位延迟；
7. 丢掉延迟高于中位数约一个标准差的样本，对剩下样本的 $\Delta$ 取算术平均，作为最终的时钟偏差。

GAMES104 课件对第 7 步的表述是“丢掉延迟约大于中位数 1.5 倍的样本”，和原文“一个标准差”略有不同，两种阈值都常见，实际效果差不多。

## “消除高阶众数”是什么意思

整个算法唯一不平凡的地方就是第 7 步。原文的解释是：五个样本都没有被重传时，延迟直方图只有一个众数（簇），聚集在中位延迟附近；如果其中一个包被 TCP 重传，它的延迟会远远落在直方图右侧，平均约为主众数中位数的两倍远。把离中位数超过一个标准差的样本砍掉，这些“杂散众数”就被剔除了，前提是它们不占样本的大多数。

所以“高阶”指的是直方图里比主众数更靠右（延迟更高）的那些众数，“消除”就是丢弃它们。用中位数而不是平均数做基准，也是为了不让离群值把基准本身拉偏。

## 和 NTP 的关系

NTP 每次交换记录四个时间戳：客户端发送 $T_1$、服务器接收 $T_2$、服务器发送 $T_3$、客户端接收 $T_4$，

$$
\theta = \frac{(T_2 - T_1) + (T_3 - T_4)}{2},\qquad
\delta = (T_4 - T_1) - (T_3 - T_2)
$$

$\theta$ 是时钟偏差，$\delta$ 是扣除服务器处理时间后的往返延迟。Simpson 的做法只用一个服务器时间戳，把服务器处理时间算进了往返延迟里，精度更低，但实现只要几十行。两者都依赖“来回延迟对称”这个假设，不对称的部分（例如上行走调制解调器、下行走卫星的连接）会原样变成时钟误差，任何只靠往返测量的方法都消除不了。NTP 在此之上还有样本滤波、多服务器选择、时钟频率修正等完整机制，精度高得多，代价是收敛以分钟计。

| | SNTP | Simpson 方法 | NTP |
| --- | --- | --- | --- |
| 每次交换的时间戳 | 客户端两个 + 服务器一个 | 同 SNTP | 四个 |
| 样本处理 | 单次 | 中位数 + 剔除离群 + 平均 | 滤波、选择、聚类、频率修正 |
| 传输协议 | UDP | 可跑在 TCP 上 | UDP |
| 收敛 | 一次交换 | 几次交换、十几秒内 | 慢 |
| 典型精度 | 取决于单次延迟 | 作者报告通常 < 100 ms | 远高于前两者 |

作者在 1997 年的即时战略游戏《NetStorm: Islands At War》里用了这个算法，报告同步误差通常小于 100 ms；因重传导致的同步失败很少见，而且出现时往往连接本身已经糟糕到会掉线。

## 在 UE 里

UE 自带一个更简单的同步时钟：`AGameStateBase::GetServerWorldTimeSeconds()`，官方描述为“服务器上模拟的 TimeSeconds，在客户端和服务器之间同步”。机制是服务器定期（`ServerWorldTimeSecondsUpdateFrequency`，设为 0 关闭）调用 `UpdateServerTimeSeconds()`，把自己的 `GetWorld()->GetTimeSeconds()` 写进复制变量 `ReplicatedWorldTimeSecondsDouble`（5.x 起为 double，UE4 是 float 的 `ReplicatedWorldTimeSeconds`）；客户端在 `OnRep_ReplicatedWorldTimeSecondsDouble()` 里算出“服务器时间 − 本地时间”，累加求平均（样本数超过 250 时把累积值折算成一个样本重新开始），再把 `ServerWorldTimeSecondsDelta` 以 0.5 的系数向这个平均值靠拢。`GetServerWorldTimeSeconds()` 返回本地 `GetTimeSeconds()` 加上这个 delta。

这套机制省带宽、足够给 UI 倒计时用，但它**不做延迟补偿**：客户端收到的服务器时间已经过去了一个单程延迟，算出的服务器时间系统性地落后约半个 RTT。更新间隔在 UE4 时代据社区文章是 5 秒，新版本的默认值以头文件为准。需要更准的时钟（射击判定、音乐节奏游戏、同时开局）时，常见做法是在 PlayerController 上用一对 RPC 自己做往返测量，也就是上面的算法：

```cpp
// MyPlayerController.h
#pragma once

#include "GameFramework/PlayerController.h"
#include "MyPlayerController.generated.h"

UCLASS()
class MYGAME_API AMyPlayerController : public APlayerController
{
	GENERATED_BODY()

public:
	// 客户端估计的服务器时间；服务器上直接返回本地时间
	double GetSyncedServerTime() const;

protected:
	virtual void ReceivedPlayer() override;

	UFUNCTION(Server, Unreliable)
	void ServerRequestTime(double ClientSendTime);

	UFUNCTION(Client, Unreliable)
	void ClientReportTime(double ClientSendTime, double ServerTime);

private:
	void SendTimeRequest();

	struct FTimeSample { double Latency; double Offset; };
	TArray<FTimeSample> Samples;
	double ServerTimeOffset = 0.0;
	FTimerHandle TimeSyncTimer;
};
```

```cpp
// MyPlayerController.cpp
#include "MyPlayerController.h"

#include "Engine/World.h"
#include "TimerManager.h"

void AMyPlayerController::ReceivedPlayer()
{
	Super::ReceivedPlayer();
	if (IsLocalController() && GetNetMode() == NM_Client)
	{
		SendTimeRequest();
		GetWorldTimerManager().SetTimer(TimeSyncTimer, this, &AMyPlayerController::SendTimeRequest, 2.0f, true);
	}
}

void AMyPlayerController::SendTimeRequest()
{
	ServerRequestTime(GetWorld()->GetTimeSeconds());
}

void AMyPlayerController::ServerRequestTime_Implementation(double ClientSendTime)
{
	ClientReportTime(ClientSendTime, GetWorld()->GetTimeSeconds());
}

void AMyPlayerController::ClientReportTime_Implementation(double ClientSendTime, double ServerTime)
{
	const double Now = GetWorld()->GetTimeSeconds();
	const double Latency = (Now - ClientSendTime) * 0.5;
	const double Offset = ServerTime + Latency - Now;

	if (Samples.IsEmpty())
	{
		ServerTimeOffset = Offset; // 第一个样本立即生效
	}
	Samples.Add({ Latency, Offset });
	if (Samples.Num() > 16)
	{
		Samples.RemoveAt(0); // 只保留最近的样本，适应网络变化
	}
	if (Samples.Num() < 5)
	{
		return;
	}

	// 按延迟排序，取中位数和标准差
	TArray<FTimeSample> Sorted = Samples;
	Sorted.Sort([](const FTimeSample& A, const FTimeSample& B) { return A.Latency < B.Latency; });
	const double Median = Sorted[Sorted.Num() / 2].Latency;

	double Mean = 0.0;
	for (const FTimeSample& S : Sorted) { Mean += S.Latency; }
	Mean /= Sorted.Num();
	double Var = 0.0;
	for (const FTimeSample& S : Sorted) { Var += FMath::Square(S.Latency - Mean); }
	const double StdDev = FMath::Sqrt(Var / Sorted.Num());

	// 丢掉比中位数高出一个标准差以上的样本，其余取平均
	double Sum = 0.0;
	int32 Count = 0;
	for (const FTimeSample& S : Sorted)
	{
		if (S.Latency <= Median + StdDev)
		{
			Sum += S.Offset;
			++Count;
		}
	}
	ServerTimeOffset = Sum / Count;
}

double AMyPlayerController::GetSyncedServerTime() const
{
	const double Local = GetWorld()->GetTimeSeconds();
	return HasAuthority() ? Local : Local + ServerTimeOffset;
}
```

这里用的是 Unreliable RPC。UE 的网络层跑在 UDP 上，不可靠 RPC 丢了就丢了，不会像 TCP 那样重传后以异常延迟到达，所以 Simpson 当年要对付的重传问题不存在；但路由排队、带宽突发、服务器一帧的处理延迟同样会产生高延迟离群值，剔除步骤仍然有用。服务器端要注意 `ServerRequestTime` 的调用频率，防止客户端刷 RPC。

## 容易踩的坑

**直接把 `GetServerWorldTimeSeconds()` 当精确的服务器时间。** 它没有扣除单程延迟，延迟 100 ms 的客户端会慢大约 50 ms，更新间隔长时还会有额外漂移。

**用平均数而不是中位数做基准。** 一个 800 ms 的离群样本就能把平均延迟拉高很多，剔除阈值跟着失效。

**把同步后的偏差瞬间套上去。** 时钟往回跳会导致倒计时倒退、插值时间轴错乱。偏差变化较大时应该平滑过渡，UE 自带实现里 0.5 的系数就是这个用途。

**混用不同的时间源。** `GetTimeSeconds()` 受时间膨胀和暂停影响，`GetRealTimeSeconds()` 不受暂停影响，`FPlatformTime::Seconds()` 是系统时间。发送、接收、使用必须是同一种时间。

**以为多采样能消除不对称延迟。** 上下行延迟不对称造成的误差在每个样本里都一样，平均不掉。
