现在假设：

```text
Python 消费解析消息
-> 文件解析成功
-> 解析后的 Markdown 已写入 S3
-> document 表状态已更新为 PARSED
-> 还没来得及发送“解析完成” Kafka 消息
-> Python 进程崩溃
```

服务重启后，这个文档已经不会重新触发后续流程了。

请你设计恢复方案，回答：

1. 谁负责发现这个任务卡住了？
2. 如何判断“文件已经解析成功，但完成消息还没发出去”？
3. 重试发送 Kafka 消息时，如何避免 Java 端重复更新状态？
4. 你会选择定时扫描任务表、Outbox 还是其他方案？为什么？

*my answer*
1.在pg里加一张Outbox表，和document表发在一起，解析成功时，把document改成Parsed，同时outbox插一条待发信息，这样可以跟踪kafka的发送状态，怎么发现我不太清楚
2.document表状态已更新为parsed，但是outbox表状态还是new
3.根据唯一值doc_id,看这条记录状态，如果已经是parsed，就不再修改，如果不是，就修改
4.定时扫描任务表 我不太理解，是要再起一个服务吗，那我有个问题，万一这个服务崩了呢，这不是无限套娃吗

*ai answer*
这次回答已经抓到 Outbox 的核心了，约 **8/10**。

### 你的方案为什么成立

解析成功时，在同一个 PostgreSQL 事务里完成：

```text
document.status = PARSED
outbox 插入一条待发送事件
事务提交
```

这样不会出现：

```text
document 已经是 PARSED
但 Kafka 事件记录完全不存在
```

因为两条数据库操作要么一起成功，要么一起回滚。

### “怎么发现”的答案

Outbox 表可以被一个 Relay 定时扫描：

```text
SELECT *
FROM outbox
WHERE status = 'NEW'
   OR (status = 'FAILED' AND next_retry_at <= now())
ORDER BY created_at
LIMIT 100;
```

Relay 发送 Kafka 成功后，把状态更新成 `SENT`。如果 Kafka 发送失败，就记录：

```text
retry_count
last_error
next_retry_at
status = FAILED
```

这个 Relay 可以：

- 放在现有 Python 服务里作为后台任务；
- 放在独立 Worker 中；
- 使用多个实例部署，提高可用性。

你问“如果 Relay 也崩了怎么办”，答案是：它崩溃期间任务不会丢，只是暂时没人发送。服务重启或其他实例接管后，会继续扫描 `NEW` 和到期的 `FAILED` 记录。任务表本身就是持久化恢复依据，不会因为 Worker 崩溃而无限套娃。

多个 Relay 实例同时运行时，需要使用数据库锁或租约，例如：

```sql
SELECT *
FROM outbox
WHERE status = 'NEW'
FOR UPDATE SKIP LOCKED
LIMIT 100;
```

也可以增加：

```text
locked_by
locked_until
```

防止两个实例同时处理同一条记录。

### 还要修正的一点

你说 Java 收到消息后根据 `doc_id` 判断状态，这还不够。因为同一个文档可能有多个版本，旧消息可能晚到。

更安全的是：

```sql
UPDATE document
SET status = 'PARSED'
WHERE document_id = ?
  AND version = ?
  AND status IN ('UPLOADING', 'PARSING');
```

如果更新行数为 0，说明：

- 已经处理过；
- 或者这是旧版本消息；
- 或者状态不允许回退。

这就是幂等和版本校验。

另外，Outbox Relay 也可能出现：

```text
Kafka 已经发送成功
但 Relay 还没来得及把 outbox 改成 SENT
```

重启后它会再次发送，所以消费者仍然必须根据 `event_id` 做幂等。Outbox 通常保证的是**至少一次投递**，不是天然的恰好一次。

### 面试中可以这样总结

> 我会在解析业务事务中同时更新 document 状态并写入 Outbox。后台 Relay 定时扫描 NEW 和可重试的 FAILED 事件，发送 Kafka 成功后更新 SENT，失败则指数退避。Relay 崩溃时事件仍保存在数据库里，重启后可以恢复。由于发送成功和标记 SENT 之间仍可能崩溃，所以 Kafka 消费端必须基于 event_id、document_id 和 version 幂等处理，并通过条件更新避免旧版本消息覆盖新状态。

### 继续追问

现在假设有两个 Python 服务实例，同时扫描到同一条 `outbox` 记录。

请你回答：

1. 如何保证只有一个实例负责发送？
2. 如果实例 A 已经拿到锁，但发送 Kafka 时宕机，实例 B 什么时候可以接管？
3. Kafka 已经发送成功，但数据库仍然是 `NEW`，重复发送后 Java 如何保证不重复更新？
4. `SENT` 状态应该在 Kafka 返回什么结果后才能写入？

ps:relay还不太理解，还没实现过