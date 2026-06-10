# 存储与持久化

本目录约 7 个存储主题组，目标是理解 block、buffer、WAL 和文件系统之间的正确性边界。

课程安排：

1. Buffer Manager：buffer identity、pin、content lock、dirty、I/O in progress 与 eviction。
2. WAL / XLog：WAL record、WAL-before-data、flush、segment 与 full page image。
3. Storage Manager / fd / md：relation fork、block addressing、segment file、fd cache 与 fsync queue。
4. Checkpointer / Background Writer / WAL Writer：脏页推进、checkpoint 边界与写入平滑。
5. HeapAM DML 与 HOT update：heap page、tuple header、insert/update/delete、HOT chain 与 page pruning。
6. B-tree search / insert / split：索引页结构、并发搜索、插入、分裂、删除与 WAL。
7. Index AM API 与 index vacuum：table AM / index AM 边界、ambuild、aminsert、amvacuumcleanup 与可见性协作。

第 1 项 `Buffer Manager：pin、dirty、I/O in progress` 建议先拆成 8 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Buffer tag、descriptor 与 buffer identity](01-buffer-tag-descriptor-identity.md)：一个 relation fork 的 block 如何映射到 shared buffer slot，为什么 `BufferTag`、`BufferDesc`、buffer content 和 local refcount 必须分开表达？
2. [Buffer lookup、partition lock 与 mapping table](02-buffer-lookup-partition-lock.md)：`ReadBuffer_common()` 如何用 buffer mapping hash 找到已有 page，为什么 lookup 需要分区锁而不能让所有 backend 竞争一把全局锁？
3. [Buffer allocation、clock sweep 与 replacement strategy](03-buffer-allocation-clock-sweep.md)：当目标 block 不在 buffer pool 中时，`StrategyGetBuffer()` 如何在 usage_count、pin、dirty 和 access strategy 之间选择 victim？
4. [Buffer pin 与 content lock 的职责边界](04-buffer-pin-content-lock-boundary.md)：为什么“防止 buffer 被替换”和“保护 page 内容并发读写”必须由 pin、shared/exclusive content lock 分别承担？
5. [Buffer read I/O、BM_IO_IN_PROGRESS 与等待协议](05-buffer-read-io-in-progress.md)：多个 backend 同时读取同一个 block 时，谁发起实际 I/O，其他 backend 如何通过 buffer state、condition variable 和 error cleanup 等待或重试？
6. [Dirty buffer、page LSN 与 MarkBufferDirty](06-buffer-dirty-page-lsn.md)：一个 page 修改后如何从 clean 变 dirty，为什么写脏页前必须比较 page LSN 和 flushed WAL LSN？
7. [Eviction、writeback 与 buffer cleanup 边界](07-buffer-eviction-writeback-cleanup.md)：一个 dirty victim 在被复用前必须经历哪些写回、校验和、WAL flush 与状态清理，哪些等待会放大 buffer miss 成本？
8. [Relation extension、bulk read 与 access strategy](08-buffer-extension-bulkread-strategy.md)：顺序扫描、批量写入和 relation extension 为什么需要不同 buffer access strategy，如何避免大查询冲刷整个 shared buffers？

第 2 项 `WAL / XLog：WAL-before-data 与 flush 边界` 建议先拆成 7 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [WAL record 结构、rmgr 与 page 修改描述](09-wal-record-rmgr-page-change.md)：为什么一次数据页修改要被压缩成 WAL record，`XLogBeginInsert()` / `XLogRegisterBuffer()` / `XLogInsert()` 如何把 buffer delta、block reference 和 rmgr 语义绑定起来？
2. [WAL insertion、WAL buffers 与 insert lock](10-wal-insertion-buffer-locking.md)：多个 backend 并发产生 WAL 时，如何在保留 LSN、拷贝 record、推进 insert position 和减少锁竞争之间折中？
3. [WAL-before-data、page LSN 与持久化顺序](11-wal-before-data-page-lsn.md)：为什么 data page 可以晚写但不能早于对应 WAL 持久化，buffer write 路径如何用 page LSN 守住 crash safety 边界？
4. [Commit record、WAL flush 与 synchronous_commit](12-wal-flush-commit-synchronous-commit.md)：事务提交为什么通常要等待 WAL flush，`synchronous_commit`、group commit 和 backend sleep/wakeup 如何在 latency 与 durability 之间取舍？
5. [WAL segment、switch、recycle 与 archive 边界](13-wal-segment-switch-recycle-archive.md)：WAL 字节流如何落到固定大小 segment 文件，segment 什么时候切换、回收、保留或交给 archiver？
6. [Full page image、torn page 与 hint bit WAL 边界](14-full-page-writes-torn-page.md)：为什么 checkpoint 后首次修改 page 通常要写 full page image，hint bit、checksum 和 torn page 风险如何影响 WAL 体积？
7. [WAL resource manager 与 redo contract](15-wal-resource-manager-redo-contract.md)：一个 rmgr 的 WAL record 必须给 crash recovery 留下什么信息，为什么 redo 要尽量幂等并能处理 page 已经部分落盘的状态？

第 3 项 `Storage Manager / fd / md：fork、segment、fd cache` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Relation fork 与 block addressing](16-relation-fork-block-addressing.md)：`main`、`fsm`、`vm`、`init` fork 如何共同表达一个 relation 的持久化状态，block number 到文件偏移的边界在哪里？
2. [smgr/md segment create、extend、truncate](17-smgr-md-segment-lifecycle.md)：一个 relation 文件如何按 1GB segment 拆分，`mdextend()`、`mdread()`、`mdwrite()`、`mdtruncate()` 如何处理稀疏增长、EOF 和并发扩展？
3. [fd.c、VFD cache 与文件描述符压力](18-fd-cache-vfd-lru.md)：PostgreSQL 为什么不直接长期持有所有 OS fd，VFD LRU 如何在大量 relation、临时文件和进程 fd 限制之间折中？
4. [临时文件、BufFile 与 spill I/O 边界](19-temp-file-buffile-spill-boundary.md)：排序、hash、materialize 等执行节点写临时文件时，如何通过 `BufFile`、临时 tablespace 和 resource cleanup 管理可删除但不可 WAL 恢复的状态？
5. [fsync request queue 与 pending operations](20-fsync-request-queue-pendingops.md)：backend 写 relation 文件后，为什么常把 fsync 责任交给 checkpointer，pending ops 如何避免“写了数据页但忘记落盘”的持久化漏洞？

第 4 项 `Checkpointer / Background Writer / WAL Writer` 建议先拆成 6 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Checkpoint lifecycle 与 redo pointer](21-checkpoint-lifecycle-redo-pointer.md)：checkpoint 如何把“从哪个 WAL LSN 开始恢复”推进到更近的位置，为什么 checkpoint record、control file 和 dirty page flush 必须按固定顺序配合？
2. [Checkpointer 脏页扫描、写回与 fsync](22-checkpointer-dirty-page-fsync.md)：checkpointer 如何遍历 buffer pool、写出 dirty buffer、处理 fsync request，并在超时、失败和重试之间维持 crash safety？
3. [Background Writer 与 reusable clean buffer](23-bgwriter-clean-buffer-smoothing.md)：bgwriter 为什么只负责提前制造 clean buffer，而不负责完成 checkpoint correctness，它如何用 buffer access strategy 和统计反馈平滑前台写入压力？
4. [WAL Writer 与异步 WAL flush](24-wal-writer-async-flush.md)：walwriter 如何把 backend 产生的 WAL 周期性刷盘，为什么它能降低提交等待但不能替代需要同步持久化的 commit flush？
5. [Checkpoint 调优、写放大与延迟尖刺](25-checkpoint-tuning-write-amplification.md)：`max_wal_size`、`checkpoint_timeout`、`checkpoint_completion_target` 如何影响脏页积累、WAL 保留、恢复时间和写 I/O 峰值？
6. [Crash restart：从 control file 到一致性点](26-crash-restart-consistency-point.md)：服务器崩溃后如何从 control file 找到 checkpoint，再通过 WAL redo 把 data directory 推进到一致状态，哪些文件系统状态仍可能需要清理？

第 5 项 `HeapAM DML 与 HOT update` 建议先拆成 8 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Heap page layout、line pointer 与 tuple header](27-heap-page-layout-tuple-header.md)：一个 heap page 如何用 page header、line pointer、tuple header 和 infomask 同时服务定位、可见性和空间复用？
2. [Heap insert、FSM 与 page selection](28-heap-insert-fsm-page-selection.md)：`heap_insert()` 如何选择目标 page、插入 tuple、更新 FSM，并生成足够的 WAL 让 crash recovery 重放插入？
3. [Heap update、new version 与 old tuple xmax](29-heap-update-version-chain.md)：一次 UPDATE 为什么通常生成新 tuple version，旧版本的 `xmax`、新版本的 `xmin` 和 ctid chain 如何表达版本演进？
4. [HOT update 与 index 维护边界](30-hot-update-index-boundary.md)：什么时候 UPDATE 可以不新增 index entry，HOT chain 如何把“索引仍指向老 line pointer”和“读者能找到新版本”同时成立？
5. [Heap delete、tuple lock 与 page pruning 入口](31-heap-delete-prune-entry.md)：DELETE 如何把 tuple 标记为已删除但不立刻释放空间，后续谁在什么可见性边界下触发 pruning？
6. [Heap page pruning、redirect/dead line pointer 与空间回收](32-heap-page-pruning-line-pointer.md)：page pruning 如何在不破坏现有 index TID 的前提下压缩 HOT chain、标记 DEAD line pointer，并把空间还给 page 内部？
7. [FSM、VM 与 heap storage 协作](33-fsm-vm-heap-storage-boundary.md)：FSM 的可用空间近似值和 VM 的 all-visible/all-frozen bit 分别服务什么路径，为什么它们是可重建的辅助状态而不是 heap tuple 真相？
8. [Heap rewrite、CLUSTER 与持久化替换边界](34-heap-rewrite-persistence-boundary.md)：表重写为什么通常创建新 relfilenode 再切换 catalog 指向，如何处理 WAL、toast、index、visibility 和失败后的旧文件清理？

第 6 项 `B-tree search / insert / split` 建议先拆成 7 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [B-tree page、high key 与 sibling link](35-btree-page-highkey-sibling.md)：PostgreSQL B-tree 为什么需要 high key、rightlink 和 page opaque，读者如何在并发 split 中仍能找到正确叶子页？
2. [B-tree search、scankey 与并发页面移动](36-btree-search-concurrent-split.md)：`_bt_search()` 如何从 root 走到 leaf，为什么搜索过程中可能要沿 rightlink 追赶 split 后的新页面？
3. [B-tree insert、unique check 与 speculative insertion](37-btree-insert-unique-speculative.md)：唯一索引插入如何在并发事务、未提交 tuple 和 speculative insertion token 之间判断冲突或等待？
4. [B-tree page split、parent insertion 与 incomplete split](38-btree-page-split-parent-insertion.md)：叶子页满时如何分裂、写 WAL、插入父页，为什么 incomplete split 标志能让后续 backend 帮忙完成结构修复？
5. [B-tree deletion、deduplication 与 bottom-up cleanup](39-btree-delete-dedup-bottomup.md)：B-tree 如何清理 dead index tuple、合并重复 key，并在版本 churn 下避免索引无限膨胀？
6. [B-tree WAL redo 与结构修改恢复](40-btree-wal-redo-structural-change.md)：split、unlink、dedup 等结构修改需要怎样的 WAL 信息，redo 如何恢复 sibling link、parent downlink 和 page flags？
7. [B-tree vacuum、cleanup lock 与 index bloat 诊断](41-btree-vacuum-cleanup-bloat.md)：VACUUM 如何扫描 B-tree、删除 dead TID、回收空页，哪些现象说明 index bloat 来自版本 churn、长事务或访问模式？

第 7 项 `Index AM API 与 index vacuum` 建议先拆成 5 个独立主题，后续生成课程时按“一个主问题一课”处理：

1. [Table AM 与 Index AM 的职责边界](42-tableam-indexam-contract.md)：heap、planner、executor 和 index AM 之间如何通过 TID、snapshot、scan key 和 callback 传递语义，为什么 index AM 不直接决定 tuple 可见性？
2. [ambuild、aminsert 与索引创建/维护路径](43-indexam-build-insert-path.md)：CREATE INDEX 和普通 DML 维护索引分别走哪些入口，bulk build、WAL、排序和并发插入的成本边界在哪里？
3. [Index scan、bitmap scan 与 recheck 语义](44-index-scan-bitmap-recheck.md)：一个 index tuple 命中后为什么仍可能需要 heap recheck，bitmap heap scan 如何在 I/O 局部性、lossy page 和可见性判断之间折中？
4. [Index vacuum callback 与 dead TID 清理](45-index-vacuum-dead-tid-callback.md)：VACUUM 如何把 heap 判断出的 dead TID 交给各 index AM，`ambulkdelete()` / `amvacuumcleanup()` 如何更新统计并释放可回收空间？
5. [非 B-tree AM 的持久化差异](46-non-btree-am-persistence-boundary.md)：GiST、GIN、BRIN、hash 等 AM 在 page layout、pending list、summarization、WAL 和 vacuum 上各有什么不同，哪些诊断结论不能从 B-tree 直接外推？
