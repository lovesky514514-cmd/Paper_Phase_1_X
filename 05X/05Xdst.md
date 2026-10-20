# 05dst.md — 第五篇（05X）评估：GameShield-Gate 预算约束的自适应安全博弈网关

> **评估对象**：`C:\Users\20050224\Desktop\05X\`（立项书 `05x.md` 38,020 B + 代码包 `05xcode.zip`）
> **评估方式**：**只读**。已读完立项书、`gameshield` 全部 5 个模块、Go `gateway/guard`、4 组测试，以及 `results/pilot.json`（4 场景 × 10 策略 × 5 种子 = 200 次实验的完整汇总）。
> **未执行**：没有跑 `study.sh`、没有起 Go 网关。数字来自包内 `results/pilot.json` 与 `docs/*.md`。

---

## 0. 一页速览

| 维度 | 评分（1–5） | 一句话依据 |
|---|:---:|---|
| 选题价值与撞题风险 | **3.5** | "攻击者会适应 + 防御有成本"是这五个里最工程化的博弈角度；但 Stackelberg + 注入已有 CM-PIUG，网关已有 ShieldMCP/CaMeL/Progent/Fides |
| 立项文档质量 | **4.0** | 四层交叉链、双层决策、10 问自查、"随机化不是安全许可"——分寸感很好 |
| 原型实现完成度 | **4.0** | **硬门与学习器在架构上彻底分开**，`Observation` 不含攻击标签，反事实随机流可复现，审计哈希链 |
| 结果可信度 | **2.0** | 主方法 `game` 在 4 个场景里**只有 1 个胜过最朴素的 `bandit`**，在 reactive 场景被 `fixed_deep` 全面压制；两个"增强模块"一个无效一个有害 |
| 工程与仓库质量 | **3.0** | 单测齐全；扣分在 `results/pilot.json` 有 **2.8 MB / 125k 行**、35 个生成页 |
| **总分** | **16.5 / 25** | 五份里排第 5 |

**一句话结论**：**这是五份里架构最正确、结果最难看的一份。** 它把"硬授权永不让步、学习算法只分配额外检查"这个分解做对了——而且做得很干净；然后在实验里证明了：

1. `game`（本方法）在 static / cycle / reactive 三个场景**都不如普通 bandit**，只在 switch 场景略好；
2. 它的**漂移检测模块完全无效**（3 个场景数值一模一样，第 4 个略差）；
3. 它的**随机化模块 3 负 1 正**；
4. 在 reactive 场景，**"永远用最深检查"这个 1 行基线（fixed_deep）在安全性和可用性上同时优于它**。

**如果只能改三件事**：

1. **做预算扫描**（§5 的 Pareto 面）。现在除 `threshold` 外所有策略都把 145 单位预算花到 144，**预算从未真正约束过任何决策**，"预算约束"这个卖点没有证据。
2. **让攻击者真的会适应**。现在的"反应型攻击者"只有一行规则（拒绝多了就从 `metadata` 换成 `memory`），离立项书 H1 的"非平稳攻防"差得很远。
3. **修那个永远为 0 的指标**。`hard_policy_violations` 在 `study.py:59-60` 是**死代码**（硬门在 `:31` 已经拦完），所以 H2 的"零违规"是代码结构决定的，不是实验测出来的。

---

## 1. 项目构成

| 层 | 内容 | 规模 |
|---|---|---|
| 立项书 | `05x.md`，与包内 `research/05x_original.md` **逐字节相同** | 369 行 |
| Python 研究层 | `allocator.py`（10 种策略）、`threats.py`（合成对手 + 检测表）、`study.py`（评测）、`model.py`、`audit.py` | 约 320 行 |
| Go 执行层 | `gateway/guard/guard.go`（参考监视器 + 票据）、`gateway/main.go` + 测试 | 约 343 行 |
| 测试 | `test_allocator / test_audit / test_study / test_threats` | 约 117 行 |
| 结果 | `results/pilot.json`（200 次运行的逐事件审计）、`results/smoke.json` | **125,437 + 3,753 行** |
| 文档 | `README / ARCHITECTURE / EXPERIMENT / QUALITY / NEXT_STEPS / TROUBLESHOOTING / GLOSSARY / CODING_STANDARDS` + **35 个生成页** | 45 个 md |

**实验规模**：4 个场景 × 10 个策略 × 5 个种子 × 100 条请求 = 20,000 个事件。

---

## 2. 优点：它真的做对了什么

### 2.1 "硬授权"与"学习分配"在架构上被彻底切开

这是本项目的核心贡献，也是它做得最好的一处。三处代码互相印证：

```python
# model.py:27-30  —— 硬门是一个纯函数，没有任何可学习参数
def hard_permit(view):
    return (view.principal == "alice" and view.resource == "sandbox/tickets"
            and view.source_id == "user-approved")
```

```python
# allocator.py:1  —— 模块级 docstring 就把边界写死了
"""Budget-aware online extra-check selection. It never grants tool permissions."""
```

```python
# study.py:31-37  —— 硬门在分配器之前，且不经过分配器
if not hard_permit(view):
    forbidden += 1
    rows.append({"decision": "hard_deny", "check": "none", "cost": 0})
    continue            # ← 学习器根本没机会介入
```

**这个顺序是对的，而且是被代码强制的**：任何策略（包括 `no_extra`、`random`、`game`）都不可能绕过 `hard_permit`。对比立项书 §2 的"层级 I（必要）：确定性 allow/deny 核心 + 博弈分配额外检查资源"——实现与设计一致。

### 2.2 真值被挡在类型之外

`model.py:20-24`：

```python
@dataclass(frozen=True)
class Attempt:
    """Ground truth is deliberately not a member of Observation."""
    view: Observation
    attack_kind: str   # Synthetic oracle only. Must not reach the allocator.
```

`Allocator.choose(view: Observation)` 的签名里只有 `Observation`，`attack_kind` **在类型上就到不了分配器**。这比"我们承诺不偷看"要硬得多——改起来必须动类型签名，review 时一眼可见。

### 2.3 反事实随机流是"计数器式"的，保证配对可比

`threats.py:10-13`：

```python
def coin(seed, event, salt):
    digest = hashlib.sha256(f"{seed}/{event}/{salt}".encode()).digest()
    return int.from_bytes(digest[:8], "big") / 2**64
```

所有随机性（攻击是否发生、信号值、是否缺失、是否越权、检测器是否命中）都由 `(seed, event_id, salt)` 唯一确定。好处：

- **同一事件在不同策略下拿到相同的"攻击是否发生"与"信号值"**，配对比较成立；
- 不受策略调用顺序影响（不像 `random.Random()` 那样依赖消耗序列）；
- `multi-seed` 复现是精确的，不是"统计上差不多"。

### 2.4 攻击者的可观测性被显式限制

`threats.py:16-17`：

```python
"""The reactive attacker only sees a public rejection count, never policy internals."""
active = "metadata" if prior_rejections < (index // 8) else "memory"
```

- 攻击者**只能看到公开的拒绝计数**，看不到策略类型、检查深度、预算余量；
- 这条限制与立项书 §1.1"攻击者能力：可观察部分防御结果"是对齐的；
- 而且 `prior_rejections` 是**该策略自己造成的**（`study.py:33,45,57` 三处递增），所以"不同策略下攻击序列不完全相同"这件事被 `EXPERIMENT.md:13` 主动声明了（"反映型对手会受到策略反馈影响，因此不同策略的攻击序列不严格相同"）。

**它知道自己引入了什么偏差，并写出来了。**

### 2.5 预算记账不会静默超支

`allocator.py:77-83`：

```python
def charge(self, check):
    cost = COST[check]
    if cost > self.remaining:
        raise ValueError("budget exceeded")
    self.remaining -= cost
```

而 `study.py:68` 用 `spent = budget - allocator.remaining` 反推消耗。**超支会抛异常而不是被截断**，所以 `spent ≤ budget` 是结构保证。这一点比"我们记录了花费"要可靠。

### 2.6 审计是可独立验证的哈希链，而且明确了它不保证什么

`study.py:120-128` 支持 `--audit-db` 写出 SQLite 审计并在结束时 `audit.verify()`，失败即抛错。

同时 `EXPERIMENT.md:9` 主动限权：

> 哈希链支持无密钥的**意外修改检测**，**不等同抗恶意管理员篡改**。

**这句话很重要**——很多项目会把哈希链说成"防篡改证明"。这里没有。

### 2.7 文档诚实度

- `QUALITY.md:7`："研究层为 V0/M1 **启发式**：额外检查的规则/窗口 bandit 和简单漂移触发器，**不是完整 Stackelberg/Nash 求解器，没有动态 regret 证明，也没有验证风险估计的校准性**"；
- `QUALITY.md:9`："任何没有超过基线的结果保留，**不删去负例**"；
- `EXPERIMENT.md:11`：把六个指标的口径逐个说明，并写"所有延迟和成本都是**虚拟单位**，不能宣称实测 P95/CPU/Token 成本"。

---

## 3. 缺点：问题在哪

### 3.1 最致命：主方法只在 1/4 的场景里赢，且在关键场景被 1 行基线压制

`results/pilot.json` 的完整汇总（VASR 越低越好，benign 越高越好，spent 上限 145）：

| 场景 | 策略 | VASR | 良性成功率 | 花费 |
|---|---|---:|---:|---:|
| static | bandit | **0.4303** | 0.9217 | 144.4 |
| static | **game** | 0.4351 | 0.9289 | 144.2 |
| static | fixed_deep | 0.3713 | **0.9554** | 144 |
| static | threshold | 0.5217 | **0.9747** | 82.8 |
| switch | bandit | 0.3505 | 0.9179 | 144.2 |
| switch | **game** | **0.3414** | 0.9289 | 144 |
| switch | fixed_deep | 0.3444 | 0.9554 | 144 |
| cycle | bandit | **0.4597** | 0.9174 | 144.4 |
| cycle | **game** | 0.4911 | 0.9258 | 142.6 |
| cycle | fixed_deep | **0.3892** | 0.9554 | 144 |
| reactive | bandit | **0.5150** | 0.9154 | 144.4 |
| reactive | **game** | 0.5536 | 0.9222 | 143.6 |
| reactive | fixed_deep | **0.4326** | **0.9554** | 144 |
| reactive | threshold | 0.6858 | **0.9747** | 70.6 |

四个场景横向比：

| 对比 | static | switch | cycle | reactive | 战绩 |
|---|---:|---:|---:|---:|:---:|
| `game` vs `bandit`（VASR 低者胜） | 0.4351 vs **0.4303** | **0.3414** vs 0.3505 | 0.4911 vs **0.4597** | 0.5536 vs **0.5150** | **1 胜 3 负** |

**最需要单独拿出来说的是 reactive 场景**（这是"适应性攻击"最核心的场景）：

```text
fixed_deep: VASR 0.4326, benign 0.9554, spent 144
game:       VASR 0.5536, benign 0.9222, spent 143.6
```

**`fixed_deep` = "每条请求都用最深的检查"，一共 3 行代码。它在安全性和可用性上同时优于 `game`，花费相当。**

一个"预算约束的自适应博弈分配器"，在它最该发挥作用的场景里，输给了"永远做最贵的检查"。这不是调参问题——`spent` 相同说明**两者花了同样的钱**，差别在于 `game` 把钱花在了不该省的请求上。

而且 `threshold`（`risk > .75 → deep; > .45 → light; else skip`，3 行）在四个场景里**良性成功率全部最高（0.9747）**，花费还只有 71–83。也就是说：**在"可用性"这一维上，两条 if 的规则从头赢到尾。**

### 3.2 两个"增强模块"的贡献是可以直接读出来的：一个无效，一个有害

立项书把创新点分成 IV 级，其中 III（防御可预测性控制）与 IV（漂移与回退）各对应一个消融策略：

| 场景 | `game` | `game_no_drift` | 漂移模块效果 |
|---|---:|---:|---|
| static | 0.4351 | 0.4351 | **完全相同** |
| switch | 0.3414 | 0.3414 | **完全相同** |
| cycle | 0.4911 | 0.4911 | **完全相同** |
| reactive | 0.5536 | **0.5488** | **变差 0.005** |

| 场景 | `game` | `game_no_mix` | 随机化模块效果 |
|---|---:|---:|---|
| static | 0.4351 | **0.4303** | 变差 0.005 |
| switch | **0.3414** | 0.3505 | **变好 0.009** |
| cycle | 0.4911 | **0.4597** | 变差 0.031 |
| reactive | 0.5536 | **0.5245** | 变差 0.029 |

**结论**：

> **漂移检测模块在 3 个场景里数值完全不变（说明它一次都没触发过有效的策略切换），在第 4 个场景里带来轻微退化。**
> **随机化模块 1 正 3 负。**

这两个"增强"正是立项书 §2 表格里的创新点 III 与 IV。现在的数据表明：**把它们删掉，方法会略好或不变。**

原因可以在代码里看到（`allocator.py:35-43`）：

```python
def _drift(self):
    if len(self.history) < 20: return False
    old = list(self.history)[:20]
    new = list(self.history)[-20:]
    return new_rate - old_rate > 0.22
```

`history` 是 `deque(maxlen=40)`，所以 `[:20]` 与 `[-20:]` 正好是**同一个缓冲区的前后两半**——这是一个"滑动窗口对比"，而不是"变化点检测"。而且触发后只做一件事：`if risk > .5: name = "deep"`（`allocator.py:65-66`），而 bandit 在 `risk > 0.83` 时本来就会升到 deep。**触发条件与已有行为高度重叠，所以经常"触发了也什么都不改变"。**

### 3.3 "零违规"是代码结构决定的，不是测出来的

`study.py:31-35`：

```python
if not hard_permit(view):
    forbidden += 1
    ...
    continue          # ← 后面所有代码都拿不到这个请求
```

`study.py:59-60`：

```python
else:
    if not hard_permit(view):
        violations += 1
```

**这两行是死代码**——能走到 `:58` 的请求，在 `:31` 时 `hard_permit` 必然为真，而 `view` 是不可变的 frozen dataclass，所以 `:59` 永远为假。

于是 `results/pilot.json` 里 40 行汇总的 `hard_policy_violations` **全部为 0**，而这个 0 不携带任何信息。

这与立项书 H2（"若所有敏感工具调用经不可绕过的参考监视器…则可以证明未授权工具调用不执行"）的关系是：**H2 在这个实现里是一个结构性性质，不需要实验验证，也不该被列为实验结果。** 现在的写法会让读者误以为"我们用一个 20,000 事件的实验证明了零违规"。

**修法**（二选一）：

- **诚实版**：把 `hard_policy_violations` 从指标表里删掉，改为在 Architecture 一节点明"由构造保证"；
- **有信息版**：加一个**故意关掉硬门**的隔离消融（`policy == "no_hard_gate"`），跑出非零违规，用来量化"如果允许学习器决定授权会发生什么"。这才是审稿人想看的东西。

### 3.4 预算从未真正约束决策：所有自适应策略都花到 144/145

| 策略 | static | switch | cycle | reactive |
|---|---:|---:|---:|---:|
| `no_extra` | 16 | 16 | 16 | 16 |
| `fixed_light` | 95.6 | 95.6 | 95.6 | 95.6 |
| `threshold` | 82.8 | 79.2 | 76.0 | 70.6 |
| `fixed_deep` / `random` / `bandit` / `game` | **144–144.4** | 144.2 | 142.6–144.4 | 143.6–144.4 |

`game`、`bandit`、`random`、`fixed_deep` **全部顶到预算上限**。这说明：

- "预算约束"在这些策略里**没有产生任何取舍**——它们就是"花到没钱为止"；
- 唯一"省着花"的 `threshold` 反而拿到了最好的可用性；
- 而立项书 §8.1 明确要求 **"图 3：预算—vASR—合法任务成功率 Pareto 面"** —— 这张图**完全不存在**，因为只跑了一个 `budget=145`。

**一个"预算约束优化"的论文，如果不扫预算，核心卖点就没有证据。** 这是最该补的实验，而且成本极低（`--budget 40/80/145/290` 跑四遍）。

### 3.5 检测器是查表，攻击者是 1 行规则——博弈模型名不副实

`threats.py:53-57`：

```python
DETECTION = {
    "light": {"benign": .06, "indirect": .65, "metadata": .30, "memory": .18, "combo": .40},
    "flow":  {"benign": .09, "indirect": .43, "metadata": .80, "memory": .50, "combo": .67},
    "deep":  {"benign": .03, "indirect": .91, "metadata": .88, "memory": .84, "combo": .92},
}
```

- **检测概率只依赖 (检查深度, 攻击类型)**，与请求内容、上下文、历史完全无关；
- 于是"深度检查"的期望收益是**固定常数**，最优策略必然会收敛到"永远 deep"（这正是 `fixed_deep` 赢的原因）；
- 而"攻击者的策略"只有 `threats.py:24-30` 的那几行：`switch` 是 60 步换一次，`cycle` 是 30 步轮换，`reactive` 是按拒绝计数在 `metadata`/`memory` 之间二选一。

**这没有构成"博弈"**：防守方的动作空间只有 4 档深度，收益是先验固定的表；进攻方只有 4 个类型且切换规则写死。立项书 §3.2 要求的"部分可观测 Stackelberg / 贝叶斯攻防博弈、估计攻击者类型"完全没有实现（`QUALITY.md:7` 也承认了）。

而 `_pick_bandit` 的奖励设计（`allocator.py:88-90`）：

```python
proxy_reward = (3.5 if flagged else 0.0) - COST[check] * .25
```

`deep` 命中一次得 3.5、成本罚 1.0；`light` 命中得 3.5、成本罚 0.25。**两者命中奖励相同**，于是 bandit 很快学会"总是 deep"——这就解释了为什么 `bandit` ≈ `game` ≈ `fixed_deep` 的花费（都是 144）。

### 3.6 风险信号带有系统性偏移，等于给了基线一个便宜的优势

`threats.py:32-36`：

```python
signal = 0.10 + 0.55 * coin(seed, str(index), "signal")     # 良性: [0.10, 0.65]
if kind != "benign":
    signal += 0.18 if kind in ("indirect","memory") else 0.27  # 攻击: +0.18 / +0.27
signal = min(signal, 1.0)
```

- 良性：`[0.10, 0.65]`
- 攻击：`[0.28, 1.00]`（indirect/memory）或 `[0.37, 1.00]`

**区间有重叠（比 04X 的干净间隙好得多），但存在系统性正向偏移。** 后果是 `threshold` 那两条 if（`.75 → deep`、`.45 → light`）已经能拿到最优的可用性——因为"风险信号高 → 多检查"这个策略在本数据里几乎就是最优解。

**改进方向**：让 `signal` 与 `kind` **条件独立**（即风险信号不携带攻击类别信息），攻击只能靠"检查深度 × 具体行为特征"识别；或者反过来，让良性请求里也有高风险的合法操作（立项书 §7.2 的"正常请求反例"要求）。

最后这一点立项书自己写了：

> **正常请求反例**：有很多和恶意内容相似但合法的任务，防止通过全拒绝虚假提高安全分。

代码里的 `benign` 就 60% 的随机请求，**没有任何"高风险但合法"的构造**。这是 `false_block_rate` 只有 0.02–0.08 的原因，也是所有策略都能有 0.92+ 良性成功率的原因。

### 3.7 其余

| # | 问题 | 证据 |
|---|---|---|
| 1 | **没有任何置信区间或显著性检验**：5 个种子只报了均值 | `study.py:96-100` |
| 2 | `game` 的 `risk > .83 → deep` 与 bandit 的 prior 高度重叠，两策略实质上很接近 | `allocator.py:70-71` |
| 3 | `latency` 是 `5 + LATENCY[check] + queue*4` 的**合成加法**，不是排队模型 | `study.py:65` |
| 4 | `missing`（4%）让 `risk_signal=None` → 直接返回 `deep` 或 `stop`，这 4% 的决策既不经过策略也不经过硬门之外的分支 | `allocator.py:49-50` |
| 5 | Python 的 `hard_permit` 与 Go 的 `guard.go` 是**两套实现**，没有交叉验证 | `model.py:27` vs `gateway/guard/guard.go` |
| 6 | `results/pilot.json` **2.8 MB / 125,437 行**，`smoke.json` 86 KB，都进了仓库 | — |
| 7 | 35 个生成页 + 38 KB `research/05x_original.md` 副本 | — |

---

## 4. 项目自己已经承认的负结果与红线

1. **"研究层为 V0/M1 启发式……不是完整 Stackelberg/Nash 求解器，没有动态 regret 证明，也没有验证风险估计的校准性"**（`QUALITY.md:7`）。
2. **"任何没有超过基线的结果保留，不删去负例"**（`QUALITY.md:9`）。
3. **"所有延迟和成本都是虚拟单位，不能宣称实测 P95/CPU/Token 成本"**（`EXPERIMENT.md:11`）。
4. **"检测概率是合成设定，并非训练出的模型；置信度未经校准"**（`EXPERIMENT.md:13`）。
5. **"反应型对手会受到策略反馈影响，因此不同策略的攻击序列不严格相同"**（`EXPERIMENT.md:13`）。
6. **"论文实验仍缺公开真实安全基准、真实 MCP/Agent 调用、强基线和统计显著性、真实运营负载、对手收益建模、漂移后收敛分析"**（`EXPERIMENT.md:15`）。

**第 6 条里的"强基线"与"统计显著性"，正好是 §3.1 缺的东西。** 它自己知道，只是还没做。

---

## 5. 改进清单

### P0 — 不做这些，"博弈 + 预算"两个词都立不住

| # | 要做什么 | 为什么 | 验收标准 |
|---|---|---|---|
| P0-1 | **预算扫描**：`--budget 40/80/145/290` × 4 场景 × 10 策略 → 画"预算 × VASR × 良性成功率"的**三维 Pareto 面** | 现在只跑 145 一档，且自适应策略全部花到 144，"预算约束"无证据（§3.4） | 出现不同策略在不同预算档位交叉的曲线；至少一档里 `game` 在**同等花费**下优于 `fixed_deep` |
| P0-2 | **让攻击者真的适应**：攻击类型由"观测到的检查深度分布"决定（而不是只看拒绝计数），并给攻击者一个**收益函数**（成功收益 − 尝试成本） | 现在是 1 行规则 + 固定切换（§3.5），无法检验 H1/H3 | 在 reactive 场景里，`fixed_deep` 的 VASR **明显上升**（因为攻击者会针对它），而 `game` 保持稳定——这才叫自适应 |
| P0-3 | **修 `hard_policy_violations`**：删掉这个恒 0 指标，或加一个"硬门关闭"的隔离消融 | `study.py:59-60` 是死代码（§3.3） | 要么指标从表里消失并改在架构章节说明"由构造保证"；要么出现一个非零值并明确标注"危险消融" |
| P0-4 | **让漂移模块真的做事**：换成长度可调的**变化点检测**（如 CUSUM），并且触发后的动作必须是 bandit 不会自发作出的 | 现在 3 个场景数值完全不变（§3.2） | `game` 与 `game_no_drift` 至少 2 个场景出现可测差异，且方向为正；否则删掉这个模块 |

### P1 — 决定能不能谈投稿

| # | 要做什么 | 验收标准 |
|---|---|---|
| P1-1 | **统计**：每个场景每个策略给 bootstrap 95% CI，并对 `game vs bandit`、`game vs fixed_deep` 做成对检验 | 现在只有 5 个种子的均值，没有任何区间（§3.7） |
| P1-2 | **加"高风险但合法"的请求族**：模拟真实里大量"看起来可疑但确实是用户授权的操作" | `false_block_rate` 不再只有 0.02–0.08；`threshold` 的优势被削弱，`game` 才有发挥空间 |
| P1-3 | **让风险信号与攻击类别条件独立**，或至少降低偏移量（0.18/0.27 → 0.05/0.10） | `threshold` 不再自动最优；`bandit` 与 `game` 出现真实分化 |
| P1-4 | 统一 Python `hard_permit` 与 Go `guard.go`：要么 Go 作为唯一实现，要么加"两实现一致性"测试 | 两条路径对同一请求给出相同判定 |
| P1-5 | 修正 bandit 的奖励：命中奖励应与**节省的花费**挂钩（例如 `3.5 - COST`），否则必然收敛到 deep | `bandit` 的花费不再顶到 144；出现"用 light 抓到部分攻击"的分层行为 |

### P2 — 仓库与可读性

| # | 要做什么 |
|---|---|
| P2-1 | `pilot.json` 2.8 MB 不该进仓库：拆成 `summary.json`（40 行）+ `pilot.sample.jsonl`（每策略 5 条）+ 生成脚本；或用 gzip |
| P2-2 | 删 35 个生成页；`research/05x_original.md` 改为指针 |
| P2-3 | `latency` 明确标注为"合成加法模型"，或在 README 表格标题写"（虚拟单位，非排队模型）" |
| P2-4 | Windows 真机跑一遍（Go 侧最可能在 Windows 编译/路径上翻车） |

---

## 6. 只改五件事的话

1. **预算扫描 + Pareto 面** → 让"预算约束"这个标题第一次有意义。
2. **让攻击者学会针对策略** → 否则 `fixed_deep` 永远赢，自适应没有存在理由。
3. **删掉/替换恒 0 的违规指标** → 别用结构性事实冒充实验结论。
4. **加"高风险但合法"请求** → 给可用性一个真实的压力，否则 `threshold` 白拿冠军。
5. **修漂移模块或删掉它** → 三个场景数值完全一样，本身就是结论：它没用。

---

## 7. 在全系列里的定位建议

| 关系 | 说明 |
|---|---|
| 与 01X（可靠性） | 01X 是"风险预算 + 硬不变量"，本文是"资源预算 + 硬授权"——**两者是同构的**：硬边界永不让步，软优化在边界内做。两篇的架构章节可以共用一套表述 |
| 与 02X（科学仿真） | 02X 的 `ABSTAIN_UNVERIFIABLE` 与本文的 `stop / review` 是同一种"拒绝执行"语义 |
| 与 03X（电力） | 03X 的"最坏情况硬约束导致 94% 拒单"与本文的"`game` 输给 `fixed_deep`"是同一类问题：**硬约束/固定高检查在小动作空间里往往就是最优** |
| 与 04X（因果） | 04X 的"选择性反馈偏差"正是本文 §3.5 的 `proxy_reward` 问题：被拦下的攻击没有真实后果标签 |

**推荐定位**：**五份里最应该"缩题重做实验"的一份。** 理由：

- 它的**架构分解是全系列最正确的**（硬门 / 学习分配 / 可审计），这部分可以直接写进论文的 Design 章节；
- 但它现在的实验**无法支撑任何一条创新声明**：主方法 1 胜 3 负、两个增强模块无效或有害、零违规是构造保证、预算从未绑定；
- 好消息是：**它的问题几乎全是"实验设计"而不是"实现缺陷"**。P0-1 到 P0-4 加起来大约 2–3 天工作量，做完之后结论可能完全反转（尤其是 P0-2 与 P1-2）。

**建议的推进顺序**：

1. **先做 P0-1**（预算扫描）：半天，最便宜，而且能立刻回答"我们的方法到底在哪个预算区间有用"。
2. **再做 P1-2 + P1-3**（高风险合法请求 + 削弱信号偏移）：这两项会改变 `threshold` 与 `fixed_deep` 的相对优势，是新方法唯一的翻盘机会。
3. **最后做 P0-2**（真自适应攻击者）：这是把"博弈"两个字坐实的关键，也是最花时间的一项。
4. 如果做完 1–3 之后 `game` 仍然稳定输给 `fixed_deep`，**诚实结论是很有价值的**：可以写成"在固定检测成本表下，自适应资源分配不优于固定最深检查；自适应的价值出现在 X 条件下"。这比强行包装成 SOTA 更可能被 C&S 或 JISA 接受。

---

## 附录：文件地图

```text
05x.md                        ← 立项书（369 行）：§2 四级创新点、§3 博弈模型、§13 十问自查
05xcode/
├─ gameshield/model.py        ← 30 行，**硬门 hard_permit + Observation/Attempt 的分离**
├─ gameshield/allocator.py    ← **90 行，10 种策略；:35-43 是无效的漂移检测**
├─ gameshield/threats.py      ← **63 行，coin 计数器随机 + DETECTION 查表 + 1 行攻击者**
├─ gameshield/study.py        ← 135 行，评测主循环；**:59-60 是死代码**
├─ gateway/guard/guard.go     ← 109 行，Go 侧参考监视器（与 Python 是两套实现）
├─ docs/EXPERIMENT.md         ← **15 行，六条口径 + 六条缺口（务必先读）**
├─ docs/QUALITY.md            ← 11 行，"启发式、非 Stackelberg、无 regret 证明"
└─ results/pilot.json         ← **2.8 MB / 125,437 行（只看 summary 段就够）**
```

**阅读顺序建议**：`results/pilot.json` 的 `summary`（先看 40 行结论有多难看）→ `docs/EXPERIMENT.md`（看口径）→ `allocator.py:35-75`（看漂移与随机化为什么无效）→ `study.py:31-67`（看硬门与死代码）→ `model.py`（看类型分离）。
