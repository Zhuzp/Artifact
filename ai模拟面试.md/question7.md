下一题：权限和安全
假设一个租户下有多个部门，用户只能检索自己有权限访问的文档。
请回答：
1. 权限数据放在 Java MySQL 还是 Python PostgreSQL？
2. Java 如何把用户权限传递给 Python？
3. pgvector 和 Elasticsearch 检索时如何做权限过滤？
4. 为什么不能先召回 20 条，再在生成前过滤？
5. 如果用户伪造 tenant_id 或 owner_ids，如何防止越权？

*my answer*
1.权限数据放在mysql，pg chunk表里存放owner_id用于检索过滤
2.通过kafka传递
3.检索时先算可访问的owner列表，然后做检索查询时用owner_ids去表里匹配owner_id
4.那可能20条都没权限查看
5.不知道

*ai answer*
你的回答大约 **7/10**，前四点基本正确，第 2 点需要调整，第 5 点是关键安全问题。

### 1. 权限数据放 MySQL，Chunk 保存 `owner_id`

对。Java MySQL 维护用户、部门、角色和文档权限；Python 的 `document_chunk` 或 ES 文档中保存检索所需的权限字段，例如：

```text
tenant_id
owner_id
department_id
visibility
```

这样 Python 才能在检索引擎内部过滤。

### 2. 查询时不建议通过 Kafka 传权限

Kafka 适合传异步事件，例如：

```text
DOCUMENT_INDEX
DOCUMENT_PERMISSION_CHANGED
```

用户实时问答时，Java 需要立刻拿到检索结果，这种调用一般使用：

```text
Java -> Python 内部 HTTP/gRPC
```

Java 在请求中传递经过认证的权限上下文：

```json
{
  "tenant_id": "t1",
  "user_id": "u1",
  "department_ids": ["d1", "d2"]
}
```

Kafka 可以用于同步权限变更或文档状态变更，但不适合每次查询临时传权限。

### 3. 召回前过滤

你的思路正确。查询时带上：

```text
tenant_id = t1
AND owner_id IN (...)
AND version = current_version
```

如果权限范围不仅是 `owner_id`，还要支持部门、角色和公开文档：

```text
tenant_id = t1
AND (
  owner_id IN (...)
  OR department_id IN (...)
  OR visibility = 'PUBLIC'
)
```

这个过滤必须进入 pgvector 和 ES 的召回查询条件。

### 4. 为什么不能召回后再过滤

你说“20 条都没权限查看”只是一个问题。更严重的是：

- 权限文档会占用召回名额；
- 过滤后可能没有足够候选；
- 不同检索器的结果可能泄露文档存在性；
- 生成模型可能在过滤前接触到无权内容；
- 可能造成越权缓存。

正确流程是：

```text
权限过滤
-> 向量/BM25 召回
-> RRF
-> Rerank
-> 生成
```

### 5. 如何防止用户伪造 `tenant_id` 或 `owner_ids`

不能信任前端传来的 `tenant_id`、`owner_ids`。应该：

1. 用户登录后拿到身份凭证；
2. Java 从服务端会话或 Token 中识别用户；
3. Java 查询或计算真实租户、部门和角色；
4. Java 通过服务间认证调用 Python；
5. Python 只接受可信的服务端权限上下文，不接受前端直接传入的权限范围；
6. Python 校验 `tenant_id` 和权限上下文的签名或内部 Token。

可以这样表达：

> `tenant_id` 和 `owner_ids` 不能由前端决定。Java 从登录身份和权限系统中计算权限范围，再通过带服务认证的内部请求传给 Python。Python 对租户字段做强制过滤，并拒绝没有合法服务身份或签名的请求。

你这题面试中的完整回答可以组织成：

> 权限的权威数据存放在 Java 的 MySQL，Python 的 Chunk 和 ES 文档保存检索所需的 `tenant_id`、`owner_id`、`department_id` 等字段。用户请求到 Java 后，Java 根据登录身份计算可访问范围，通过内部 HTTP 或 gRPC 将权限上下文传给 Python；Kafka 用于权限变更的异步同步，不用于实时查询传参。Python 在向量检索和 BM25 查询阶段就加入权限过滤和当前版本过滤，再进行 RRF、Rerank 和生成。前端不能直接传 tenant 或 owner 范围，Python 只信任经过服务认证的 Java 请求。

