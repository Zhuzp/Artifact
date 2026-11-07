事件入口：
ingest/event → Kafka 用户事件 → Java 匹配
           → Kafka 执行任务 → Java 执行器

人群入口：
发布 → Kafka 人群任务 → Java 分页拆分成员
     → Kafka 执行任务 → Java 执行器

**画布编辑**
*涉及表*
`journey_workflow`-`JourneyEntity`
主要key:uuid,status(draft running scheduled),workflow_version,nodes_json,edges_json,schedule_time,revision,ui_visualization,deleted_at
*主流程*
JourneyController收到/api/journey请求后，转给JouneyService。
涉及到Journey内容修改的，需要根据revision加乐观锁，sql修改时根据uuid和当前revision去筛选。乐观锁实现在JourneyEditLock里。
JourneyService负责Journey的增删改
`POST /journey` `PUT /journey` `DELETE /journey` `GET /journey/get/{uuid}`
/journey/get/{uuid}：走 JourneyService.get，会更新 viewed_at，缺少 Start 节点时还会补建。

改节点则交给JourneyGraphService
`POST /node` `PUT /node` `DELETE /node` `PUT /edge` `DELETE /edges`
1.新增修改节点就是先根据uuid获取整个journey，然后取出nodes_json，在里面新增一条节点对象，再保存

发布、升版本、暂停交给JourneyLifecycleServie
`POST /journey/bump-version`
bump-version是想修改一条正在running的Journey时调用的，他会把workflow_version+1,running状态改为draft并标记is_older_version_running为true，然后再改节点和连线。

列表和筛选交给JourneyQueryService。
 `GET /journey` `GET /journey/versions?uuid=`


“暂停”建议改成“切换 suppress 标志”。
Lifecycle 对应接口应补上 POST /journey/submit 和 POST /journey/suppress。当前 toggleSuppress 只修改布尔字段，不改变 status；从当前匹配和执行代码看，也没有根据该字段阻止执行，因此暂时不能概括为已实现完整暂停功能。

**发布**
*journey_published_version*
主要Key:workflow_uuid,workflow_version,definition_json

*主流程*
JourneyLifecycleService.submit 做发布。先用JourneyDefinitionValidator.validate()做校验，检查画布合法性。通过后生成新版本workflowVersion，状态改成running，或者改成scheduled并记下将来的时间。
然后调用JourneyPublishedVersionService.publish，将当前旅程的节点、连线、配置和引用模板的实际内容保存为不可变发布快照，存入journey_published_version表definition_json字段。之后改画布、改模版都不影响已经发布的这一版。执行任务及等待恢复始终按绑定的旅程 UUID 和发布版本读取快照。

3. Audience入口
 1. 起点是Audiences 时，submit把状态写成 running 并提交发布版本之后，会调用AudienceEnqueueService.enqueueIfAudienceStart，该方法会判断起点是不是audience人群触发，是的话就调用audienceFanoutService.create，这个方法会在audience_delivery_job里插入一条数据，状态改为SNAPSHOTTING。
 2. AudienceTaskDispatcher会定时200ms扫描audience_delivery_job库，dueJobs 查询一批到期SNAPSHOTTING任务。循环每个job，claimJob原子抢占(就是a查询到这条数据后，会将next_dispatch_at往后推30秒，其他查到这条数据想要update的，筛选条件next_dispatch_at已经>now了，就跳过这条任务了)，抢到就JourneyRunPublisher 往 rengage.audience-jobs 发一条AudienceJobMessage消息,消息里带着 jobId 和当前 step,通知worker开始做快照分页。
 3. AudienceJobConsumer接收到消息后，会调用AudienceFanoutService.snapshotPage(这个方法是分页拉取当前人群成员，默认500人，每个人用userId做CRC32，写入audience_delivery_member，写完后游标停在这一页最后一个人，定时器领走，再发一条kafka消息，直到小于500，说明这个人群结束了)。
 分片完成后，调用AudienceTaskStore.finishSnapshot，根据已经存在的member记录，自动在audience_delivery_shard生成shard分片主记录，状态为READY，交给Dispatcher后续调度投递。
 4. AudienceTaskDispatcher扫描 dueShards（audience_delivery_shard，status=READY，next_dispatch_at<=now），发送AudienceShardMessage。
 5. AudienceShardConsumer接到消息后，会调用AudienceShardService.dispatch(具体看代码吧，还不太理解)，向 Redis 要一个许可。拿到许可后，JourneyRunPublisher 往 rengage.journey-runs 发一条这个人的 run。
 6. JourneyRunConsumer收到消息后，用 `runId` 做幂等控制，启动单用户画布旅程，由 `LocalJourneyExecutor` 驱动节点流转；支持三种结果：成功完成、遇到等待节点挂起、异常失败(抛异常并进入重试，耗尽后进入死信)

*这是dispatch流程*
1. 检查 job 是否为 DISPATCHING，否则结束。
2. 开启事务，FOR UPDATE 锁住该 shard，检查状态为 READY，数据库游标等于当前预期游标；旧消息游标不匹配就退出。
3. 查询该分片中 id > cursor_id 的下一位成员。没有下一位，就把 shard 标成 DONE。
4. 向 Redis 申请全局 fanout 许可，当前配置每秒 200 个。最多等待 1 秒；没拿到就暂停本轮，保留进度。
5. 根据 jobId + workflowUuid + userId 生成稳定 runId，携带 job 固定的 workflowVersion，发送 JourneyRunMessage，等待 Kafka ACK。
6. ACK 成功后更新 shard 游标并提交该成员的事务，然后处理下一位。

**定时发布**
*如何定时发布*
 1. 打开处于 Draft 状态的 Journey。
 2. 点击右上角 Set to Live。
 3. 在 Set Journey to Live 弹窗中，找到 Schedule start。
 4. 打开 Specific date and time 开关。
 5. 选择 Date、Time、Timezone，点击 Confirm。
*主流程*
1. 发布后，校验画布，版本加一，状态改为scheduled,scheduleTime 写成 18:00，startTime 留空。scheduleTime 必须是将来的时间。JourneyPublishedVersionService.publish 仍会把这一版画布存成快照。
如果这次定时发布时，上一版还在 running，旧版本会继续接事件。新版本要等到点才替换它。
2. 本进程定时任务JourneyScheduleDispatcher，默认每秒调一次。查status=scheduled且schedule_time 已经到了的旅程，一次最多100条。
3. 到点后，JourneyScheduleDispatcher.activateDue把这条旅程从scheduled更新成running,startTime协程现在。更新条件要求状态仍是scheduled，所以两个实例同时扫到时，只有一个能更新成功。
更新成功后，执行AudienceEnqueueService.enqueueIfAudienceStart。剩下的就跟其他的一样了。


**事件入口**
1. 发布后，客户系统有用户触发事件的时候，会调用/ingest/event,IngestController调用UserEventPublisher.publish, 把这条事件写到rengage.user-events。
2. UserEventMatchConsumer接收到消息后，调用LiveEventIndex.match("signup")，LiveEventIndex先查Redis。有缓存就直接用。没有缓存时，JourneyPublishedVersionService.activeWorkflows读出当前已发布的journey，再由EventJourneyDispatcher.startEventKeys看每个jouerney的起点。trigger_type 是 Event，并且 event_type、event_type_id 或起点条件里的事件名对上 signup，这条旅程才算命中。命中结果写回 Redis，下次同样的事件名不用再扫库。一个没命中，这个消费者就结束，不会进任何journey
3. 命中几条旅程就往rengage.journey-runs发几条消息。每条带这个旅程的uuid,workflow_version,id和邮箱，trigger写成EVENT。（JourneyRunPublisher发布消息）
4. JourneyRunConsumer接收到消息后，用 `runId` 做幂等控制，启动单用户画布旅程，由 `LocalJourneyExecutor` 驱动节点流转；支持三种结果：成功完成、遇到等待节点挂起、异常失败(抛异常并进入重试，耗尽后进入死信)。
同一条 rengage.user-events 消息还有另一个监听者 UserEventWaitResumeConsumer。它不负责启动新旅程，只唤醒已经停在「等待这个事件」节点上的人。（这个流程还不太懂）


`LocalJourneyExecutor` 
一次执行怎么走
 1. run 和 resumeAfterNode 都进 walk。

 2. 用 workflowUuid 和 workflowVersion 读取已发布快照，不读当前草稿。
 把这次 runId 记到 journey_execution_instance。同一个 runId 再进来时，必须还是同一个旅程、版本和用户。
 3. 向 Redis 要一次 journey-execution 许可，默认每秒 100 次。
 从起点开始做广度优先。resumeAfterNode 不重新执行等待节点，直接把它的出边放进队列。
 每个节点先在 journey_node_execution 占一个检查点，再执行。这个节点以前已经成功过，就直接用当时的结果，不再执行一次。
节点返回 WAIT 时，它的下游不入队。整次结果是 waiting。全部走完是 ok，节点抛错是 error，画布没有节点是 empty。

**邮件投递的完整闭环**
邮件节点写 outbox → 后台限流发送 → 成功或退避重试 → 耗尽后 DEAD → 人工重放。尤其要讲清：Journey SUCCESS 不代表邮件送达，SMTP 也无法保证严格只发送一次。
 1. 进入邮件节点后，会调用LocalJourneyExecutor.handleEmail，这个方法里会调用JourneyMailOutbox.enqueue。用发布快照里的模版（也就是journey_published_version表definition_json），插入journey_mail_outbox，写入 outbox 的 payload_json。
 2. 执行器入队后就继续走下游，不等邮件发送成功。
 3. JourneyMailOutbox.dispatchDue默认每秒跑一次。 在journey_mail_outbox表查PENDING状态且next_attempt_at已到的记录，一次最多50条。每条单开事务，用SELECT...FOR UPDATE锁住。
 4. EmailSendService.sendHtmlWithDeliveryId
 先校验 + 兜底补全所有邮件字段->判断是否是干跑模式 dry-run->确认发邮件的 Spring 组件准备好了->组装一封标准 MIME HTML 邮件->拿限流令牌->调用 SMTP 发送->发送出异常（网络、SMTP 报错、服务商拒收等）
 5. 发送失败就重试，重试次数用完，默认最多5次，状态改为DEAD,定时器不再捞。
 6. 人工把 DEAD 重新放回队列。白名单里的运维调用 POST /api/journey-operations/deliveries/{id}/replay，实现是 JourneyMailOutbox.replay。只有状态是DEAD的行会被改为PENDING,attempts归零



1. 执行器内部怎么跑。
LocalJourneyExecutor 如何遍历图、选择分支、执行节点；等待时保存什么，恢复时从哪里继续。还要知道当前限制，例如节点检查点按每个实例的节点唯一，不能直接支持循环多轮执行。

2. 幂等和事务边界——最值得准备。
能解释清楚三层保护：消息 runId 去重、节点检查点、邮件 outbox。重点回答：
- Kafka 收到消息，但数据库游标没提交就宕机，会怎样？
- 邮件节点完成，后面节点失败，重试会不会重复生成邮件？
- 同一用户两个事件同时唤醒等待，如何只推进一次？
- 客户重复调用 HTTP，为什么不一定能去重？


4. 容量、性能和排障。
理解 Kafka partition、消费者并发、业务分片、全局限流之间的关系。能回答“十万用户要多久”“积压了怎么看”“数据库慢还是消费者慢怎么区分”。
这块最好实际跑一次压测，记录机器配置、并发、吞吐、耗时和瓶颈。配置的每秒 1,000 不是实测吞吐，不能直接当项目成绩。

你的贡献和设计取舍。
准备两三个你实际解决的问题：原来有什么问题、为什么选这个方案、改了什么、怎么验证、还有什么限制。像“为什么数据库保存任务后还需要 Kafka”“为什么采用整份 JSON 存图”“为什么发布快照独立于编辑主表”，都很适合追问。