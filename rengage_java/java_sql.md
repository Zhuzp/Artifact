`schema.sql` 里一共 19 张表。按一条用户从进人群、进旅程、到发出邮件的顺序，每张表存的是下面这些东西。

## 画布和内容

**`journey_workflow`** 是营销人员正在编辑的那份旅程。一行是一条旅程：名称、状态（草稿、已排期、运行中）、当前版本号、节点和连线的 JSON、定时发布时间和时区、进入人数和完成人数。发布之后这里仍会被改；正在跑的人不会读这张表上的新图。

**`journey_published_version`** 是发布那一刻冻住的旅程。主键是旅程 ID 加版本号，`definition_json` 里是当时的节点、连线和用到的模板正文。执行器按任务上的版本号读这张表，所以发布后改画布不会改到已经在跑的人。

**`content_template`** 是内容模板。一行是一封邮件、一条短信或一条推送的草稿：`type` 区分通道，`subject` 是标题，`content` 是编辑器 JSON，`html_content` 是发出去的 HTML。旅程发布时会把当时的模板抄进 `journey_published_version`，之后改模板不影响已发布的版本。

**`dm_event_schema`** 是事件目录。一行是一种用户事件的名字、标识字段和属性结构，给数据管理和条件节点对照用。用户真正触发的事件不进这张表，事件本身走 Kafka。

## 人群

**`audience`** 是人群定义。一行是一个分群：名字、筛选条件 JSON、人数 `size`、租户。它不存具体有哪些人。

**`audience_user`** 是用户目录。一行是一个终端用户：`user_id`、邮箱、手机号、推送令牌、国家、设备、活跃分、是否弃购。表空的时候会种入 600 个演示用户。短信和推送以后就从这里取手机号和令牌。

**`audience_member`** 是某个人群某一版的成员名单。重建分群时写入新的 `snapshot_version`，旧版本里没被正在进行的扫描钉住的行会被删掉。同一人可以出现在多个版本里，靠 `(人群, 版本, 用户)` 唯一。

**`audience_scan`** 钉住一次读取。批量任务开始翻某一版成员时，记下 `snapshot_version` 和当时最大的成员 `id`（`high_water`）。之后只按 `id` 往后翻，重建出的新版本不会插进这次扫描。

## 一次群发任务

发布一条「按人群启动」的旅程时，在同一个数据库事务里写这三张表的第一张，HTTP 请求本身不展开人群。

**`audience_delivery_job`** 是这次群发的总账。一行是一个任务：要跑哪个旅程的哪个版本、哪几个人群、分成几片、翻页大小、当前翻到哪一页。状态从 `SNAPSHOTTING`（还在抄名单）到 `DISPATCHING`（正在给人生成执行任务），结束是 `DISPATCHED` 或 `EMPTY`。`DISPATCHED` 只表示每个人的执行任务已经发出，不表示邮件已经送达。

**`audience_delivery_member`** 是这次任务抄下来的人。一行是「这个任务里的这个用户」，带邮箱和分片号 `shard_no`。`(job_id, user_id)` 唯一，多个人群里的同一个人只留一条。

**`audience_delivery_shard`** 是一片的进度。一行是「这个任务的第 N 片」，`cursor_id` 记已经推进到成员表的哪一行，状态是 `READY` 或 `DONE`。进程挂了之后靠这张表接着发，不会从头再扫一遍。

## 一个人跑一次旅程

**`journey_run_log`** 是一次执行的结果摘要。一行对应一个 `run_id`（同一次旅程、同一个用户、同一次进入）。状态是 `PROCESSING`、`SUCCESS` 或 `WAITING`。Kafka 重复投递时先看这张表，已有记录就不再跑一遍。执行失败不会在这里写成失败，整笔回滚，交给消息重试。

**`journey_execution_instance`** 把这个 `run_id` 绑死到「哪个旅程、哪个版本、哪个用户」。主键就是 `run_id`。重试如果带着同一个 `run_id` 却指向另一个人或另一个版本，会被拒绝。

**`journey_node_execution`** 是节点检查点。一行是「这次运行的这个节点已经做完」，结果记在 `outcome`。节点事务提交后，重试看到同一条就直接沿用上次的结果，不会把这个节点再执行一次。邮件节点因此不会因为后面的节点失败而再入队一封。

**`journey_workflow_log`** 是给人看的流水。发布、进入、跳过节点、邮件入队、跑完、出错，各写一行标题和描述。它只追加，不参与去重，也不能靠它判断邮件有没有发出去。

## 停下来等

**`journey_wait`** 是一个人停在等待节点上。事件等待时 `wait_kind` 是 `EVENT`，`wait_event` 是要等的事件名；定时等待时是 `TIMER`，`due_at` 是到期时间。状态从 `WAITING` 到 `PROCESSING`，然后是 `DONE` 或 `CANCELLED`。一个节点等多个事件时，会为每个事件插一行，共用同一个 `activation_id`。

**`journey_wait_activation`** 保证这组等待只被唤醒一次。一行就是一个 `activation_id`，状态 `WAITING` 或 `DONE`。第一个匹配的事件把它改成 `DONE`，其余兄弟等待行改成 `CANCELLED`。

**`journey_wait_event`** 是进来的用户事件和当时正在等的那些行的对照。处理前先把匹配到的 `journey_wait` 主键列表冻进 `wait_ids_json`，状态 `READY` → `PROCESSING` → `DONE`。重放同一事件不会改这份名单，也不会去唤醒事件到达之后才新建的等待。

## 发出去和失败收容

**`journey_mail_outbox`** 是待发邮件。邮件节点在自己的事务里插入一行，正文冻在 `payload_json` 里，此时还不连 SMTP。调度器之后按 `PENDING` 且到期的记录去发。发成功是 `SENT`，邮件开关关着是 `DRY_RUN`，多次失败是 `DEAD`。`delivery_id` 由旅程、版本、`run_id` 和节点算出来，同一次节点重试插不进第二行。

**`journey_dead_letter`** 是 Kafka 上没处理掉的消息。一行是一条失败消息：来自哪个 topic、分区和偏移，原始内容，最后的错误。状态 `DEAD` 可以人工重放，重放后改成 `REPLAYED`。这里的 `channel` 是消费链路的名字（例如事件匹配、等待恢复），不是短信或推送通道。