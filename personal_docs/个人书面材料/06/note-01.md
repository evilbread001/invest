# 售后故障诊断 Agent 架构方案（重设计）

> 一份"如果从零做"的技术方案。用于团队技术评审与架构沉淀。
> 作者：gerui1 · 日期：2026-08-13

---

## 0 · TL;DR

工业售后故障诊断这类问题**不适合单框架方案**，应该分三层：

1. **语言层**：LLM（Claude / GPT / 本地 Qwen）负责 NLU + NLG
2. **编排层**：LangGraph 负责流程、HITL、状态持久化、审计
3. **推理层**：规则引擎（故障树）主 + LLM RAG 兜底

单选一个"agent 框架"（LangGraph / AutoGen / CrewAI）想解决全部问题，是这类项目普遍平庸的根因。

---

## 1 · 从问题形状出发（不是从框架出发）

选型之前先做三个判断：

### 判断 1：领域知识可不可编码？

工业故障诊断在多数行业里是**成熟领域**——FTA / FMEA / 8D 是几十年的方法论积累。老师傅的诊断链路 **80% 可编码**为故障树。

- 可编码占比高 → **规则引擎/故障树是主武器**，LLM 是辅助
- 可编码占比低 → LLM 推理为主，规则为辅

结论：本场景**规则为主**是对的。不该被"没有 LLM 就不是 AI agent"的舆论带偏。

### 判断 2：诊断错了代价大不大？

售后是**责任场景**——错判要赔钱、要派人、要召回。这决定两件硬需求：

- **可审计**：每一步为什么这么判要能回放
- **可人工卡点**：高风险动作（不可逆处置、退换货、上门）前必须 HITL

结论：编排层必须原生支持 checkpoint 和 interrupt。

### 判断 3：状态生命周期多长？

售后单从提交到闭环可能横跨**数天**（等技工到现场、等备件、等客户配合）。这决定：

- **持久化**：状态不能只在内存
- **断点续跑**：不能重头来一遍
- **局部重跑**：技工重测某一项后，只重跑相关分支

结论：编排层必须支持长事务 + 状态回放。

---

## 2 · 三层架构

```
                       ┌──────────────────────────┐
                       │   触发层（TableTrigger）  │
                       │  Postgres 轮询工单表      │
                       └──────────────┬───────────┘
                                      │
     ┌────────────────────────────────▼────────────────────────────────┐
     │  Layer 2  编排层  ·  LangGraph StateGraph                        │
     │                                                                  │
     │   ┌────────────┐    ┌────────────┐    ┌────────────┐            │
     │   │ perception │───▶│  classify  │───▶│  diagnose  │───▶ ...    │
     │   └────────────┘    └─────┬──────┘    └─────┬──────┘            │
     │                           │                  │                    │
     │        checkpoint / interrupt / audit_log 一等公民                │
     └───────────┬──────────────────────┬───────────────────┬───────────┘
                 │                      │                    │
        ┌────────▼────────┐   ┌────────▼────────┐  ┌───────▼─────────┐
        │ Layer 1 语言层  │   │  Layer 3 推理层  │  │  Layer 1 语言层 │
        │                 │   │                  │  │                 │
        │  LLM 结构化输出 │   │  故障树（主）    │  │  LLM 生成       │
        │  cause_type     │   │  ↓ 兜底          │  │  8D 报告        │
        │  分类           │   │  LLM RAG 历史工单│  │                 │
        └─────────────────┘   └──────────────────┘  └─────────────────┘
```

### Layer 1 · 语言层

**职责**：所有涉及自然语言理解和生成的地方。

**落点**（这类系统里的三处刚需）：

| 位置 | 输入 | 输出 | 为什么必须 LLM |
|---|---|---|---|
| 报障文本 → cause_type 分类 | "一按启动就跳，还冒烟" | `{cause_type: "过流保护", conf: 0.87}` | 用户不会用规范术语 |
| 故障树叶节点追问话术 | 当前证据 + 缺失字段 | "麻烦拍一下电机铭牌照片，我需要额定电流值" | 静态模板问不到点子上 |
| 诊断结果 → 8D 报告 | root_causes + diagnosis_paths | 完整 8D 报告文字 | 拼接模板质量差 |

**选型**：
- 云端优先：Claude Sonnet / GPT-4-class
- 有合规要求：Qwen / DeepSeek 本地部署
- 硬性要求：**结构化输出**（tool use / JSON mode）。这层的下游是规则引擎，schema 不能飘

**反模式**：让 LLM 直接输出诊断结论。LLM 只做"翻译"（口语 ↔ 结构化）和"话术生成"，不做判定。

### Layer 2 · 编排层

**职责**：把 Layer 1 和 Layer 3 串成业务流程，管住状态、HITL、审计。

**核心节点**（对照现有 BPMN 4 步，逻辑上等价）：

```
START → perception → classify → diagnose → report → END
                        ↓          ↓          ↓
                    可能 WAITING   ↓      可能 WAITING
                       (追问)  可能 WAITING  (客户确认)
                                (等技工回单)
```

**LangGraph 的价值兑现在这几个点**：

1. **State schema 显式**（TypedDict）—— 对齐现有 `wf.data["context"]`，字段可平移
2. **conditional_edges 由代码决定**—— 高风险动作、审计合规这些边**不给 LLM 决定**
3. **`interrupt()` 一等公民**—— 替代现在"抛 `RequestUserInput` 异常 → 引擎 catch → status=WAITING → cancel" 的绕圈写法
4. **Checkpoint 天然对齐长事务**—— `SqliteSaver`/`PostgresSaver`，thread_id = session_id
5. **time-travel 支持局部重跑**—— 对应现有 `OWLRecoveryEngine` 的 DFS 局部重跑，但不用自己造

**为什么不选其它编排框架**：

| 框架 | 不选的原因 |
|---|---|
| AutoGen | 多 agent 对话式协商——售后是流程化的，不是"多位专家吵一架"。收敛性差、token 贵 |
| CrewAI | Role-based 抽象太浅，HITL / 长事务 / 审计全要自己 hack |
| BPMN + SpiffWorkflow | 现有方案。能用，但 HITL 靠异常控制流，没原生 LLM 集成；未来接 LLM 麻烦 |
| 自研 | checkpoint / interrupt / time-travel 自己写要造半年轮子，不划算 |

### Layer 3 · 推理层

**职责**：从"症状 + 证据"推理出"根因 + 建议动作"。

**架构**：**规则引擎主 + LLM RAG 兜底**，三层降级。

```
    用户症状 + Layer 1 结构化字段
             │
             ▼
    ┌─────────────────┐
    │  故障树（主）    │  ← 覆盖高频故障（80%）
    │  OWL / DSL /    │     置信度 ≥ 阈值 → 输出根因
    │  Drools 都行    │     置信度 < 阈值 → 走兜底
    └────────┬────────┘
             │  低置信 or 无匹配
             ▼
    ┌─────────────────┐
    │ LLM RAG 兜底     │  ← 覆盖长尾故障（15%）
    │ 检索历史工单 +  │     置信度 ≥ 阈值 → 类比推理输出根因
    │ 设备手册        │     置信度 < 阈值 → 转人工
    └────────┬────────┘
             │  低置信 or 拒答
             ▼
    ┌─────────────────┐
    │  转人工（5%）   │  ← 高价值信号：这条应该被沉淀为新规则
    └─────────────────┘
```

**为什么不纯 LLM**：售后错判成本是真金白银，LLM 幻觉的代价太高。规则引擎在高频故障上是**成本/风险最优**。

**为什么不纯规则**：长尾永远存在，KB 里没有的怪问题需要 LLM 类比历史工单。

**关键工程点**：
- 三层之间的**置信度阈值**要单独可配（不同故障类别阈值不同）
- 转人工不是终点，是**反馈信号**——每一单人工介入都要沉淀为新规则的候选
- 规则引擎技术选型开放：现有 OWL、开源 Drools/Nools、甚至纯 Python DSL 都行。**这层的技术选型不重要，重要的是接口**：`diagnose(state) -> (root_causes, confidence, evidence)`

---

## 3 · State schema（关键字段）

```python
class DiagnosisState(TypedDict, total=False):
    # 触发
    session_id: str                       # 对齐现有 workflow_id
    ticket_id: str
    source_event: dict                    # 原始工单

    # Layer 1 产出
    cause_type: str                       # LLM 分类结果
    cause_confidence: float
    extracted_symptoms: list[dict]        # LLM 抽取的症状

    # Layer 3 产出
    diagnosis_paths: list[str]            # 故障树走过的路径
    root_causes: list[dict]               # [{cause, confidence, source: "rule"|"rag"|"human"}]
    top_hypothesis: dict

    # 控制字段（代码写死路由用，不给 LLM 决定）
    needs_human_approval: bool            # 高风险动作触发
    fallback_to_llm: bool                 # 规则引擎低置信触发
    escalate_to_human: bool               # LLM 兜底也低置信触发

    # Layer 1 产出
    report_8d: str                        # LLM 生成的 8D 报告

    # 审计
    audit_log: Annotated[list[dict], operator.add]  # 每步追加
    llm_calls: Annotated[list[dict], operator.add]  # LLM 调用记录（成本/延迟）
```

---

## 4 · Trade-off 与已知局限

| 决策 | 好处 | 代价 |
|---|---|---|
| 分三层 | 每层做擅长的事，可独立演进 | 复杂度比"一个框架搞定"高一档 |
| 规则引擎主 | 高频故障可控、可审计 | 每个新故障树都要人工建模 |
| LLM 兜底 | 覆盖长尾、无需穷举建模 | 需要历史工单积累到一定规模才有效 |
| LangGraph 编排 | HITL/checkpoint 原生 | LangGraph 生态相对新，中文社区弱 |
| 结构化输出强制 | 下游可解析 | LLM 偶发不遵守 schema，需重试逻辑 |

**明确不做的事**：
- 不把 LLM 放在路由决策上——所有 conditional_edges 由代码决定
- 不搞多 agent 协商——单主流程 + 子工具就够
- 不追求"agent 自主学习"——沉淀新规则由人工审阅

---

## 5 · 迁移路径（对现有 BPMN + OWL 系统）

如果要把现有生产系统演进到这套架构，**不整体重写**。分四步：

**Phase 1（低风险，1-2 周）**：在现有 BPMN 里嵌入 LLM 节点
- `nlp分类` 从"读字段"改成"真调 LLM"，保留兜底
- 8D 报告加一个 LLM 润色环节
- 目的：验证 LLM 在业务场景的可用性和成本

**Phase 2（中等风险，2-4 周）**：单个故障树接入 LLM RAG 兜底
- 挑一个长尾故障多的类别（比如少锡）
- 故障树走完低置信度 → LLM RAG over 历史工单
- 目的：验证三层降级机制

**Phase 3（较高风险，4-8 周）**：新增业务用 LangGraph，老业务保留 BPMN
- 新故障类型直接用 LangGraph 建图
- 老 BPMN 保持不动
- 目的：新老共存，团队渐进熟悉 LangGraph

**Phase 4（可选，长期）**：老业务迁移或不迁
- 如果 LangGraph 版稳定运行 3-6 个月，考虑把老 BPMN 也迁过来
- 如果老 BPMN 一直稳定，**不迁也 OK**——架构演进不追求"技术栈统一"这种虚荣指标

---

## 6 · 对现有系统的批判性观察

**做得好的地方**：
- OWL 故障树是这个领域的正确抽象（AND/OR/权重/多分支）
- 状态持久化 + 快照回放的设计是长事务系统必须的
- 审计日志（`execution_history`）从一开始就做，为售后合规打下基础

**可以改进的地方**：
- `nlp分类` 名不副实——实际是字段读取，不是 NLP。这是最容易接入 LLM 的一处
- HITL 靠抛异常 + 引擎 catch + cancel，控制流绕。有原生 `interrupt` 后可以直接扁平化
- 双层持久化（BPMN 层 + OWL 层各自快照）复杂度高，如果统一到 LangGraph checkpointer 可以少一层
- 没有 LLM 兜底路径，长尾故障只能转人工——这是最可惜的一点

---

## 7 · 一句话总结

> **售后故障诊断 agent 不是一个"用什么 agent 框架"的问题，而是一个"三层职责怎么分"的问题。**
>
> 语言层用 LLM，编排层用 LangGraph，推理层用规则引擎 + LLM 兜底。
> 每一层都为自己擅长的事负责，谁也别越界。
>
> 现有 BPMN + OWL 系统在推理层的方向是对的，缺的是语言层，
> 而编排层是否要换 LangGraph 应该由具体痛点驱动，不是技术洁癖。

---

## 附录 · 常见技术评审追问 & 回答骨架

**Q: 为什么不整体用 LangGraph 重写？**
> 现有 BPMN + OWL 在推理层的表达能力其实比 LangGraph 边模型更强（权重、Serial/Parallel、DFS 重跑）。硬迁是降级。分层的价值就是让每层做擅长的事。

**Q: LangGraph 相比 BPMN 的核心优势是什么？**
> 三件：`interrupt` 是一等公民（不用异常控制流做 HITL）、checkpoint 天然对齐长事务、对 LLM 节点亲和。但成熟度和稳定性 BPMN 更好。是否切换要看痛点是否够强。

**Q: LLM 幻觉怎么办？**
> 分两层防：一是**结构化输出**强制 schema，schema 校验失败自动重试或降级；二是**LLM 不做判定**——分类结果、兜底诊断都要落到 State 里作为"证据"，最终决策由代码逻辑（阈值、路由函数）做出。LLM 是"翻译官"和"检索员"，不是"法官"。

**Q: 规则引擎和 LLM 怎么协作？**
> 三层降级：规则主 → LLM 兜底 → 人工。每层有置信度阈值。人工介入的 case 沉淀为新规则候选，闭环回路完整。

**Q: 状态怎么持久化？**
> LangGraph `PostgresSaver`，thread_id = session_id。所有 checkpoint 落库，支持任意点回放 + 局部重跑。审计合规是一等公民不是附加需求。

**Q: 团队要学习成本吗？**
> 有。LangGraph 的图 + State + reducer 抽象对做过后端的人不难，一周能上手；难的是**理解什么该给 LLM、什么该写死**，这个是经验，不是文档。团队推广建议先做一个纵切片（一个故障类别），跑稳了再横向铺开。
