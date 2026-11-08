# 资料索引

整理日期：2026-10-19；目录名称更新：2026-11-08。

维护位置确认（2026-10-19）：后续材料统一更新在 `/Users/gerui1/invest-/personal_docs` 下，现有正文位于其中的 `个人书面材料` 目录。采用编号加英文用途的目录名及中性文件名，不再更新之前位于主目录或 Documents 下的材料副本；原代码仓库路径仅作为实现参考。

本目录作为后续项目经历整理、简历提炼的主维护位置。原文件保留，未移动源码、未删除旧资料。旧文件不会自动同步更新；以后优先编辑本目录内的材料。

## 优先阅读

- **三个项目口述稿合集**：`05-study-and-practice/project-walkthroughs-v1-2026-11-08.md`。含完整中英文版本，后续统一在此维护。

1. **项目经历总档**：`01-project-notes/notes.md`。包含售后客服 Agent＋知识库、产线制程诊断 Agent、压装曲线判异三个项目的材料与事实依据。客服部分于 2026-11-08 按最新本地仓库更新：第 2—3 节讲上下文、跨轮检索、多设备草稿与业务回执；第 5.2—5.8 节讲 L0/L1/L3 评测、模拟用户、字段 GT、DSH 失败分析及线上质检。
2. **产线诊断当前确认版**：`03-diagnosis-demo/overview.md`。最新产品逻辑、DSH 与 BPMN/OWL 接入、对话推进、专家出口及故障树补全流程。
3. **当前文字稿**：`02-profile-drafts/Agent开发-v2-2026-10-19.md`。面向 Agent 开发岗位，基于项目总档和产线诊断当前确认版重新提炼；仅 Markdown，不生成 HTML。`02-profile-drafts/draft.md` 与两份既有 HTML 保留为旧版。
4. **方向与岗位观察**：`05-study-and-practice/note-03-2026-10-19.md`。北京、上海、深圳的外企工业 AI 相关岗位快照、官方入口及匹配判断；当前偏好不限于 Agent 开发。
5. **平台观察与沟通顺序**：`05-study-and-practice/note-04-2026-10-19.md`。BOSS、猎聘的国内企业岗位线索、核验边界和先后沟通建议。
6. **项目问答手册 V1**：`05-study-and-practice/note-05-v1-2026-10-19.md`。产线诊断16题、客服/RAG20题、压装12题、跨项目4题；含优先12题、口述答案、继续追问和两套练习路线。

### 当前文字稿维护说明

- 文件命名采用“面向岗位-v版本-日期”；后续新方向另建版本，不覆盖其他方向。
- V2 聚焦售后客服与知识库、产线制程诊断、压装曲线判异三个项目，保留内测与协作原型的必要状态说明。
- 公司全称在旧稿中未填写，V2 保留一处待补；任职起始时间、联系方式与教育经历沿用旧稿，发送前核对。
- 客服拦截率按内部模拟评测口径表述，不改成线上解决率；压装采用已有部署覆盖与专利成果，未将旧稿中待核的 99.5% 写成实测结论。

## 目录说明

| 目录 | 内容 |
|---|---|
| `01-project-notes` | 三个项目的完整文字材料，后续提炼简历的主要依据 |
| `02-profile-drafts` | 当前岗位方向文字稿及旧版存档；已有 HTML 未随 V2 更新 |
| `03-diagnosis-demo` | 当前确认版、引擎接入说明、Demo README 和历史实施计划 |
| `04-patent-materials/ref` | 压装曲线专利；项目分析正文在总档第三部分 |
| `05-study-and-practice` | 项目问答、Agent 学习笔记、故障树生成质量评估方案、方向及平台观察 |

售后客服 Agent 和知识库的详细材料已在项目经历总档第一部分，不另外生成一份重复正文。历史售后诊断方案不等于售后客服 Agent 项目，也不等于本次产线诊断当前实现。

客服本轮仅更新技术底稿，不覆盖口述合集或岗位文字稿。当前本地实现、已部署能力与历史评测结果在总档中分别记录；55% 的当前讲述口径与独立 300 条约 50% 不合并。评测新增本地文件不视为已发布，未运行生产评测或写入客户系统。

## 使用规则

- 当前确认版优先于历史方案；实现状态、生产状态和未来建议按文中标记区分。
- 不同日期的测试数量和模型权限问题是阶段记录，不相互覆盖。用户反馈的“接口不存在”尚未定位，不当作已修复。
- 本目录的代码路径仍指向原仓库，以便后续核对实现。代码和运行环境未复制到这里。
- `03-diagnosis-demo/readme.md` 是仓库说明的归档副本，启动命令应在 `/Users/gerui1/production-diagnosis-dsh-demo` 执行，不是在本材料目录执行。
- 本次不归档 API Key、`.env`、生产会话数据、其他个人证件或财税材料，也未重新下载飞书原始附件。

## 来源清单（14 份保留的既有文件）

- `/Users/gerui1/project_experience_notes.md` → `01-project-notes/notes.md`
- `/Users/gerui1/resume_content.md` → `02-profile-drafts/draft.md`
- `/Users/gerui1/Downloads/resume_style1.html` → `02-profile-drafts/layout-01.html`
- `/Users/gerui1/Downloads/resume_style2.html` → `02-profile-drafts/layout-02.html`
- `/Users/gerui1/production-diagnosis-dsh-demo/docs/current-version-2026-10-19.md` → `03-diagnosis-demo/overview.md`
- `/Users/gerui1/production-diagnosis-dsh-demo/docs/engine-integration.md` → `03-diagnosis-demo/details.md`
- `/Users/gerui1/production-diagnosis-dsh-demo/README.md` → `03-diagnosis-demo/readme.md`
- `/Users/gerui1/production-diagnosis-dsh-demo/docs/superpowers/plans/2026-10-19-tree-completion.md` → `03-diagnosis-demo/log/note-01.md`
- `/Users/gerui1/production-diagnosis-dsh-demo/docs/superpowers/plans/2026-10-19-collaborative-diagnosis.md` → `03-diagnosis-demo/log/note-02.md`
- `/Users/gerui1/production-diagnosis-dsh-demo/docs/superpowers/plans/2026-10-16-demo.md` → `03-diagnosis-demo/log/note-03.md`
- `/Users/gerui1/production-diagnosis-dsh-demo/docs/superpowers/plans/2026-10-19-supplemental-checks.md` → `03-diagnosis-demo/log/note-04.md`
- `/Users/gerui1/Downloads/CHJ2504000697CN01-技术交底书-智能工业-专利技术交底书-压装曲线.docx` → `04-patent-materials/ref/attachment.docx`
- `/Users/gerui1/agent_learning_notes.md` → `05-study-and-practice/note-01.md`
- `/Users/gerui1/fault_tree_eval_proposal.md` → `05-study-and-practice/note-02.md`
