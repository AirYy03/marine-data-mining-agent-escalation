# 海洋数据挖掘课程大作业研究与实验报告

**基于状态证据与恢复收益的水下机械臂 Workflow–Agent 选择性接管研究**

> **文档性质**：研究过程回顾 + 方法设计 + 对照实验方案（**实验设计版，待实测**）  
> **整理日期**：2026 年 11 月 9 日  
> **版本**：v2.1（补入真实公开仓库与聊天分享入口）  
> **研究对象**：水下机械臂放置阶段“下降动作完成，但物体稳定支撑未获确认”  
> **研究范围**：单一异常入口、选择性诊断接管、受约束工具调用；不研究完整抓放任务的全部异常

**公开仓库**：[AirYy03/marine-data-mining-agent-escalation](https://github.com/AirYy03/marine-data-mining-agent-escalation)。  
**本次聊天分享链接**：[查看本次课程讨论](https://chatgpt.com/share/6af1d7d9-e370-83ee-8f11-7b318c0ef676)。  
**提交完整性**：当前是实验设计稿，不是已完成实验的最终报告；已收录公开仓库和用户提供的真实聊天分享链接，实测结果仍待补充。研究过程摘要不替代完整逐字聊天导出；若教师要求完整 Markdown 记录，应另附真实导出。

**关于结果真实性的声明**：截至本报告整理时，当前讨论尚未提供本课题 A/B/C/D 四组对照实验、D1 收益预测训练或消融实验的实测日志。因此本文的算法、参数、样本量和研究假设属于**待实施设计**；任何示例配置均不是已取得的实验结果。现有水下机械臂工程进展不能等同于本课程实验已经完成。正式提交“实验结果与讨论”前，必须以实际运行记录替换报告中的待填内容。

---

## 阅读目录

| 部分 | 内容 |
|---|---|
| [1. 课程启发与问题演进](#section-1) | 从课程知识到研究质疑的完整路径 |
| [2. 文献与创新边界](#section-2) | 已有方法、工程组合、待验证改进 |
| [3. 研究范围](#section-3) | 仅研究下降完成后的支撑状态不确定 |
| [4. 输入、输出与分工](#section-4) | 接管器、Agent、技能执行与安全监督 |
| [5. 算法流程](#section-5) | V0/V1、收益成本模型与逻辑伪代码 |
| [6. 异常仿真场景](#section-6) | S1—S6、扰动、随机种子、真值隔离 |
| [7. 对照和消融](#section-7) | A/B/C/D、证据、历史、成本与交叉消融 |
| [8. 评价指标](#section-8) | 效果、覆盖率、安全、耗时与调用成本 |
| [9. 数据与统计](#section-9) | 配对实验、划分、泛化与统计分析 |
| [10. 可证伪假设](#section-10) | 明确允许方案无效或不需要 Agent |
| [11. 实验结果记录](#section-11) | 待测表格，不使用虚构结果 |
| [12. 贡献与局限](#section-12) | 课程已有知识和拟验证的增量 |
| [13. 总结与实施顺序](#section-13) | 后续执行条件及研究结论边界 |
| [参考文献](#references) | 可核查的原始来源 |

**阅读提示**：本文件采用 UTF-8、标准 Markdown 表格、配对代码围栏及 GitHub 支持的数学分隔符。部分纯文本编辑器不会渲染公式，但仍可查看公式源码；公式旁的文字给出相同含义。

## 摘要

在海洋数据挖掘课程中学习 AI 智能体、工具调用、Workflow 与 Agent 的区别，以及“AI 智能体也是分布式系统”的可靠性问题后，研究关注点由“机器人能否调用大模型”转向“何时有必要将特定决策交给大模型”。结合水下机械臂的抓取—搬运—放置任务，本文选取一个范围较小且具有明确物理意义的异常：机械臂已报告完成下降动作，但视觉和接触/载荷证据尚不足以确认物体已稳定获得支撑。针对传统 Workflow 可以进行有限重试、而 Agent 可能增加诊断灵活性但同时带来推理延迟与计算成本的矛盾，设计了基于**状态证据有效性、Workflow 诊断历史和 Agent 条件增量收益**的选择性接管方案。系统分离接管判断、高层技能选择、独立安全监督和物理状态验证：简单问题优先由 Workflow 处理，只有预计 Agent 能带来足以抵消成本的额外诊断价值时才考虑接管；无有效诊断途径时安全终止。报告回顾从课程启发、文献核查、创新性质疑到算法收缩的推导过程，给出初始启发式版本 V0 与待学习版本 V1，设计四组主对照、六类分级仿真异常、关键消融和多维评价指标，并明确如何隔离仿真真值与在线观测。本研究不预设 Agent 优于规则系统，目标是通过可复现实验识别其真实适用边界。

**关键词**：海洋数据挖掘；水下机械臂；Workflow；AI Agent；异常诊断；状态证据；成本敏感决策；选择性接管；运行时安全监督

---

<a id="section-1"></a>

## 1. 课程学习背景与研究问题的形成

### 1.1 课程中最直接相关的知识

| 课程知识点 | 核心理解 | 与研究任务的对应关系 |
|---|---|---|
| Workflow（工作流） | 由开发者预先定义执行路径、分支与重试逻辑；可以包含状态机、行为树与传感器反馈 | 抓取、搬运、下降、确认支撑、释放的既有正常执行流程 |
| Agent（智能体） | 模型在约束内依据任务目标、实时信息和工具返回结果动态决定下一步 | 异常后选择补充观测、姿态检查等诊断技能 |
| 工具调用 | 高层决策与外部功能通过受约束的接口交互 | 感知、接触检查、技能执行与状态查询封装为工具 |
| “AI 智能体也是分布式系统” | 工具超时、反馈延迟、重复调用、状态不一致都会影响决策正确性 | 运动完成不等于稳定落座；超时不等于动作未执行 |
| 数据挖掘与实验验证 | 通过有效特征、运行日志、对照实验评估某项策略是否有可重复收益 | 从状态证据及历史信息预测 Agent 接管的增量价值 |

本报告依据本次讨论中已经明确的课程要求组织：选取足够小的知识点，展示逐步追问、质疑与方案修改，提出可验证的延伸并保留真实 AI 使用过程。尚未取得完整评分细则，不额外假定字数、截止时间或教师模板。

需要强调：**Workflow 不是“完全没有判断能力”**。成熟状态机、行为树也可以根据传感器数据重新观测、重新规划、有限重试并安全停止。Agent 的区别主要在于部分工具选择与任务编排权由运行时模型动态承担，而非“有无 if-else”或“是否使用 AI”。这一认识与已有 Agent 工程实践中的概念区分一致 [1]。

### 1.2 从课程知识到当前机械臂问题

水下机械臂的正常任务可表示为：

```text
目标感知 → 抓取规划 → 闭爪抓取 → 抬升 → 搬运
       → 持物下降 → 支撑确认 → 张爪释放 → 安全撤回
```

起初提出的方案是：**正常流程使用 Workflow，异常时调用 Agent**。随后发现，这一表述过于宽泛：抓取失败重试、接触状态重检、超过次数安全停止，都能用状态机完成，且类似混合架构已有先例。因此逐步将研究范围缩小至：

> **机械臂已完成下降动作，夹爪保持闭合，但支撑状态仍为 `UNKNOWN`；系统应继续常规诊断、调用 Agent，还是停止自动处理？**

选取该节点的依据是：它位于物理风险明确的释放动作之前；其观测可能不充分；它既能测试工具调用决策，也能测试反馈一致性与安全控制，而不必重新研究整个抓放任务。

### 1.3 研究思路的逐步演进与关键质疑

| 阶段 | 最初想法或疑问 | 反思得到的结论 | 形成的设计决定 |
|---|---|---|---|
| ① 课程启发 | Agent 能否替代复杂流程？ | Workflow 自身就能处理大量异常 | 不直接用 Agent 代替正常控制 |
| ② 联系实际 | 水下抓放连续操作中哪里会出问题？ | 全部异常范围太大 | 只研究放置后的支撑状态不确定 |
| ③ 方法质疑 | “失败就重试”不是 if-else 就能做吗？ | 是；简单异常无需 Agent | 设置强规则 Workflow 基线 |
| ④ 文献调查 | 混合控制是否已有人做？ | 状态机+LLM、行为树+VLM、学习恢复策略均有先例 | 不宣称首创混合架构 |
| ⑤ 缩小研究边界 | 只研究 `SUPPORT_UNKNOWN` 是否足够？ | 可以，但需区分诊断成功与物理放置成功 | 第一阶段只研究诊断与接管 |
| ⑥ 证据反思 | 视觉与接触结果不同就该叫 Agent 吗？ | 不一定；可能是过期数据或不同物理语义 | 加入证据有效性、时效性、一致性检查 |
| ⑦ 成本反思 | 每次矛盾都叫 Agent 值得吗？ | Agent 可能只会重复 Workflow 的检查 | 比较接管后相对于 Workflow 的增量收益和成本 |
| ⑧ 创新性质疑 | 四因素加权是不是换了几个 if-else？ | 是；仅规则融合没有明显方法创新 | 将 V0 规则与 V1 数据驱动收益预测严格区分 |
| ⑨ 安全问题 | Agent 可能选错技能，如何防止危险？ | 工具白名单不等于物理安全 | Agent 只申请技能，由独立监督器批准与监控 |
| ⑩ 实验设计 | D 比 A/B/C 好，是否证明模块有用？ | 不能；需控制变量并消融 | 四组主实验 + 单因素/交叉消融 + 泛化测试 |

这条演进路径体现了课程研究中的一个重要原则：**不预先断言 Agent 必须有用，而是把“是否有用、何时有用”本身当作待验证问题。**

---

<a id="section-2"></a>

## 2. 文献调查与创新边界

### 2.1 已核对的代表性工作

| 文献 | 正式发表信息 | 主要机制 | 对本研究的限制或启示 |
|---|---|---|---|
| **HeRo** [2] | ICRA 2025 | 状态机组织多机器人任务，监测故障后结合 LLM 与 PDDL 生成恢复逻辑 | “Workflow+异常时 LLM 恢复”已有先例 |
| **BETR-XP-LLM** [3] | ICRA 2025 | 当现有规划器不能处理错误时，以 LLM 扩展行为树，并更新策略 | “仅在传统方法不足时调用 LLM”已有先例 |
| **Ahmad 等** [4] | IEEE CASE 2025 | VLM 检查技能条件，反应式规划器调整行为树及恢复方案 | VLM 诊断、执行历史与任务修正已有相关研究 |
| **Sentinel** [5] | CoRL 2024（PMLR 论文集于 2025 年发布） | 低成本统计监测与 VLM 监测分别识别不同失败模式 | 高成本模型不必负责所有监测；检测和恢复需区分 |
| **RecoveryChaining** [6] | IROS 2025 | 故障时切换到学习型恢复策略，再返回常规模型驱动控制器 | 自主恢复与控制权交还不依赖 LLM |
| **AgentChord** [7] | RSS 2026 | Agent 在执行前预编译任务与恢复分支，运行时监测器触发 | 在线临时调用 Agent 可能存在额外延迟 |
| **Banerjee 等** [8] | ACM/IEEE HRI 2026 | 结合模块不确定性和人工求助成本，分开研究“向谁求助”“何时求助” | 不确定性与成本敏感接管的思想已有先例 |
| **Ota 等** [9] | ICRA 2024 | 借助触觉/力反馈估计外部接触区域以支持稳定放置 | 接触与稳定落座已有专门研究，不能等同为新问题 |
| **Ghoummaid 与 Shalit** [10] | NeurIPS 2024 | 研究何时自主行动、何时转交专家，讨论选择性求助策略 | 增量效用学习属于既有研究范式 |

上述文献之间的实验平台、任务和指标不同，因此**不能直接横向比较论文中的成功率**。本表主要用于界定方法先例与研究空缺，不用于证明本文方法已经优于它们。

本次复核的范围是论文题名、正式发表信息和与本研究直接相关的方法概述，并非穷尽数据库的系统综述。可由原始来源核查的实验依据包括：HeRo 在 ARIAC 所设制造任务与故障条件中报告 100% 完成率 [2]；Sentinel 报告组合监测比单独使用任一监测器多检测出 18% 的失败，这不是本文方法的提升，也不应擅自改写为 18 个百分点 [5]；RecoveryChaining 在三个多步骤任务中评价恢复策略，并展示仿真到实体机器人的迁移 [6]；Ahmad 等在 ABB YuMi 与 AI2-THOR 上研究失败处理 [4]。这些结果各有任务与观测假设，不能据此推断本课题中的 Agent 一定有效。

尤其需要区分：Sentinel 主要研究**失败检测**；RecoveryChaining 使用**强化学习恢复策略而非 LLM**；AgentChord 着重**执行前准备恢复分支**；HRI 2026 的求助对象为**人类**。它们是相关先例，不是与本研究完全等价的四组实验。


### 2.2 严格区分已有方法、工程组合与待验证部分

**已有知识与成熟技术**：状态机、行为树、工具调用、传感器时效性检查、运行时安全监督、成本敏感决策、选择性求助、主动信息获取。  
**工程组合**：给水下机械臂的 `SUPPORT_UNKNOWN` 节点增加 Workflow/Agent 切换、工具白名单和独立状态验证器。  
**待验证的研究增量**：在统一工具与安全约束下，利用物理与反馈证据、诊断历史，估计 **Agent 相对强规则 Workflow 的条件净收益**，并证明该估计能在未知扰动条件下改善“诊断效果—调用成本”权衡。

这仍然不能直接声称“理论首创”。如果实验表明强规则 Workflow 或轻量级工具选择器达到同等效果且成本更低，结论应是当前受限场景不需要复杂 Agent。

---

<a id="section-3"></a>

## 3. 研究问题、系统假设和范围

### 3.1 唯一异常入口

在时刻 $t_0$，系统满足：

$$
\mathrm{MotionCompleted}=1,\qquad
\mathrm{GripperClosed}=1,\qquad
S_{\mathrm{support}}=\mathrm{UNKNOWN}.
$$

“下降动作完成”指运动执行器报告完成，不代表物体真实接触或稳定。稳定放置至少涉及**几何接触、载荷转移和释放后稳定性**，三者不可混为一谈；水下还需考虑浮力和水流对支撑关系的影响。

真实状态与在线观测分别定义为：

- $S^{\mathrm{true}}$：仿真离线评价使用的真实支撑状态；
- $o_t$：加入遮挡、噪声、延迟后的在线可观测信息；
- $S_t^{\mathrm{obs}}\in\{\mathrm{SUPPORTED},\mathrm{UNSUPPORTED},\mathrm{UNKNOWN}\}$：独立验证器输出。

**禁止在线决策组件读取 $S^{\mathrm{true}}$**。

水下浮力使载荷证据的可辨识性依赖物体：完全浸没且忽略动态项时，表观重量近似为 `mg - rho*g*V`。接近中性浮力时，未观测到显著载荷变化不能直接推出未接触；具有正浮力时，下方平台接触也不能独自保证释放后的稳定停留。本文不提出新接触估计器，统一使用经过单独标定的状态验证器；触觉与力觉对释放前接触稳定性判断的相关研究见 [9]。

### 3.2 研究阶段边界

| 研究内容 | 本课程第一阶段 |
|---|---|
| 已有抓取、抬升、搬运、持物下降 | 固定，不做算法优化 |
| 支撑状态 `UNKNOWN` 的成因与证据 | 纳入 |
| Workflow 与 Agent 接管时机 | 核心研究 |
| Agent 选择受约束诊断技能 | 纳入 |
| 新的接触控制、运动规划、自由轨迹生成 | 不纳入 |
| Agent 直接输出关节、速度、力矩 | 禁止 |
| 物理重新落座/受限下降 | 只作为后续扩展，须先独立验证 |
| 张爪和撤回 | 使用现有 Workflow 与独立释放许可，不由 Agent 直接控制 |
| 完整抓放任务成功率 | 仅在后续真正执行释放和撤回的闭环实验中作为主指标 |

**当前首要评价对象是“正确确认支撑状态”而非“恢复了完整放置”。**如果诊断出物体确实悬空，属于诊断正确，但不是放置成功。

### 3.3 研究问题

- **RQ1（必要性）**：相同观测、技能库和安全约束下，LLM Agent 是否存在相对于强规则 Workflow 的状态依赖性诊断收益？
- **RQ2（接管）**：状态证据、诊断历史与估计增量收益能否帮助决定何时调用 Agent，并改善诊断效果与成本的权衡？
- **RQ3（可靠性）**：在数据过期、矛盾、工具超时等条件下，受约束接管是否保持正确的状态管理与独立安全边界？

---

<a id="section-4"></a>

## 4. 系统架构、输入与输出

### 4.1 分层系统结构

```text
水下机械臂原有 Workflow
（抓取 → 搬运 → 下降 → 支撑确认）
                   │
           支撑状态 UNKNOWN
                   │
                   ▼
       状态读取与证据有效性检查
                   │
                   ▼
       Workflow 低成本常规诊断
                   │
             仍然 UNKNOWN
                   │
                   ▼
       选择性接管决策器（V0 / V1）
          ├─ 继续 Workflow
          ├─ 调用 Agent ──→ 选择诊断技能
          └─ 安全处置           │
                              ▼
                 工具权限、参数与安全监督
                              │
                              ▼
                    技能执行 / 结果查询
                              │
                              ▼
               新状态证据 + 独立支撑验证
                   │
       ┌───────────┼─────────────┐
       ▼           ▼             ▼
   SUPPORTED   UNSUPPORTED     UNKNOWN
       │           │             │
  交还 Workflow  禁止释放       预算内重判
  释放前再审核   安全处置       或安全处置
```

所有有物理副作用的命令，包括正常 Workflow 发出的命令，均应通过共同的安全监督与控制入口。独立验证器不根据 Agent 的语言结论直接授权张爪。

### 4.2 状态与证据输入

$$
x_t=(R_t,E_t,H_t,T_t,B_t),
$$

- $R_t$：关节/夹爪/运动状态、任务及动作 ID；
- $E_t$：视觉间隙与姿态估计、接触/载荷反馈、有效标志、采集时间、估计不确定性；
- $H_t$：已经尝试的工具、请求 ID、执行状态、是否取得新信息；
- $T_t$：`UNKNOWN` 持续时间；
- $B_t$：剩余诊断时间、工具次数、Agent 推理次数。

每项证据应携带“对象 ID、任务 ID、动作 ID、时间戳、测量值、质量状态、估计误差”等元数据。接触反馈仅在当前平台具有真实传感器或可信观测模型时使用；**仿真引擎的直接接触真值不得包装成传感器工具输出**。

### 4.3 决策与执行输出

接管决策：

$$
g(x_t)\in\{W,A,S\},
$$

分别表示 Workflow、Agent 和 Safe Stop。Agent 的输出只允许是结构化工具请求：

```json
{
  "task_id": "placement_001",
  "state_version": "v17",
  "decision": "CALL_TOOL",
  "tool": "inspect_object_pose",
  "arguments": {},
  "reason_code": "POSE_UNCERTAIN"
}
```

初期允许的只读/非运动诊断工具：`refresh_robot_state`、`reobserve_placement`、`inspect_object_pose`、`recheck_contact`；另外可请求 `safe_terminate`。不允许 Agent 直接调用 `open_gripper`、低层运动命令或修改安全阈值。后续若增加 `bounded_seating_probe`，只能选择经过独立验证的固定技能配置，且必须满足运行时接触/运动约束。

---

<a id="section-5"></a>

## 5. 选择性接管算法

### 5.1 Step 1：读取状态并验证证据

首先读取与当前放置动作关联的快照，检查：

1. **完整性**：测量是否缺失、饱和或明显无效；
2. **时效性**：$t_{\mathrm{now}}-t_i\le\tau_i$，且关键测量采集于最后一次相关动作之后；
3. **一致性**：两种证据是否对应同一目标、同一动作及可比时刻；
4. **物理语义**：视觉“几何接触”和载荷“部分承重”能否同时成立，不能将其机械地判为矛盾。

输出证据类别 `VALID / INVALID / STALE / MISALIGNED` 和物理证据关系 `CONSISTENT / CONFLICT / INCONCLUSIVE`。时效阈值应依据采样率、估计误差和运动状态标定。

### 5.2 Step 2：Workflow 优先处理明确低成本问题

例如数据过期则刷新；视频遮挡但有新视角则按规则重新观察；接触反馈偶发异常则在预算内重检。每次记录新证据是否真正提高可判定性。若已能确认支撑，则终止诊断并交还正常流程；若确认为未支撑，则禁止释放；如仍为 UNKNOWN，才评估是否升级。

“重复两次仍 UNKNOWN”本身不能证明需要 Agent：还要区分“重检正逐步产生新信息”和“同一种检查一直无效”。

### 5.3 Step 3：V0 初始可实现版本

在尚无可靠收益训练数据时，采用保守可诊断性门控：

$$
g_0(x_t)=
\begin{cases}
W,&\text{有明确且预算内的常规诊断机会};\\
A,&\text{常规诊断难以推进，存在替代信息路径且预算允许};\\
S,&\text{无安全可行的新诊断途径或预算耗尽}.
\end{cases}
$$

**V0 仍是规则方法，是实验基线而不是原创学习算法。**安全故障优先触发底层保护，不能等待以上比较。

### 5.4 Step 4：V1 数据驱动收益—成本决策

令 $W$ 表示继续强规则 Workflow，$A$ 表示调用 Agent。两者的终点、剩余预算和技能集合必须相同。

$$
\Delta P(x)=P(\text{正确诊断}\mid x,A)-P(\text{正确诊断}\mid x,W),
$$

$$
\Delta T(x)=\mathbb E[T\mid x,A]-\mathbb E[T\mid x,W],
$$

$$
\Delta C(x)=\mathbb E[C\mid x,A]-\mathbb E[C\mid x,W].
$$

定义预计净增量效用：

$$
\boxed{V(x)=\Delta P(x)-\lambda_T\Delta T(x)-\lambda_C\Delta C(x)}.
$$

$\lambda_T$ 和 $\lambda_C$ 是在开发阶段事先确定的效用换算权重，不能直接把概率、秒数和货币数值相加。若模型已通过独立验证并具备可信不确定性估计，可在可行与预算约束下，仅当预计净收益的保守下界大于零时才调用 Agent；不能把 Bootstrap 区间自动解释为安全保证。无有效新信息途径时选择 S。

**如何获得训练数据**：对同一保存的初始或中间诊断状态，分别运行 W 和 A，记录正确诊断、诊断时长、工具调用和计算成本；必要时使用多个随机种子重复。训练目标可为配对经验净收益 $y_i=U_i^A-U_i^W$。这是一种条件增量收益估计，**属于已有成本敏感决策与 Learning to Defer 思想的具体应用**；其是否对该场景有价值必须通过测试判断。


### 5.4.1 初始实现的决策时域与收益数据

为避免训练标签与实际执行策略不一致，第一版 D1 在**一个预先固定的决策检查点**选择是否接管，例如完成最多一次明确的数据刷新之后。检查点和前缀诊断规则在开发集固定，并记录其成本。D1 选择 W 后由固定强规则策略执行剩余诊断；选择 A 后由固定 Agent 执行剩余诊断；本 Episode 不再反复改派。因而配对标签对应同一检查点、同一剩余预算下两种完整后续策略的差异。

B 仍然在共同的基本安全检查后立即调用 Agent，C 仍然使用事先固定的 N 次 Workflow 尝试，A 始终不接管。D1 的前缀不能另获额外工具次数或时间。若以后改成每一步重新评估，必须另采集中间状态并明确后续策略，否则不能直接复用单次转交的收益标签。

建议分别学习成功率和耗时/资源成本，而不是只学习一个固定权重的总分。这样可以对同一成功率预测进行成本项消融，也可在合理范围内测试不同调用延迟。收益估计使用配对仿真结果，不使用 LLM 自报的“成功概率”。样本不足时先报告经验均值与不确定性，V0 是可运行的启发式基线，不伪装成最优策略。

所有动作价值均是**尚待统计估计**的量。所谓“反事实”指从可重置仿真状态分支执行得到的对照，不意味着观测了真实世界中同一事件的两个结果，也不自动获得因果无偏性。仅采集 V0 曾调用 Agent 的状态会造成选择偏差，训练数据必须覆盖其他安全可评估状态。

### 5.5 Step 5：Agent 工具选择、安全审核与结果校验

Agent 只获得经过验证的状态快照、诊断历史、剩余预算和当前可用工具；执行器对请求依次实施：白名单校验、参数校验、技能前置条件检查、物理安全检查。真正运动的技能还需要执行中持续监督。工具返回状态必须区分 `SUCCEEDED / FAILED / TIMEOUT / REJECTED / UNKNOWN`；出现超时时，先按唯一请求 ID 查询原任务执行状态，不得盲目重复具有物理副作用的动作。工具成功不等于稳定支撑成功；执行后仍需独立验证。

释放许可：

$$
\mathrm{ReleaseAllowed}=
\mathrm{SupportVerified}\land
\mathrm{StabilityVerified}\land
\mathrm{EvidenceFresh}\land
\mathrm{SafetyApproved}.
$$

许可应绑定当前物体、动作与状态版本，在物体状态变化或证据过期后失效。**Agent 无权直接令 `ReleaseAllowed=True`。**

### 5.6 端到端伪代码（逻辑伪代码，非实机控制程序）

```text
Algorithm 1: 受约束的放置后诊断（逻辑伪代码，不连接真实电机）
Input:
  task, policy(A/B/C/D0/D1), observations, skills, safety, verifier,
  budget, optional_gain_model, frozen_configuration
Output:
  diagnosis ∈ {SUPPORTED, UNSUPPORTED, UNRESOLVED},
  stop_reason, execution_status, history, elapsed_time, resource_cost

history ← empty
owner ← NONE                  # 尚未选定后续诊断策略
start_wall_clock_and_budget()

while budget.not_exhausted():
    snapshot ← observations.read_versioned_snapshot(task)
    if basic_robot_state_unavailable(snapshot) or safety.hazard(snapshot):
        safety.request_prevalidated_disposition()
        return UNRESOLVED, SAFETY_OR_STATE_FAILURE, history

    evidence ← validate_quality_age_alignment_and_semantics(snapshot)
    diagnosis ← verifier.evaluate_using_observations_only(evidence)

    if diagnosis == SUPPORTED:
        # 只交还流程；释放前须再次验证最新状态。
        return SUPPORTED, RETURN_TO_WORKFLOW_FOR_RELEASE_RECHECK, history
    if diagnosis == UNSUPPORTED:
        safety.request_prevalidated_disposition()
        return UNSUPPORTED, SUPPORT_NOT_ESTABLISHED, history

    # A/B/C/D 的区别只在策略选择。共同安全机制不得删除。
    if owner == NONE:
        route ← policy.select_at_its_defined_checkpoint(
            snapshot, evidence, history, remaining_budget,
            optional_gain_model, frozen_configuration)
        if route == SAFE:
            safety.request_prevalidated_disposition()
            return UNRESOLVED, NO_JUSTIFIED_CONTINUATION, history
        if route == WORKFLOW_STEP_BEFORE_CHECKPOINT:
            request ← fixed_workflow.select_next_tool(evidence, history)
        else:
            owner ← route      # 第一版 D1 一次选定 W 或 A 后锁定

    if owner == WORKFLOW:
        request ← fixed_workflow.select_next_tool(evidence, history)
    if owner == AGENT:
        request, inference_cost ← agent.propose_whitelisted_tool(
            observable_snapshot, history, allowed_tools, remaining_budget)
        charge_inference_even_when_invalid_or_timed_out(inference_cost)

    if budget.exhausted() or request.is_stop_or_unavailable():
        safety.request_prevalidated_disposition()
        return UNRESOLVED, NO_ACTION_OR_BUDGET_EXHAUSTED, history

    # 推理期间状态可能变化，不能只审核推理前的旧快照。
    fresh_snapshot ← observations.read_versioned_snapshot(task)
    approval ← safety.validate_schema_permission_preconditions_and_state(
        request, fresh_snapshot)
    if not approval.allowed:
        append_rejection_and_charge_decision_budget(history, request, approval)
        continue

    request_id ← unique_id()
    result ← skills.execute_via_single_authorized_entry(
        request, request_id, approval.state_version, budget.deadline)
    # 副作用动作执行中由独立控制器持续监控；取消请求不等于硬件急停。
    if result.is_timeout_or_unknown():
        result ← reconcile_existing_request_without_reissuing(request_id)
        if result.remains_unknown():
            safety.request_prevalidated_disposition()
            return UNRESOLVED, EXECUTION_STATE_UNKNOWN, history

    history.append(request_id, request, result, measured_time, measured_cost)
    budget.charge_all_measured_costs()
    # 工具完成后重新读取；任何文本“已落座”都不能自行更新释放许可。

safety.request_prevalidated_disposition()
return UNRESOLVED, BUDGET_EXHAUSTED, history
```

以上伪代码中的函数为接口契约而非已实现代码；所有终止路径都保留独立诊断标签，避免将正确识别未支撑与未知停止混成同一结果。对于已知未支撑的物理恢复，第一阶段直接交回既定安全流程，第二阶段才另行开放经过验证的恢复技能。ROS 2 Action 的目标、反馈、结果和取消语义可参考官方设计文档 [11]；请求超时后的具体去重和状态核对应由应用实现。

设计细节：接管决策器只能使用当前及过去可获取的信息；每次重评不允许使用之后才产生的工具结果。安全层对 Workflow 和 Agent 一视同仁。

---

<a id="section-6"></a>

## 6. 仿真场景与数据生成

### 6.1 六类分级异常

**所有 Episode 共用入口**：下降完成、夹爪闭合、在线验证器返回 `UNKNOWN`。初始 `UNKNOWN` 必须由传感器观测经过相同验证器得到，而不能仅靠强制改写状态字段产生。

| 场景 | 物理状态 / 观测故障 | L1 → L3 难度控制 | 主要测试能力 |
|---|---|---|---|
| S1 短暂误报 | 物体可能已稳定，接触反馈短暂噪声/丢帧 | 丢帧时长、噪声幅度增大 | 避免不必要调用 Agent |
| S2 未完全接触 | 真实物体仍存在间隙 | 间隙相对深度估计标准差减小，例如约 $3\sigma_d$ 到 $0.5\sigma_d$ | 正确识别未支撑 |
| S3 姿态偏移 | 偏斜、局部接触或支撑不足 | 平移偏差、倾角及支撑区域变化 | 区分接触与稳定支撑 |
| S4 多源证据矛盾 | 视觉估计与接触/载荷信息给出难以兼容的证据 | 视觉/载荷偏差、持续性与不确定性 | 选择互补诊断工具 |
| S5 复合异常 | 遮挡、噪声、姿态偏移和延迟组合 | 2 类轻扰动 → 多类强扰动 | 未见组合泛化 |
| S6 无法安全继续 | 工具失效或存在独立保护应处理的危险条件 | 可用工具减少、状态可诊断性降低 | 避免无效接管与不必要停止 |

上表等级为**待标定的实验因素**，不是已验证的难度次序。较大倾角或间隙有时反而更容易识别；正式分级以预实验中的可观测性与诊断所需信息为依据，而非单纯增大物理偏差。S4 只有对同一个物理命题、时间与对象都可比时才定义为“真实冲突”；几何接触与部分载荷并不天然矛盾。S6 应区分“可正确诊断但无法物理恢复”与“本身不可诊断”。S6 中入口即被共同安全机制禁止继续的 Episode 单列为安全压力测试，不混入主要接管效果排名；否则零调用可能仅由共同保护产生。不能让某一个场景编号固定对应单一真实标签，否则容易形成学习捷径。

### 6.2 扰动与随机种子

对每个 Episode 固定：物理状态、相机观测、接触观测、故障注入配置及随机种子。

建议独立记录：`seed_physics`、`seed_vision`、`seed_contact`、`seed_agent`。所有 A/B/C/D 从**同一恢复后的初始状态**独立执行；最好预先生成可重放的传感器噪声时间序列，避免因为调用工具次数不同导致后续噪声错位。不能假设同一个 GPU/网络调用种子必然带来完全确定的 LLM 响应。

如果研究水下保持姿态期间的物理变化，仿真物理时间应与等待/推理时段按统一规则推进。不能一组在 LLM 推理时冻结水动力，另一组继续积分，然后直接比较风险。纯冻结快照诊断可以另作受控子实验，但不能把其安全性外推为动态水下保持安全。

实验生成器、在线观测包装器、策略执行和离线评价分进程或通过最小权限接口隔离。记录依赖版本、模型版本、提示词版本、配置摘要与时间基准。物体/机器人状态重置还须覆盖速度、夹持约束、控制器内部状态及传感器缓冲区；仅恢复位置并不能保证等价初始条件。

### 6.3 真值隔离

```text
仿真物理真值 ─────────────────────────→ 仅离线评价器
      │
      ▼
传感器仿真（噪声、遮挡、延迟、量程）
      │
      ▼
Observation API ──→ Workflow / Agent / 在线支撑验证器
```

**在线禁止访问**：仿真真实接触点、真实位姿、真实稳定标签、故障注入种类、等级、随机种子、离线虚拟释放测试结果。`inspect_object_pose` 必须使用带误差的感知结果，不能返回物理引擎直接状态。可将离线虚拟释放作为稳定性评价依据，但其观察窗及位置/姿态容差须事先固定，且不可反馈给在线系统。

---

<a id="section-7"></a>

## 7. 对照实验与消融实验

### 7.1 主实验 A/B/C/D

| 组别 | 方法 | 接管逻辑 | 必须保持不变的内容 |
|---|---|---|---|
| A | **强规则 Workflow** | 基于数据有效性与历史选择诊断工具，**不调用 Agent** | 全部工具、安全与验证器 |
| B | **Always-Agent** | 每次初始 UNKNOWN 立即调用 Agent | 与 C/D 相同 LLM、工具和预算 |
| C | **Fixed-Retry** | A 的 Workflow 先尝试固定 $N$ 次；仍 UNKNOWN 再调用 Agent | $N$ 在开发集固定，不用测试集选值 |
| D0 | **Evidence Gate** | 根据证据、诊断进展、可诊断性、预算启发式接管 | 明确属于规则版本 |
| D1 | **Utility-Guided（研究目标）** | 估计 Agent 相对 A 的条件净收益后接管 | 模型只能使用训练/开发数据 |

**用户提出的四组方案保持不变**：A/B/C/D 为主框架；D0 与 D1 只是 D 的两个阶段。A 必须是强基线，不能人为削弱到只能重复同一工具。B/C/D 的 Agent 使用相同模型、提示词结构和工具列表。给所有组相同总墙钟时间和工具调用上限；Agent 推理与 D1 预测器计算开销都计入成本。

对于 D1，课程首轮采用第 5.4.1 节的“**固定检查点、每个 Episode 最多一次接管选择**”协议；训练集必须覆盖该检查点的状态和历史。剩余诊断由已经选定的策略执行。后续多轮重新路由属于单独扩展，不能与首轮结果混称。

### 7.2 主消融设计

| 方法 | 变化 | 回答的问题 |
|---|---|---|
| D1-Full | 完整模型 | 参考 |
| D1-w/o-E | 收益预测器不输入证据质量、时效性、矛盾等特征 | 状态证据是否有独立贡献？ |
| D1-w/o-H | 收益预测器不输入工具历史和诊断进展 | 诊断历史是否有用？ |
| D1-w/o-Cost | 保留收益概率预测，决策中将时间/资源成本权重置零 | 成本项是否抑制低收益调用？ |
| D0（w/o learned gain） | 用解释性门控替代学习的收益估计 | 收益学习是否必要？ |
| Retry-only（参考） | 只按固定次数转交 | 仅靠恢复次数是否足够？ |

**非常重要**：`w/o-E` 仅删除“接管收益模型中的证据特征”，**不能关闭基本传感器有效性校验或安全监督**。`w/o-H` 也不能删除底层必须保存的工具执行 ID 与重复动作保护。删去输入后应在相同训练集上重新训练相应预测器，避免单纯喂零向量形成非公平对照。还需检查历史摘要、剩余预算是否间接泄露被删除特征。

### 7.3 证据 × 成本交叉消融

设计 $2\times2$ 因子：

| 组别 | 证据特征 E | 成本项 C |
|---|---|---|
| E0C0 | 无 | 无 |
| E1C0 | 有 | 无 |
| E0C1 | 无 | 有 |
| E1C1 | 有 | 有（Full） |

若用同一预注册效用指标 $J$ 评价，可计算交互作用：

$$
I_{EC}=(J_{11}-J_{01})-(J_{10}-J_{00}).
$$

完整模型最好并不等于有协同作用，需要报告交互差异及配对置信区间。额外可改变 Agent 延迟（例如模拟 0.5/2/5 秒）检验成本敏感性；这些仅是测试档位，并非实测模型耗时。

---

<a id="section-8"></a>

## 8. 评价指标：防止“永远停止”虚假优秀

### 8.1 结果分类

每次 Episode 的诊断终点：`SUPPORTED`、`UNSUPPORTED` 或 `UNRESOLVED`。另记录**实际工具执行安全违规**与**是否完成完整放置**，不能互相代替。

- 正确识别“未稳定支撑”：**诊断成功，但放置未成功**；
- 因信息不足安全停止：**安全处置，不属于正确诊断**；
- 工具执行成功：**不等于支撑状态确认成功**；
- 未经过真实张爪/撤回的纯诊断实验：**不得宣称已完成安全放置**。

### 8.2 建议的核心指标及公式

令 $N$ 为全部 Episode 数，$N_{\mathrm{decided}}$ 为系统给出明确二值支撑判断的次数：

| 维度 | 指标 | 计算与解释 |
|---|---|---|
| 诊断有效性 | **CDR 正确诊断率** | $N_{\mathrm{correct}}/N$；未解决计为不正确诊断 |
| 诊断覆盖 | **DC 覆盖率** | $N_{\mathrm{decided}}/N$；越高不一定越安全 |
| 已决策风险 | **SR 已决策错误率** | $N_{\mathrm{wrong}}/N_{\mathrm{decided}}$；分母为 0 时应报告 N/A |
| 危险判断 | **FSR 错误确认支撑率** | 真实未支撑中被错误判断为 SUPPORTED 的比例 |
| 接管效率 | **AER Agent 接管率** | 至少调用一次 Agent 的 Episode 比例 |
| 模型开销 | **AAC 平均 Agent 调用次数** | 所有 Episode 的推理请求总数 / $N$ |
| 诊断耗时 | **T_diag 平均/P95** | 从 UNKNOWN 触发到终止或诊断完成的实际墙钟耗时 |
| 限时有效性 | **CDR(τ)** | 截至 $\tau$ 秒正确完成诊断的 Episode 比例 |
| 工具成本 | **NTC** | 每个 Episode 的平均诊断技能调用数 |
| 资源成本 | **C_compute** | LLM + 收益预测器的实际资源开销（明示单位） |
| 安全 | **SVR** | 发生至少一次实际安全违规的 Episode 比例 |
| 决策安全 | **UAPR** | 出现不安全工具建议的 Episode 比例（与实际执行区分） |
| 过度保守 | **PSR** | 离线确认为可诊断的 Episode 中过早停止的比例 |
| 安全终止 | **ASR** | 离线确认为不可安全继续的 Episode 中正确停止的比例 |

对二值支撑真值且所有已决策结果都能判对错的数据集：

$$
\boxed{CDR=DC\,(1-SR)}.
$$

**“始终停止”策略**：$CDR=0$、$DC=0$、完整放置成功率为 0；其 SR 应为 N/A，而不是 0%。这样不会靠拒绝所有任务取得“完美诊断准确率”。还需单列 PSR（不必要停止）和不同场景下的 ASR（合理停止）。

### 8.3 完整闭环扩展指标

只有后续真正允许统一的受限物理恢复、释放与撤回，才加入：

$$
STSR=\frac{N_{\mathrm{stable\ placement+release+retreat+no\ violation}}}{N}.
$$

同时报告“可恢复 Episode 上的任务成功率”，前提是离线可恢复标签有可靠依据，并固定分母。禁止将纯诊断阶段的 CDR 伪装为 STSR。

### 8.4 如何综合比较而不随意加总分？

建议采用**安全约束 → 诊断有效性约束 → 比较成本**的顺序：

1. 任何方法都必须遵守共同的释放与运动安全规则，记录实际危险请求和违规；
2. 在预先定义的 CDR、DC 下限及 FSR 风险要求下，比较不同方法的时间/计算成本；
3. 同时绘制 `CDR(τ)`、`CDR–AER` 和 `Coverage–Risk` 曲线；
4. 对“只会停止”的方法，CDR/DC/完整任务成功均为零，无法因零实际违规而获得优势。

所有约束阈值应在测试前固定；有限样本中未观察到安全违规，不等于真实风险为零。风险—覆盖率联合分析的相关理论背景见 [12]。

PSR/ASR 的离线分母尤其要审慎：找到一条在统一预算下可靠成功的诊断序列，可以构成“已证实可诊断”的正例；尚未找到序列不能证明“不可能诊断”。标签证据不足的 Episode 应标记 `UNKNOWN_FEASIBILITY` 并单列，不强行归入不可恢复。所有已决策正确性使用独立评价真值，而非把在线验证器的判断当成真值。

统计“总诊断时间”时保留提前停止样本，同时以 `CDR(τ)` 区分迅速放弃和迅速解决。零危险事件但 CDR/DC 为零的策略，只能被称为停止基线，不能被称为优秀自主恢复方法。

---

<a id="section-9"></a>

## 9. 数据划分、复现与统计分析计划

### 9.1 先完成可行性验证

建议初期按 $6$ 场景 × $3$ 难度 × 每组合 $5$ 个不同随机种子生成：

$$
6\times3\times5=90\text{ 个初始 Episode}.
$$

A/B/C/D 各运行一次，约 $360$ 次策略执行（不含 LLM 随机重复）。用于验证能否稳定生成真实 `UNKNOWN`、工具接口能否运行、状态能否重置和真值是否隔离。**这些数量仅为工程可行性建议，不保证统计显著性。**

### 9.2 正式训练/开发/测试

初步建议每个“场景 × 难度”组合：训练 10–15、开发 5、测试 10 个独立初始状态；并额外设置未见扰动组合或更强度的独立测试集。测试至少 $6\times3\times10=180$ 个 Episode；四组约 $720$ 次执行。重复的 Agent 随机运行须按初始 Episode 分组分析。

- **训练集**：配对运行 W 与 A，形成增量收益标签，训练 D1；
- **开发集**：固定 C 的重试次数、D 的门槛、成本权重和停止预算；
- **测试集**：冻结算法后正式评价，**禁止用测试结果再调参**。

同一物理状态及其所有派生执行分支必须属于同一数据划分。若 D1 面向中间诊断状态接管，训练数据也必须包含相应中间状态，不能只用初始状态的收益标签代替。对每个场景需报告生成 `UNKNOWN` 的接受比例，防止筛选偏差。

### 9.3 统计方法

- 正确诊断率、未解决率等二值指标报告比例和 95% 置信区间；
- 同一初始 Episode 上的方法差异使用配对比较（例如二值结局的 McNemar 检验）；
- 时间、调用成本使用按 Episode 分组的配对 Bootstrap 置信区间；
- 报告 S1–S6、L1–L3 的分场景结果，不只汇总平均值；
- 讨论效果大小和置信区间，而不只汇报 $p$ 值；
- 对未见组合与不同 Agent 延迟单独报告泛化/敏感性结果。

---

<a id="section-10"></a>

## 10. 研究假设与可证伪标准

| 假设 | 需检验的命题 | 可用于反驳的结果 |
|---|---|---|
| **H1：状态依赖收益** | Agent 相对 Workflow 的增量诊断价值随证据状态而变化 | 各状态差异很小，或 Agent 始终不优于 Workflow |
| **H2：证据价值** | 证据质量/矛盾特征有助于收益预测及接管 | D1-w/o-E 与 Full 预测与决策效果相同 |
| **H3：历史价值** | 诊断历史减少重复无效检查 | 去掉历史后无差异，或 Full 反而更差 |
| **H4：成本价值** | 随 Agent 延迟和计算成本变化，成本敏感决策有更好的效用权衡 | Full 与 w/o-Cost 无差异或效用更低 |
| **H5：强规则可能足够** | 强规则 Workflow 或简单重试可能达到相同诊断效果且成本更低 | 若数据支持此项，应承认不必使用复杂 Agent |

此外需要独立验证：安全监督器拒绝不合法工具请求；物理状态不明时不授权释放；工具超时不盲目重复物理动作。这些**安全验证是系统基本要求，不应算成 D1 相对 A/B/C 的独有创新点**。

---

<a id="section-11"></a>

## 11. 实验结果与讨论（待实际运行后填写）

> 本节为记录模板。报告当前不包含实测数据，所有占位符均以“待测”表示，**不得将示例数字写成真实结果**。

### 11.1 主对照结果表

| 方法 | CDR ↑ | DC ↑ | FSR ↓ | AER ↓* | 平均 T_diag ↓* | P95 T_diag ↓* | 平均计算成本 ↓* | SVR ↓ |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| A 强规则 Workflow | 待测 | 待测 | 待测 | 0%（定义） | 待测 | 待测 | 待测 | 待测 |
| B UNKNOWN 立即 Agent | 待测 | 待测 | 待测 | 100%（定义） | 待测 | 待测 | 待测 | 待测 |
| C 固定次数接管 | 待测 | 待测 | 待测 | 待测 | 待测 | 待测 | 待测 | 待测 |
| D0 证据门控 | 待测 | 待测 | 待测 | 待测 | 待测 | 待测 | 待测 | 待测 |
| D1 预计收益接管 | 待测 | 待测 | 待测 | 待测 | 待测 | 待测 | 待测 | 待测 |

\* AER、时间和成本不能脱离正确诊断率与安全性单独判断“越低越好”。B 的 100% 指所有合法进入本研究入口的 Episode 都立即请求 Agent；若有共同安全机制在入口直接阻断，则须按实际分析口径另报。

### 11.2 消融结果表

| 方法 | CDR | AER | T_diag | 预测净收益 MAE | 未见异常组合表现 |
|---|---|---|---|---|---|
| D1-Full | 待测 | 待测 | 待测 | 待测 | 待测 |
| D1-w/o-E | 待测 | 待测 | 待测 | 待测 | 待测 |
| D1-w/o-H | 待测 | 待测 | 待测 | 待测 | 待测 |
| D1-w/o-Cost | 待测 | 待测 | 待测 | 待测 | 待测 |
| D0 / w/o learned gain | 待测 | 待测 | 待测 | 不适用 | 待测 |

### 11.3 必须如实回答的结果解释问题

1. D 是否在**相近 CDR/DC 和安全约束**下减少了 Agent 调用或总成本？
2. D 的收益是来自证据信息、诊断历史还是成本权重？消融和接管日志能否追踪？
3. D 相对于强规则 A 的额外收益是否足以覆盖收益预测器的训练与维护成本？
4. 如果物理状态本身无法由当前传感器辨别，Agent 有没有被错误地宣传为能够“恢复”？
5. 若 D 不优于 A/C，是否应简化系统、调整问题设置或停止在此场景引入 Agent？

**预先允许的消极结论**：在当前固定放置任务、有限工具集合与预算下，强规则 Workflow 已经足够；Agent 的条件增量收益不足以抵消推理与维护成本。这同样是一项有效课程研究结论。

---

<a id="section-12"></a>

## 12. 课程知识、方法贡献与局限性总结

### 12.1 已有知识在本研究中的具体落地

- Workflow 与 Agent 的分工：正常执行和明确诊断走规则，开放式诊断工具选择在必要时交给 Agent。
- 工具调用与反馈循环：使用结构化参数、工具结果、诊断历史与版本化状态。
- 分布式系统可靠性：采集时间戳、请求 ID、执行状态查询、超时与重复调用保护。
- 海洋数据挖掘：研究水下视觉、接触观测、时效性和历史证据如何影响决策收益。
- 安全与任务分离：高层自主决策不得绕过独立释放许可与运行时保护。

### 12.2 本研究**提出但尚未验证**的具体改进

1. **针对 `SUPPORT_UNKNOWN` 的结构化证据表示**：把证据有效性、时效性、物理语义一致性与诊断历史区分开；
2. **面向 Agent 条件增量收益的接管选择**：不把“异常严重”直接等同于“值得调用”，而是比较 W/A 的诊断效果与额外成本；
3. **面向该问题的配对仿真协议与消融**：同一状态、相同工具与安全规则、冻结测试数据，分析改进究竟来自哪个模块。

这些增量是否足以支撑方法论文，需进一步与成本敏感求助、主动感知和机器人恢复研究比较；**不能仅凭本报告直接声称新算法首创**。

### 12.3 局限与风险

- 没有高质量、相互补充的诊断工具时，Agent 无法创造新的物理证据；
- 规则 Workflow 可能已经能解决主要异常，D1 增益可能很小或为负；
- 以仿真得到的条件收益训练模型，可能无法直接泛化到真实水下传感器噪声和动力学；
- 观测冲突可能来自物理语义不同，而不是真实传感器错误；
- 少量配对实验估计的增量收益方差较大，预测区间不自动等于安全保证；
- 共享安全监督器可以使实际违规事件稀少，因此还必须测量危险建议、过度保守和诊断覆盖；
- 如果第一阶段只允许诊断工具，则研究结论不能扩展成“机械臂完成放置恢复”的证明。

---

<a id="section-13"></a>

## 13. 总结与下一步实施顺序

本研究从课程中的 Workflow、Agent 与分布式可靠性知识出发，以水下机械臂真实存在的放置确认环节为对象，经过“异常触发是否必要—状态机能否替代—已有研究是否先行—证据是否可信—调用是否值得—执行是否安全”的连续质疑，将宽泛的机器人智能体设想收缩为一个可证伪的问题：**同一个支撑状态 UNKNOWN 下，Agent 什么时候能相对强规则 Workflow 提供正的诊断净收益？**

拟采用以下实施顺序：

1. 核实现有仿真平台的视觉、接触/载荷观测接口和状态保存能力；先构建统一的 `UNKNOWN` 触发器及独立离线真值评价。
2. 构造 S1–S6 分级 Episode 和可重放随机种子，完成真值隔离检查。
3. 固定工具、安全监督和预算，运行 A/B/C/D0，判断 Agent 是否确实存在可利用的状态依赖性优势。
4. 只有在配对实验显示有合理增量空间后，采集 W/A 成对收益数据、训练 D1，冻结阈值与成本权重。
5. 在独立测试集进行主对照、单因素消融、证据×成本交叉消融和未见组合泛化实验。
6. 按 CDR/DC/FSR、耗时/调用成本、SVR/UAPR/PSR 等指标报告实测结论；不因结果未支持 D1 而改变评价口径。

**本报告的核心研究立场**：Agent 不是越多越好。与其预设“异常应由大模型接管”，不如用可复现、可审计的证据检验“何时值得接管”，并保留“不需要 Agent”这一同样成立的结论。

---

<a id="references"></a>

## 参考文献与可核查链接

> 2026 年 11 月 9 日复核：以下主要题名、年份和方法概述可在所列出版社、正式论文集、作者机构或作者预印本入口核查。此处不是穷尽式综述，也不将摘要中的定量结论当作跨论文可比结果。网页可访问性可能变化；IEEE 等入口可能需要机构访问权限。

[1] Anthropic. **Building Effective Agents**. 2024. [官方工程文章](https://www.anthropic.com/research/building-effective-agents).

[2] Tang, R. et al. **HeRo: A State Machine-Based, Fault-Tolerant Framework for Heterogeneous Multi-Robot Collaboration**. *IEEE ICRA*, 2025. DOI: 10.1109/ICRA55743.2025.11128674. [IEEE](https://ieeexplore.ieee.org/abstract/document/11128674).

[3] Styrud, J. et al. **Automatic Behavior Tree Expansion with LLMs for Robotic Manipulation**. *IEEE ICRA*, 2025. [arXiv](https://arxiv.org/abs/2409.13356) · [代码与论文说明](https://github.com/jstyrud/BETR-XP-LLM).

[4] Ahmad, F. et al. **A Unified Framework for Real-Time Failure Handling in Robotics Using Vision-Language Models, Reactive Planner and Behavior Trees**. *IEEE CASE*, 2025. DOI: 10.1109/CASE58245.2025.11164021. [IEEE](https://ieeexplore.ieee.org/abstract/document/11164021) · [arXiv](https://arxiv.org/abs/2503.15202).

[5] Agia, C. et al. **Unpacking Failure Modes of Generative Policies: Runtime Monitoring of Consistency and Progress (Sentinel)**. *CoRL 2024*, proceedings in *PMLR*, Vol. 270, 2025. [PMLR](https://proceedings.mlr.press/v270/agia25a.html).

[6] Vats, S. et al. **RecoveryChaining: Learning Local Recovery Policies for Robust Manipulation**. *IEEE IROS*, 2025. DOI: 10.1109/IROS60139.2025.11245856. [IEEE](https://ieeexplore.ieee.org/abstract/document/11245856) · [作者机构全文入口](https://www.merl.com/publications/TR2025-152).

[7] Xu, S. et al. **From Reaction to Anticipation: Proactive Failure Recovery through Agentic Task Graph for Robotic Manipulation (AgentChord)**. *RSS*, 2026. DOI: 10.15607/RSS.2026.XXII.180. [RSS 论文集](https://www.roboticsproceedings.org/rss22/p180.html) · [arXiv](https://arxiv.org/abs/2605.11951).

[8] Banerjee, R. et al. **A Human-in-the-Loop Confidence-Aware Failure Recovery Framework for Modular Robot Policies**. *ACM/IEEE HRI*, 2026. DOI: 10.1145/3757279.3788668. [项目主页](https://emprise.cs.cornell.edu/modularhil/) · [arXiv](https://arxiv.org/abs/2602.10289).

[9] Ota, K. et al. **Tactile Estimation of Extrinsic Contact Patch for Stable Placement**. *IEEE ICRA*, 2024. DOI: 10.1109/ICRA57147.2024.10611504. [MERL](https://www.merl.com/publications/TR2024-053).

[10] Ghoummaid, M., Shalit, U. **When to Act and When to Ask: Policy Learning With Deferral Under Hidden Confounding**. *NeurIPS*, 2024. [NeurIPS 论文](https://proceedings.neurips.cc/paper_files/paper/2024/hash/660cf2a1eabe448920a3ab6754555adb-Abstract-Conference.html).

[11] Open Robotics. **Actions — ROS 2 Design**. 官方接口设计文档，2019，后续修订。用于目标标识、执行反馈、结果和取消语义；非新近科研论文。[官方文档](https://design.ros2.org/articles/actions.html).

[12] El-Yaniv, R., Wiener, Y. **On the Foundations of Noise-free Selective Classification**. *Journal of Machine Learning Research*, 11:1605–1641, 2010。用于风险—覆盖率背景，明确不属于近三年文献。[JMLR](https://www.jmlr.org/papers/v11/el-yaniv10a.html).


---

## 附录 A：建议的单次 Episode 配置（示例）

```yaml
episode_id: S4_L2_0007
phase: SUPPORT_VERIFICATION
trigger: SUPPORT_UNKNOWN
motion_completed: true
gripper_closed: true
scenario_for_evaluator_only: S4
difficulty_for_evaluator_only: L2
seeds:
  physics: 1024
  visual: 2048
  contact: 3072
  agent: 4096
perturbation_config_for_simulator_only:
  visual_gap_bias_sigma: 2.0
  contact_bias_sigma: 2.5
  sensor_delay_ms: 100
budget:
  max_wall_time_s: 15    # 初始建议，开发集标定
  max_tool_calls: 3
  max_agent_decisions: 2
permissions:
  agent_direct_joint_control: false
  agent_direct_release: false
  bounded_physical_recovery: false
```

**注意**：`scenario_for_evaluator_only`、`difficulty_for_evaluator_only`、种子与扰动参数绝不能包含在给 Agent 的在线请求中。

## 附录 B：正式实验前的检查清单

- [ ] 当前任务确实只从“下降完成 + 夹爪闭合 + 支撑 UNKNOWN”进入。
- [ ] 真实接触状态不会通过任何工具泄露给 Agent 或 Workflow。
- [ ] 不把几何接触、载荷转移、稳定支撑混为一谈。
- [ ] 工具的超时、重复请求与执行状态未知均可正确处理。
- [ ] 所有物理动作都通过独立安全监督，释放许可不能由 Agent 自行修改。
- [ ] A 是强规则 Workflow，四组使用同样工具与预算。
- [ ] C 的重试次数在开发集固定；D1 不使用测试集训练或调参。
- [ ] 配对 Episode、传感器噪声、种子和 LLM 输出可保存复查。
- [ ] 完成 CDR/DC/SR/FSR、时间/成本及安全指标的自动统计。
- [ ] 永远停止的方法 CDR/DC/STSR 记零；已决策错误率 SR 记 N/A。
- [ ] 未执行实验不写成已完成成果，不把假设数字写成真实结果。


## 附录 C：AI 讨论记录与提交状态

本报告源于多轮讨论。讨论中先提出混合架构，再主动追问状态机替代性、已有研究重合、规则堆叠、证据矛盾与接管收益的区别，最后形成对照、泛化、评价与消融方案。该过程的整理属于研究记录，不等于已执行实验。

- 本次聊天分享链接：[用户提供的本次课程讨论](https://chatgpt.com/share/6af1d7d9-e370-83ee-8f11-7b318c0ef676)。
- 原始逐条聊天 Markdown：当前包中未附；不能将精简研究历程标记为完整逐字导出。
- 公开 GitHub 仓库：[https://github.com/AirYy03/marine-data-mining-agent-escalation](https://github.com/AirYy03/marine-data-mining-agent-escalation)，仅用于本次课程作业。
- 研究过程索引：项目包内 `docs/research_process.md`，只作阅读导航。
- 实验数据与可运行研究代码：当前未提供，当前包不包含其他研究项目代码。

已收录用户提供的真实分享入口；访问与覆盖范围以该链接实际显示的内容为准。若教师要求完整导出，分享链接不能自动代替完整 Markdown 导出。公开内容限本次“海洋数据挖掘”课程作业，不要求、也不授权公开其他项目。

## 附录 D：本次格式与口径修订

1. 单一文档主标题，增加可点击目录与稳定锚点。
2. 数学表达改为 GitHub 支持的 `$...$` / `$$...$$`，移除不兼容的反斜杠外层分隔写法。
3. 保持 UTF-8 文本，检查代码围栏闭合、表格列数、JSON/YAML 示例与文内引用。
4. 区分 `UNSUPPORTED` 的正确诊断与 `UNRESOLVED` 的未知停止，避免伪代码中的标签混淆。
5. 明确 D1 的单次选择检查点、后续策略时域、预算与配对训练标签，避免模型与评估定义错位。
6. 增加执行前新状态审核、失败调用成本计入、动态仿真时间、可诊断标签证据及原始聊天缺项说明。
7. 当前所有结果仍为“待测”；没有依据本次重排宣称算法有效或实验已经完成。

8. v2.1 补入用户提供的真实聊天分享链接及独立公开仓库地址；未修改实验待测状态，未加入其他项目代码。
