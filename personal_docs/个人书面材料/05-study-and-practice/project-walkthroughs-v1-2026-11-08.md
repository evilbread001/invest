# Project Walkthroughs / 项目口述稿合集

更新日期：2026-11-08。三个项目的口述稿统一在本文件维护，不再保留独立分篇。

## Contents / 目录

1. Production Diagnosis Agent / 产线诊断 Agent
2. Customer Service Agent and RAG / 售后客服 Agent 与 RAG
3. Press-fit Curve Anomaly Detection / 压装曲线判异
4. Reference Notes / 材料备注（不纳入口述正文）

## 1. Production Diagnosis Agent / 产线诊断 Agent

### 中文完整版

这个项目主要面向制造现场的质量工程师。过去出现制程质量问题后，工程师需要从多个系统查询和分析生产、检测数据，再依赖专家经验逐项排查。我们的目标是自动完成这些系统数据的查询与分析，并将专家的排查经验沉淀为结构化故障树，为后续问题诊断提供指导。

产品以异常工单为入口，根据故障现象和工艺信息匹配相应故障树，再按照其中的排查逻辑推进诊断。能够通过数据完成的检查由系统自动执行，需要现场确认的步骤则生成任务交给工程师，系统根据反馈继续排查，最终汇总诊断结论和报告。同时，经过确认的新经验可以纳入故障树的后续版本，用于同类问题的排查。

我在项目中担任工程侧负责人，负责推进整个产品的工程化落地和持续迭代。在诊断执行侧，我基于 BPMN 开发诊断流程，打通工单触发、自动分析、人工反馈和恢复执行；将 3σ 判异、共性分析等通用算法模型封装为可复用组件，并结合分析结果和排查记录，实现诊断报告自动生成。

在诊断知识侧，我负责故障树的生命周期管理，覆盖草稿编辑、校验和版本发布；同时基于 DSH、Skill 和 Tool Calling 开发大模型自动建树模块，从 8D 报告、历史工单、FMEA 文档中提取失效模式、原因关系和排查步骤，联动对话式编辑器，完成从自动生成、交互修订到人工审核发布的流程。

目前产品已经接入 8 家客户，覆盖超过 20 条产线。项目将系统数据查询分析、专家经验排查和诊断报告生成串联起来，减少工程师跨系统查数和手工整理的工作，并通过故障树的构建与持续维护，把依赖个人的排查经验沉淀为可复用的诊断知识。按当前业务工作量测算，预计平均每家客户每周可净节省约 **18 个工程师人时**。

### Full English Version

This project serves quality engineers in manufacturing. Previously, when a process-quality issue occurred, engineers had to retrieve and analyze production and inspection data from multiple systems, then investigate possible causes based on expert experience. Our goal was to automate those data queries and analyses, and capture expert diagnostic experience in structured fault trees to guide future investigations.

The product is driven by quality-issue tickets. It identifies the relevant fault tree based on the reported issue and process context, then follows the diagnostic steps in that tree. Checks that can be performed using data are executed automatically, while those requiring on-site inspection are presented as tasks for engineers. Their feedback guides the next step, and the findings are consolidated into a diagnostic report. Newly confirmed knowledge can also be incorporated into subsequent fault-tree versions for similar issues.

I was the engineering lead, responsible for the product’s engineering implementation and ongoing iterations. On the diagnostic execution side, I developed the BPMN-based workflow, connecting ticket triggers, automated analysis, engineer feedback, and execution resumption. I also encapsulated general-purpose analysis methods, including three-sigma anomaly detection and commonality analysis, into reusable components, and implemented automatic report generation from analysis results and investigation records.

On the diagnostic knowledge side, I developed fault-tree lifecycle management, covering draft editing, validation, and version publication. I also built the automatic fault-tree generation module using DSH, Skills, and Tool Calling. It extracts failure modes, causal relationships, and diagnostic steps from 8D reports, historical tickets, and FMEA documents, and connects automatic generation with conversational editing and human-reviewed publication.

The product currently serves eight customers across more than twenty production lines. It connects data retrieval and analysis, expert-guided investigation, and diagnostic reporting, reducing manual cross-system data gathering and report preparation. Through fault-tree development and ongoing maintenance, it turns individual expertise into reusable diagnostic knowledge. Based on current workload estimates, the projected average net saving is approximately **18 engineering person-hours per customer per week**.

## 2. Customer Service Agent and RAG / 售后客服 Agent 与 RAG

### 中文完整版

希沃原本已经有一套客服智能问答 Agent，但拦截率较低，替代人工、降本增效的作用不够明显，同时知识库比较散乱。我们的目标是在现有系统基础上，通过知识库重构和对话能力优化提升拦截率，并打通维修填单等功能，让 Agent 不只是回答问题，还能承接后续服务流程。

产品从用户消息出发，识别咨询意图；用户上传图片时，通过多模态模型提取设备 SN 和故障线索，再结合设备信息和故障描述检索适用知识，提供排查指引。根据用户反馈，Agent 继续排查，或者进入维修填单流程，通过多轮对话收集信息并同步到业务系统，必要时转交人工客服。

我主要负责售后 RAG 知识库及入库、检索链路的从 0 到 1 建设。我将分散的原始资料整理为结构化知识卡，完成向量化和增量更新；基于 PostgreSQL、pgvector，围绕标题、相似问题、故障描述和解决方案四个字段，构建向量与字面匹配的八路召回，通过加权 RRF 融合排序，并支持 Cross-Encoder 重排、产品范围筛选和已失败方案排除，为 Agent 提供适用的知识依据。

此外，我负责故障分流和维修填单相关的 Skill 开发，实现多轮字段提取、缺失项追问，以及同一会话中多台设备的信息处理。通过结构化业务状态和工具执行结果，维护当前设备、已收集信息和填单进度，并区分本地草稿与业务接口同步成功的结果，减少重复追问和设备信息混用。

在效果验证方面，我参与搭建了知识库检索和端到端对话两层评测体系。检索层面，评估用户问题能否命中适用知识卡及相关结果的排序表现，区分检索失败与知识缺失。端到端层面，基于 1000 条真实客服会话，利用另一个 Agent 提取用户意图并模拟多轮交互，评估拦截率，同时检查知识使用、工具调用和填单字段采集情况。我负责分析 Bad Case，并通过回归测试验证知识库和 Skill 更新的效果。

项目目前处于内测阶段。通过知识库、业务 Skill 和对话流程的持续迭代，系统在内部模拟评测中的拦截率由约 13% 提升至 55%。这里的拦截率指用户未选择转人工的会话占比，不等同于已验证的问题解决率。项目的整体价值，是在已有智能问答基础上，进一步打通知识支持的故障排查与维修填单流程，提升 Agent 独立承接售后服务的能力。

### Full English Version

Seewo already had an intelligent customer-service agent, but its no-handoff rate was low, limiting its ability to reduce manual support workload and costs. Its knowledge base was also fragmented. Our goal was to upgrade the existing system by improving knowledge retrieval and conversation handling, increasing the no-handoff rate, and integrating repair-ticket collection so the agent could support service workflows beyond question answering.

The agent processes customer messages and identifies the intent. When customers upload images, a vision-language model extracts device serial numbers and fault clues. The system uses the device information and reported symptoms to retrieve applicable knowledge and guide troubleshooting. Based on customer feedback, it continues the investigation, collects repair-ticket details and synchronizes them with the business system, or transfers the conversation to human support when necessary.

My main responsibility was building the after-sales RAG knowledge base and its ingestion and retrieval pipeline from the ground up. I converted fragmented source material into structured knowledge cards and implemented embedding and incremental updates. Using PostgreSQL and pgvector, I combined vector search and literal matching across four fields—titles, similar questions, fault descriptions, and solutions—to create eight retrieval channels. I used weighted RRF to fuse their rankings, with support for cross-encoder reranking, product-scope filtering, and exclusion of previously unsuccessful solutions.

I also developed business Skills for fault triage and repair-ticket collection. This included extracting fields across multiple turns, asking for missing information, and handling multiple devices within one conversation. By combining structured business state with tool execution results, the agent could track the current device, collected information, and ticket progress, while distinguishing locally collected drafts from successful updates to the business system.

I also worked on building a two-level evaluation framework. At the retrieval level, we evaluated whether customer queries retrieved applicable knowledge cards and how relevant results were ranked, helping distinguish retrieval failures from gaps in the knowledge base. At the end-to-end level, we used 1,000 real customer conversations as source material and a separate agent to extract user intent and simulate multi-turn interactions. We assessed the no-handoff rate and reviewed knowledge use, tool calls, and ticket-field collection. I analyzed bad cases and used regression testing to check the effects of knowledge-base and Skill updates.

The project is currently in internal testing. Through iterations across the knowledge base, business Skills, and conversation handling, the system’s no-handoff rate in internal simulated evaluations increased from approximately 13% to 55%. This measures the proportion of conversations in which the user did not choose a human handoff, rather than a verified resolution rate. The broader value was extending the existing agent from question answering to knowledge-grounded troubleshooting and repair-ticket collection.

## 3. Press-fit Curve Anomaly Detection / 压装曲线判异

### 中文完整版

这个项目面向压缩机压装过程的质量监测。传统方式主要检查最终压力、位移，或者依赖人工配置的判异框。但终值达标不代表整个压装过程正常，零件或型号变化后，也可能需要反复调整判异边界。我们的目标是通过完整的压力—位移曲线识别过程异常，减少对人工配置判异边界的依赖。

产品采集压装曲线并进行预处理，再通过互补的方法，分别检查关键区间是否越界、局部阶段特征是否异常，以及整体曲线是否偏离正常模式。发现可疑曲线后，系统通过异常卡片将原始曲线和零件信息提供给现场人员复判。

我主要负责算法研发。在统计和特征判异方面，我通过关键点识别与 3σ 标定优化监控区间，将曲线划分为五个阶段，提取峰谷值、斜率、区间长度、面积及标准差等特征，再通过孤立森林识别异常特征组合，补充单纯依赖固定判异框的局限。

在整体序列判异方面，我基于 PyTorch 开发 LSTM 自编码器，对曲线进行长度统一，使用正常压力—位移序列训练模型，通过重构误差识别偏离正常模式的曲线。整体思路是结合明确的区间规则、局部阶段特征和整体序列建模，而不是只依赖一个深度学习模型。

项目整体覆盖 3 条线体、9 个压装点位，采用异常推卡、人工复判的方式使用，当前项目材料记录的平均准确率为 99.5% 以上。相关方案申请了发明专利《一种基于LSTM的压装工艺质量的曲线模型判异方法》，我是第二发明人，申请已通过初步审查。

### Full English Version

This project focuses on quality monitoring in compressor press-fitting. Traditional checks relied on final force and displacement values or manually configured monitoring envelopes. However, an acceptable endpoint does not necessarily mean the entire process was normal, and changes in parts or product models could require repeated adjustment of those boundaries. Our goal was to identify anomalies from the full force–displacement curve and reduce reliance on manually configured detection boundaries.

The system collects press-fit curves and preprocesses them before applying complementary detection methods. These methods examine whether key regions exceed expected limits, whether individual stages have unusual features, and whether the overall curve deviates from normal patterns. Suspicious curves are delivered to production staff as anomaly cards for manual review, together with the original curve and part information.

I was primarily responsible for algorithm development. For statistical and feature-based detection, I used key-point identification and three-sigma calibration to improve monitoring regions. I divided curves into five stages and extracted features such as peaks, slopes, interval lengths, areas, and standard deviations, then used Isolation Forest to identify unusual combinations of those features.

For sequence-level detection, I developed an LSTM autoencoder in PyTorch. I standardized the curve lengths, trained the model on normal force–displacement sequences, and used reconstruction error to detect deviations from the learned patterns. The idea was to combine explicit monitoring rules, local-stage features, and overall sequence modeling, rather than rely on a single deep-learning model.

At the project level, the solution covers three production lines and nine press-fit stations, using anomaly alerts followed by human review. The current project material reports average accuracy above 99.5%. The work also resulted in an invention patent application, A Curve-Model Anomaly Detection Method for Press-Fitting Process Quality Based on LSTM, on which I am the second inventor; the application has passed preliminary examination.

## Reference Notes / 材料备注

以下备注保留原稿口径，不纳入口述正文。

### Production Diagnosis Agent / 产线诊断 Agent

- 2026-11-07 用户明确补充：目前已接入 8 家客户，覆盖超过 20 条产线。
- 该覆盖口径替代此前拟用的“10 家客户、40+ 条产线”规模假设；本稿不沿用尚未核实的“20+ 工艺场景”。
- 本稿以覆盖规模、交付能力及预计工时收益说明成果；“18 人时／客户／周”保留为工作量测算，不表述为实测效果。
- 覆盖范围指整体产品，不据此推断 DSH 协作原型或自动建树模块已部署至全部客户、全部产线。
- 结构：第一段背景与目标；第二段产品逻辑；第三、四段个人工程职责；第五段效果与价值。

### Press-fit Curve Anomaly Detection / 压装曲线判异

- 2026-11-07 用户明确：主要负责算法，边缘端部署链路不是本人开发。部署覆盖与异常推卡属于项目整体成果，不写为个人开发贡献。
- 当前主文字稿仍有“边缘部署”个人工作条目，后续优化时需按本次明确的职责修正；本次仅保存口述稿，未修改主文字稿。
- 99.5% 沿用当前材料口径，评测样本、统计方式及各算法贡献仍待核实；不补写未经提供的消融结果。
- 多策略互补不等于所有点位均以相同融合方式上线，不由整体准确率推断某一模型的独立收益。
- 专利状态沿用用户此前内部系统查询结果，不代表本次重新查询；英文专利名称为说明性译名。
