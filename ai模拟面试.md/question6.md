
**下一题：数据库和 API 设计**

请你设计知识库中的三张核心表：

```text
document
document_version
index_task
```

请回答：

1. 这三张表分别存什么？
2. 为什么不把所有字段都放在 `document` 一张表里？
3. 如何表示“用户上传了新版本，但旧版本仍然可检索”？
4. 如何保证同一文档同一版本不会创建两个索引任务？
5. Java 的 MySQL 和 Python 的 PostgreSQL/pgvector 中，哪些数据应该分别存在哪里？

这一题因为我项目里目前是不支持历史文档检索的，所以没有document_version这个表。
目前的做法是document里加个current_version字段，每次新文档上传后version+1,等向量完成后过几分钟再清除旧文档