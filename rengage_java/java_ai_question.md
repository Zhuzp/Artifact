
*Journey 从什么状态变成 running？*
这是立即发布的情况。定时发布则是：draft → scheduled → 到时间后 running。

*发布时保存了哪个版本？*
发布时会确定本次发布版本号，把节点、连线、相关模板内容保存成不可变快照。后续执行任务绑定这个版本，运营修改草稿不会影响它。
发布后会在journey_published_version表里存一条数据，主键是 (workflow_uuid, workflow_version)，只读，节点、连线和模板都在这行的 definition_json 里。

*/api/ingest/event 在 Kafka 开启时做了什么？*
但真正匹配的不是 Kafka，而是 Java 中的 UserEventMatchConsumer：
- Kafka 负责保存和传递事件消息。
- UserEventMatchConsumer 消费消息，执行匹配逻辑。

*谁负责生成 Journey 执行任务？*
UserEventMatchConsumer 本身就是 Java 代码。它匹配到 Journey 后，直接构造 JourneyRunMessage，这条消息就是执行任务，再把任务发到 Kafka。
随后 JourneyRunConsumer 消费这个任务，开始执行。

*谁真正执行节点？*
- JourneyRunConsumer：接收任务、检查重复、记录执行情况。
- LocalJourneyExecutor：真正读取画布、执行节点、推进流程。
所以这条链路可以这样讲：
用户上报事件，Java 把事件发到 Kafka；匹配消费者找到对应 Journey，生成执行任务再发到 Kafka；执行消费者接收任务，调用执行器推进节点。

*为什么匹配到 Journey 后，还要把执行任务再发一次 Kafka，而不直接调用执行器？*
匹配到 Journey 后，把执行任务写入 Kafka，是为了将事件匹配和 Journey 执行解耦。高峰时任务可以在 Kafka 中排队，由消费者按处理能力执行，也便于重试和扩展消费者。若消费者与 API 在同一进程，资源隔离仍有限；需要独立扩容时再部署 worker 实例。

*Kafka 可能把同一条 Journey 执行消息投递两次。系统怎样避免同一个用户重复执行 Journey，甚至重复收到邮件？*
JourneyRunConsumer 拿到消息，会先判断runId是否存在，存在就直接跳过

*为什么“先查 runId 是否存在，再插入记录”还不够，必须有数据库唯一索引？*
原因是两个消费者可能同时查到“记录不存在”，然后都尝试插入。唯一约束会让其中一个插入成功，另一个因重复键失败，从数据库层面挡住并发重复执行。
唯一索引是数据库加的`UNIQUE KEY idx_run_id (run_id)`,runId如果两次插入，id值都一样的话，第二次就会失败。也就是一个runId只能执行一次。

*邮件已经发送成功，但消费者在 Kafka 确认消息前崩溃了。消息重试时，系统怎样避免把同一封邮件再发一次？*
如果 Journey 执行记录和 Outbox 已提交，但 Kafka offset 还没确认，消息再次投递时，消费者会根据 runId 跳过这次 Journey，避免重复创建执行任务。
邮件派发任务如果发送成功，却在把 Outbox 状态更新成 SENT 前崩溃，重试时仍可能再次发送。防止这种重复要靠邮件侧的幂等能力或稳定的投递 ID；runId 主要防的是 Journey 重复执行，不能单独保证邮件绝不重复。

*为什么 Journey 消费者不直接同步调用邮件服务，而是先把待发送邮件写入 Outbox？*
Outbox 的核心就是先把待发送邮件可靠地存进数据库，再由后台任务轮询发送。发送失败会按重试策略重新处理，超过次数后进入 DEAD，可以人工重放。

*journey走到邮件发送节点后，流程是什么样的*
1. Journey 走到邮件节点后，把收件人、邮件内容和状态 PENDING 写入 journey_mail_outbox。
2. 后台定时任务轮询到期的 PENDING 记录，按 delivery_id 找到这条邮件任务并调用发送函数。这里不是 Kafka 消费者。
3. 发送成功后，根据 delivery_id 把状态更新为 SENT。
4. 发送失败则增加 attempts，设置下次重试时间；达到最大次数后改成 DEAD，之后可以人工重放。

*为什么 Journey 执行到 Delay 节点时，不能让当前线程 sleep 几分钟，而要把等待状态保存到数据库，之后再恢复执行？*
Delay 可能持续几分钟甚至几天，sleep 会一直占着线程；大量用户同时等待时，线程很快就会被占满。
更合适的做法是：把 Journey 执行记录、等待节点和到期时间存到数据库，然后释放线程。到期后由后台任务领取这条等待记录，用原来的 runId 和发布版本从 Delay 节点后继续执行。

*如果 Journey 服务在 Delay 等待期间重启，用户还会不会继续执行？为什么？*
Delay 场景里，服务重启后用户仍会继续执行：等待记录和到期时间保存在数据库journey_wait.due_at里，服务恢复后定时任务会扫描到期记录，再从等待节点后继续跑。服务停机期间可能延迟恢复，但等待记录不会只存在内存里而丢失。

*Delay 到期恢复时，为什么要带回原来的 runId 和 workflowVersion？如果恢复时读取 Journey 最新版本，可能会出什么问题？*
如果恢复时读最新版本，等待中的用户可能会接着跑运营刚修改过的流程，导致前后配置不一致。
再精确一点：
- workflowUuid 标识是哪条 Journey。
- workflowVersion 锁定这次执行使用的发布版本。
- runId 标识这个用户进入的这一次执行，便于去重和恢复。
例如用户按版本 3 走到 Delay 等待；运营发布了版本 4。这个用户醒来后仍应从版本 3 的 Delay 后继续，新进入的用户才使用版本 4。

*如果 Delay 到期时有两个 worker 同时扫到同一条等待记录，怎么避免两边都恢复执行、重复发邮件？*
1. Worker 查出到期且状态为 WAITING 的记录。
2. Worker 用带状态条件的更新，把它认领为 PROCESSING。
3. 只有更新成功、影响行数为 1 的 worker 才继续恢复执行；其他 worker 更新不到这条记录，就跳过。
4. 执行成功后改为 DONE。

*worker 把 Delay 记录认领为 PROCESSING 后，如果执行过程中崩溃了，这条等待记录怎么避免永远卡住？*
这个项目的做法是给 PROCESSING 加上领取时间 resumed_at：worker 崩溃后，如果记录持续处于 PROCESSING 超过 2 分钟，定时任务会把它重新捞出来尝试认领。
这不是检测到“服务重启”才处理，而是后台定时扫描数据库。重新执行时会沿用原来的 runId 和 workflowVersion；节点执行记录也会拦住已完成节点，降低恢复重试造成重复副作用的风险。
简单说：
- [JourneyTimerDispatcher.java (line 19)](/D:/code/rengage/backend-canvas/src/main/java/ai/rengage/canvas/journey/JourneyTimerDispatcher.java:19) 是定时调度员。Spring 每 500 毫秒调用 dispatchDue()；它请 Repository 找到期记录，认领成功后调用执行器，让 Journey 从 Delay 节点后继续。
- [JourneyWaitRepository.java (line 37)](/D:/code/rengage/backend-canvas/src/main/java/ai/rengage/canvas/journey/JourneyWaitRepository.java:37) 是数据库访问层。它负责查询到期等待记录，以及用 SQL 把记录从 WAITING 改为 PROCESSING 来认领，最后更新完成状态。
调用顺序是：Dispatcher 定时启动 → Repository 查记录并认领 → Dispatcher 调执行器恢复流程 → Repository 更新等待状态。

*resumeAfterNode() 恢复 Journey 时，为什么从 Delay 节点的下一个节点继续，而不是重新执行 Delay 节点？*
resumeAfterNode() 接收等待节点的 nodeUuid，把它当作已经完成的节点，只把它的下游节点加入执行队列。如果重新执行 Delay，就会再次创建一个新的等待记录，流程会不断重复等待。

*为什么 Event 等待恢复时要先把目标等待记录固定下来，再逐条恢复？*
用户只上报了一次“下单事件”。
系统收到它时，会先记下：这次事件只负责唤醒当时已经在等它的等待记录。流程恢复后，发了欢迎邮件，又走到第二个“等待下单事件”，这时它会继续等，不会被刚才那一次事件立刻再次唤醒。要等用户之后再上报一次下单事件，才会继续发优惠券。
所以不是“把这个 Journey 的所有等待都恢复”，而是“恢复事件到达时已经存在、且匹配这条事件的等待”。这样一次事件就不会让流程连续跳过好几个等待节点。

*同一个用户事件因为 Kafka 重试被消费了两次，怎么避免同一个等待中的 Journey 被恢复两次？*
消费者用稳定的 eventId 做幂等，重复消息跳过；去重记录通过数据库唯一约束保证并发下也只有一次成功。

*如果消费者已经把等待记录标成 PROCESSING，但还没恢复流程就崩溃了，系统应该怎么处理？*
Event 恢复时，JourneyWaitService 会在一个事务里把事件记录从 READY 改为 PROCESSING，把等待记录从 WAITING 改为 DONE，然后调用执行器继续流程。如果恢复失败并抛出异常，事务回滚：事件记录回到 READY，等待记录也回到 WAITING，Kafka 消费失败后可以重试。关键代码在 [JourneyWaitService.java (line 46)](/D:/code/rengage/backend-canvas/src/main/java/ai/rengage/canvas/journey/JourneyWaitService.java:46) 和 [JourneyWaitEventStore.java (line 52)](/D:/code/rengage/backend-canvas/src/main/java/ai/rengage/canvas/journey/JourneyWaitEventStore.java:52)。
Delay 的状态不一样：调度器先把等待记录改成 PROCESSING；如果恢复失败，代码尝试释放回 WAITING，再抛异常。由于也在事务里，失败时事务回滚；如果进程直接崩溃，记录可能留在 PROCESSING，超过两分钟后可再次被扫描认领。代码在 [JourneyTimerDispatcher.java (line 20)](/D:/code/rengage/backend-canvas/src/main/java/ai/rengage/canvas/journey/JourneyTimerDispatcher.java:20)。

*怎么判断一个旧版本已经可以安全清理了？*
核心就是确认没有仍在执行中的用户实例引用这个版本。不过“用户进过流程”不等于一直不能删：流程已结束的用户不再需要它；正在等待 Event 或 Delay 的用户仍需要它。
实际清理前要检查该版本是否还有活跃的 run 或等待记录引用。没有引用了，才可以清理。