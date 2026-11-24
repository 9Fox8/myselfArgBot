# AGENTS.md — AI 编码助手常驻上下文

> 项目：`9Fox8/myselfArgBot` — 企业级 RAG 知识库平台
> 供进入本项目的所有 AI 编码助手读取（Claude Code / CodeBuddy / Codex / WorkBuddy）。
> CodeBuddy 默认加载根目录 `CODEBUDDY.md`；无该文件时回退读取 `AGENTS.md`。
> 规则在**新会话开始时**注入——改动本文件后请开新会话，当前会话不会自动重载。
> 最后更新：2026-11-24（草稿，待 Fox 审签）

## 一、项目身份（不可漂移）

**企业级 RAG 知识库平台** — Fox 的 Java→AI 转型第二个简历项目。
定位：Java 后端底色 × AI 工程能力，**不换语言**；与「AI 面试辅助平台」形成互补——那个讲 RAG 应用，这个讲 RAG 的工程治理（权限 / 评估 / 增量 / 审计）。
第一质量目标不是「功能全」，而是**每个设计取舍都能在面试里讲清楚**。

## 二、技术栈锁定（未经批准不得升级/替换）

> ⚠️ 下表为草稿默认值，标「待确认」的项需 Fox 拍板后才冻结为红线。

| 项 | 锁定版本 | 红线 | 状态 |
|---|---|---|---|
| JDK | 21 | 不得降到 17（records / 虚拟线程是讲解点） | ✅ |
| Spring Boot | 4.0.0 | 不混用 3.x 依赖坐标 | ✅ |
| Spring AI | 2.0.1 | starter 用 `spring-ai-starter-model-*`，勿用 1.x 旧命名 | ✅ |
| 向量库 | PostgreSQL + pgvector | dev 用 H2 + SimpleVectorStore；**检索层权限过滤必须在向量查询时生效** | ⏳ 待确认 |
| 数据库 | PostgreSQL + Spring Data JPA | 不引入 MyBatis-Plus（Boot 4 兼容性未确认） | ⏳ 待确认 |
| 文档解析 | Apache Tika / PDFBox | 解析产物落对象存储或本地目录，不写回源文档 | ⏳ 待确认 |
| 权限模型 | RBAC + 资源级 ACL（预留 ReBAC） | 过滤在检索层，不在展示层 | ✅ |

## 三、分层与目录约定

```
com.fox.enterprisekb
├── config/          SpringAiConfig / VectorStoreConfig / SecurityConfig / DevInfraConfig(dev)
├── controller/      薄层：参数校验 + 协议转换，不写业务逻辑
├── service/         业务编排：DocumentService / IngestionService / RetrievalService / ChatService / AuditService
├── entity/          JPA 实体：KnowledgeDocument / DocumentChunk / AccessPolicy / QueryAudit / Tenant
├── repository/      Spring Data 仓储
├── security/        权限上下文 + 检索过滤（ACL Filter）← 企业场景核心
├── parser/          文档解析与切分（Tika / PDFBox / Chunker）
├── dto/             UploadRequest / ChatRequest / Citation / IngestionReport
├── prompt/          Prompts.java —— Prompt 唯一存放处（版本资产，可 diff）
└── eval/            检索评估（hit@1/3/5 + MRR 基线）
```

- dev profile = H2 + `SimpleVectorStore`，零外部基础设施；prod = PostgreSQL + pgvector。
- `docs/` 存原始文档与索引产物，**不进 Git**；`spec/` 是 SDD 工作流目录（见第八节）。
- `src/main/resources/` 只放配置模板，密钥走环境变量。

## 四、编码规范（强制）

1. **Controller 薄层**：只做参数校验与 DTO 转换，业务一律下沉 Service。
2. **模型交互只走 `ChatClient`**：不在 Service 里手拼 HTTP 调 OpenAI 兼容接口。
3. **Prompt 集中管理**：一律写进 `prompt/Prompts.java`，改动带版本与意图注释，禁止散落字符串字面量。
4. **RAG 上下文手动拼装**：保持「每行可解释」，不擅自改用 `QuestionAnswerAdvisor` 等黑盒封装；要改先写 ADR 并经人工确认。
5. **检索必须带权限过滤**：任何向量查询都要传 ACL 过滤条件；新增检索入口时优先检查这一条。
6. **chunk id 固定格式** `kb-<tenantId>-<docId>-<hash>-<index>`，保证重复摄入幂等与局部重建；改格式=全量重建索引。
7. **引用必带定位**：返回给前端的每条 Citation 含 `doc_id` + `chunk_id` + 原文 offset；无来源时走拒答分支。
8. **命名**：类 UpperCamel、方法/变量 lowerCamel、常量 UPPER_SNAKE；包名小写无下划线。
9. **异常处理与日志**：Service 抛业务异常带上下文；Controller 统一兜底；SLF4J 参数化日志，不打印密钥、密钥文件内容、完整文档正文。
10. **配置与数据模型同步**：新增配置项同时更新 `application.yml`、`application-dev.yml`、`README.md`；新增表/字段同步更新 entity、`docs/技术方案.md`、`spec/plan.md`。

## 五、测试要求

- 核心逻辑（解析切分、chunk 幂等、权限过滤、引用回溯、摄入重跑）必须有 JUnit 5 单测；外部依赖用 Mockito 打桩。
- **权限负向用例必测**：无权用户对无权文档的查询必须返回 0 条，且不得进入模型上下文。
- 检索相关改动必须附 hit@1/3/5 + MRR 基线对比输出，不得只凭感觉调阈值。
- **禁止删除或跳过失败测试来「让它过」**；失败先把错误原文回传，定位后再改实现。

## 六、红线（HARD NO）

- 不把 API Key / Token 写进代码、yml、日志；一律环境变量。
- 不在检索层之外做权限过滤（展示层隐藏是漏洞，不是防护）。
- 不在无来源时让模型自由发挥生成答案。
- 不在 prod 配置使用 `SimpleVectorStore` 或 H2。
- 不开 `ddl-auto: create`（仅 update）。
- 不擅自升级 Spring AI / Boot 版本来「修编译错」——按官方文档对齐 API 用法。
- 不修改用户上传的原始文档；索引可重建，源文件不可变。
- 不把向量库数据、原始文档、审计日志产物提交进 Git。
- 不在未读 `spec/` 文档的情况下直接实现大功能（小 bug 走 Fast Path 可跳过）。

## 七、验证命令（Definition of Done）

```bash
mvn test                                             # 单测全绿
mvn spring-boot:run -Dspring-boot.run.profiles=dev   # dev 档快速启动
python scripts/eval_retrieval.py                     # 检索基线对比（Phase 1 后）
```

**DoD 清单**：编译通过 ✓ ｜ 单测 PASS ✓ ｜ 权限负向用例 PASS ✓ ｜ 引用可定位 ✓ ｜ 新逻辑有单测 ✓ ｜ Prompt 改动记版本 ✓ ｜ 文档已回写 ✓ ｜ 变更摘要已输出 ✓

## 八、与 SDD 工作流的关系

`spec/` 目录是本项目的 SDD 流水线，对齐 GitHub Spec Kit + 百信银行 SPEC×Harness 双轮：

```
输入材料 → constitution → spec(SPEC四类) → clarify → checklist → plan → tasks
        → analyze → implement → verify(harness门禁) → sync/archive
```

- **大功能**（新模块、跨 3+ 文件）：走完整流程，见 `spec/README.md`。
- **小改动**（配置、单文件 bug）：Fast Path = `tasks → implement → verify`，不写 spec。
- **存量迭代**：走 `changes/[name]/proposal.md` 变更提案，审查通过后归档到主 spec。
- 三个硬卡点（spec / plan / 每任务验证后）；AI 不得未经确认连续跨阶段执行。
- **SPEC 管「做什么」，Harness 管「做到什么标准算合格」，二者缺一不可**。
