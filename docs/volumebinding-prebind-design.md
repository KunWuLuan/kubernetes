# 设计文档：VolumeBinding 绑定写路径的冲突重试

- 状态：草案（v3，需求收敛版）
- 日期：2026-09-08
- 涉及组件：kube-scheduler（`pkg/scheduler/framework/plugins/volumebinding`，仅 binder 内部）
- 相关代码基线：master @ 0ba07e0866b
- 范围声明：本版只解决**写冲突型**部分失败。供给失败重试、annotation 属主与接管、失败补偿、孤儿清理均为后续演进（§7）。

---

## 1. 需求理解

### 1.1 背景：绑定写路径

`WaitForFirstConsumer` 存储类下，VolumeBinding 插件在 PreBind 阶段把调度决策写回 apiserver（`bindAPIUpdate`，binder.go:515）：

- **静态绑定**：逐个 `Update` PV，写入 `spec.claimRef`（binder.go:549）；
- **动态供给**：逐个 `Update` PVC，写入 annotation `volume.kubernetes.io/selected-node=<node>`（binder.go:565）。

两个写都是**基于调度器缓存快照的全量 PUT**（携带 resourceVersion 前置条件），且**均无重试**：任何并发变更导致的 409 都会立即终结整个绑定周期。

### 1.2 问题链路（本设计唯一针对的场景）

多 PVC pod，其中某个 PVC 在快照之后被做过一次**不影响任何调度决策的更新**（用户加 label、其他 controller 写 annotation 等）：

```
T0  pod 选中 nodeA；节点资源在本调度器缓存中被本 pod 预留
    （schedule_one.go:200，assume 发生在 PreBind 之前）
T1  bindAPIUpdate：
    PV#1 claimRef 写入成功（PV#1 带节点亲和）；
    PVC#2 的 Update 返回 409——期间那个无害更新抬升了 resourceVersion
    （binder.go:565，全量 PUT + 无重试）
T2  周期失败：释放 nodeA 预留；广播 AssignedPodDelete 主动激活其他 pod
    （schedule_one.go:434-444）；pod 进 BackoffQ
T3  PV controller 异步完成 PV#1 → PVC#1 的绑定
    （已写入的不回滚，binder.go:545 自认 "no API rollback"）
T4  被激活的其他 pod 占满 nodeA
T5  pod 重试：PVC#1 已 Bound 且 PV#1 带节点亲和
    → checkBoundClaims 排除一切非 nodeA 节点
    → nodeA 资源不足 → FailedScheduling → Pending 循环
```

最终故障是两个独立因子的复合：

```
P(长期 Pending) = P(部分写入留下节点钉死) × P(该节点资源被他人占用)
```

而第一因子的种子，是一次**完全可以被吸收的冲突**：写入意图只是"给对象加一个 annotation / 一个 claimRef"，冲突的另一方只是改了 label。系统却用最昂贵的方式响应——作废整个周期、释放节点预留、把部分不可逆进度（claimRef 写入、供给触发）留在后台继续固化。

**核心洞察：修复时机决定修复成本。** 在 PreBind 内部重试，节点预留仍在本 pod 手中、部分写入尚未引发任何异步后果，一次带重试的写就能让周期正常完成；出了 PreBind 再处理，同样的问题就升级为"重调度 + 资源竞争 + 拓扑钉死"的复合故障。本设计把修复点放在写失败的那一刻。

### 1.3 需求

| # | 需求 |
|---|---|
| R1 | 不影响调度决策的并发更新（metadata 级：label、无关 annotation 等）不得导致绑定周期失败 |
| R2 | 重试不得弱化正确性：不得把 selected-node / claimRef 写到决策输入已变化、或已归他人所有的对象上——重试语义必须与现状的整对象 CAS 等价 |
| R3 | 重试延迟有界，落在现有 bindTimeout 预算内，不增加正常路径（无冲突时）的 API 调用 |
| R4 | 零跨组件契约变化：PV controller 与 external provisioner 感知不到本改动 |

### 1.4 非目标

- 不处理供给失败（provisioner 失败/超时）语义——checkBindings 行为不变；
- 不处理重试预算耗尽后的部分写入残留（补偿/回滚）——见 §7 残余风险；
- 不引入 annotation 属主、接管、孤儿清理等新跨组件约定；
- 不处理双活调度器场景。

---

## 2. 根因（代码事实）

| # | 事实 | 位置 |
|---|---|---|
| 1 | PVC 写为全量 PUT，基于 informer 快照的 resourceVersion，任何并发变更（含无害的）都 409 | binder.go:565 |
| 2 | PV 写（claimRef）同样形态，同样问题 | binder.go:549 |
| 3 | 两处写均无重试，任一失败立即返回，defer 只回滚本地 assume cache 中未处理部分，apiserver 已写入部分不回滚 | binder.go:527-536, 545 |
| 4 | 失败路径释放节点预留并主动激活其他 pod，为 T4 的资源竞争开门 | schedule_one.go:434-444 |

根因一句话：**写意图（单字段）与写形式（全量 CAS）不匹配，且失败处理没有分层**——无害冲突与决策失效冲突被同等对待，一律作废周期。

---

## 3. 方案：带不变式校验的冲突重试（rebase-on-conflict）

### 3.1 机制

对 `bindAPIUpdate` 中的每个 PV/PVC 写，替换为如下循环：

```
writeObject(snapshotObj):                        # 首次尝试与现状完全一致
  newObj, err := Update(snapshotObj)             # PUT，快照 RV 前置条件
  if err == nil: return newObj
  if !IsConflict(err): return err                # 非 409 维持现状：立即失败

  for attempt in 1..maxRetries:                  # 仅 409 进入重试
    live, err := Get(name)                       # 直连 apiserver（不走 informer——
    if NotFound: abort("object deleted")         #   informer 陈旧正是 409 的来源）
    if !invariantsHold(live): abort("见 3.2")     # 决策输入变化/他人所有 → 放弃
    if alreadyApplied(live): return live         # 幂等分支，见 3.4
    rebased := live + 本写入意图                  # 仅叠加我们的字段
    newObj, err := Update(rebased)               # PUT，live 的 RV 前置条件
    if err == nil: return newObj
    if !IsConflict(err): return err
  abort("retry budget exhausted")
```

要点：

- **正常路径零开销**：无冲突时行为与现状逐字节一致（同样一次 PUT）。
- **重试即 rebase**：每次重试基于直连 GET 的最新对象重建写入意图，等价于三方合并（base=快照，ours=annotation/claimRef，theirs=live），ours 只包含我们真正要写的字段，theirs 的其余内容原样保留——不覆盖任何并发写。
- **abort 的语义与现状一致**：周期失败、Unreserve、重调度。即"决策真的失效了"时，系统响应不变，本设计只是把"决策没失效"的那类冲突从 abort 里摘出来。
- 成功后仍将返回对象写回 `binding.pv` / `claimsToProvision[i]`（现状逻辑），保证 checkBindings 的 resourceVersion 比较（binder.go:638/671）语义不变。

### 3.2 不变式清单（R2 的落地）

重试放弃（abort）的判据采用一条简单规则：**metadata 级差异 → 吸收（rebase 后继续）；spec/status 级差异 → 放弃**。spec 与 status 正是调度决策的全部输入域。

**PVC 写（annotation）**：

| 检查 | 通过条件 | 不通过的含义 |
|---|---|---|
| UID | `live.UID == snapshot.UID` | 删除重建，对象已是另一个 PVC |
| deletionTimestamp | nil | 正在删除 |
| Spec | `DeepEqual(live.Spec, snapshot.Spec)` | 决策输入变化：storageClassName / 容量 / accessModes / volumeMode / selector / volumeName（volumeName 出现即已被绑定） |
| Status | `DeepEqual(live.Status, snapshot.Status)` | phase 漂移（如 Lost） |
| selected-node | 缺失，或 `== 本节点` | 值为其他节点 → 已被其他写方（共享 PVC 的另一个 pod、另一调度器实例）占有 |

**PV 写（claimRef）**：

| 检查 | 通过条件 | 不通过的含义 |
|---|---|---|
| UID / deletionTimestamp | 同上 | 同上 |
| Spec（剔除 claimRef） | `DeepEqual(live.Spec, snapshot.Spec 剔除 claimRef 后)` | 匹配基础变化：nodeAffinity / 容量 / accessModes——本节点选择对这些有依赖 |
| claimRef | nil，或指向本 PVC（幂等） | 指向其他 PVC → 该 PV 已被他人绑定 |
| Status | phase 仍为 Available | 已被他人绑定/释放 |

基线细节：比较基线取 assume 前的决策对象。PVC 的比较不涉及 metadata（annotation 不参与），assumed 克隆可直接作基线；PV 的 assumed 克隆已含我们写入的 claimRef，故 Spec 比较须剔除 claimRef 字段并单独按上表校验（无需额外保留原始对象）。

### 3.3 重试预算

- `maxRetries = 3`，退避 100ms / 200ms / 400ms（建议加抖动），整体受 ctx 与剩余 bindTimeout 约束；
- 只对 `apierrors.IsConflict` 重试；网络错误、4xx 语义错误等维持现状立即失败（收窄爆炸半径）；
- 常量起步，不进 VolumeBindingArgs；灰度数据支持后再议配置化（§8）。

### 3.4 安全性论证

1. **时间窗安全**：重试全程位于 PreBind 内，节点预留由本 pod 持有（schedule_one.go:200），不存在与后续调度尝试的竞态；写意图的拓扑目标与被预留保护的决策一致。
2. **CAS 等价**：现状的全量 PUT 提供整对象乐观锁；rebase 重试在每次写入时仍带 live 的 RV 前置条件（GET 与 PUT 之间再有并发变更 → 409 → 下一轮重试），合并语义上等价于"容忍 metadata 差异的 CAS"。§3.2 的不变式把"容忍范围"显式限定在决策输入域之外，满足 R2。
3. **幂等分支**：PUT 成功但响应丢失（超时）时，重试的 GET 会看到 annotation/claimRef 已是本意写入——直接返回成功，不重写。这同时覆盖了"409 返回但服务端实际未冲突"的边角。

### 3.5 残余风险（诚实边界）

| 残余 | 概率 | 处置 |
|---|---|---|
| 无害更新高频持续，3 次预算耗尽 | 低（需要同一对象在 ~1s 内被连续修改 4 次） | 维持现状失败路径；§4 指标监控其发生率，若真实存在再调预算 |
| 决策输入真实变化的冲突 | 任意 | 本来就该失败重调度——这是正确行为，不是本设计要消除的 |
| 非 409 失败（网络、apiserver 5xx）后的部分写入 | 不变 | 现状问题，属 §7 补偿机制的范围 |

---

## 4. 可观测性

- 新指标：`volumebinding_write_conflict_retry_total{object="pvc"|"pv", outcome="success"|"idempotent"|"aborted"|"exhausted"}`
  - `success` 应占绝对多数（衡量本设计生效度）；
  - `exhausted` 监控 §3.5 第一项残余；
  - `aborted` 区分量级：若 spec 变化类占主导，说明集群存在真实的决策失效竞争，与本设计无关但值得知道。
- 事件：exhausted/aborted 时在现有失败事件中附一句具体原因（"PVC x updated concurrently: spec changed / owned by other writer / budget exhausted"），替代现状无指向的泛化错误。

---

## 5. 测试计划

**单元测试**（扩展 `TestBindPodVolumes` / 新增 `TestBindAPIUpdateRetry`）：

1. PVC#2 首次 PUT 409 + live 仅多一个 label → 重试成功 → 周期完成，annotation 正确，live 的 label 保留在最终对象上；
2. 409 + live 的 spec.storageClassName 已变 → abort，错误信息含原因；Unreserve/回滚行为与现状一致；
3. 409 + live 的 selected-node 已是其他节点 → abort；
4. 409 + live 的 selected-node 已是本节点（响应丢失场景）→ 幂等成功，无第二次写；
5. 连续 3 次 409 → exhausted abort；deferred 回滚只覆盖未处理对象（现状逻辑回归）；
6. PV 路径同构用例：claimRef 幂等、nodeAffinity 变化 abort、被他人绑定 abort；
7. 非 409 错误 → 立即失败，无重试（现状回归）。

**集成测试**：

- 多 PVC pod + 注入"快照后 label 更新"：断言 pod 绑定 nodeA 成功、无重调度、无 Pending；
- 多 PVC pod + 注入"快照后 spec 更新"：断言失败重调度（现状语义保持）。

---

## 6. 交付与回退

| 项 | 内容 |
|---|---|
| PR1 | PVC 写路径重试（3.1-3.3）+ 单测 1-5 + 指标 |
| PR2 | PV 写路径重试（同构）+ 单测 6 |
| gate | 建议 `VolumeBindingConflictRetry`（默认值随上游评审定，提供灰度回退） |
| 回退 | gate 关闭即完全回到现状；无 API/契约残留 |

改动全部位于 binder.go 内部，无新 annotation、无跨组件语义变化，满足 R4——这也是它适合作为整个问题族第一个落地 PR 的原因。

---

## 7. 范围外（后续演进，前版设计已覆盖）

| 机制 | 针对的残余问题 | 为何本期不做 |
|---|---|---|
| 失败补偿（test+remove 清理已写部分） | 非 409 失败/exhausted 后的部分写入残留 | 重试已把冲突类部分失败概率压至近零；剩余为低频长尾 |
| 供给失败重武装（周期内重试 provisioner 瞬时失败） | 2a 型：PVC#1 供给成功固化拓扑 + PVC#2 瞬时失败 | 独立问题域（供给侧而非写侧），需独立 gate 与 PV controller 行为核验 |
| annotation 属主与接管 | bindTimeout 超时后的无主残留 | 新跨组件约定（owner annotation），需 KEP；依赖本期指标验证残余规模后再立项 |
| 孤儿清理 / 点名式诊断事件 | pod 删除后残留、诊断体验 | 随属主机制一并考虑 |

演进路径：本期指标（尤其 `exhausted` 与冲突分布）直接决定上述各项的优先级排序。

---

## 8. 开放问题

1. maxRetries / 退避常量与 bindTimeout 的关系是否需要显式约束（当前仅受 ctx 控制）？——建议实现时 clamp 到剩余预算。
2. 重试期间 checkBindings 的 1s 轮询是否需要感知写重试进行中？——不需要：轮询在 `bindAPIUpdate` 返回后才启动（binder.go:491-499），时序天然隔离。
3. 直连 GET 绕过 informer 的成本：仅在冲突路径发生（每 pod 至多几次），可接受；但需注意与 scheduler 的 API 限流（APF）共存——实现时确认 client 侧 priority level。
4. exhausted 场景的事件是否直接复用现有 PreBind 失败事件流（pod 事件）还是新增 PVC 事件？倾向前者（用户视角一致），实现时定。
