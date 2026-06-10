# PostgreSQL WAL-before-data、page LSN 与持久化顺序

## 课程定位
本节主题：PostgreSQL 如何用 page LSN 把数据页写出和 WAL flush 绑定起来，从而允许数据页延迟写出，又禁止数据页早于描述它的 WAL 持久化。
前置知识：
- 已理解 shared buffer、buffer descriptor、pin、content lock 与 `BM_DIRTY`。
- 已理解 WAL record 是对数据页修改的 redo 描述。
- 已理解 checkpoint / bgwriter / backend 都可能把 dirty buffer 写出。
本节唯一主问题：
为什么 PostgreSQL 允许 dirty data page 晚写，却绝不允许它早于对应 WAL 持久化？
本节围绕的核心矛盾：
数据页写出必须尽量异步、合并、由后台推进。
但 crash recovery 又要求磁盘上的每个数据页状态都能用已经持久化的 WAL 解释。
一句话运行模型：
调用方修改 page 后用 `XLogInsert()` 得到 WAL record end pointer，再把这个位置写入 `pd_lsn`；任何写出永久数据页的路径最终进入 `FlushBuffer()`，先 `XLogFlush(PageGetLSN(page))`，再 `smgrwrite()` 数据页。
本节主流程：
access method 修改 page -> `MarkBufferDirty()` -> `XLogInsert()` -> `PageSetLSN()` -> checkpoint/bgwriter/backend eviction 进入 `FlushBuffer()` -> `XLogFlush(page_lsn)` -> data page write。
生命周期 / ownership / cleanup：access method 拥有 page 修改窗口，buffer manager 拥有 dirty/writeback 生命周期，WAL subsystem 拥有 flush 进度，ERROR 后由 ResourceOwner 和 buffer I/O cleanup 收尾。
错误路径 / 异常路径包括 page LSN 未设置、fake LSN 被错误用于 permanent page、dirty flag 丢失、I/O 写出失败，以及 WAL flush 失败。
学完后应该能独立判断：
- `PageSetLSN()` 应该由谁调用。
- `MarkBufferDirty()` 和 page LSN 为什么不是同一件事。
- `FlushBuffer()` 为什么是 WAL-before-data 的最后防线。
- permanent、unlogged、temporary relation 为什么不使用同一个持久化协议。
- 一个看似无害的写出顺序调整会怎样破坏 crash safety。

## 源码基线
本课基于本地源码：
`/home/nail/postgres-lab`
源码基线：
`master = bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`
该提交为：
`bd4bd30ce6 Use correct type for catalog_xmin`
本课引用的行号均来自这个本地基线。
如果你在别的 PostgreSQL 版本阅读，函数名大多稳定，但行号、注释和边界细节可能不同。
特别要注意：
本基线里 `BM_JUST_DIRTIED` 已不存在。
本课不讲旧版本 `BM_JUST_DIRTIED` 的历史协议。
本课只讲当前基线如何靠 dirty flag、content lock、page LSN 和 WAL flush 保持顺序。

## 阅读路径
核心阅读顺序如下：
1. `src/include/storage/bufpage.h`
2. `src/backend/access/transam/xloginsert.c`
3. `src/backend/access/heap/heapam.c`
4. `src/backend/access/nbtree/nbtinsert.c`
5. `src/backend/storage/buffer/bufmgr.c`
6. `src/backend/access/transam/xlog.c`
为什么不是先读 `bufmgr.c`？
因为 `FlushBuffer()` 只能解释“写出前检查什么”。
它不能解释 page LSN 从哪里来。
page LSN 是 access method 调用 `XLogInsert()` 后写入 page header 的。
所以阅读顺序应该先找“谁产生 LSN”，再看“谁消费 LSN”。
补充边界阅读：
- `src/include/utils/rel.h`
- `src/include/catalog/pg_class.h`
- `src/backend/storage/buffer/localbuf.c`
- `src/backend/access/transam/README`
这些文件不是本节主线，但能解释 permanent / unlogged / temp / wal_level=minimal 的边界。

## 本节先给结论
`PageGetLSN()` 读取的是 page header 里的 `pd_lsn`。
`PageSetLSN()` 写入的也是 page header 里的 `pd_lsn`。
`pd_lsn` 表示：
这个 page 当前内容依赖的最近一条 WAL record 的 end pointer。
`XLogInsert()` 返回的正是这个 end pointer。
`XLogInsertRecord()` 也返回 record end pointer。
源码注释直接说：
这个返回值可作为受影响数据页的 LSN。
`MarkBufferDirty()` 不生成 WAL。
`MarkBufferDirty()` 不设置 page LSN。
`MarkBufferDirty()` 不把数据写到磁盘。
它只把 buffer descriptor 的 `BM_DIRTY` 置上。
page LSN 是 page 内容的一部分。
dirty flag 是 buffer manager 的内存状态。
二者必须分离。
普通 WAL-logged 修改的典型顺序是：
1. 持有 page 的 exclusive content lock。
2. 修改 page 内容。
3. 调用 `MarkBufferDirty()`。
4. 注册 WAL record 数据。
5. 调用 `XLogInsert()` 得到 `recptr`。
6. 调用 `PageSetLSN(page, recptr)`。
7. 退出 critical section 并释放 content lock。
写出 dirty permanent buffer 的典型顺序是：
1. 持有 page 的 share-exclusive 或 exclusive content lock。
2. 进入 `FlushBuffer()`。
3. 读取 `BufferGetLSN(buf)`。
4. 如果 buffer 带 `BM_PERMANENT`，调用 `XLogFlush(page_lsn)`。
5. 更新 checksum。
6. 调用 `smgrwrite()` 写 data page。
7. `TerminateBufferIO()` 清理 I/O 状态并把 buffer 标成 clean。
这里的核心不变量是：
永久数据页落盘时，WAL flush LSN 必须大于等于该页的 page LSN。
用符号写就是：
`persistent_data_page_lsn <= durable_wal_flush_lsn`
这个不变量不是由 `MarkBufferDirty()` 保证的。
它由调用方设置 page LSN，加上 `FlushBuffer()` 写出前 `XLogFlush(page_lsn)` 共同保证。

## 1. 三个 LSN 位置不要混淆
读这节课时，先区分三个位置。
第一个位置是 WAL record 的位置。
它在 WAL byte stream 里。
`XLogInsert()` 负责分配、组装并插入 record。
返回值是 record end pointer。
第二个位置是 page header 的 `pd_lsn`。
它在 relation data page 里。
`PageSetLSN()` 把 WAL record end pointer 写到这里。
第三个位置是 WAL flush position。
它在 WAL 子系统的共享状态里。
`XLogFlush(record)` 负责把 WAL 至少刷到请求位置。
三者关系如下：
```text
page modification
  -> XLogInsert() returns record end pointer
  -> PageSetLSN(page, record_end)
  -> later FlushBuffer()
       -> PageGetLSN(page)
       -> XLogFlush(page_lsn)
       -> smgrwrite(page)
```
这个顺序允许 WAL 先插入但不马上 flush。
也允许 dirty page 在 buffer pool 里停留很久。
但只要 page 真要写到 relation 文件，`FlushBuffer()` 就会把 WAL flush 补齐。
这就是 WAL-before-data。
不要把它理解为“每次修改 page 都同步刷 WAL”。
PostgreSQL 的性能正是来自这两个动作的解耦：
WAL insert 可以在前台快速完成。
WAL flush 可以由 commit、walwriter、page flush 或其他路径推进。
data page write 可以由 checkpoint、bgwriter、backend eviction 或显式 flush 推进。
但最终 data page write 前必须看 page LSN。

## 2. `bufpage.h`：page LSN 是 page header 的字段
先读 `src/include/storage/bufpage.h`。
`PageHeaderData` 在 `bufpage.h:184-197`。
第一个字段就是：
```c
PageXLogRecPtr pd_lsn;
```
源码注释在 `bufpage.h:187-188` 说明：
它是最后一次修改该 page 的 WAL record 后一字节位置。
换句话说，page LSN 不是 record start。
它是 record end。
这和 `XLogInsert()` 返回值完全对齐。
`bufpage.h:152-155` 的注释直接点明用途：
dirty buffer 不能写到磁盘，除非 xlog 已经至少 flush 到 page LSN。
这段注释是本节最重要的接口契约之一。
它不属于 heap。
它不属于 btree。
它属于所有标准 PostgreSQL page。
`PageGetLSN()` 在 `bufpage.h:410-414`：
```c
static inline XLogRecPtr
PageGetLSN(const PageData *page)
{
    return PageXLogRecPtrGet(&((const PageHeaderData *) page)->pd_lsn);
}
```
`PageSetLSN()` 在 `bufpage.h:416-420`：
```c
static inline void
PageSetLSN(Page page, XLogRecPtr lsn)
{
    PageXLogRecPtrSet(&((PageHeader) page)->pd_lsn, lsn);
}
```
这两个函数只是 page header accessor。
它们不检查 relation 是否 permanent。
它们不检查 WAL 是否已经 flush。
它们不检查 buffer 是否 dirty。
它们也不负责加锁。
调用方必须在正确的锁和 WAL 协议下使用它们。
`bufpage.h:93-104` 还有一个容易忽略的实现细节。
LSN 用 `PageXLogRecPtr` 存储为一个 64-bit 值。
源码注释说这样是为了允许 atomic loads/stores。
但由于历史原因，小端平台上读写时会做高低 32 bit 交换。
所以课程和调试时都应该用 `PageGetLSN()` / `PageSetLSN()`。
不要直接把 `pd_lsn.lsn` 当作可读的 native LSN。
这也是为什么源码里有 `BufferGetLSNAtomic()`。
有些调用者只持有 share lock，需要 tear-free 地读取 LSN。
`bufmgr.c:4710-4749` 专门处理这个边界。

## 3. `XLogInsert()` 返回的是 page 应该记录的 LSN
再读 `src/backend/access/transam/xloginsert.c`。
`XLogInsert()` 在 `xloginsert.c:481-540`。
它的接口注释在 `xloginsert.c:470-479`。
关键语义是：
返回值是 XLOG pointer to end of record。
这个返回值可以作为受影响数据页的 LSN。
它表示 WAL 必须 flush 到哪里，数据页才能写出。
这段注释不是辅助说明。
它是 access method 和 buffer manager 之间的契约。
`XLogInsert()` 主体里会：
1. 检查 `XLogBeginInsert()` 是否已经调用。
2. 组装已注册的数据和 buffer references。
3. 调用 `XLogInsertRecord()`。
4. 如果 full page image 判断过期，重试。
5. 重置 insert 状态。
6. 返回 `EndPos`。
关键代码在 `xloginsert.c:529-535`：
```c
rdt = XLogRecordAssemble(...);
EndPos = XLogInsertRecord(...);
```
`XLogInsertRecord()` 在 `xlog.c:783-1041`。
它的注释在 `xlog.c:777-781` 重复了同一个契约：
返回 record end pointer。
该返回值可作为受影响数据页的 LSN。
`XLogInsertRecord()` 内部真正做两件事。
第一，保留 WAL 空间。
`xlog.c:902-903` 调用 `ReserveXLogInsertLocation()` 得到 `StartPos` 和 `EndPos`。
第二，把 WAL record 拷贝到 WAL buffers。
`xlog.c:959-961` 调用 `CopyXLogRecordToWAL()`。
注意：
`XLogInsert()` 完成后，WAL record 已经进入 WAL buffers。
但这不等于 WAL record 已经 fsync。
这也是为什么 `FlushBuffer()` 之后还要调用 `XLogFlush(page_lsn)`。
`XLogInsert()` 负责产生可排序、可 flush 的 WAL 位置。
`XLogFlush()` 负责把这个位置推进到持久化介质。
page LSN 把二者和 data page 写出连接起来。

## 4. Heap insert：最小可读的主流程
先看 `heap_insert()` 的关键片段。
位置在 `src/backend/access/heap/heapam.c:2055-2167`。
这段代码是最适合初学者跟读的 WAL-before-data 示例。
调用方先进入 critical section。
`heapam.c:2057`：
```c
START_CRIT_SECTION();
```
随后修改 page 内容。
`heapam.c:2059-2060` 调用 `RelationPutHeapTuple()` 把 tuple 放入 page。
如果 page 原来 all-visible，还会清掉 page flag 和 visibility map。
`heapam.c:2062-2068` 做这个动作。
如果需要设置 prune hint，`heapam.c:2084-2085` 更新 `pd_prune_xid`。
这些动作都改变了 page 内容。
然后调用：
`heapam.c:2087`：
```c
MarkBufferDirty(buffer);
```
这一行只说明 buffer 需要写回。
它还没有产生 WAL。
也还没有设置 page LSN。
接着进入 WAL 分支。
`heapam.c:2090`：
```c
if (RelationNeedsWAL(relation))
```
如果关系需要 WAL，heap 会构造 `xl_heap_insert`。
它注册 record body。
它注册受影响的 buffer。
关键调用在 `heapam.c:2140-2157`：
```c
XLogBeginInsert();
XLogRegisterData(...);
XLogRegisterBuffer(0, buffer, REGBUF_STANDARD | bufflags);
XLogRegisterBufData(...);
```
然后调用 `XLogInsert()`。
`heapam.c:2162`：
```c
recptr = XLogInsert(RM_HEAP_ID, info);
```
最后设置 page LSN。
`heapam.c:2164`：
```c
PageSetLSN(page, recptr);
```
这就是最典型的调用方协议：
先 dirty。
再 insert WAL。
再把 WAL record end pointer 写入 page header。
最后退出 critical section。
`heapam.c:2167`：
```c
END_CRIT_SECTION();
```
为什么 `MarkBufferDirty()` 在 `XLogInsert()` 之前？
不是因为 dirty flag 需要知道 WAL LSN。
恰好相反。
dirty flag 不知道 LSN。
这样做是为了 checkpoint 的并发判断。
`SyncOneBuffer()` 在 `bufmgr.c:4149-4157` 的注释解释了这个顺序。
它可以不拿 content lock 先检查 `BM_DIRTY`。
前提是 access method 在 `XLogInsert()` 前先把 page 标 dirty。
如果 checkpointer 刚检查完某页 clean，另一个 backend 才把它 dirty，那么新 WAL record 必然在 checkpoint redo point 之后。
这个 checkpoint 不需要写这个新 dirty page。
下一个 checkpoint 会处理它。
如果反过来先 `XLogInsert()`，再 `MarkBufferDirty()`，checkpoint 可能在两者之间开始。
它可能看到 page clean，却把 redo point 推进到无法恢复该 page 修改的位置。
所以“先 dirty，再 WAL”是 checkpoint 协议的一部分。
“先 WAL flush，再 data write”是写出协议的一部分。
二者不是同一层顺序。

## 5. Heap update / delete：多 page 时同一个 LSN 可覆盖多个 page
`heap_update()` 的关键片段在 `heapam.c:4050-4108`。
UPDATE 可能只修改一个 old page。
也可能 old tuple 和 new tuple 位于不同 page。
源码先改旧 tuple 的 `xmax`、`ctid` 等字段。
然后必要时清 visibility map。
如果新旧 page 不同，先 dirty 新 page。
`heapam.c:4078-4080`：
```c
if (newbuf != buffer)
    MarkBufferDirty(newbuf);
MarkBufferDirty(buffer);
```
随后如果 `RelationNeedsWAL(relation)`，调用 `log_heap_update()`。
`heapam.c:4097-4102`：
```c
recptr = log_heap_update(...);
```
这个 `recptr` 是描述本次 update 的 WAL record end pointer。
如果新旧 page 不同，两个 page 都要设置为同一个 LSN。
`heapam.c:4103-4107`：
```c
if (newbuf != buffer)
    PageSetLSN(newpage, recptr);
PageSetLSN(page, recptr);
```
这说明 page LSN 的含义不是“每个 page 自己有一个 WAL record”。
它的含义是“这个 page 当前内容依赖的最新 WAL record end pointer”。
一个 WAL record 可以描述多个 page 的修改。
这些 page 可以共享同一个 `recptr`。
`heap_delete()` 也一样。
关键片段在 `heapam.c:3000-3098`。
源码先把 tuple header 改成 deleted/locked 状态。
`heapam.c:3020` 调用 `MarkBufferDirty(buffer)`。
`heapam.c:3069-3072` 开始注册 WAL record 和 buffer。
`heapam.c:3093` 调用：
```c
recptr = XLogInsert(RM_HEAP_ID, XLOG_HEAP_DELETE);
```
`heapam.c:3095` 设置：
```c
PageSetLSN(page, recptr);
```
所以 heap 的 insert、update、delete 都体现同一个模式。
page 内容由 heap 修改。
dirty flag 由 heap 调 `MarkBufferDirty()`。
WAL record 由 heap 注册并插入。
page LSN 由 heap 用 `XLogInsert()` 返回值设置。
buffer manager 不参与构造 WAL record。
buffer manager 只在写出时消费 page LSN。

## 6. B-tree insert / split：为什么 fake LSN 也会出现
再看 `src/backend/access/nbtree/nbtinsert.c`。
B-tree 更能说明 page LSN 不只是 heap tuple 的东西。
它是 page 级通用协议。
简单 leaf insert 的关键片段在 `nbtinsert.c:1297-1421`。
源码进入 critical section。
它修改 leaf page。
可能还修改 meta page。
可能还清 child page 的 incomplete split flag。
然后 dirty 这些 buffer。
`nbtinsert.c:1306`：
```c
MarkBufferDirty(buf);
```
`nbtinsert.c:1315` 可能 dirty `metabuf`。
`nbtinsert.c:1329` 可能 dirty `cbuf`。
随后判断：
`nbtinsert.c:1333`：
```c
if (RelationNeedsWAL(rel))
```
需要 WAL 时注册 buffer 和 data。
`nbtinsert.c:1409`：
```c
recptr = XLogInsert(RM_BTREE_ID, xlinfo);
```
不需要 WAL 时不是简单跳过 LSN。
`nbtinsert.c:1411-1412`：
```c
else
    recptr = XLogGetFakeLSN(rel);
```
随后对所有被修改的 page 设置 LSN。
`nbtinsert.c:1414-1419`：
```c
if (BufferIsValid(metabuf))
    PageSetLSN(metapg, recptr);
if (!isleaf)
    PageSetLSN(BufferGetPage(cbuf), recptr);
PageSetLSN(page, recptr);
```
为什么 non-WAL path 还需要 fake LSN？
`xloginsert.c:555-559` 的注释解释：
某些 index AM 用 LSN 检测并发 page 修改。
不是所有 index page 都有真实 WAL。
所以 `XLogGetFakeLSN()` 提供一个序列值。
fake LSN 不代表 WAL byte stream 中真实可 flush 的位置。
这正是 `FlushBuffer()` 必须区分 `BM_PERMANENT` 的原因之一。
如果 unlogged index page 的 fake LSN 被拿去 `XLogFlush()`，可能会要求 flush 到 WAL 插入位置之后。
`bufmgr.c:4558-4568` 的注释专门防止这个灾难。
B-tree split 更明显。
关键片段在 `nbtinsert.c:1952-2095`。
一次 split 可能修改：
- 原左页。
- 新右页。
- 原右兄弟页。
- child page。
源码先 `MarkBufferDirty()` 多个 buffer。
`nbtinsert.c:1974-1980` dirty left、right、sibling。
`nbtinsert.c:1993` 可能 dirty child。
需要 WAL 时：
`nbtinsert.c:2083`：
```c
recptr = XLogInsert(RM_BTREE_ID, xlinfo);
```
不需要 WAL 时：
`nbtinsert.c:2086`：
```c
recptr = XLogGetFakeLSN(rel);
```
随后：
`nbtinsert.c:2088-2093` 对所有被修改 page 设置同一个 `recptr`。
B-tree new root 也一样。
`nbtinsert.c:2598-2601` dirty left child、new root、meta page。
`nbtinsert.c:2639` 需要 WAL 时插入 `XLOG_BTREE_NEWROOT`。
`nbtinsert.c:2642` 否则取 fake LSN。
`nbtinsert.c:2644-2646` 设置三个 page 的 LSN。
这些调用点共同说明：
page LSN 是 access method 维护的 page header 状态。
它既服务 WAL-before-data。
也可能服务 AM 内部并发检测。
但只有真实 WAL LSN 才能用于 WAL flush。

## 7. `MarkBufferDirty()` 与 page LSN 的职责分离
现在回到 `src/backend/storage/buffer/bufmgr.c`。
`MarkBufferDirty()` 在 `bufmgr.c:3156-3205`。
函数注释在 `bufmgr.c:3147-3153`。
核心句子是：
`Marks buffer contents as dirty (actual write happens later).`
这个函数的职责非常窄。
它只负责 buffer manager 的 dirty 状态。
shared buffer 路径上，它要求：
- buffer ID 有效。
- buffer 被当前 backend pin。
- 当前 backend 持有 exclusive content lock。
这些断言在 `bufmgr.c:3162-3174`。
然后它读取 `BufferDesc.state`。
如果 header 被 `BM_LOCKED`，就等待。
最后在 CAS loop 中 OR 上 `BM_DIRTY`。
关键语句在 `bufmgr.c:3188-3192`：
```c
Assert(BUF_STATE_GET_REFCOUNT(buf_state) > 0);
buf_state |= BM_DIRTY;
pg_atomic_compare_exchange_u64(...);
```
成功后，如果旧状态不是 dirty，才计入 `shared_blks_dirtied`。
`bufmgr.c:3196-3204` 做这个 accounting。
这里没有 `XLogBeginInsert()`。
这里没有 `XLogInsert()`。
这里没有 `PageSetLSN()`。
这里没有 `XLogFlush()`。
这里也没有 `smgrwrite()`。
所以：
`MarkBufferDirty()` 不是 durability API。
它是 buffer manager 状态 API。
page LSN 是 page 内容的一部分。
dirty flag 是 buffer descriptor 的状态位。
如果让 `MarkBufferDirty()` 自动设置 page LSN，它必须知道：
- 本次修改属于哪个 rmgr。
- WAL record 是否已经注册完整。
- 一个 WAL record 修改了哪些 page。
- 是否需要 FPI。
- 是否是 RelationNeedsWAL false。
- 是否需要 fake LSN。
- 是否是 hint bit。
这些信息都不在 buffer manager。
它们在 heap、btree、visibility map、FSM、VM 或其他 access method。
因此职责必须分离。
调用方负责描述修改并设置 page LSN。
buffer manager 负责记住该 buffer 需要写回。
写出路径负责按 page LSN 刷 WAL。
这三个职责不能合并。

## 8. 为什么要先 `MarkBufferDirty()` 再 `XLogInsert()`
这点容易和 WAL-before-data 混淆。
普通修改里，源码通常先 `MarkBufferDirty()`，再 `XLogInsert()`。
这不是违反 WAL-before-data。
因为 WAL-before-data 说的是：
WAL 必须先于 data page 持久化。
它不要求 WAL record 必须先于内存里的 page 修改。
内存里的 page 修改在 shared buffers 中。
只要它还没写到 relation 文件，就没有破坏 crash recovery。
`SyncOneBuffer()` 的注释在 `bufmgr.c:4149-4157` 解释了先 dirty 的必要性。
checkpointer / bgwriter 可以先在 header lock 下检查 `BM_DIRTY`。
它们不想为每个 buffer 都拿 content lock。
这个优化成立的条件是：
access method 在 `XLogInsert()` 前先把 page dirty。
如果 checkpointer 看见 page clean，然后另一个 backend 之后才 dirty 并插入 WAL，checkpoint redo point 在这条新 WAL 之前。
该 checkpoint 不需要写这个 page。
如果改成先 `XLogInsert()` 后 `MarkBufferDirty()`，就会出现危险窗口。
checkpoint 可能在 WAL record 已经存在、page 还没 dirty 时扫描 buffer pool。
它可能不写这个 page。
随后它记录一个新的 checkpoint。
如果 crash recovery 从这个 checkpoint 后开始，之前那条 WAL record 可能不再被 replay。
但数据页修改也没有写到磁盘。
这就是丢修改。
所以这里有两个顺序：
内存修改协议：
```text
modify page
  -> MarkBufferDirty()
  -> XLogInsert()
  -> PageSetLSN()
```
持久化协议：
```text
XLogFlush(page_lsn)
  -> smgrwrite(data page)
```
第一个顺序服务 checkpoint 的 dirty page 集合。
第二个顺序服务 crash recovery 的 WAL-before-data。
它们共同组成完整协议。

## 9. `FlushBuffer()`：WAL-before-data 的最后防线
`FlushBuffer()` 在 `bufmgr.c:4512-4628`。
这是本节最重要的消费端。
它的调用方已经持有 buffer content lock。
断言在 `bufmgr.c:4520-4521`：
```c
Assert(BufferLockHeldByMeInMode(buf, BUFFER_LOCK_EXCLUSIVE) ||
       BufferLockHeldByMeInMode(buf, BUFFER_LOCK_SHARE_EXCLUSIVE));
```
share-exclusive 已经足够。
因为写出时不需要逻辑修改 page。
但必须阻止别人同时 compact、改 tuple、设重要字段或改 LSN。
随后调用 `StartSharedBufferIO()`。
`bufmgr.c:4523-4529`：
如果另一个 backend 已经完成写出，就直接返回。
如果当前 backend 成功开始 I/O，它拥有这个 buffer 的写出权。
然后找到 `SMgrRelation`。
`bufmgr.c:4537-4539`。
接着读取 page LSN。
`bufmgr.c:4547-4551`：
```c
recptr = BufferGetLSN(buf);
```
源码注释强调：
当前 backend 至少持有 share-exclusive content lock。
因此 flush 期间 LSN 不会变化，也不会 torn。
这点很关键。
否则 flusher 可能读到半个旧 LSN、半个新 LSN。
或者读完 LSN 后 page 又被改成依赖更靠后的 WAL record。
当前锁协议避免了这类问题。
然后是本节核心语句。
`bufmgr.c:4553-4571`：
```c
if (pg_atomic_read_u64(&buf->state) & BM_PERMANENT)
    XLogFlush(recptr);
```
注释说明：
这实现了 WAL 的基本规则。
log updates must hit disk before data-file changes they describe do。
也就是说：
只要 buffer 是 permanent，写 data page 前就必须让 WAL 至少 flush 到 page LSN。
随后才进入真正的数据页写出。
`bufmgr.c:4573-4590`：
1. 取 `bufBlock`。
2. 更新 page checksum。
3. 调用 `smgrwrite()`。
最后：
`bufmgr.c:4615-4618` 调用 `TerminateBufferIO()`。
它结束 `BM_IO_IN_PROGRESS`。
它把 buffer 标成 clean。
它唤醒等待者。
因此 `FlushBuffer()` 的顺序可以压缩成：
```text
StartSharedBufferIO()
  -> read page_lsn while holding content lock
  -> if BM_PERMANENT: XLogFlush(page_lsn)
  -> PageSetChecksum()
  -> smgrwrite()
  -> TerminateBufferIO(clean)
```
如果只记一个函数，就记 `FlushBuffer()`。
它是所有 permanent dirty data page 写出前的 WAL-before-data gate。

## 10. `XLogFlush()`：请求 flush 到指定 record 位置
`XLogFlush()` 在 `src/backend/access/transam/xlog.c:2800-2977`。
接口注释在 `xlog.c:2794-2798`：
确保所有 XLOG data through 给定位置都 flush 到磁盘。
它的参数名叫 `record`。
在 `FlushBuffer()` 中，这个参数就是 page LSN。
`xlog.c:2820-2822` 先做 quick exit。
如果请求位置已经小于等于 `LogwrtResult.Flush`，直接返回。
这意味着 `FlushBuffer()` 调 `XLogFlush(page_lsn)` 不一定真的 fsync。
如果 commit、walwriter 或其他 backend 已经把 WAL flush 过了，它只是一次快速检查。
如果还没 flush 到请求位置，`XLogFlush()` 会进入 critical section。
它会等待相关 WAL insertions 完成。
`xlog.c:2858-2866` 说明这一点。
然后争用 `WALWriteLock`。
`xlog.c:2868-2883`。
拿到锁后再检查是否已被别人 flush。
`xlog.c:2885-2891`。
如果仍未满足，就调用 `XLogWrite()`。
`xlog.c:2922-2927`：
```c
WriteRqst.Write = insertpos;
WriteRqst.Flush = insertpos;
XLogWrite(WriteRqst, insertTLI, false);
```
最后会检查：
`xlog.c:2965-2969`。
如果 `LogwrtResult.Flush < record`，报错。
这段注释还提到一个现实问题：
如果 data page 上的 LSN 损坏，可能请求 flush 到 WAL 末尾之后的位置。
这在 `bufmgr.c` 调用时不会被升级为 PANIC。
因为从数据页读到坏 LSN 不应该直接拖垮整个系统。
这也说明 page LSN 是强协议字段。
它错了，写出路径就会要求 WAL 子系统满足一个错误的持久化位置。

## 11. checkpoint 写出路径
checkpoint 写 dirty buffer 的入口是 `BufferSync()`。
位置在 `bufmgr.c:3561-3825`。
注释在 `bufmgr.c:3550-3558`：
checkpoint time 写出 dirty shared buffers。
普通 checkpoint 默认只写 permanent dirty buffers。
shutdown checkpoint、end-of-recovery checkpoint 或 `CHECKPOINT_FLUSH_UNLOGGED` 会写 unlogged buffers。
这个边界由 mask 控制。
`bufmgr.c:3576-3583`：
```c
uint64 mask = BM_DIRTY;
if (!(shutdown/end_of_recovery/flush_unlogged))
    mask |= BM_PERMANENT;
```
所以普通 checkpoint 不负责把 unlogged relation 的普通 fork 刷成 crash-safe 状态。
unlogged relation crash 后本来就要丢弃。
`BufferSync()` 先扫描整个 buffer pool。
对需要写的 buffer 设置 `BM_CHECKPOINT_NEEDED`。
`bufmgr.c:3585-3595` 的注释说明：
这样只写 checkpoint 开始时已经 dirty 的 page。
checkpoint 过程中后来 dirty 的 page 不属于本次 checkpoint 的必要集合。
这和 `MarkBufferDirty()` 必须早于 `XLogInsert()` 的协议相连。
真正写出时，`BufferSync()` 会调用 `SyncOneBuffer()`。
`bufmgr.c:3772-3774`：
```c
if (state & BM_CHECKPOINT_NEEDED)
    SyncOneBuffer(...)
```
`SyncOneBuffer()` 在 `bufmgr.c:4138-4198`。
它先在 header lock 下判断 `BM_VALID` 和 `BM_DIRTY`。
如果需要写，就 pin buffer。
然后调用：
`bufmgr.c:4185`：
```c
FlushUnlockedBuffer(bufHdr, NULL, IOOBJECT_RELATION, IOCONTEXT_NORMAL);
```
`FlushUnlockedBuffer()` 在 `bufmgr.c:4635-4642`。
它获取 share-exclusive content lock。
然后调用 `FlushBuffer()`。
因此 checkpoint 路径最终仍然进入：
```text
FlushBuffer()
  -> XLogFlush(page_lsn)
  -> smgrwrite()
```
checkpoint 自己不重写 WAL-before-data 协议。
它只是选择哪些 dirty buffer 需要推进。

## 12. background writer 写出路径
background writer 的入口是 `BgBufferSync()`。
位置在 `bufmgr.c:3840-4118`。
它的目标不是完成 checkpoint correctness。
它的目标是提前制造 reusable clean buffers。
`bufmgr.c:4048-4052` 的注释说明：
bgwriter 从 `next_to_clean` 向前扫描 dirty reusable buffers。
它会受到 `bgwriter_lru_maxpages` 等限制。
真正写每个 buffer 的地方仍然是：
`bufmgr.c:4062`：
```c
SyncOneBuffer(next_to_clean, true, wb_context);
```
`skip_recently_used = true`。
这意味着 bgwriter 会跳过 pinned 或 recently-used 的 buffer。
它不想干扰活跃工作集。
但只要它真的写一个 permanent dirty buffer，路径仍然进入 `FlushBuffer()`。
所以 bgwriter 也可能触发 `XLogFlush(page_lsn)`。
这点对性能诊断很重要。
有时你看到前台 backend 没有提交等待，却仍然有 WAL flush。
原因可能是后台写 dirty page 时发现 page LSN 超过当前 WAL flush position。
WAL-before-data 不只发生在 commit。
它也发生在 data page writeback。

## 13. backend eviction 写出路径
前台 backend 也可能写 dirty page。
典型场景是 buffer allocation 需要复用 victim buffer。
关键片段在 `bufmgr.c:2585-2639`。
当 victim 是 dirty buffer 时，backend 必须先把它写出。
源码先尝试拿 share-exclusive content lock。
`bufmgr.c:2590-2603` 注释解释：
写出需要 share-exclusive lock，否则可能把别人正在 compact 的半状态 page 写到磁盘。
这里用 conditional lock 避免死锁。
如果使用 access strategy ring，还会做一个优化判断。
`bufmgr.c:2613-2627`：
如果 victim 来自 ring，且是 permanent buffer，且 `XLogNeedsFlush(BufferGetLSN(buf_hdr))`，strategy 可以拒绝这个 victim。
原因是复用它会触发 WAL flush。
对 bulk read / bulk write ring 来说，这可能把一次 buffer miss 放大成 WAL fsync latency。
注意注释说：
这个判断只适用于 permanent buffers。
unlogged buffers 可以有 fake LSN。
因此 `XLogNeedsFlush()` 对它们没有意义。
如果不拒绝 victim，backend 调用：
`bufmgr.c:2634`：
```c
FlushBuffer(buf_hdr, NULL, IOOBJECT_RELATION, io_context);
```
所以前台 backend eviction 也是同一个最终路径。
这就是为什么一个读请求可能被脏页写回和 WAL flush 拖慢。
它本来只是想读另一个 block。
但它选中的 victim 是 dirty permanent page。
该 page 的 LSN 又超过 durable WAL flush position。
于是它必须先 flush WAL，再写 data page，再复用 buffer。

## 14. 其他显式 flush 路径
`FlushRelationBuffers()` 在 `bufmgr.c:5171-5240` 之后继续实现 shared buffer 扫描。
注释在 `bufmgr.c:5153-5167`：
它把一个 relation 的 dirty pages 写到 kernel disk buffers。
通常调用方应持有 `AccessExclusiveLock`，防止别人继续 dirty。
对于 temporary relation，它走 local buffer 分支。
`bufmgr.c:5177-5208` 会调用 `FlushLocalBuffer()`。
对于 shared buffers，它仍会调用 `FlushUnlockedBuffer()`。
`FlushDatabaseBuffers()` 在 `bufmgr.c:5535-5568`。
它扫描某个 database 的 dirty buffers。
注释在 `bufmgr.c:5530-5531` 说明：
不关心 temporary relation pages。
因为它们对数据库级持久化 flush 不重要。
真正写出 shared buffer 的地方是：
`bufmgr.c:5562`：
```c
FlushUnlockedBuffer(bufHdr, NULL, IOOBJECT_RELATION, IOCONTEXT_NORMAL);
```
`FlushOneBuffer()` 在 `bufmgr.c:5575-5589`。
它要求调用方已经 pin 且锁住 buffer。
然后直接：
`bufmgr.c:5588`：
```c
FlushBuffer(bufHdr, NULL, IOOBJECT_RELATION, IOCONTEXT_NORMAL);
```
这些路径说明：
PostgreSQL 没有在每个调用点复制 WAL-before-data 逻辑。
它把最终顺序收敛到 `FlushBuffer()`。
这也是内核代码常见结构：
上游路径选择对象和时机。
下游共享函数守住正确性不变量。

## 15. permanent / unlogged / temp：三个边界
现在讲关系持久化边界。
`pg_class.h:183-185` 定义：
```c
#define RELPERSISTENCE_PERMANENT 'p'
#define RELPERSISTENCE_UNLOGGED  'u'
#define RELPERSISTENCE_TEMP      't'
```
这三个值不是只影响 SQL 语义。
它们直接影响 buffer 和 WAL 协议。
`rel.h:624-629` 定义 `RelationIsPermanent()`。
只有 `relpersistence == RELPERSISTENCE_PERMANENT` 为真。
`rel.h:632-642` 定义 `RelationNeedsWAL()`。
它要求 relation 是 permanent。
但 permanent 不必然每次都需要 WAL。
在 `wal_level=minimal` 下，如果当前事务新建或截断了 relfilenode，某些修改可以跳过 WAL。
`rel.h:635-637` 的注释指向 `src/backend/access/transam/README`。
README 的 “Skipping WAL for New RelFileLocator” 说明：
当前事务会在 commit 前写并 fsync 受影响 blocks。
因此这类跳过 WAL 的修改不是靠 page LSN + WAL redo 恢复。
它靠 commit 前同步新文件内容。
这是一条特殊但受控的持久化路径。
不要把它误读成“permanent page 可以随便没有 WAL”。
`RelationUsesLocalBuffers()` 在 `rel.h:645-649`。
temporary relation 使用 local buffers。
这意味着 temp relation 根本不在 shared buffer pool 的同一套写出路径里。

## 16. `BM_PERMANENT`：FlushBuffer 判断的是 buffer 状态
`FlushBuffer()` 不直接看 `RelationNeedsWAL()`。
它看 `BM_PERMANENT`。
`BM_PERMANENT` 在 buffer descriptor state 中。
buffer 分配时设置。
关键片段在 `bufmgr.c:2331-2338`：
```c
if (relpersistence == RELPERSISTENCE_PERMANENT || forkNum == INIT_FORKNUM)
    set_bits |= BM_PERMANENT;
```
注释说明：
永久关系的 buffer 必须每次 checkpoint 写出。
unlogged buffers 只需要在 shutdown checkpoint 写出。
但 unlogged relation 的 `init` fork 要像 permanent relation 一样处理。
原因是 unlogged relation crash 后会从 init fork 重建。
init fork 本身必须持久存在。
这就是为什么判断条件不是单纯的 `relpersistence == PERMANENT`。
`BufferIsPermanent()` 在 `bufmgr.c:4686-4707`。
它也只看 `BM_PERMANENT`。
local buffer 直接返回 false。
这说明 permanent 边界进入 buffer manager 后，被编码成 buffer state flag。
`FlushBuffer()` 只需要消费这个 flag。
`bufmgr.c:4570-4571`：
```c
if (state & BM_PERMANENT)
    XLogFlush(recptr);
```
所以：
`RelationNeedsWAL()` 是 access method 生成 WAL 的判断。
`BM_PERMANENT` 是 buffer 写出前是否要按 page LSN flush WAL 的判断。
这两个判断相关，但不等价。

## 17. unlogged relation：不是 crash-safe data page
unlogged relation 的普通数据 fork 不写普通 WAL。
crash 后它的内容不保证保留。
所以普通 unlogged page 写出时，不应该要求 WAL-before-data。
`FlushBuffer()` 的注释在 `bufmgr.c:4558-4568` 说得很直接。
WAL-before-data 规则不适用于 unlogged relations。
大多数 unlogged pages 没有真实 LSN。
有些 index AM 会使用 fake LSN 检测并发 page 修改。
fake LSN 可能超过 WAL insertion point。
如果拿 fake LSN 调 `XLogFlush()`，可能导致系统范围的灾难。
因此非 permanent buffer 跳过 `XLogFlush()`。
这解释了 `BM_PERMANENT` 判断的必要性。
它不是性能优化。
它是正确性边界。
B-tree 的 `XLogGetFakeLSN()` 调用正好对应这个注释。
`nbtinsert.c:1412`、`2086`、`2642` 都可能在 `RelationNeedsWAL(rel)` 为 false 时取 fake LSN。
这些 LSN 可写入 page header。
但它们不代表 WAL byte stream 的 durable position。
所以不能用于 WAL flush。

## 18. temporary relation：local buffers，不进 shared WAL-before-data gate
temporary relation 使用 local buffers。
这由 `RelationUsesLocalBuffers()` 决定。
local buffer 属于单个 backend。
其他 backend 看不到。
crash 后也不需要恢复。
`localbuf.c` 是补充阅读。
`FlushLocalBuffer()` 在 `localbuf.c:182-218`。
它和 `FlushBuffer()` 很像。
但它没有 `XLogFlush(PageGetLSN(page))`。
它只做：
1. `StartLocalBufferIO()`。
2. 找到 smgr relation。
3. `PageSetChecksum()`。
4. `smgrwrite()`。
5. 统计 temp relation write。
6. 清 dirty。
`MarkLocalBufferDirty()` 在 `localbuf.c:500-522`。
它只更新 local buffer 的 dirty state 和统计。
这说明 temp relation 的数据页写出不是 crash recovery 协议的一部分。
它只是当前 backend 的临时文件 I/O。
因此不要在 temp relation 上寻找 WAL-before-data 行为。
它没有这个持久化承诺。

## 19. crash safety 反例一：数据页早于 WAL 落盘
假设有人把 `FlushBuffer()` 改成先 `smgrwrite()` 再 `XLogFlush(page_lsn)`。
序列如下：
```text
T1 modifies heap page P
T1 inserts WAL record R
T1 sets PageSetLSN(P, end(R))
flusher writes P to relation file
system crashes before WAL containing R is flushed
```
磁盘上可能已经有包含修改的数据页 P。
但 WAL 中没有 record R。
crash recovery 从最近 checkpoint 开始读取 WAL。
它看不到 R。
因此它无法解释 P 上的修改。
如果 P 是 heap page，可能出现 tuple header 已更新但事务状态 WAL 不完整。
如果 P 是 btree page，可能出现 sibling link、high key 或 parent downlink 与 WAL 历史不一致。
如果 P 是 visibility map 相关 page，可能出现 heap 与 VM 的持久化状态不匹配。
最要命的是：
recovery 没有“根据数据页推导缺失 WAL”的能力。
WAL 是恢复真相来源。
数据页只能是 WAL 已持久化历史的某个结果。
因此数据页不能早于 WAL 落盘。

## 20. crash safety 反例二：page LSN 没有设置
假设 access method 调了 `XLogInsert()`，但忘记 `PageSetLSN(page, recptr)`。
序列如下：
```text
page P has old LSN L0
T1 modifies P
T1 inserts WAL record R with end L1
T1 forgets PageSetLSN(P, L1)
flusher writes P with page LSN still L0
crash
recovery replays R
```
磁盘上的 P 可能已经包含 T1 的修改。
但它的 page LSN 仍然是 L0。
redo 看到 record R 时，可能认为 P 还没应用过 R。
于是它再次应用 R。
有些 redo 操作设计得尽量幂等。
但不能把“忘记设置 page LSN”当成可接受行为。
page LSN 是 redo 判断是否需要应用 record 的重要输入。
如果 page 内容和 page LSN 不一致，recovery 就失去了判断基础。
所以 page LSN 必须和 page 内容一起落盘。
这也是它放在 page header 中的原因。

## 21. crash safety 反例三：dirty flag 没有设置
再假设 access method 修改 page、插入 WAL、设置 page LSN，但忘记 `MarkBufferDirty()`。
短期看，查询可能正常。
因为修改还在 shared buffer 里。
但 buffer manager 认为这个 buffer 是 clean。
checkpoint 扫描时可能不会写它。
victim eviction 也可能直接复用它。
更危险的是 checkpoint。
如果 WAL record 已经在 checkpoint redo point 之前，而 page 修改没有被写出，checkpoint 可能仍然完成。
之后 crash recovery 从较新的 checkpoint 开始。
它不再 replay 那条旧 record。
但 data page 上也没有修改。
这就是 silent data loss。
所以 dirty flag 是 checkpoint correctness 的输入。
page LSN 是 WAL-before-data 的输入。
两者缺一不可。

## 22. crash safety 反例四：用 fake LSN 去 flush WAL
再看 unlogged index 的 fake LSN。
B-tree 在 `RelationNeedsWAL(rel)` 为 false 时可能调用 `XLogGetFakeLSN()`。
fake LSN 可以写入 page。
它用于检测 page 是否被并发修改。
但它不是 WAL byte stream 的真实位置。
如果 `FlushBuffer()` 对所有 dirty buffer 都调用 `XLogFlush(PageGetLSN(page))`，就可能请求 WAL flush 到一个根本不存在的位置。
`bufmgr.c:4562-4568` 的注释正是在防这个问题。
所以 `BM_PERMANENT` 判断不能删。
它不只是为了少一次 flush。
它防止 fake LSN 被误解释为 durable WAL LSN。

## 23. 成本、资源、观测与诊断入口
运行时最容易观察的是三个现象。
第一个是 WAL insert LSN 与 WAL flush LSN 的差距。
SQL 可以看：
```sql
select pg_current_wal_insert_lsn(), pg_current_wal_flush_lsn();
```
差距越大，说明有 WAL 已插入但未 flush。
但这不一定是问题。
如果没有 commit 等待，也没有 dirty page 需要写出，这个差距可以暂时存在。
第二个是 dirty page 写出会触发 WAL flush。
可以在 `FlushBuffer()` 的 `XLogFlush(recptr)` 处打断点。
当 dirty permanent page 的 LSN 超过 `LogwrtResult.Flush`，写出路径会推进 WAL。
第三个是 page header LSN。
如果安装了 `pageinspect`，可以用 `page_header(get_raw_page(...))` 查看页面 LSN。
例如：
```sql
create extension if not exists pageinspect;
select * from page_header(get_raw_page('t', 0));
```
这能把 SQL 级写入和 page header 状态连起来。
不过要注意：
`get_raw_page()` 读的是磁盘文件页面，不是 shared buffer 中尚未写出的内存页面。
如果想看内存里的 page LSN，需要 gdb 或源码插桩。
第四个入口是统计。
`pg_stat_bgwriter`、I/O stats 和 `pg_stat_wal` 可以帮助判断是谁推进写出和 WAL。
但统计一般只能看到结果。
它不能直接告诉你某个 `FlushBuffer()` 是否因为 page LSN 调用了真实 fsync。
这个问题通常要结合 gdb、tracepoint、perf 或日志插桩。

## 24. 源码跟读练习一：从 page header 开始
目标：
确认 page LSN 的存储位置和 accessor。
步骤：
1. 打开 `src/include/storage/bufpage.h`。
2. 阅读 `bufpage.h:93-104`。
3. 阅读 `bufpage.h:184-188`。
4. 阅读 `bufpage.h:410-420`。
回答：
`pd_lsn` 为什么是 page header 的第一个字段？
`PageGetLSN()` 和 `PageSetLSN()` 有没有任何 WAL flush 逻辑？
为什么不能直接读取 `pd_lsn.lsn`？
如果一个 page 没有被格式化，能不能安全设置 LSN？
参考 `xloginsert.c:1205-1212`。
`log_newpage()` 对 `PageIsNew(page)` 做了特殊判断。

## 25. 源码跟读练习二：从 `XLogInsert()` 到 heap page
目标：
确认 `XLogInsert()` 返回值怎样进入 heap page header。
步骤：
1. 打开 `xloginsert.c:470-540`。
2. 找到 `XLogInsert()` 返回 `EndPos`。
3. 打开 `xlog.c:777-781`。
4. 确认 `XLogInsertRecord()` 注释同样说明返回 record end pointer。
5. 打开 `heapam.c:2055-2167`。
6. 跟读 `MarkBufferDirty()`、`XLogInsert()`、`PageSetLSN()` 的相对顺序。
回答：
heap insert 为什么先修改 page，再写 WAL record？
为什么这不违反 WAL-before-data？
如果 `RelationNeedsWAL(relation)` 为 false，这段 heap insert 会不会调用 `PageSetLSN()`？
这个问题要和 `wal_level=minimal` 的新 relfilenode规则联系起来看。

## 26. 源码跟读练习三：从 btree split 看多 page LSN
目标：
确认一个 WAL record 可以成为多个 page 的 LSN。
步骤：
1. 打开 `nbtinsert.c:1952-2095`。
2. 列出 split 修改的 buffer。
3. 找到所有 `MarkBufferDirty()`。
4. 找到 `XLogInsert()` 或 `XLogGetFakeLSN()`。
5. 找到所有 `PageSetLSN()`。
回答：
为什么 left page、right page、sibling page 可能共享同一个 `recptr`？
为什么 non-WAL path 仍然设置 page LSN？
fake LSN 能不能交给 `XLogFlush()`？
哪个函数防止了这个错误？

## 27. 源码跟读练习四：从写出路径收敛到 `FlushBuffer()`
目标：
确认 checkpoint、bgwriter、backend eviction 都共享同一个 WAL-before-data gate。
步骤：
1. 打开 `bufmgr.c:3561-3825`。
2. 找到 `BufferSync()` 调用 `SyncOneBuffer()` 的位置。
3. 打开 `bufmgr.c:3840-4118`。
4. 找到 `BgBufferSync()` 调用 `SyncOneBuffer()` 的位置。
5. 打开 `bufmgr.c:2585-2639`。
6. 找到 backend eviction 调用 `FlushBuffer()` 的位置。
7. 打开 `bufmgr.c:4512-4628`。
8. 跟读 `XLogFlush(recptr)` 和 `smgrwrite()` 的顺序。
回答：
为什么 `SyncOneBuffer()` 可以先不拿 content lock 检查 `BM_DIRTY`？
为什么真正写出时必须拿 share-exclusive content lock？
为什么 backend eviction 可能因为 dirty victim 而等待 WAL flush？

## 28. 实验一：用 gdb 观察 heap insert 设置 page LSN
准备：
使用 `/home/nail/postgres-lab` 编译出的本地 PostgreSQL。
初始化一个测试实例。
建议打开 debug symbols。
步骤：
1. 启动 postgres。
2. 用 psql 创建表。
3. 找到 backend pid。
4. gdb attach 到该 backend。
5. 在 `heap_insert` 附近设置断点。
6. 在 `XLogInsert` 返回后观察 `recptr`。
7. 在 `PageSetLSN(page, recptr)` 前后观察 `PageGetLSN(page)`。
SQL 示例：
```sql
create table wal_lsn_demo(id int, payload text);
insert into wal_lsn_demo values (1, repeat('x', 100));
```
gdb 观察点：
```gdb
break heap_insert
break XLogInsert
break PageSetLSN
```
如果内联函数不好断，可以在调用点附近打行号断点。
观察目标：
`XLogInsert()` 返回的 `recptr` 与随后 page header 里的 LSN 一致。
`MarkBufferDirty()` 发生在 `XLogInsert()` 前。
退出 critical section 前 page LSN 已经设置完成。

## 29. 实验二：观察 `FlushBuffer()` 先刷 WAL 再写 data page
目标：
确认 dirty permanent buffer 写出前会调用 `XLogFlush(page_lsn)`。
步骤：
1. 创建普通 permanent table。
2. 插入一些数据。
3. 在 `FlushBuffer()` 设置断点。
4. 在 `XLogFlush()` 设置断点。
5. 执行 `checkpoint;`。
SQL 示例：
```sql
create table flush_demo(id int, payload text);
insert into flush_demo
select g, repeat('payload', 100)
from generate_series(1, 10000) g;
checkpoint;
```
gdb 观察点：
```gdb
break FlushBuffer
break XLogFlush
break smgrwrite
```
观察目标：
进入 `FlushBuffer()` 后先读 `recptr = BufferGetLSN(buf)`。
如果 `BM_PERMANENT` 为真，会进入 `XLogFlush(recptr)`。
随后才调用 `smgrwrite()`。
如果 WAL 已经 flush 到 `recptr`，`XLogFlush()` 可能 quick exit。
这仍然满足协议。

## 30. 实验三：比较 permanent、unlogged、temp
目标：
确认三类 relation 的写出边界不同。
SQL：
```sql
create table p_demo(id int);
create unlogged table u_demo(id int);
create temporary table t_demo(id int);
insert into p_demo values (1);
insert into u_demo values (1);
insert into t_demo values (1);
checkpoint;
```
观察点：
普通表使用 shared buffers。
unlogged 表也使用 shared buffers，但普通 fork 不应作为 crash-safe WAL-redo 数据处理。
temporary 表使用 local buffers。
gdb 断点：
```gdb
break FlushBuffer
break FlushLocalBuffer
break XLogFlush
```
观察目标：
ordinary permanent table 的 dirty page 写出会经过 `FlushBuffer()` 并按 `BM_PERMANENT` 触发 `XLogFlush()`。
unlogged ordinary fork 的 shared buffer 可能进入 `FlushBuffer()`，但 `BM_PERMANENT` 不成立时会跳过 `XLogFlush()`。
temporary table 的 local buffer 写出走 `FlushLocalBuffer()`。
`FlushLocalBuffer()` 没有 WAL flush。
注意：
unlogged relation 的 `init` fork 是特殊边界。
buffer 分配时 `forkNum == INIT_FORKNUM` 会设置 `BM_PERMANENT`。
这不是普通 unlogged data fork。

## 31. 讨论题

1. 为什么 `MarkBufferDirty()` 和 `PageSetLSN()` 必须是两个不同层次的状态，而不是合并成一个“page 已经 WAL-protected”的标志？
2. `FlushBuffer()` 为什么是 WAL-before-data 的最后防线？checkpoint、bgwriter 和 backend eviction 分别怎样到达这个 gate？
3. 如果前台读路径复用到 dirty victim，为什么它可能被 WAL flush 和 data page writeback 拖慢？
4. permanent、unlogged ordinary fork、unlogged init fork 和 temp relation 的写出边界分别在哪里？
5. fake LSN 为什么不能交给 `XLogFlush()`？源码中哪个检查保护这个边界？
6. `pg_stat_bgwriter`、wait event 和 `pg_stat_wal` 各能看到什么，哪些 page-level LSN 关系仍然只能靠断点或源码推断？
7. 如果一个 access method 修改 page 后忘记设置 page LSN，后续 crash recovery 和 dirty page flush 会分别遇到什么风险？
8. 本节的可迁移规律是什么：怎样用“对象内版本号 + 写出前 durable gate”保护异步 writeback？

## 32. 常见误区
误区一：
`MarkBufferDirty()` 会设置 page LSN。
不对。
`MarkBufferDirty()` 只设置 `BM_DIRTY`。
page LSN 由 access method 在 `XLogInsert()` 后设置。
误区二：
WAL-before-data 要求每次 page 修改都立即 flush WAL。
不对。
它只要求 data page 写到 relation 文件前，相关 WAL 已经持久化。
误区三：
checkpoint 是唯一写 dirty page 的进程。
不对。
checkpoint、bgwriter、backend eviction、显式 flush 都可能写出 dirty page。
误区四：
unlogged relation 完全没有 LSN。
不准确。
某些 unlogged index page 可以有 fake LSN。
但 fake LSN 不能用于 WAL flush。
误区五：
page LSN 只用于 recovery。
不完整。
它主要服务 crash recovery 和 WAL-before-data。
但一些 index AM 也把 LSN 用作并发修改检测。
误区六：
permanent relation 一定每次修改都 `RelationNeedsWAL()`。
不完整。
`wal_level=minimal` 下当前事务新建或截断的 relfilenode 可能走 WAL-skipping 规则。
这种路径依赖 commit 前写并 fsync affected blocks。

## 33. 本节小结
本节的核心不变量是：
永久数据页落盘时，WAL 必须已经 flush 到该页的 page LSN。
`PageGetLSN()` / `PageSetLSN()` 只是 page header accessor。
它们读写 `pd_lsn`。
`XLogInsert()` 返回 record end pointer。
这个返回值就是普通 WAL-logged page 修改应该写入 page header 的 LSN。
`MarkBufferDirty()` 只设置 buffer descriptor 上的 dirty state。
它不生成 WAL。
它不设置 page LSN。
它也不持久化数据。
调用方必须先把 buffer 标 dirty，再插入 WAL，再设置 page LSN。
这个顺序服务 checkpoint 的 dirty page 判断。
`FlushBuffer()` 是 WAL-before-data 的统一写出 gate。
它读取 page LSN。
如果 buffer 是 `BM_PERMANENT`，就先 `XLogFlush(page_lsn)`。
然后才 `smgrwrite()` 数据页。
checkpoint、bgwriter、backend eviction 和显式 flush 最终都收敛到这个 gate。
permanent、unlogged、temporary relation 的边界不能混淆。
permanent buffer 需要 WAL-before-data。
unlogged 普通数据 fork crash 后会丢弃，可能有 fake LSN，但不能拿 fake LSN flush WAL。
temporary relation 使用 local buffers，不参与 shared buffer 的 crash recovery 协议。
如果你调试脏页写出、checkpoint 延迟、backend eviction 卡顿或疑似 WAL flush 等待，先问三个问题：
这个 buffer 是否 `BM_PERMANENT`？
这个 page 的 `PageGetLSN()` 是多少？
当前 WAL flush position 是否已经覆盖它？
这三个问题通常能把现象直接拉回源码。
