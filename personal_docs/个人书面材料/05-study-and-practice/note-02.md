# 故障树生成质量评估方案

## 一、背景与目标

### 1.1 业务背景

我们使用大语言模型（LLM Agent）自动生成故障树（OWL格式），作为售后诊断工作流的推理依据。需要构建一套系统化的评估体系，衡量生成故障树的质量，并通过 Bad Case 分析引导持续优化。

### 1.2 评估框架

遵循 Benchmark 评估的标准范式：

```
input（8D报告 + 会议纪要）
     │
     ▼
predict = Agent(input)     ← Agent 是黑盒
     │
     ▼
metrics = eval(predict, truth)   ← 定义指标体系
     │
     ▼
bad case analysis → 优化方向
```

其中：
- **input**：8D报告 + 质量专家交流的会议纪要
- **predict**：Agent 生成的 OWL 故障树
- **truth**：专家人工构建的标准故障树（当前 3 棵）
- **metrics**：本方案定义的 5 层 8 个指标
- **bad case analysis**：自动分类、计数、给出优化建议

### 1.3 参考资料

| 来源 | 核心观点 |
|---|---|
| Anthropic《Demystifying Evals for AI Agents》 | 三层评分器（代码/模型/人工）、pass@k 指标、从真实失败案例起步 |
| 42章经播客《聊透 Benchmark 与 Evaluation》 | Eval 是 AI 下半场核心竞争力，定义问题比解决问题更重要 |
| DeepEval 开源框架 | Python 生态、pytest 风格、内置 G-Eval（LLM-as-Judge）、支持自定义 Metric |

---

## 二、故障树结构说明

### 2.1 OWL 格式概述

故障树以 OWL（Web Ontology Language）格式存储，所有信息编码在 RDF/XML 结构中。

#### 2.1.1 Schema 定义（类与属性）

OWL 文件首先定义了本体的 Schema：

**类（Class）**：
- `DiagnosticNode`：唯一的节点类型，所有故障现象、诊断步骤、根本原因均为该类的实例

**对象属性（ObjectProperty）**：
- `next_nodes`：节点间的父子关系（边），domain 和 range 均为 `DiagnosticNode`

**数据属性（DatatypeProperty）**：

| 属性名 | 数据类型 | 说明 |
|---|---|---|
| `conditions` | string（JSON） | 边上的路由条件，包含诊断动作、期望值、权重等 |
| `diagnosis_function_name` | string | 诊断函数名称（可选） |
| `is_multi_next` | boolean | 是否可同时展开多个子节点 |
| `is_backtrack` | boolean | 是否支持回溯 |
| `measure_ids` | string | 关联的测量项 ID（可选） |

OWL Schema 示例：
```xml
<owl:Class rdf:about="#DiagnosticNode">
  <rdfs:subClassOf rdf:resource="http://www.w3.org/2002/07/owl#Thing"/>
</owl:Class>

<owl:ObjectProperty rdf:about="#next_nodes">
  <rdfs:domain rdf:resource="#DiagnosticNode"/>
  <rdfs:range rdf:resource="#DiagnosticNode"/>
</owl:ObjectProperty>

<owl:DatatypeProperty rdf:about="#conditions">
  <rdfs:domain rdf:resource="#DiagnosticNode"/>
  <rdfs:range rdf:resource="http://www.w3.org/2001/XMLSchema#string"/>
</owl:DatatypeProperty>

<owl:DatatypeProperty rdf:about="#is_multi_next">
  <rdfs:domain rdf:resource="#DiagnosticNode"/>
  <rdfs:range rdf:resource="http://www.w3.org/2001/XMLSchema#boolean"/>
</owl:DatatypeProperty>
```

#### 2.1.2 节点定义（NamedIndividual）

每个诊断节点是一个 `NamedIndividual`，通过 `rdf:about` 指定节点名称（中文），通过 `next_nodes` 声明子节点关系，通过 `conditions` 携带边上的路由条件。

**叶子节点**（无子节点）：
```xml
<owl:NamedIndividual rdf:about="#检具不合规">
  <rdf:type rdf:resource="#DiagnosticNode"/>
</owl:NamedIndividual>
```

**中间/根节点**（有子节点 + 路由条件）：
```xml
<owl:NamedIndividual rdf:about="#匹配不良">
  <rdf:type rdf:resource="#DiagnosticNode"/>
  <next_nodes rdf:resource="#匹配超差"/>
  <next_nodes rdf:resource="#检具不合规"/>
  <next_nodes rdf:resource="#尺寸合规"/>
  <conditions>{"next_node_name": "匹配超差", "test_names": [...], "weight": 0.99}</conditions>
  <conditions>{"next_node_name": "检具不合规", "test_names": [...], "weight": 0.99}</conditions>
  <is_multi_next rdf:datatype="http://www.w3.org/2001/XMLSchema#boolean">true</is_multi_next>
</owl:NamedIndividual>
```

#### 2.1.3 路由条件（conditions）详解

`conditions` 是挂在边上的 JSON 字符串，每条边对应一个 condition 对象。一个父节点有 N 个子节点，就有 N 条 conditions。

**conditions JSON 结构**：

```json
{
  "next_node_name": "目标子节点名称",
  "test_names": [...],      // 人工检测路由（二选一）
  "model_names": [...],     // 自动模型路由（二选一）
  "weight": 0.92            // 排查优先级权重（0-1）
}
```

**路由方式一：人工检测（test_names）**

由工程师在现场执行检查，根据检查结果决定是否走这条边。

```json
{
  "next_node_name": "匹配超差",
  "test_names": [
    {"test_name": "排查检具", "person_name": "工程师"},
    {"test_name": "!复测尺寸", "person_name": "工程师"}
  ],
  "weight": 0.99
}
```

| 字段 | 说明 |
|---|---|
| `test_name` | 工程师需要执行的检查动作 |
| `!` 前缀 | 表示"该检查未通过"时触发此路由。如 `!复测尺寸` = 复测尺寸不合格时走这条边 |
| 无 `!` 前缀 | 表示"该检查通过"时触发此路由。如 `排查检具` = 检具排查正常时走这条边 |
| `person_name` | 执行人角色 |

多个 test_name 之间是 AND 关系——所有检查条件都满足时才走这条边。

**路由方式二：自动模型判断（model_names）**

由系统自动化检测模型给出判断结果，根据结果值（expected_value）决定是否走这条边。

```json
{
  "next_node_name": "壳体机台不符合要求",
  "model_names": [
    {"model_name": "阈值判断", "con_value": "FAIL"}
  ],
  "weight": 0.92
}
```

| 字段 | 说明 |
|---|---|
| `model_name` | 自动检测模型的名称（如"阈值判断"、"克重检查"、"变形趋势分析"等） |
| `con_value` | **期望值/路由条件值**（expected_value）。当模型输出等于此值时走这条边。如 `"FAIL"` 表示检测不通过时进入该子节点 |

**两种路由的对比**：

| 维度 | test_names（人工检测） | model_names（自动判断） |
|---|---|---|
| 执行方式 | 工程师现场操作 | 系统自动化模型 |
| 触发条件 | `!` 前缀控制通过/未通过 | `con_value` 匹配模型输出值 |
| 典型场景 | 目视检查、手动测量、排查操作 | 阈值判断、克重检查、变形趋势分析 |
| 示例 | `!白光扫描` → 扫描结果异常 | `阈值判断: FAIL` → 超出阈值 |

#### 2.1.4 权重（weight）

每条边携带 `weight` 字段（0-1），表示该故障原因的排查优先级。值越高，售后工程师应越优先排查该方向。

> 注：当前 weight 由人工经验填写，暂不纳入评估范围。后续当 weight 有数据支撑时（如基于历史频次计算），可扩展评估。

### 2.2 树结构示例

以"匹配不良"故障树为例，整体结构如下：

```
匹配不良（根节点，is_multi_next=true）
│
├─[排查检具 AND !复测尺寸, w=0.99]─→ 匹配超差（中间节点）
│   ├─[阈值判断=FAIL, w=0.97]─→ 灯罩机台不符合要求（叶子）
│   ├─[变形趋势分析=FAIL, w=0.93]─→ 灯罩注塑变形（叶子）
│   ├─[克重检查=FAIL, w=0.95]─→ 灯罩克重不符合要求（叶子）
│   ├─[!库龄检查, w=0.94]─→ 灯罩单件存放时间>3天（叶子）
│   ├─[!晃动定位销, w=0.73]─→ 定位销松动（叶子）
│   ├─[!排查导柱, w=0.69]─→ 导柱松动、歪斜...（叶子）
│   ├─ ... (30+ 子节点)
│   └─[!平头百分表, w=0.52]─→ 测量面差时使用平头表头...（叶子）
│
├─[!排查检具, w=0.99]─→ 检具不合规（叶子）
│
└─[排查检具 AND 复测尺寸, w=0.99]─→ 尺寸合规（中间节点）
    ├─[!检具强度检查, w=0.97]─→ 客户端检具强度不足（叶子）
    ├─[!定位基准确认, w=0.99]─→ 定位偏差（叶子）
    ├─[!卡扣安装频次排查, w=0.95]─→ RPS点不贴合（叶子）
    └─ ... (6 个子节点)
```

说明：
- `[...]` 内为边上的路由条件：人工检测项或模型判断 + 权重
- `w=` 为 weight 值
- 根节点 `is_multi_next=true` 表示可同时探索多个分支
- 叶子节点无子节点，代表最终的故障根因或终止状态

---

## 三、指标体系设计

### 3.1 总览

评估分为 5 层，从基础到深入递进：

| 层级 | 评估维度 | 评分器类型 | 指标 |
|---|---|---|---|
| Layer 1 | 结构合规性 | 代码评分器 | OWL 格式合法性、DAG 合规性 |
| Layer 2 | 节点覆盖度 | 代码 + 模型评分器 | Node Recall、Node Precision、Node F1 |
| Layer 3 | 边准确性 | 模型评分器 | Edge Recall、Edge Precision |
| Layer 4 | 诊断动作准确性 | 模型评分器 | Diagnostic Action Recall |
| Layer 5 | Bad Case 分析 | 模型评分器 | 分类统计 + 优化建议 |

### 3.2 Layer 1：结构合规性（门槛指标）

**目的**：确保生成的 OWL 文件在结构上是合法的，不通过则后续指标无意义。

**指标**：

| 检查项 | 说明 | 计算方式 |
|---|---|---|
| OWL 格式合法性 | 文件能否被 rdflib 正常解析 | `rdflib.Graph().parse()` 是否报错 |
| 单根节点 | 是否有且仅有一个根节点（无入边的节点） | 遍历图结构检查 |
| DAG 无环 | 是否为有向无环图 | DFS 环检测 |
| 无孤立节点 | 是否存在既无入边也无出边的节点 | 遍历检查 |

**评分**：全部通过 = 1.0，任一不通过 = 0.0（布尔门槛）

### 3.3 Layer 2：节点覆盖度

**目的**：评估生成树是否覆盖了标准树中的关键节点，以及是否产生了"幻觉节点"。

**核心挑战**：生成树和标准树的节点名称不会完全一致（如"灯罩注塑变形" vs "灯罩注塑变形，变形量未符合设计要求"），需要语义匹配。

**语义匹配方法**：使用 LLM-as-Judge（DeepEval G-Eval）逐对判断节点是否语义等价。

匹配 Prompt：
> 你是故障树节点匹配评审员。判断以下两个故障诊断节点是否描述同一个故障原因或诊断步骤。
> 允许用词差异，但核心故障机理或诊断动作必须一致。
> 节点A：「{truth_node}」
> 节点B：「{predict_node}」
> 输出：match / no_match，并给出简短理由。

**指标**：

| 指标 | 公式 | 业务意义 |
|---|---|---|
| Node Recall（节点召回率） | 匹配上的 truth 节点数 / truth 总节点数 | 标准树的关键节点是否被覆盖？低 → 遗漏关键故障原因 |
| Node Precision（节点精确率） | 匹配上的 predict 节点数 / predict 总节点数 | 生成的节点是否都是有意义的？低 → 幻觉节点多 |
| Node F1 | 2 * Recall * Precision / (Recall + Precision) | 综合评价 |

### 3.4 Layer 3：边（父子关系）准确性

**目的**：评估生成树的拓扑结构是否与标准树一致。节点存在还不够，挂在哪个父节点下同样关键。

**计算方法**：
1. 基于 Layer 2 的节点匹配结果，建立 truth 节点 → predict 节点的映射表
2. 遍历 truth 树的每条边 (parent, child)，检查 predict 树中对应的映射节点是否也构成父子关系

**指标**：

| 指标 | 公式 | 业务意义 |
|---|---|---|
| Edge Recall（边召回率） | 匹配上的边数 / truth 总边数 | 标准树的诊断路径是否被保留？低 → 层级结构错乱 |
| Edge Precision（边精确率） | 匹配上的边数 / predict 总边数 | 生成的边是否都是合理的？低 → 多余/错误的父子关系 |

**示例**：
- 标准树：匹配不良 → 匹配超差 → 定位销松动
- 生成树：匹配不良 → 定位销松动（跳过了"匹配超差"这一中间层）
- 结果：节点"定位销松动"匹配上了（Layer 2 通过），但边关系错误（Layer 3 未通过）

### 3.5 Layer 4：诊断动作准确性

**目的**：评估边上携带的诊断步骤（conditions 中的 test_names）是否与标准树一致。

**计算方法**：对于每条匹配上的边，用 LLM-as-Judge 比较两棵树中该边的诊断动作列表。

评估 Prompt：
> 标准树中，从「{truth_parent}」到「{truth_child}」的诊断步骤是：{truth_test_names}
> 生成树中，对应边的诊断步骤是：{predict_test_names}
> 请判断：
> 1. 生成的诊断步骤是否覆盖了标准步骤的核心动作？
> 2. 是否有遗漏的关键检查项？
> 3. 是否有多余的无关检查项？
> 输出：coverage_score（0-1）、missing_actions、extra_actions

**指标**：

| 指标 | 公式 | 业务意义 |
|---|---|---|
| Diagnostic Action Recall | 所有匹配边的 coverage_score 均值 | 诊断步骤是否完整？低 → 工程师拿到的排查指引不够用 |

### 3.6 Layer 5：Bad Case 分析与优化建议

**目的**：汇总所有失败项，自动分类统计，生成可执行的优化方向。

**步骤**：

1. **收集失败项**：从 Layer 2-4 中提取所有未匹配的节点、边、诊断动作
2. **LLM 自动分类**：

| 问题类型 | 说明 | 示例 |
|---|---|---|
| 遗漏（Missing） | 标准树有，生成树没有 | 缺少"灯罩脱模角度设计过大"节点 |
| 幻觉（Hallucination） | 生成树有，标准树没有 | 多出"线路接触不良"（该现象不属于此故障） |
| 错挂（Misplaced） | 节点存在但父子关系错误 | "定位销松动"应在"匹配超差"下，实际挂在根节点下 |
| 诊断偏差（Diagnostic Gap） | 边上的诊断步骤不准确 | 缺少"白光扫描"检查步骤 |

3. **频次统计**：按问题类型 + 具体内容计数，排序
4. **生成优化建议**：

| 问题类型 | 典型优化方向 |
|---|---|
| 遗漏多 | 补充更多高质量输入（8D报告覆盖更多场景） |
| 幻觉多 | 优化 Agent prompt，约束生成范围 |
| 错挂多 | 优化 Agent 的树构建逻辑，强化层级结构 prompt |
| 诊断偏差多 | conditions 模板细化，补充领域知识上下文 |

---

## 四、技术实现（基于 DeepEval）

### 4.1 项目结构

```
fault_tree_eval/
├── parsers/
│   └── owl_parser.py              # OWL → Python 树结构解析
├── matchers/
│   └── semantic_matcher.py        # 节点语义匹配（LLM-as-Judge）
├── metrics/
│   ├── structural_validity.py     # Layer 1: 结构合规性
│   ├── node_coverage.py           # Layer 2: 节点覆盖度
│   ├── edge_accuracy.py           # Layer 3: 边准确性
│   ├── diagnostic_action.py       # Layer 4: 诊断动作准确性
│   └── bad_case_analyzer.py       # Layer 5: Bad Case 分析
├── test_fault_tree.py             # DeepEval 测试入口
├── report_generator.py            # 评估报告生成
└── datasets/
    ├── truth/                     # 3 棵专家标准树 .owl
    └── inputs/                    # 3 组输入材料（8D报告 + 纪要）
```

### 4.2 各层级的 DeepEval 实现方式

| 评估层级 | DeepEval 实现方式 | 原因 |
|---|---|---|
| Layer 1 结构合规性 | `BaseMetric` 自定义代码 | 纯代码检查（解析OWL、检测环、检查根节点），不需要 LLM |
| Layer 2 节点覆盖度 | `BaseMetric` + 内部调用 LLM 做语义匹配 | 需要先用 LLM 匹配节点，再用代码算 Recall/Precision |
| Layer 3 边准确性 | `BaseMetric` 自定义代码 | 基于 Layer 2 的映射表做代码计算 |
| Layer 4 诊断动作 | `GEval` 或 `BaseMetric` + LLM | 用 LLM 比较两棵树同一条边上的诊断步骤 |
| Layer 5 Bad Case | 独立分析模块（不是 Metric） | 汇总所有 Metric 的失败项，用 LLM 总结 |

说明：
- **Layer 1、3** 是纯代码计算，不涉及 LLM，用 `BaseMetric` 自定义即可
- **Layer 2** 虽然用 `BaseMetric`，但内部需要调用 LLM 做节点名称的语义匹配（因为生成树和标准树的用词不会完全一致）。不直接用 G-Eval 的原因是：OWL 文件较大（30+ 节点），一次性丢给 LLM 打总分不够精确，逐节点匹配更可控、bad case 更可追溯
- **Layer 4** 每条边的 conditions 数据量小，适合用 G-Eval 让 LLM 整体判断诊断步骤的覆盖度
- **Layer 5** 不是一个评估指标，而是汇总分析模块，独立于 DeepEval 的 Metric 体系

### 4.3 核心代码骨架

#### 4.3.1 OWL 解析器

```python
from rdflib import Graph, Namespace, URIRef
import json

class FaultTreeNode:
    def __init__(self, name: str):
        self.name = name
        self.children: list[str] = []            # 子节点名称列表
        self.conditions: list[dict] = []         # 边上的条件（test_names等）

class FaultTree:
    def __init__(self, nodes: dict[str, FaultTreeNode], root: str):
        self.nodes = nodes      # {name: FaultTreeNode}
        self.root = root        # 根节点名称

    def get_all_nodes(self) -> list[str]:
        return list(self.nodes.keys())

    def get_all_edges(self) -> list[tuple[str, str]]:
        edges = []
        for name, node in self.nodes.items():
            for child in node.children:
                edges.append((name, child))
        return edges

    def get_edge_conditions(self, parent: str, child: str) -> dict:
        node = self.nodes.get(parent)
        if node:
            for cond in node.conditions:
                if cond.get("next_node_name") == child:
                    return cond
        return {}

def parse_owl(owl_path: str) -> FaultTree:
    """将 OWL 文件解析为 FaultTree 结构"""
    g = Graph()
    g.parse(owl_path, format="xml")

    NS = Namespace("http://example.org/fault_tree.owl#")
    nodes = {}

    # 1. 提取所有 DiagnosticNode
    for s in g.subjects(predicate=URIRef("http://www.w3.org/1999/02/22-rdf-syntax-ns#type"),
                        object=NS.DiagnosticNode):
        name = str(s).split("#")[-1]
        nodes[name] = FaultTreeNode(name)

    # 2. 提取 next_nodes 关系
    for s, p, o in g.triples((None, NS.next_nodes, None)):
        parent = str(s).split("#")[-1]
        child = str(o).split("#")[-1]
        if parent in nodes:
            nodes[parent].children.append(child)

    # 3. 提取 conditions
    for s, p, o in g.triples((None, NS.conditions, None)):
        parent = str(s).split("#")[-1]
        if parent in nodes:
            cond = json.loads(str(o))
            nodes[parent].conditions.append(cond)

    # 4. 找根节点（没有任何节点指向它的节点）
    all_children = set()
    for node in nodes.values():
        all_children.update(node.children)
    roots = [n for n in nodes if n not in all_children]

    return FaultTree(nodes=nodes, root=roots[0] if roots else None)
```

#### 4.3.2 语义匹配器

```python
from deepeval.models import DeepEvalBaseLLM

class SemanticNodeMatcher:
    """使用 LLM 进行节点语义匹配"""

    MATCH_PROMPT = """你是故障树节点匹配评审员。判断以下两个故障诊断节点是否描述同一个故障原因或诊断步骤。
允许用词差异，但核心故障机理或诊断动作必须一致。

节点A：「{node_a}」
节点B：「{node_b}」

仅输出 JSON：{{"match": true/false, "reason": "简短理由"}}"""

    def __init__(self, model: DeepEvalBaseLLM):
        self.model = model
        self._cache = {}

    def match(self, node_a: str, node_b: str) -> tuple[bool, str]:
        """返回 (是否匹配, 理由)"""
        key = (node_a, node_b)
        if key in self._cache:
            return self._cache[key]

        prompt = self.MATCH_PROMPT.format(node_a=node_a, node_b=node_b)
        response = self.model.generate(prompt)
        result = json.loads(response)

        self._cache[key] = (result["match"], result["reason"])
        return self._cache[key]

    def build_mapping(self, truth_nodes: list[str], predict_nodes: list[str]) -> dict:
        """建立 truth → predict 的最佳匹配映射"""
        mapping = {}  # truth_node -> predict_node
        used_predict = set()

        for tn in truth_nodes:
            best_match = None
            for pn in predict_nodes:
                if pn in used_predict:
                    continue
                matched, reason = self.match(tn, pn)
                if matched:
                    best_match = pn
                    break
            if best_match:
                mapping[tn] = best_match
                used_predict.add(best_match)

        return mapping
```

#### 4.3.3 Layer 1 — 结构合规性指标

```python
from deepeval.metrics import BaseMetric
from deepeval.test_case import LLMTestCase

class StructuralValidityMetric(BaseMetric):
    """Layer 1: OWL 结构合规性检查"""

    def __init__(self):
        self.threshold = 1.0
        self.score = 0

    @property
    def __name__(self):
        return "Structural Validity"

    def measure(self, test_case: LLMTestCase, *args, **kwargs) -> float:
        owl_content = test_case.actual_output
        checks = {}

        # 检查1: OWL 可解析
        try:
            tree = parse_owl_from_string(owl_content)
            checks["owl_parseable"] = True
        except Exception as e:
            checks["owl_parseable"] = False
            self.score = 0.0
            self.reason = f"OWL 解析失败: {str(e)}"
            self.success = False
            return self.score

        # 检查2: 单根节点
        all_children = set()
        for node in tree.nodes.values():
            all_children.update(node.children)
        roots = [n for n in tree.nodes if n not in all_children]
        checks["single_root"] = len(roots) == 1

        # 检查3: DAG 无环
        checks["no_cycles"] = not self._has_cycle(tree)

        # 检查4: 无孤立节点
        nodes_with_edges = set()
        for node in tree.nodes.values():
            if node.children:
                nodes_with_edges.add(node.name)
            for child in node.children:
                nodes_with_edges.add(child)
        checks["no_orphans"] = nodes_with_edges == set(tree.nodes.keys())

        failed = {k: v for k, v in checks.items() if not v}
        self.score = 1.0 if not failed else 0.0
        self.reason = f"未通过: {list(failed.keys())}" if failed else "全部通过"
        self.success = self.score >= self.threshold
        return self.score

    def _has_cycle(self, tree: FaultTree) -> bool:
        visited = set()
        in_stack = set()

        def dfs(node_name):
            visited.add(node_name)
            in_stack.add(node_name)
            node = tree.nodes.get(node_name)
            if node:
                for child in node.children:
                    if child in in_stack:
                        return True
                    if child not in visited:
                        if dfs(child):
                            return True
            in_stack.remove(node_name)
            return False

        for name in tree.nodes:
            if name not in visited:
                if dfs(name):
                    return True
        return False

    async def a_measure(self, test_case: LLMTestCase, *args, **kwargs) -> float:
        return self.measure(test_case, *args, **kwargs)

    def is_successful(self) -> bool:
        return self.success
```

#### 4.3.4 Layer 2 — 节点覆盖度指标

```python
class NodeCoverageMetric(BaseMetric):
    """Layer 2: 节点覆盖度（Recall / Precision / F1）"""

    def __init__(self, matcher: SemanticNodeMatcher, threshold: float = 0.7):
        self.matcher = matcher
        self.threshold = threshold
        self.score = 0
        self.node_mapping = {}      # 供 Layer 3 使用
        self.missing_nodes = []     # 供 Layer 5 使用
        self.hallucinated_nodes = []

    @property
    def __name__(self):
        return "Node Coverage"

    def measure(self, test_case: LLMTestCase, *args, **kwargs) -> float:
        truth_tree = parse_owl_from_string(test_case.expected_output)
        predict_tree = parse_owl_from_string(test_case.actual_output)

        truth_nodes = truth_tree.get_all_nodes()
        predict_nodes = predict_tree.get_all_nodes()

        # 建立语义匹配映射
        self.node_mapping = self.matcher.build_mapping(truth_nodes, predict_nodes)

        matched_truth = set(self.node_mapping.keys())
        matched_predict = set(self.node_mapping.values())

        # 计算指标
        recall = len(matched_truth) / len(truth_nodes) if truth_nodes else 0
        precision = len(matched_predict) / len(predict_nodes) if predict_nodes else 0
        f1 = (2 * recall * precision / (recall + precision)) if (recall + precision) > 0 else 0

        # 记录 bad case
        self.missing_nodes = [n for n in truth_nodes if n not in matched_truth]
        self.hallucinated_nodes = [n for n in predict_nodes if n not in matched_predict]

        self.score = f1
        self.reason = (
            f"Recall={recall:.2%}, Precision={precision:.2%}, F1={f1:.2%}\n"
            f"遗漏节点({len(self.missing_nodes)}个): {self.missing_nodes[:5]}...\n"
            f"幻觉节点({len(self.hallucinated_nodes)}个): {self.hallucinated_nodes[:5]}..."
        )
        self.success = self.score >= self.threshold
        return self.score

    async def a_measure(self, test_case: LLMTestCase, *args, **kwargs) -> float:
        return self.measure(test_case, *args, **kwargs)

    def is_successful(self) -> bool:
        return self.success
```

#### 4.3.5 Layer 3 — 边准确性指标

```python
class EdgeAccuracyMetric(BaseMetric):
    """Layer 3: 边（父子关系）准确性"""

    def __init__(self, node_coverage_metric: NodeCoverageMetric, threshold: float = 0.6):
        self.node_coverage = node_coverage_metric
        self.threshold = threshold
        self.score = 0
        self.missed_edges = []
        self.extra_edges = []

    @property
    def __name__(self):
        return "Edge Accuracy"

    def measure(self, test_case: LLMTestCase, *args, **kwargs) -> float:
        truth_tree = parse_owl_from_string(test_case.expected_output)
        predict_tree = parse_owl_from_string(test_case.actual_output)

        mapping = self.node_coverage.node_mapping  # truth -> predict
        reverse_mapping = {v: k for k, v in mapping.items()}  # predict -> truth

        truth_edges = truth_tree.get_all_edges()
        predict_edges = set(predict_tree.get_all_edges())

        # 检查 truth 的每条边在 predict 中是否存在
        matched = 0
        self.missed_edges = []
        for (tp, tc) in truth_edges:
            pp = mapping.get(tp)
            pc = mapping.get(tc)
            if pp and pc and (pp, pc) in predict_edges:
                matched += 1
            else:
                self.missed_edges.append((tp, tc))

        # 检查 predict 中多余的边
        truth_edges_set = set(truth_edges)
        self.extra_edges = []
        for (pp, pc) in predict_edges:
            tp = reverse_mapping.get(pp)
            tc = reverse_mapping.get(pc)
            if not tp or not tc or (tp, tc) not in truth_edges_set:
                self.extra_edges.append((pp, pc))

        recall = matched / len(truth_edges) if truth_edges else 0
        precision = matched / len(predict_edges) if predict_edges else 0

        self.score = recall  # 主指标用 recall
        self.reason = (
            f"Edge Recall={recall:.2%}, Edge Precision={precision:.2%}\n"
            f"缺失的边({len(self.missed_edges)}条): {self.missed_edges[:5]}...\n"
            f"多余的边({len(self.extra_edges)}条): {self.extra_edges[:5]}..."
        )
        self.success = self.score >= self.threshold
        return self.score

    async def a_measure(self, test_case: LLMTestCase, *args, **kwargs) -> float:
        return self.measure(test_case, *args, **kwargs)

    def is_successful(self) -> bool:
        return self.success
```

#### 4.3.6 Layer 4 — 诊断动作准确性

```python
class DiagnosticActionMetric(BaseMetric):
    """Layer 4: 诊断动作（conditions中test_names）准确性"""

    EVAL_PROMPT = """你是故障树诊断动作评估员。请比较标准树和生成树中同一条边上的诊断步骤。

标准树边: 从「{truth_parent}」到「{truth_child}」
标准诊断步骤: {truth_actions}

生成树边: 从「{predict_parent}」到「{predict_child}」
生成诊断步骤: {predict_actions}

请评估生成的诊断步骤对标准步骤的覆盖程度。
输出 JSON：
{{
  "coverage_score": 0.0-1.0,
  "missing_actions": ["缺失的关键步骤"],
  "extra_actions": ["多余的步骤"],
  "reason": "简短评价"
}}"""

    def __init__(self, model, node_coverage_metric: NodeCoverageMetric, threshold: float = 0.6):
        self.model = model
        self.node_coverage = node_coverage_metric
        self.threshold = threshold
        self.score = 0
        self.details = []

    @property
    def __name__(self):
        return "Diagnostic Action Recall"

    def measure(self, test_case: LLMTestCase, *args, **kwargs) -> float:
        truth_tree = parse_owl_from_string(test_case.expected_output)
        predict_tree = parse_owl_from_string(test_case.actual_output)
        mapping = self.node_coverage.node_mapping

        scores = []
        self.details = []

        for (tp, tc) in truth_tree.get_all_edges():
            pp, pc = mapping.get(tp), mapping.get(tc)
            if not pp or not pc:
                continue

            truth_cond = truth_tree.get_edge_conditions(tp, tc)
            predict_cond = predict_tree.get_edge_conditions(pp, pc)

            truth_actions = self._extract_actions(truth_cond)
            predict_actions = self._extract_actions(predict_cond)

            if not truth_actions:
                continue

            # LLM 评估
            prompt = self.EVAL_PROMPT.format(
                truth_parent=tp, truth_child=tc, truth_actions=truth_actions,
                predict_parent=pp, predict_child=pc, predict_actions=predict_actions,
            )
            response = self.model.generate(prompt)
            result = json.loads(response)
            scores.append(result["coverage_score"])
            self.details.append({
                "edge": f"{tp} → {tc}",
                **result
            })

        self.score = sum(scores) / len(scores) if scores else 0
        self.reason = f"平均覆盖率={self.score:.2%}，评估了{len(scores)}条边"
        self.success = self.score >= self.threshold
        return self.score

    def _extract_actions(self, condition: dict) -> list[str]:
        actions = []
        for test in condition.get("test_names", []):
            actions.append(test.get("test_name", ""))
        return actions

    async def a_measure(self, test_case: LLMTestCase, *args, **kwargs) -> float:
        return self.measure(test_case, *args, **kwargs)

    def is_successful(self) -> bool:
        return self.success
```

#### 4.3.7 Layer 5 — Bad Case 分析器

```python
class BadCaseAnalyzer:
    """Layer 5: 汇总所有失败项，自动分类并生成优化建议"""

    ANALYSIS_PROMPT = """你是 AI 故障树生成系统的评估分析师。以下是本轮评估中发现的所有问题：

## 遗漏的节点（标准树有，生成树没有）
{missing_nodes}

## 幻觉节点（生成树有，标准树没有）
{hallucinated_nodes}

## 缺失的边（父子关系未正确建立）
{missed_edges}

## 多余的边
{extra_edges}

## 诊断动作偏差
{diagnostic_gaps}

请完成以下分析：
1. 将所有问题按类型分类（遗漏/幻觉/错挂/诊断偏差）
2. 在每个类型内，按相似性聚类（如"多个零件的存放时间节点都遗漏了"）
3. 统计每类问题的数量，按严重程度和频次排序
4. 对 Top-3 高频问题，给出具体的优化建议：
   - 是否需要补充更多输入数据？
   - 是否需要优化 Agent 的 prompt？
   - 是否需要调整 Agent 的树构建流程？
   - 是否需要引入更多领域知识上下文？

输出结构化 JSON。"""

    def __init__(self, model):
        self.model = model

    def analyze(self,
                node_metric: NodeCoverageMetric,
                edge_metric: EdgeAccuracyMetric,
                action_metric: DiagnosticActionMetric) -> dict:

        prompt = self.ANALYSIS_PROMPT.format(
            missing_nodes=json.dumps(node_metric.missing_nodes, ensure_ascii=False),
            hallucinated_nodes=json.dumps(node_metric.hallucinated_nodes, ensure_ascii=False),
            missed_edges=json.dumps(node_metric.missed_edges, ensure_ascii=False),
            extra_edges=json.dumps(edge_metric.extra_edges, ensure_ascii=False),
            diagnostic_gaps=json.dumps(action_metric.details, ensure_ascii=False),
        )

        response = self.model.generate(prompt)
        return json.loads(response)
```

#### 4.3.8 评估入口：两种模式

DeepEval 支持两种评估模式，分别适用于不同场景：

**模式 A：已有 OWL 文件，直接评估**

适用于 Agent 已经生成好了 OWL 文件，只需评估树的质量。手动构造 `LLMTestCase`，其中 `input` 仅作标识用途（不参与指标计算），真正参与评估的是 `actual_output` 和 `expected_output`。

```python
from deepeval import evaluate
from deepeval.test_case import LLMTestCase
from deepeval.dataset import EvaluationDataset

def run_evaluation_mode_a():
    """模式 A：已有 OWL 文件，直接评估"""

    # 1. 构造测试用例
    test_cases = []
    for case_name in ["匹配不良", "故障现象B", "故障现象C"]:
        truth_owl = read_file(f"datasets/truth/{case_name}.owl")
        predict_owl = read_file(f"datasets/predict/{case_name}.owl")

        test_cases.append(LLMTestCase(
            input=f"故障现象：{case_name}",  # 标识用，不参与指标计算
            actual_output=predict_owl,        # Agent 生成的树
            expected_output=truth_owl,        # 专家标准树
        ))

    # 2. 初始化指标
    matcher = SemanticNodeMatcher(model=your_llm)
    structural = StructuralValidityMetric()
    node_coverage = NodeCoverageMetric(matcher=matcher)
    edge_accuracy = EdgeAccuracyMetric(node_coverage_metric=node_coverage)
    diagnostic_action = DiagnosticActionMetric(
        model=your_llm,
        node_coverage_metric=node_coverage
    )

    # 3. 运行评估
    results = evaluate(
        test_cases=test_cases,
        metrics=[structural, node_coverage, edge_accuracy, diagnostic_action],
    )

    # 4. Bad Case 分析
    analyzer = BadCaseAnalyzer(model=your_llm)
    report = analyzer.analyze(node_coverage, edge_accuracy, diagnostic_action)
    print_report(results, report)
```

**模式 B：从 input 端到端自动评估（推荐）**

适用于每次迭代 Agent 后，一键跑评估。利用 DeepEval 内置的 `Golden` + `EvaluationDataset.generate_test_cases()` 机制，**自动调用 Agent 生成树、收集输出、组装 TestCase、跑评估**，无需手动操作。

核心概念：
- `Golden`（黄金数据）= 只有 `input`（8D 报告）和 `expected_output`（标准树），没有 `actual_output`。相当于"试卷"（只有题目和标准答案）
- `generate_test_cases(fn=...)` = DeepEval 内置方法，自动遍历每个 Golden，调用你提供的 Agent 函数，把返回值填入 `actual_output`，生成完整的 `LLMTestCase`。相当于"让考生答题"
- 评估 = "判卷"

```
Golden（试卷：input + expected_output）
   │
   ▼  generate_test_cases(fn=your_agent)  ← DeepEval 自动调用
   │
LLMTestCase（答卷：input + actual_output + expected_output）
   │
   ▼  evaluate(metrics=[...])
   │
评估结果（分数 + bad case）
```

**第一步：包装 Agent 建树函数**

将你的多步骤 Agent 包装成一个签名为 `(str) -> str` 的函数：

```python
def build_fault_tree(input_text: str) -> str:
    """
    输入：8D 报告 + 会议纪要的文本
    输出：OWL 格式的故障树字符串
    """
    # 你的多步骤 Agent 逻辑
    owl_string = your_agent_pipeline(input_text)
    return owl_string
```

如果 Agent 当前是通过 IDE 按钮触发、读写文件的形式，简单包一层即可：

```python
import subprocess

def build_fault_tree(input_text: str) -> str:
    # 写临时输入文件
    with open("/tmp/agent_input.txt", "w") as f:
        f.write(input_text)

    # 调用已有的建树流程
    subprocess.run([
        "python", "your_agent_main.py",
        "--input", "/tmp/agent_input.txt",
        "--output", "/tmp/agent_output.owl"
    ], check=True)

    # 读取生成的 OWL
    with open("/tmp/agent_output.owl", "r") as f:
        return f.read()
```

**第二步：端到端评估**

```python
from deepeval.dataset import EvaluationDataset, Golden
from deepeval import evaluate

def run_evaluation_mode_b():
    """模式 B：从 input 端到端自动评估"""

    # 1. 准备 Golden 数据（"试卷"）
    dataset = EvaluationDataset(goldens=[
        Golden(
            input=read_file("datasets/inputs/匹配不良.txt"),
            expected_output=read_file("datasets/truth/匹配不良.owl")
        ),
        Golden(
            input=read_file("datasets/inputs/故障现象B.txt"),
            expected_output=read_file("datasets/truth/故障现象B.owl")
        ),
        Golden(
            input=read_file("datasets/inputs/故障现象C.txt"),
            expected_output=read_file("datasets/truth/故障现象C.owl")
        ),
    ])

    # 2. 让 Agent "考试"—— DeepEval 自动调用 build_fault_tree
    #    遍历每个 Golden，执行 build_fault_tree(golden.input)
    #    把返回的 OWL 字符串填入 actual_output，生成 LLMTestCase
    dataset.generate_test_cases(fn=build_fault_tree)

    # 3. 初始化指标
    matcher = SemanticNodeMatcher(model=your_llm)
    structural = StructuralValidityMetric()
    node_coverage = NodeCoverageMetric(matcher=matcher)
    edge_accuracy = EdgeAccuracyMetric(node_coverage_metric=node_coverage)
    diagnostic_action = DiagnosticActionMetric(
        model=your_llm,
        node_coverage_metric=node_coverage
    )

    # 4. "判卷"
    results = evaluate(
        test_cases=dataset.test_cases,
        metrics=[structural, node_coverage, edge_accuracy, diagnostic_action],
    )

    # 5. Bad Case 分析
    analyzer = BadCaseAnalyzer(model=your_llm)
    report = analyzer.analyze(node_coverage, edge_accuracy, diagnostic_action)
    print_report(results, report)

if __name__ == "__main__":
    run_evaluation_mode_b()
```

**两种模式的对比：**

| 维度 | 模式 A（直接评估） | 模式 B（端到端） |
|---|---|---|
| 输入 | 已有的 OWL 文件 | 原始 8D 报告 + 纪要 |
| Agent 调用 | 不调用，只评估 | DeepEval 通过 `generate_test_cases(fn=...)` 自动调用 |
| 适用场景 | 快速检查某棵已生成的树 | 迭代优化 Agent 后一键跑评估 |
| pass@k | 不支持（只有一棵树） | 支持：对同一 input 跑 k 次，统计一致性 |
| DeepEval 机制 | 手动构造 `LLMTestCase` | `Golden` + `generate_test_cases(fn=...)` |

---

## 五、产品形态：CLI 命令行工具

### 5.1 定位

评估系统以 **CLI 命令行工具**（`fault-tree-eval`）作为入口，可直接在终端运行，也可被 IDE 建树工具集成调用。

设计原则：
- 轻量、无依赖外部服务（除 LLM API）
- 输出人类可读的终端报告 + 机器可读的 JSON 文件
- 后续可被 IDE 插件封装为按钮式操作

### 5.2 项目目录结构

评估工具集成在数据科学项目中，与故障树文件共存：

```
project/
├── fault_trees/
│   ├── truth/                        # 专家标准树（ground truth）
│   │   ├── 匹配不良.owl
│   │   ├── 故障B.owl
│   │   └── 故障C.owl
│   └── generated/                    # Agent 生成的树
│       ├── 匹配不良_v1.0.owl
│       └── 匹配不良_v1.1.owl
├── eval/
│   ├── cli.py                        # CLI 入口
│   ├── parsers/
│   │   └── owl_parser.py             # OWL 解析器
│   ├── matchers/
│   │   └── semantic_matcher.py       # 语义匹配器
│   ├── metrics/
│   │   ├── structural_validity.py    # Layer 1
│   │   ├── node_coverage.py          # Layer 2
│   │   ├── edge_accuracy.py          # Layer 3
│   │   ├── diagnostic_action.py      # Layer 4
│   │   └── bad_case_analyzer.py      # Layer 5
│   └── reports/                      # 评估报告输出
│       ├── eval_20260323_001.json
│       └── eval_20260323_001.txt
└── ...
```

### 5.3 CLI 命令设计

#### 5.3.1 单棵树评估

```bash
fault-tree-eval run \
  --predict fault_trees/generated/匹配不良_v1.1.owl \
  --truth fault_trees/truth/匹配不良.owl \
  --output eval/reports/
```

#### 5.3.2 批量评估

自动匹配 `generated/` 和 `truth/` 目录下同名文件，批量评估所有故障树：

```bash
fault-tree-eval run-all \
  --predict-dir fault_trees/generated/ \
  --truth-dir fault_trees/truth/ \
  --output eval/reports/
```

#### 5.3.3 查看历史报告

```bash
# 查看最近一次评估结果
fault-tree-eval report --latest

# 对比两个版本的评估结果
fault-tree-eval report --compare v1.0 v1.1
```

#### 5.3.4 查看 Bad Case 明细

```bash
fault-tree-eval badcase --run-id 20260323_001
```

### 5.4 终端输出示例

```
$ fault-tree-eval run --predict generated/匹配不良_v1.1.owl --truth truth/匹配不良.owl

故障树评估 -- 匹配不良
============================================

Layer 1  结构合规性              PASS
  OWL格式: PASS  DAG无环: PASS  单根: PASS

Layer 2  节点覆盖度              0.78
  Recall: 82%  Precision: 75%  F1: 78%
  遗漏: 7个  幻觉: 10个

Layer 3  边准确性                0.68
  Edge Recall: 68%  Edge Precision: 62%
  错挂: 8条

Layer 4  诊断动作覆盖率          0.71
  评估了 35 条边

============================================
综合得分: 0.72

Bad Case Top-3:
  1. 叶子节点遗漏(12次) -> 补充更多输入场景
  2. 层级错挂(8次) -> 优化树构建prompt
  3. 诊断步骤模糊(5次) -> 细化conditions模板

完整报告: eval/reports/eval_20260323_001.json
```

### 5.5 JSON 报告结构

每次评估生成的 JSON 报告包含完整的可编程数据：

```json
{
  "run_id": "20260323_001",
  "timestamp": "2026-03-23T14:30:00",
  "test_case": "匹配不良",
  "predict_file": "fault_trees/generated/匹配不良_v1.1.owl",
  "truth_file": "fault_trees/truth/匹配不良.owl",
  "metrics": {
    "structural_validity": {
      "score": 1.0,
      "details": {"owl_parseable": true, "single_root": true, "no_cycles": true, "no_orphans": true}
    },
    "node_coverage": {
      "recall": 0.82,
      "precision": 0.75,
      "f1": 0.78,
      "missing_nodes": ["灯罩脱模角度设计过大...", "..."],
      "hallucinated_nodes": ["线路接触不良", "..."]
    },
    "edge_accuracy": {
      "recall": 0.68,
      "precision": 0.62,
      "missed_edges": [["匹配超差", "定位销松动"], "..."],
      "extra_edges": [["匹配不良", "定位销松动"], "..."]
    },
    "diagnostic_action": {
      "average_coverage": 0.71,
      "evaluated_edges": 35,
      "details": [
        {
          "edge": "匹配不良 -> 匹配超差",
          "coverage_score": 0.85,
          "missing_actions": [],
          "extra_actions": ["额外检查项"]
        }
      ]
    }
  },
  "bad_case_analysis": {
    "summary": [
      {"type": "遗漏", "count": 12, "description": "叶子节点遗漏", "suggestion": "补充更多输入场景"},
      {"type": "错挂", "count": 8, "description": "层级关系错误", "suggestion": "优化树构建prompt"},
      {"type": "诊断偏差", "count": 5, "description": "诊断步骤模糊", "suggestion": "细化conditions模板"}
    ]
  },
  "overall_score": 0.72
}
```

### 5.6 与 IDE 集成

IDE 建树工具可通过以下方式集成评估：

```python
# IDE 建树完成后的回调
import subprocess
import json

def on_tree_generated(predict_owl_path: str, truth_owl_path: str):
    """建树完成后自动触发评估"""
    result = subprocess.run(
        ["fault-tree-eval", "run",
         "--predict", predict_owl_path,
         "--truth", truth_owl_path,
         "--format", "json"],
        capture_output=True, text=True
    )
    report = json.loads(result.stdout)

    # 在 IDE 中展示评估结果
    show_eval_panel(report)
```

后续演进路径：
1. **当前**：CLI 命令行，开发人员手动运行或 IDE 自动调用
2. **中期**：IDE 内嵌评估面板，建树后自动弹出评估结果
3. **远期**：Web Dashboard，展示历史趋势、多版本对比、团队协作 review

---

## 六、评估报告详细样例

```
╔══════════════════════════════════════════════════════════╗
║           故障树生成质量评估报告                           ║
║           评估时间: 2026-03-23                            ║
╠══════════════════════════════════════════════════════════╣
║                                                          ║
║ ▶ 测试用例: 匹配不良                                      ║
║                                                          ║
║ Layer 1 结构合规性                                        ║
║   OWL格式:  PASS    DAG无环:  PASS    单根:  PASS         ║
║                                                          ║
║ Layer 2 节点覆盖度                                        ║
║   Recall: 82%    Precision: 75%    F1: 78%               ║
║   遗漏节点(7个):                                          ║
║     - 灯罩脱模角度设计过大导致脱卡后匹配不良                ║
║     - 转盘与支撑块间隙不均匀，导致受力不均匀                ║
║     - ...                                                ║
║   幻觉节点(10个):                                         ║
║     - 线路接触不良（不属于此故障类型）                       ║
║     - ...                                                ║
║                                                          ║
║ Layer 3 边准确性                                          ║
║   Edge Recall: 68%    Edge Precision: 62%                ║
║   典型错挂:                                               ║
║     - "定位销松动" 应挂在 "匹配超差" 下，实际挂在根节点下   ║
║                                                          ║
║ Layer 4 诊断动作覆盖率                                    ║
║   平均覆盖率: 71%                                        ║
║   缺失: "排查检具→复测尺寸" 路径的诊断步骤未生成           ║
║                                                          ║
║ Layer 5 Bad Case 分析                                    ║
║   Top-3 问题:                                            ║
║   1. 叶子节点遗漏(12次)                                   ║
║      → 建议: 输入中补充更多历史 8D 报告覆盖更多场景         ║
║   2. 层级错挂(8次)                                        ║
║      → 建议: 优化 Agent 树构建 prompt，强化层级约束        ║
║   3. 诊断步骤模糊(5次)                                    ║
║      → 建议: 在 prompt 中补充 conditions 填写模板          ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```

---

## 七、迭代优化流程

```
┌──────────────┐     ┌───────────────┐     ┌──────────────┐
│ 第N版 Agent   │────▶│  评估管线      │────▶│  评估报告     │
│ (prompt/流程) │     │  (5层指标)     │     │  (分数+明细)  │
└──────────────┘     └───────────────┘     └──────┬───────┘
       ▲                                          │
       │              ┌───────────────┐           │
       └──────────────│  Bad Case 分析 │◀──────────┘
         针对性优化     │  Top-N 问题    │
                      │  优化建议      │
                      └───────────────┘
```

每次迭代：
1. 运行评估，记录各层指标分数
2. 查看 Bad Case 分析，找到 Top 问题
3. 针对性优化 Agent（改 prompt、改流程、补数据）
4. 重新生成故障树，再次评估
5. 对比前后分数变化，确认改进方向正确

建议记录每次评估的分数趋势，形成类似：

| 版本 | Node F1 | Edge Recall | Diagnostic Recall | 主要改动 |
|---|---|---|---|---|
| v1.0 | 65% | 52% | 58% | 基线版本 |
| v1.1 | 78% | 68% | 71% | 优化了树结构 prompt |
| v1.2 | 82% | 75% | 74% | 补充了 conditions 模板 |
| v1.3 | 85% | 80% | 82% | 增加了输入数据量 |

---

## 八、后续扩展方向

### 8.1 扩充标准树

当前仅有 3 棵标准树，统计意义有限。建议逐步扩充至 10-20 棵，覆盖更多故障类型，使评估结果更稳健。

### 8.2 引入 8D 报告路径回放

在标准树数量充足后，可补充你同事方案中的"8D 报告路径回放"作为第二验证维度：用历史 8D 报告检验生成的树是否能覆盖真实案例的排查路径。

### 8.3 Weight 评估

当 weight 字段有了数据支撑（如基于历史频次计算），可新增 Weight 合理性指标：高频历史故障的 weight 是否更高、权重排序是否与专家经验一致。

### 8.4 pass@k 一致性评估

参考 Anthropic 文章的建议，对同一输入多次生成故障树（如 k=5），统计：
- pass@k（至少一次达标的概率）→ 衡量 Agent 能力上限
- pass^k（全部达标的概率）→ 衡量 Agent 一致性/稳定性
