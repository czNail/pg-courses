# PostgreSQL fsync request queue 与 pending operations

## 课程定位
本节主题：backend 写 relation 文件以后，PostgreSQL 如何把“这个文件段需要 fsync”的责任交给 checkpointer，并在 checkpoint 的同步阶段兑现这个责任。
本节只讲一条主线：`md.c` 写文件，`checkpointer.c` 投递和吸收 fsync request，`sync.c` 维护 pending operations，checkpoint 最后完成 data file fsync 与 delayed unlink。
本节主流程：`mdwritev()` / `mdextend()` 登记 dirty segment -> `RegisterSyncRequest()` 投递 request -> checkpointer `AbsorbSyncRequests()` 记入 `pendingOps` / `pendingUnlinks` -> checkpoint sync phase 执行 fsync 或合法 cancel -> post-checkpoint cleanup delayed unlink。

本节唯一主问题：
data page 已经被 backend、bgwriter 或 checkpointer 写进内核 page cache 后，PostgreSQL 怎样保证 checkpoint 对外承诺完成之前，对应 relation segment 已经被 fsync 或被合法取消？

本节围绕的核心矛盾：
前台 backend 不能在每次写 relation 文件后都同步等待 fsync，否则延迟不可接受；但 checkpoint 一旦推进 redo 起点，就不能丢失任何本应持久化的数据文件变化。fsync request queue 的作用是在延迟、合并和集中处理 fsync 的同时，不丢失 crash-safety 责任。

读完本节，你应该能回答：
- 当前基线里真实入口是不是 `RegisterSyncRequest()`。
- `RegisterSyncRequestType` 在当前基线里是否存在。
- `SyncRequestType` 枚举的四类请求分别是什么。
- backend 写 relation segment 后为什么首选让 checkpointer fsync。
- checkpointer 共享 request queue 的结构是什么。
- `pendingOps` hash table 与 `pendingUnlinks` 链表分别保存什么。
- `RememberSyncRequest()` 怎样处理 sync、unlink、forget、filter。
- `ProcessSyncRequests()` 如何划定本轮 checkpoint 要 fsync 的 entry。
- `mdwritev()`、`mdextend()`、`mdzeroextend()`、`mdtruncate()` 在哪里登记 dirty segment。
- `mdunlink()` 为什么要延迟删除普通 relation main fork 的第一个 segment。
- `SYNC_FORGET_REQUEST` 与 `SYNC_FILTER_REQUEST` 解决什么删除边界。
- `RelationTruncate()` 为什么设置 `DELAY_CHKPT_START` 和 `DELAY_CHKPT_COMPLETE`。
- checkpoint 的 write phase、sync phase、checkpoint record、post-checkpoint unlink 的顺序是什么。
- fsync failure 为什么不能被静默忽略。
- 如果漏登记 fsync、过早 unlink 或 truncate 后不 fsync，会出现什么崩溃持久化漏洞。

## 源码基线
源码仓库：`/home/nail/postgres-lab`
基线：`master` = `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`
本节重点阅读：
- `src/backend/storage/sync/sync.c`
- `src/include/storage/sync.h`
- `src/backend/storage/smgr/md.c`
- `src/backend/postmaster/checkpointer.c`
- `src/backend/catalog/storage.c`
辅助核对：
- `src/backend/access/transam/xlog.c`
- `src/backend/storage/buffer/bufmgr.c`
- `src/backend/storage/smgr/smgr.c`
- `src/backend/storage/smgr/bulk_write.c`
- `src/include/storage/md.h`
- `src/include/storage/smgr.h`
- `src/include/storage/fd.h`

---

## 1. 先给结论
当前基线里没有 `RegisterSyncRequestType()` 这个函数。
真实入口是 `RegisterSyncRequest(const FileTag *ftag, SyncRequestType type, bool retryOnError)`。
`SyncRequestType` 是枚举，不是函数。
它在 `src/include/storage/sync.h` 中定义。
本基线的 `SyncRequestType` 有四个值：
- `SYNC_REQUEST`
- `SYNC_UNLINK_REQUEST`
- `SYNC_FORGET_REQUEST`
- `SYNC_FILTER_REQUEST`
`SYNC_REQUEST` 表示某个 file tag 需要在 checkpoint sync phase 被 fsync。
`SYNC_UNLINK_REQUEST` 表示某个文件可以在下一个 checkpoint 完成后 unlink。
`SYNC_FORGET_REQUEST` 表示取消某个具体 file tag 的 pending fsync。
`SYNC_FILTER_REQUEST` 表示按 handler 的 match 函数取消一批 pending fsync 和 pending unlink。
主线可以压成一条链：
1. `md.c` 写 relation segment。
2. `register_dirty_segment()` 构造 `FileTag`。
3. 它调用 `RegisterSyncRequest(&tag, SYNC_REQUEST, false)`。
4. 普通 backend 没有本地 `pendingOps`，所以请求进入 checkpointer 共享队列。
5. checkpointer 调用 `AbsorbSyncRequests()`。
6. `AbsorbSyncRequests()` 把共享队列请求交给 `RememberSyncRequest()`。
7. `RememberSyncRequest()` 写入 checkpointer 本地 `pendingOps`，或写入 `pendingUnlinks`。
8. checkpoint 的 sync phase 调用 `ProcessSyncRequests()`。
9. `ProcessSyncRequests()` 遍历 `pendingOps` 并调用 handler 的 sync 回调。
10. relation 文件使用 `SYNC_HANDLER_MD`，最终进入 `mdsyncfiletag()` 和 `FileSync()`。
这套机制解决的不是“buffer 是否 dirty”。
它解决的是：某个进程已经把脏页写进内核 page cache 后，谁负责在 checkpoint 完成前把对应 data file fsync 到稳定存储。

## 2. 写和 fsync 是两件事
`FlushBuffer()` 写 data page 之前会先按 page LSN 调用 `XLogFlush()`。
这是 WAL-before-data 的边界。
随后它调用 `smgrwrite(..., false)`。
`false` 表示不跳过 fsync 登记。
md handler 最终在 `mdwritev()` 里执行文件写。
文件写成功后，`mdwritev()` 调用 `register_dirty_segment()`。
这一步不是立即把数据刷到稳定存储。
它只是把“这个 relation segment 需要 fsync”登记出来。
普通 backend 首选把这个 fsync 责任交给 checkpointer。
checkpoint 完成之前，checkpointer 会集中执行这些 fsync。
这样做有三个原因。
第一，避免每个 backend 在用户查询路径上频繁 fsync。
第二，允许同一个 segment 的多次写入合并成一次 fsync。
第三，让 checkpoint 的 write phase 和 sync phase 有统一顺序。
但这不是无条件异步。
如果 checkpointer queue 满了，并且 compact 后仍无法投递，backend 会 fallback 自己 fsync。
所以 correctness 规则是：写完 relation 文件后，要么成功投递 fsync request，要么当前 backend 自己完成 fsync。

## 3. `FileTag` 是请求身份
`src/include/storage/sync.h` 定义了 `FileTag`。
它当前包含：
- `handler`
- `forknum`
- `rlocator`
- `segno`
`handler` 决定由哪个 `SyncOps` 函数组处理。
relation 文件使用 `SYNC_HANDLER_MD`。
`forknum` 是 relation fork。
`rlocator` 是 tablespace、database、relfilenumber 组合。
`segno` 是 relation segment 号。
这说明 fsync request 的粒度不是 relation。
它是 relation fork 的某个 segment。
这很重要，因为大 relation 会拆成多个 segment 文件。
fsync 的对象是文件，不是逻辑 relation。
queue 里也不能保存 fd。
fd 是进程本地资源。
queue 必须保存跨进程可解释的文件身份。
这就是 `FileTag` 的作用。

## 4. 两层 pending：共享队列与本地表
本节最容易混淆的是 pending 到底在哪里。
第一层在 `checkpointer.c`。
这是共享内存里的 request queue。
结构是 `CheckpointerShmemStruct`。
相关字段包括：
- `num_requests`
- `max_requests`
- `head`
- `tail`
- `requests[]`
每个 `requests[]` slot 是 `CheckpointerRequest`。
`CheckpointerRequest` 包含 `SyncRequestType type` 和 `FileTag ftag`。
这是普通 backend 给 checkpointer 投递请求的 ring buffer。
第二层在 `sync.c`。
这是 checkpointer 本地的 `pendingOps`。
`pendingOps` 是 `HTAB *`。
key 是 `FileTag`。
entry 是 `PendingFsyncEntry`。
`PendingFsyncEntry` 包含 `tag`、`cycle_ctr`、`canceled`。
这是 checkpoint sync phase 真正遍历的 fsync pending table。
第三个结构也在 `sync.c`。
它叫 `pendingUnlinks`。
它是 `List *`，元素是 `PendingUnlinkEntry`。
它保存 checkpoint 完成后才能 unlink 的文件。
`pendingUnlinks` 用链表，不用 hash table，因为源码假设 unlink request 不常重复。

## 5. `storage.c` 也有 pending，但不是同一个
`src/backend/catalog/storage.c` 中有 `pendingSyncHash`。
它不是 checkpointer fsync queue。
它也不是 `sync.c` 的 `pendingOps`。
它处理 transaction end 时“永久 relation 创建了，但某些页面跳过 WAL”的同步问题。
`RelationCreateStorage()` 在 permanent relation 且 `!XLogIsNeeded()` 时调用 `AddPendingSync()`。
`AddPendingSync()` 把 `RelFileLocator` 放入 `pendingSyncHash`。
事务提交时 `smgrDoPendingSyncs(true, ...)` 处理它。
这个函数会选择：
- 对小 relation 生成 `log_newpage_range()` WAL。
- 对大 relation 或发生 truncate 的 relation 调用 `smgrdosyncall()`。
`PendingRelSync` 里的 `is_truncated` 很关键。
如果文件曾经变短，只靠当前内容生成 WAL 可能不能清除 crash 后复活的尾部 blocks。
所以 truncated relation 必须 sync。
这一套是 commit-time sync。
本节主线是 checkpoint-time sync。
不要把 `pendingSyncHash` 和 `pendingOps` 混为一谈。

## 6. `InitSync()`：普通 backend 没有 `pendingOps`
`sync.c` 的 `InitSync()` 只在两类进程中创建 `pendingOps`。
条件是：
```c
if (!IsUnderPostmaster || AmCheckpointerProcess())
```
也就是 standalone backend 或 checkpointer auxiliary process。
普通 backend 在 postmaster 管理下运行。
它不会创建本地 `pendingOps`。
所以普通 backend 调用 `RegisterSyncRequest()` 时，不能本地记账。
它必须调用 `ForwardSyncRequest()` 投递到 checkpointer。
standalone 模式没有 checkpointer。
所以 standalone backend 需要本地 `pendingOps`。
`pendingOps` 的 memory context 是 `pendingOpsCxt`。
源码允许它在 critical section 内分配内存。
原因是 `AbsorbSyncRequests()` 可能已经从共享队列清掉请求。
如果这时无法写入本地 pending table，系统就可能忘记必须 fsync 的文件。
源码选择在这种场景下 PANIC。
这是持久化 correctness 的要求，不是普通内存分配策略。

## 7. `RegisterSyncRequest()` 的分支
`RegisterSyncRequest()` 先看 `pendingOps` 是否存在。
如果存在，说明当前进程可以本地记录请求。
它直接调用 `RememberSyncRequest(ftag, type)`。
如果不存在，说明是普通 backend。
它循环调用 `ForwardSyncRequest(ftag, type)`。
如果投递成功，返回 true。
如果投递失败且 `retryOnError=false`，返回 false。
如果投递失败且 `retryOnError=true`，等待 10ms 后重试。
这个 `retryOnError` 决定调用方能不能自己解决。
普通 dirty fsync request 使用 `false`。
因为 queue 满了可以由当前 backend 自己 fsync。
unlink、forget、filter 通常使用 `true`。
因为这些请求是删除协议的一部分，不能简单本地替代。

## 8. `ForwardSyncRequest()`：共享 ring queue
`ForwardSyncRequest()` 在 `checkpointer.c`。
它是普通 backend 投递 sync request 的函数。
它先检查是否在 postmaster 下。
如果不是，返回 false。
它禁止 checkpointer 自己调用。
然后拿 `CheckpointerCommLock`。
如果 checkpointer 没运行，返回 false。
如果队列满，先尝试 `CompactCheckpointerRequestQueue()`。
如果 compact 后仍不能腾出空间，返回 false。
插入请求时，它写入 `requests[tail]`。
然后推进 `tail`。
再递增 `num_requests`。
如果 queue 超过半满，它在释放 lock 后唤醒 checkpointer。
共享队列大小由 `CheckpointerShmemInit()` 设置。
`max_requests = Min(NBuffers, MAX_CHECKPOINT_REQUESTS)`。
`MAX_CHECKPOINT_REQUESTS` 是 `10000000`。
队列大小与 `NBuffers` 相关，是因为每个 shared buffer 写出都可能生成 fsync request。

## 9. 为什么队列允许重复
`ForwardSyncRequest()` 通常不在 backend 侧去重。
源码注释说，这是为了避免长时间持有 `CheckpointerCommLock`。
backend 写路径很热。
如果每次写出页面都扫描队列或查 hash，锁争用会很重。
重复请求本来就常见。
同一个 relation segment 可能被多个 dirty buffer 写出。
最终只需要一次 fsync。
所以共享队列允许重复。
去重主要交给 checkpointer 本地的 `pendingOps` hash table。
当 queue 满时才调用 `CompactCheckpointerRequestQueue()`。
compact 会用临时 hash table 找重复的 `CheckpointerRequest`。
它标记前面重复项可跳过，保留后出现的请求。
为什么不随便从后往前删？
因为两个相同 sync request 之间可能夹着 `SYNC_FORGET_REQUEST` 或 `SYNC_FILTER_REQUEST`。
compact 不能改变这些请求的相对语义。
这就是 fsync queue compact 的 correctness 边界。

## 10. `AbsorbSyncRequests()`：从共享队列搬到本地
checkpointer 不直接在共享 queue 上 fsync。
它先调用 `AbsorbSyncRequests()`。
这个函数只在 checkpointer 进程中做事。
如果 `!AmCheckpointerProcess()`，直接 return。
它的步骤是：
1. 拿 `CheckpointerCommLock`。
2. 从 ring buffer 中最多复制 `CKPT_REQ_BATCH_SIZE` 个请求到本地数组。
3. 推进 shared `head`。
4. 减少 shared `num_requests`。
5. 释放 lock。
6. 对复制出来的每个请求调用 `RememberSyncRequest()`。
7. 如果队列还有请求，继续循环。
`CKPT_REQ_BATCH_SIZE` 是 `10000`。
这样避免一次持锁处理整个 queue。
函数在清掉共享队列后进入 critical section。
原因是清掉 queue 后，如果无法写入本地 pending table，就会丢失 fsync request。
丢失 fsync request 会让 checkpoint 错误成功。
所以源码要求这种失败必须 PANIC。

## 11. `RememberSyncRequest()`：四类请求落地
`RememberSyncRequest()` 是 checkpointer side 的落表函数。
它要求 `pendingOps` 已存在。
收到 `SYNC_REQUEST` 时，它在 `pendingOps` 中 `HASH_ENTER`。
如果 entry 是新的，或之前已 canceled，就设置 `cycle_ctr = sync_cycle_ctr` 并清掉 canceled。
如果 entry 已经存在且未 canceled，它不更新 `cycle_ctr`。
这是有意的。
`cycle_ctr` 必须代表最早的 pending fsync 请求。
如果重复请求刷新 counter，旧请求可能被误认为新请求，从而跳过本轮 checkpoint。
收到 `SYNC_FORGET_REQUEST` 时，它在 `pendingOps` 中查找完全匹配的 `FileTag`。
如果找到，就设置 `entry->canceled = true`。
收到 `SYNC_FILTER_REQUEST` 时，它遍历 `pendingOps`。
handler 相同且 `sync_filetagmatches()` 返回 true 的 entry 会 canceled。
它还遍历 `pendingUnlinks`。
匹配的 unlink entry 也会 canceled。
收到 `SYNC_UNLINK_REQUEST` 时，它分配 `PendingUnlinkEntry`。
它设置 `tag`、`cycle_ctr = checkpoint_cycle_ctr`、`canceled = false`。
然后 append 到 `pendingUnlinks`。
所以 sync requests 进入 `pendingOps`。
delayed unlink requests 进入 `pendingUnlinks`。
forget/filter 是取消语义。

## 12. `ProcessSyncRequests()`：执行 pending fsync
`ProcessSyncRequests()` 是 checkpoint sync phase 的核心。
它只应该在拥有 `pendingOps` 的进程中运行。
如果没有 `pendingOps`，它 ERROR。
函数开头先调用 `AbsorbSyncRequests()`。
这是为了兜住最紧的 race。
某个 backend 可能在 `BufferSync()` 扫到 buffer 之前，把该 buffer 写出并清掉 dirty bit。
backend 在清 dirty bit 前会登记 fsync request。
因此 checkpointer 在 `BufferSync()` 完成后、真正 fsync 前必须 absorb。
接着 `ProcessSyncRequests()` 处理 `sync_cycle_ctr`。
它推进 counter，用来区分：
- sync phase 开始前已有的旧 entry。
- sync phase 中后来吸收的新 entry。
新 entry 不属于本轮 checkpoint。
如果本轮一直处理新 entry，在高写入负载下 checkpoint 可能永远结束不了。
扫描 hash table 时，如果 `entry->cycle_ctr == sync_cycle_ctr`，说明它是新 entry。
函数跳过它。
旧 entry 会调用 handler 的 sync callback。
成功后从 `pendingOps` 删除。

## 13. `sync_cycle_ctr` 的失败恢复
`ProcessSyncRequests()` 有一个静态变量 `sync_in_progress`。
进入 fsync loop 前，它设置为 true。
函数成功结束后，它设置为 false。
如果上一次 fsync loop 中途 ERROR，下一次进来会看到 `sync_in_progress` 仍为 true。
这时它先遍历所有 `pendingOps` entry。
把它们的 `cycle_ctr` 强制设置为当前 `sync_cycle_ctr`。
然后再推进 counter。
这样残留 entry 都会被下一轮视为旧 entry。
源码这样做是为了避免多次 checkpoint failure 后 counter wraparound 导致旧 entry 被误判为新 entry。
这条路径正常系统很少走。
但它说明 pending ops 表必须能承受 checkpoint sync 中途失败。

## 14. `syncsw[]` 与 md handler
`sync.c` 不知道 relation path 怎么拼。
它通过 `syncsw[]` 函数表调用 handler。
`SYNC_HANDLER_MD` 的回调是：
- `mdsyncfiletag`
- `mdunlinkfiletag`
- `mdfiletagmatches`
`ProcessSyncRequests()` 对 fsync entry 调用：
```c
syncsw[entry->tag.handler].sync_syncfiletag(&entry->tag, path)
```
对 relation 文件，这就是 `mdsyncfiletag()`。
`mdsyncfiletag()` 通过 `FileTag` 打开或复用目标 segment 文件。
然后调用 `FileSync(file, WAIT_EVENT_DATA_FILE_SYNC)`。
如果是临时打开的 file，它 fsync 后关闭。
它还把 path 写到输出 buffer，用于错误信息。
这说明 queue 中保存的是文件身份，而不是 fd。

## 15. `md.c` 的登记点
relation 文件的底层写入在 `md.c`。
本节关注这些函数：
- `mdextend()`
- `mdzeroextend()`
- `mdwritev()`
- `mdtruncate()`
- `mdregistersync()`
`mdextend()` 写一个新 block。
写成功后，如果 `skipFsync=false` 且不是 temp relation，就调用 `register_dirty_segment()`。
`mdzeroextend()` 批量扩展零页。
它对涉及的 segment 同样登记 dirty segment。
`mdwritev()` 写已有 blocks。
写成功后也登记 dirty segment。
`mdtruncate()` 改变文件长度。
它对被 truncate 的 segment 调用 `FileTruncate()` 后，也登记 dirty segment。
`mdregistersync()` 用于“之前写时跳过 fsync 登记，现在补登记整个 relation fork”。
它遍历 active 和 inactive segments，并对每个 segment 调用 `register_dirty_segment()`。
这在 bulk write 路径中很重要。

## 16. `register_dirty_segment()` 的 fallback
`register_dirty_segment()` 是 `md.c` 中的静态函数。
它用 `INIT_MD_FILETAG()` 构造 `FileTag`。
然后断言 temp relation 不应到这里。
temp relation 不需要 fsync，也不需要 delayed unlink 保护。
它调用：
```c
RegisterSyncRequest(&tag, SYNC_REQUEST, false)
```
这里 `retryOnError=false`。
如果返回 true，请求已经本地记录或成功投递给 checkpointer。
如果返回 false，说明无法投递。
这时 `register_dirty_segment()` 调用 `FileSync(seg->mdfd_vfd, WAIT_EVENT_DATA_FILE_SYNC)`。
也就是 backend 自己 fsync 当前 segment。
源码还记录 IO stats。
所以 queue 满不是 correctness failure。
queue 满会让 backend 做昂贵的本地 fsync。
这是性能退化，不是允许丢请求。

## 17. `mdunlink()` 的 delayed unlink
`mdunlink()` 的注释是本节关键。
普通 relation drop 时，main fork 的第一个 segment 不会直接 unlink。
它会先 truncate 到 0。
然后登记 `SYNC_UNLINK_REQUEST`。
真正 unlink 发生在 checkpoint 完成后的 `SyncPostCheckpoint()`。
为什么要这样做？
因为 relfilenumber 可能复用。
源码注释给出危险场景：
1. 删除一个 relation 并提交。
2. 立即 unlink 旧文件。
3. 创建新 relation，碰巧得到相同 relfilenumber。
4. crash 发生在下一个 checkpoint 前。
5. recovery 重放旧 relation 删除和新 relation 创建。
6. 如果新 relation 内容不是逐页 WAL 记录，而是依赖 fsync 持久化，内容可能丢失。
保留 0 长度占位文件能阻止 relfilenumber 分配复用该文件名。
checkpoint 完成后，旧删除已经越过恢复边界。
这时再 unlink 才安全。
额外 segment 可以更早 unlink。
其他 fork 也不需要用 main fork 首段阻止 relfilenumber 复用。
temp relation 也不用这套 dance。
redo 场景也可以立即删除，因为 redo 中不会创建冲突的新 relation。

## 18. `pendingUnlinks` 与 `checkpoint_cycle_ctr`
`register_unlink_segment()` 调用：
```c
RegisterSyncRequest(&tag, SYNC_UNLINK_REQUEST, true)
```
这里 `retryOnError=true`。
delayed unlink request 不能因为 queue 满而丢。
`RememberSyncRequest()` 收到 `SYNC_UNLINK_REQUEST` 后，append 一个 `PendingUnlinkEntry` 到 `pendingUnlinks`。
entry 的 `cycle_ctr` 等于当前 `checkpoint_cycle_ctr`。
`SyncPreCheckpoint()` 会先调用 `AbsorbSyncRequests()`。
这样 checkpoint 开始前已经转发的 unlink request 会被吸收到本地链表。
随后 `SyncPreCheckpoint()` 递增 `checkpoint_cycle_ctr`。
checkpoint 过程中才来的 unlink request 会记录新 counter。
`SyncPostCheckpoint()` 只处理早于当前 checkpoint 的 unlink entry。
如果遇到 `entry->cycle_ctr == checkpoint_cycle_ctr`，说明到了当前 checkpoint 期间的新 entry。
因为新 entry append 在链表尾部，遍历可以停止。
这个 counter 保证文件不会在覆盖它的 checkpoint 完成前被删除。

## 19. `SYNC_FORGET_REQUEST`
forget request 取消一个具体 `FileTag` 的 pending fsync。
md 层入口是 `register_forget_request()`。
它调用：
```c
RegisterSyncRequest(&tag, SYNC_FORGET_REQUEST, true)
```
删除额外 segment 前，`mdunlinkfork()` 会先发 forget。
普通立即删除路径也会先 forget，再 unlink。
为什么不能等 fsync 时看到 ENOENT 就忽略？
因为 ENOENT 可能是合法删除，也可能是真错误。
正确协议是：删除路径先发送 forget。
checkpointer absorb 后把对应 `pendingOps` entry 标记为 canceled。
`ProcessSyncRequests()` 如果 fsync 失败并看到文件可能已删除，会先 absorb。
如果合法 forget 到达，entry 变 canceled，retry loop 结束。
如果没有合法 cancel，第二次失败会报错。
所以 forget request 是“这个 fsync 不再需要”的证明。

## 20. `SYNC_FILTER_REQUEST`
filter request 取消一批请求。
md handler 的 match 函数是 `mdfiletagmatches()`。
当前实现按 database OID 匹配。
`ForgetDatabaseSyncRequests(Oid dbid)` 构造一个特殊 `FileTag`。
它只关心 `rlocator.dbOid`。
然后调用：
```c
RegisterSyncRequest(&tag, SYNC_FILTER_REQUEST, true)
```
`RememberSyncRequest()` 收到 filter request 后遍历 `pendingOps`。
匹配的 fsync entry 被标 canceled。
它还遍历 `pendingUnlinks`。
匹配的 unlink entry 也被标 canceled。
这主要服务于 drop database。
drop database 删除整个数据库目录前，必须让 checkpointer 忘记该 database 相关的 pending fsync 和 pending unlink。
否则 checkpointer 之后可能尝试 fsync 或 unlink 已经删除的路径。

## 21. `mdtruncate()` 为什么登记 fsync
truncate 改变文件长度。
文件长度变化也需要持久化。
`mdtruncate()` 从最后一个打开 segment 往前处理。
如果某个 segment 完全超出新大小，它调用 `FileTruncate(..., 0, ...)`。
然后对非 temp relation 调用 `register_dirty_segment()`。
如果某个 segment 是最后保留 segment，但需要缩短长度，它调用 `FileTruncate()` 到目标字节数。
然后也登记 dirty segment。
原因是 crash 后文件长度可能恢复到旧值。
如果 checkpoint 已经成功，而 truncate 没 fsync，恢复可能从该 checkpoint 开始。
此时 truncate WAL record 可能早于 redo point，不会重放。
旧 trailing blocks 就可能复活。
这会破坏 relation size、FSM、visibility map 或页面可见性假设。
所以 truncate 后必须让 checkpoint sync phase fsync 对应 segment。

## 22. `RelationTruncate()` 的两个 delay flag
`RelationTruncate()` 在 `storage.c`。
它处理 relation-level truncate。
源码注释说明它和 checkpoint 有两类 race。
第一类 race：truncate 可能丢弃 checkpoint 原本要写出的 dirty buffers。
如果 checkpoint record 成功，但磁盘文件还没 truncate，recovery 从该 checkpoint 开始时可能看到旧尾部 blocks。
所以 truncate critical work 完成前，要延迟 checkpoint complete。
这需要 `DELAY_CHKPT_COMPLETE`。
第二类 race：`smgrtruncate()` 会调用 `RegisterSyncRequest()`。
如果 truncate WAL record 早于并发 checkpoint 的 redo pointer，但 sync request 晚于该 checkpoint 的 `ProcessSyncRequests()`，checkpoint 就会漏掉文件长度 fsync。
所以还需要 `DELAY_CHKPT_START`。
`RelationTruncate()` 设置：
```c
MyProc->delayChkptFlags |= DELAY_CHKPT_START | DELAY_CHKPT_COMPLETE;
```
如果 relation needs WAL，它先写 `XLOG_SMGR_TRUNCATE`。
然后 `XLogFlush(lsn)`。
之后在 critical section 中调用 `smgrtruncate()`。
完成后清掉两个 delay flags。
这说明 truncate 同时跨越 WAL、buffer、文件长度和 checkpoint 边界。

## 23. checkpoint 同步顺序
checkpoint 主入口是 `CreateCheckPoint()`。
完整细节在 `xlog.c`。
本节只关心同步顺序：
1. `CreateCheckPoint()` 调用 `SyncPreCheckpoint()`。
2. checkpoint 确定 redo point。
3. 它等待 `DELAY_CHKPT_START` 的事务离开危险区。
4. 它调用 `CheckPointGuts(checkPoint.redo, flags)`。
5. `CheckPointGuts()` 写出 SLRU 和 shared buffers。
6. `CheckPointGuts()` 调用 `ProcessSyncRequests()`。
7. 它等待 `DELAY_CHKPT_COMPLETE` 的事务离开危险区。
8. 它写 checkpoint WAL record。
9. 它 `XLogFlush(recptr)`。
10. 它更新 control file。
11. 它调用 `SyncPostCheckpoint()`。
`CheckPointGuts()` 内部顺序也很关键：
- `CheckPointCLOG()`
- `CheckPointCommitTs()`
- `CheckPointSUBTRANS()`
- `CheckPointMultiXact()`
- `CheckPointPredicate()`
- `CheckPointBuffers(flags)`
- `ProcessSyncRequests()`
所以 checkpoint 先 write dirty data，再 fsync queued files。
checkpoint record flush 在 `ProcessSyncRequests()` 之后。
delayed unlink 在 checkpoint record flush 和 control file 更新之后。
不能把 `SyncPostCheckpoint()` 提前。
否则可能过早删除占位文件。
不能把 `ProcessSyncRequests()` 放在 `CheckPointBuffers()` 前。
否则 write phase 产生的 fsync request 可能漏掉。

## 24. `BufferSync()` 与 backend write race
`BufferSync()` 先扫描 buffer pool。
它给 checkpoint 开始时需要写的 dirty buffers 标记 `BM_CHECKPOINT_NEEDED`。
checkpoint 过程中后来变 dirty 的 buffers 不属于本轮 checkpoint。
写出一个 buffer 时，路径是：
1. `SyncOneBuffer()`。
2. `FlushUnlockedBuffer()`。
3. `FlushBuffer()`。
4. `XLogFlush(page_lsn)`。
5. `smgrwrite(..., false)`。
6. `mdwritev()`。
7. `register_dirty_segment()`。
8. `TerminateBufferIO()` 清 dirty。
也就是说，dirty bit 被清掉前，fsync request 已经登记或已 fallback 本地 fsync。
这解释了 `ProcessSyncRequests()` 开头的 `AbsorbSyncRequests()`。
如果 backend 在 checkpointer 扫描期间帮忙写出了 checkpoint-needed buffer，checkpointer 在 `BufferSync()` 之后 absorb，就能看到它的 fsync request。

## 25. checkpoint 期间持续 absorb
checkpointer 不只在 `ProcessSyncRequests()` 开头吸收请求。
`CheckpointerMain()` 主循环会 absorb。
`CheckpointWriteDelay()` 控速时会 absorb。
即使不 sleep，它也会每 `WRITES_PER_ABSORB` 次写操作 absorb 一次。
`WRITES_PER_ABSORB` 是 `1000`。
`ProcessSyncRequests()` fsync loop 中每 `FSYNCS_PER_ABSORB` 次 fsync absorb 一次。
`FSYNCS_PER_ABSORB` 是 `10`。
`SyncPostCheckpoint()` 删除 delayed unlink 时每 `UNLINKS_PER_ABSORB` 次 unlink absorb 一次。
`UNLINKS_PER_ABSORB` 是 `10`。
这些 absorb 是为了防止共享 request queue 长时间没人清。
如果 checkpointer 在长 write 或 long fsync 阶段不 absorb，queue 会变满。
queue 满会让 backend fallback 自己 fsync。
这会显著拖慢写路径。

## 26. fsync failure 处理
`ProcessSyncRequests()` 对每个旧 entry 调用 handler sync。
如果 `enableFsync` 为 false，它跳过实际 fsync。
这符合 `fsync=off` 的配置语义，但这会放弃正常 crash 持久性保证。
`enableFsync` 为 true 时，fsync failure 会进入 retry 逻辑。
第一次失败如果 errno 表示文件可能已删除，函数不会立刻忽略。
Unix 上 `FILE_POSSIBLY_DELETED(errno)` 主要是 `ENOENT`。
Windows 上还包括 `EACCES`。
第一次可能删除失败后，函数 DEBUG1 记录并调用 `AbsorbSyncRequests()`。
这样可以吸收删除路径此前发送的 forget/filter cancel。
如果 entry 变成 canceled，retry loop 结束。
如果没有 canceled，第二次失败会通过 `data_sync_elevel(ERROR)` 报错。
其他非“可能删除”错误也会报错。
所以 ENOENT 不是无条件忽略。
只有删除路径提供了合法 cancel，fsync 缺失才被视为正常。

## 27. checkpoint failure 与 PANIC
fsync error 在 checkpoint 中通常会让 checkpoint 失败。
`CheckpointerMain()` 捕获 checkpoint ERROR 后，会更新共享状态。
它递增 `ckpt_failed`。
它把 `ckpt_done` 设置为 `ckpt_started`。
它广播等待 checkpoint 完成的 condition variable。
等待者会看到 checkpoint request failed。
这比推进一个不安全 checkpoint 正确。
还有更严重的情况。
`AbsorbSyncRequests()` 已经从共享 queue 清掉请求后，如果不能写入本地 pending ops，源码要求 PANIC。
原因是系统已经丢失了需要 fsync 的知识。
继续运行可能让 checkpoint 错误成功。
PANIC 触发重启和 crash recovery，比继续提交错误状态安全。

## 28. immediate sync 与 queued sync
不是所有写入都适合只登记 queued sync。
`smgrimmedsync()` 是立即 sync 某个 relation fork。
md handler 对应 `mdimmedsync()`。
`mdimmedsync()` 会 sync active 和 inactive segments。
源码注释说明 `smgrDoPendingSyncs()` 依赖这一点。
如果 relation 曾经更长，后来 truncate 让某些 segment inactive，crash 后这些 inactive segment 也不能带着旧内容复活。
`smgrdosyncall()` 更强。
它先 `FlushRelationsAllBuffers()`。
然后对每个 relation 的每个 fork 调 immediate sync。
它用于 `smgrDoPendingSyncs()` 的 commit-time sync。
bulk write 也会在某些情况下 immediate sync。
`bulk_write.c` 记录开始时的 `RedoRecPtr`。
结束时如果 `GetRedoRecPtr()` 变了，说明期间发生 checkpoint。
这个 checkpoint 可能已经错过 bulk write 早期写出的 pages。
此时不能只 `smgrregistersync()` 给未来 checkpoint。
必须 `smgrimmedsync()`。

## 29. 反例一：漏登记 fsync
假设 `mdwritev()` 写 permanent relation 成功后没有调用 `register_dirty_segment()`。
随后 checkpoint 发生。
`BufferSync()` 看到 buffer 已 clean。
`ProcessSyncRequests()` 中没有该 segment 的 pending entry。
checkpoint record 被写出并 flush。
control file 指向该 checkpoint。
随后系统 crash。
恢复从这个 checkpoint redo point 开始。
如果对应 page 的 WAL record 早于 redo point，就不会 replay。
但 data file 的写入可能只在内核 page cache 中，从未 fsync 到稳定存储。
crash 后磁盘仍是旧 page。
数据库也没有 WAL 可以补。
这就是崩溃持久化漏洞。
它不需要磁盘损坏。
只需要未 fsync 的写入在 crash 后丢失。

## 30. 反例二：过早 unlink main fork 首段
假设 `mdunlink()` 删除普通 relation 时直接 unlink main fork 第一个 segment。
旧 relfilenumber 文件名马上消失。
随后新 relation 复用同一个 relfilenumber。
新 relation 内容通过跳过 WAL 的路径写入，并依赖 commit 前 fsync。
crash 发生在下一个 checkpoint 前。
recovery 从旧 checkpoint 开始。
它会重放旧 relation 删除和新 relation 创建。
如果删除语义影响了新文件，而新 relation 内容没有逐页 WAL，内容可能永久丢失。
PostgreSQL 用 0 长度占位文件阻止这个 relfilenumber 复用。
checkpoint 完成后再 unlink。
这就是 delayed unlink 的 crash recovery contract。

## 31. 反例三：truncate 后不 fsync 文件长度
假设 relation 从 10000 blocks truncate 到 100 blocks。
WAL 记录了 truncate。
buffer manager 丢弃了尾部 buffers。
`FileTruncate()` 也调用了。
但没有登记 dirty segment。
checkpoint 成功。
随后 crash。
文件系统恢复后，文件长度可能仍是旧长度。
如果 truncate WAL record 早于 checkpoint redo point，它不会被 replay。
旧尾部 blocks 重新出现。
这些 blocks 可能包含旧 tuple、旧 FSM 或旧 visibility map 状态。
这会破坏 relation size 和可见性假设。
所以 `mdtruncate()` 必须登记 fsync。
`RelationTruncate()` 还必须用 delay checkpoint flags 避免并发 checkpoint 漏掉该 sync。

## 32. 反例四：随便忽略 ENOENT
假设 `ProcessSyncRequests()` 对 fsync ENOENT 直接忽略。
这似乎解决了 drop relation race。
但它也会掩盖真实错误。
比如 forget request 丢了。
或者文件被错误删除。
如果 fsync 阶段直接忽略 ENOENT，checkpoint 可能成功。
control file 会推进。
recovery 之后可能缺少本应存在的数据文件内容。
本基线的策略更严格。
第一次可能删除错误后，先 absorb。
只有看到合法 cancel，才停止 fsync 该 entry。
否则第二次失败报错。
删除路径必须先通知 checkpointer。
fsync 路径不能自己猜测“文件没了就是正常”。

## 33. request 类型速查
`SYNC_REQUEST` 的典型入口是 `register_dirty_segment()`。
它也可由 `mdregistersync()` 间接产生。
它表示某个 segment 需要 fsync。
`SYNC_UNLINK_REQUEST` 的入口是 `register_unlink_segment()`。
它用于 ordinary relation main fork 首段 delayed unlink。
`SYNC_FORGET_REQUEST` 的入口是 `register_forget_request()`。
它用于删除前取消具体 segment 的 pending fsync。
`SYNC_FILTER_REQUEST` 的入口是 `ForgetDatabaseSyncRequests()`。
它用于 drop database 时取消一个 database 相关的 pending requests。
四种请求共用 `RegisterSyncRequest()`。
但它们的 retry 策略不同。
普通 dirty sync 可以 fallback backend fsync。
unlink、forget、filter 必须可靠投递。

## 34. 从 UPDATE 到 fsync queue
一次普通 UPDATE 修改 heap page。
heap AM 生成 WAL。
page LSN 更新。
buffer 标 dirty。
后来 checkpoint 或 backend eviction 写出 buffer。
`FlushBuffer()` 先 `XLogFlush(page_lsn)`。
然后 `smgrwrite(..., false)`。
md handler 进入 `mdwritev()`。
`mdwritev()` 调用 `FileWriteV()`。
写成功后 `register_dirty_segment()`。
`register_dirty_segment()` 调用 `RegisterSyncRequest(SYNC_REQUEST)`。
普通 backend 进入 `ForwardSyncRequest()`。
请求写入 `CheckpointerShmem->requests[]`。
checkpointer 后续 `AbsorbSyncRequests()`。
`RememberSyncRequest()` 把 `FileTag` 放进 `pendingOps`。
checkpoint sync phase 调用 `ProcessSyncRequests()`。
md handler 最终 `FileSync()`。
checkpoint record 最后才写出和 flush。

## 35. 从 DROP 到 delayed unlink
`RelationDropStorage()` 把 relation 加入 `pendingDeletes`。
事务提交时 `smgrDoPendingDeletes(true)` 处理删除。
它调用 smgr unlink。
md handler 进入 `mdunlink()`。
普通 relation main fork 首段：
1. `mdunlinkfork()` truncate 文件到 0。
2. 调用 `register_unlink_segment()`。
3. 发送 `SYNC_UNLINK_REQUEST`。
4. checkpointer absorb。
5. `RememberSyncRequest()` append `pendingUnlinks`。
6. checkpoint 完成后 `SyncPostCheckpoint()` 调用 `mdunlinkfiletag()`。
额外 segment：
1. truncate。
2. `register_forget_request()`。
3. 立即 unlink。
这两个路径的差别来自 relfilenumber 复用保护。

## 36. 从 TRUNCATE 到 fsync
`RelationTruncate()` 准备 main fork、FSM fork、visibility map fork。
它调用 `RelationPreTruncate(rel)`。
它设置 checkpoint delay flags。
如果 relation needs WAL，它写 `XLOG_SMGR_TRUNCATE`。
它立即 `XLogFlush(lsn)`。
然后进入 critical section。
它调用 `smgrtruncate()`。
smgr 层先移除不该存在的 buffers。
md handler 进入 `mdtruncate()`。
`mdtruncate()` 调 `FileTruncate()`。
`mdtruncate()` 调 `register_dirty_segment()`。
请求进入 `RegisterSyncRequest(SYNC_REQUEST)`。
checkpoint sync phase fsync 对应 segment。
完成后 `RelationTruncate()` 清掉 delay flags。
这条路径的三个关口是：
- truncate WAL 先持久化。
- 文件 truncate 必须执行。
- 文件长度变化必须在 checkpoint 完成前 fsync。

## 37. pending ops 不是 WAL
fsync request queue 不写 WAL。
`pendingOps` 也不写 WAL。
它们是内存结构。
这不矛盾。
如果系统在 checkpoint 完成前 crash，新的 checkpoint record 尚未成为恢复起点。
恢复会从更早 checkpoint 开始，并 replay 后续 WAL。
所以内存里的 pending fsync request 丢失是可以接受的。
前提是对应 checkpoint 没有成功完成。
如果 checkpoint 已经成功，所有对应 data file fsync 必须已经完成。
这就是 pending ops 的生命周期。
它只需要在运行时可靠驱动 checkpoint。
它不需要在 crash 后恢复。
但运行时不能丢请求。
所以 `AbsorbSyncRequests()` 清共享队列后不能失败后继续运行。

## 38. bgwriter 与 WAL writer 的边界
background writer 也会写 dirty buffers。
它的目标是提前制造 reusable clean buffers。
它不是 checkpoint correctness 的最终负责人。
但它写 buffer 也走 `FlushBuffer()`。
所以它同样触发 `smgrwrite(..., false)` 和 `register_dirty_segment()`。
backend、bgwriter、checkpointer 写出的 relation pages 都汇聚到 md 层登记 dirty segment。
WAL writer 是另一套机制。
它负责 WAL buffers 到 WAL files 的 write/flush。
本节的 queue 是 data file fsync queue。
data page write 之前需要 WAL flush。
checkpoint record 之前需要 data file fsync。
WAL writer 优化前者。
checkpointer sync manager 保障后者。

## 39. 生命周期 / ownership / cleanup
fsync request 的生命周期从 `md.c` 写文件后开始。`register_dirty_segment()` 构造 `FileTag`，普通 backend 通过 `RegisterSyncRequest(SYNC_REQUEST, retryOnError=false)` 交给 checkpointer；如果 queue 满且 compact 后仍无法投递，backend 必须 fallback 自己 `FileSync()`。

共享 queue 只是投递通道，owner 是 checkpointer shared memory。真正的 pending state 在 checkpointer 本地 `pendingOps` hash table，key 是 `FileTag`，entry 记录 cycle 和 cancel 状态。checkpoint sync phase 成功处理后，entry 被移除。

unlink 是另一条生命周期。`SYNC_UNLINK_REQUEST` 进入 `pendingUnlinks`，checkpoint 完成后 `SyncPostCheckpoint()` 调 handler unlink；`SYNC_FORGET_REQUEST` 和 `SYNC_FILTER_REQUEST` 负责取消合法删除对象的 pending fsync。这样删除路径不会让 fsync 阶段自己猜测 ENOENT 是否安全。

cleanup 的关键边界是：pending ops 不写 WAL。checkpoint 未完成就 crash，恢复从旧 checkpoint 开始 replay；checkpoint 一旦完成，所有本轮应 fsync 的 data file 必须已经 fsync 或合法 cancel。

## 40. 成本、资源与观测诊断
成本来自三处：共享 request queue 的争用和 compact，checkpointer 本地 `pendingOps` 的 hash 合并和遍历，以及 sync phase 的真实 fsync latency。变量包括 dirty relation segment 数、backend 写入并发、checkpoint 频率、WAL 产生速度、磁盘 flush latency、unlink/filter 请求数量。

资源压力会传播：backend 写 buffer 触发 md dirty request；bgwriter 和 checkpointer 写 buffer 也走同一路径；queue 满会把 fsync 延迟推回前台 backend；sync phase 变慢会拉长 checkpoint total time，并影响 WAL 回收、redo distance 和后续 checkpoint 压力。

能直接观察的是 checkpoint 日志中的 write/sync/total time、`pg_stat_bgwriter`/`pg_stat_checkpointer` 相关累计、`pg_stat_io` 的 data file fsync/write、wait event 和 `log_checkpoints`。只能推断的是 queue 压力、`pendingOps` entry 数、compact 触发次数和合法 cancel 是否刚好发生；这些需要 gdb 断点或临时 instrumentation。

## 41. 常见误区
- write 成功不等于持久化；没有 fsync，crash 后仍可能丢。
- checkpoint 写出 dirty buffers 不够；还必须处理 `ProcessSyncRequests()`。
- 共享 queue 不是最终状态；最终 pending fsync 在 checkpointer 本地 `pendingOps`。
- `pendingSyncHash` 不是 `pendingOps`；前者是 catalog storage 的 commit-time relation sync，后者是 checkpoint-time fsync table。
- fsync 的 ENOENT 不能无条件忽略；必须由 forget/filter cancel 解释。
- queue 满不是 correctness 失败；正确 fallback 是当前 backend 自己 fsync。

## 42. 课堂实验
实验 1：确认当前基线真实命名。

```bash
cd /home/nail/postgres-lab
rg -n "RegisterSyncRequest|RegisterSyncRequestType|SyncRequestType" \
  src/include/storage/sync.h src/backend/storage/sync/sync.c
```

结论应是 `SyncRequestType` 是 enum，`RegisterSyncRequest()` 是函数，当前基线没有 `RegisterSyncRequestType()`。

实验 2：静态画出 write 到 queue。

```bash
rg -n "FlushBuffer|smgrwrite\\(|mdwritev|register_dirty_segment|ForwardSyncRequest" \
  src/backend/storage src/backend/postmaster src/include
```

画出 `FlushBuffer()` 到 `ForwardSyncRequest()` 的调用链，并标注 buffer 清 dirty 前 fsync request 已登记或已 fallback fsync。

实验 3：观察 checkpoint sync phase。

```conf
log_checkpoints = on
```

制造写入负载后执行：

```sql
CHECKPOINT;
```

观察 checkpoint complete 日志中的 write time、sync time、total time。再用 gdb 断点 `register_dirty_segment`、`RegisterSyncRequest`、`ForwardSyncRequest`、`RememberSyncRequest` 对齐一次 request 的流转。

## 43. 讨论题
1. 为什么 fsync request 的粒度是 relation fork segment，而不是 relation 或 fd？
2. 为什么普通 backend 没有本地 `pendingOps`，而 checkpointer 有？
3. queue 允许重复请求会带来什么成本？为什么仍然可以接受？
4. `SYNC_FORGET_REQUEST` 和 `SYNC_FILTER_REQUEST` 为什么不能由 fsync 阶段简单判断 ENOENT 替代？
5. `mdunlink()` 为什么延迟删除普通 main fork 首段？
6. `RelationTruncate()` 的 `DELAY_CHKPT_START` 和 `DELAY_CHKPT_COMPLETE` 分别防什么窗口？
7. pending ops 为什么不需要 crash 后恢复？
8. 哪些指标能看到 sync phase 变慢，哪些内部状态只能断点或推断？

## 44. 本节小结
本节核心链路是：relation 文件写成功后，`md.c` 构造 `FileTag`，通过 `RegisterSyncRequest()` 投递 fsync 责任；checkpointer absorb 到 `pendingOps`；checkpoint sync phase 调 handler fsync；checkpoint 完成后再处理 delayed unlink。

核心状态边界是：共享 queue 是跨进程投递通道，`pendingOps` 是 checkpointer 本地 truth，`pendingUnlinks` 保存 checkpoint 后才能删除的对象。`FileTag` 用 handler、locator、fork 和 segment 表达跨进程文件身份，不能保存 fd。

异常和 fallback 是正确性核心：queue 满时 backend 自己 fsync；fsync ENOENT 只有在吸收到合法 cancel 后才安全；unlink/filter 不能丢；`AbsorbSyncRequests()` 清 queue 后如果不能记住请求必须 PANIC。pending ops 不写 WAL，因为它只承诺当前 checkpoint 完成前的运行时义务。

可观测入口是 checkpoint 日志、`pg_stat_io`、wait event 和断点；要区分 write phase 变慢、sync phase 变慢、queue pressure 和真实存储 fsync latency。可迁移规律是：异步合并可以优化延迟，但系统必须保留一个不能丢的责任账本；只要对外发布了更晚的恢复起点，这本账就必须已经兑现。
