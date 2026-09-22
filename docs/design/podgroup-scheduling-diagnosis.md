# PodGroup 调度诊断的 Plugin-Model 支持

- 跟踪 issue：[kubernetes/kubernetes#141025](https://github.com/kubernetes/kubernetes/issues/141025)
- 相关工作：[#138991](https://github.com/kubernetes/kubernetes/issues/138991)（单 pod 的机器可读诊断）、
  [#140670](https://github.com/kubernetes/kubernetes/pull/140670) 与
  [#141860](https://github.com/kubernetes/kubernetes/pull/141860)（CompositePodGroup condition 与层级传播）
- SIG：sig-scheduling
- 文中所有行号均以基线提交 `7c2b6c32644`（2026-07-28）为准。

## 目录

- [摘要](#摘要)
- [动机](#动机)
- [提案](#提案)
- [详细设计](#详细设计)
- [备选方案](#备选方案)
- [测试计划](#测试计划)
- [后续工作](#后续工作)
- [缺陷](#缺陷)
- [实施历史](#实施历史)

---

## 摘要

本提案为 PodGroup 调度失败提供一份聚合的、人类可读的诊断信息，写入
`PodGroup.status.conditions[type=PodGroupInitiallyScheduled].message`。

诊断以**节点**为聚合维度，将 placement 内的节点划分为两类：接收过 gang pod 之后开始拒绝的
**饱和节点**，以及自始拒绝全部 pod 的**排除节点**；并为每一类附上**按失败原因聚类**的分布。
该划分直接对应运维可采取的动作，且在 5000 节点规模的集群上仍然只渲染为若干行。

实现分四个阶段，前三个阶段均可独立发布：

| 阶段 | 内容 | 是否修改 plugin model |
|---|---|---|
| 1 | 节点二分类与原因聚类 | 否 |
| 2 | 为 `fwk.Status` 增加 `RejectionCode` | 是（纯增量） |
| 3 | 按 `PodSignature` 分区渲染 | 否 |
| 4 | 节点资源数字（未排期） | 否 |

阶段 1 不修改 framework、不引入任何推导、不在调度热路径上产生任何开销，全部内容均可在当前的
plugin model 上运行。节点资源数字（形如 `cpu 60/64 used, 60 by this gang`）被推迟至阶段 4，
理由见[非目标](#非目标)。

## 动机

### 问题陈述

当一个 PodGroup（gang）调度失败时，呈现给用户的诊断信息不足以支撑排查。issue
[#141025](https://github.com/kubernetes/kubernetes/issues/141025) 所描述的期望消息形如：

```
Gang scheduling failed: 30/100 pods placed, minCount=80.

Placement by node (30 pods):
  node-A: 15 pods (cpu: 60/64)
  node-B: 10 pods (cpu: 40/64)

Failed pods (70), by failure signature:
  45 pods: Insufficient cpu
  15 pods: Insufficient cpu + untolerated taint

Top rejecting nodes:
  node-A: rejected 70/70 failed pods (Insufficient cpu — 60/64 cpu used by placed pods)
```

该期望消息隐含了三段结构：按节点的放置情况、按 pod 的失败签名、以及拒绝最多的节点。当前的
condition 消息只能给出组级的一句概括，既没有节点维度，也没有原因分布，运维无法据此判断
应当扩容、应当放宽约束，还是应当降低 `minCount`。

review 过程中，macsko 提出了如下意见：

> implementing more advanced messages, e.g., your proposed CPU counting might require
> non-trivial changes in scheduler plugin model.

本提案对该意见的回应是：其中被点名的资源计数确实存在计账正确性方面的审查成本，因此本期
不予实现；而剩余的诊断内容不需要任何 plugin model 改动即可交付。详见
[约束与注意事项](#约束与注意事项)。

### 目标

- 为失败的 PodGroup 提供一份聚合诊断，使运维能够直接判断失败属于容量问题还是约束问题。
- 精确界定：实现该诊断所需的 plugin model 表面究竟为何，并将必需的部分与仅提升质量的部分
  区分开。
- 所提出的机制在调度热路径上零开销或接近零开销。
- 机制的价值不局限于 gang 调度：单 pod 诊断（#138991）应能消费同一份数据。
- 本期交付**按失败原因聚类**的诊断结果，即说明哪些节点在接收过 gang pod 之后开始拒绝、
  哪些节点自始拒绝、哪些节点未被评估，并为每一类附上原因分布。本期不包含任何节点资源量。

### 非目标

- **节点资源数字（`cpu 60/64 used, 60 by this gang`），本期不予实现。** 该部分是 maintainer
  最可能重点审查的内容，因为它要求聚合器的算术逐一镜像各插件的计账规则，而 DRA extended
  resources、pod-level resources 与 in-place vertical scaling 各自存在细节差异，任何一处
  偏差都会导致诊断输出一个看似权威、实则错误的数字，其危害大于缺失该数字。节点分类的结论
  并不依赖这些数字（见[分类不依赖 reason 文本](#分类不依赖-reason-文本)），因此本期无需承担
  该项风险。推导过程见[后续工作](#后续工作)。
- CompositePodGroup 的 condition patch，以及失败原因沿层级的传播。该部分已由
  [#140670](https://github.com/kubernetes/kubernetes/pull/140670) 与
  [#141860](https://github.com/kubernetes/kubernetes/pull/141860) 承载，详见
  [与其他工作的关系](#与其他工作的关系)。
- 修改调度算法。本文只涉及上报。
- 支持配置了 filter extender 的集群。extender 不参与 framework 的 reason 体系，被其剔除的节点
  以及未被扫描的节点只能落入不带 reason 的默认 `absentNodesStatus`，节点分类在该场景下不可靠。
  阶段 1 检测到 `hasExtenderFilters()` 为真时退出渲染，保留现有消息，见
  [不支持 extender 场景](#不支持-extender-场景)。
- 将机器可读诊断做成 API 表面（属于 #138991 的范畴）。本文档的全部内容都承载于
  `metav1.Condition.Message` 之中。
- 最终 condition 消息的具体措辞。布局**在**范围内，但仅限于它会改变 plugin model 必须提供
  什么的部分——聚合维度的选择之所以需要论证，是因为它决定了所需的插件侧支持，而非为了
  敲定文案。

## 提案

### 用户故事

**作为集群运维人员**，当 gang 调度失败时，我希望从 PodGroup 的 condition 中直接判断失败属于
容量不足还是约束不满足，以便决定应当扩容、应当放宽污点与亲和性约束，还是应当降低 `minCount`，
而不必逐个查看数十乃至数百个 pod 的 condition。

**作为工作负载控制器的作者**，我希望获得一个语义明确的组级失败信号。当前的组级消息无法区分
「gang 自身消耗尽了节点容量」与「节点对该 gang 自始不可用」，而这两者的处置方式完全相反。
下游已经出现对该信号的需求，例如
[kubernetes-sigs/lws#1056](https://github.com/kubernetes-sigs/lws/issues/1056) 正在讨论
`RecreateGroupAfterStart` 在原生 gang 语义下的行为。

**作为自动化工具与 AI agent 的作者**，我希望诊断消息的结构稳定、基数有界、截断显式，
以便可以依赖其分段进行解析，而不必担心消息长度随集群规模无界增长。

### 方案概述

聚合器在 `submitPodGroupAlgorithmResult` 处已经持有完整的 pod 到 node 的放置映射，以及每个
失败 pod 完整的 `FitError`。本提案不新增任何数据采集，只改变**读取方向**：不是遍历 pod 并
收集其原因集合，而是遍历各失败 pod 的 `NodeToStatus`，将观测归到节点名下。

在此基础上，依据每个节点的「接收下标集」与「拒绝下标集」将其划分为饱和与排除两类，
并按失败原因聚类后渲染为一张节点表：

```
pod group is unschedulable, minCount (80) is not yet satisfied: 30 scheduled, 0 remaining
30/100 pods placed in this cycle, 50 nodes in placement.

Saturated nodes (3) - accepted some gang pods, rejected others:
  node-A: accepted 15, rejected 70 (Insufficient cpu)
  node-B: accepted 10, rejected 70 (Insufficient cpu)
  node-C: accepted  5, rejected 70 (Insufficient memory)

Excluded nodes (47) - rejected every pod:
  35 nodes: untolerated taint
  12 nodes: node affinity mismatch
```

首行原样保留现有的组级 status 消息，使 PlacementFeasible 给出的原因不被丢弃，也使
[#141860](https://github.com/kubernetes/kubernetes/pull/141860) 的祖先前缀仍然附加在第一行。

`rejected` 统计的是**失败 pod** 的数量，而非全部 pod。`SchedulePod` 会丢弃成功 pod 的诊断，
因此一个成功 pod 在某节点上被拒的事实不可恢复；上例中 30 个已放置 pod 与 70 个失败 pod 之和为
100，而每行 `accepted + rejected` 为 85、80、75，差额即那些通过了该节点筛选、最终却落在别处的
已放置 pod。

两类节点各自对应一种互斥的处置方向，这是选择节点维度的核心理由，论证见
[为何以节点为默认维度](#为何以节点为默认维度)。

### 分阶段实施

- **阶段 1 —— 不修改 framework，不进行推导。** 输出节点二分类表与按失败原因聚类的直方图，
  聚类键为 `(plugin, reason)`。全部内容均可在当前的 plugin model 上运行。
- **阶段 2 —— `RejectionCode`。** 为 `fwk.Status` 增加一个机器可读、稳定的拒绝标识，取代
  reason 串作为聚类键，使直方图在插件将变量拼入 reason 时不再退化。该阶段为纯增量改动，
  改善阶段 1 的输出而无需重构它。
- **阶段 3 —— 按 `PodSignature` 分区。** 为每个 pod shape 渲染一张节点表，消除异构 gang 与
  交错两项限制，并将饱和推断升级为确定性结论。该阶段复用 opportunistic-batching 工作已经
  建成的基础设施，不需要新的 framework 表面。
- **阶段 4 —— 节点资源数字（未排期）。** 为饱和节点行附加 `cpu 60/64 used, 60 by this gang`。
  该阶段需要一次以计账正确性为主题的独立 review，因此不与前三个阶段捆绑发布。

阶段 1 的规模符合本次发布的预期，且不包含 review 意见所点名的资源计数。

### 约束与注意事项

- 阶段 1 不修改 framework，因此不引入 feature gate、不改变任何插件的契约、不影响任何
  out-of-tree 插件。
- 阶段 2 为 `fwk.Status` 增加一个字段与两个方法，现有调用点无需改动；未设置该字段的
  `Status` 与当前行为完全一致，因此插件可以增量迁移。
- 诊断的全部计算**仅在失败路径上**发生，成功的调度周期不产生任何额外开销。
- 消息体积受 `metav1.Condition.Message` 的 32 768 字符上限约束，构建器执行一个远低于该上限
  的预算，详见[消息体积预算](#消息体积预算)。

### 风险与缓解

| 风险 | 缓解 |
|---|---|
| 消息体积超出 condition 上限 | 分段预算与显式截断；节点维度使各段基数天然有界 |
| 插件将变量插入 reason，导致聚类退化为一节点一桶 | 阶段 1 采用 `(plugin, reason)` 复合键并设置桶数上限；阶段 2 以 `RejectionCode` 根治 |
| 节点采样导致拒绝统计的分母不可靠 | 仅从**失败**的 pod 构建拒绝统计；该约束写入 helper 的 doc comment |
| TAS 下单个 placement 的结果被误读为全局失败 | 消息显式标注其描述的是哪一个 placement |
| 交错情形下措辞暗示一次并不存在的干净转折 | 使用 `accepted X, rejected Y`，不使用 `then` |
| 热路径开销 | 全部计算仅在失败路径上执行；阶段 2 仅增加一个 16 字节的 string header，无新增分配 |
| 与 #141860 的前缀嵌套叠加 | 体积预算以「本层消息」为计算基准 |

---

## 详细设计

### 当前的失败上报模型

#### `fwk.Status` 只承载字符串

`staging/src/k8s.io/kube-scheduler/framework/interface.go:115`：

```go
type Status struct {
	code    Code
	reasons []string
	err     error
	// plugin is an optional field that records the plugin name causes this status.
	// It's set by the framework when code is Unschedulable, UnschedulableAndUnresolvable or Pending.
	plugin string
}
```

`Message()` 即 `strings.Join(s.Reasons(), ", ")`，没有任何结构化载荷。插件名有记录，但只有
一个字符串。

#### 插件算出了结构化数据，随后将其丢弃

`pkg/scheduler/framework/plugins/noderesources/fit.go:692`：

```go
type InsufficientResource struct {
	ResourceName v1.ResourceName
	// We explicitly have a parameter for reason to avoid formatting a message on the fly
	// for common resources, which is expensive for cluster autoscaler simulations.
	Reason    string
	Requested int64
	Used      int64
	Capacity  int64
	Unresolvable bool
}
```

阶段 4 所需的全部数据就在这里，但本期并不使用它。`Filter` 在 `fit.go:685` 将其压扁：

```go
failureReasons := make([]string, 0, len(insufficientResources))
for i := range insufficientResources {
	failureReasons = append(failureReasons, insufficientResources[i].Reason)
	if insufficientResources[i].Unresolvable {
		statusCode = fwk.UnschedulableAndUnresolvable
	}
}
return fwk.NewStatus(statusCode, failureReasons...)
```

`Requested`、`Used`、`Capacity`、`ResourceName` 从未离开插件。该结构体唯一存活的地方是
kubelet 准入路径（`pkg/scheduler/eventhandlers.go:859`），它直接调用导出的
`noderesources.Fits()`，绕开了 `Status`。

`Reason` 字段上的那句注释本身即说明该路径对开销的敏感程度：该字段存在的唯一理由，就是避免
对每个被拒节点执行一次 `fmt.Sprintf`。任何在此处新增分配的方案都必须以此为基准来衡量。

#### 聚合器当前已经能看到什么

每个失败 pod 的 `FitError` 都被保留（`pkg/scheduler/framework/types.go:1439`）：

```go
type FitError struct {
	Pod         *v1.Pod
	NumAllNodes int
	Diagnosis   Diagnosis
}

type Diagnosis struct {
	NodeToStatus         *NodeToStatus
	UnschedulablePlugins sets.Set[string]
	PendingPlugins       sets.Set[string]
	PreFilterMsg         string
	PostFilterMsg        string
}
```

`NodeToStatus`（`pkg/scheduler/framework/interface.go:33`）持有一个显式的
`map[string]*fwk.Status` 记录被拒节点，外加一个 `absentNodesStatus`（同文件 `:38`）覆盖所有
不在 map 中的节点。每个节点只记录**第一个**拒绝它的 Filter 插件的 status——`RunFilterPlugins`
在第一个非 success 时即返回（`pkg/scheduler/framework/runtime/framework.go:1126`）。

在 `submitPodGroupAlgorithmResult` 中，每个 `algorithmResult` 都带有 `podInfo`、
`scheduleResult.SuggestedHost` 与 `status`，因此聚合器手上已经具备完整的 pod 到 node 放置
映射，以及每个 pod 完整的 `FitError`。**本提案不新增任何数据采集。**

### 数据可得性分析

将期望消息拆解为其数据需求，逐项核对当前可得性：

| # | 数据需求 | 当前可得性 |
|---|---|---|
| R1 | 哪些 pod 被放置，放置在哪个节点 | **可得** —— `algorithmResult.scheduleResult.SuggestedHost` |
| R2 | 哪些 pod 从未被评估 | **可得，但仅为隐式** —— 见下 |
| R3 | 节点级资源总量（`cpu: 60/64`） | **可推导，本期不予实现** —— 见[后续工作](#后续工作) |
| R4 | 每个**节点**的拒绝原因 | **可得，但为自由字符串** —— 反查各失败 pod 的 `NodeToStatus` |
| R5 | 某次拒绝涉及哪一种资源 | **不可得** —— 需要解析 `"Insufficient cpu"` |
| R6 | 某节点拒绝了多少个 pod | **可得，且为常量** —— 失败 pod 不存在可行节点，故 placement 内每个节点都拒绝了每一个失败 pod，计数恒等于失败 pod 数，见[聚合规则](#聚合规则) |

R4 与 R6 都是**节点维度**的表述。其数据来源仍然是各 pod 的 `FitError`，但读取方向相反：
不是遍历 pod 并收集其原因集合，而是遍历 pod 的 `NodeToStatus`，将观测归到节点名下。该方向
差异决定了整个设计的形状。

其中只有 R5 构成真正的 plugin-model 缺口。R3 恰恰是被 review 意见点名为需要 "non-trivial
changes in scheduler plugin model" 的部分，但它实际上不需要修改任何插件即可推导得出；本期
推迟该项并非因为无法实现，而是因为其正确性论证的成本高于它在本期所能提供的信息价值。

#### R2：将「未评估」显式化

`podGroupSchedulingDefaultAlgorithm` 在 `PlacementFeasible` 返回 Unschedulable 时立即跳出
pod 循环（`schedule_one_podgroup.go:635`），因此 `result.podResults` 比 pod 列表短。随后
`completePodGroupAlgorithmResult`（`schedule_one_podgroup.go:712`）以一批条目补齐，这些条目
的 `status` 是**组级** status 的克隆，`scheduleResult` 为零值。

补齐之后，「从未被评估」只能由条目的形状推断——克隆的组 status 加空的 `SuggestedHost`。
这一推断是脆弱的：一个**确实被评估过**、且被同一个组级 status 拒绝的 pod，与之无法区分。

有两个地方需要这一区分：

- **节点表的分母。** 未被评估的 pod 不得计入任何节点的 `rejectedAt` 集合，否则一个节点会
  凭空承担它从未见过的 pod 的拒绝数。组级消息首行的 `30/100 pods placed` 亦来自该计数。
- **pod 自身的 condition。** issue 第 1 节要求未被评估的 pod 陈述 "pod was not evaluated:
  gang scheduling aborted early"，而非复述一条组级失败原因，使用户误以为是该 pod 自身不满足
  条件。

处理方式：为 `algorithmResult` 增加一个显式的 `notEvaluated bool`，由
`completePodGroupAlgorithmResult` 在其合成的条目上置位。该字段纯属 `pkg/scheduler` 内部，
不涉及 framework 表面。

#### R5：聚类键的缺口

本期的输出即**按失败原因聚类**——排除节点的原因直方图，以及饱和节点行末尾的原因标注。聚类
需要一个键，而当前唯一可用的键是字面字符串 `"Insufficient cpu"`。以它作为键意味着：

- 需要解析一段不属于任何兼容性契约的消息文本；
- 对标量资源会静默失效——其 reason 由 `fmt.Sprintf("Insufficient %v", rName)` 构造，可以
  包含任意资源名；
- 非 `noderesources` 插件无法参与。

其直接后果是**排除节点按原因折叠**（R4）变得脆弱：按原始消息串归并虽然可行，但任何将变量
插入 reason 的插件（节点名、计数、拓扑 key）都会使每个节点产生一个独立的桶，于是直方图退化
为一节点一行，折叠完全失效。

在移除资源数字之后，该直方图构成本期消息中**唯一**的信息载体，因此键的稳定性由「影响输出
质量」上升为「本期主要风险」。阶段 1 的缓解方式是以 `(plugin, reason)` 复合键取代裸 reason
串：`RunFilterPlugins` 在写入节点状态之前调用 `status.SetPlugin(pl.Name())`
（`pkg/scheduler/framework/runtime/framework.go:1128`），因此 `Status.Plugin()` 在
`NodeToStatus` 的每个条目上都已填充，既不需要插件配合，也不增加任何开销。该复合键至少能够
将不同插件产生的相同文案区分开；同一插件将变量插入 reason 的情形仍会退化，该部分由阶段 2 的
`RejectionCode` 解决。

#### R3：本期不予实现

本期不渲染资源数字。该数据可以推导，推导过程亦属正确，但它是整份提案中唯一需要向 maintainer
论证「聚合器的算术与各插件的计账规则保持一致」的部分，而节点分类并不依赖它。完整的推导过程、
正确性论证与推迟理由见[后续工作](#后续工作)。

### 聚合维度：按节点而非按 pod

issue #141025 的期望布局有三段——按节点的放置情况、按 pod 的失败签名、拒绝最多的节点——其中
第一段与第三段本来就是节点维度的，只是从两个角度描述同一批节点。将二者合并为一张按节点的表
严格更有信息量，因为关于一个节点，最关键的事实是**转折点**：它在开始拒绝之前接收了多少个
gang pod。

#### 节点二分类

该转折点将每个节点划入以下两类之一：

| 类别 | 特征 | 对运维的意义 |
|---|---|---|
| **Saturated（饱和）** | 接收过至少一个 pod，之后拒绝了后续 pod | 该节点被 gang 自身消耗殆尽。属于容量问题——更大的节点、更多的节点、或更小的 `minCount` 可以解决。 |
| **Excluded（排除）** | 接收 0 个 pod，所看到的 pod 全部拒绝 | 该节点对该 gang 自始不可用。属于约束问题（污点、亲和性、选择器）或既有负载——为 gang 增加容量无效。 |

#### 不存在第三类节点

初稿曾设一个 **Unused（未触及）** 类，其特征为「接收 0，拒绝 0」，用以表示从未被评估的节点。
该类在机制上不可观测，已从本设计中移除。论证分两层。

**其一，产生 `FitError` 的 pod 必然评估了全部候选节点。** `findNodesThatPassFilters`
（`schedule_one.go:779`）的提前终止只有两个触发条件：已找到的可行节点数超过 `numNodesToFind`
（`schedule_one.go:827`），或某个 Filter 返回 `fwk.Error`（`schedule_one.go:819`）。前者以存在
至少一个可行节点为前提，与「该 pod 最终失败」互斥；后者使 `SchedulePod` 返回 error 而非
`FitError`（`schedule_one.go:582`），不产生任何诊断输出。因此在产生 `FitError` 的路径上，
`Parallelizer().Until` 会遍历 `numAllNodes` 的每一个下标，不存在被跳过的节点。

**其二，`NodeToStatus` 在结构上不区分「未评估」与「已拒绝」。** `NewDefaultNodeToStatus`
（`pkg/scheduler/framework/interface.go:43`）将 `absentNodesStatus` 默认置为
`UnschedulableAndUnresolvable`，而 `Get`（同文件 `:56`）对任何不在 map 中的节点一律返回该默认
值。于是「接收 0，拒绝 0」的节点在数据上并不存在：一个节点要么在 map 中带有显式拒绝，要么走
absent 分支同样得到拒绝。第三类是一个恒为空的集合。

在 framework 内部，节点未被逐一评估的情形只有 PreFilter 收窄候选集：当 `preRes.AllNodes()` 为假
时，扫描范围被限定为 `preRes.NodeNames`（`schedule_one.go:680`），其余节点从未进入 `checkNode`。
但该路径同时调用 `SetAbsentNodesStatus`，为这些节点赋予 `node(s) didn't satisfy plugin(s) ...`
这一**可区分的原因**（`schedule_one.go:689`）。因此它们表现为排除节点中的一个独立原因桶，而非
一个独立的节点类别。

存在一个 framework 之外的例外：配置了 filter extender 时，扫描可能先因找到足够可行节点而 cancel，
随后 extender 将这些节点全部剔除，pod 仍以 `FitError` 结束，而未被扫描的节点只能落入不带 reason
的默认 `absentNodesStatus`。该场景下节点分类不再可靠，阶段 1 显式不予支持，见
[不支持 extender 场景](#不支持-extender-场景)。

「未被评估」这一事实确实存在，但它位于 **pod 层级**而非节点层级。gang 在 `PlacementFeasible`
处提前中止时，被跳过的 pod 完全不持有 `Diagnosis`，此时不存在任何节点级观测可供分类。该事实由
R2 的 `notEvaluated` 标志承载，并在组级消息中以单独一行陈述，见
[R2：将「未评估」显式化](#r2将未评估显式化)。

#### 渲染形式与措辞约束

渲染效果见[方案概述](#方案概述)。该示例对应两项措辞约束：

- 使用 `accepted X, rejected Y`，而不使用 `accepted X, then rejected Y`。在交错情形下
  （拒绝 pod 2、接收 pod 5、拒绝 pod 9），"then" 会暗示一次实际上并不存在的干净转折，
  参见[该维度的边界](#该维度的边界)。
- 括号内仅放置原因，不放置资源量。分类结论由计数承载，消息不得暗示任何具体的资源数字。

#### 为何以节点为默认维度

- **边界落在运维真正能够处置的对象上。** 一行一个节点直接对应一个补救动作。按 pod 的签名
  列表无法做到这一点：`45 pods: Insufficient cpu` 完全没有说明失败发生在*何处*。
- **基数是收敛的。** 饱和节点数至多等于已放置 pod 数；排除节点坍缩为原因直方图，而
  `absentNodesStatus` 桶从构造上即坍缩为一行。5000 节点集群上的 100 pod gang 仍然只渲染为
  若干行。
- **它重建了 issue 所要求的因果链。** 「node-A 先接收了 15 个 gang pod，随后以 Insufficient
  cpu 拒绝了 70 个失败 pod」——该陈述**即为**饱和节点所在的那一行，不需要在两个独立段落之间交叉
  比对。因果关系由计数承载；具体消耗了多少 cpu 属于阶段 4 才附加的信息。

#### 分类不依赖 reason 文本

这一点需要单独说明，因为它改变了各阶段的分量。

R5 最初的重要性来自归因需求：若要陈述「node-A 拒绝这些 pod 是因为 cpu，而这些 cpu 正是 gang
自身所消耗」，聚合器就必须知道该次拒绝涉及哪一种资源，也就必须解析 `"Insufficient cpu"`。

在节点维度下，因果论断由**观测的结构**承载，而非由 reason 文本承载。一个在接收了 15 个 pod
之后才开始拒绝的节点，从构造上即可说明它对该 gang 是结构可用的——不存在污点、不存在亲和性
不匹配、不存在选择器问题，因为已经有 15 个 pod 落在其上。那么在 pod 15 与 pod 16 之间发生
变化的因素，只能是该 gang 自身的消耗。**即使每一个 reason 字符串都完全不可解析，该分类依然
成立。**

因此 `RejectionCode` 由**必需**降级为**提升质量**：它使聚类键在插件将变量拼入 reason 时保持
稳定，并在阶段 4 引入资源数字之后能够准确指明被耗尽的资源类型。两者均具实际价值，但都不再
构成承重结构。由此得到三项推论：

- 诊断工作不再被 framework 改动阻塞。它可以在当前的 plugin model 上落地，此后再逐步获得
  精度——这正是 review 中所要求的节奏（"adding more context to existing messages should be
  suitable for the incoming release"）。
- 在移除 R3 之后，该结论更为明确：本期不包含任何推导，聚合器只执行计数与分组，不产出任何
  一个需要与插件算术对账的数字。
- 备选方案 B 的论据相应削弱：其优势在于提供权威数字，而本期并不输出任何数字。

#### 评估顺序的可还原性

分类需要知道某节点上的「接收」发生在「拒绝」**之前**。`podGroupSchedulingDefaultAlgorithm`
在一个对 `queuedPodInfos` 的单线程顺序循环中 append 至 `result.podResults`
（`schedule_one_podgroup.go:605`），因此切片下标即为评估顺序。对节点 `N`：

```
acceptedAt(N)  = { i : podResults[i] 被放置在 N 上 }
rejectedAt(N)  = { i : podResults[i].FitError 中 N 拒绝了它 }
saturated(N)   = acceptedAt(N) ≠ ∅  且  max(acceptedAt) < max(rejectedAt)
```

不需要新增任何记账。

#### 该维度的边界

- **异构 gang。** 若 pod 形状不同，`node-A rejected 70` 可能掩盖了其中 60 个卡在 cpu、
  10 个卡在设备插件的事实。在 pod 形状可用之前，行内携带原因直方图而非单一原因，代价是行
  更长。这不是永久限制，见阶段 3。
- **交错。** 异构 gang 可能产生这样的节点：拒绝 pod 2、接收 pod 5、拒绝 pod 9。上述
  `saturated` 定义仍能正确分类，但该行应当同时报出两个计数，而不应暗示一次干净的转折。
- **小 gang。** issue 原本对不超过 10 个 pod 的 gang 采用逐 pod 布局。本设计**不保留该双
  模式**，理由见下节。
- **TAS。** 该表描述的是**单个** placement 的模拟结果（`schedule_one_podgroup.go:1041`
  上报的是 `anyResult`），因此节点计数的分母是该 placement 的节点集
  （`schedule_one.go:590` 的 `NumNodesInPlacement`），而非整个集群。placement 之外的节点不会
  进入 `ListNodesInPlacement`（`schedule_one.go:635`），在本设计中不构成一个节点类别。

#### 单一渲染器与三层分工

issue #141025 提议按 gang 大小分两种布局：不超过 10 个 pod 时逐 pod 列出，超过时才聚合。
本设计不采用该双模式，理由有二。

**其一，「小 gang 逐 pod 更紧凑」这一前提不成立。** 逐 pod 行的形式为
`pod-3: node-A: Insufficient cpu; node-B: untolerated taint`，它需要列出**节点**。在 5000
节点集群上，该行无法真正列出 5000 个节点，只能退化为「该 pod 的原因直方图」。而节点表在小
gang 下本来就很小：3 个 pod 至多产生 3 行饱和节点，加一行排除节点直方图。
因此节点表在两种规模下都紧凑，双模式换不来任何体积收益。

**其二，逐 pod 视图与 pod 自身的 condition 重复。** 每个失败 pod 的 condition 中已经带有它
自己的 `FitError`——这正是 issue 第 1 节（pod 级：保留个体诊断）所要保证的事情。将同样的信息
在 PodGroup condition 中再抄写一遍，只会造成两处可能不一致。PodGroup condition 应当回答
pod condition **无法回答**的问题：跨 pod 的节点占用与饱和关系。

因此 `buildPodGroupDiagnosis` 只有一个渲染器——始终是节点表。分层由此变得清晰：

| 层级 | 回答的问题 |
|---|---|
| Pod condition | 该 pod 为何未被调度？（逐 pod 的 `FitError`；gang 路径上通常是组级消息，见下） |
| PodGroup condition | 该 gang 为何未凑够 `minCount`？（节点占用、饱和、排除，以及是否提前中止） |
| CompositePodGroup condition | 哪些子组可行、哪些不可行？（由 #140670 与 #141860 承载） |

少一个渲染器，也就少一套需要维护与测试的布局分支。

#### 诊断不得写入组级 status

上表描述的是**目标**分层，而非 gang 路径的现状。当前实现中，被组级拒绝的已放置 pod 拿到的是
组级消息（`schedule_one_podgroup.go:869`），等待抢占的 pod 直接拿到 `podGroupResult.status`
（同文件 `:865`），`completePodGroupAlgorithmResult` 合成的未评估 pod 同样是组级 status 的克隆
（同文件 `:712`）；这些 status 经 `handleSchedulingFailure` 写入每个 pod 的 `PodScheduled`
condition 与 `FailedScheduling` 事件（`schedule_one.go:1152`、`:1181`、`:1253`、`:1259`）。

由此得到一条硬约束：**诊断只写入 PodGroup condition，不得写入 `podGroupResult.status`。**
而 PodGroup condition 的消息当前正是取自 `podGroupResult.status.Message()`
（`schedule_one_podgroup.go:902`），因此实现时必须将二者解耦——在 `submitPodGroupAlgorithmResult`
中单独构造 condition 的 `Message`，保持组级 status 的消息不变。否则一个数 KB 的节点表会按 pod 数
复制进上百个 pod 的 condition 与事件，既超出[消息体积预算](#消息体积预算)只对单层消息做预算的
假设，也会削弱事件按 message 去重的效果。

### 阶段 1：诊断构建器

`pkg/scheduler/schedule_one_podgroup.go` 中新增的非导出 helper：

- `algorithmResult.notEvaluated bool`，由 `completePodGroupAlgorithmResult` 在其合成的条目上置位，
  对应 R2。
- `classifyNodes(podResults []algorithmResult) nodeClassification` —— 遍历 `podResults` 一次，为
  每个节点建立「接收计数」与「拒绝原因直方图」，随后将节点划分为 saturated / excluded 两类。
  这是节点维度聚合的核心，规则见[聚合规则](#聚合规则)。`numAllNodes` 由各失败 pod 的
  `FitError.NumAllNodes` 取最大值，不作为参数传入，以避免重读 snapshot。
- `nodeClassification` 的方法 `saturatedNodes()`、`nodeReason(node)`、`excludedCount()` 与
  `excludedNodeHistogram()` —— 分别给出按接收数降序、名称升序排列的饱和节点，某节点的代表性
  原因，排除节点数，以及按 `reasonKey` 折叠的排除节点直方图。直方图的桶内计数为**不同节点数**。
- `dominantReason(counts, text)` —— 在多个原因中取出现次数最多者，并以键序打破平局，使渲染在
  观测到相同集群状态的各周期之间稳定。该函数同时服务于「某节点的代表性原因」与「缺席余量的
  代表性原因」两处选择。
- `reasonKey(status *fwk.Status) string` —— 聚类键。阶段 1 采用 `(status.Plugin(), reason)`
  复合键；`Plugin()` 已由 framework 在三种拒绝 code 上填充完毕，无需插件配合。阶段 2 落地后
  改为优先取 `RejectionCode()`，其为空时回退至该复合键。
- `(*Scheduler).buildPodGroupDiagnosis(podGroupResult) string` —— 组装消息，返回空串表示不渲染、
  由调用方保留现有消息。只存在一种布局（节点表），不按 gang 大小分支；逐 pod 的细节保留在各
  pod 自身的 condition 中。`waitingOnPreemption` 与 `hasExtenderFilters()` 两项前置条件在此处
  判定。
- `truncateDiagnosis(message) string` 与 `countLabel(count, noun)` —— 前者兜住消息预算的硬上限并显式
  标注截断，后者处理直方图行的单复数。

`classifyNodes` 是唯一需要完整结果集的 helper；其余均作用于单个节点或单个桶，因此可以独立
进行单元测试。`buildPodGroupDiagnosis` 只依赖 `Scheduler.Extenders`，可用一个不含 client 与
queue 的空 `Scheduler` 直接测试。

阶段 1 **不包含**任何资源算术：不引入 `nodeResourceUsage`，不读取 `snapshot.NodeInfos()`，
也不调用 `CalculateResource()`。聚合器只执行三项操作——计数、分组、渲染。

#### 聚合规则

节点分类所需的两个计数并不等价，其中一个是常量。

**拒绝计数恒等于失败 pod 数。** 一个以 `FitError` 结束的 pod 不存在可行节点，因此 placement 内
的每个节点对它而言，要么在 `NodeToStatus` 中带有显式拒绝，要么落入 absent 分支同样得到拒绝。
于是对任意节点 `v` 都有 `rejected(v) = N_failed`，无需逐 pod 逐节点累加。分类由此退化为一个
判断：`accepted(v) > 0` 即为饱和，否则为排除。`accepted` 直接来自 pod 到 node 的放置映射，代价
为 O(已放置 pod 数)。

**节点总数不需要重读 snapshot。** `FitError.NumAllNodes` 已在调度周期内捕获
（`schedule_one.go:590`），排除节点数即 `numAllNodes` 减去饱和节点数。这一点是必要的：
`submitPodGroupAlgorithmResult` 在算法返回之后执行，此时重新调用 `ListNodesInPlacement` 读到的
可能已是下一个周期正在变更的 snapshot。

**原因归属按显式条目与缺席余量分别处理。**

- 以节点名为键，将各失败 pod 的显式条目并入同一张 map：对每个失败 pod 调用
  `ForEachExplicitNode`（`pkg/scheduler/framework/interface.go:89`），累加 `explicitCount(v)` 并将
  其 reason 计入 `v` 的直方图。总代价为 `Σ Len_i`，严格小于该周期已经完成的 Filter 评估次数。
- map 之外的节点数为 `numAllNodes - len(map)`。这些节点对每一个失败 pod 都是缺席的，其
  `absentCount(v) = N_failed`，原因统一取自各 pod 的 `absentNodesStatus`，因此整个余量折叠为一个
  桶，代价 O(1)。其中若有节点曾接收过 gang pod，则按放置映射改列入饱和节点，不进入该桶。

因此 `classifyNodes` 的复杂度为 **O(Σ Len_i + pod 数)**，而非 O(pod 数 × 节点数)。

**一个已知近似。** 若某节点对一部分 pod 是显式拒绝、对另一部分 pod 是缺席，其完整直方图还需要
将 `N_failed - explicitCount(v)` 归入缺席原因。同构 gang 下缺席原因唯一，该归并为 O(1)；异构
gang 下各 pod 的 PreFilter 候选集不同，缺席原因不唯一。这一情形正是阶段 3 按 `PodSignature`
分区所要消除的。阶段 1 取该节点直方图中出现次数最多的原因作为代表，并在 `classifyNodes` 的
doc comment 中声明这是近似。

#### 不支持 extender 场景

上述规则依赖「失败 pod 的每个节点都带有可归因的拒绝原因」，而 filter extender 会破坏该前提。
`numNodesToFind` 只在既无 extender filter 又无 scoring 时才被压成 1（`schedule_one.go:788`），
因此在配置了 filter extender 的集群中，扫描仍会在找到 `numFeasibleNodesToFind` 个可行节点后
cancel（`:826-828`），随后 `findNodesThatPassExtenders`（`:700`、`:894`）可能将这些可行节点全部
剔除，pod 依旧以 `FitError` 结束。此时未被扫描的节点不在 `NodeToStatus` 中，会落入默认的
`absentNodesStatus`——`UnschedulableAndUnresolvable` 且**不带 reason**——在节点表中表现为一个
空原因的排除桶，并让从未见过该 pod 的节点承担拒绝计数。

阶段 1 的处理方式是显式退出而非降级渲染：`buildPodGroupDiagnosis` 以 `sched.hasExtenderFilters()`
（`schedule_one.go:769`）为前置条件，该函数返回真时不渲染节点表，保留现有的组级 status 消息。
extender 不参与 framework 的 reason 体系，在不为其引入稳定拒绝标识的前提下，任何渲染都会误导。
若日后需要支持，其前提与阶段 2 的 `RejectionCode` 相同——先让拒绝原因变得机器可读。

### 阶段 2：`RejectionCode`

为 `fwk.Status` 增加一个字段，作为拒绝原因的机器可读、稳定标识。

`staging/src/k8s.io/kube-scheduler/framework/interface.go`：

```go
// RejectionCode is a stable, machine-readable identifier for why a plugin rejected
// a node. Unlike the human-readable reasons, it is safe to group and compare on.
// The empty value means the plugin did not classify its rejection.
type RejectionCode string

const (
	RejectionInsufficientResource RejectionCode = "InsufficientResource"
	RejectionUntoleratedTaint     RejectionCode = "UntoleratedTaint"
	RejectionNodeAffinity         RejectionCode = "NodeAffinityMismatch"
	// ...
)

func (c RejectionCode) For(r v1.ResourceName) RejectionCode

func (s *Status) WithRejectionCode(c RejectionCode) *Status // 可链式，nil 安全
func (s *Status) RejectionCode() RejectionCode              // nil 安全，返回 ""
```

资源类的拒绝以资源名限定 code，例如 `"InsufficientResource:cpu"`。四种常见情况使用包级常量，
因此热路径上只是赋一个预构造好的 string header，零分配：

```go
var (
	rejectionInsufficientCPU    = fwk.RejectionInsufficientResource.For(v1.ResourceCPU)
	rejectionInsufficientMemory = fwk.RejectionInsufficientResource.For(v1.ResourceMemory)
	// ...
)
```

标量资源需要一次拼接，但 `fit.go` 本来就为它们支付了一次 `fmt.Sprintf`，因此增量开销被当前
已有的开销所限定。

`noderesources` 迁移：`fit.go` 保留 `Reason` 字符串用于人类可读消息，额外设置 code：

```go
return fwk.NewStatus(statusCode, failureReasons...).
	WithRejectionCode(insufficientResources[0].RejectionCode)
```

`InsufficientResource` 增加一个 `RejectionCode` 字段，在 `fitsRequest` 中以预构造常量填充。
多种资源同时不足时，第一个作为主 code；完整的人类可读列表仍留在 `reasons` 中。

聚合器随后：

- 将**排除节点**按 `RejectionCode` 归并成桶，而非按消息串归并——桶的键稳定，不会因为插件在
  reason 中插入节点名而退化为一节点一桶；
- 对**饱和节点**，以其 `InsufficientResource:<r>` code 稳定地指明被耗尽的资源类型。本期仅
  打印资源名；待阶段 4 引入资源数字之后，再以该 code 查取对应的数值行；
- 当 `RejectionCode()` 为空时回退至 `(plugin, reason)` 复合键，未迁移的插件优雅降级而非
  直接失效。

**开销。** 每个 `Status` 增加 16 字节。`Status` 本来就是每个被拒节点每个 pod 各分配一个，
且本来就持有一个 `[]string`（24 字节）及其底层数组，因此相对增幅很小，且没有新增分配。

**影响范围。** staging framework 包中一个增量字段加两个方法；迁移一个插件（`noderesources`）
以验证模式；其他插件增量选择加入。现有行为完全不变，未设置 rejection code 的 `Status` 与
当前表现一致。

### 阶段 3：按 `PodSignature` 分区

异构 gang 与交错两项限制来自同一个根因：一张节点表中混入了彼此不可比较的 pod。而调度器
**已经具备**修复它所需的等价关系。

`fwk.PodSignature`（`staging/src/k8s.io/kube-scheduler/framework/interface.go:765`）由
`frameworkImpl.SignPod`（`pkg/scheduler/framework/runtime/framework.go:884`）从各插件的
`SignFragment` 构造。其契约恰好就是诊断所需要的：

> Any two pods with the same signature should get the same feasibility and scoring for
> the same set of nodes in the same state.

关键在于，它**已经挂在聚合器手上的那个对象上**：`QueuedPodInfo.PodSignature`
（`pkg/scheduler/framework/types.go:700`），由调度队列的 `signPod`
（`pkg/scheduler/backend/queue/scheduling_queue.go:2185`）填充，而 `algorithmResult.podInfo`
**就是**一个 `*framework.QueuedPodInfo`。零新增管道。

按 signature 对 gang 的 pod 分区，随后为每个 signature 类渲染一张节点表：

```
Gang scheduling failed: 30/100 pods placed, minCount=80. 2 pod shapes.

Shape 1 (80 pods, e.g. worker-0: cpu=4, mem=8Gi):
  Saturated nodes (3):
    node-A: accepted 15, rejected 50 (Insufficient cpu)
    node-B: accepted 10, rejected 50 (Insufficient cpu)
    node-C: accepted  5, rejected 50 (Insufficient cpu)
  Excluded nodes (47): 35 untolerated taint, 12 node affinity mismatch

Shape 2 (20 pods, e.g. ps-0: cpu=8, mem=16Gi, nvidia.com/gpu=1):
  Excluded nodes (50): 50 Insufficient nvidia.com/gpu
```

需要说明的是，shape 标注中的 `cpu=4, mem=8Gi` 是**代表性 pod 自身的请求量**，来自
`PodInfo.CalculateResource()`，与阶段 4 所推迟的**节点级资源总量**并非同一项数据。它既不涉及
snapshot 读取，也不涉及任何需要与插件计账规则对账的算术，因此不受本期范围调整的影响。

该阶段改善两件事：

1. **行变得无歧义。** 同一个类内部，feasibility 按定义是一致的，因此某节点的拒绝原因是节点的
   属性，而非混合 pod 形状造成的假象。每行的原因直方图坍缩为单一原因。
2. **饱和推断变得确定。** 在混合表中，「先接收后拒绝」只能**概率性地**推出是 gang 消耗——一个
   异构 gang 完全可能先接收一个小 pod，然后因为一个**只对大 pod 成立的结构性原因**拒绝了大
   pod。在 signature 类内部这不可能发生：feasibility 相同意味着 pod `i` 与 pod `j` 之间唯一
   变化的就是 state，而该 state 变化就是 gang 自身的消耗。整个节点维度设计所依赖的那个论断，
   **由推断变为定理**。

shape 数量本身就是诊断信息。它同时解决了 `schedule_one_podgroup.go:878` 挂着的那条 TBD：

> TBD: Add a message to status if the pod used features for which finding a placement
> cannot be guaranteed, such as heterogeneous pod group or using inter-pod dependencies.

一个具有 N > 1 个不同 signature 的 gang **就是**异构 pod group，不需要任何新的启发式判断。

实现上增加一个 helper，作用在 `classifyNodes` **之上**：

- `partitionBySignature(podResults []algorithmResult) []shapeClass` —— 按
  `podInfo.PodSignature` 分组，保留评估顺序以便序号命名，将 signature 为 nil 的 pod 收进
  `unclassified` 类；当只有一个不同 signature 或一个都没有时，返回单个无名类。

`buildPodGroupDiagnosis` 随后对每个类各执行一遍阶段 1 的流水线，而非整体执行一遍。阶段 1 的
任何 helper 都不需要改签名或改行为。

该阶段之所以不并入阶段 1，原因有四：

- **signature 是可选的。** `OpportunisticBatching` 门控关闭时 `SignPod` 返回 nil
  （`pkg/features/kube_features.go:837`）；且只要**任何一个**已启用的 Filter/Score 插件未
  实现 `SignPlugin`，`enableSignatures` 就被整体关闭（`framework.go:868`）。因此分区必须
  是严格机会性的：nil 或单 signature 的 gang 原样回退到阶段 1 的表。
- **signature 是不透明的。** `PodSignature` 是 marshal 出来的 JSON blob，不是标签。消息需要
  短句柄，因此构建器按评估顺序分配序号名（`Shape 1`、`Shape 2`），并打印一个代表性 pod 名及其
  资源请求作为人类可读的注解。该渲染是本阶段唯一真正新增的代码。
- **基数需要上限。** 病态情况下可能一个 pod 一个 signature。构建器至多渲染 3 个类，超出部分
  显示 `... and N more shapes`；当类数接近 pod 数时回退到不分区的表，这本身就是「signature
  未在有效聚类」的信号。
- **陈旧性。** `QueuedPodInfo.Update` 会清空缓存的 signature
  （`pkg/scheduler/framework/types.go:711`），因此中途被更新的 pod 可能 signature 为 nil 而
  同伴不是。signature 为 nil 的 pod 收进一个 `unclassified` 桶，而不是将整个 gang 拖回平表。

这些都不阻塞阶段 1：分区是架在 `classifyNodes` **之上**的一层分组，后者继续对交给它的任意
pod 子集工作。

### 消息体积预算

`metav1.Condition.Message` 的上限为 32 768 字符
（`staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/types.go:1693`）。构建器执行一个远低于
该上限的预算：

- 饱和节点：至多 10 行，超出显示 `... and N more saturated nodes`；
- 排除节点：至多 5 个原因桶，超出显示 `... and N more reasons`；
- 组级提前中止说明：至多一行；
- signature 类（阶段 3）：至多 3 个，超出显示 `... and N more shapes`。

截断在文本中是显式的，因此被截断的消息绝不会读起来像完整的。节点维度聚合使该预算很容易守住：
饱和行数被已放置 pod 数所限，排除段按原因桶折叠后同样有界。

上述上限之外另有一道兜底：`truncateDiagnosis` 以 16 384 字符为硬上限，超出部分截断并追加
`... diagnosis truncated`。该值取上限的一半，为 #141860 逐层嵌套的前缀以及插件可能产生的超长
reason 文本留出余量。

此外，需要为 [#141860](https://github.com/kubernetes/kubernetes/pull/141860) 的传播机制保留
余量。该 PR 会为子组消息添加 `ancestor composite pod group "X" is unschedulable: ` 前缀并
逐层嵌套，CompositePodGroup 的深度上限为 4。每层前缀约为 60 至 80 字符，最坏情况下合计约
320 字符，相对预算可以忽略；但构建器的上限应当以「本层消息」为计算基准，而不应以嵌套完成后
的总长度为基准。

### 准确性约束

以下五项必须写入相应 helper 的 doc comment 与消息措辞，否则诊断会产生误导：

- **节点采样。** `findNodesThatPassFilters` 一旦找到 `numFeasibleNodesToFind` 个可行节点即
  停止（`pkg/scheduler/schedule_one.go:787`、`:827`）。对在 framework 内所有节点上都失败的 pod
  而言这无关紧要——所有节点都被评估了——但对一个成功的 pod，`NodeToStatus` 是不完整的，因此
  聚合器只从**失败**的 pod 构建拒绝统计。该规则存在一个例外，见[不支持 extender 场景](#不支持-extender-场景)。
- **缺席节点按节点名求并集，而非按 pod 累加乘数。** `FitError.Error()` 中的
  `absentNodesStatus × (NumAllNodes - Len())`（`pkg/scheduler/framework/types.go:1495`）统计的是
  **单个 pod** 的「原因 → 节点数」。若将其跨 pod 累加，得到的是 (节点, pod) 对的数量而非不同
  节点的数量：当多个 pod 在 PreFilter 阶段即失败时（`schedule_one.go:648` 只调用
  `SetAbsentNodesStatus`，map 为空、`Len() == 0`），每个这样的 pod 都会把 placement 内全部节点
  各计一次拒绝，70 个失败 pod × 5000 节点将渲染为 `Excluded nodes (350000)`；同一节点还可能
  同时落入饱和类与排除类。正确的聚合规则见[聚合规则](#聚合规则)。
- **TAS placement。** 在 `TopologyAwareWorkloadScheduling` 下，上报的是任意一个 placement
  的结果（`schedule_one_podgroup.go:1041` 取 `anyResult`）。消息必须说明它描述的是哪一个
  placement，否则用户会将单个 placement 的失败读成全局失败。
- **不得暗示资源数字。** 本期不输出资源量，措辞亦不得暗示。`accepted 15, rejected 70
  (Insufficient cpu)` 所陈述的是「该节点先接收后拒绝，拒绝原因为 cpu」，而非「cpu 的使用量为
  多少」。任何形如 `60/64` 的注解均须等待阶段 4。
- **等待抢占不是终局失败。** `submitPodGroupAlgorithmResult` 对「等待抢占」与「确实不可调度」
  写入的是同一个 `PodGroupInitiallyScheduled=False` / `Reason=Unschedulable` condition，两个分支
  目前只体现在日志上（`schedule_one_podgroup.go:898-909`）；抢占路径还会将 PodGroupPostFilter 的
  结果追加进组级 status（同文件 `:799`）。若在两种情形下都渲染 `Gang scheduling failed: ...` 加
  节点表，就会在抢占仍可能成功时给出终局失败结论与「扩容 / 放宽约束」的处置建议，并覆盖掉现有
  的抢占信息。因此诊断以 `podGroupResult.waitingOnPreemption`（同文件 `:543`）为前置条件：该字段
  为真时不渲染节点表，保留现有消息。

### Feature gate 与兼容性

改动为纯增量，不修改任何 API 类型。

- 阶段 1 与阶段 3 不引入 framework 表面，因此不需要 feature gate。更丰富的 condition 消息
  可以搭乘已有的 `GenericWorkload` gate，因为它只在 PodGroup 路径上产生。
- 阶段 2 增加的 framework 字段本身亦不需要 gate：未设置该字段的 `Status` 与当前行为完全一致。
- 阶段 1 会在当前 PodGroup condition 的消息**之后追加**节点表，组级 status 的 `Message()` 原样
  保留为首行，且仅在 `waitingOnPreemption` 为假时追加。依赖该文案首行的下游不受影响；解析整段
  消息的下游需要一并评估，参见[与其他工作的关系](#与其他工作的关系)。

### 可扩展性

- **热路径开销为零。** 全部计算仅在失败路径上执行。阶段 1 不修改 framework；阶段 2 每个
  `Status` 增加 16 字节，且无新增分配。
- **失败路径的复杂度。** `classifyNodes` 的复杂度为 O(Σ Len_i + pod 数)，推导见
  [聚合规则](#聚合规则)。即使按 O(pod 数 × 节点数) 估算，该代价亦不致命：同一周期内
  `RunFilterPluginsWithNominatedPods` 已经在同等数量的 (pod, 节点) 对上执行过完整的 Filter
  插件链，一次 map 查找与之相比可以忽略；且它只在 gang 失败时发生一次，而非每 pod 每节点发生。
  聚合在 pod 循环内增量完成，不引入独立的异步阶段。诊断输入（各失败 pod 的 `NodeToStatus` 与
  放置映射）存活于 `podResults`，在 `submitPodGroupAlgorithmResult` 返回后即被丢弃；改为异步
  计算需要先将全部输入复制出来，其代价高于分类本身。`updatePodGroupCondition`
  （`schedule_one_podgroup.go:943`）本就是一个 gang 一个周期只调用一次的唯一 patch 点，因此
  「在最后一次 patch 时汇总」即为现状，无需额外机制。
- **消息基数有界。** 5000 节点集群上的 100 pod gang 渲染为至多 10 行饱和节点与至多 5 个原因
  桶。集群规模不进入消息长度。

### 可观测性与排查

诊断消息将两类节点与两种处置方向一一对应，并以一行组级说明覆盖提前中止的情形，运维可据此
直接决策：

| 观测到的形态 | 结论 | 处置方向 |
|---|---|---|
| 饱和节点占多数 | gang 自身消耗尽了可用容量 | 扩容、使用更大的节点、或降低 `minCount` |
| 排除节点占多数 | 节点对该 gang 自始不可用 | 检查污点、亲和性、选择器与既有负载；扩容无效 |
| 节点表为空且存在未评估 pod | gang 经由 `PlacementFeasible` 提前中止 | 检查 `minCount` / `minGroupCount` 是否可满足 |

诊断**不**回答的问题，需要由其他层级承担：单个 pod 为何在某个节点上被拒，见该 pod 自身的
condition；哪个子组导致 CompositePodGroup 失败，见 #141860 的层级传播。

### 与其他工作的关系

[#141860](https://github.com/kubernetes/kubernetes/pull/141860) 实现了失败原因沿
CompositePodGroup 层级的传播，其正文声明 `Fixes #141025`。该 PR 与本提案分别位于三层分工表的
第三行与第二行，两者并不重叠，但需要明确边界，以免 issue 在该 PR 合并之后被关闭，使第二行
失去载体。

- **维度不同。** #141860 处理纵向关系，即某个组被哪一个祖先阻挡；本提案处理组内的横向关系，
  即该组为何未能凑齐 `minCount`。
- **数据来源不同。** #141860 的整个 patch 中不出现 `NodeToStatus`、`SuggestedHost`、
  `InsufficientResource`、`PodSignature` 或 `RejectionCode`，即它不读取任何节点维度或资源
  维度的数据，因此不会与本提案在实现上产生冲突。
- **它并未覆盖 R2。** #141860 为**未被评估的子组**合成 `group was not evaluated`，或令其继承
  `failedAncestor` 的状态，但未修改 `completePodGroupAlgorithmResult`，因此 **pod 级**的
  「未被评估」信号在其合并之后依然不存在。这一点反而为本提案的 R2 提供了先例：同一概念在组级
  已被接受，pod 级只是同一处理方式的延伸。
- **消息会被再次包装。** 本提案产出的 PodGroup 消息将成为 #141860 所加前缀的内层内容，
  体积预算已为此保留余量。

其他相关工作：

- [#138991](https://github.com/kubernetes/kubernetes/issues/138991) 提议单 pod 的机器可读
  诊断，并明确排除 gang 场景。阶段 2 的 `RejectionCode` 位于 framework 层，两者可以消费同一
  份数据。
- [#140670](https://github.com/kubernetes/kubernetes/pull/140670) 承载 CompositePodGroup
  的 condition patch，是本提案第三层分工的实现。
- [kubernetes-sigs/lws#1056](https://github.com/kubernetes-sigs/lws/issues/1056) 正在引用
  当前的 gang 失败文案（`minCount (2) cannot be satisfied`）作为论据，讨论
  `RecreateGroupAfterStart` 在原生 gang 语义下的行为。该 issue 的方案主张读取 `PodScheduled`
  condition 而非解析消息文本，因此不受本提案影响；但这印证了「消息文本不应被当作契约」这一
  判断，也进一步支持阶段 2 引入稳定的机器可读标识。

---

## 备选方案

### 备选方案一：在 `Status` 上挂完整的结构化详情

让插件挂一个带类型的载荷：

```go
type Status struct {
	// ...
	details []RejectionDetail // 或 interface{} / any 类型的逃生舱
}

type RejectionDetail struct {
	Code      RejectionCode
	Resource  v1.ResourceName
	Requested int64
	Used      int64
	Capacity  int64
}
```

表达力最强：数字来自真正做决策的那个插件，因此不存在聚合器的推导与插件的算术产生偏差的风险
（DRA extended resources、pod-level resources、in-place vertical scaling 都有微妙的计账
规则，任何推导都必须一一镜像）。该风险正是本期将资源数字整体推迟的原因；若阶段 4 最终确认
推导无法覆盖全部计账规则，本方案的优势将重新变得重要。

**开销。** 这个版本才配得上 review 中所说的 "non-trivial"。`Filter` 每 pod 每节点执行一次；
5000 节点集群中一个不可调度的 pod 会产生多达 5000 个 `Status` 对象，每个现在都要携带一个
48 字节结构体的切片。一个 100 pod 的 gang 即产生 50 万条详情记录，并一直保留至周期结束。
要使其可负担就需要一套门控机制——一个穿过 `CycleState` 的 "diagnostics enabled" 标志，或者
一套采样策略——而该门控本身就是新的 plugin-model 表面，每个 Filter 插件都必须遵守。

**影响范围。** staging 包新增类型、每个插件都必须遵守的门控契约，以及门控开启与关闭两种模式
下的行为差异（测试必须双向覆盖）。

### 备选方案二：按需的 `Explain` 扩展点

增加一个只在失败路径上调用的可选接口：

```go
type ExplainPlugin interface {
	Plugin
	// Explain returns structured detail about why this plugin would reject the pod
	// on this node. Called only when building a diagnosis, for a bounded sample.
	Explain(ctx context.Context, state fwk.CycleState, pod *v1.Pod, nodeInfo fwk.NodeInfo) RejectionDetail
}
```

热路径零开销：成功周期与普通失败周期均不发生变化。诊断构建器挑选一个有界的（失败 pod，拒绝
节点）样本集，请插件解释。

**致命问题。** 到构建诊断时，`revertFns.revert()` 已经将所有试探性 assume 撤销。在 revert
之后的 snapshot 上重跑 `Explain`，回答的是另一个问题：`pod-3` 在 `node-A` 上被拒**正是因为
pod-1 与 pod-2 当时被试探性放在了那里**，而 revert 之后它们已不在，因此 `Explain` 很可能报告
该 pod 放得下。诊断会自相矛盾。

要修复就必须二选一：将 revert 推迟至诊断之后（使集群状态为一个上报需求充当人质，并改变
`revertFns` 的 LIFO 契约含义），或者为了诊断重新 assume 这些放置（在失败路径上重跑带副作用的
Reserve 插件）。两者都比问题本身更糟。

存在一个可行的窄版本——在拒绝发生时即调用 `Explain`，并以标志门控——但那实际上就是多绕一层
的备选方案一。

### 对比与选择理由

| | 本提案（原因聚类 + `RejectionCode`） | 备选方案一（结构化详情） | 备选方案二（按需 `Explain`） |
|---|---|---|---|
| 热路径分配 | 无 | 每个被拒节点一个切片 | 无 |
| `Status` 体积 | +16 B | +24 B 起 | 不变 |
| 本期是否输出资源数字 | **否**（推迟至阶段 4；若实现，为推导值，可能与插件算术存在偏差） | 是，来自插件 | 是，但针对的是错误的集群状态 |
| revert 之后仍正确 | 是 | 是 | **否** |
| 是否需要门控机制 | 否 | 是 | 否 |
| 必须改动的插件 | 起步 1 个，其余可选加入 | 最终所有 Filter 插件 | 任何希望参与的插件 |
| 是否同时服务 #138991 | 是 | 是 | 是 |

本提案是唯一既能填补 R5、又不引入热路径回退（备选方案一）或正确性风险（备选方案二）的方案。

在将资源数字推迟之后，本提案原先唯一的弱点——推导所得的数字可能与插件的计算产生偏差——在本期
已不存在：阶段 1 与阶段 2 不输出任何数字，只执行计数与分组。该弱点在后续确实引入数字时依然是
可控的：

- 推导覆盖了用户真正会询问的那些资源（cpu、memory、ephemeral-storage、pods、标量资源），它们
  均已被 `NodeInfo.Requested` 计入；
- 当聚合器无法为某个 code 推导出数字时，它只打印 reason 而不附加数字注解，不进行猜测；
- 若未来确实出现需要由插件提供数字的场景，`RejectionCode` 与「此后再增加一个可选详情载荷」是
  前向兼容的——本提案是备选方案一的子集，而非竞争方向。

### 其他被否决的做法

- **解析 reason 字符串。** 零 framework 改动，对当前的 `noderesources` 有效。否决理由：消息
  文本不属于任何兼容性契约，且对任何将变量插入 reason 的插件都会失效。阶段 1 以
  `(plugin, reason)` 复合键缓解该问题，但仅作为阶段 2 落地之前的过渡手段，并在测试中显式
  固定其退化行为。
- **在 `CycleState` 上开一条旁路。** 插件将结构化详情写入一个约定的 `CycleState` key，而不是
  写入 `Status`。否决理由：`CycleState` 是 per-pod-per-placement 的，且在 revert 时被丢弃，
  因此数据仍然必须在 `Status` 已经流经的那些点上被拷出——同样的结果，更多的管道。
- **以 Event 而非 condition 承载诊断。** Event 上限接近 1 KiB，且会被聚合与去重，恰好摧毁了
  这条消息存在的意义所在的节点级细节。condition 才是正确的归宿；Event 可以携带一个指向它的
  指针。
- **按 gang 大小提供逐 pod 与聚合两种布局。** 否决理由见
  [单一渲染器与三层分工](#单一渲染器与三层分工)：逐 pod 行在大集群上无法真正列出节点，只能
  退化为原因直方图，因此双模式换不来体积收益；且逐 pod 视图与 pod 自身的 condition 重复。
- **以 pod 失败签名作为主聚合维度。** 即 issue 原文中的 `Failed pods (70), by failure
  signature` 一段。否决理由：`45 pods: Insufficient cpu` 不说明失败发生在何处，无法映射到
  任何补救动作；且该维度与节点维度描述的是同一批观测，合并至节点表严格更有信息量。

---

## 测试计划

### 阶段 1

- `classifyNodes` 的表驱动测试，覆盖每个类别与棘手情况：先接收后拒绝的节点（saturated）；
  全部拒绝的节点（excluded）；map 中无条目、仅由 `absentNodesStatus` 覆盖的节点；在一个节点
  上交错拒绝／接收／拒绝的异构 gang；在任何 pod 被评估之前即被 `PlacementFeasible` 中止、因而
  节点表为空的 gang。
- `reasonKey` 的表驱动测试：同一插件的同一 reason 归入同一桶；不同插件产生的相同文案分入两个
  桶；`Plugin()` 为空时退化至裸 reason 且不发生 panic。
- `classifyNodes` 的计数回归测试：构造多个在 PreFilter 阶段即失败的 pod（`NodeToStatus` 为空、
  `Len() == 0`），断言排除节点数等于 `numAllNodes` 减去饱和节点数，而**不是**失败 pod 数与节点
  数的乘积。该用例固定[聚合规则](#聚合规则)中按节点名求并集的要求，防止日后退回到
  `FitError.Error()` 的按 pod 乘数写法。
- `excludedNodeHistogram` 将 `absentNodesStatus` 余量折叠为**一个**桶，桶内计数为不同节点数而非
  (节点, pod) 对数；同时断言一个既接收过 pod 又拒绝过 pod 的节点只出现在饱和类中，不会同时
  出现在排除类中。
- `buildPodGroupDiagnosis` 在 `hasExtenderFilters()` 为真时返回空，调用方保留现有的组级 status
  消息，见[不支持 extender 场景](#不支持-extender-场景)。
- `waitingOnPreemption` 为真时不渲染节点表，且现有的抢占信息不被覆盖。
- 断言诊断消息只出现在 PodGroup condition 中：同一 gang 的各 pod 的 `PodScheduled` condition 与
  `FailedScheduling` 事件消息保持为原有的组级 status 消息，不含节点表。
- 增加一个将节点名插入 reason 的插件用例，断言阶段 1 下它确实退化为一节点一桶。该用例的作用是
  将已知限制固定在测试中，而非假定其不存在；阶段 2 落地后，该用例改为断言折叠恢复。
- `schedule_one_podgroup_test.go` 中增加一个部分放置的 gang 用例，断言完整渲染出的 condition
  消息；再增加一个提前中止的 gang 用例，断言 `notEvaluated` 的措辞。

### 阶段 2

- `RejectionCode.For` 与 `Status.WithRejectionCode` 往返的单元测试，包含 nil `Status`
  接收者。
- 断言 `noderesources.Filter` 对 cpu、memory、ephemeral-storage、pods 与一个标量资源设置了
  预期的 code，且人类可读消息未变（防止现有测试中出现意外的消息回归）。
- 以带 code、不带 code、混合三种输入重跑 `excludedNodeHistogram` 的表，证明插件增量迁移期间
  回退路径仍然有效。
- 在大节点集上对比 `RunFilterPlugins` 改动前后的 benchmark，证明热路径未受影响。

### 阶段 3

- `partitionBySignature` 的表驱动测试：全部为 nil signature 时坍缩成一个类；只有一个不同
  signature 时渲染为不分区；两个 shape 正确分区；signed 与 nil 混合时 nil 的进入
  `unclassified` 而不干扰其余；一个 pod 一个 signature 时触发回退到平表。
- 一个 integration 风格的用例：关闭 `OpportunisticBatching` 时，输出与阶段 1 **逐字节相同**，
  以此证明分区确实是机会性的。
- 一个异构 gang 用例：断言分区之后每个饱和行的原因直方图坍缩为单一原因——这是「行变得无
  歧义」这一论断的可观测证据。

### 阶段 4

- `nodeResourceUsage` 的表驱动测试，针对一个含既有 pod 的 snapshot 断言「基线加增量」的算术，
  证明 revert 之后没有重复计算。
- 针对 DRA extended resources、pod-level resources 与 in-place vertical scaling 各增加一个
  用例，断言推导值与对应插件的判定一致；若无法保证一致，则断言不输出数字。

---

## 后续工作

### 阶段 4：节点资源数字（未排期）

本节保留被移出本期范围的资源数字推导。保留的理由是：该推导本身成立，且其正确性论证依赖于
调度器现有的 revert 时序，论证过程不易重建。将推导与推迟理由一并记录于此，可以使阶段 4 在
启动时无需重新分析。

#### 推迟的理由

节点资源数字是整份提案中唯一需要向 maintainer 论证「聚合器的算术与各插件的计账规则保持一致」
的部分。`NodeInfo.Requested` 对以下情形的计账规则各不相同：

- DRA extended resources；
- pod-level resources；
- in-place vertical scaling。

任何一处偏差都会导致诊断输出一个看似权威、实则错误的数字，而错误数字的危害大于缺失数字。
与此同时，节点分类的结论并不依赖这些数字（见
[分类不依赖 reason 文本](#分类不依赖-reason-文本)），因此本期不实现该项不会削弱诊断的核心
价值。

阶段 4 若要启动，需要一次以计账正确性为主题的独立 review，并逐一覆盖上述三类资源。若届时
确认推导无法覆盖全部计账规则，则应改为
[备选方案一](#备选方案一在-status-上挂完整的结构化详情)。

#### 推导

对每个接收了至少一个 gang pod 的节点 `N`：

```
allocatable(N)  = snapshot.NodeInfos().Get(N).GetAllocatable()
baseline(N)     = snapshot.NodeInfos().Get(N).GetRequested()
gangDelta(N)    = Σ podResult.podInfo.CalculateResource()，对所有 SuggestedHost == N 的已放置 pod
used(N)         = baseline(N) + gangDelta(N)
```

三条性质保证其正确性：

1. **revert 已经完成。** `runRootSchedulingAlgorithm` defer 了 `revertFns.revert()`
   （`schedule_one_podgroup.go:469`），它会 unreserve 并 forget 每一个被试探性 assume 的
   gang pod。`submitPodGroupAlgorithmResult` 在其之后运行，因此
   `snapshot.NodeInfos().Get(N).GetRequested()` 正是 gang 之前的基线——恰好是需要叠加 gang
   自身贡献的那个量，不存在重复计算。
2. **`Allocatable` 不受 assume/forget 影响。** 它来自 Node 对象。
3. **Pod 的请求量已被缓存。** `PodInfo.CalculateResource()`
   （`pkg/scheduler/framework/types.go:1376`）会 memoize 至 `pi.cachedResource`，而调度周期
   早已填充该字段。求和的复杂度为 O(已放置 pod 数)，无需重算。

因此 placement 叙事——`node-A: 15 pods (cpu: 60/64)`——的代价为：每个不同节点一次 snapshot
查找，加上每个已放置 pod 一次整数加法，且**仅在失败路径上**发生。

#### 与 `RejectionCode` 的先后关系

阶段 2 引入的 `InsufficientResource:<r>` code 是本节的前置条件：只有当聚合器能够确定某次拒绝
涉及哪一种资源时，才能从资源表中挑选对应的一行，而不是将 memory、pods 等全部打印。这也是
阶段 4 必须排在阶段 2 之后的原因。

---

## 缺陷

- **阶段 1 的聚类键仍可能退化。** 在插件将变量插入 reason 的情况下，`(plugin, reason)` 复合键
  无法阻止直方图退化为一节点一桶。该限制在阶段 2 落地之前持续存在，并已固定在测试中。
- **饱和节点行缺少量化信息。** 本期只能给出原因，无法给出资源量。运维可以据此判断属于容量
  问题，但仍需自行查看节点以确定缺口大小。
- **小 gang 下节点表可能显得冗长。** 3 个 pod 的 gang 至多产生 3 行饱和节点，而逐 pod 布局
  在此规模下更短。本设计以「单一渲染器、无布局分支」换取了该情形下的少量冗余。
- **消息是自由文本，非机器可读。** 结构化诊断属于 #138991 的范畴。本提案的 `RejectionCode`
  为其提供了可复用的基础，但 condition 消息本身仍不可作为契约解析。
- **提前中止的成因未加区分。** 组级说明将「节点表为空」归因于 `PlacementFeasible` 提前中止，
  但中止可能发生在 pod 循环之前（`schedule_one_podgroup.go:589`，此时不存在任何节点级观测），
  也可能发生在循环之内（同文件 `:635`，此时已有部分 pod 被完整评估）。两者的运维含义不同：
  前者说明约束在任何放置发生之前即不可满足，后者说明放置进行到一半才失去可行性。建议在实现时
  于组级说明中区分二者，并同时给出未评估的 pod 数。

---

## 实施历史

- 2026-07-29：issue [#141025](https://github.com/kubernetes/kubernetes/issues/141025) 提出
  PodGroup / CompositePodGroup 调度失败诊断不足的问题。
- 2026-09-04：[#141860](https://github.com/kubernetes/kubernetes/pull/141860) 提出
  CompositePodGroup 的层级状态传播，覆盖本提案三层分工中的第三层。
- 2026-09-22：本文档初稿，以「检验 review 意见」为叙事主线。
- 2026-09-22：依据 review 意见，将节点资源数字移出本期范围，仅保留按失败原因聚类；引入
  `(plugin, reason)` 复合键作为阶段 1 的聚类键；补充与 #141860 的分工界定；按 Kubernetes
  proposal 的常规结构重排全文。
- 2026-09-22：移除「未触及节点」类别。经核对 `findNodesThatPassFilters` 的提前终止条件与
  `NodeToStatus` 的 absent 语义，确认「接收 0，拒绝 0」的节点不可观测；节点分类收敛为饱和与
  排除两类，「未被评估」改由 pod 层级的 `notEvaluated` 标志与组级说明承载。论证见
  [不存在第三类节点](#不存在第三类节点)。
- 2026-09-22：依据 review 意见修正阶段 1 的聚合规格。缺席节点不再按 pod 累加
  `FitError.Error()` 的乘数，改为按节点名求并集，并给出
  `rejected(v) = N_failed` 的推导，复杂度收敛为 O(Σ Len_i + pod 数)，见[聚合规则](#聚合规则)。
  同时补充三项此前缺失的约束：filter extender 场景显式不支持；`waitingOnPreemption` 为真时不
  渲染节点表；诊断只写入 PodGroup condition，不得写入组级 status，以免按 pod 数扇出至各 pod 的
  condition 与事件。
- 2026-09-22：阶段 1 落地于 `pkg/scheduler/schedule_one_podgroup.go`，含 `classifyNodes`、
  `excludedNodeHistogram`、`reasonKey`、`buildPodGroupDiagnosis` 与 `truncateDiagnosis`，以及
  `algorithmResult.notEvaluated`；`submitPodGroupAlgorithmResult` 改为单独构造 condition 的
  `Message`，组级 status 保持不变。实现与本设计的两处差异：节点表**追加**在组级 status 消息
  之后而非替换它，以保留 PlacementFeasible 的原因并使 #141860 的祖先前缀仍附加于首行；
  `rejected` 统计失败 pod 而非全部 pod，因为 `SchedulePod` 会丢弃成功 pod 的诊断。
