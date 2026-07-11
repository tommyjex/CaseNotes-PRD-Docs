# 心理咨询个案助手 App 开发流程与技术选型计划

## Summary

本计划面向一个从 0 到 1 的心理咨询个案助手手机 App，覆盖 Android 与 iOS 两端。当前仓库 `/Users/bytedance/CaseNotes-Frontend` 为空目录且不是 Git 仓库，因此建议先完成产品定义、Flutter 工程初始化、flutter_easy_ui 设计系统、云端服务与 AI 工作流设计，再进入 MVP 迭代开发。

已确认的关键方向：

- 第一阶段目标：MVP 验证。
- 数据策略：云端优先。
- 客户端框架：Flutter。
- UI 框架：flutter_easy_ui。
- 设计协作：弃用 Figma，直接在 Flutter 内基于 flutter_easy_ui 完成设计规范、组件样例页和页面落地。

MVP 的核心闭环建议定义为：咨询师创建个案 -> App 内录音或上传已有录音文件 -> 调用云端 API 转写 -> 咨询师编辑确认转写文本 -> 结构化整理 -> AI 生成摘要/风险提示/报告草稿 -> 咨询师确认与编辑 -> 生成跟进提醒。

## Current State Analysis

### 仓库状态

- 当前目录为空：没有现有 Flutter 工程、README、接口文档或设计规范。
- 当前目录不是 Git 仓库：后续应先初始化 Git、基础工程和协作规范。
- 没有发现既有后端、数据库、Flutter 设计规范文件或 API schema。

### 产品风险与约束

心理咨询记录属于高度敏感数据，MVP 即使不追求完整商业化上线，也需要把以下约束前置：

- 用户隐私：咨询记录、个案资料、风险提示和报告都应默认按敏感数据处理。
- 权限边界：MVP 可先做单咨询师账号，但数据模型要保留后续机构/督导/多角色扩展空间。
- AI 辅助定位：风险提示和报告应定位为“辅助草稿/提醒”，不能替代专业诊断或危机干预判断。
- 审计与可追溯：AI 输出应保留输入版本、模型版本、生成时间、人工确认状态。
- 跨端一致性：Android/iOS 使用同一 Flutter 代码库，平台差异主要集中在推送、文件权限、录音权限和应用上架合规。

## Proposed Changes

### 1. 产品与迭代流程

建议采用 4 个阶段推进。

#### 阶段 0：需求澄清与边界定义

产出物：

- MVP PRD。
- 信息架构图。
- 核心用户旅程。
- 数据分类与隐私分级表。
- 风险提示免责声明与人工确认流程。

建议锁定的 MVP 功能范围：

- 账号登录与基础个人资料。
- 个案管理：新建、编辑、搜索、归档。
- 咨询记录：录音和上传音频为核心入口，调用云端 API 生成转写文本，同时支持手动文本补录；支持标签、主诉、来访目标、干预方式、下次计划等结构化字段。
- AI 整理：将原始记录整理成 SOAP/DAP 等可配置模板。
- AI 总结：生成本次摘要、阶段性摘要、报告草稿。
- 风险提示：基于规则 + LLM 辅助识别自伤、自杀、暴力、虐待、严重危机等风险线索。
- 跟进提醒：按个案、咨询记录或风险等级生成待办提醒。

建议暂缓到二期的能力：

- 完整机构管理后台。
- 会中实时字幕式转写，MVP 先支持录音结束后上传云端转写。
- 督导协作。
- 复杂量表库。
- 多模型自动评测平台。
- 完整电子签名和付费订阅。

#### 阶段 1：Flutter 设计系统与交互样例

Flutter 侧直接负责设计和落地：

- App 页面样例：登录、首页、个案列表、个案详情、录音、上传音频、转写任务、咨询记录编辑、AI 结果确认、报告预览、提醒中心、设置。
- 设计系统：颜色、字体、间距、圆角、阴影、图标、表单、按钮、标签、状态提示，统一沉淀在 Flutter 主题和组件代码中。
- 组件状态：加载、空状态、错误、风险等级、AI 生成中、人工已确认。
- 设计 Token：以 Dart 常量、ThemeExtension 和 flutter_easy_ui 主题配置管理。

Flutter 侧承接方式：

- 用 flutter_easy_ui 承载基础组件风格。
- 将颜色、字号、间距、圆角、风险等级色沉淀为 Flutter ThemeExtension 和 App 级设计常量。
- 建立 `app_theme`、`app_tokens`、`app_components` 三层：
  - `app_tokens`：Dart 原子设计值，如颜色、间距、圆角、字号。
  - `app_theme`：Flutter ThemeData 与 flutter_easy_ui 主题适配。
  - `app_components`：业务组件，如风险标签、咨询记录卡片、AI 结果确认卡。
- 建立 `design_gallery` 或组件样例页，用真机/模拟器直接验证样式，而不是通过 Figma 对稿。

#### 阶段 2：Flutter MVP 工程开发

推荐客户端技术选型：

- Flutter：统一 Android/iOS 代码库。
- Dart：Flutter 默认语言。
- flutter_easy_ui：基础 UI 组件与样式体系。
- Riverpod：状态管理，适合模块化、可测试、依赖注入清晰。
- go_router：路由与深链管理。
- dio：HTTP 客户端。
- retrofit.dart 或 chopper：接口声明式封装，降低手写请求成本。
- freezed + json_serializable：不可变数据模型与 JSON 序列化。
- drift 或 sqflite：本地缓存与离线草稿。
- flutter_secure_storage：保存 token、密钥片段等敏感本地凭据。
- record 或 flutter_sound：App 内录音，具体选型需结合 iOS/Android 权限、后台行为和格式支持验证。
- file_picker：上传已录制好的音频文件。
- permission_handler：管理麦克风、文件访问等系统权限。
- firebase_messaging / APNs / 厂商推送适配：提醒推送，具体服务按部署区域和上架要求确定。
- sentry_flutter 或同类方案：崩溃与性能监控。

推荐 Flutter 目录结构：

```text
lib/
  app/
    router/
    theme/
    bootstrap/
  core/
    network/
    storage/
    auth/
    errors/
    utils/
  features/
    auth/
    audio/
    cases/
    session_notes/
    ai_assistant/
    reports/
    reminders/
    settings/
  shared/
    widgets/
    models/
    constants/
```

各 feature 内建议按 `data / domain / presentation` 分层：

- `data`：API、DTO、本地缓存。
- `domain`：实体、用例、业务规则。
- `presentation`：页面、组件、Riverpod provider。

#### 阶段 3：云端服务与 AI 工作流

MVP 推荐采用“模块化单体后端 + 可替换 AI Provider”的方式，避免一开始拆成过多微服务。

推荐后端技术选型：

- API 服务：NestJS 或 FastAPI。
- 推荐优先级：若团队前后端偏 TypeScript，选 NestJS；若 AI/算法同学深度参与，选 FastAPI。
- 数据库：PostgreSQL。
- 缓存与任务队列：Redis。
- 异步任务：BullMQ/NestJS Queue 或 Celery/RQ，取决于后端语言。
- 对象存储：保存录音文件、导出的 PDF 报告和附件，录音文件必须使用私有桶和短期签名访问。
- 认证：JWT + Refresh Token，生产阶段可接入 OAuth/手机号/企业身份。
- API 文档：OpenAPI/Swagger。
- 部署：Docker Compose 起步，后续迁移到 Kubernetes 或云托管平台。

核心后端模块：

- Auth：账号、会话、权限。
- Case：个案基础资料、标签、状态。
- SessionNote：咨询记录、结构化字段、版本。
- AudioAsset：录音文件、上传状态、格式、时长、对象存储地址。
- TranscriptionJob：云端转写任务、供应商、状态、错误码。
- Transcript：转写文本、人工编辑状态、确认状态。
- AIJob：AI 任务、输入快照、输出、状态、模型信息。
- RiskAssessment：风险提示、风险等级、证据片段、人工确认。
- Report：报告模板、生成记录、导出文件。
- Reminder：跟进提醒、推送状态、完成状态。
- AuditLog：敏感操作审计。

AI 工作流建议：

- 第一步：录音文件上传到私有对象存储。
- 第二步：创建云端转写任务，轮询或回调获取转写文本。
- 第三步：咨询师查看并编辑转写文本。
- 第四步：规则预筛，识别明确风险关键词、空内容、超长内容、敏感字段。
- 第五步：LLM 结构化整理，输出固定 JSON schema。
- 第六步：后端校验 JSON schema，失败则重试或进入人工编辑。
- 第七步：风险提示独立生成，不与普通摘要混在同一个输出里。
- 第八步：咨询师人工确认后，才把 AI 结果标记为正式摘要/报告内容。

AI 输出建议拆成 4 类：

- `structured_note`：结构化咨询记录。
- `session_summary`：本次咨询摘要。
- `risk_hint`：风险提示与证据片段。
- `follow_up_suggestion`：跟进建议与提醒草稿。

#### 阶段 4：测试、验收与试点

MVP 验收建议围绕“专业可用性 + 数据安全 + 端到端闭环”。

测试策略：

- Flutter 单元测试：domain use case、数据模型、录音/转写状态、风险等级映射。
- Flutter Widget 测试：表单、记录卡片、AI 结果确认组件。
- API 集成测试：个案、音频上传、转写任务、记录、AI 任务、提醒。
- AI schema 测试：固定样例输入必须输出合法 JSON。
- 安全测试：越权访问、token 过期、敏感日志、导出文件权限。
- 人工验收：邀请 3-5 位咨询师用脱敏样例跑完整流程。

MVP 成功标准：

- 咨询师能完成录音或上传音频，并在转写完成后编辑确认咨询记录。
- AI 能稳定生成可编辑的结构化摘要和报告草稿。
- 风险提示能给出风险等级、依据片段和人工确认入口。
- 跟进提醒能从咨询记录或 AI 建议生成并按时提醒。
- 所有敏感数据传输和存储有基础加密措施。
- AI 输出不会直接覆盖人工内容，必须经过确认。

## Recommended Technical Architecture

### 总体架构

```text
Flutter App (Android/iOS)
  -> API Gateway / Backend API
    -> PostgreSQL
    -> Redis Queue
    -> Object Storage
    -> AI Provider Gateway
      -> LLM / Embedding / Moderation Provider
```

### 客户端推荐栈

| 领域 | 推荐选型 | 原因 |
| --- | --- | --- |
| 跨端框架 | Flutter | Android/iOS 一套代码，适合 MVP 快速验证 |
| UI 框架 | flutter_easy_ui | 符合用户指定方向，用于统一组件风格 |
| 状态管理 | Riverpod | 可测试、依赖注入清晰、适合 feature 模块化 |
| 路由 | go_router | 官方生态成熟，支持嵌套路由和深链 |
| 网络 | dio + retrofit.dart | 拦截器、重试、错误处理和接口声明更清晰 |
| 数据模型 | freezed + json_serializable | 减少样板代码，提升类型安全 |
| 本地缓存 | drift 或 sqflite | 支持离线草稿和列表缓存 |
| 安全存储 | flutter_secure_storage | 存储 token 等敏感凭据 |
| 录音 | record 或 flutter_sound | 支持 App 内录音，需真机验证权限和格式 |
| 文件选择 | file_picker | 支持上传已有录音文件 |
| 权限 | permission_handler | 管理麦克风、文件访问权限 |
| 监控 | sentry_flutter | 崩溃、性能、错误追踪 |

### 后端推荐栈

| 领域 | 推荐选型 | 原因 |
| --- | --- | --- |
| API 服务 | NestJS 或 FastAPI | NestJS 适合工程化 API，FastAPI 适合 AI 团队快速协作 |
| 数据库 | PostgreSQL | 关系数据、JSONB、审计、查询能力均衡 |
| 缓存/队列 | Redis | AI 任务异步化、提醒任务、短期缓存 |
| 文件 | 对象存储 | 录音文件、PDF 报告、附件 |
| 转写 | 云端 Speech-to-Text API | 支持录音转写，建议异步任务化 |
| API 契约 | OpenAPI | 前后端并行开发和接口 Mock |
| 部署 | Docker 起步 | 方便本地、测试、生产环境一致 |

### AI 推荐策略

MVP 不建议一开始训练私有模型，推荐使用成熟 LLM API + 强 schema 约束 + 人工确认。

关键设计：

- Prompt 模板版本化。
- AI 输出必须是 JSON schema。
- 风险提示单独链路处理。
- 每次生成保存输入摘要、输出、模型、prompt 版本、人工确认状态。
- 对心理危机类风险提示增加免责声明和紧急资源提示。

### 数据模型初稿

核心实体：

- User：咨询师账号。
- ClientCase：个案。
- SessionNote：咨询记录。
- AudioAsset：录音文件。
- TranscriptionJob：转写任务。
- Transcript：转写文本。
- AIJob：AI 生成任务。
- AIResult：AI 输出结果。
- RiskHint：风险提示。
- Report：报告。
- Reminder：跟进提醒。
- AuditLog：审计日志。

关键关系：

- 一个 User 拥有多个 ClientCase。
- 一个 ClientCase 拥有多个 SessionNote、Report、Reminder。
- 一个 SessionNote 可触发多个 AIJob。
- 一个 AIJob 生成一个或多个 AIResult。
- RiskHint 应绑定到 SessionNote 或 Report，并保留证据片段。

## Assumptions & Decisions

### 已确定

- MVP 优先，不在第一期追求完整机构版。
- 云端优先，支持多端同步和服务端 AI。
- Flutter 作为 Android/iOS 统一客户端技术。
- UI 使用 flutter_easy_ui，并在 Flutter 代码内沉淀设计系统和组件规范。

### 建议决策

- MVP 以“录音/上传音频 -> 云端转写 -> 编辑确认文本”为核心入口，文本输入作为补录和编辑方式。
- MVP 后端先做模块化单体，不急于微服务化。
- AI 输出必须走“生成草稿 -> 人工确认 -> 正式入库”的流程。
- 风险提示不作为诊断结论，只作为辅助提示。
- 所有敏感记录默认加密传输，生产环境启用数据库加密、对象存储私有桶和审计日志。

### 待后续确认

- 目标市场与合规要求：国内、海外，或仅内部试点。
- 登录方式：手机号、邮箱、企业账号，还是第三方身份。
- AI Provider：内部模型、火山方舟、OpenAI、Claude，或可插拔多供应商。
- 转写 Provider：云端 Speech-to-Text API，需确认格式、时长、并发、价格和数据保留策略。
- 报告模板：SOAP、DAP、BIRP、自定义模板，或国内咨询机构常用模板。
- 音频保留策略：原始音频长期保留、用户手动删除，还是转写完成后自动清理。
- 是否需要与日历、飞书、微信、系统日历联动提醒。

## Suggested Roadmap

### 第 1-2 周：产品与设计准备

- 完成 MVP PRD。
- 明确数据字段、报告模板、风险等级定义。
- 完成 Flutter 组件样例页和关键流程静态页面，包括录音、上传、转写状态和失败重试。
- 完成技术架构评审。

### 第 3-4 周：工程基础

- 初始化 Flutter 工程。
- 接入 flutter_easy_ui、Riverpod、go_router、dio。
- 建立主题、Dart 设计 Token、基础组件和组件样例页。
- 初始化后端 API、PostgreSQL、Redis、OpenAPI。
- 完成 Auth、Case、AudioAsset、TranscriptionJob、Transcript、SessionNote 基础接口。

### 第 5-6 周：核心业务闭环

- 完成个案管理、录音/上传、云端转写、咨询记录、记录详情。
- 完成转写任务和 AI 任务创建、轮询、结果展示。
- 完成结构化整理和摘要生成。
- 完成风险提示结果卡片与人工确认。

### 第 7-8 周：报告、提醒与验收

- 完成报告草稿生成与导出。
- 完成跟进提醒和推送。
- 完成安全检查、AI 输出样例测试。
- 组织咨询师试点和反馈收集。

## Verification Steps

计划阶段验证：

- 检查 MVP PRD 是否覆盖核心闭环：录音/上传、转写、记录、整理、总结、风险提示、报告、提醒。
- 检查 flutter_easy_ui 主题、Dart 设计 Token 和 Flutter 业务组件是否覆盖关键页面状态。
- 检查 API schema 是否支持前后端并行开发。
- 检查风险提示是否有人工确认和免责声明。

开发阶段验证：

- Flutter 执行 `flutter analyze` 和单元测试。
- 后端执行 API 集成测试和 schema 校验测试。
- AI 任务用脱敏样例跑批测试，验证 JSON 输出稳定性。
- 通过 Android/iOS 真机测试录音、上传、转写、AI 结果、报告、提醒链路。
- 使用安全 checklist 检查敏感数据日志、权限、存储和传输。

试点阶段验证：

- 至少 3-5 位目标咨询师完成真实工作流试用。
- 收集 AI 摘要可用率、风险提示误报/漏报反馈、报告编辑耗时。
- 记录最影响留存的 3 个问题，作为下一轮迭代输入。

## Immediate Next Steps

1. 先确认本计划中的 MVP 范围是否符合预期。
2. 再补充目标市场、登录方式、AI Provider、转写 Provider、报告模板和音频保留策略。
3. 之后可进入 PRD、Flutter 页面清单、组件样例页、Flutter 工程初始化和后端 API schema 设计。
