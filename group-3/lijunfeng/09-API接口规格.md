# 09 — API 接口规格（引导版模板）

---

| 项 | 值 |
|---|---|
| 模块编号 | M1-QA |
| 模块名称 | 投研AI研报解读助手 |
| 文档版本 | v0.1 |
| 阶段 | Design（How — 契约真源） |
| Base URL | `/api/v1/agent` |

---

> **本文是全部 API 端点的契约真源**。`05` 定义"用户要什么"，**09（本文）定义"后端必须返回什么"**，`13` 的测试断言以本文为准。

## 1. 端点总览

| # | 端点 | 方法 | 功能 | 成功码 | 对应 REQ |
|---|------|------|------|--------|----------|
| 1 | `/api/v1/agent/capabilities` | GET | 能力探测 | 200 | — |
| 2 | `/api/v1/agent/ask` | POST | 自然语言问答 | 200 | REQ-M1QA-002 |
| 3 | `/api/v1/agent/sessions` | GET | 会话列表 | 200 | REQ-M1QA-001 |
| 4 | `/api/v1/agent/sessions` | POST | 新建会话 | 201 | REQ-M1QA-001 |
| 5 | `/api/v1/agent/sessions/<id>` | DELETE | 删除会话 | 200 | REQ-M1QA-001 |
| 6 | `/api/v1/agent/sessions/<id>/records` | GET | 问答记录 | 200 | REQ-M1QA-003 |
| 7 | `/api/v1/agent/reports` | GET | 研报列表（按行业分类） | 200 | REQ-M1QA-001 AC-001-01 |
| 8 | `/api/v1/agent/reports/<id>/reading` | GET | 单篇研报AI解读结果 | 200 | REQ-M1QA-001 AC-001-02 |
| 9 | `/api/v1/agent/reports/<id>/reading` | POST | 触发研报AI解读 | 201 | REQ-M1QA-001 AC-001-02 |
| 10 | `/api/v1/agent/reports/compare` | POST | 多研报数据对比 | 200 | REQ-M1QA-001 AC-001-03 |

## 2. 统一响应规范

### 成功响应

```json
{ "traceId": "tr_abc123...", /* 业务字段 */ }
```

### 错误响应

```json
{ "error": { "code": "EMPTY_QUERY", "message": "请输入问题", "details": {}, "traceId": "tr_..." } }
```

### 错误码清单

| HTTP | error.code | 触发条件 | details |
|------|-----------|----------|---------|
| 400 | `EMPTY_QUERY` | query 为空/null | `{}` |
| 400 | `INVALID_QUERY` | query 超 500 字符 | `{"max_length":500}` |
| 400 | `MISSING_SESSION_ID` | session_id 缺失 | `{}` |
| 404 | `SESSION_NOT_FOUND` | session_id 不存在 | `{"session_id":"..."}` |
| 404 | `REPORT_NOT_FOUND` | report_id 不存在 | `{"report_id":"..."}` |
| 400 | `INVALID_COMPARE_PARAMS` | 对比参数不合法（非同时间段/少于2份） | `{"reason":"..."}` |
| 500 | `INTERNAL_ERROR` | 服务器内部错误 | `{}` |

## 3. POST /ask — 自然语言问答

**请求体**：

| 字段 | 类型 | 必填 | 约束 | 说明 |
|------|------|------|------|------|
| `query` | string | **是** | 1–500 字符 | 用户提问原文 |
| `session_id` | string | **是** | UUID | 目标会话 ID |

**成功响应**（200）：

| 字段 | 类型 | 必有 | 说明 |
|------|------|------|------|
| `traceId` | string | 是 | 链路追踪 ID |
| `answer` | string | 是 | 答案文本 |
| `llm_used` | boolean | 是 | 是否使用真实 LLM |
| `model` | string\|null | 是 | 模型标识 |
| `response_time_ms` | integer | 是 | 响应耗时（毫秒） |
| `answer_source` | string | 是 | copaw / bailian / demo |

## 4. POST /sessions — 新建会话

**请求体**：

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| `title` | string | 否 | "新会话" | 会话标题 |

**成功响应**（201）：

| 字段 | 类型 | 必有 | 说明 |
|------|------|------|------|
| `traceId` | string | 是 | 链路追踪 ID |
| `session_id` | string | 是 | 新建会话的 UUID |
| `title` | string | 是 | 会话标题 |
| `created_at` | string | 是 | ISO-8601 创建时间 |
| `query_count` | integer | 是 | 初始为 0 |

## 5. GET /sessions — 会话列表

> 无请求体，返回 sessions 数组。

**成功响应**（200）：

| 字段 | 类型 | 必有 | 说明 |
|------|------|------|------|
| `traceId` | string | 是 | 链路追踪 ID |
| `sessions` | array | 是 | 会话列表 |
| `sessions[].session_id` | string | 是 | 会话 UUID |
| `sessions[].title` | string | 是 | 会话标题 |
| `sessions[].created_at` | string | 是 | ISO-8601 创建时间 |
| `sessions[].query_count` | integer | 是 | 累计问答次数 |

## 6. DELETE /sessions/<id> — 删除会话

> 路径参数 `session_id`，无请求体。

**成功响应**（200）：

| 字段 | 类型 | 必有 | 说明 |
|------|------|------|------|
| `traceId` | string | 是 | 链路追踪 ID |
| `message` | string | 是 | 确认消息，如 "会话已删除" |

**副作用**：级联删除该会话下所有关联的 QARecord 记录。

## 7. GET /sessions/<id>/records — 问答记录

> 路径参数 `session_id`，返回 records 数组。

**成功响应**（200）：

| 字段 | 类型 | 必有 | 说明 |
|------|------|------|------|
| `traceId` | string | 是 | 链路追踪 ID |
| `session_id` | string | 是 | 会话 UUID |
| `records` | array | 是 | 问答记录列表 |
| `records[].id` | string | 是 | 记录 ID |
| `records[].query` | string | 是 | 用户提问原文 |
| `records[].answer` | string | 是 | 答案文本 |
| `records[].llm_used` | boolean | 是 | 是否使用真实 LLM |
| `records[].answer_source` | string | 是 | copaw / bailian / demo |
| `records[].response_time_ms` | integer | 是 | 响应耗时 |
| `records[].timestamp` | string | 是 | ISO-8601 记录时间 |

## 8. GET /reports — 研报列表（按行业分类）

> 返回按申万二级行业分类的研报列表，含每个行业的研报数量。

**查询参数**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `industry` | string | 否 | 按申万二级行业筛选 |
| `date_from` | string | 否 | 起始日期（ISO-8601） |
| `date_to` | string | 否 | 截止日期（ISO-8601） |

**成功响应**（200）：

| 字段 | 类型 | 必有 | 说明 |
|------|------|------|------|
| `traceId` | string | 是 | 链路追踪 ID |
| `industries` | array | 是 | 行业分类列表 |
| `industries[].name` | string | 是 | 申万二级行业名称 |
| `industries[].report_count` | integer | 是 | 该行业研报数量 |
| `industries[].reports` | array | 是 | 研报摘要列表 |
| `industries[].reports[].report_id` | string | 是 | 研报 ID |
| `industries[].reports[].title` | string | 是 | 研报标题 |
| `industries[].reports[].source` | string | 是 | 来源券商 |
| `industries[].reports[].publish_date` | string | 是 | 发布日期 |

## 9. GET/POST /reports/<id>/reading — 研报AI解读

### GET — 获取已有解读结果

**成功响应**（200）：

| 字段 | 类型 | 必有 | 说明 |
|------|------|------|------|
| `traceId` | string | 是 | 链路追踪 ID |
| `report_id` | string | 是 | 研报 ID |
| `rating` | string | 是 | 评级（买入/增持/中性/减持/卖出） |
| `target_price` | number\|null | 是 | 目标价 |
| `key_points` | array | 是 | 核心观点列表 |
| `financial_forecast` | object\|null | 是 | 财务预测（营收增速、PE 等） |
| `reading_time` | string | 是 | 解读生成时间 |

### POST — 触发新的AI解读

**请求体**：无（使用路径参数 report_id）

**成功响应**（201）：同 GET 响应字段。

## 10. POST /reports/compare — 多研报数据对比

**请求体**：

| 字段 | 类型 | 必填 | 约束 | 说明 |
|------|------|------|------|------|
| `report_ids` | array | **是** | ≥ 2 个 ID | 参与对比的研报 ID 列表 |
| `date_from` | string | **是** | ISO-8601 | 对比时间段起始 |
| `date_to` | string | **是** | ISO-8601 | 对比时间段截止 |
| `metrics` | array | 否 | — | 指定对比指标（默认全部） |

**成功响应**（200）：

| 字段 | 类型 | 必有 | 说明 |
|------|------|------|------|
| `traceId` | string | 是 | 链路追踪 ID |
| `compare_id` | string | 是 | 对比结果 ID |
| `date_range` | object | 是 | `{from, to}` 时间段 |
| `reports` | array | 是 | 参与对比的研报信息 |
| `comparison_table` | array | 是 | 对比表行数据 |
| `comparison_table[].metric` | string | 是 | 指标名称 |
| `comparison_table[].values` | object | 是 | 各研报对应值 `{report_id: value}` |

## 11. 参数校验规则汇总

| 端点 | 字段 | 规则 | 失败 HTTP | error.code |
|------|------|------|-----------|-----------|
| POST /ask | `query` | 非空/非空白 | 400 | `EMPTY_QUERY` |
| POST /ask | `query` | ≤ 500 字符 | 400 | `INVALID_QUERY` |
| POST /ask | `session_id` | 非空 | 400 | `MISSING_SESSION_ID` |
| POST /ask | `session_id` | 存在于 sessions | 404 | `SESSION_NOT_FOUND` |
| DELETE /sessions/<id> | `session_id` | 存在于 sessions | 404 | `SESSION_NOT_FOUND` |
| GET /sessions/<id>/records | `session_id` | 存在于 sessions | 404 | `SESSION_NOT_FOUND` |
| GET /reports/<id>/reading | `report_id` | 存在于 reports | 404 | `REPORT_NOT_FOUND` |
| POST /reports/compare | `report_ids` | ≥ 2 个有效 ID | 400 | `INVALID_COMPARE_PARAMS` |
| POST /reports/compare | `date_from/to` | 必须为同一时间段 | 400 | `INVALID_COMPARE_PARAMS` |

---

| 版本 | 日期 | 说明 |
|------|------|------|
| v0.1 | 2026-04-14 | 首版填写 |
