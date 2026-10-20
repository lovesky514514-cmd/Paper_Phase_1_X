# 04dst.md — 第四篇（04X）评估：CausalGuard-RCA 预算约束的安全干预式根因诊断

> **评估对象**：`C:\Users\20050224\Desktop\04X\`（立项书 `04x.md` 46,205 B + 代码包 `04xcode.zip`）
> **评估方式**：**只读**。已读完立项书、全部 8 个 Python 模块与 5 个测试、Go 采集端、`results/pilot.json`（120 例 × 5 策略 + 按模式拆分）。
> **未执行**：没有跑 `python -m causalguard.study`。数字来自包内 `results/pilot.json` 与 `docs/EXPERIMENT.md`。

---

## 0. 一页速览

| 维度 | 评分（1–5） | 一句话依据 |
|---|:---:|---|
| 选题价值与撞题风险 | **4.0** | "可修复原因 ≠ 唯一根因""干预有风险、有预算"是全系列最锐利的切口；但 Who&When / AgentRx / DoVer / REFLECT 已把赛道挤满 |
| 立项文档质量 | **4.5** | 四个概念严格定义、12 项消融、三层实验结构、"不允许把摘要重写当复现"——研究素养很高 |
| 原型实现完成度 | **3.5** | 配对沙箱与安全许可做得好，但**两个上报指标是硬编码常量**（见 §3.2） |
| 结果可信度 | **1.5** | `anomaly` 由隐藏故障直接生成且**有干净间隙**；于是 `value` 与 `anomaly` 分毫不差（见 §3.1） |
| 工程与仓库质量 | **3.5** | 单测 5 组、Go 采集端只读；扣分在硬编码指标与 37 个生成页 |
| **总分** | **17 / 25** | 五份里排第 4 |

**一句话结论**：**这是五份里"研究问题问得最漂亮、实验把它做废了"的一份。** 它想证明"在有限干预预算下，选对干预比选对异常点更重要"——但在它的合成数据里，**可观测的 `anomaly` 分数是由隐藏故障集合直接生成的，并且故障与非故障之间有干净的 0.61 / 0.63 分界**。于是"按异常分排序"就已经最优，`value` 策略再聪明也只能和它打平。

**如果只能改三件事**：

1. **打断 `anomaly` 与标签的耦合**：让 `anomaly` 与故障只有概率相关（例如非故障 `U(0.2,0.8)`、故障 `U(0.4,0.9)`，并加入"高异常但无因果作用"的干扰段）。现在 `generate.py:42` 是确定性分离。
2. **修 `visible_ancestors` 的"断链"问题**：缺失一个中间 span 会让**它上面所有 span 一起消失**（`graph.py:17-18`）。这一个 bug 把 single 模式的成绩上限压到 0.50，并让所有策略并列。
3. **把 `abstained_unique` 和 `unsafe_replays` 从常量变成观测量**。现在 `diagnose.py:155` 写死 `True`、`:159` 写死 `0`，而这两个字段正好对应立项书的两条主指标。

---

## 1. 项目构成

| 层 | 内容 | 规模 |
|---|---|---|
| 立项书 | `04x.md`，与包内 `research/04x_original.md` **逐字节相同** | 536 行 |
| 研究层 | `generate.py`（合成 SCM）、`sandbox.py`（配对回放）、`diagnose.py`（干预选择）、`graph.py`、`budget.py`、`traces.py`、`study.py` | 约 426 行 |
| Go 采集端 | `collector/ingest/collector.go`（只读元数据）+ 测试 | 约 211 行 |
| 测试 | `test_diagnose / test_graph / test_model / test_replay / test_study` | 约 206 行 |
| 结果 | `results/pilot.json`（120 例 × 5 策略，含 `by_mode` 拆分） | 6,725 行 |
| 文档 | `README / ARCHITECTURE / EXPERIMENT / QUALITY / NEXT_STEPS / TROUBLESHOOTING / GLOSSARY / CODING_STANDARDS` + **37 个生成页** | 47 个 md |

**实验规模**：120 个合成案例，3 类因果逻辑（`single` / `or` / `and`）× 3 种 DAG（`chain` / `fork` / `join`），缺失率 0.15，回放预算 48。**没有真实轨迹、没有官方基线复现**（`EXPERIMENT.md:9` 自认）。

---

## 2. 优点：它真的做对了什么

### 2.1 配对反事实的设计是对的：同一噪声流给"改前"和"改后"

这是本项目的技术核心，而且写得比大多数原型都正确。`sandbox.py:51-57`：

```python
def _failed(self, repaired, trial):
    active = self.__faults - repaired
    structural = bool(active) if self.__mode in {"single","or"} else (active == self.__faults)
    # Independent nuisance generated from the same random stream for paired trials.
    digest = hashlib.sha256(f"{self.__seed}/{trial}".encode()).digest()
    noise = random.Random(int.from_bytes(digest[:8], "big")).random()
    return structural or noise < self.__nuisance
```

关键在第 55–56 行：**噪声由 `(seed, trial)` 决定，与 `repaired` 无关**。所以：

- `before = _failed(∅, t)` 与 `after = _failed(repaired, t)` 用的是**同一个随机数**；
- 两者之差只反映"修复"本身，不混入随机波动；
- 这是配对反事实（paired counterfactual）的正确写法，也是 `diagnose.py:132-135` 能算出干净 `improvement` 的前提。

绝大多数"反事实归因"原型在这里会各跑一次随机，导致差值被噪声吞掉。

### 2.2 安全回放被做成**六字段可检查契约**，不是一句承诺

`sandbox.py:9-21`：

```python
@dataclass(frozen=True, slots=True)
class ReplayPermit:
    isolated: bool = True
    egress_blocked: bool = True
    snapshot_complete: bool = True
    user_authorized: bool = True
    state_version_matches: bool = True
    all_writes_mocked: bool = True

    @property
    def allowed(self):
        return all((self.isolated, self.egress_blocked, self.snapshot_complete,
                    self.user_authorized, self.state_version_matches, self.all_writes_mocked))
```

六个字段一一对应立项书 §3.5 的六条边界：隔离、出站阻断、快照完整、用户授权、状态版本、写入全模拟。**任何一条为假，`paired_replay` 直接抛 `ReplayDenied`**（`:60-61`），诊断降级为 `replay_contract_not_satisfied`（`diagnose.py:110-111`）。

这比"我们在沙箱里做诊断"这种描述强一个量级——它是可以被审稿人逐条检查的。

### 2.3 外部写操作有**两道**拦截，而不是一道

`diagnose.py:66-67` 在生成候选时就过滤掉 `external_write`：

```python
eligible = [s for s in visible_ancestors(trace) if s.operation in {"read", "mock_write"}]
```

然后 `:123-125` 在循环里**再查一次**：

```python
span_ops = {span.span_id: span.operation for span in trace.spans}
if any(span_ops[t] == "external_write" for t in targets):
    continue
```

注释写得很清楚："check once more so a future selector cannot bypass the safety boundary"（`:121-122`）。**这是"纵深防御"的意识**——不依赖上游筛选的正确性。

### 2.4 三类因果逻辑是对的真问题

`generate.py:50-51` 与 `sandbox.py:53` 的组合语义：

| 模式 | 失败条件 | 最小修复集 | 对应立项书的说法 |
|---|---|---|---|
| `single` | 有任一故障 | 那一个 | 单根因 |
| `or` | 有任一故障（需全修） | **两个一起** | "A、B 任一都可能独立导致失败，单独修 A 仍失败" |
| `and` | **两个同时存在**才失败 | **任一单修** | "A、B 同时出现才失败，任一替换都可能恢复" |

**这三种恰好覆盖了立项书 §3.4 想说清的多原因/交互效应**，而且 `ground_repair_sets` 与语义是自洽的。这比只做"单根因"的实验有信息量得多。

### 2.5 诚实度

- `EXPERIMENT.md:11` 主动说明 `lower_bound` **"不是严格的净因果效应 CI，更不是跨域可信安全证书"**；
- `EXPERIMENT.md:25` 直接写 **"目前 `value` 未超过简单 `anomaly`；没有证明统计显著新优势"**；
- `EXPERIMENT.md:9` 说明五个基线 **"全是本项目自行实现的本地基线，不是官方 AgentRx/DoVer 复现"**；
- `QUALITY.md:7` 列出七项"尚未实现"。

**它没有把"打平"包装成"相当"或"具有可比性"。** 这一点值得保留。

---

## 3. 缺点：问题在哪

### 3.1 最致命：`anomaly` 由隐藏标签直接生成，且有干净间隙

`generate.py:42` 是一行决定全篇命运的数据生成代码：

```python
anomaly = (0.63 + 0.37 * rng.random()) if name in faults else (0.06 + 0.55 * rng.random())
```

拆开看：

| | 取值区间 |
|---|---|
| 故障段 | `[0.630, 1.000]` |
| 非故障段 | `[0.060, 0.610]` |

**中间有 0.02 的绝对间隙。** 也就是说，`Span.anomaly` 这个"可观测分数"**完美地、确定性地**编码了隐藏的 `faults` 集合。任何按 `anomaly` 降序排列的选择器，都会**无一例外**先测到真故障段。

后果直接体现在数据里：

| 策略 | 沙箱确认修复率 | 真值最小集合命中 | 平均回放 |
|---|---:|---:|---:|
| **value（本方法）** | **54.17%** | **54.17%** | 36.67 |
| **anomaly（简单基线）** | **54.17%** | **54.17%** | 36.67 |
| random | 45.00% | 15.00% | 36.67 |
| uncertainty | 35.00% | 6.67% | 36.67 |
| cost_only | 39.17% | 39.17% | 36.67 |

**`value` 与 `anomaly` 三个数字完全一致（54.17 / 54.17 / 36.67）** —— 不是巧合，而是因为两者的排序基础都是同一个泄漏了标签的分数。

> 这张表实际测量的是：**"你的选择器离 `-anomaly` 排序有多近"**。
> `value` 的 `+1.25` 配对加成与 `-0.08` 成本惩罚（`diagnose.py:82-83`）**没有产生任何可测量的差异**。

这意味着立项书 §0 的第 1 条核心主张（"受约束的干预选择能在同预算下更高效地发现可修复故障"）**在当前数据上不可能被证实**——因为"最朴素的选择器"已经是理论最优。

### 3.2 两个上报指标是**硬编码常量**

`diagnose.py:155-160`：

```python
return Diagnosis(trace.trace_id, strategy, tuple(minimal), tuple(evidence),
                 True,                    # ← abstained_unique 永远 True
                 reason, round(coverage, 4),
                 {...}, 0,                # ← unsafe_replays 永远 0
                 round(...), "...")
```

对照 `study.py:44`：

```python
"unsafe_replays": sum(r["unsafe_replays"] for r in rows),   # 恒为 0
```

问题有两层：

1. **`abstained_unique=True` 是写死的**，意味着"弃权"这个行为**从来没有被真正判定过**。立项书 §3.4 把"abstain on unique root cause"当作主创新之一，结果这个字段是个常量。
2. **`unsafe_replays=0` 是写死的**，而立项书 §5.3 把 `Unsafe replay count` 列为安全指标，并写"目标应为 0"。它确实是 0，但**不是测量出来的 0，是字面量 0**。这在论文里是不能接受的——审稿人看到"我们的不安全回放数为 0"一定会问"你怎么测的"，答案是"我们 print 了 0"。

**修法很简单**（各约 10 行）：`unsafe_replays` 改成"回调放时 `permit.allowed` 失败的次数 + `external_write` 被拦截的次数"；`abstained_unique` 改成 `len(minimal) != 1`。

### 3.3 `visible_ancestors` 的"断链"把成绩上限压死了

`graph.py:13-20`：

```python
queue = list(terminal.parents)
while queue:
    current = queue.pop()
    if current in found: continue
    if current not in by_id:
        continue          # ← 关键：遇到缺失节点直接放弃，不再往上走
    found.add(current)
    queue.extend(by_id[current].parents)
```

`by_id` 只包含**被观测到的** span（缺失的已被 `generate.py:40-41` 跳过）。所以在 `chain` 拓扑下：

```text
finish ← report ← cache ← analyze ← validate ← fetch
```

**只要 `report` 缺失，整个祖先集合就是空集**，候选数为 0，诊断必然失败。更一般地，要走到 `fetch` 需要 `report/cache/analyze/validate` 全部存在，概率 `0.85⁴ ≈ 0.52`。

这解释了数据里一个很反直觉的现象：

| 策略 | single 模式确认率 | or 模式 | and 模式 |
|---|---:|---:|---:|
| value | 0.500 | 0.525 | 0.600 |
| anomaly | 0.500 | 0.525 | 0.600 |
| random | **0.500** | 0.250 | 0.600 |
| cost_only | **0.500** | 0.075 | 0.600 |
| uncertainty | 0.475 | 0.000 | 0.575 |

**在 single 模式下，连 `random` 都拿到了 0.500——和本方法一模一样。** 原因就是：single 模式只有 1 个故障段，而候选池里只有 5–6 个单点集 + 15 个配对集，**在 48 次回放预算（6 个候选）内，随机也几乎一定会测到那个故障段**；剩下的失败全部来自"链断了，故障段压根不在候选池里"。

所以 single 模式的 0.50 **不是策略能力的度量，而是"数据完整性"的度量**。立项书 RQ1（"同预算下比随机更高效地发现可修复故障"）在 single 模式上**根本无法被检验**。

**修法**：`Trace` 里保留完整的父子指针表（拓扑），只把缺失 span 的**载荷**去掉。这样"看到 report 但看不到 cache"仍然能沿 `report→cache`（指针存在）继续向上找 `analyze`。这是"部分可观测"的正确建模，也是 10 行以内的改动。

### 3.4 `and` 模式下"任意含故障的集合"都算修复，导致最小性失效

`sandbox.py:53`：

```python
structural = bool(active) if mode in {"single","or"} else (active == self.__faults)
```

在 `and` 模式下，失败条件是"两个故障**都在**"。所以：

- 修复 `{f1}` → `active = {f2} ≠ faults` → 不失败 → **成功**；
- 修复 `{f1, x}`（x 是无关段）→ `active = {f2}` → **也成功**。

于是 `and` 模式下**任何包含至少一个真故障的候选集都会产生正效应**。而 `diagnose.py:146` 的 minimal 过滤：

```python
minimal = sorted((s for s in supported if not any(set(t) < set(s) for t in supported)))
```

**只能在"已被测试且已支持"的集合之间去最小性**。如果 `{f1}` 恰好没被测试，而 `{f1, x}` 被测到了，那么 `{f1, x}` 会被当成"最小修复集"，计入 `verified_repair_rate`，但**不计入** `exact_repair_set_hit_rate`。

这正是 `random` 那一行的来源：**verified 45.00% vs exact 15.00%**。差的 30 个百分点全是"含故障但并不最小的集合"。

**结论：`verified_repair_rate` 这个名字在 `and` 模式下名不副实。** 建议改名为 `any_effective_intervention_rate`，并把 `exact_repair_set_hit_rate` 升为**唯一主指标**——否则论文会用一个宽口径指标拿到好看的数，再被审稿人拆穿。

### 3.5 4 次试验 + 0.45 阈值 = 只接受满分

`diagnose.py:135`：

```python
lower = _wilson_lower(improvements, trials) if harms == 0 else 0.0
```

`trials = 4`、`min_lower_bound = 0.45`。算一下 Wilson 下界：

| improvements / 4 | Wilson 95% 下界 | 是否 ≥0.45 |
|---:|---:|:---:|
| 4 | 0.510 | ✅ |
| 3 | 0.301 | ❌ |
| 2 | 0.150 | ❌ |

**只有 4/4 才能通过。** 而每次试验有 4% 的固有噪声（`nuisance = 0.04`），所以即使候选集完全正确，**单次"4/4"的概率也只有 0.96⁴ = 84.9%**。也就是说：**约 15% 的"其实找对了"的案例，会因为一次随机抖动被判为"未找到支持"。**

这是一个可以算出来的系统性扣分，而且它与方法好坏无关。**建议**：`trials` 提到 10，`min_lower_bound` 相应重算（10/10 下界约 0.72，8/10 约 0.49），让判定对单次抖动不敏感；或者改用"效应显著 >0"的配对检验而不是"下界 ≥ 常数"。

### 3.6 所有策略的预算消耗**完全相同**（36.67），预算模块未被检验

五个策略的平均回放都是 **36.67**，一分不差。这说明：

- 每个案例都测到预算耗尽（或候选耗尽），**策略之间没有"省预算"的差异**；
- 立项书 §3.3 的四个预算维度（`B_token / B_time / B_replay / B_risk`）里，只有 replay 真正起作用，其余三个都是从 replay 线性推出来的（`budget.py:29-32`：`tokens = 6n`、`time = 2n`）；
- **`max_risk_units = 0` 且 `risky` 参数永远不被传 `True`** → 风险预算这一维**完全没启用**。

所以"预算约束的干预选择"这个卖点在数据里只体现为"48 次回放够测 6 个候选"。**必须做预算扫描**（`--replays 8/24/48/96`），否则"效率"两个字没有证据。

### 3.7 其余

| # | 问题 | 证据 |
|---|---|---|
| 1 | 缺失率恒为 0.15，没有做 0/20/40/60% 的鲁棒性曲线（立项书 §5.4 A9 要求分级） | `study.py:16-17` |
| 2 | 没有种子重复：只有 seed=19 一个点，`EXPERIMENT.md` 也没有 CI | `study.py:52` |
| 3 | 候选池是"单点 + 全部配对"，规模随 span 数平方增长（n=6 → 21 个候选，只测 6 个），但**没有任何候选剪枝** | `diagnose.py:68-70` |
| 4 | `wall_ms` 被记录但从未用于任何结论 | `diagnose.py:159` |
| 5 | `cost_only` 排序是 `(len, ids)`，等价于"按字母序测单点"，作为"成本优先"基线过于弱 | `diagnose.py:74-75` |
| 6 | `Traces` 无真实数据；`collector` 的 Go 端只被 `scripts/demo.py` 用过，未进实验 | `README.md:24` |
| 7 | 37 个生成页 + 46 KB `research/04x_original.md` 副本 | — |

---

## 4. 项目自己已经承认的负结果与红线

1. **"目前 `value` 未超过简单 `anomaly`；没有证明统计显著新优势"**（`EXPERIMENT.md:25`）。
2. **"`lower_bound` 不是严格的净因果效应 CI，更不是跨域可信安全证书"**（`EXPERIMENT.md:11`）。
3. **"全是本项目自行实现的本地基线，不是官方 AgentRx/DoVer 复现"**（`EXPERIMENT.md:9`）。
4. **"`intervention_supported_in_synthetic_sandbox` 不等于生产因果证明；只有合成 SCM 有干预真值"**（`README.md:29`）。
5. **"Go 本地监听无身份认证，不可暴露到公网或作为生产安全边界"**（`QUALITY.md:7`）。

**第 1 条是整份工作的核心事实。** 我做的分析（§3.1/§3.3/§3.4）解释了这个事实**为什么必然发生**——这是对作者最有用的补充。

---

## 5. 改进清单

### P0 — 不做这些，方法无法被评估

| # | 要做什么 | 为什么 | 验收标准 |
|---|---|---|---|
| P0-1 | **打断 `anomaly` 与标签的耦合**：非故障 `U(0.2,0.8)`、故障 `U(0.4,0.9)`（重叠），并加入 1–2 个"高异常但无因果作用"的干扰 span | 现在 `anomaly` 是标签的确定性函数（§3.1） | `anomaly` 基线的确认率**下降**（目标 ≤0.40），而 `value` 若仍有优势则必须给出配对 CI 不含 0 |
| P0-2 | **修 `visible_ancestors` 的断链**：用"完整拓扑 + 缺失载荷"代替"缺失即断链" | 一个缺失 span 会抹掉整条上游（§3.3），把成绩上限压到 0.50 | single 模式下所有策略的确认率从 0.50 升到 ≈0.72；`random` 明显低于 `anomaly` |
| P0-3 | **把 `abstained_unique` 与 `unsafe_replays` 变成真实计算** | 现在是字面量 `True` / `0`（§3.2），而它们对应两条主指标 | 两个字段随输入变化；新增单测：人为构造不安全回放，`unsafe_replays` 必须 >0 |
| P0-4 | **主指标换成 `exact_repair_set_hit_rate`**，或把 `verified_repair_rate` 改名 | `and` 模式下任意含故障的集合都被计入（§3.4） | 论文主表以"最小集合命中"为准；"任意有效干预"作为辅助指标并注明口径 |

### P1 — 决定论文能不能站住

| # | 要做什么 | 验收标准 |
|---|---|---|
| P1-1 | **预算扫描**：`--replays 8/24/48/96` × 5 策略，出"预算—确认率"曲线 | 能看出"本方法在低预算时更优"或"根本无差别"，二者都是有价值的结论 |
| P1-2 | `trials` 4 → 10，`min_lower_bound` 重算；或改用配对显著性检验 | 4% 噪声不再导致 15% 的假阴性（§3.5） |
| P1-3 | 缺失率 0 / 0.15 / 0.4 / 0.6 四档 + 种子 ≥5 | 出"缺失率—确认率"曲线；立项书 RQ3 才有答案 |
| P1-4 | 加一个**作弊上界**策略（直接读 `ground_repair_sets`）作为天花板 | 读者能看出当前方法离天花板多远 |
| P1-5 | 接真实轨迹：AgentRx（115 条已标注）或 Who&When，**先统一定义**"观测故障点"与"干预修复点" | 立项书 §5.1 L1 的最低要求；否则只能叫"synthetic only" |

### P2 — 可读性与工程

| # | 要做什么 |
|---|---|
| P2-1 | 删 37 个生成页；`research/04x_original.md` 改为指针 |
| P2-2 | 启用 `max_risk_units > 0` 与 `risky=True` 路径，让"风险预算"这一维真的存在 |
| P2-3 | `cost_only` 基线换成"按干预成本升序"而不是"按字母序"，否则这个对照不公平 |
| P2-4 | 把 Go collector 接进实验（至少让 trace 从真实 HTTP 采集产生），而不是只在 demo 里出现 |

---

## 6. 只改五件事的话

1. **打断 `anomaly` 泄漏** → 让选择策略第一次有存在意义。
2. **修断链 bug** → 让 single 模式下不再全员并列 0.50。
3. **两个硬编码指标改成真实计算** → 否则"弃权"和"零危险回放"都不能写。
4. **主指标换成最小集合命中** → 别用宽口径指标拿好看的数。
5. **预算扫描 + 多种子** → 让"效率"与"鲁棒"两个词有证据。

---

## 7. 在全系列里的定位建议

| 关系 | 说明 |
|---|---|
| 与 01X（可靠性） | 01X 的 `evidence_required` 字段可以直接吃本文的"证据等级"分级；两篇的"预算"概念可以统一成"风险预算 vs 回放预算" |
| 与 02X（科学仿真） | 02X 的 `PASS_WITH_EVIDENCE` 与本文的 `intervention_supported_in_synthetic_sandbox` 是同一种"证据强度标签"，值得统一命名 |
| 与 03X（电力） | 03X 的"隐藏断线/降额"就是本文的"隐藏混杂"，两篇的 Threats to Validity 可以共用一段 |
| 与 05X（安全博弈） | 05X 的"选择性反馈偏差"（被拦的动作没有真实标签）正是本文"只测到被允许的干预"的上位问题 |

**推荐定位**：**问题立意最好、但必须重做数据生成的一份。** 判断依据：

- 它问的是全系列里最难的因果问题（"可修复原因 ≠ 唯一根因"），而且**三种因果逻辑的设计是正确且有价值的资产**；
- 配对沙箱、六字段许可、双重外部写拦截，这三块工程质量足以支撑后续迭代；
- 但它的**数据生成器把答案写进了特征**，导致所有策略并列。修 P0-1 + P0-2 是**半天到一天的工作量**，做完之后实验结果会完全变样——这也是我建议把它排在"值得再抢救一次"的第一位的原因。

**建议的推进顺序**：

1. 先做 P0-2（断链 bug）：纯代码修复，不需要改数据，立刻能看到 single 模式从 0.50 升到 ~0.72，且 `random` 掉下来。
2. 再做 P0-1（anomaly 重叠化）：需要改生成器，但只改一行分布。
3. 然后跑 P1-1（预算扫描），看 `value` 在**低预算**区间是否终于胜过 `anomaly` ——立项书的假设本来就应该在低预算下成立（预算充足时谁都能测到）。
4. 如果低预算下仍然打平，**诚实地把它写成"干预选择在合成因果基准上无优势"的负结果论文**，投 IST/JSS 这类接受实证的刊，同时保留基准与工具作为主要贡献。

---

## 附录：文件地图

```text
04x.md                        ← 立项书（536 行）：§1.2 四概念定义、§2 十篇撞题、§5.4 十二项消融
04xcode/
├─ causalguard/generate.py    ← **53 行，`anomaly` 泄漏标签就发生在 :42**
├─ causalguard/sandbox.py     ← **66 行，配对噪声流 + ReplayPermit（写得最好）**
├─ causalguard/graph.py       ← **21 行，断链 bug 就在 :17-18**
├─ causalguard/diagnose.py    ← 160 行，干预排序 + Wilson 下界 + **两个硬编码字段**
├─ causalguard/budget.py      ← 32 行，四维预算（风险维未启用）
├─ causalguard/study.py       ← 77 行，评测入口
├─ docs/EXPERIMENT.md         ← **29 行，负结果表 + 五条口径说明**
├─ docs/QUALITY.md            ← 7 行，七项未实现
└─ results/pilot.json         ← 6,725 行，120 例 × 5 策略 + by_mode
```

**阅读顺序建议**：`generate.py:42`（先看数据是怎么造的）→ `graph.py:13-20`（再看候选是怎么丢的）→ `docs/EXPERIMENT.md`（看结论）→ `sandbox.py`（看配对设计）→ `diagnose.py:117-160`（看策略与硬编码）。
