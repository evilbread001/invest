# 资料索引

整理日期：2026-10-19。

本目录作为后续项目经历整理、简历提炼的主维护位置。原文件保留，未移动源码、未删除旧资料。旧文件不会自动同步更新；以后优先编辑本目录内的材料。

## 优先阅读

1. **项目经历总档**：`01/notes.md`。包含售后客服 Agent＋知识库、产线制程诊断 Agent、压装曲线判异三个项目的材料与事实依据。
2. **产线诊断当前确认版**：`03/overview.md`。最新产品逻辑、DSH 与 BPMN/OWL 接入、对话推进、专家出口及故障树补全流程。
3. **简历文字稿**：`02/draft.md`。这是已有简历稿，本次只是归档，尚未按照最新项目材料重新提炼。

## 目录说明

| 目录 | 内容 |
|---|---|
| `01` | 三个项目的完整文字材料，后续提炼简历的主要依据 |
| `02` | 简历文字稿、两份既有 HTML 样式；样式中的内容可能落后于总档 |
| `03` | 当前确认版、引擎接入说明、Demo README 和历史实施计划 |
| `04/ref` | 压装曲线专利；项目分析正文在总档第三部分 |
| `05` | Agent 学习笔记、故障树生成质量评估方案 |
| `06` | 早期售后诊断重设计与框架理想架构，作为历史参考，不视作当前实现 |

售后客服 Agent 和知识库的详细材料已在项目经历总档第一部分，不另外生成一份重复正文。历史售后诊断方案不等于售后客服 Agent 项目，也不等于本次产线诊断当前实现。

## 使用规则

- 当前确认版优先于历史方案；实现状态、生产状态和未来建议按文中标记区分。
- 不同日期的测试数量和模型权限问题是阶段记录，不相互覆盖。用户反馈的“接口不存在”尚未定位，不当作已修复。
- 本目录的代码路径仍指向原仓库，以便后续核对实现。代码和运行环境未复制到这里。
- `03/readme.md` 是仓库说明的归档副本，启动命令应在 `/Users/gerui1/production-diagnosis-dsh-demo` 执行，不是在本材料目录执行。
- 本次不归档 API Key、`.env`、生产会话数据、其他个人证件或财税材料，也未重新下载飞书原始附件。

## 来源清单（16 份既有文件）

- `/Users/gerui1/project_experience_notes.md` → `01/notes.md`
- `/Users/gerui1/resume_content.md` → `02/draft.md`
- `/Users/gerui1/Downloads/resume_style1.html` → `02/layout-01.html`
- `/Users/gerui1/Downloads/resume_style2.html` → `02/layout-02.html`
- `/Users/gerui1/production-diagnosis-dsh-demo/docs/current-version-2026-10-19.md` → `03/overview.md`
- `/Users/gerui1/production-diagnosis-dsh-demo/docs/engine-integration.md` → `03/details.md`
- `/Users/gerui1/production-diagnosis-dsh-demo/README.md` → `03/readme.md`
- `/Users/gerui1/production-diagnosis-dsh-demo/docs/superpowers/plans/2026-10-19-tree-completion.md` → `03/log/note-01.md`
- `/Users/gerui1/production-diagnosis-dsh-demo/docs/superpowers/plans/2026-10-19-collaborative-diagnosis.md` → `03/log/note-02.md`
- `/Users/gerui1/production-diagnosis-dsh-demo/docs/superpowers/plans/2026-10-16-demo.md` → `03/log/note-03.md`
- `/Users/gerui1/production-diagnosis-dsh-demo/docs/superpowers/plans/2026-10-19-supplemental-checks.md` → `03/log/note-04.md`
- `/Users/gerui1/Downloads/CHJ2504000697CN01-技术交底书-智能工业-专利技术交底书-压装曲线.docx` → `04/ref/attachment.docx`
- `/Users/gerui1/agent_learning_notes.md` → `05/note-01.md`
- `/Users/gerui1/fault_tree_eval_proposal.md` → `05/note-02.md`
- `/Users/gerui1/aftersale_diagnosis_agent_redesign.md` → `06/note-01.md`
- `/Users/gerui1/agent_ideal_architecture.md` → `06/note-02.md`
