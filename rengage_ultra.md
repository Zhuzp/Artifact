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

**AI对话**

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

# **元数据过滤 Metadata Filter**

向量库（PGVector、Pinecone 这类）里面，每一条向量化文档，除了`embedding向量`和`文本chunk`，**还会附带一堆元数据 metadata（结构化标签）**。
元数据过滤 = **在向量相似度检索之前 / 同时，先根据 metadata 条件做筛选，只在满足条件的 chunk 里面做向量检索**。

> 
> 不是靠 LLM 语义过滤，**是数据库层面的条件过滤，纯代码执行，不耗大模型 token，速度很快**。

 **重排模型提升召回质量**
 1. *BAAI/bge-reranker-v2-m3（首选默认）*
   - 0.56B，轻量，中英 + 多语言，海外英文文档也够用；社区最成熟，生产踩坑少。
   - 适合大多数客户，GPU 开销低，可以单独部署成一个独立 rerank 服务。

如果是给客户自己配置
**你们公司（平台侧）**：提前把各类重排模型的适配器代码全部写好（BGE、Qwen、Cohere、Voyage），做好抽象接口、工厂类、降级逻辑、页面配置面板。
- 你们负责托管**自托管重排服务**（比如平台统一部署一套 BGE-reranker，所有租户共用，作为默认选项）