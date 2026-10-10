现在假设有两个 Python 服务实例，同时扫描到同一条 outbox 记录，请回答：
1. 如何保证只有一个实例负责发送？
2. 如果实例 A 已经拿到锁，但发送 Kafka 时宕机，实例 B 什么时候可以接管？
3. Kafka 已经发送成功，但数据库仍然是 NEW，重复发送后 Java 如何保证不重复更新？
4. SENT 状态应该在 Kafka 返回什么结果后才能写入？
请按面试方式直接回答。

*my answer*
1.在查这条数据的时候，加上排他锁
2.要等到下一次relay循环扫描
3.java根据唯一doc_id来查到记录，如果状态已经是parsed，就跳过。不是才更新。
4.Kafka  返回ack，但是我不太懂ack是啥

*ai answer*
这题大约 **7/10**，核心方向对了。

### 1. 排他锁

对，但不能只说“加排他锁”，要说明如何避免锁被长期占用。常见写法是：

```sql
SELECT *
FROM outbox
WHERE status = 'NEW'
  AND (locked_until IS NULL OR locked_until < NOW())
FOR UPDATE SKIP LOCKED
LIMIT 100;
```

拿到记录后更新：

```text
locked_by = worker-1
locked_until = 当前时间 + 60 秒
status = PROCESSING
```

`SKIP LOCKED` 允许第二个 Worker 跳过已经被第一个 Worker 锁住的记录，继续处理其他任务。

### 2. A 崩溃后 B 什么时候接管

你说“下一次扫描”是对的，但还需要一个**租约超时时间**。

例如 A 抢到任务后设置：

```text
locked_until = 10:00:60
```

如果 A 在发送过程中宕机，B 下一轮扫描发现：

```text
locked_until < 当前时间
```

就可以重新接管。实际时间可以是几十秒或几分钟，取决于发送超时和任务耗时。

### 3. 重复发送时 Java 怎么幂等

你的思路正确，但只用 `doc_id` 不够。

同一个文档可能有：

- 解析事件；
- 索引事件；
- 不同版本事件。

所以消息要带：

```text
event_id
document_id
version
event_type
```

Java 可以用 `event_id` 建消费记录表，或者使用条件更新：

```sql
UPDATE document
SET status = 'PARSED'
WHERE document_id = ?
  AND version = ?
  AND status = 'PARSING';
```

第二次重复消费时，状态已经是 `PARSED`，更新行数为 0，于是直接认为已处理。

### 4. Kafka ACK 是什么

Kafka Producer 发送消息后，不是调用函数返回就一定代表 Kafka 已经收到。Producer 会等待 Kafka Broker 返回确认结果，这个确认就是 ACK。

可以简单理解为：

```text
Python Relay -> Kafka Broker
Python Relay <- Broker ACK
```

ACK 表示 Broker 已经按照配置接收并处理了这条消息。

常见配置：

```text
acks=0
```

Producer 不等待确认，速度快，但可能丢消息。

```text
acks=1
```

只等待 Leader 副本确认。

```text
acks=all
```

等待所有同步副本确认，可靠性更高，通常配合副本和 `min.insync.replicas` 使用。

对 Outbox 来说，通常要在 Kafka Producer 返回成功确认后，才把：

```text
outbox.status = SENT
```

如果发送超时或返回错误，就保留为 `NEW` 或改成 `FAILED`，等待重试。

但要记住一个边界：

```text
Kafka 已成功
-> Relay 还没来得及更新 SENT
-> Relay 崩溃
```

重启后可能重复发送。因此 Outbox 通常是至少一次投递，消费端必须幂等。

面试中你可以这样完整回答：

> 我会通过数据库锁或租约保证同一条 Outbox 记录只有一个 Worker 处理，查询时使用 `FOR UPDATE SKIP LOCKED`，并设置 `locked_until`。如果 Worker 宕机，租约过期后其他实例可以接管。Kafka Producer 收到 Broker 的成功 ACK 后，再把 Outbox 更新为 SENT。由于 ACK 成功和数据库更新之间仍可能崩溃，消息可能重复发送，所以 Java 消费端通过 event_id 去重，并结合 document_id、version 做条件更新，保证重复消息不会重复改变业务状态。

