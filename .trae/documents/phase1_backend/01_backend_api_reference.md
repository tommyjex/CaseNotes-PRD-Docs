# CaseNotes 后端接口文档

## 1. 文档信息

| 字段 | 内容 |
| --- | --- |
| 项目 | CaseNotes Server |
| 文档类型 | Phase 1 后端接口文档 |
| 生成依据 | `/Users/bytedance/CaseNotes-Server` 当前 FastAPI 实现 |
| API 前缀 | `/api/v1` |
| OpenAPI 契约 | `openapi.json` |
| 目标读者 | Flutter 客户端、后端、测试、产品 |
| 更新时间 | 2026-07-11 |

## 2. 服务基础信息

当前后端使用 FastAPI 实现，应用入口为：

```text
/Users/bytedance/CaseNotes-Server/app/main.py
```

本地启动方式：

```bash
cd /Users/bytedance/CaseNotes-Server
source .venv/bin/activate
uvicorn app.main:app --reload
```

默认文档入口：

```text
Swagger UI: /api/v1/docs
ReDoc:      /api/v1/redoc
OpenAPI:   /api/v1/openapi.json
```

## 3. 认证与通用约定

### 3.1 认证方式

除登录、刷新、登出、健康检查外，业务接口均要求认证。

客户端应在请求头携带：

```http
Authorization: Bearer <access_token>
```

登录接口返回：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `access_token` | string | 短期访问令牌 |
| `refresh_token` | string | 刷新令牌 |
| `token_type` | string | 当前为 `bearer` |
| `expires_at` | datetime | access token 过期时间 |

### 3.2 数据隔离

当前 MVP 以咨询师个人数据范围为主。个案、咨询记录、录音、转写、AI 摘要、风险提示、报告和提醒都按 `owner_user_id` 隔离。

跨用户访问会返回 `404 NOT_FOUND`，避免通过错误信息泄露对象存在性。

### 3.3 时间格式

时间字段使用 ISO 8601 datetime 字符串，例如：

```text
2026-07-11T14:30:00Z
```

### 3.4 当前 Mock 能力边界

当前实现用于 MVP 联调验证，以下能力仍是本地/mock：

- 上传 URL 返回 `mock://local-recordings/...`，尚未接真实对象存储。
- 转写任务由接口状态机模拟，尚未接真实 ASR Provider。
- AI 摘要、风险提示和报告草稿由本地 deterministic `FakeAIProvider` 生成，尚未接真实 LLM Provider。

## 4. 通用错误响应

后端通过统一异常处理返回结构化错误。常见形态：

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Resource not found.",
    "details": {}
  },
  "request_id": "..."
}
```

常见错误：

| HTTP 状态码 | code 示例 | 说明 |
| --- | --- | --- |
| 400 | `INVALID_AUDIO_FORMAT` | 请求参数或业务参数不合法 |
| 401 | `INVALID_CREDENTIALS` / `UNAUTHENTICATED` | 未登录或登录凭证无效 |
| 403 | `INVALID_UPLOAD_TOKEN` | 上传 token 无效或无权限 |
| 404 | `NOT_FOUND` | 资源不存在或不属于当前用户 |
| 409 | `INVALID_STATE` / `AI_INPUT_REQUIRED` | 当前状态不允许执行该动作 |
| 422 | FastAPI validation error | 请求体或参数校验失败 |

## 5. 状态枚举

| 类型 | 枚举 |
| --- | --- |
| 个案状态 | `active`、`archived` |
| 咨询记录状态 | `draft`、`submitted` |
| 录音来源 | `app_recording`、`uploaded_file` |
| 录音上传状态 | `created`、`uploading`、`uploaded`、`verified`、`upload_failed`、`cancelled`、`transcription_queued` |
| 转写任务状态 | `queued`、`processing`、`succeeded`、`failed`、`cancelled` |
| AI 摘要状态 | `pending_review`、`confirmed`、`rejected` |
| 风险等级 | `none`、`low`、`medium`、`high`、`urgent` |
| 风险类型 | `self_harm`、`suicide`、`violence`、`abuse`、`emotional_crisis`、`other` |
| 风险状态 | `pending_confirmation`、`confirmed`、`dismissed` |
| 报告状态 | `ai_draft`、`editing`、`confirmed` |
| 报告范围 | `latest`、`recent_n`、`date_range` |
| 提醒来源 | `manual`、`ai_suggestion`、`risk_signal` |
| 提醒状态 | `pending`、`done`、`cancelled` |

## 6. 接口总览

| 分组 | 方法 | 路径 | 认证 | 说明 |
| --- | --- | --- | --- | --- |
| 系统 | GET | `/api/v1/health` | 否 | 服务健康检查 |
| 系统 | GET | `/api/v1/health/db` | 否 | 数据库连接健康检查 |
| 认证 | POST | `/api/v1/auth/register` | 否 | 邮箱注册并返回 token |
| 认证 | POST | `/api/v1/auth/login` | 否 | 邮箱密码登录 |
| 认证 | POST | `/api/v1/auth/refresh` | 否 | 刷新 token |
| 认证 | POST | `/api/v1/auth/logout` | 否 | 撤销 refresh token |
| 认证 | POST | `/api/v1/auth/change-password` | 是 | 修改当前用户密码 |
| 当前用户 | GET | `/api/v1/me` | 是 | 获取当前咨询师资料 |
| 个案 | POST | `/api/v1/cases` | 是 | 创建个案 |
| 个案 | GET | `/api/v1/cases` | 是 | 查询个案列表 |
| 个案 | GET | `/api/v1/cases/{case_id}` | 是 | 查看个案详情 |
| 个案 | PATCH | `/api/v1/cases/{case_id}` | 是 | 更新个案 |
| 个案 | POST | `/api/v1/cases/{case_id}/archive` | 是 | 归档个案 |
| 咨询记录 | POST | `/api/v1/cases/{case_id}/records` | 是 | 创建咨询记录 |
| 咨询记录 | GET | `/api/v1/cases/{case_id}/records/{record_id}` | 是 | 查看咨询记录 |
| 咨询记录 | PATCH | `/api/v1/cases/{case_id}/records/{record_id}` | 是 | 更新咨询记录 |
| 录音 | POST | `/api/v1/cases/{case_id}/recordings` | 是 | 创建录音元数据 |
| 录音 | GET | `/api/v1/recordings/{recording_id}` | 是 | 查看录音状态 |
| 录音 | POST | `/api/v1/recordings/{recording_id}/upload-url` | 是 | 获取上传 URL |
| 录音 | POST | `/api/v1/recordings/{recording_id}/complete-upload` | 是 | 完成上传确认 |
| 录音 | POST | `/api/v1/recordings/{recording_id}/retry-upload` | 是 | 重新申请上传 |
| 录音 | POST | `/api/v1/recordings/{recording_id}/cancel` | 是 | 取消录音上传 |
| 转写 | POST | `/api/v1/recordings/{recording_id}/transcriptions` | 是 | 创建转写任务 |
| 转写 | GET | `/api/v1/transcriptions/{task_id}` | 是 | 查看转写任务 |
| 转写 | PATCH | `/api/v1/transcriptions/{task_id}` | 是 | 更新转写状态/文本 |
| 转写 | DELETE | `/api/v1/transcriptions/{task_id}` | 是 | 删除转写任务 |
| 转写 | POST | `/api/v1/transcriptions/{task_id}/retry` | 是 | 重试失败转写 |
| AI 摘要 | POST | `/api/v1/records/{record_id}/ai-summaries` | 是 | 生成 AI 摘要 |
| AI 摘要 | GET | `/api/v1/ai-summaries/{summary_id}` | 是 | 查看 AI 摘要 |
| AI 摘要 | PATCH | `/api/v1/ai-summaries/{summary_id}` | 是 | 编辑 AI 摘要 |
| AI 摘要 | POST | `/api/v1/ai-summaries/{summary_id}/confirm` | 是 | 确认 AI 摘要 |
| 风险提示 | POST | `/api/v1/records/{record_id}/risk-signals` | 是 | 生成风险提示 |
| 风险提示 | PATCH | `/api/v1/risk-signals/{risk_id}` | 是 | 修改风险提示 |
| 风险提示 | POST | `/api/v1/risk-signals/{risk_id}/confirm` | 是 | 确认风险提示 |
| 风险提示 | POST | `/api/v1/risk-signals/{risk_id}/dismiss` | 是 | 标记误报 |
| 风险提示 | POST | `/api/v1/risk-signals/{risk_id}/reminders` | 是 | 从风险创建提醒 |
| 报告 | POST | `/api/v1/cases/{case_id}/reports` | 是 | 生成报告草稿 |
| 报告 | GET | `/api/v1/reports/{report_id}` | 是 | 查看报告草稿 |
| 报告 | PATCH | `/api/v1/reports/{report_id}` | 是 | 编辑报告草稿 |
| 报告 | POST | `/api/v1/reports/{report_id}/confirm` | 是 | 确认报告 |
| 报告 | GET | `/api/v1/reports/{report_id}/copy-data` | 是 | 获取报告复制内容 |
| 提醒 | POST | `/api/v1/reminders` | 是 | 创建提醒 |
| 提醒 | GET | `/api/v1/reminders` | 是 | 查询提醒列表 |
| 提醒 | POST | `/api/v1/reminders/{reminder_id}/complete` | 是 | 完成提醒 |

## 7. 系统接口

### 7.1 GET `/api/v1/health`

用途：检查 API 服务是否可用。

响应字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `status` | string | `ok` |
| `app_name` | string | 应用名称 |
| `version` | string | 应用版本 |

### 7.2 GET `/api/v1/health/db`

用途：检查数据库连接是否可用。

响应字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `status` | string | `ok` |

## 8. 认证接口

### 8.1 POST `/api/v1/auth/register`

用途：使用邮箱注册咨询师账号。注册成功后直接返回 token pair，客户端可立即使用 access token 调用 `/api/v1/me` 和后续业务接口。

请求体：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `email` | string | 是 | 注册邮箱，服务端会执行 trim + lowercase 规范化 |
| `password` | string | 是 | 密码，长度 8 到 256 |
| `full_name` | string | 是 | 咨询师姓名，长度 1 到 100 |
| `organization_name` | string/null | 否 | 机构名称，最长 150 |
| `practitioner_role` | string/null | 否 | 执业角色，最长 100 |

响应：`201 Created`，`TokenResponse`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `access_token` | string | 访问令牌 |
| `refresh_token` | string | 刷新令牌 |
| `token_type` | string | `bearer` |
| `expires_at` | datetime | access token 过期时间 |

关键行为：

- 密码只保存哈希，不保存明文。
- 邮箱按 trim + lowercase 后保存和去重。
- 注册成功会创建 refresh session。
- 注册成功会写入注册审计事件，但不得记录明文密码或 token。

常见错误：

| HTTP 状态码 | code | 说明 |
| --- | --- | --- |
| 409 | `EMAIL_ALREADY_REGISTERED` | 邮箱已注册 |
| 422 | FastAPI validation error | 请求体校验失败，例如密码过短 |

### 8.2 POST `/api/v1/auth/login`

用途：使用邮箱和密码登录。

请求体：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `email` | string | 是 | 登录邮箱 |
| `password` | string | 是 | 密码 |

响应：`TokenResponse`

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `access_token` | string | 访问令牌 |
| `refresh_token` | string | 刷新令牌 |
| `token_type` | string | `bearer` |
| `expires_at` | datetime | access token 过期时间 |

常见错误：`401 INVALID_CREDENTIALS`

### 8.3 POST `/api/v1/auth/refresh`

用途：使用 refresh token 换取新的 token pair。

请求体：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `refresh_token` | string | 是 | 刷新令牌 |

响应：`TokenResponse`

常见错误：`401 INVALID_CREDENTIALS`

### 8.4 POST `/api/v1/auth/logout`

用途：撤销 refresh token。

请求体：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `refresh_token` | string | 是 | 需要撤销的刷新令牌 |

响应：`204 No Content`

### 8.5 POST `/api/v1/auth/change-password`

用途：已登录用户修改自己的密码。

认证：需要 Bearer token。

请求体：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `current_password` | string | 是 | 当前密码，长度 1 到 256 |
| `new_password` | string | 是 | 新密码，长度 8 到 256 |

响应：`204 No Content`

关键行为：

- 必须先校验当前密码。
- 修改成功后更新密码哈希。
- 修改成功后撤销该用户当前所有 refresh session，旧 refresh token 不能继续刷新。
- 修改密码成功或失败会写入审计事件，但不得记录明文密码或 token。

常见错误：

| HTTP 状态码 | code | 说明 |
| --- | --- | --- |
| 401 | `UNAUTHORIZED` | 未登录或 access token 无效 |
| 401 | `INVALID_CREDENTIALS` | 当前密码错误 |
| 422 | FastAPI validation error | 请求体校验失败，例如新密码过短 |

## 9. 当前用户接口

### 9.1 GET `/api/v1/me`

用途：获取当前登录咨询师资料。

认证：需要 Bearer token。

响应字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | string | 用户 ID |
| `email` | string | 邮箱 |
| `full_name` | string | 姓名 |
| `organization_name` | string/null | 机构名称 |
| `practitioner_role` | string/null | 执业角色 |

## 10. 个案接口

### 10.1 POST `/api/v1/cases`

用途：创建个案。MVP 不强制真实姓名，`nickname_or_code` 必填即可。

请求体：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `nickname_or_code` | string | 是 | 昵称或编号 |
| `gender` | string/null | 否 | 性别 |
| `age_group` | string/null | 否 | 年龄段 |
| `presenting_complaint` | string/null | 否 | 主诉 |
| `tags` | string[] | 否 | 标签 |
| `notes` | string/null | 否 | 备注 |

响应：`CaseResponse`

关键字段：`id`、`owner_user_id`、`nickname_or_code`、`status`、`last_session_at`、`created_at`、`updated_at`。

### 10.2 GET `/api/v1/cases`

用途：查询当前用户的个案列表。

Query 参数：

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `query` | string | 无 | 按昵称/编号、主诉、备注搜索 |
| `tag` | string | 无 | 按标签过滤 |
| `status` | `active`/`archived`/`all` | `active` | 个案状态过滤 |

排序：默认按 `last_session_at` 倒序，空值靠后，再按 `created_at` 倒序。

### 10.3 GET `/api/v1/cases/{case_id}`

用途：查看个案详情。

Path 参数：

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `case_id` | string | 个案 ID |

### 10.4 PATCH `/api/v1/cases/{case_id}`

用途：更新个案基础信息或状态。

请求体：`CaseUpdateRequest`，字段同创建接口，均为可选；`status` 可为 `active` 或 `archived`。

### 10.5 POST `/api/v1/cases/{case_id}/archive`

用途：归档个案。

状态变化：`status` 更新为 `archived`。

## 11. 咨询记录接口

### 11.1 POST `/api/v1/cases/{case_id}/records`

用途：创建咨询记录，可先保存草稿。

请求体：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `session_date` | datetime/null | 否 | 咨询时间，不传则使用当前时间 |
| `duration_minutes` | integer/null | 否 | 咨询时长，0 到 1440 |
| `modality` | string/null | 否 | 咨询方式 |
| `raw_text` | string/null | 否 | 人工原始文本 |
| `transcription_text` | string/null | 否 | 转写文本 |
| `main_complaint_change` | string/null | 否 | 主诉变化 |
| `emotion_state` | string/null | 否 | 情绪状态 |
| `intervention` | string/null | 否 | 干预方式 |
| `goal` | string/null | 否 | 本次目标 |
| `next_plan` | string/null | 否 | 下次计划 |
| `tags` | string[] | 否 | 标签 |
| `status` | `draft`/`submitted` | 否 | 默认 `draft` |

响应额外计算字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `has_ai_input` | boolean | `raw_text` 或 `transcription_text` 非空 |
| `ai_ready` | boolean | 当前等同于 `has_ai_input` |

隐私提示：`raw_text`、`transcription_text`、结构化咨询字段都属于高敏数据。

### 11.2 GET `/api/v1/cases/{case_id}/records/{record_id}`

用途：查看咨询记录详情。

### 11.3 PATCH `/api/v1/cases/{case_id}/records/{record_id}`

用途：编辑咨询记录草稿、转写文本或结构化字段。

请求体：`RecordUpdateRequest`，字段同创建接口，均为可选。

## 12. 录音与上传接口

### 12.1 POST `/api/v1/cases/{case_id}/recordings`

用途：创建录音元数据，支持 App 内录音和上传已有文件。

请求体：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `record_id` | string/null | 否 | 关联咨询记录 ID |
| `local_recording_id` | string/null | 否 | 客户端本地录音 ID，用于恢复上传 |
| `source` | `app_recording`/`uploaded_file` | 是 | 录音来源 |
| `consent_confirmed` | boolean | 否 | 是否已确认录音知情同意 |
| `original_filename` | string | 是 | 文件名 |
| `mime_type` | string | 是 | MIME 类型 |
| `file_size_bytes` | integer/null | 否 | 文件大小 |
| `duration_seconds` | integer/null | 否 | 音频时长 |
| `checksum` | string/null | 否 | 文件校验值 |

支持格式：

| 扩展名 | MIME |
| --- | --- |
| `.m4a` | `audio/mp4`、`audio/m4a`、`audio/x-m4a` |
| `.mp3` | `audio/mpeg`、`audio/mp3` |
| `.wav` | `audio/wav`、`audio/wave`、`audio/x-wav` |

初始状态：`upload_status = created`

### 12.2 GET `/api/v1/recordings/{recording_id}`

用途：查看录音元数据和上传状态。

响应关键字段：`upload_status`、`upload_url`、`failure_reason`、`retryable`。

### 12.3 POST `/api/v1/recordings/{recording_id}/upload-url`

用途：获取上传地址。

请求体：

| 字段 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `expires_in_seconds` | integer | 900 | 上传地址有效期，60 到 3600 秒 |

响应字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `upload_url` | string | 当前为 `mock://local-recordings/...` |
| `upload_token` | string | 上传确认 token |
| `expires_at` | datetime | 过期时间 |
| `method` | string | 当前为 `PUT` |
| `headers` | object | 上传所需 header |
| `fields` | object | 上传元数据 |

前置条件：

- 录音未取消。
- `consent_confirmed = true`。
- 文件名和 MIME 类型合法。

状态变化：`created` 或 `upload_failed` -> `uploading`

### 12.4 POST `/api/v1/recordings/{recording_id}/complete-upload`

用途：客户端上传完成后通知后端校验。

请求体：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `upload_token` | string | 是 | 上传 token |
| `original_filename` | string/null | 否 | 可覆盖文件名 |
| `mime_type` | string/null | 否 | 可覆盖 MIME |
| `file_size_bytes` | integer/null | 否 | 文件大小 |
| `duration_seconds` | integer/null | 否 | 音频时长 |
| `checksum` | string/null | 否 | 校验值 |
| `simulate_success` | boolean | 否 | 当前 mock 校验结果，默认 `true` |

状态变化：

- 成功：`uploading` -> `verified`
- 失败：`uploading` -> `upload_failed`

### 12.5 POST `/api/v1/recordings/{recording_id}/retry-upload`

用途：对可重试的上传失败录音重新申请上传地址。

前置条件：`upload_status = upload_failed` 且 `retryable = true`

### 12.6 POST `/api/v1/recordings/{recording_id}/cancel`

用途：取消录音上传，并取消关联的排队/处理中转写任务。

状态变化：

- 录音：`upload_status = cancelled`
- 活跃转写任务：`status = cancelled`

## 13. 转写任务接口

### 13.1 POST `/api/v1/recordings/{recording_id}/transcriptions`

用途：为已校验录音创建转写任务。

请求体：

| 字段 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `provider` | string | `mock-local` | 转写 Provider |

前置条件：录音状态为 `verified` 或 `transcription_queued`

状态变化：

- 新建 `TranscriptionTask(status=queued)`
- 录音状态更新为 `transcription_queued`

### 13.2 GET `/api/v1/transcriptions/{task_id}`

用途：查询转写任务状态和结果。

响应字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | string | 转写任务 ID |
| `recording_id` | string | 录音 ID |
| `status` | string | 转写任务状态 |
| `provider` | string | Provider |
| `provider_job_id` | string/null | Provider 任务 ID |
| `transcript_text` | string/null | 转写文本 |
| `failure_reason` | string/null | 失败原因 |
| `retryable` | boolean | 是否可重试 |
| `completed_at` | datetime/null | 完成时间 |

### 13.3 PATCH `/api/v1/transcriptions/{task_id}`

用途：更新转写任务状态或文本。当前 MVP 用于模拟 ASR 回调或人工校正。

请求体：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `status` | `queued`/`processing`/`succeeded`/`failed`/`cancelled` |
| `provider_job_id` | string/null |
| `transcript_text` | string/null |
| `failure_reason` | string/null |
| `retryable` | boolean/null |

当状态为 `succeeded` 且存在 `transcript_text` 时，会同步到关联咨询记录的 `transcription_text`。

### 13.4 DELETE `/api/v1/transcriptions/{task_id}`

用途：删除转写任务，并清理同步到咨询记录的转写文本。

响应：`204 No Content`

### 13.5 POST `/api/v1/transcriptions/{task_id}/retry`

用途：重试失败且可重试的转写任务。

前置条件：`status = failed` 且 `retryable = true`

状态变化：重置为 `queued`

## 14. AI 摘要接口

### 14.1 POST `/api/v1/records/{record_id}/ai-summaries`

用途：基于咨询记录的 `transcription_text` 或 `raw_text` 生成结构化摘要。

请求体：

| 字段 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `template` | `SOAP` | `SOAP` | 摘要模板 |

前置条件：咨询记录必须有 `transcription_text` 或 `raw_text`。

响应关键字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `status` | string | 初始为 `pending_review` |
| `template` | string | 当前为 `SOAP` |
| `summary_json` | object | 结构化摘要 |
| `model_name` | string | 当前 Provider 模型名 |
| `prompt_version` | string | Prompt 版本 |
| `schema_version` | string | Schema 版本 |
| `input_snapshot_hash` | string | 输入快照哈希 |
| `generated_by_ai` | boolean | 是否 AI 生成 |
| `disclaimer` | string | AI 免责声明 |

常见错误：`409 AI_INPUT_REQUIRED`

### 14.2 GET `/api/v1/ai-summaries/{summary_id}`

用途：查看 AI 摘要详情。

### 14.3 PATCH `/api/v1/ai-summaries/{summary_id}`

用途：编辑摘要内容或状态。

请求体：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `status` | `pending_review`/`confirmed`/`rejected` |
| `summary_json` | object/null |

如果更新 `summary_json`，响应中的 `generated_by_ai` 会变为 `false`，表示已经被人工编辑。

### 14.4 POST `/api/v1/ai-summaries/{summary_id}/confirm`

用途：确认 AI 摘要。

状态变化：`status = confirmed`，记录 `confirmed_by` 和 `confirmed_at`。

## 15. 风险提示接口

### 15.1 POST `/api/v1/records/{record_id}/risk-signals`

用途：基于咨询记录生成风险提示。风险提示只作为专业参考，不构成诊断结论。

请求体：当前为空对象 `{}`。

响应字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `level` | string | 风险等级 |
| `category` | string | 风险类型 |
| `status` | string | 初始为 `pending_confirmation` |
| `evidence_excerpt` | string | 证据片段摘要 |
| `suggested_actions` | string[] | 建议动作 |
| `disclaimer` | string | 风险提示免责声明 |

常见错误：`409 AI_INPUT_REQUIRED`

### 15.2 PATCH `/api/v1/risk-signals/{risk_id}`

用途：人工修改风险等级、类型、状态、证据摘要或建议动作。

### 15.3 POST `/api/v1/risk-signals/{risk_id}/confirm`

用途：确认风险提示。

请求体：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `handled_note` | string/null | 否 | 处置说明 |

状态变化：`status = confirmed`

### 15.4 POST `/api/v1/risk-signals/{risk_id}/dismiss`

用途：标记风险提示为误报。

状态变化：`status = dismissed`

### 15.5 POST `/api/v1/risk-signals/{risk_id}/reminders`

用途：从风险提示创建跟进提醒。

请求体：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `title` | string/null | 否 | 提醒标题，不传则自动生成 |
| `remind_at` | datetime | 是 | 提醒时间 |
| `note` | string/null | 否 | 备注 |

响应：`ReminderResponse`

## 16. 报告草稿接口

### 16.1 POST `/api/v1/cases/{case_id}/reports`

用途：基于个案咨询记录生成报告草稿。

请求体：

| 字段 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `range_type` | `latest`/`recent_n`/`date_range` | `latest` | 报告范围 |
| `recent_count` | integer | 3 | 最近 N 次，1 到 50 |
| `start_at` | datetime/null | 无 | 日期范围开始 |
| `end_at` | datetime/null | 无 | 日期范围结束 |

响应字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `status` | string | 初始为 `ai_draft` |
| `range_type` | string | 报告范围 |
| `content_json` | object | 报告结构化内容 |
| `source_record_ids` | string[] | 来源咨询记录 |
| `disclaimer` | string | AI 报告免责声明 |

说明：

- 仅纳入已确认风险作为正式风险内容。
- 未确认风险会以待确认数量提示，不作为确认结论。
- PDF 导出不在当前 MVP 实现范围内。

### 16.2 GET `/api/v1/reports/{report_id}`

用途：查看报告草稿。

### 16.3 PATCH `/api/v1/reports/{report_id}`

用途：编辑报告草稿。

请求体：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `status` | `editing`/null |
| `content_json` | object/null |

如果更新 `content_json` 且报告未确认，状态会变为 `editing`。

### 16.4 POST `/api/v1/reports/{report_id}/confirm`

用途：确认报告。

状态变化：`status = confirmed`，记录 `confirmed_at`。

### 16.5 GET `/api/v1/reports/{report_id}/copy-data`

用途：获取适合 App 复制的报告内容。

响应字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `report_id` | string | 报告 ID |
| `plain_text` | string | 可复制纯文本 |
| `content_json` | object | 结构化报告 |
| `source_record_ids` | string[] | 来源咨询记录 |
| `disclaimer` | string | 免责声明 |

## 17. 跟进提醒接口

### 17.1 POST `/api/v1/reminders`

用途：手动创建提醒，或从 AI 建议/风险提示创建提醒。

请求体：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `case_id` | string | 是 | 个案 ID |
| `source_type` | `manual`/`ai_suggestion`/`risk_signal` | 否 | 默认 `manual` |
| `source_id` | string/null | 否 | 非手动来源时必填 |
| `title` | string | 是 | 标题 |
| `remind_at` | datetime | 是 | 提醒时间 |
| `note` | string/null | 否 | 备注 |

响应字段包含计算字段：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `is_overdue` | boolean | 待处理且提醒时间早于当前时间 |

### 17.2 GET `/api/v1/reminders`

用途：查询提醒列表。

Query 参数：

| 参数 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `status` | `pending`/`done`/`all` | `pending` | 提醒状态过滤 |

排序：按状态、提醒时间、创建时间倒序。

### 17.3 POST `/api/v1/reminders/{reminder_id}/complete`

用途：完成提醒。

状态变化：`status = done`，记录 `completed_at`。

## 18. MVP 端到端调用流程

以下流程可作为 Flutter 端联调和测试用例主线：

```text
1. POST /api/v1/auth/register 或 POST /api/v1/auth/login
2. GET /api/v1/me
3. POST /api/v1/cases
4. POST /api/v1/cases/{case_id}/records
5. POST /api/v1/cases/{case_id}/recordings
6. POST /api/v1/recordings/{recording_id}/upload-url
7. POST /api/v1/recordings/{recording_id}/complete-upload
8. POST /api/v1/recordings/{recording_id}/transcriptions
9. PATCH /api/v1/transcriptions/{task_id}
10. POST /api/v1/records/{record_id}/ai-summaries
11. PATCH /api/v1/ai-summaries/{summary_id}
12. POST /api/v1/ai-summaries/{summary_id}/confirm
13. POST /api/v1/records/{record_id}/risk-signals
14. PATCH /api/v1/risk-signals/{risk_id}
15. POST /api/v1/risk-signals/{risk_id}/confirm 或 /dismiss
16. POST /api/v1/risk-signals/{risk_id}/reminders
17. POST /api/v1/cases/{case_id}/reports
18. PATCH /api/v1/reports/{report_id}
19. POST /api/v1/reports/{report_id}/confirm
20. GET /api/v1/reports/{report_id}/copy-data
21. POST /api/v1/reminders/{reminder_id}/complete
```

账号安全补充流程：

```text
1. POST /api/v1/auth/login
2. POST /api/v1/auth/change-password
3. 使用旧 refresh token 调用 POST /api/v1/auth/refresh 会返回 401
4. 使用新密码重新 POST /api/v1/auth/login 获取新的 token pair
```

关键约束：

- 未登录不能访问业务数据。
- 注册邮箱会进行 trim + lowercase 规范化，重复邮箱不能注册。
- 修改密码必须校验当前密码；成功后旧 refresh token 失效。
- 未确认录音知情同意不能获取上传 URL。
- 录音未 `verified` 不能创建转写任务。
- 咨询记录没有 `raw_text` 或 `transcription_text` 时不能生成 AI 摘要或风险提示。
- AI 输出默认是草稿或待确认状态。
- 风险提示必须人工确认、修改或标记误报。
- 未确认风险不会作为正式风险结论进入报告。

## 19. 隐私与安全注意事项

以下数据按高敏数据处理：

- 录音文件与录音元数据。
- 转写文本。
- 咨询记录原文和结构化字段。
- AI 摘要正文。
- 风险证据片段和处置说明。
- 报告正文。
- 提醒内容。

客户端联调注意事项：

- 不要在普通日志中打印 token、密码、转写全文、咨询记录原文、风险证据全文或报告正文。
- 锁屏通知、Toast、错误弹窗不应展示敏感详情。
- App 本地缓存如保存录音或草稿，应使用加密存储。
- AI 摘要、风险提示、报告草稿均需展示“仅供专业参考，需人工确认”的产品边界。

## 20. OpenAPI 契约文件

机器可读 OpenAPI 文件位于：

```text
/Users/bytedance/CaseNotes-PRD/.trae/documents/phase1_backend/openapi.json
```

建议用途：

- Flutter 客户端生成 API client。
- 测试侧生成接口覆盖清单。
- 后端变更时做契约 diff。
- 与本文档交叉校验请求/响应 schema。
