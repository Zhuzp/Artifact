下一题：Function Calling 工具设计
假设用户说：
“给最近 30 天购买过高端套餐、但最近 7 天没有打开邮件的客户生成一个 Journey。”

Agent 需要调用：
query_audience
generate_journey
请回答：
1. 你会如何定义这两个工具的参数 Schema？
2. Agent 是一次性调用两个工具，还是分步骤调用？为什么？
3. 工具返回的数据应该以什么格式交给模型？
4. 如果 query_audience 返回 20 万个用户，应该怎么处理？
5. 如果模型生成的 Journey DSL 不符合 Schema，下一步怎么做？
6. generate_journey 是否可以直接发布？为什么？

*my answer*
1.不知道
2.分步调用，第二步依赖第一步的结果
3.json格式
4.不知道
5.重试，重试两次就展示给用户。
6.不可以，这一步只是建草稿，真正发布要运营人员确认点击发布

这题大约 **7/10**。第 2、3、5、6 点方向是对的，主要需要补齐工具 Schema 和大结果集处理。

### 1. 工具参数不能只依赖自然语言

工具要定义明确的结构化参数。

例如：

```json
{
  "name": "query_audience",
  "description": "查询符合条件的人群",
  "parameters": {
    "type": "object",
    "properties": {
      "purchase_days": {
        "type": "integer",
        "minimum": 1,
        "maximum": 365
      },
      "package_level": {
        "type": "string",
        "enum": ["standard", "premium", "enterprise"]
      },
      "email_opened_within_days": {
        "type": "integer",
        "minimum": 0,
        "maximum": 365
      },
      "limit": {
        "type": "integer",
        "minimum": 1,
        "maximum": 1000
      }
    },
    "required": [
      "purchase_days",
      "package_level",
      "email_opened_within_days"
    ]
  }
}
```

`generate_journey` 可以接收：

```json
{
  "audience_id": "aud_123",
  "goal": "促销召回",
  "channels": ["email", "sms"],
  "steps": [
    {
      "type": "email",
      "wait_days": 2
    }
  ]
}
```

参数 Schema 的作用是：

- 限制类型和范围；
- 约束枚举值；
- 防止模型少传参数；
- 让工具执行器可以安全校验；
- 为模型提供清晰的调用格式。

### 2. 分步骤调用

你的回答正确：

```text
用户需求
-> query_audience
-> 得到 audience_id 和统计结果
-> generate_journey
-> 生成 Journey DSL
```

第二步依赖第一步的人群结果，因此不适合一次性并行调用。

### 3. 工具返回 JSON

正确，但不能把 20 万个用户全部塞回上下文。工具应该返回：

```json
{
  "audience_id": "aud_123",
  "matched_count": 200000,
  "conditions": {
    "package_level": "premium",
    "purchased_within_days": 30,
    "opened_email_within_days": 7
  },
  "sample_users": [],
  "expires_at": "2026-09-25T12:00:00Z"
}
```

模型真正需要的是：

```text
audience_id
matched_count
筛选条件
```

而不是 20 万条用户明细。

### 4. 大结果集怎么处理

这是你需要补的重点。

不要把用户明细返回给模型，而是：

```text
数据库中保存筛选条件和结果快照
-> 返回 audience_id
-> Journey 节点引用 audience_id
```

例如：

```text
audience_id = aud_123
count = 200000
```

发送或执行 Journey 时，再由后端根据 `audience_id` 分页读取用户。

如果必须查看明细，则使用：

- 分页；
- 游标；
- 导出文件；
- 异步任务；
- 只返回统计和少量样本。

另外还要防止模型构造任意 SQL，工具只允许传结构化过滤条件，真正的 SQL 由后端生成。

### 5. Schema 校验失败

你的“重试两次后展示”方向对，但不要让模型盲目重试。流程应该是：

```text
生成 DSL
-> JSON Schema 校验
-> 失败时把具体路径和错误传给模型
-> 重新生成
-> 超过上限后进入人工修正或返回用户
```

例如错误信息：

```text
steps[1].wait_days 必须是大于 0 的整数
```

### 6. 不能直接发布

你的回答完全正确：

```text
生成 Journey 草稿
-> Schema 校验
-> 合规校验
-> 前端展示
-> 运营确认
-> 后端再次校验
-> 发布
```

发布是有副作用的动作，不能只依赖模型或前端按钮。后端还要检查：

```text
当前用户是否有发布权限
草稿是否属于当前租户
草稿是否已经审核
是否已经发布过
```

面试中可以这样回答：

> 工具使用结构化 Schema，限制参数类型、枚举和范围。Agent 先调用人群查询，工具只返回 audience_id、匹配数量和筛选条件，不把大量用户明细放入上下文；Journey 工具再引用 audience_id 生成 DSL。DSL 必须经过 JSON Schema 和业务规则校验，失败时把具体错误反馈给模型，超过重试次数后交给用户处理。生成和发布严格分离，发布必须经过人工确认和后端权限校验。

**下一题：Agent 状态和上下文管理**

假设一次 Agent 执行包含：

```text
识别意图
-> 查询人群
-> 生成文案
-> 生成 Journey DSL
-> Schema 校验
-> 合规校验
-> 等待用户确认
```

请回答：

1. 你会在 State 中保存哪些关键字段？
2. 为什么不能只把所有历史对话直接拼进 Prompt？
3. 用户确认之前，如何让流程暂停并在之后继续？
4. 如果 Schema 校验连续失败，如何避免 Agent 无限循环？
5. 服务重启后，如何从“等待用户确认”这个节点继续？