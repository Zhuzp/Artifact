**AI Knowledge Center**
Content Hub需要加一个功能，模版打开率、点击率、转化率等效果数据，数据达到一定标准后，属于优质模版，通过kafka自动同步到python知识库。


- 品牌规范：语气、品牌名称、禁用词
- 产品知识：功能、价格、卖点、适用人群
- 行业最佳实践：不同场景和渠道的运营方法
- 合规资料：GDPR、TCPA、CAN-SPAM
- 渠道规范：字符限制、退订文本、变量规范

支持PDF,DOCX,Markdown,TXT,CSV / XLSX,HTML,PNG、JPG、JPEG、WebP

前端上传
→ Java 保存原文件到 S3
→ Java 在 MySQL 创建知识资产记录
→ Java 发 Kafka 解析消息
→ Python 下载原文件并解析
→ Python 把解析产物存回 S3
→ Python 发 Kafka 解析完成消息
→ Java 更新状态为待审核 
→ 用户审核
→ Java 发 Kafka 索引消息
→ Python 分块、向量化、写 PGVector 和 ES
→ Python 发 Kafka 索引完成消息
→ Java 更新状态为可用

文本状态
审核状态：未提交 → 待审核 → 已通过 / 已拒绝
处理状态：已上传 → 解析中 → 已解析 → 索引中 → 可用

PDF,DOCX,Markdown,TXT解析为md
上传的图片和提出的图片生成content.md,metadata.json
Excel解析为rows.jsonl，preview.md，metadata.json

合规文档->Java 确定性规则
上传合规文档
→ Python 解析成 Markdown
→ 人工审核文档
→ 分块向量化
→ AI 从文档中提取“候选规则”
→ 合规人员审核候选规则
→ 发布为 Java 可执行规则

候选规则应该为固定JSON

规则要保留来源：
来源文档
具体章节/chunk
规则版本
审核人
生效时间

**AI的能力**

*模块1：AI 多渠道营销文案生成（核心P0）*

根据场景生成全新 Email/SMS/Push/In-App 文案
参考品牌规范、产品知识、历史优质模板和行业实践生成邮件模板（模型输出结构化 Email DSL,也就是json数据，再由前端去组装   ）
改写当前模板：正式、温柔、促销、简洁
对当前内容做合规预检
推荐现有模板和已审核图片素材


LangGraph：
识别意图
→ 提取 Campaign Brief
→ 检查必要信息
→ 缺信息则追问
→ 生成检索计划
→ 混合检索
→ 生成结构化内容
→ 格式校验
→ Java 合规校验
→ 必要时修复
→ 返回结果

意图可以固定为：
GENERATE        生成新文案
REWRITE         改写当前内容
GENERATE_AB     生成A/B版本
ADAPT_CHANNEL   转换渠道
LOCALIZE        翻译和本地化
CHECK_COMPLIANCE 合规检查
SEARCH_TEMPLATE 搜索现有模板 

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

*模块2：AI 一句话生成营销旅程 Journey（核心P0）*
功能简介：运营自然语言描述业务流程，AI自动解析、生成标准化Journey画布（触发、等待、分支、触达、退出），自动适配海外区域合规策略。
- 自然语言转完整旅程画布
- 生成可直接编辑的草稿，不自动上线

*模块3：AI 人群洞察分析（核心P0）*
功能简介：针对选定用户人群，AI自动拆解人群画像、行为特征、触达偏好、合规风险、运营建议，输出完整洞察报告。
- 人群基础画像自动总结
- 用户行为偏好、渠道偏好分析
- 自动给出文案、渠道、触达时机建议

给美国地区近30天加购未购买的人群生成一封弃购召回邮件，推广 Summer Sale，优惠 15%，语气友好，生成两个版本。

用户描述需求
→ 识别生成意图
→ 补全营销任务
→ Java 获取业务事实
→ Python 拆解检索任务
→ 混合检索不同知识源
→ 重排并组装上下文
→ LLM 生成结构化邮件
→ Java 合规规则校验
→ AI 定向修复
→ 前端预览、编辑、保存
