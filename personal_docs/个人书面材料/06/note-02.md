# Agent 框架理想架构总结

> 基于 2026-07-09 "agent 框架梳理升级" 会议纪要 + 文字记录整理

---

## 一、背景

Agent 框架从 2025 年底开发至今约 7-8 个月，因赶交付节点 + 工作站/站点双部署 + 数据出网 + 手机 APP 适配等约束，一路做了很多妥协。本次会议目标：**先不管现状怎么改，先把"理想状态下应该长什么样"对齐**，明确各服务的定位与边界，再规划迁移路径。

---

## 二、核心设计原则

### 1. 部署模型：只保留两种，不再混部

- **工作站版**（用户私有化部署）— 手机 APP 需要在工厂生产网使用的场景必选
- **站点版**（如阿里云）— 对网络不敏感的用户使用
- **不再存在"一部分服务在工作站、一部分在站点"的混部形态**（维护成本太高，等于三个版本）
- 存量：星宇、德赛已完成迁移到工作站；**威迈斯需要迁到工作站**

### 2. 前后端分离

- 前端项目只保留两个：`LS-AI-coach`（开发态）+ `LS-Lison`（运行态展示）
- 手机 APP 与 Web 未来只连一套服务，不再兼容两套 API

### 3. 框架代码统一封装到 `lads`

- 所有 agent（不论 workflow-base 还是未来的 LLM-base/Hermes）都使用同一套底层框架
- 星宇负责推动 `LS-Lison`，明强 + 姬亮联合推进封装

---

## 三、服务分类总览

按**生命周期阶段**把 8 个服务分成三类：**开发态 / 运行态 / 基础能力**。开发态服务用于"造 agent"，运行态服务用于"跑 agent"，基础能力服务被两侧共同引用。

| 分类 | 服务 | 定位 |
|---|---|---|
| **开发态**（Design-time） | ① LS-AI-coach（前端） | Agent / 感知计算 开发与配置界面 |
| | ② Agent Management（后端） | Agent 元信息 / 版本 / 实例 / 部署 / 回测 |
| **运行态**（Runtime） | ③ LS-Lison（前端） | 工单播报、详情、Todo 展示 |
| | ④ Agent Runtime（后端） | 感知 + 决策 + 生成 Todo 的执行容器 |
| | ⑤ iwork（后端） | Todo 生命周期 / 人事匹配 / 催办 / 日报周报 |
| | ⑥ LS-Socket（后端） | 唯一的 Socket Server，前端↔Agent 双向通信 |
| **基础能力**（Foundation） | ⑦ lads（框架库） | BPMN / workflow 引擎 / Agent 基础封装 |
| | ⑧ Ontology（后端） | 知识库 / 角色配置 / 未来的上下文构建 + Action 执行 |

---

## 四、开发态服务（Design-time）

**面向 Agent 开发者**：负责 agent 的"造 / 配 / 管 / 测"，不承担业务侧运行流量。

### ① LS-AI-coach — 开发态前端

- 编辑器（故障树、SOP、workflow 等）
- Agent 配置页面（skill / tools / MCP 连接）
- 实例管理 / 推送管理
- 感知计算开发页面

### ② Agent Management — 管理后端（原 iwork 的 catalog 部分，改名）

- **Agent 元信息管理**：配置、概述、skill、tools、MCP
- **版本管理 / 生命周期**
- **实例部署**（通过 Airflow 编排容器）
- **回测能力**
- **FTA 服务并入**（故障树版本 / 实例的生命周期维护）
- 负责人：**刘翔宇**

---

## 五、运行态服务（Runtime）

**面向业务用户**：Agent 跑起来后，端到端支撑"感知 → 决策 → 生成 todo → 分派与催办 → 展示"整条链路。

### ③ LS-Lison — 运行态前端

- 工单播报列表 / 详情
- minio 上传
- Todo 展示

### ④ Agent Runtime — 运行时执行容器（原 XX-agent 运行服务）

- 每类 agent 一个容器
- **感知**：check 节点 / API / 未来 ontology function
- **决策**：workflow SOP / LLM + skills
- 生成 todo 清单 → 交给 iwork
- **容器内内嵌 Socket Client**，自己加入房间收消息（无需外部路由）

### ⑤ iwork — Todo 生命周期管理服务

- 承接 LS-Lison 后端 + 原 iwork todo 能力
- **Todo 生命周期**：产生 → 指派 → 反馈 → 催办 → 完成
- **人事匹配**（涉及用户体系，必须是 private 站点应用）
- 日报 / 周报 / 未办事项汇总
- 对外提供 todo 播报列表、详情等查询接口

### ⑥ LS-Socket — 唯一的 Socket Server

- 由 LS-Socket + Lison-Socket 合并而来
- Web / 手机 APP / Agent Runtime 都作为 Client 连它
- 通过 `session_id` 房间匹配，**取消原本的显式消息路由**

---

## 六、基础能力（Foundation）

**被开发态和运行态共同引用**：既是 agent 开发时的编程框架，也是 agent 运行时调用的知识/能力底座。

### ⑦ lads — Agent Framework（框架库，非独立服务）

- 底层框架：BPMN、重跑、workflow 引擎、决策/感知封装
- 所有 agent 引用同一套（不论 workflow-base 还是 LLM-base）
- 星宇 / 明强 / 姬亮联合推进封装

### ⑧ Ontology — 本体与知识服务

- **当前**：CRUD 接口 + 人事匹配配置 + 知识库
- **未来演进**：真正的 Ontology 能力
  - 上下文构建
  - Action 执行（如 `generate_8D_report` 等算子）
  - FTA 生成类接口逐步迁移过来

---

## 七、数据/消息流向（理想态）

```
[开发者]                     [业务用户]
   │                             │
   ▼                             ▼
LS-AI-coach              LS-Lison / 手机 APP
   │                             │
   ▼                    ┌────────┴────────┐
Agent Management        │                 │
   │                    ▼                 ▼
   │ 部署/回测        iwork            LS-Socket ◄───┐
   ▼                (查询接口)        (双向通道)     │
[容器编排 Airflow]        ▲                 ▲        │
   │                     │ todo            │ 消息    │
   ▼                     │                 │        │
Agent Runtime ───────────┴─────────────────┘        │
   │                                                 │
   └──► 引用 lads / 调用 Ontology ◄──────────────────┘
```

---

## 八、关键决策与变化点

| 主题 | 现状 | 理想状态 |
|---|---|---|
| Socket 服务 | LS-Socket（给 APP）+ Lison-Socket（给 Web）两套 | 合并成 1 个 Socket Server，Agent 容器内嵌 Client 加入房间 |
| 消息路由 | iwork 承担路由，需要查容器 IP/端口再转发 | 取消显式路由，靠 Socket 房间的 session_id 自然分发 |
| iwork 定位 | 承担管理 + 路由 + todo 三种混杂能力 | 拆分：管理 → Agent Management；路由 → 取消；本体 → 纯 Todo 生命周期管理 |
| LS-Lison 后端 | 承接 todo 展示查询接口 | 能力并入 iwork（横向扩展成完整 todo 管理） |
| 前端边界 | AI-coach + LS-Lison 页面互相混淆（如连小山配置在 AI-coach） | 严格按"开发态 vs 运行态"分离 |
| FTA 服务 | 独立服务 | 并入 Agent Management；生成类算子迁到 Ontology |
| 部署形态 | 工作站/站点混部，三份版本 | 全工作站 or 全站点，两份版本 |

---

## 九、遗留问题（待苏永杰对齐）

1. **手机 APP Token 问题（阻塞项）**
   - 现状：手机 APP 用 Flat 登录、Flat token，无法访问工作站的 iwork / agent / lison-service
   - 原因：工作站无 JWT token 体系
   - 需要与宫雷雷对齐 token 方案，否则"手机 APP 只能连 Frappe"的约束会一直存在，无法真正做到一套服务

2. **Agent Management 是否做成 Frappe**
   - 偏元信息管理，从元信息角度做 Frappe 合适
   - 但需要通过 Airflow 操控容器生命周期，需再权衡

3. **Ontology 升级路径**
   - 如何把真 Ontology 能力（上下文构建 + Action 执行）逐步注入

---

## 十、待办

| # | 事项 | 负责人 |
|---|---|---|
| 1 | 联系雷雷确认手机 APP 无法访问 iwork/agent/lison-service 的具体原因 | 苏永杰 |
| 2 | 与宫雷雷对齐手机 APP 访问服务的 Token 解决方案 | 苏永杰 |
| 3 | 找涛哥对齐整体升级方案 + Ontology 能力补齐路线 | 苏永杰 |
| 4 | 截图会议中的理想架构图 + 内容总结，发群 | 葛蕊 |
| 5 | 整理各组件责任范围共识文档，苏永杰核对后发群 | 葛蕊 |
| 6 | 推动威迈斯项目迁到工作站 | — |
| 7 | 逐步将 FTA 服务并入 Agent Management、生成接口迁到 Ontology | — |
