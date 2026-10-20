# 02dst.md — 第二篇（02X）评估：TwinProof-Gate 科学仿真与数字孪生的证据校准

> **评估对象**：`C:\Users\20050224\Desktop\02X\`（立项书 `02x.md` 44,642 B + 代码包 `02xcode.zip`）
> **评估方式**：**只读**。已读完立项书、Go 求解器与测试、Python 全部 7 个模块、`results/pilot.json`（48 例 × 4 策略的逐条记录）。
> **未执行**：没有编译 Go、没有跑 `cli.py evaluate`。数字来自包内 `results/pilot.json` 与 `docs/QUALITY.md`。

---

## 0. 一页速览

| 维度 | 评分（1–5） | 一句话依据 |
|---|:---:|---|
| 选题价值与撞题风险 | **3.5** | "仿真跑通 ≠ 物理可信"是真命题，也是全系列最有领域壁垒的一个；但 MCP-SIM/ALL-FEM/GENIUS/SimBench 已把 2026 的位置占满 |
| 立项文档质量 | **5.0** | **五份里最严谨**：三层概念区分、10 类故障注入、8 组基线、明确的"不能声称"清单、"把仿真叫 DT 会直接拒稿" |
| 原型实现完成度 | **4.0** | 真的写了一个独立的 Go 传热求解器 + 制造解（MMS）+ 独立离散残差；孪生更新门控也实现了 |
| 结果可信度 | **1.5** | **评价 oracle 就是验证器自己的判据**，于是 FAR=0 与"误拒率=0"都是定义出来的（见 §3.1） |
| 工程与仓库质量 | **4.0** | 单测 24 项、SQLite 连接回收写对、`PACKAGE_SHA256.json`、错误注入齐全；扣分在 35 个生成页 |
| **总分** | **18 / 25** | 五份里排第 2 |

**一句话结论**：**这是五份里"文档最像要发 RESS/AEI、代码最能站住，但实验最不能被引用"的一份。** 它把最难的一件事——**用与 Agent 无关的物理证据去否证一次仿真**——真的实现出来了；然后在最后一步把自己毁了：判定"真值"的那个函数，和验证器做检查的那个函数，**是同一个函数、同一个容差**。

**如果只能改三件事**：

1. **把 oracle 换成独立的、更高分辨率的参考解**（`n=257` 或解析级数解），并且用**与验证器不同的容差**。现在 `reference_error(case,run) <= 0.17` 两边共用（`evaluation.py:45` vs `evidence.py:105`）。
2. **出一张"故障 × 检查层"的边际检出矩阵**。数据里 `records[].fault` 已经有了，但从没算过——我替你算完了，见 §3.3，结论是**最贵的 12 单位网格细化层检出 0 例**。
3. **让 `reference_error` 的容差随网格尺度变化**。现在 n=9/17/33 共用一个 0.17 K，导致 20 个"无故障"案例里有 3 个被判无效——那是离散误差，不是故障。

---

## 1. 项目构成

| 层 | 内容 | 规模 |
|---|---|---|
| 立项书 | `02x.md`，与包内 `research/02x_original.md` **逐字节相同** | 565 行 |
| Go 求解器 | `solver/main.go`（二维稳态传热，有限差分 + 迭代）+ 测试 | 约 202 行 |
| Python 研究层 | `model.py`、`evidence.py`（独立检查）、`solver.py`（调用 Go）、`evaluation.py`（评测）、`twin.py`（孪生门控）、`server.py` | 约 404 行 |
| 测试 | `test_physics.py`（89 行）、`test_twin.py`、`test_evaluation.py`、`test_http.py` | 约 256 行 |
| 结果 | `results/pilot.json`（48 例 × 4 策略逐条记录 + 汇总 + Wilson 区间） | 1,807 行 |
| 文档 | `README / ARCHITECTURE / EXPERIMENTS / QUALITY / NEXT_STEPS / THREAT_MODEL / GLOSSARY / CODING_STANDARDS` + `docs/adr/` + **35 个生成页** | 46 个 md |

物理问题很小但完整：**单位正方形上的稳态热传导**，左/右边界定温 300K/320K，内部有正弦源项，存在解析（制造）解 `T = 300 + 20x + A·sin(πx)·sin(πy)`。所以"真值"是可以精确写出来的——这是这个选题能成立的关键前提。

---

## 2. 优点：它真的做对了什么

### 2.1 它真的写了一个独立求解器，而不是调库然后把结果当真值

`evidence.py:11-15` 的 `grid_reference()`：

```python
def grid_reference(case, n):
    """Independent manufactured exact solution (NOT returned by the Go solver)."""
    return [case.left_k + (case.right_k-case.left_k)*x/(n-1)
            + case.amplitude_k*math.sin(math.pi*x/(n-1))*math.sin(math.pi*y/(n-1))
            for y in range(n) for x in range(n)]
```

这句注释是全篇的支点：**参考解来自数学，不来自求解器**。配合 Go 端真正实现有限差分求解，本项工作具备"用一份独立计算否证另一份计算"的最小闭环。这一点很多同类原型是没有的（它们用"LLM 觉得对不对"当证据）。

### 2.2 四类物理检查都是"独立可重算"的

`evidence.py` 里四层检查各自独立实现，不共用求解器的中间量：

| 检查 | 成本 | 做什么 | 能手算吗 |
|---|---:|---|---|
| `readback` | 1 | 求解器回读的 8 个字段与 case 一致 | 能 |
| `solver_status` | 1 | 求解器真报 `converged` | 能 |
| `boundary` | 1 | 边界 4 条边上温度与解析边界值误差 ≤1e-7 | 能 |
| `pde_residual` | 2 | **独立重算**离散 `-kΔu - q` 的残差 | 能 |
| `mms_reference` | 3 | 与制造解的最大偏差 ≤0.17 K | 能 |
| `three_mesh_convergence` | 12 | n=9/17/33 三次独立求解，误差比 >3 | 能 |

**残差是独立重算的，不是从求解器里拿的**（`residual_norm` 自己算五点差分 + 自己算源项）。这是"独立证据"三个字唯一站得住的实现方式。

### 2.3 三道"防作弊"的隔离都是真的

1. **故障元数据不进验证器**：`evaluation.py:19-20` 注释写明 "Ground-truth injection metadata is never passed to the Verifier"，`cases()` 生成时只把 `fault` 交给 `runner.run()`，不给 `Verifier`。
2. **故障不进求解器输入**：`model.py:34-38` 的 `solver_input()` 注释 "Observations and fault truth never become verifier evidence"。
3. **风险分层不来自 Agent 自报**：`evidence.py:110-112`

```python
elevated = case.n <= 9 or abs(case.amplitude_k) >= 20 or abs(case.right_k-case.left_k) >= 80
```

这句话是刻意写的——分层的依据是**任务的物理类型参数**，不是 Agent 说"我这个很危险"。对照第一、五篇里"让 LLM 给自己的风险打分"，这里的方向是对的（虽然规则本身很粗糙，见 §3.5）。

### 2.4 孪生更新门控是全系列最完整的一段"拒绝逻辑"

`twin.py:47-72` 的 `update()` 有六道门，每道门对应一种具体的不可信：

| 门 | 拒因 | 条件 |
|---|---|---|
| 证据 | `FROZEN_UNVERIFIABLE` | 决策不是 `PASS_WITH_EVIDENCE` / 回读不一致 / 报告摘要与模型不一致 |
| 来源 | `FROZEN_BAD_SOURCE` | 观测不是 `synthetic`、传感器不是 `center`、值非有限 |
| 时效 | `FROZEN_STALE_OBSERVATION` | 观测时间在未来或超过 120 s |
| 偏差 | `FROZEN_DRIFT` | 中心点温度与观测差 > 0.6 K |
| 版本 | `FROZEN_VERSION_CONFLICT` | `BEGIN IMMEDIATE` 下发现版本或时间戳不前进 |

**"版本冲突"那一支用了 `BEGIN IMMEDIATE` + 版本比较**（`:64-67`），也就是"不允许用一份旧模型的结论覆盖新状态"。这正是立项书 §3.4 说的"不满足条件时冻结孪生校准结果"。而且它明确写了 `SYNTHETIC_UPDATE_ACCEPTED`，不冒充真实孪生。

### 2.5 一个真实的工程坑被记下来了

`twin.py:29-35`：

```python
# sqlite3 connection context managers commit but do not close: close explicitly.
db = sqlite3.connect(self.path, timeout=10)
try:
    with db: yield db
finally:
    db.close()
```

这个坑（`with sqlite3.connect(...)` 只 commit 不 close，长跑会泄漏句柄）在 Python 项目里极常见，作者不但绕开了，还在注释里写清楚了为什么。

### 2.6 报告口径是对的

`evaluation.py:27-36` 的 Wilson 区间实现正确，并且对 0 事件也给出上界 `[0, 0.110]`——**这正是"零次错误 ≠ 零风险"的正确表达**。`QUALITY.md:20` 也主动把这句话写出来了。

---

## 3. 缺点：问题在哪

### 3.1 最致命：评价 oracle 与验证器用的是**同一个判据**

这是整份工作最关键的一处。对照两段代码：

```python
# evaluation.py:43-46  —— 实验的"真值"
reference_good = reference_error(case, run) <= 0.17
truth_valid = check_readback(case, run) and reference_good and run["solver_status"] == "converged"
```

```python
# evidence.py:88-105  —— 验证器做的事
if not check("readback",      1, lambda: check_readback(case,run)):            return report
if not check("solver_status", 1, lambda: run.get("solver_status")=="converged"): return report
if not check("boundary",      1, lambda: boundary_error(run) <= 1e-7):          return report
if not check("pde_residual",  2, lambda: residual_norm(run) <= 1e-4):           return report
if not check("mms_reference", 3, lambda: reference_error(case,run) <= self.tolerance_k): return report
```

`self.tolerance_k` 的默认值是 `0.17`（`evidence.py:60`）。

**于是 `allowed == True` 必然推出 `truth_valid == True`。** 因为验证器通过的路径上，`readback`、`converged`、`reference_error <= 0.17` 三项都已成立——它们正好是 `truth_valid` 的三个合取项。

后果逐条：

| 报告里的指标 | 数值 | 实际含义 |
|---|---|---|
| `fixed` 的 FAR | **0.00%** | 定义上不可能非零。不是"检查有效"，是"真值=检查" |
| `adaptive` 的 FAR | **0.00%** | 同上 |
| `unnecessary_rejection_rate` | **0.00%**（四策略全部） | 同样定义上为 0 |
| `valid_completion_rate` | **35.42%**（四策略**完全相同**） | 因为四者放行的恰好是同一批 17 个案例 |

**唯一能出现 FAR > 0 的，只有那两个会"跳过检查"的策略：`exit_only`（87.1%）与 `random`（6.45%）。** 也就是说：这张 FAR 表实际上只测量了"跳过检查的代价"，没有测量"检查之间谁更强"。

这不算造假——作者没有偷偷读取真值，`records` 里也都是逐条可核对的。但它意味着：**论文最核心的指标（FAR）在当前实验设计下是一个恒等式，不能用来支撑任何方法学结论。**

### 3.2 第二个后果：最贵的一层验证，检出贡献是 0

因为 `fixed` 与 `adaptive` 在**所有**准确率指标上完全相同（FAR 0.00、有效完成率 35.42、误拒率 0.00、弃权率 0.00），差别只有成本：

| 策略 | 平均验证单位 | p95 实测耗时 |
|---|---:|---:|
| `fixed`（每例都跑三层网格） | **8.917** | 27.23 ms |
| `adaptive`（只在高风险例跑） | **7.167** | 25.54 ms |

省了 **19.6%** 的抽象成本，但**一个判定都没变**。这说明 `three_mesh_convergence`（成本 12，占 `fixed` 总成本的 62%）**在这 48 个案例里从未拒绝过任何一例**。

为什么？因为 `errors[0] > errors[1] > errors[2]` 且比值 >3，在"解是对的"时候必然成立（h 减半、误差降约 4 倍），在"解是错的"时候也往往成立（错误同比例缩小）。**这个检查测的是"求解器是否在收敛"，而不是"解是否物理正确"** ——它抓不住 `fake_converged`（残差被容差掩盖）和 `tamper_center`（只改中心点）。

### 3.3 我替它算了缺失的那张表：故障 × 检查层

数据里 `records[].fault` 与 `verify_cost_units` 都有，但项目从没做过这个分析。按成本推断每类故障"终止在哪一层"（成本 = 该层累计值）：

| 故障类型 | 例数 | 终止成本 | 实际拦住它的层 | 这层是物理检查吗 |
|---|---:|---:|---|---|
| `missing_source` | 4 | 1 | `readback` | ❌ 纯元数据比对 |
| `reverse_source` | 4 | 1 | `readback` | ❌ |
| `swap_boundaries` | 4 | 1 | `readback` | ❌ |
| `wrong_conductivity` | 4 | 1 | `readback` | ❌ |
| `short_iteration` | 4 | 2 | `solver_status` | ❌ 读退出码 |
| `fake_converged` | 4 | 5 | `pde_residual` | ✅ |
| `tamper_center` | 4 | 5 | `pde_residual` | ✅ |
| （无故障） | 20 | 18.2 | 全部通过 | — |

**读这张表的方式**：7 类故障里 **5 类（20 个案例）被 1–2 个单位的元数据/状态码检查拦住**，只有 2 类（8 个案例）真正需要物理残差；而 12 单位的网格细化层**一例都没抓到**。

这意味着立项书 §0 主张的第 1 条（"独立证据契约，把执行成功和物理可信区分开"）是**成立的**——`converged` 确实骗过了 27/31 个无效案例——但主张的第 2 条（"误差-不确定性预算驱动的动态验证"）**在本数据上只体现为省钱，不体现为更准**。

### 3.4 容差不随网格变化，把离散误差算成了"无效"

`reference_error(case, run) <= 0.17` 对 n=9、17、33 **用同一个 0.17 K**。但离散误差按 h² 下降，n=9 时远大于 n=33。数据里 20 个"无故障"案例中：

- **17 个被判有效**，**3 个被判无效**——而那 3 个恰好是 `n=9` 的粗网格案例，偏差来自**离散误差**，不是任何注入故障。

所以 `valid_completion_rate = 35.42%` 这个数里混进了两件事：故障注入（真该拒）与粗网格精度不足（不该算成"物理不正确"）。**建议把容差写成 `tol(n) = C·(1/(n-1))²`，并在 `threats` 里说明这是离散误差而不是模型误差** ——这也正好对应立项书 §5.4"明确解误差、离散误差、模型形式误差、测量误差"的要求（文档要求了，代码没做）。

### 3.5 `elevated` 是三条写死的规则，不是"风险模型"

```python
elevated = case.n <= 9 or abs(case.amplitude_k) >= 20 or abs(case.right_k-case.left_k) >= 80
```

- `n <= 9`：粗网格 → 近似正确（确实更容易错）；
- `|amp| >= 20`：源项大 → 与误差容差的关联并**没有在代码或文档里论证**；
- `|Δk| >= 80`：温差大 → 但本实验里 `right_k - left_k` **恒等于 20**（`evaluation.py:22-23`），这一条**永远为假**。

第三条是死分支——实验数据里根本不存在温差 80 的案例。所以"三层风险分层"实际只有两条规则在工作，而且没有任何校准证据说明它们对。

### 3.6 第三个贡献（孪生更新门控）**没有任何实验结果**

立项书 §0 列了三条主创新，第 3 条是"可信的孪生更新门控"。`twin.py` 实现了、`test_twin.py` 也测了，但：

- `evaluation.py` 只 `import` 了 `Case`、`Verifier`、`check_readback`、`reference_error`，**完全没有调用 `TwinStore`**；
- `results/pilot.json` 里没有任何 `SYNTHETIC_UPDATE_ACCEPTED` / `FROZEN_*` 的统计。

也就是说：**三条主张里有一条（而且是唯一涉及"数字孪生"的那条）连一个合成数据点都没有。** 而立项书标题里恰恰有 "Digital Twin"。

### 3.7 其余

| # | 问题 | 证据 |
|---|---|---|
| 1 | 只有 1 个种子（19）、48 个案例，无法给任何 CI 之外的统计结论 | `evaluation.py:38` |
| 2 | `budget=20` 与检查成本之和恰好 20（1+1+1+2+3+12），预算几乎不会 bind，于是"预算约束"这个卖点没被检验 | `evidence.py:88-114` |
| 3 | 抽象"验证单位"与实测耗时的换算关系完全没有解释：省 19.6% 单位只换来 1.7 ms | `evaluation.py:68-69` |
| 4 | `random` 策略的 FAR = 0.5 × 幸存案例数，是硬币的产物，不是"随机基线"的合理形态 | `evidence.py:97-103` |
| 5 | `exit_only` 的 `abstention_rate = 8.33%`（4 例未收敛），与"只信退出码"的语义混在一起，容易被误读为"它也会弃权" | `pilot.json` |
| 6 | 35 个生成页 + 44 KB 的 `research/02x_original.md` 副本 | — |

---

## 4. 项目自己已经承认的负结果与红线

1. **"当前任务太简单、同质且标签源是 MMS，固定检查与自适应均未产生可观察的有效完成率优势"**（`QUALITY.md:20`）。
2. **"零次错误事件不等于零真实风险，必须结合 Wilson 区间"**（`QUALITY.md:20`）。
3. **"实验只构成工程 smoke test，不能用来宣称 SCI 创新被证实"**（`QUALITY.md:20`）。
4. **"未接 FEniCSx、COMSOL、弹性 PDE、实时传感器、真实 Agent 或公开强基线"**（`README.md:39`）。
5. **"预算是相对计量单位，验证选择是启发式，非经校准的误接受风险上界"**（`README.md:39`）。

**第 1 条几乎就是 §3.1/§3.2 的自我预告**——它说的是"没优势"，我做的是"为什么必然没优势"。两者不矛盾，后者更有用。

---

## 5. 改进清单

### P0 — 不做这些，核心指标是恒等式

| # | 要做什么 | 为什么 | 验收标准 |
|---|---|---|---|
| P0-1 | **把 oracle 与验证器彻底分开**：真值改用 **n=257 的收敛解**（或解析级数解的高阶截断），容差独立设定（例如 0.05 K），且**不告诉验证器**这个容差 | 现在两边同一个 `reference_error(...) <= 0.17`（§3.1） | 至少出现 1 个"验证器 PASS 但 oracle 判无效"的案例（FAR > 0 且非零事件），并给出 Wilson 上界 |
| P0-2 | **出"故障 × 检查层"边际检出矩阵**：每类故障在哪一层被拒、每层独有贡献多少 | 现在最贵的层贡献为 0 却占了 62% 成本（§3.2/§3.3） | 一张 7×6 的表 + 每层"删掉它 FAR 会怎样"的消融 |
| P0-3 | **容差随网格缩放**：`tol(n) = C·h²`，并报告 C 的选取依据 | 3 个 n=9 的无故障案例被误判为无效（§3.4） | `valid_completion_rate` 不再混入离散误差；3 个案例回到有效集 |
| P0-4 | **给孪生更新门控做实验**：合成传感器序列 + 4 种漂移（阶跃/斜坡/离群/离线），报 `ACCEPTED / FROZEN_*` 分布与更新误差 | 第三条主张零数据（§3.6） | 至少 200 次更新尝试、六类拒因各出现 ≥10 次 |

### P1 — 决定能不能投

| # | 要做什么 | 验收标准 |
|---|---|---|
| P1-1 | 把 `elevated` 换成**在 dev 集上拟合的粗判器**（逻辑回归即可），并报其在 test 上的覆盖率与 ECE | 有"风险分层是否校准"的证据；删掉永远为假的 `Δk>=80` 分支 |
| P1-2 | 种子 1 → 5，案例 48 → ≥200，跨策略配对 + bootstrap CI | 每个策略的成本差异有区间，不是单点 |
| P1-3 | **补"不可检出"故障族**：残差与边界都通过、但物理上错的案例（例如材料在子区域错、几何拓扑变了导致选择静默失效） | 验证器出现真实的假阴性，FAR 曲线才有区分度 |
| P1-4 | 接 FEniCSx（至少一个 PDE）+ 一个弹性问题 | 满足立项书 §5.1 的 T1；跨引擎泛化（RQ4）才谈得上 |
| P1-5 | 把"验证单位"换算成真实成本（Go 求解调用次数 × 实测 ms），并说明两者为何不一致 | 19.6% 单位节省对应多少真实时间，一目了然 |

### P2 — 仓库与可读性

| # | 要做什么 |
|---|---|
| P2-1 | 删 35 个生成页；`research/02x_original.md` 改为指针 |
| P2-2 | `random` 策略改成"等预算随机选择检查子集"，而不是抛硬币决定要不要做参考校验 |
| P2-3 | Windows 真机跑通 `check_windows.bat`（Go 求解器在 Windows 下的编译/路径问题最可能翻车） |
| P2-4 | `solver.py` 与 Go 之间的协议加版本号，避免 Go 端指标变化后 Python 静默失效 |

---

## 6. 只改五件事的话

1. **独立 oracle**（高分辨率参考 + 独立容差）→ 让 FAR 从恒等式变成指标。
2. 故障 × 检查层矩阵 → 立刻能看出"最贵那层没用"，这是论文最有价值的消融。
3. 容差随 h² 缩放 → 别把离散误差当故障。
4. 给孪生门控补实验 → 否则标题里的 "Digital Twin" 站不住。
5. 校准 `elevated` → 让"动态验证"从三条 if 变成有依据的分层。

---

## 7. 在全系列里的定位建议

| 关系 | 说明 |
|---|---|
| 与 01X（可靠性） | 两篇的决策语义可以统一成一个 4 值格：`ALLOW / DEFER / ABSTAIN / ESCALATE`；01X 缺"证据"这一维度，02X 补上 |
| 与 03X（电力） | 03X 的"独立 AC oracle"与本文的"独立 MMS oracle"是同一个方法论问题——两篇都应该读一下 03X 的 `oracle.py`，它的独立性做得比本文好一点（不同求解器 + 不同方程写法） |
| 与 04X（因果） | 04X 的"证据等级"字段可以直接拿来给本文的每条 check 打标 |
| 与 05X（安全） | 05X 的"硬约束永不学习"对应本文的 `ABSTAIN_UNVERIFIABLE`——都不允许模型自己给自己发许可 |

**推荐定位**：**五份里学术上限最高、但落地成本也最高的一份。** 判断依据：

- 它的领域壁垒最厚（PDE / 数值分析 / 数字孪生），一旦做出来别人很难快速跟进；
- 它的实验**技术栈已经打通**（Go 求解器 + MMS + 独立残差 + SQLite 孪生），剩下的是**评价设计**问题，不是从零开发；
- 但它对团队的要求也最高：立项书 §14 承认"首个瓶颈在 PDE 物理/数值分析，而不是 Agent 框架"。

**给学弟的最短路径**（我建议按这个顺序）：

1. 第 1 周只做 P0-1：写一个 `n=257` 的解析级数解当 oracle，容差独立设 0.05 K，看 FAR 是否变成非零——**这一步就能判断这个题目有没有戏**。
2. 第 2 周做 P0-2 的那张矩阵（数据已在 `pilot.json` 里，纯分析，不需要新实验）。
3. 如果 P0-1 之后 FAR 依然接近 0，说明**你的检查太强、题目太平**，应该按 P1-3 去构造"不可检出故障"，而不是调参。

---

## 附录：文件地图

```text
02x.md                          ← 立项书（565 行）：§1.1 三层区分、§5.2 十类故障、§11 十条警示
02xcode/
├─ solver/main.go               ← 149 行，Go 二维稳态传热（独立求解器）
├─ python/twinproof/model.py    ← 57 行，类型 + 单位校验（fault 不进验证器）
├─ python/twinproof/evidence.py ← **133 行，六层检查 + tolerance_k=0.17（核心）**
├─ python/twinproof/evaluation.py ← **74 行，oracle 与真值定义（问题所在）**
├─ python/twinproof/twin.py     ← 72 行，**孪生更新六道门（最完整的一段）**
├─ docs/QUALITY.md              ← 27 行，**"没有可观察优势"的自述**
├─ docs/EXPERIMENTS.md          ← 17 行，实验协议与缺口
└─ results/pilot.json           ← 1,807 行，48 例逐条 + Wilson 区间
```

**阅读顺序建议**：`evaluation.py:43-46`（先看真值怎么定义）→ `evidence.py:88-105`（再看验证器怎么判）→ 两段对照就能发现问题 → 然后读 `twin.py`（这是它最好的代码）→ 最后看 `pilot.json` 的 `summary`。
