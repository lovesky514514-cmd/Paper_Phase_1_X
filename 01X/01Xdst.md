# 01dst.md — 第一篇（01X）评估：RAB-Gate 风险预算驱动的运行时保障网关

> **评估对象**：`C:\Users\20050224\Desktop\01X\`（立项书 `01x.md` 43,878 B + 代码包 `01xcode.zip`）
> **评估方式**：**只读**。已读完立项书全文、全部 5 个 Go 源文件、5 个 Python 模块、测试与 `results/pilot.json` / `ood_pilot.json` 的完整原始数据。
> **未执行**：没有编译、没有跑 `scripts/check.py`。文中所有实验数字都来自包内 `results/*.json`，不是我的复验。
> **说明**：本轮 `01dst.md` 对应新的 `01X` 项目（RAB-Gate）。上一轮针对 `01`（RAGuard）的报告在本轮开始时已不在工作区。

---

## 0. 一页速览

| 维度 | 评分（1–5） | 一句话依据 |
|---|:---:|---|
| 选题价值与撞题风险 | **4.0** | "可靠性工程 × Agent 网关"是五个里最自然的交叉；但 STPA+网关这条线已被 ICSE-NIER 2026 等占住，立项书自己列了 10 篇 |
| 立项文档质量 | **4.5** | 危险语义合约、Hoeffding 上界、预算约束、10 类指标、Go/No-Go、期刊分级——可执行度很高 |
| 原型实现完成度 | **4.5** | **Go 引擎是全系列最扎实的一块**：版本乐观锁、危险互锁、预测压力、执行后强制回读、幂等指纹 |
| 结果可信度 | **2.0** | 主方法 STC 21.2% vs 固定规则 43.8%，但**这个负结果可以被一个常数完全解释**（见 §3.1） |
| 工程与仓库质量 | **4.0** | 三层测试（Go 单测 / Go race / 真实 HTTP）；扣分在 29 个生成页与 Windows 未真机验证 |
| **总分** | **19 / 25** | 五份里排第 1 |

**一句话结论**：**这是五份里"工程最硬、实验最可惜"的一份。** 它的安全合约写得比大多数硕士项目都细（执行后必须回读才能再次动作），风险上界的统计学也是对的；但它的先导实验**根本没在可行预算下跑过一次**——`task_budget = 0.72` 比它自己的两次数值动作所需的风险值（0.783）小 8.6%，于是 50% 的任务在结构上注定失败。

**如果只能改三件事**：

1. 把 `contracts/risk_samples.json` 的 `task_budget` 从 **0.72 改到 ≥0.85**，重跑 `evaluate.py`。我预判 `ours` 会从 0.21 跳到 ~0.42，与 B2 持平。
2. 让风险估计**依赖状态**（现在只依赖工具名），否则"风险预算"只能数次数，无法区分危险的阀门动作和安全的阀门动作。
3. 修 `B3`：它现在与 `B2` 数值完全相同，"点估计 vs 置信上界"这个对照实际上没有发生。

---

## 1. 项目构成

| 层 | 内容 | 规模 |
|---|---|---|
| 立项书 | `01x.md`，与包内 `research/01x_original.md` **逐字节相同** | 502 行 |
| Go 执行面 | `internal/core/model.go`（状态、互锁、风险账本）、`internal/httpapi/server.go`、`cmd/rabgate` | 约 495 行 |
| Go 测试 | `core_test.go`（179 行）、`server_test.go` | 约 234 行 |
| Python 研究层 | `experiment.py`（六策略对照）、`risk.py`（Hoeffding 上界）、`client.py`、`demo.py` | 约 228 行 |
| 结果 | `results/pilot.json`（500 任务 × 6 策略）、`results/ood_pilot.json`（drift=0.2） | 各 344 行 |
| 文档 | `README / ARCHITECTURE / EXPERIMENTS / QUALITY / NEXT_STEPS / THREAT_MODEL / GLOSSARY / CODING_STANDARDS / HAZARD_CONTRACT` + `docs/adr/` + **29 个生成页** | 41 个 md |

**两套独立的东西**（这点它自己说清楚了，`EXPERIMENTS.md:3`）：Go 是真实本地 HTTP 状态流（验证执行正确性），Python 是离线策略对照（验证统计流程可复现）。

---

## 2. 优点：它真的做对了什么

### 2.1 Go 引擎把"安全合约"写成了可执行状态机

`internal/core/model.go:245-281` 的 `precondition()` 是我在这一整套代码里见过最像工程安全逻辑的东西：

```go
case "change_valve":
    if a.Value != 0 && a.Value != 1 { return "valve value must be 0 or 1" }
    if !s.SensorHealthy || s.SensorAgeMS > 250 { return "sensor evidence is invalid or stale" }
    if s.Alarm || s.Pressure > 700 { return "interlock: alarm or high pressure" }
    predicted := s.Pressure
    if a.Value == 1 && s.Valve == 0 { predicted += 80 }
    if predicted > 820 { return "pressure invariant would be violated" }
```

三个细节值得单独表扬：

1. **检查的是"预测后的压力"而不是"当前压力"**（`predicted`），也就是说它拦的是"这一步会把系统推出不变量"，而不是"现在的状态已经不对"。这是跳闸逻辑（interlock）的正确写法。
2. **传感器新鲜度作为前置条件**（`SensorAgeMS > 250` 直接拒），对应立项书 §3.3"拒绝仅凭 Agent 的自然语言声明"。
3. **`apply()` 里执行完写入 `SensorAgeMS = 300`**（`:294`），于是**下一次动作必须先回读**才能通过前置条件——把"读后写"的顺序要求编码进了状态机，而不是靠调用方自觉。

第 3 点尤其难得：它让"证据时效"成为一个**卡在状态机里的硬约束**，而不是文档里的一句提醒。

### 2.2 风险上界的统计学做法是对的

`risk.py:7-27` 与 `model.go:52-67` 两边一致：

```python
adjustment = (1 - confidence) / len(hazards)          # Bonferroni
p = k / n
p = min(1.0, p + sqrt(log(1 / adjustment) / (2 * n)))  # Hoeffding 单侧上界
score += p * h["severity"]
```

- 多重危险做了 **Bonferroni 校正**（`1-α` 分摊到每个危险事件），这是审稿人会追问的点；
- **样本不足时返回 `None` 而不是 0**（`risk.py:21-22`、`model.go:60-62`），随后决策变成 `escalate`（`model.go:195-198`）——"不知道"不会被当成"安全"；
- 代码注释明确写"这是 severity-weighted risk **proxy**，never an accident probability"（`model.go:50-51`）。

立项书 §4.3 要求"Clopper–Pearson 或其他适合的上置信界"，这里用的是 Hoeffding（对均值的界更宽松），口径一致、方向正确。

### 2.3 幂等重放带指纹校验，不是无脑重放

`model.go:169-177`：

```go
if prior, exists := t.requests[a.RequestID]; exists {
    if prior.fingerprint != fingerprint {
        return Decision{}, errors.New("request_id reused with different payload")
    }
    result.Decision = "replay"
    ...
}
```

**同一个 `request_id` 换参数 → 直接报错，而不是把旧结果还回去。** 这是幂等实现里最容易被漏掉的一步（很多实现只按 ID 查缓存）。

### 2.4 只读与写入被严格分开，并且有安全回退

- `readOnly()`（`:244`）把 `read_pressure / read_case / hold` 划为只读，**不消耗风险预算**；
- `hold` 作为 process 域的显式安全回退（`:208-218` 两处 `SuggestedAction = "hold"`）；
- `block / defer / escalate / allow` 四态分离，`defer` 与 `block` 语义不同（`defer` 是"预算留给后面"，`block` 是"前置条件不满足"）。

### 2.5 诚实度与自检结构都在线

- `QUALITY.md:22` 直接给出负结果并写"**不能据此声称提出的方法取得研究胜利**"；
- `EXPERIMENTS.md:9` 逐条列出五个隐含限制（缺真实 Agent、缺 P50/P95、缺独立危险标签、缺校准、缺区间）；
- `NEXT_STEPS.md` 六条全是"还需要什么"，没有一条是"我们做到了什么"；
- 三层测试：Go 单测 10 项 + `go vet` + `go test -race` + Python 15 项（其中 6 项**真的启动 Go 进程走 HTTP**）。

---

## 3. 缺点：问题在哪

### 3.1 最致命：这个负结果可以被一个常数完全解释

`QUALITY.md:22` 报告的对照是：B2 固定规则 STC 43.8%，本方法 21.2%，危险写 11 vs 10。

我把 `results/pilot.json` 的五行数据和 `risk_samples.json` 的常数一起算了一遍，结论是：**本方法的 21.2% 不是"安全—可用性权衡"，而是"预算设小了 8.6%"**。

推导（可逐位复核）：

```text
risk_samples.json:  task_budget = 0.72
process.change_valve 的两个危险: (k=5, n=1000, sev=5), (k=3, n=1000, sev=2), confidence=0.99
  Bonferroni: adjustment = 0.01/2 = 0.005
  Hoeffding 项: sqrt(ln(200)/2000) = 0.05147
  h1: (0.005+0.05147)*5 = 0.2823
  h2: (0.003+0.05147)*2 = 0.1089
  UpperRisk(process.change_valve) = 0.3913
```

于是 `experiment.py:72-77` 的判断变成：

```text
第 1 次 change_valve:  room = 0.72,  estimate 0.3913 ≤ 0.72 → 允许, spent = 0.3913
第 2 次 change_valve:  room = 0.72 - 0.3913 = 0.3287
                       estimate 0.3913 ≤ 0.3287  →  假  →  BLOCK
```

**而 Python 的 process 任务模板恰好就是 `read_pressure → change_valve → change_valve`（`experiment.py:37`），所以 100% 的 process 任务都在最后一步被拒，STC 恒为 0。**

总 STC = (0 + 0.42)/2 ≈ **0.21**，与报告的 0.212 完全吻合。也就是说：

> **"本方法把 process 域任务完成率打到 0"，唯一原因是 0.72 < 2 × 0.3913 = 0.7826。**

只要 `task_budget ≥ 0.79`，两次数值动作就能通过，`ours` 会立刻回到与 B2 同一水平（~0.42），差别只剩"多少动作被预算挡住"。**所以到目前为止，这套机制从未在一个可行预算下被真正检验过。** 这是整份工作最需要立刻修的一点，而且成本极低（改一个数字 + 重跑）。

### 3.2 风险模型没有实例级判别力，与立项书 §4.2 直接脱节

立项书 §4.2 定义的是：

```text
U_h(a_t, s_t, D_t)     ← 危险概率上界依赖动作、状态、历史证据
r_t(a_t) = Σ_h L_h · U_h(a_t, s_t, D_t)
```

而代码里（`model.go:194`）：

```go
increment, estimated = UpperRisk(e.cfg, t.state.Domain+"."+a.Operation)
```

**`s_t` 被完全丢掉了。** `risk_samples.json` 一共只有 4 个 stratum（`process.change_valve`、`workflow.approve_case`、`workflow.issue_credit`、`workflow.send_notice`），所以：

- 每一个 `change_valve` 动作拿到的风险增量**永远等于 0.3913**，与压力高低、阀门状态、传感器是否刚回读、前序是否报警全都无关；
- 风险预算因此退化成一个**"你还能做几次同类动作"的计数器**，而不是"这个动作此刻有多危险"的估计。

这恰好把立项书最核心的一句话架空了——§4.2 的解释第 1 条说"`Σ L_h U_h` 是保守代价代理量"，但代理量如果不含状态，它就无法区分同一类动作中的危险与安全实例。**审稿人只要看这一个函数，就会问"你的不确定性到底在估什么"。**

### 3.3 六个"策略"实际上只有三个行为

看 `results/pilot.json` 五种子均值：

| 策略 | STC | unsafe | blocked |
|---|---:|---:|---:|
| B0_no_guard | 0.638 | 146 | 0 |
| B1_read_only | 0.000 | 0 | 500 |
| **B2_fixed_rules** | **0.438** | **11** | **268** |
| **B3_point_threshold** | **0.438** | **11** | **268** |
| B4_fixed_budget | 0.000 | 7 | 500 |
| ours_upper_dynamic | 0.212 | 10 | 386 |

**B2 与 B3 三个指标完全相同（438/11/268）**，这不是巧合：

- `B3` 用点估计（`upper=False`），`process.change_valve` 的点估计是 `5/1000*5 + 3/1000*2 = 0.031`；
- 阈值是 `estimate <= 0.20`（`experiment.py:69`）；
- `0.031 ≪ 0.20`，所以 **B3 从来不会因为风险被拒**，它的行为等价于"只看 `observed_safe` 就直接放行"——也就是 B2。

同理 `B4` 的阈值是 `task_budget/3 = 0.24`，`approve_case`=0.153 能过、`issue_credit`=0.270 过不去，于是 workflow 全拒、process 也全拒 → STC 恒 0。

**结论：这张表比较的不是"三种估计方法"，而是三个不同的人为常数阈值（0.20 / 0.24 / 0.72 预算）。** 立项书 H1 说"相比 …点估计策略…能改善联合指标"——这个对照在数据里根本不存在。要让它存在，阈值必须落在**会 bind**的地方（即 0.01–0.05 量级，而不是 0.20）。

### 3.4 离线任务模板与 Go 的互锁互相矛盾

这是我单独挑出来的一条，因为它会让"完成率"这个指标失去意义。

- **Go 侧**：`change_valve` 执行后把 `SensorAgeMS` 设为 300，而前置条件要求 `SensorAgeMS ≤ 250`。所以**两次连续的 `change_valve` 在真实网关里必然失败**，中间必须夹一次 `read_pressure`。
- **Python 侧**：process 任务序列写死为 `("read_pressure", "change_valve", "change_valve")`（`experiment.py:37`），而 `gate()` 里**没有任何 `precondition` 逻辑**。

于是：离线实验里被记为"如果放行就能完成"的那些 process 任务，**在 Go 里本来就完不成**。这会让 `safe_task_completion` 的绝对值（43.8% / 21.2%）失去工程含义——它衡量的是一个不存在的任务模板。

### 3.5 被测量到的主要是"观测噪声"，不是风险预算

`experiment.py:41-43`：

```python
is_safe = rng.random() >= 0.14 + drift        # ~86% 真安全
observation_flip = rng.random() < 0.13 + drift/2
observed_safe = is_safe != observation_flip   # 13% 的观测是错的
```

所有策略（除 B0/B1）**第一道门都是 `if not item.observed_safe: return False`**（`:55-56`）。所以：

- 残余的 `unsafe_executions` 几乎全部来自"真不安全但观测成安全"的那 13%×14% 组合；
- 一旦 `observed_safe` 过滤掉大部分，**风险预算模块能起作用的样本所剩无几**；
- 换句话说：**这张表主要在测"13% 的观测错误率"，风险模型的边际贡献被压到噪声级别。**

更好的设计是让 `observed_safe` 与 `true_safe` 的相关性可变（例如 0.95/0.8/0.6 三档），并把"观测可靠性"作为一个显式的实验轴——立项书 §6.2 的"传感器异常"轴本来就是这个意思，但代码里没有实现。

### 3.6 缺 P50/P95、缺成本、缺校准

`EXPERIMENTS.md:11` 自己承认："缺少基线真实 Agent、P50/P95 延迟、独立工程危险标签、校准测试、置信区间及人类专家审核。Brier 等统计指标尚未具备必要观测条件。"

对照立项书 §6.4 的 10 类指标：UER ✅、STC ✅、FBR ❌、**Risk Calibration ❌**、Budget Violation ✅（但恒 0）、P50/P95 ❌、Cost ❌、Abstention ✅（部分）、Robustness ⚠、Evidence Coverage ❌。

**尤其是"Risk Calibration"缺失是硬伤**：整篇论文的卖点是"带统计置信度的风险上界"，却没有任何可靠性图 / Brier / 覆盖率证据说明这个上界真的覆盖了。

另外 `proxy_budget_violations` 在两张表里**恒为 0**——因为 `allowed` 条件本身要求 `estimate <= room`，超限在结构上不可能发生。这个指标目前是装饰性的。

### 3.7 OOD 只是把同一个生成器的参数放大

`ood_pilot.json`（drift=0.2）：

| 策略 | STC | unsafe | blocked |
|---|---:|---:|---:|
| B0_no_guard | 0.302 | 332 | 0 |
| B2_fixed_rules | 0.156 | 37 | 392 |
| ours_upper_dynamic | 0.080 | 30 | 449 |

`drift` 只做了两件事：`is_safe` 阈值从 0.14 抬到 0.34、翻转率从 0.13 抬到 0.23。**分布的形状没变、生成机制没变、危险类型没变。** 所以这不是立项书 H2 说的"工具故障率、危险状态分布、传感器缺失和恶意工具结果发生变化"，而只是"同一个伯努利换了个 p"。

而且 ours 依然只是"更少放行、略少危险"（37→30），代价是 STC 0.156→0.080。**与 pilot 完全同构的负结论。**

### 3.8 其他工程问题

| # | 问题 | 证据 |
|---|---|---|
| 1 | **无身份/授权层**，任何人都能对任何 task_id 发动作 | `QUALITY.md:11` 自认；`server.go` 无 auth 中间件 |
| 2 | Go 账本在内存，重启即丢，不能证明 exactly-once | `model.go:112-118` + `QUALITY.md:11` |
| 3 | `remainingRiskSteps()` 是写死的任务模板估计（按 domain 返回 2/1/3/2/1），与真实任务图无关 | `model.go:333-349` |
| 4 | `Decision.Snapshot` 里带完整状态，包括 `approved/credited`，若前端误用可能泄露业务进度 | `model.go:99` |
| 5 | `hold` 被算作 readOnly，因此拒绝时的 `SuggestedAction="hold"` 可能被 Agent 当成"可以随便执行" | `model.go:244` |
| 6 | 29 个 `docs/files/*.md` 生成页；`research/01x_original.md` 是 43 KB 逐字节副本 | — |

---

## 4. 项目自己已经承认的负结果与红线

1. **"不能据此声称提出的方法取得研究胜利"**（`QUALITY.md:22`）。
2. **"Hoeffding 上界需要统计样本代表性等前提，此包数据为虚构；不能宣称概率事故保证"**（`QUALITY.md:12`）。
3. **"阶段预算分配为可检验启发式，不是经过优化的最优算法"**（`QUALITY.md:13`）。
4. **"离线三步合成任务未建模完整物理状态转移……没有工程领域的独立标签"**（`QUALITY.md:14`）。
5. **"不能把离线模拟的效果直接当成 Go 联机吞吐或真实安全性"**（`EXPERIMENTS.md:3`）。

**这五条让这份包不会被误读**——尤其第 5 条，主动切断了"离线数字冒充系统性能"的最常见包装路径。

---

## 5. 改进清单

### P0 — 不做这些，方法一次都没被真正检验过

| # | 要做什么 | 为什么 | 验收标准 |
|---|---|---|---|
| P0-1 | `task_budget` 0.72 → **≥0.85**，并加 `--budget-scan 0.5/0.72/0.9/1.2`，输出 STC–unsafe–budget 的 Pareto 曲线 | 现在 50% 任务被常数卡死（§3.1） | 在 0.9 预算下 `ours` 的 STC ≥ 0.40，且 unsafe 仍显著低于 B2；曲线成为论文主图 |
| P0-2 | **风险估计加状态维度**：`UpperRisk(stratum, state_bucket)`，bucket 至少含 pressure 区间、valve 状态、sensor age 是否新鲜 | 现在风险与 `s_t` 无关（§3.2），与立项书 §4.2 不符 | 同一 `change_valve` 在不同状态下得到不同风险值；新增单测断言 bucket 差异 |
| P0-3 | **重做 B3**：点估计阈值降到会 bind 的位置（如 0.02），或直接删掉 B3 并声明"点估计与硬阈值在本设置下无差别" | B2 与 B3 三项指标完全相同（§3.3） | B2 与 B3 出现可区分的 STC 或 unsafe 差异，或在论文里如实说明二者等价 |
| P0-4 | **统一离线/在线任务语义**：让 `experiment.gate()` 复用 Go 的 `precondition`，或把 process 模板改成 `read → valve → read → valve` | 现在离线模板在 Go 里必然失败（§3.4） | 同一条任务序列在 Python 与 Go 下得到相同的可完成性判定 |

### P1 — 决定论文能不能过审

| # | 要做什么 | 验收标准 |
|---|---|---|
| P1-1 | **补校准评估**：风险上界的实测覆盖率、可靠性图、Brier、按 stratum 分层 | 有一张"预测风险 vs 实际危险率"图；覆盖率与名义 confidence 的偏差被如实报告 |
| P1-2 | **补 P50/P95 与成本**：Go HTTP 路径的端到端延迟、每动作平均耗时、额外读回次数 | 指标的取值来自实测而不是合成常数；离线与在线分开报 |
| P1-3 | **把观测可靠性做成实验轴**：`observed_safe` 与 `true_safe` 的相关性 0.95/0.8/0.6 三档 | 能回答"当传感器不可靠时，风险预算是否比固定规则更稳"——这才是 H2 真正该问的 |
| P1-4 | 给 `proxy_budget_violations` 一个可能非零的实现（例如允许策略"超支但记录"） | 该指标不再恒为 0；或明确从指标表里删掉 |
| P1-5 | 补一个**跨领域**场景（第二个 process 模型：温度—加热器），而不是靠同一模板参数化 | 满足立项书 §6.1"阶段 II 工程域"的最低要求 |

### P2 — 可读性与部署

| # | 要做什么 |
|---|---|
| P2-1 | 加最小身份层（token → principal 映射即可），并补"跨 task 越权"测试 |
| P2-2 | 删 29 个生成页；`research/01x_original.md` 改为指针 |
| P2-3 | 在 Windows 真机跑一遍 `check_windows.bat` 并留下日志（目标读者就是 Windows 学生） |
| P2-4 | `remainingRiskSteps()` 改成从任务图读，或在注释里明确它是"demo 模板常数" |

---

## 6. 只改五件事的话

1. `task_budget` 改大 + 预算扫描 → 让方法第一次跑在可行区间。
2. 风险估计加状态 → 让"不确定性"真的有内容。
3. 修 B3 的阈值 → 让"点估计 vs 上界"这个对照成立。
4. 统一离线/在线任务语义 → 让 STC 有工程含义。
5. 补校准曲线 → 让"统计置信度"这个卖点有证据。

---

## 7. 在全系列里的定位建议

| 关系 | 说明 |
|---|---|
| 与 02X（科学仿真） | 02X 的"PASS_WITH_EVIDENCE vs PASS_UNVERIFIED"与本文的 `defer/block/escalate` 是同一套决策语义，可以共享一个策略中间表示 |
| 与 03X（电力） | 03X 的"`max_{u∈U_t}` 最坏情况硬约束"就是本文风险上界的一个变体；两篇的"过度保守"问题是同一个病 |
| 与 04X（因果诊断） | 04X 的"证据等级"可直接给本文的 `evidence_required` 字段提供分级 |
| 与 05X（安全博弈） | 05X 的"硬授权永不让步 + 软资源优化"分层，正好是本文的架构描述的一个更严谨版本 |

**推荐定位**：**五份里最应该先做、也最可能先出结果的一份**，理由不是它已经领先，而是：

- 负结果的成因是**可修复的单点**（一个常数），而不是方法论缺陷；
- 它的 Go 引擎已经具备"可信执行面"的雏形（版本、互锁、幂等、审计），这是别人要花 4–6 周才有的东西；
- 目标刊 RESS / Safety Science 都吃"工程可靠性 + 可复现实验"这一套，与它的强项吻合。

**给学弟的第一周任务**（我建议直接照抄）：

1. 把 `task_budget` 改成 0.9，跑 `evaluate.py --tasks 100 --seeds 5`，把新表贴进 `docs/QUALITY.md`；
2. 把 `B3` 的阈值改成 0.02，看它是否终于与 B2 分开；
3. 读立项书 §4.2，写出 `U_h(stratum, state_bucket)` 的 5 行分桶规则，在 Go 里加一个单测。

---

## 附录：文件地图

```text
01x.md                          ← 立项书（502 行）：§3.2 危险合约、§4.2 数学、§6.4 指标
01xcode/
├─ contracts/risk_samples.json  ← **只有 4 个 stratum + task_budget=0.72（问题源头）**
├─ contracts/HAZARD_CONTRACT.md ← 危险合约的文档形态
├─ internal/core/model.go       ← 349 行，**互锁 + 风险账本 + 幂等（最核心）**
├─ internal/httpapi/server.go   ← 126 行，严格 JSON + 无认证（注意）
├─ python/rabgate/experiment.py ← 153 行，**六策略 gate()（负结果在此产生）**
├─ python/rabgate/risk.py       ← 27 行，Hoeffding + Bonferroni（写得正确）
├─ docs/EXPERIMENTS.md          ← 20 行，**实验口径与五个隐含限制**
├─ docs/QUALITY.md              ← 23 行，**负结果表 + 三处自我限权**
└─ results/pilot.json           ← 344 行，五个种子 × 六策略的原始数据
```

**阅读顺序建议**：`contracts/risk_samples.json` → `python/rabgate/experiment.py`（`gate()` 那 30 行）→ `internal/core/model.go` 的 `precondition` 与 `Execute` → `docs/QUALITY.md`。
