# Paper_Phase_1_X

**Agent & Gateway Research — Cross-Disciplinary Phase I**

本仓库用于整理和推进第一阶段的五项交叉学科科研课题，主要围绕 **LLM Agent、后端网关、可靠性工程、科学计算、电力系统、因果推断及网络安全博弈** 展开研究。

项目尝试将智能体系统的工具调用、任务规划与执行控制能力引入不同工程领域，探索跨学科环境下的可靠性、安全性、可验证性及优化问题。

五项研究采用独立立项、独立实现和独立验证的方式推进，当前主要处于研究原型与实验验证阶段，尚未形成经充分验证的论文成果。

## 01X — Agent × 可靠性工程 × 安全控制

**RAB-Gate: Risk-Budgeted Runtime Assurance for Tool-Using Agents**

本课题研究工具调用型 Agent 在复杂执行任务中的累积风险与安全控制问题。单次工具操作符合安全要求，并不意味着连续执行的整体任务具有足够的可靠性。研究尝试融合可靠性工程中的风险评估、故障树分析与运行时安全控制方法，建立风险预算驱动的动态安全决策网关，在保障关键执行约束的同时，提高任务安全完成率，并分析风险控制策略对系统可用性及执行成本的影响。

## 02X — Agent × 科学计算 × 数字孪生

**TwinProof-Gate: Evidence-Calibrated Runtime Assurance for Agent-Driven Scientific Digital Twins**

本课题研究 Agent 自动调用科学计算工具时的物理可信性验证与数字孪生状态更新问题。传统自动化系统通常关注仿真任务是否成功运行，却难以保证计算结果满足物理规律和数值精度要求。研究尝试通过独立物理证据检查、数值误差评估、动态验证资源分配及观测数据校准机制，构建科学计算验证网关，使 Agent 生成的仿真结果在满足预设可信条件后才能参与数字孪生更新。

## 03X — Agent × 电力系统 × 优化控制

**GridUQ-Gate: Uncertainty-Calibrated Sequential Safety Validation for Agent-Assisted Power Dispatch**

本课题研究 Agent 辅助电力系统调度时的多步安全验证及不确定性控制问题。电网调度具有复杂的物理约束，单步操作安全并不代表连续操作始终满足运行要求。研究尝试结合电力潮流计算、状态估计、序列决策及不确定性量化方法，在 Agent 与电力仿真环境之间构建独立验证网关，对调度动作进行分层物理检查与风险评估，探索系统安全、运行经济性和计算成本之间的协调优化。

## 04X — Agent × 因果推断 × 故障诊断

**CausalGuard-RCA: Budgeted and Safe Interventional Diagnosis for Partially Observable Tool-Using Agents**

本课题研究复杂 Agent 工作流中的故障传播、因果识别和多根因诊断问题。系统最终发生异常的位置不一定是真正的故障来源，传统日志分析也难以区分因果关系与统计相关性。研究尝试融合结构因果模型、反事实分析和安全干预策略，在有限观测与计算预算下主动验证故障假设，实现多个潜在根因的识别与排序，并通过可信度评估和弃权机制降低错误归因风险。

## 05X — Agent × 网络安全 × 博弈论

**GameShield-Gate: Budget-Constrained Adaptive Security Games for Tool-Using LLM Agents**

本课题研究工具调用型 Agent 在动态对抗环境下的安全防御和资源分配问题。面对能够根据防御行为调整攻击策略的攻击者，固定检测规则难以始终保持最佳防护效果。研究尝试融合博弈论、在线决策与网络安全防御机制，构建预算约束下的自适应安全网关，根据攻击行为和风险变化动态分配验证资源，同时保持基础权限控制及关键执行安全约束不被学习策略绕过。

---

## Repository Structure

```text
Paper_Phase_1_X/
│
├── 01X/
│   ├── 01x.md
│   ├── 01xcode.zip
│   └── 01Xdst.md
│
├── 02X/
│   ├── 02x.md
│   ├── 02xcode.zip
│   └── 02Xdst.md
│
├── 03X/
│   ├── 03x.md
│   ├── 03xcode.zip
│   └── 03Xdst.md
│
├── 04X/
│   ├── 04x.md
│   ├── 04xcode.zip
│   └── 04Xdst.md
│
├── 05X/
│   ├── 05x.md
│   ├── 05xcode.zip
│   └── 05Xdst.md
│
├── README.md
├── LICENSE
└── 交叉审查结果.md
```

### File Description

| 文件 | 内容 |
|---|---|
| `XXx.md` | 对应课题的研究方案、创新点、数学模型、关联论文、实验设计和 SCI 期刊分析 |
| `XXxcode.zip` | 对应课题的研究代码原型、算法实现、测试程序及运行说明 |
| `XXXdst.md` | DeepSeek 对研究方案与实现路径提出的独立评审及改进意见 |
| `交叉审查结果.md` | 五个交叉学科项目的代码实现符合性检查、实验问题和后续修复建议 |

`XX` 表示研究编号 `01–05`，具体文件命名以各目录中的实际文件为准。

## Research Workflow

研究按照以下流程推进：

1. **Research Design** — 明确跨学科研究问题、核心假设及主要创新点。
2. **Literature Review** — 检索相关研究，检查创新重叠与研究价值。
3. **Prototype Development** — 实现核心算法、网关模块及实验环境。
4. **Independent Review** — 结合外部评审意见检查算法设计与研究方法。
5. **Experimental Validation** — 开展基线对照、消融实验、鲁棒性测试和统计分析。
6. **Paper Preparation** — 根据真实实验结果完善研究内容并选择合适的 SCI 期刊。

## Current Status

**Phase I — Cross-Disciplinary Research Prototypes**

五项研究已建立初步代码原型，并完成基础测试及部分合成实验。

当前仍存在以下共同研究任务：

- 完善核心算法，并验证相对于已有方法的实际优势。
- 引入具有代表性的公开基准、真实数据或专业仿真环境。
- 完成强基线对照、消融实验与多随机种子重复实验。
- 明确算法理论保证、实验适用条件及研究局限。
- 根据实验结果评估进一步投稿的可行性。

现阶段的代码可作为后续科研开发基础，但基础测试通过不等于论文创新已经得到证实。

## Research Objectives

本阶段的主要目标是探索 Agent 与不同学科方法结合时的实际研究价值，通过可复现实验检验所提出的算法与系统设计。

各课题独立开展研究，不预设算法优越性，也不将尚未完成验证的理论假设视为已证实的成果。

最终根据实验结果、创新性及期刊适配程度决定是否继续开发、调整研究方向或准备 SCI 论文投稿。

## License

This repository is distributed under the [MIT License](LICENSE).

第三方数据集、软件依赖、研究文献及其他引用资源遵循各自的许可协议和学术引用要求。
