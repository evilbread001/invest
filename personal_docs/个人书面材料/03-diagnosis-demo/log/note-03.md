# 产线诊断 DSH Demo Implementation Plan

**Goal:** 按已讨论的任务卡＋对话方案提供独立、可运行的本地 Demo。

**Architecture:** 真实 DSH AgentLoop / ToolRuntime 驱动诊断工具；独立确定性诊断状态机维护前置条件、历史优先级、证据及人工确认。网页与模型工具共享后端状态，模型仅提议写操作，网页确认后提交。

**Tech Stack:** Node 24、TypeScript、DSH rc.7、原生 HTTP、无构建前端、node:test。

**Spec:** 当前对话方案及 `/Users/gerui1/project_experience_notes.md` 第 2 项目第 8–9 节。

## Global Constraints
- 不修改既有仓库；本地软链接复用已安装依赖，不读取已有凭据。
- 模拟数据和脚本模型明确标识；真实模式使用显式环境变量和兼容 chat-completions 的端点。
- 不控制真实产线，不自动确认根因，不发布正式故障树。
- 本地单进程 JSON 保存领域状态；DSH 会话日志只在内存，重启后用业务快照和对话摘要重建上下文。

## Review Focus
- 暂缓/未知不得成为正常或排除原因；安全前置不可被自然语言绕过。
- 模型工具不可跨会话、不可直接提交结果；提议必须确认且校验版本。
- 重复请求和过期卡片不覆盖新结果，模型故障不改变领域结果。
- 模型无凭据不得假装真实推理；错误不泄漏密钥。
- 前端不将模型/用户文本作为 HTML，附件限制类型大小。

## Tasks
- [x] 先写诊断状态测试：历史排序、前置门禁、暂缓、人工提议确认、版本冲突、幂等、重启恢复。实现 `src/diagnosis.ts`。
- [x] 写 DSH 集成和模型适配测试，实现 `src/runtime.ts`、`src/diagnosis-plugin.ts`、`src/adapters.ts`。
- [x] 写 HTTP 行为测试，实现 `src/server.ts` 和任务/对话/历史三栏页面。
- [x] 运行全量 Demo 测试、类型检查与页面/API 冒烟；记录结果，提供 README 启动命令和限制。

## Execution decisions
- 用户已要求按方案直接实现，本次由主助手连续完成，不创建分支或提交。
- 不嵌入带真实数据库副作用的 PCBA 包；用同边界的本地状态机演示，并说明后续 Python 引擎接入点。
- 验证完成：13 项自动化测试＋类型检查通过。浏览器验证卡片反馈、暂缓确认、措施推荐及模拟分析；发现并修复报告按钮忙碌后未恢复问题。源码复核发现并用回归测试修复非确认动作夹带 proposalId 的参数混用问题。
- DSH 的 `restrict` 仅限制全局工具，本 Demo 使用作用域本地工具，设为空全局允许集；不把本地工具名错误传入全局限制接口。
