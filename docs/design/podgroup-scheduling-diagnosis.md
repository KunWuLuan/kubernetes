# PodGroup 调度诊断的 Plugin-Model 支持

跟踪 issue：[kubernetes/kubernetes#141025](https://github.com/kubernetes/kubernetes/issues/141025)

## 动机

Issue #141025 提议为失败的 PodGroup / CompositePodGroup 调度提供一份聚合的、人类可读的
诊断信息。目标消息形如：

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

review 中 macsko 提出了阻塞性意见：

> implementing more advanced messages, e.g., your proposed CPU counting might require
> non-trivial changes in scheduler plugin model.

本文档检验这个论断：把目标消息拆解成它的数据需求，说明当前 plugin model 哪些能提供、
哪些不能，并对比三种填补方案。

### 目标

- 精确界定：要实现 #141025 的诊断，scheduler plugin model 究竟必须暴露什么。
- 提出一种在调度热路径上零开销或接近零开销的机制。
- 让该机制的价值不局限于 gang 调度——单 pod 诊断（#138991）应能消费同一份数据。

### 非目标

- 最终 condition 消息的具体措辞。布局**在**范围内，但仅限于它会改变 plugin model
  必须提供什么的部分——"按节点聚合"一节之所以存在，是因为聚合维度的选择决定了需要
  多少插件侧支持，而不是为了敲定文案。
- CompositePodGroup 的 condition patch。已由
  [#140670](https://github.com/kubernetes/kubernetes/pull/140670) 覆盖。
- 修改调度算法。本文只涉及上报。
- 把机器可读诊断做成 API 表面（#138991）。本文档所有内容都装在
  `metav1.Condition.Message` 里。

---

## 背景：当前的失败上报模型

### `fwk.Status` 只承载字符串

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

`Message()` 就是 `strings.Join(s.Reasons(), ", ")`。没有任何结构化载荷。插件名有记录，
但只有一个字符串。

### 插件算出了结构化数据，然后把它丢掉

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

`cpu: 60/64` 需要的一切就在这里。但 `Filter` 在 `fit.go:685` 把它压扁了：

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

`Requested`、`Used`、`Capacity`、`ResourceName` 从未离开插件。这个结构体唯一存活的地方
是 kubelet 准入路径（`pkg/scheduler/eventhandlers.go:859`），它直接调用导出的
`noderesources.Fits()`，绕开了 `Status`。

`Reason` 字段上的那句注释本身就是这条路径有多在意开销的证据：这个字段存在的唯一理由，
就是避免对每个被拒节点做一次 `fmt.Sprintf`。

### 聚合器今天已经能看到什么

每个失败 pod 的 `FitError` 都被保留了（`pkg/scheduler/framework/types.go:1439`）：

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
`map[string]*fwk.Status` 记录被拒节点，外加一个 `absentNodesStatus` 覆盖所有不在 map
里的节点。每个节点只记录**第一个**拒绝它的 Filter 插件的 status——`RunFilterPlugins`
在第一个非 success 时就返回（`pkg/scheduler/framework/runtime/framework.go:1126`）。

在 `submitPodGroupAlgorithmResult` 中，每个 `algorithmResult` 都带着 `podInfo`、
`scheduleResult.SuggestedHost` 和 `status`，所以聚合器手上有完整的 pod → node 放置映射
和每个 pod 完整的 `FitError`。

---

## 拆解目标消息

| # | 数据需求 | 今天可得？ |
|---|---|---|
| R1 | 哪些 pod 被放置了，放在哪个节点 | **可以** —— `algorithmResult.scheduleResult.SuggestedHost` |
| R2 | 哪些 pod 从未被评估 | **可以，但只是隐式的** —— 见下 |
| R3 | 节点级资源总量（`cpu: 60/64`） | **可推导** —— 见下 |
| R4 | 每个**节点**的拒绝原因 | **可以，但是自由字符串** —— 反查各失败 pod 的 `NodeToStatus` |
| R5 | 某次拒绝是关于哪种资源的 | **不行** —— 需要解析 `"Insufficient cpu"` |
| R6 | 某节点拒绝了多少个 pod | **可以** —— `ForEachExplicitNode` + `absentNodesStatus` × `(NumAllNodes - Len())` |

R4 和 R6 都是**节点维度**的表述。数据来源仍然是各 pod 的 `FitError`，但读取方向是反的：
不是"遍历 pod，收集它的原因集合"，而是"遍历 pod 的 `NodeToStatus`，把观测归到节点名下"。
这个方向的差别决定了后面整个设计的形状，见"按节点聚合"。

**只有 R5** 是真正的 plugin-model 缺口。而 R3——恰恰是被点名为需要"non-trivial changes
in scheduler plugin model"的那部分——不碰任何插件就能推导出来。

### R2：把"未评估"变成显式的

`podGroupSchedulingDefaultAlgorithm` 在 `PlacementFeasible` 返回 Unschedulable 时立刻
跳出 pod 循环（`schedule_one_podgroup.go:635`），所以 `result.podResults` 比 pod 列表短。
随后 `completePodGroupAlgorithmResult`（`schedule_one_podgroup.go:712`）用一批条目补齐，
这些条目的 `status` 是**组级** status 的克隆，`scheduleResult` 是零值。

补齐之后，"从未被评估"只能从条目的形状去推断——克隆的组 status 加空的 `SuggestedHost`。
这很脆弱：一个**确实被评估过**、且被同一个组级 status 拒绝的 pod，与之无法区分。

两个地方需要这个区分：

- **节点表的分母。** 未评估的 pod 不能算进任何节点的 `rejectedAt` 集合，否则一个节点会
  凭空背上它其实从没看过的 pod 的拒绝数。`Unused nodes (35): not evaluated, gang aborted
  after pod 30/100` 这一行里的 `30/100` 也来自这个计数。
- **pod 自身的 condition。** Issue 第 1 节要求未评估的 pod 说 "pod was not evaluated: gang
  scheduling aborted early"，而不是复述一条组级失败原因，让用户以为是这个 pod 自己不行。

修法：给 `algorithmResult` 加一个显式的 `notEvaluated bool`，由
`completePodGroupAlgorithmResult` 在它合成的条目上置位。纯 `pkg/scheduler` 内部，
不涉及 framework 表面。

### R3 的推导

对每个接收了至少一个 gang pod 的节点 `N`：

```
allocatable(N)  = snapshot.NodeInfos().Get(N).GetAllocatable()
baseline(N)     = snapshot.NodeInfos().Get(N).GetRequested()
gangDelta(N)    = Σ podResult.podInfo.CalculateResource()，对所有 SuggestedHost == N 的已放置 pod
used(N)         = baseline(N) + gangDelta(N)
```

三条性质保证它正确：

1. **revert 已经发生了。** `runRootSchedulingAlgorithm` defer 了
   `revertFns.revert()`（`schedule_one_podgroup.go:469`），它会 unreserve 并 forget 每一个
   被试探性 assume 的 gang pod。`submitPodGroupAlgorithmResult` 在其之后运行，所以
   `snapshot.NodeInfos().Get(N).GetRequested()` 正是 gang 之前的基线——恰好就是我们要
   叠加 gang 自身贡献的那个量。不存在重复计算。
2. **`Allocatable` 不受 assume/forget 影响。** 它来自 Node 对象。
3. **Pod 的请求量已经缓存了。** `PodInfo.CalculateResource()`
   （`pkg/scheduler/framework/types.go:1376`）会 memoize 到 `pi.cachedResource`，而调度周期
   早已填充过它。求和是 O(已放置 pod 数)，无需重算。

所以 placement 叙事——`node-A: 15 pods (cpu: 60/64)`——的代价是：每个不同节点一次 snapshot
查找，加上每个已放置 pod 一次整数加法，且**只在失败路径上**发生。

### R5：为什么这是真缺口

要写出饱和节点那一行的后半段

```
node-A: accepted 15, then rejected 70 (Insufficient cpu; cpu 60/64 used, 60 by this gang)
                                       ^^^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
                                       来自 NodeToStatus   来自 R3 的推导
```

聚合器必须知道 `node-A` 上的拒绝是*关于 cpu 的*，才能把这两半接起来——从 R3 算出的资源表里
挑出 cpu 那一行，而不是把 memory、pods 全打印出来。而今天唯一的信号就是字面字符串
`"Insufficient cpu"`。匹配它意味着：

- 去解析一段并不属于任何兼容性契约的消息文本；
- 对标量资源静默失效——它们的 reason 由 `fmt.Sprintf("Insufficient %v", rName)` 构造，
  可以包含任意资源名；
- 非 `noderesources` 插件无法参与。

同一个缺口也让**排除节点按原因折叠**（R4）变得脆弱：按原始消息串归并是能跑，但任何把变量
插进 reason 的插件（节点名、计数、拓扑 key）都会让每个节点产生一个独立的桶，于是
`excludedNodeHistogram` 退化成一节点一行，彻底摧毁折叠。

---

## 按节点聚合，而不是按 pod 聚合

Issue #141025 的大 gang 布局有三段——按节点的 placement、按 pod 的失败签名、top rejecting
nodes——其中第一段和第三段本来就是节点维度的，只是从两个角度描述同一批节点。把它们合并成
一张按节点的表严格更有信息量，因为关于一个节点，最有意思的事实是**转折点**：它在开始拒绝
之前接收了多少个 gang pod。

这个转折点把每个节点分成三类之一：

| 类别 | 特征 | 告诉运维什么 |
|---|---|---|
| **Saturated（饱和）** | 接收过 ≥ 1 个 pod，之后拒绝了后续 pod | gang 自己把这个节点吃满了。容量问题——更大的节点、更多节点、或更小的 `minCount` 能解决。 |
| **Excluded（排除）** | 接收 0 个 pod，看到的 pod 全拒 | 这个节点对该 gang 从来就不可用。约束问题（污点、亲和性、选择器）或既有负载——给 gang 加容量没用。 |
| **Unused（未触及）** | 接收 0、拒绝 0 | 从未被评估：gang 经由 `PlacementFeasible` 提前中止，或 pod 根本没走到那一步。调度器没够到的余量。 |

渲染效果：

```
Gang scheduling failed: 30/100 pods placed, minCount=80.

Saturated nodes (3) — accepted pods, then ran out:
  node-A: accepted 15, then rejected 70 (Insufficient cpu; cpu 60/64 used, 60 by this gang)
  node-B: accepted 10, then rejected 70 (Insufficient cpu; cpu 40/64 used, 40 by this gang)
  node-C: accepted  5, then rejected 70 (Insufficient memory)

Excluded nodes (12) — rejected every pod:
  9 nodes: untolerated taint
  3 nodes: node affinity mismatch

Unused nodes (35): not evaluated, gang aborted after pod 30/100.
```

### 为什么这该是默认维度

- **边界落在运维真正能动手的对象上。** 一行一个节点直接对应一个补救动作。按 pod 的签名
  列表做不到这点："45 pods: Insufficient cpu" 完全没告诉你*在哪儿*。
- **基数是收敛的。** 饱和节点数至多等于已放置 pod 数。排除节点坍缩成 reason 直方图，而
  `absentNodesStatus` 桶（`pkg/scheduler/framework/interface.go:38`）从构造上就坍缩成一行。
  5000 节点集群上的 100 pod gang 依然只渲染成几行。
- **它重建了 issue 想要的因果链。** "pod-1 和 pod-2 在 node-A 上吃了 8 cpu，所以 pod-3 得到
  Insufficient cpu"——这**就是**饱和节点那一行，不需要在两个独立段落之间交叉比对。

### 它同时削弱了对 R5 的依赖

这一点值得单独点出来，因为它改变了推荐方案的分量。

R5 之所以重要，是为了归因：要说"node-A 拒绝这些 pod *是因为 cpu，而这些 cpu 正是 gang 自己
消耗的*"，聚合器就必须知道拒绝涉及哪种资源，也就得去解析 `"Insufficient cpu"`。

在节点维度下，因果论断由**观测的结构**承载，而不是由 reason 文本承载。一个接收了 15 个 pod
之后才开始拒绝的节点，从构造上就说明它对这个 gang 是结构可用的——没有污点、没有亲和性不匹配、
没有选择器问题，因为已经有 15 个 pod 落上去了。那么 pod 15 到 pod 16 之间变化的东西，只能是
这个 gang 自身的消耗。**即使每个 reason 字符串都完全不可解析，这个分类依然成立。**

因此 `RejectionCode` 从**必需**降级为**提升质量**：它让行显示 `cpu 60/64` 而不是
`Insufficient cpu`，并让排除节点的直方图在插件把变量拼进 reason 时仍然稳定。两者都有实际
价值，但都不再是承重结构。

对计划的影响：

- 诊断工作不再被 framework 改动阻塞。它可以在今天的 plugin model 上落地，之后再获得精度
  ——这正是 macsko 要的节奏（"adding more context to existing messages should be suitable
  for the incoming release"）。
- 方案 B 的论据进一步削弱：它的优势是权威数字，而现在这些数字只是一个不依赖它们也成立的
  论断上的装饰。

### 顺序是可还原的

分类需要知道某节点上的"接收"发生在"拒绝"**之前**。`podGroupSchedulingDefaultAlgorithm`
在一个对 `queuedPodInfos` 的单线程顺序循环里 append 到 `result.podResults`
（`schedule_one_podgroup.go:605`），所以切片下标就是评估顺序。对节点 `N`：

```
acceptedAt(N)  = { i : podResults[i] 被放置在 N 上 }
rejectedAt(N)  = { i : podResults[i].FitError 中 N 拒绝了它 }
saturated(N)   = acceptedAt(N) ≠ ∅  且  max(acceptedAt) < max(rejectedAt)
```

不需要新增任何记账。

### 这个维度的边界

- **异构 gang。** 如果 pod 形状不同，"node-A rejected 70" 可能掩盖了其中 60 个卡在 cpu、
  10 个卡在设备插件。在 pod 形状可用之前，行里带 reason 直方图而不是单一 reason，代价是
  行更长。这不是永久限制——见下文"按 pod signature 分区"。
- **小 gang。** issue 原本对 ≤10 pod 用逐 pod 布局。本设计**不保留这个双模式**，理由见
  下节。
- **交错。** 异构 gang 可能产生这样的节点：拒绝 pod 2、接收 pod 5、拒绝 pod 9。上面定义的
  `saturated` 仍能正确分类，但该行应该同时报出两个计数，而不是暗示成一次干净的转折。
- **TAS。** 这张表描述的是**单个** placement 的模拟（`schedule_one_podgroup.go:1041` 上报
  的是 `anyResult`），所以 "unused nodes" 的计数范围是该 placement 的节点集，而不是整个集群。

### 只有一个渲染器：为什么不保留逐 pod 布局

Issue #141025 提议按 gang 大小分两种布局：≤10 pod 逐 pod 列出，>10 pod 才聚合。我最初照搬了
这个双模式，但它有两个问题。

**第一，"小 gang 逐 pod 更紧凑"是错的。** 逐 pod 那一行的形式是
`pod-3: node-A: Insufficient cpu; node-B: untolerated taint`——它要列出**节点**。在 5000 节点
集群上这一行无法真的列 5000 个节点，只能退化成"该 pod 的原因直方图"。而节点表在小 gang 下
本来就很小：3 个 pod 至多产生 3 行饱和节点，加一行排除节点直方图、一行未触及节点。所以
节点表在两种规模下都紧凑，双模式换不来任何体积收益。

**第二，逐 pod 视图与 pod 自身的 condition 重复。** 每个失败 pod 的 condition 里已经带着它
自己的 `FitError`——这正是 issue 第 1 节（pod 级：保留个体诊断）要保证的事情。把同样的信息
在 PodGroup condition 里再抄一遍，只是让两处可能不一致而已。PodGroup condition 应该回答
pod condition **回答不了**的问题：跨 pod 的节点占用与饱和关系。

因此 `buildPodGroupDiagnosis` 只有一个渲染器——始终是节点表。分层由此变得干净：

| 层级 | 回答的问题 |
|---|---|
| Pod condition | 这个 pod 为什么没调度上？（它自己的 `FitError`，逐节点原因） |
| PodGroup condition | 这个 gang 为什么没凑够 minCount？（节点占用、饱和、排除、未触及） |
| CompositePodGroup condition | 哪些子组可行、哪些不可行？（由 #140670 承载） |

少一个渲染器，也就少一套需要维护和测试的布局分支。

---

## 按 pod signature 分区

上面的异构 gang 和交错两个限制来自同一个根因：一张节点表混进了彼此不可比较的 pod。
而调度器**已经有**修复它所需的等价关系。

`fwk.PodSignature`（`staging/src/k8s.io/kube-scheduler/framework/interface.go:765`）由
`frameworkImpl.SignPod`（`pkg/scheduler/framework/runtime/framework.go:884`）从各插件的
`SignFragment` 构造。它的契约恰好就是诊断需要的：

> Any two pods with the same signature should get the same feasibility and scoring for
> the same set of nodes in the same state.

关键在于，它**已经挂在聚合器手上的那个对象上**了：`QueuedPodInfo.PodSignature`
（`pkg/scheduler/framework/types.go:700`），由调度队列的 `signPod`
（`pkg/scheduler/backend/queue/scheduling_queue.go:2185`）填充，而 `algorithmResult.podInfo`
**就是**一个 `*framework.QueuedPodInfo`。零新增管道。

### 它带来什么

按 signature 对 gang 的 pod 分区，然后每个 signature 类渲染一张节点表：

```
Gang scheduling failed: 30/100 pods placed, minCount=80. 2 pod shapes.

Shape 1 (80 pods, e.g. worker-0: cpu=4, mem=8Gi):
  Saturated nodes (3):
    node-A: accepted 15, then rejected 65 (Insufficient cpu; cpu 60/64 used, 60 by this gang)
  Excluded nodes (12): 9 untolerated taint, 3 node affinity mismatch

Shape 2 (20 pods, e.g. ps-0: cpu=8, mem=16Gi, nvidia.com/gpu=1):
  Excluded nodes (50): 50 Insufficient nvidia.com/gpu
```

改善的不止一件事，而是两件：

1. **行变得无歧义。** 同一个类内部，feasibility 按定义是一致的，所以某节点的拒绝原因是
   节点的属性，而不是混合 pod 形状造成的假象。每行的 reason 直方图坍缩成单一 reason。
2. **饱和推断变得无懈可击。** 在混合表里，"先接收后拒绝"只能**概率性地**推出是 gang 消耗
   ——一个异构 gang 完全可能先接收一个小 pod，然后因为一个**只对大 pod 成立的结构性原因**
   拒绝了大 pod。在 signature 类内部这不可能发生：feasibility 相同意味着 pod `i` 和 pod `j`
   之间唯一变化的就是 state，而这个 state 变化就是 gang 自身的消耗。整个节点维度设计所
   依赖的那个论断，**从推断变成了定理**。

shape 数量本身就是诊断信息。它同时解决了 `schedule_one_podgroup.go:878` 挂着的那条 TBD：

> TBD: Add a message to status if the pod used features for which finding a placement
> cannot be guaranteed, such as heterogeneous pod group or using inter-pod dependencies.

一个有 N > 1 个不同 signature 的 gang **就是**异构 pod group——不需要任何新的启发式判断。

### 为什么它是后续阶段而非阶段 1

- **signature 是可选的。** `OpportunisticBatching` 门控关闭时 `SignPod` 返回 nil
  （`pkg/features/kube_features.go:837`）；而且只要**任何一个**启用的 Filter/Score 插件
  没实现 `SignPlugin`，`enableSignatures` 就被整体关掉（`framework.go:868`）。所以分区必须
  是严格机会性的：nil 或单 signature 的 gang 原样回退到阶段 1 的表。
- **signature 是不透明的。** `PodSignature` 是 marshal 出来的 JSON blob，不是标签。消息
  需要短句柄，所以构建器按评估顺序分配序号名（`Shape 1`、`Shape 2`），并打印一个代表性
  pod 名及其资源请求作为人类可读的注解。这个渲染是唯一真正新增的代码。
- **基数需要上限。** 病态情况下可能一个 pod 一个 signature。构建器最多渲染 3 个类，超出
  显示 `... and N more shapes`；当类数接近 pod 数时回退到不分区的表（这本身就是"signature
  没在有效聚类"的信号）。
- **陈旧性。** `QueuedPodInfo.Update` 会清空缓存的 signature
  （`pkg/scheduler/framework/types.go:711`），所以中途被更新的 pod 可能 signature 为 nil 而
  同伴不是。signature 为 nil 的 pod 收进一个 `unclassified` 桶，而不是把整个 gang 拖回平表。

这些都不阻塞阶段 1：分区是架在 `classifyNodes` **之上**的一层分组，后者继续对交给它的
任意 pod 子集工作。

---

## 方案对比

### 方案 A —— 推导数字，给 `Status` 加一个 `RejectionCode`

给 `fwk.Status` 加一个字段：拒绝原因的机器可读、稳定标识。

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
```

资源类的拒绝用资源名限定 code，例如 `"InsufficientResource:cpu"`。四种常见情况用包级常量，
所以热路径只是赋一个预构造好的 string header，零分配：

```go
var (
	rejectionInsufficientCPU    = fwk.RejectionInsufficientResource.For(v1.ResourceCPU)
	rejectionInsufficientMemory = fwk.RejectionInsufficientResource.For(v1.ResourceMemory)
	// ...
)
```

标量资源需要一次拼接，但 `fit.go` 本来就为它们付了一次 `fmt.Sprintf`，所以增量开销被
今天已有的开销所限定。

`Status` 增大一个 16 字节的 string header。通过 builder 方法设置，现有调用点无需改动：

```go
func (s *Status) WithRejectionCode(c RejectionCode) *Status
func (s *Status) RejectionCode() RejectionCode
```

聚合器随后：

- 把**排除节点**按 `RejectionCode` 归并成桶，而不是按消息串归并——桶的键稳定，不会因为
  插件在 reason 里插了节点名就退化成一节点一桶；
- 对**饱和节点**，用它的 `InsufficientResource:<r>` code 去 R3 的表里查 `<r>` 那一行，
  只打印被耗尽的那种资源；
- 当 `RejectionCode()` 为空时回退到人类可读的 reason 串，未迁移的插件优雅降级而非直接失效。

注意这两条都是**节点维度**的读取：聚合器遍历各失败 pod 的 `NodeToStatus` 只是为了把观测
归到节点名下，它从不按 pod 分组输出。

**开销。** 每个 `Status` 16 字节。`Status` 本来就是每个被拒节点每个 pod 各分配一个，并且
本来就持有一个 `[]string`（24 字节）加它的底层数组。相对增幅很小，且没有新增分配。

**影响范围。** staging framework 包里一个增量字段加两个方法；迁移一个插件
（`noderesources`）来验证模式；其他插件增量选择加入。

### 方案 B —— 在 `Status` 上挂完整的结构化详情

让插件挂一个带类型的载荷：

```go
type Status struct {
	// ...
	details []RejectionDetail // or an interface{} / any-typed escape hatch
}

type RejectionDetail struct {
	Code      RejectionCode
	Resource  v1.ResourceName
	Requested int64
	Used      int64
	Capacity  int64
}
```

表达力最强：数字来自真正做决策的那个插件，所以不存在聚合器的推导与插件的算术产生偏差的
风险（DRA extended resources、pod-level resources、in-place vertical scaling 都有微妙的
计账规则，R3 的推导必须一一镜像）。

**开销。** 这个版本才配得上 macsko 说的 "non-trivial"。`Filter` 每 pod 每节点跑一次；
5000 节点集群里一个不可调度的 pod 会产生多达 5000 个 `Status` 对象，每个现在都要带一个
48 字节结构体的切片。一个 100 pod 的 gang 就是 50 万条详情记录，一直保留到周期结束。要让它
可负担就需要一套门控机制——一个穿过 `CycleState` 的 "diagnostics enabled" 标志，或者一套
采样策略——而这个门控本身就是新的 plugin-model 表面，每个 Filter 插件都必须遵守。

**影响范围。** staging 包新增类型、每个插件都必须遵守的门控契约，以及门控开/关两种模式下
的行为差异（测试必须双向覆盖）。

### 方案 C —— 按需的 `Explain` 扩展点

加一个只在失败路径上调用的可选接口：

```go
type ExplainPlugin interface {
	Plugin
	// Explain returns structured detail about why this plugin would reject the pod
	// on this node. Called only when building a diagnosis, for a bounded sample.
	Explain(ctx context.Context, state fwk.CycleState, pod *v1.Pod, nodeInfo fwk.NodeInfo) RejectionDetail
}
```

热路径零开销：成功周期和普通失败周期什么都不变。诊断构建器挑一个有界的（失败 pod，拒绝
节点）样本集，请插件解释。

**致命问题。** 到构建诊断时，`revertFns.revert()` 已经把所有试探性 assume 都撤销了。
在 revert 之后的 snapshot 上重跑 `Explain`，回答的是另一个问题：`pod-3` 在 `node-A` 上被拒
**正是因为 pod-1 和 pod-2 当时被试探性放在了那里**，而 revert 之后它们不在了，所以
`Explain` 很可能报告这个 pod 放得下。诊断会自相矛盾。

要修就得二选一：把 revert 推迟到诊断之后（让集群状态为一个上报需求做人质，并改变
`revertFns` LIFO 契约的含义），或者为了诊断重新 assume 这些放置（在失败路径上重跑带副作用
的 Reserve 插件）。两者都比问题本身更糟。

有一个可行的窄版本——在拒绝发生时调用 `Explain`，用标志门控——但那就是多绕一层的方案 B。

### 对比

| | A：推导 + `RejectionCode` | B：结构化详情 | C：按需 `Explain` |
|---|---|---|---|
| 热路径分配 | 无 | 每个被拒节点一个切片 | 无 |
| `Status` 体积 | +16 B | +24 B 起 | 不变 |
| 数字是否权威 | 推导得来，可能与插件算术偏差 | 是，来自插件 | 是，但针对的是错误的集群状态 |
| revert 之后仍正确 | 是 | 是 | **否** |
| 是否需要门控机制 | 否 | 是 | 否 |
| 必须改动的插件 | 起步 1 个，其余可选加入 | 最终所有 Filter 插件 | 任何想参与的插件 |
| 是否同时服务 #138991 | 是 | 是 | 是 |

### 推荐：方案 A，分三阶段

A 是唯一既能填补 R5、又不引入热路径回退（B）或正确性风险（C）的方案。它唯一的弱点——推导
出的数字可能与插件的计算产生偏差——是可控的：

- 推导覆盖了用户真正会问的那些资源（cpu、memory、ephemeral-storage、pods、标量资源），
  它们都已被 `NodeInfo.Requested` 计入；
- 当聚合器无法为某个 code 推导出数字时，它打印 reason 但不加数字注解，而不是瞎猜；
- 如果未来确实出现需要插件提供数字的场景，`RejectionCode` 与"之后再加一个可选详情载荷"
  是前向兼容的——A 是 B 的子集，不是竞争方向。

由于节点维度聚合把 R5 从"必需"降级为"提升质量"，A 可以干净地拆成三个可独立发布的阶段：

- **阶段 1 —— 不动 framework。** 节点分类表、推导出的资源数字、排除节点的 reason 直方图。
  这一阶段的所有内容都在今天的 plugin model 上运行。
- **阶段 2 —— `RejectionCode`。** 用稳定的 code 替代 reason 串分组，并让饱和节点行能点名
  耗尽的资源。纯增量，改善阶段 1 的输出而无需重构它。
- **阶段 3 —— 按 `PodSignature` 分区。** 每个 pod shape 渲染一张节点表，消除异构 gang 和
  交错两个 caveat，并把饱和推断升级为保证。搭乘已有基础设施；见"按 pod signature 分区"。

阶段 1 是 macsko 要的、适合本 release 的量级。阶段 2 是本文档要论证的 plugin-model 工作。
阶段 3 完全不需要新的 framework 表面——它消费的是 opportunistic-batching 工作已经建好的东西。

---

## 详细设计（方案 A）

### 1. Framework 改动（阶段 2）

`staging/src/k8s.io/kube-scheduler/framework/interface.go`：

- `type RejectionCode string` 及上面那些常量。
- `func (c RejectionCode) For(r v1.ResourceName) RejectionCode` —— 返回
  `RejectionCode(string(c) + ":" + string(r))`。
- `Status.rejectionCode RejectionCode` 字段。
- `Status.WithRejectionCode(RejectionCode) *Status`（可链式，nil 安全）。
- `Status.RejectionCode() RejectionCode`（nil 安全，返回 `""`）。

现有行为完全不变；没有设置 rejection code 的 `Status` 与今天表现一致。

### 2. `noderesources` 迁移（阶段 2）

`fit.go` 保留 `Reason` 字符串用于人类可读消息，额外设置 code：

```go
return fwk.NewStatus(statusCode, failureReasons...).
	WithRejectionCode(insufficientResources[0].RejectionCode)
```

`InsufficientResource` 增加一个 `RejectionCode` 字段，在 `fitsRequest` 中用预构造常量填充。
多种资源同时不足时，第一个作为主 code；完整的人类可读列表仍留在 `reasons` 里。

### 3. 诊断构建器（阶段 1）

`pkg/scheduler/schedule_one_podgroup.go` 中新增的非导出 helper：

- `algorithmResult.notEvaluated bool`，由 `completePodGroupAlgorithmResult` 置位 —— R2。
- `classifyNodes(podResults []algorithmResult) nodeClassification` —— 按评估顺序遍历
  `podResults` 一次，为每个节点建立"接收下标集"和"拒绝下标集"，然后把节点划分为
  saturated / excluded / unused。这是节点维度聚合的核心。
- `nodeResourceUsage(snapshot, node string, placed []*algorithmResult) resourceUsage`
  —— R3，用于给饱和行加上 `cpu 60/64 used, 60 by this gang` 的注解。
- `excludedNodeHistogram(nodes []nodeEntry) []reasonBucket` —— 按 reason 折叠排除节点。
  阶段 2 落地后按 `RejectionCode` 归并，在此之前按 reason 串归并。
- `buildPodGroupDiagnosis(...) string` —— 组装消息。只有一种布局（节点表），不按 gang 大小
  分支；逐 pod 的细节留在各 pod 自己的 condition 里。

`classifyNodes` 是唯一需要完整结果集的 helper；其余都作用于单个节点或单个桶，因此可以
独立做单测。

### 3b. Signature 分区（阶段 3）

额外一个 helper，作用在 `classifyNodes` **之上**：

- `partitionBySignature(podResults []algorithmResult) []shapeClass` —— 按
  `podInfo.PodSignature` 分组，保留评估顺序以便序号命名，把 signature 为 nil 的 pod 收进
  `unclassified` 类；当只有一个不同 signature 或一个都没有时，返回单个无名类。

`buildPodGroupDiagnosis` 随后对每个类各跑一遍阶段 1 的流水线，而不是整体跑一遍。阶段 1
的任何 helper 都不需要改签名或改行为。

### 4. 消息体积预算

`metav1.Condition.Message` 上限 32 768 字符
（`staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/types.go:1693`）。构建器执行一个远低于
该上限的预算：

- 饱和节点：至多 10 行，超出显示 `... and N more saturated nodes`；
- 排除节点：至多 5 个 reason 桶，超出显示 `... and N more reasons`；
- 未触及节点：永远只有一行。

截断在文本中是显式的，所以被截断的消息绝不会读起来像完整的。节点维度聚合让这个预算很容易
守住：饱和行数被已放置 pod 数所限，另外两段从构造上就有界。

### 5. 需要写进措辞的准确性 caveat

- **节点采样。** `findNodesThatPassFilters` 一旦找到 `numFeasibleNodesToFind` 个可行节点
  就停止（`pkg/scheduler/schedule_one.go:787`）。对一个到处都失败的 pod 而言这无关紧要
  ——所有节点都被评估了——但对一个成功的 pod，`NodeToStatus` 是不完整的。聚合器只从**失败**
  的 pod 构建拒绝统计，所以分母是可靠的；这个约束必须写进 helper 的 doc comment，以免日后
  被违反。
- **缺席节点。** `NodeToStatus.Len()` 只统计显式被拒的节点。计数必须加上
  `absentNodesStatus` × `(NumAllNodes - Len())`，与 `FitError.Error()` 保持一致
  （`pkg/scheduler/framework/types.go:1495`）。
- **TAS placement。** 在 `TopologyAwareWorkloadScheduling` 下，上报的是任意一个 placement
  的结果（`schedule_one_podgroup.go:1041` 取 `anyResult`）。消息必须说明它描述的是哪个
  placement，否则用户会把单个 placement 的失败读成全局失败。

### 6. 测试计划

阶段 1：

- `classifyNodes` 的表驱动测试，覆盖每个类别和棘手情况：先接收后拒绝的节点（saturated）；
  全拒的节点（excluded）；什么都没碰到的节点（unused）；在一个节点上交错
  拒绝/接收/拒绝的异构 gang；在任何 pod 被评估前就被 `PlacementFeasible` 中止的 gang。
- `nodeResourceUsage` 的表驱动测试，针对一个含既有 pod 的 snapshot 断言"基线加增量"的
  算术，证明 revert 之后没有重复计算。
- 测试 `excludedNodeHistogram` 用正确的乘数把 `absentNodesStatus` 桶折叠进来，而不是丢掉它。
- `schedule_one_podgroup_test.go` 中加一个部分放置的 gang 用例，断言完整渲染出的 condition
  消息；再加一个提前中止的 gang 用例，断言 `notEvaluated` 的措辞。

阶段 2：

- `RejectionCode.For` 和 `Status.WithRejectionCode` 往返的单测，包含 nil `Status` 接收者。
- 单测断言 `noderesources.Filter` 对 cpu、memory、ephemeral-storage、pods 和一个标量资源
  设置了预期的 code，且人类可读消息未变（防止现有测试里出现意外的消息回归）。
- 用带 code、不带 code、混合三种输入重跑 `excludedNodeHistogram` 的表，证明插件增量迁移
  期间回退路径仍然有效。
- 在大节点集上对比 `RunFilterPlugins` 改动前后的 benchmark，证明热路径未受影响。

阶段 3：

- `partitionBySignature` 的表驱动测试：全 nil signature 坍缩成一个类；单个不同 signature
  渲染成不分区；两个 shape 正确分区；signed 与 nil 混合时 nil 的进 `unclassified` 而不
  干扰其余；一个 pod 一个 signature 时触发回退到平表。
- 一个 integration 风格的用例：关闭 `OpportunisticBatching` 时，输出与阶段 1 **逐字节相同**，
  以此证明分区确实是机会性的。
- 一个异构 gang 用例：断言分区之后每个饱和行的 reason 直方图坍缩成单一 reason —— 这是
  "行变得无歧义"这一论断的可观测证据。

### 7. 兼容性

纯增量。不改任何 API 类型。framework 字段本身不需要 feature gate；更丰富的 condition 消息
可以搭乘已有的 `GenericWorkload` gate，因为它只在 PodGroup 路径上产生。

---

## 考虑过但否决的方案

- **解析 reason 字符串。** 零 framework 改动，对今天的 `noderesources` 有效。否决理由：
  消息文本不是兼容性契约，且对任何把变量插进 reason 的插件都会失效。
- **在 `CycleState` 上开一条旁路。** 插件把结构化详情写进一个约定的 `CycleState` key，
  而不是写到 `Status` 上。否决理由：`CycleState` 是 per-pod-per-placement 的，且在 revert
  时被丢弃，所以数据仍然必须在 `Status` 已经流经的那些点上被拷出来——同样的结果，更多的管道。
- **用 Event 而非 condition 承载诊断。** Event 上限接近 1 KiB，而且会被聚合/去重，恰好
  摧毁了这条消息存在的意义所在的节点级细节。condition 才是正确的归宿；Event 可以携带一个
  指向它的指针。
