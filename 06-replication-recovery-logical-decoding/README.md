# 复制、恢复与逻辑解码

本目录约 9 个复制、恢复与逻辑解码主题组，目标是理解 WAL 如何支撑 physical streaming、standby recovery、timeline 切换、logical decoding 和 subscription apply。本目录侧重复制、恢复和解码边界，不重复展开 02 中 WAL record 生成、WAL buffer、fsync、checkpoint I/O 等持久化细节。

课程安排：

1. WAL Sender / WAL Receiver 与 physical streaming。
2. Replication Slot 与 WAL / xmin 保留边界。
3. Recovery / REDO / Hot Standby conflict。
4. Timeline、promotion 与历史分叉。
5. Reorder Buffer。
6. Logical Decoding。
7. Output Plugin。
8. Logical apply worker / subscription。
9. 复制延迟、诊断与边界。

第 1 项 `WAL Sender / WAL Receiver 与 physical streaming` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [物理复制连接握手与复制协议入口](01-physical-replication-handshake.md)：standby 如何通过 replication connection、IDENTIFY_SYSTEM、START_REPLICATION 和 system identifier 校验，确认自己正在追随同一个集群历史？
2. [WAL Sender 主循环与发送位置推进](02-walsender-mainloop-send-position.md)：`walsender` 如何在读取 WAL、等待新 WAL、发送 CopyData 和处理反馈之间循环，为什么 sent、write、flush、apply 位置必须分开记录？
3. [WAL Receiver 接收、写入与状态发布](03-walreceiver-write-status-report.md)：`walreceiver` 如何把主库发来的 WAL 流写成本地 segment，并通过 shared state、latch 和状态上报让 startup process 与主库看到接收进度？
4. [同步复制反馈与 commit 等待边界](04-synchronous-replication-feedback-boundary.md)：主库提交事务时为什么可能等待 standby 的 write、flush 或 apply 反馈，`synchronous_commit` 和 synchronous standby 选择如何影响事务可见延迟？
5. [复制协议 keepalive、超时与断线重连](05-replication-keepalive-timeout-reconnect.md)：primary 和 standby 如何用 keepalive、reply_requested、wal_sender_timeout、wal_receiver_timeout 和 reconnect 逻辑区分正常空闲、网络中断和 standby 落后？

第 2 项 `Replication Slot 与 WAL / xmin 保留边界` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Replication Slot 类型与生命周期](06-replication-slot-types-lifecycle.md)：physical slot、logical slot、temporary slot 和 persistent slot 分别保护什么状态，创建、持有、释放和崩溃恢复时哪些字段必须可靠保存？
2. [restart_lsn 与 WAL 保留边界](07-slot-restart-lsn-wal-retention.md)：为什么一个 slot 的 `restart_lsn` 会阻止主库回收旧 WAL，`max_slot_wal_keep_size`、checkpoint 和 WAL recycling 如何把保留需求变成磁盘压力？
3. [catalog_xmin / xmin 与 VACUUM 清理边界](08-slot-xmin-vacuum-boundary.md)：logical slot 为什么会发布 `catalog_xmin`，physical standby feedback 为什么会影响 xmin horizon，哪些场景会让 VACUUM 无法移除旧 tuple 或 catalog 版本？
4. [Slot 持久化、失效与安全删除](09-slot-persistence-invalidation-drop.md)：replication slot 如何落盘、崩溃后如何恢复，WAL 丢失、数据库删除、catalog xmin 过旧和手工 drop 分别会让 slot 进入什么边界状态？
5. [Slot 监控与保留风险诊断](10-slot-monitoring-retention-risk.md)：如何从 `pg_replication_slots`、WAL 目录增长、inactive slot、restart_lsn 滞后和 vacuum bloat 判断问题来自消费者停滞、反馈缺失还是保留策略过宽？

第 3 项 `Recovery / REDO / Hot Standby conflict` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Startup process 与恢复状态机](11-startup-process-recovery-state-machine.md)：standby 启动后如何在 archive recovery、streaming recovery、crash recovery 和 consistent state 之间推进，哪些条件决定数据库何时可以接受只读查询？
2. [REDO record 回放与一致性边界](12-redo-replay-consistency-boundary.md)：startup process 如何按 LSN 顺序应用 WAL record，full-page image、resource manager redo routine 和 checkpoint redo pointer 如何保证数据页回到一致状态？
3. [Hot Standby snapshot 与只读查询可见性](13-hot-standby-snapshot-visibility.md)：standby 上没有本地写事务时，只读查询如何获得可用 snapshot，running-xacts WAL record 和 known-assigned XID 如何让 recovery 中的 MVCC 可见性成立？
4. [Hot Standby conflict 类型与取消策略](14-hot-standby-conflict-cancel-policy.md)：REDO 需要清理 tuple、访问关系文件、推进锁或删除数据库时，为什么 standby 查询会被取消，`max_standby_streaming_delay` 和 feedback 分别牺牲什么？
5. [Recovery pause、target 与可控恢复](15-recovery-pause-target-control.md)：PITR 或 standby 调试时如何用 recovery target、pause、resume 和 promote 控制恢复位置，目标时间、XID、name、LSN 与 timeline 如何共同决定停止点？

第 4 项 `Timeline、promotion 与历史分叉` 建议先拆成 4 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Timeline ID 与历史文件](16-timeline-id-history-file.md)：为什么 promotion 后必须产生新的 timeline，`.history` 文件如何记录父 timeline、分叉 LSN 和恢复路径，避免两个不同历史被误认为同一条 WAL 流？
2. [Promotion、end-of-recovery checkpoint 与新历史起点](17-promotion-end-of-recovery-checkpoint.md)：standby promotion 时 startup process 如何结束 recovery、写 end-of-recovery checkpoint，并把只读恢复实例切换成可写 primary？
3. [Follow timeline 与级联复制切换](18-follow-timeline-cascading-standby.md)：级联 standby 如何发现上游 timeline 变化并继续追随正确历史，为什么 `recovery_target_timeline` 会影响 PITR 和故障转移后的可恢复范围？
4. [Timeline 分叉诊断与错误边界](19-timeline-fork-diagnostics.md)：遇到 requested timeline、missing history file、WAL segment belongs to a different timeline 等错误时，如何判断问题是归档缺失、恢复目标错误还是接到了错误上游？

第 5 项 `Reorder Buffer` 建议先拆成 4 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Reorder Buffer 的存在理由与事务重组边界](20-reorder-buffer-transaction-reassembly.md)：logical decoding 按 WAL 顺序读取变更，为什么还需要按事务提交顺序输出，`ReorderBufferTXN` 如何把跨 record、跨 subxact 的变更重新组织起来？
2. [变更缓存、spill 到磁盘与内存压力](21-reorder-buffer-spill-memory-pressure.md)：大事务或长事务会如何撑大 reorder buffer，什么时候需要把 change spill 到磁盘，`logical_decoding_work_mem` 如何在吞吐、内存和磁盘 I/O 之间折中？
3. [Subtransaction、toast 与 tuple image 组装](22-reorder-buffer-subxact-toast-assembly.md)：logical decoding 如何处理 subtransaction commit/abort、TOAST chunk、old key 和 tuple image，为什么输出插件看到的必须是完整且语义稳定的行级变更？
4. [Reorder Buffer cleanup 与错误恢复](23-reorder-buffer-cleanup-error-recovery.md)：消费者断开、slot 释放、事务 abort 或 decoding ERROR 时，reorder buffer 中的内存、spill 文件和事务状态如何清理，哪些状态必须保留给下次继续解码？

第 6 项 `Logical Decoding` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Logical decoding snapshot 构造](24-logical-decoding-snapshot-build.md)：创建 logical slot 时为什么不能立刻解码所有变更，snapbuild 如何从 WAL 中收集 running transactions、catalog xmin 和一致性点？
2. [WAL record 到 logical change 的转换](25-wal-record-to-logical-change.md)：heap、transaction、relation、database 等 WAL record 如何被 logical decoding 解释成 insert、update、delete、truncate、message 和 commit 事件？
3. [Catalog 访问与历史元数据可见性](26-logical-decoding-catalog-visibility.md)：解码旧 WAL 时为什么还需要访问当时有效的 catalog 元数据，`catalog_xmin` 如何保护 relation schema、type 和 replica identity 的可见版本？
4. [Decoding context、slot 消费位置与确认反馈](27-decoding-context-confirmed-flush.md)：logical decoding 如何用 `confirmed_flush_lsn` 表示消费者已经安全接收的位置，为什么读取位置、输出位置和确认位置不能混为一谈？
5. [Two-phase、streaming transaction 与解码扩展边界](28-logical-decoding-2pc-streaming-boundary.md)：prepared transaction 和 in-progress 大事务为什么需要特殊解码协议，streaming mode 如何降低 apply 端等待整事务提交的压力？

第 7 项 `Output Plugin` 建议先拆成 4 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Output plugin callback 生命周期](29-output-plugin-callback-lifecycle.md)：output plugin 如何通过 startup、begin、change、commit、message、truncate 和 shutdown callback 接管 logical change 输出，哪些 callback 只定义格式而不改变解码正确性？
2. [OutputPluginOptions 与协议能力协商](30-output-plugin-options-protocol.md)：plugin 如何声明 textual / binary output、streaming、two-phase 和 origin 过滤能力，客户端 options 如何影响输出内容和协议行为？
3. [Relation schema、replica identity 与行过滤边界](31-output-plugin-relation-replica-identity.md)：输出 UPDATE / DELETE 时为什么需要 replica identity，plugin 何时能输出 old key、完整 old tuple 或只输出不可定位的变更？
4. [Output plugin 错误、事务边界与兼容性](32-output-plugin-error-compatibility.md)：plugin 在输出过程中 ERROR 会如何影响 slot 消费位置，格式升级、schema 变更和消费者兼容性应该在哪些边界上显式处理？

第 8 项 `Logical apply worker / subscription` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Subscription 元数据与 launcher 调度](33-subscription-metadata-launcher.md)：`pg_subscription`、`pg_subscription_rel` 和 launcher 如何决定启动哪些 apply worker / table sync worker，为什么 subscription 是 catalog 状态而不是单纯连接配置？
2. [Apply worker 连接、拉流与事务应用](34-apply-worker-stream-apply-transaction.md)：apply worker 如何连接 publisher、消费 logical replication protocol，并把远端事务按 commit 顺序应用到本地表？
3. [Initial table sync 与增量复制衔接](35-initial-table-sync-catchup.md)：新订阅或新表加入时，table sync worker 如何拷贝初始数据、追赶增量变更，并把 `pg_subscription_rel` 状态推进到可由主 apply worker 接管？
4. [冲突、约束与 apply ERROR 边界](36-logical-apply-conflict-error-boundary.md)：目标端唯一约束、缺失行、权限、schema 不匹配或 replica identity 不足时，apply worker 为什么会停在事务边界，哪些问题必须由运维或应用语义解决？
5. [Origin、loop prevention 与多源复制边界](37-replication-origin-loop-prevention.md)：logical replication 如何用 replication origin 区分变更来源，为什么双向复制、级联逻辑复制和手工写入需要显式处理回环与冲突策略？

第 9 项 `复制延迟、诊断与边界` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Physical replication lag 的分层诊断](38-physical-replication-lag-diagnostics.md)：如何用 sent_lsn、write_lsn、flush_lsn、replay_lsn、replay timestamp 和 wait events 判断延迟卡在主库发送、网络、standby 写盘、REDO 回放还是查询冲突？
2. [Logical replication lag 的分层诊断](39-logical-replication-lag-diagnostics.md)：如何用 confirmed_flush_lsn、restart_lsn、subscription stats、apply worker wait event 和目标端事务压力判断延迟来自解码、传输、apply 还是冲突重试？
3. [WAL 保留、磁盘增长与可用性取舍](40-wal-retention-availability-tradeoff.md)：复制链路中断时，slot、wal_keep_size、archive、max_slot_wal_keep_size 和 backup retention 如何共同决定是保护消费者继续追赶，还是优先保护 primary 磁盘可用性？
4. [Standby 查询、反馈与主库膨胀取舍](41-standby-feedback-bloat-tradeoff.md)：`hot_standby_feedback` 可以减少查询取消，为什么也可能放大主库 bloat，如何结合 conflict 计数、xmin 滞留和业务查询时长判断取舍？
5. [复制、恢复与解码的故障边界图](42-replication-recovery-decoding-boundary-map.md)：面对延迟、WAL 缺失、slot 膨胀、apply 停滞或 timeline 错误时，如何先判断问题属于 physical streaming、recovery、timeline、logical decoding、output plugin 还是 subscription apply？
