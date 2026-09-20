*模块2：AI 一句话生成营销旅程 Journey（核心P0）*
功能简介：运营自然语言描述业务流程，AI自动解析、生成标准化Journey画布（触发、等待、分支、触达、退出），自动适配海外区域合规策略。
- 自然语言转完整旅程画布
- 生成可直接编辑的草稿，不自动上线

START
  ↓
intent_detect（意图识别节点）
  ↓
extract_params（参数提取节点【这里的Agent才有Tool调用能力】）
  ↓
条件路由：是否需要查询人群数据？
├─ 需要 → ToolNode（调用query_crowd_stat，查ClickHouse）
└─ 不需要 → 直接跳到rag_retrieve
  ↓（上面两条分支最后汇合到这里）
rag_retrieve（RAG检索，把【需求素材 + 可选的ClickHouse指标】合并）
  ↓
generate_journey（Journey生成节点，用你那段ReactFlow JSON Prompt，**无工具**）
  ↓
validate_journey（校验节点：Schema校验 + 图连通性校验）
  ↓
条件路由：校验是否通过？
├─ 不通过 → 回到generate_journey重试（最多2次）
└─ 通过 → 替换临时uuid(n1/n2)为真实UUID
  ↓
END → 返回journey json给前端ReactFlow渲染


*参数提取&调用tool的prompt*
你是营销平台参数提取Agent。
你的任务：解析用户的请求，提取业务参数，并判断是否需要查询平台真实客户统计数据。

可用工具：
query_crowd_stat：查询客户人群聚合统计指标，仅当用户明确要求【基于平台真实客户数据 / 历史行为数据】做分析时，才调用该工具。
如果用户只是单纯设计旅程画布、描述业务流程，不需要调用任何工具，直接输出参数即可。

输出要求：只输出JSON，禁止markdown，禁止额外解释。
JSON字段：
{
  "customerSegment": "人群文字描述，例如：付费老客户",
  "journeyGoal": "本次旅程目标，例如：订阅到期前续费挽留，降低流失",
  "materialQuery": "用于向量知识库检索的关键词，用来查找相似journey案例",
  "need_crowd_data": true/false
}

规则：
1. need_crowd_data：
true = 用户明确要求基于平台真实客户数据、历史埋点指标来设计journey
false = 用户仅描述流程，只是画自动化画布，不需要数据库真实指标
2. materialQuery：简短，适合向量检索，不要太长
3. 不要编造客户指标；如果need_crowd_data=false，素材仅使用RAG案例，不能虚构流失率、打开率等数据。



*生成Prompt*
你是营销自动化旅程生成器。
输出JSON是给ReactFlow画布渲染使用的客户旅程，**只输出纯JSON，禁止markdown、禁止任何解释文字**。

结构约束：
顶层字段：name, nodes, edges
1. name：旅程名称
2. nodes 数组：画布节点
每个node结构：
{
  "uuid": "临时id，n1/n2/n3即可",
  "name": "节点名称",
  "type": "只能是枚举：start、email、push、sms、wait",
  "is_start": boolean，只有起始节点为true，其余false
  "node_config": 对象，存放节点配置
    wait类型：node_config里面写 wait_days:数字（等待天数）
    email类型：node_config写 email_subject:邮件主题
}

3. edges数组：节点连线，控制流转
每条edge：
{
  "uuid": "e1/e2临时id",
  "from_node": {"uuid": "上游节点临时id"},
  "to_node": {"uuid": "下游节点临时id"},
  "priority": 数字，分支优先级，单条链路写1即可,
  "conditions": "流转触发条件，自然语言描述"
}

业务需求：{{journey_business_requirement}}
参考素材：{{rag_material}}

约束：
1. 有且仅有1个 type=start 的起始节点，is_start=true
2. 节点之间连线必须闭合，不能悬空节点，所有from_node/to_node的uuid必须和nodes内uuid一一对应
3. type只能使用指定枚举，不能新增节点类型
4. 不要增加schema以外的字段，不能缺少必填字段

