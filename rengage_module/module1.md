**模块1：AI 多渠道营销文案生成（核心P0）**
根据场景生成全新 Email/SMS/Push/In-App 文案
参考品牌规范、产品知识、历史优质模板和行业实践生成邮件模板（模型输出结构化 Email DSL,也就是json数据，再由前端去组装   ）
改写当前模板：正式、温柔、促销、简洁
对当前内容做合规预检
推荐现有模板和已审核图片素材


*一次真实调用*
1. 用户在前端输入需求：`给客户10086写老客户召回邮件，推广知识库文档解析功能`
2. 请求发给**Java 后端**，Java 做租户鉴权、参数基础校验，转发请求到 Python LangGraph Agent 服务
3. 【意图识别节点】LLM 判断意图：`email_template`（邮件模板生成）
4. 【参数提取节点】LLM 提取结构化参数：customerId、受众、邮件目的、产品、语气
5. 【拉取业务数据节点】代码固定调用 Java HTTP 接口 → Java 查询 MySQL，返回客户结构化信息（客户名称、套餐等级等）存入 state
6. 【RAG 检索节点】PGVector 复合检索，取出产品知识库相关文档片段（产品卖点、功能描述）存入 state
7. 将【用户原始需求 + 提取参数 + MySQL 客户数据 + PG 检索产品资料】全部组装进 Prompt，交给 LLM
8. LLM 生成邮件草稿
9. 【合规检查节点】校验草稿：敏感词、夸大宣传，不合规则回到生成节点重写
10. 合规通过，得到最终邮件内容，Python 返回结果给 Java 服务
11. Java 可以可选：把新模板存入 MySQL，再将最终内容返回前端展示

*假如是生成邮件模版*
1. LLM 判断场景，**挑选 templateId**，同时生成 blocks 数组
2. JSON Schema 校验：
- 校验 templateId 是否在枚举列表（防止 AI 瞎编模板）
- 校验 blocks 里面的 type 是否为合法组件类型
- 在Python校验，如果有问题就把msg返回给llm，去修改
3. Java 后端根据`templateId`从数据库**取出对应的预制 MJML 模板字符串**
4. Handlebars 把 blocks 数据填充进这套 MJML 模板
5. 编译 MJML → 前端预览；用户还可以进入 MJML 编辑器二次拖拽修改
# Handlebars（作用：把校验好的 JSON，填充进 MJML 模板字符串）
> **Handlebars 是模板渲染引擎，只负责变量替换、循环、if 判断；不做校验。**
> 输入：MJML 模板字符串 + 已经 Schema 校验通过的 JSON 数据
> 输出：填充完成的完整 MJML 文本，之后交给 mjml 编译成 HTML

*邮件模版生成Prompt*
你是邮件内容生成器，只输出严格JSON，**禁止任何解释文字、禁止markdown、不要```json标记**。

输出结构规则：
1. 根字段：templateId, subject, preheader, blocks
2. templateId 只能选枚举：["TPL_001","TPL_002","TPL_003","TPL_004"]
3. blocks是数组，每个block的type只能是：["hero_banner","two_column_text_image","bullet_list","cta_block","text_block"]
4. 不能新增schema之外的字段，不能少必填字段
5. 每个block只填写该type对应的字段，不要填充无关字段

参考输出样例：
{
  "templateId": "TPL_001",
  "subject": "New document parsing feature released",
  "preheader": "Try our knowledge base parsing function",
  "blocks": [
    {
      "type": "hero_banner",
      "imageUrl": "https://cdn.xxx/banner.png",
      "altText": "document parse banner",
      "title": "New Release: Document Knowledge Extraction"
    }
  ]
}

你的任务：根据用户需求生成邮件JSON。

*邮件文案生成Prompt*
你是营销邮件文案专家。
根据目标人群、营销目标、参考素材，生成营销邮件，输出JSON。
输出字段：email_subject邮件标题、email_body邮件正文、cta_text按钮文案。
约束：
1. 邮件必须包含合规退订提示；
2. 文案贴合目标人群特征；
3. 语言简洁，适合SaaS营销邮件；
4. 只输出纯JSON，禁止markdown、额外解释。

目标人群：{{customerSegment}}
营销目标：{{campaign_goal}}
文案风格：{{email_tone}}
参考素材：{{rag_material}}



*节点 3：意图识别 Prompt（只干一件事：分类）*
你是意图分类器。
用户请求是生成营销相关内容，只能输出下面其中一个单词，不要多余文字：
marketing_copy（营销短文案）、email_template（邮件模板）、rewrite（改写文案）
用户输入：{user_query}

```
["生成营销邮件","人群洞察分析","生成客户Journey","修改已有邮件","查询历史文案"]
```

*节点 4：参数提取 Prompt（干一件事：抽取字段）*
你是参数提取助手，基于用户原始请求和识别出来的意图【{intent}】，提取下面字段，严格输出JSON，不要解释。
字段列表：customerId、recipient、email_purpose、product、tone、call_to_action
用户输入：{user_query}

*合规检查节点*
先本地正则匹配敏感词，再用llm做深度校验
Prompt:
你是海外营销内容合规审核员，检查这封营销邮件文案，遵循CAN-SPAM、GDPR规则。
检查清单：
1. 邮件主题不能存在误导、虚假紧急信息；
2. 营销邮件必须包含：有效的实体公司地址、清晰一键退订(unsubscribe)入口提示；
3. 禁止绝对化用语：best、No.1、100% guarantee、permanent等；
4. 禁止虚构产品效果、承诺确定收益；
5. 面向欧盟用户：文案不可假定用户已同意接收营销邮件(opt-in)；
6. 不得伪造发件人身份，不能伪装成账单、官方通知。
输出JSON：is_compliance_ok(bool), reason(违规点说明)
待审核文案：{draft}

## 节点清单

1. **节点 1：意图 & 参数解析节点（parse_input）**
接收用户原始 prompt，提取：营销目标、客户类型、邮件风格、可选的图片素材 ID。
输出结构化输入信息，给后面 RAG 使用。
2. **节点 2：RAG 检索节点（retrieve_material）**
检索知识库：产品介绍、卖点、历史营销文案、图片素材信息
输出参考素材（就是你前面说的：检索计划、元数据过滤、重排、检索评估）
3. **节点 3：LLM 生成邮件 JSON 节点（generate_email_json）**
把用户需求 + RAG 素材一起丢给大模型（Function Calling）
输出原始 JSON 字符串
4. **节点 4：JSON 结构校验节点（schema_validate）**
清洗 markdown 标记、json.loads、JSON Schema 校验
✅ 校验通过 → 往下走
❌ 校验失败 → **路由回节点 3 重试**（最多重试 2 次，重试耗尽走失败分支）
5. **节点 5：文案合规检查节点（content_compliance_check）**
遍历 blocks 里面所有文本，做合规检查（广告话术、海外营销合规、敏感词）
✅ 合规通过 → 继续
❌ 不合规 → 路由回节点 3，附带合规错误信息，重写 JSON
6. **节点 6：组装返回数据节点（assemble_result）**
把校验完成的邮件 JSON，带上模板描述、素材信息，打包，返回给 Java 后端

> 
> 这里**不做 Handlebars 渲染**！渲染交给 Java 后端 / 前端预览，Agent 只产出 JSON。
7. **节点 7：异常兜底节点（fallback_error）**
重试耗尽、大模型报错、RAG 超时，统一走到这个节点，返回友好错误信息，供前端展示


*边界和坑点（项目落地重点）*
1. 接口异常处理（LangGraph 状态容错）
Java 服务可能超时 / 数据库查询失败，这个节点要做异常捕获：
   - 捕获 http 超时 / 500：可以设置降级策略，**不带客户个性化信息继续生成通用邮件**，state 里标记警告，不直接中断整个 Agent 流程（LangGraph 很适合做这种降级分支）
   - 重试：可以给这个节点加 1~2 次重试。
2. 区分两种数据，不要混在一起
   - MySQL：结构化、业务、租户、客户、模板元数据（走 Java 接口）
   - PGVector：文档向量化片段（Python 直接查询）

*降级策略*
## 场景 1：Java 接口查询客户信息失败（重点，前面聊的节点）
1. **捕获异常**：在 LangGraph 的`fetch_business_data`节点，捕获 httpx 请求的超时、连接异常、5xx 错误
2. **降级标记**：往 state 写入`business_meta = None`，增加标记`customer_api_fallback = true`
3. **文案生成节点读取标记**
   - 如果`customer_api_fallback=true`，Prompt 里就**不再要求填充客户个性化字段**
   - 改成生成通用模板邮件，去掉`尊敬的【客户姓名】`这种动态变量，写成通用版本

> 
> 例子：
> 正常版：尊敬的张先生，您的标准版知识库…
> 降级保底版：尊敬的客户，我们新上线知识库文档解析功能…

4. 日志埋点：记录告警（推 Grafana / 告警），告诉运维：客户信息接口异常，已经触发降级

## 二、场景 2：PGVector 向量检索 RAG 失败

正常：拿到产品参考文档片段，文案基于真实产品资料生成
故障：向量库超时、连接失败
降级策略：

- state 标记`rag_fallback = true`
- 生成节点 Prompt：**不再使用检索文档，改用内置固定产品基础话术**（提前写死在 Prompt system 里的产品简介）

> 
> 缺点：可能缺少最新产品细节，但至少能产出可用邮件，不会直接失败。

## 三、场景 3：合规检查节点降级（可选）

正常：LLM 做合规校验，不合规就循环重写
故障：合规 LLM 调用超时
降级策略：

- 跳过 AI 合规检查，**切换为轻量本地敏感词正则过滤**（提前内置敏感词列表）

> 
> 风险：没有大模型深度合规，只做简单词匹配，适合临时兜底，后续后台记录待人工复核。

## 四、降级 和 熔断、重试 的区别（面试常问）

1. **重试（retry）**：失败了，马上再试 1~2 次。比如 Java 接口超时，先重试一次，重试仍然失败，**再触发降级**。重试放在降级前面！

> 顺序：调用接口 → 失败 → 重试 (1~2 次) → 重试依旧失败 → 开启降级

2. **熔断（circuit breaker）**：短时间大量请求连续失败，直接断开，一段时间内不再发起请求，直接走降级。适合雪崩防护（推荐用在调用 Java 接口这里，比如 tenacity + pybreaker）
3. **降级（fallback）**：依赖挂了，换保底输出，业务不中断。

> 执行顺序：**重试 → 熔断判断 → 降级兜底**