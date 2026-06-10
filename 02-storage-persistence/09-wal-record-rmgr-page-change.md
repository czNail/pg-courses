# PostgreSQL WAL record 结构、rmgr 与 page 修改描述

## 课程定位

本节主题：PostgreSQL 如何把一次 page 修改描述成一条 WAL record。
前置知识：已理解 shared buffer 中的 page 修改、WAL-before-data 的目标，以及 crash recovery 需要按 WAL 重建数据页。
本节唯一主问题：一次 page 修改怎样被编码成既能由通用 WAL 层保存、又能由 rmgr 在 redo 侧精确解释的 record？
本节核心矛盾：WAL record 格式必须足够通用，才能承载 heap、btree 等不同模块；但 redo 又必须足够具体，才能恢复每个 page 的真实状态。
本节主流程：access method 登记 buffer、main data 和 block data -> `XLogRecordAssemble()` 决定 FPI 与 record 布局 -> `XLogInsert()` 返回 end LSN -> recovery 侧按 rmgr 解码并 redo。
错误路径 / 异常路径主要发生在 record 构造协议错误、block reference 不一致、FPI 与 delta 边界错误，以及 recovery decode 发现 CRC、长度或 rmgr 信息不合法时。
观测与诊断入口是 `pg_waldump` 的 rmgr/info/block references、WAL record LSN、checkpoint 后 FPI 变化，以及对 `XLogRecordAssemble()` 和 rmgr redo 入口的断点。
这里的“描述”不是把页面前后两个版本做通用 byte diff。
它更像一份给 crash recovery 的可执行说明。
这份说明由通用 WAL record 容器承载，再交给具体 resource manager，也就是 rmgr，解释和重放。
本节只做一件事：读懂 WAL record 从构造到 redo 的最小闭环。
读完本节，你应该能回答：

- `XLogBeginInsert()` 到 `XLogInsert()` 中间到底登记了什么。
- `XLogRegisterBuffer()` 和 `XLogRegisterData()` 的语义边界是什么。
- block reference、block data、main data 在 record 中怎样分工。
- record header 里的 `xl_rmid` 和 `xl_info` 为什么足以把 record 分派给正确 redo 逻辑。
- page 修改为什么要被写成 WAL record，而不是只写“把某页某偏移改成某字节”。
- buffer delta 和 full-page image，也就是 FPI，什么时候互相替代，什么时候必须同时存在。
- redo 例程为什么必须处理“页面已经部分落盘”与“页面从 FPI 恢复”的两种状态。
- heap insert/update 和 btree insert/split 如何使用同一个 WAL record 框架表达完全不同的页面修改。

## 源码基线

源码仓库：`/home/nail/postgres-lab`
基线：`master` = `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`
本节重点阅读：

- `src/backend/access/transam/xloginsert.c`
- `src/backend/access/transam/xlogreader.c`
- `src/backend/access/transam/rmgr.c`
- `src/backend/access/transam/xlog.c`
- `src/backend/access/transam/xlogrecovery.c`
- `src/backend/access/transam/xlogutils.c`
- `src/include/access/xlogrecord.h`
- `src/include/access/xlog_internal.h`
- `src/include/access/xlogreader.h`
- `src/include/access/xloginsert.h`
- `src/include/access/xlogutils.h`
- `src/include/access/rmgr.h`
- `src/include/access/rmgrlist.h`
- `src/include/access/heapam_xlog.h`
- `src/include/access/nbtxlog.h`
- `src/backend/access/heap/heapam.c`
- `src/backend/access/heap/heapam_xlog.c`
- `src/backend/access/nbtree/nbtinsert.c`
- `src/backend/access/nbtree/nbtxlog.c`
注意一个版本差异：
这个基线里没有 `src/backend/access/transam/xlogrecord.c`。
record 格式定义在 `xlogrecord.h`。
record 读取和 decode 逻辑在 `xlogreader.c`。
所以本节在讲“读取侧 record 解析”时，实际对应的是 `xlogreader.c`。
辅助核对：

- `src/backend/access/transam/README`

---

## 1. 先给结论

一次持久化 page 修改通常会生成一条 WAL record。
这条 record 至少要回答三个问题。
第一，redo 时要找哪些 page。
这由 block reference 表达。
一个 block reference 包含 relation locator、fork number、block number，以及这个 block 在 record 内的编号。
第二，redo 时要执行哪类动作。
这由 record header 里的 `xl_rmid` 和 `xl_info` 表达。

`xl_rmid` 选择 rmgr。

`xl_info` 在该 rmgr 内选择具体 record 类型和少量标志。
第三，redo 时需要哪些参数和 payload。
这由 main data 和 per-block data 表达。
main data 是整条 record 的 rmgr-specific 数据。
per-block data 是绑定到某个 block reference 的 rmgr-specific 数据。
例如 heap insert 的目标页是 block 0。

`xl_heap_insert` 小结构进 main data。
tuple header 的必要字段和 tuple data 进 block 0 的 data。
redo 侧先按 `RM_HEAP_ID` 进入 `heap_redo()`，再按 `XLOG_HEAP_INSERT` 进入 `heap_xlog_insert()`，最后从 block 0 的 data 拼回 tuple 并插入页面。
如果这次 record 带了 full-page image，情况会变化。
FPI 是某个 block 的整页镜像，主要用来抵御 torn page。
当 FPI 足以恢复目标页时，同一个 block 上原本登记的 per-block delta 通常会被省略。
但 main data 不会因为 FPI 自动省略。
有些场景还会通过 `REGBUF_KEEP_DATA` 明确要求：即使有 FPI，也保留 block data。
这是 logical decoding 等消费者需要从 WAL 直接看到 tuple 数据时会用到的边界。
所以 WAL record 不是单纯的“逻辑行变更”。
它也不是通用二进制 patch。
它是一个通用物理容器，加上 rmgr 自己定义的 redo 语义。

---

## 2. 从调用序列看一条 record 怎样产生

WAL 插入路径的入口在 `xloginsert.c`。
文件开头的注释已经给出基本过程。
构造一条 WAL record 先调用 `XLogBeginInsert()`。
然后调用若干 `XLogRegister*()`。
这些登记的数据先放在 backend-private 的工作区。
最后 `XLogInsert()` 把工作区组装成 `XLogRecData` 链并插入 WAL。
常见形态如下：
```c
XLogBeginInsert();
XLogRegisterBuffer(0, buffer, REGBUF_STANDARD);
XLogRegisterData(&xlrec, SizeOfSomething);
XLogRegisterBufData(0, payload, payload_len);
recptr = XLogInsert(RM_SOME_ID, XLOG_SOME_ACTION);
PageSetLSN(page, recptr);
```
这段伪代码里有两条边界很重要。

`XLogRegisterData()` 登记的是 main data。
redo 侧通过 `XLogRecGetData()` 读取。

`XLogRegisterBufData()` 登记的是某个 block reference 的 data。
redo 侧通过 `XLogRecGetBlockData(record, block_id, &len)` 读取。

`XLogRegisterBuffer()` 登记的是 page 身份和页面内容指针。
它不是把 page data 立即复制进 WAL。
真正是否包含 FPI，要到 `XLogRecordAssemble()` 里根据 redo pointer、`full_page_writes`、page LSN 和 flags 决定。

`XLogInsert()` 返回的是这条 record 的 end LSN。
调用者通常把它写回所有被修改页面的 page LSN。
这就是后续写脏页时判断“对应 WAL 是否已经 flush”的依据。

---

## 3. `XLogBeginInsert()`：开启一次构造

`XLogBeginInsert()` 做的事很少，但它建立了构造 record 的状态边界。
它断言当前没有已经登记的 block 和 main data。
它检查 `XLogInsertAllowed()`。
如果当前处于 recovery，不能产生新的普通 WAL record。
它还检查不能嵌套调用。
也就是说，一个 backend 在同一时刻只能有一个正在构造的 WAL record。
这和 `registered_buffers`、`mainrdata_head`、`rdatas` 这些 backend-private 全局工作区相匹配。
调用 `XLogBeginInsert()` 之后，后续 `XLogRegisterBuffer()`、`XLogRegisterData()`、`XLogRegisterBufData()` 都会假定这个状态存在。
如果某个路径决定不再插入 record，可以调用 `XLogResetInsertion()` 清理工作区。
常规路径则由 `XLogInsert()` 在结束时自动 reset。
这里不要把 `XLogBeginInsert()` 理解成“开始写 WAL buffer”。
它还没有占用 WAL 字节位置。
它只是开始收集 record 的描述材料。
真正保留 WAL 空间发生在后面的 `XLogInsertRecord()`。

---

## 4. `XLogRegisterBuffer()`：给 page 修改命名

`XLogRegisterBuffer(uint8 block_id, Buffer buffer, uint8 flags)` 登记一页。
源码注释直接说：每个被 WAL-logged operation 修改的 page 都必须调用它。
这个要求不是为了让 WAL 变好看。
它是为了 crash safety。
如果修改了某个 buffer，却没有把它登记成 block reference，redo 侧就无法可靠定位和修复这个页面。
更糟的是，FPI 逻辑也无从知道这个页面是否需要整页镜像来防 torn page。

`XLogRegisterBuffer()` 会从 buffer tag 取出：

- `RelFileLocator`
- `ForkNumber`
- `BlockNumber`
它还保存当前 `Page` 指针。
这些信息后面会进入 `XLogRecordBlockHeader` 后面的 block tag 部分。
默认情况下，assert 版本会要求这个 buffer 已经 dirty，并且调用者持有 exclusive 或 share-exclusive content lock。
少数路径可以传 `REGBUF_NO_CHANGE`，表示有意登记 clean buffer。
这再次说明 WAL 不是自由拼装的数据包。
它和 buffer 修改协议绑定。
典型顺序是：修改 page，`MarkBufferDirty()`，登记 buffer 和 payload，`XLogInsert()`，再 `PageSetLSN()`。
在很多实际代码里，修改和 WAL 生成处于 critical section。
失败时宁可 PANIC，也不能留下“page 已改但 WAL 没有完整生成”的中间状态。

---

## 5. block id 是 rmgr 内部约定

`block_id` 是 record 内的编号。
它不是 block number。
block number 是 relation fork 内的物理页号。
block id 是这条 WAL record 里“第几个页面引用”。

`xlogrecord.h` 里定义 `XLR_MAX_BLOCK_ID` 为 32。
也就是说 block reference 编号范围是 0 到 32。

`xloginsert.h` 里的普通工作区默认只准备到 `XLR_NORMAL_MAX_BLOCK_ID`，也就是最高 block id 4。
多数 record 足够用了。
需要更多 block reference 的路径要先调用 `XLogEnsureRecordSpace()`。
redo 例程和生成 record 的代码必须约定每个 block id 的含义。
heap update 约定 block 0 是 new page。
如果 old page 不同，则 block 1 是 old page。
btree split 约定 block 0 是 original/left page，block 1 是 new right page，block 2 可能是右兄弟，block 3 可能是 child left sibling。
这些含义不写在通用 WAL 层。
它们属于 rmgr 的 record contract。

---

## 6. `XLogRegisterData()`：main data

`XLogRegisterData(const void *data, uint32 len)` 把数据追加到 main data 链。
源码注释说，这些数据会在 replay 时通过 `XLogRecGetData()` 取得。
如果调用多次，redo 侧看到的是一个连续 main data chunk。
main data 不和某个 page 绑定。
它表达整条 action 的参数。
例如：

- heap insert 的 `xl_heap_insert`。
- heap update 的 `xl_heap_update`。
- btree insert 的 `xl_btree_insert`。
- btree split 的 `xl_btree_split`。
main data 会一直进入 record。
它不会因为某个 block 带了 FPI 而自动省略。
这点和 block data 不同。
因此适合放“无论是否 FPI，redo 或其他 WAL 消费者都需要知道”的记录级信息。
如果 main data 小于 256 字节，record header 区会使用 short data header。
如果更大，则使用 long data header。
这对应 `XLR_BLOCK_ID_DATA_SHORT` 和 `XLR_BLOCK_ID_DATA_LONG`。
main data 在 record 的 fragment 顺序里按约定放在最后。

`DecodeXLogRecord()` 看到 main data header 后会停止解析 fragment header，转而开始复制 payload。

---

## 7. `XLogRegisterBufData()`：page delta 的主要载体

`XLogRegisterBufData(uint8 block_id, const void *data, uint32 len)` 把数据追加到某个 block reference 上。
这个 block id 必须已经通过 `XLogRegisterBuffer()` 或 `XLogRegisterBlock()` 登记。
同一个 block id 可以多次追加。
redo 侧看到的是该 block 的连续 data chunk。
每个 block 的 data length 存在 `XLogRecordBlockHeader.data_length` 里。
这个字段是 `uint16`。
所以单个 block 上登记的 block data 上限是 65535 字节。
源码注释里也解释了原因。
如果重建 page 修改需要超过 `BLCKSZ` 量级的数据，通常就该考虑记录 full page image，而不是继续堆 delta。
block data 常常是 page delta。
但这里的 delta 是 rmgr 自己定义的 payload，不是通用 diff。
heap insert 的 block 0 data 是 tuple header 必要字段和 tuple bytes。
btree insert 的 block 0 data 是 index tuple。
btree split 的 block 1 data 是右页需要恢复的 tuples。
redo 侧知道这些 bytes 的格式，是因为 `xl_rmid/xl_info` 选择了对应 redo 分支。
通用 WAL 层只负责保存、decode、对齐和交付。
它不理解这些 bytes 的业务含义。

---

## 8. `XLogInsert()`：组装并插入

`XLogInsert(RmgrId rmid, uint8 info)` 是构造路径的结束点。
它要求之前已经调用 `XLogBeginInsert()`。
它检查 `info` 里只有允许的 bit 被调用者设置。
调用者能设置 rmgr 高 4 位信息，以及少数允许传入的通用 flag。
其余低位由 WAL 层内部使用。
然后它进入一个 loop。
每次循环先读取当前 full-page-write 决策需要的信息。
也就是 `RedoRecPtr` 和 `doPageWrites`。
然后调用 `XLogRecordAssemble()` 组装 record。
最后调用 `XLogInsertRecord()` 真正保留 WAL 空间并复制 record。
为什么需要 loop？

因为组装 record 时还没有拿 WAL insertion lock。
在这段时间里 checkpoint 可能推进，`RedoRecPtr` 可能变化，`full_page_writes` 也可能变化。

`XLogInsertRecord()` 拿锁后会复查。
如果发现某些本来没带 FPI 的 page 现在必须带 FPI，就返回 invalid LSN，让 `XLogInsert()` 重新组装。
成功后 `XLogInsert()` reset 构造工作区，并返回 record end LSN。
这条 end LSN 通常用于被修改 page 的 `PageSetLSN()`。

---

## 9. record 总体布局

`xlogrecord.h` 给出 WAL record 的通用布局。
顺序是：

1. 固定大小的 `XLogRecord` header。
2. 零个或多个 `XLogRecordBlockHeader`。
3. 可选的 origin 和 top-level xid fragment。
4. 可选的 main data header。
5. block data payload。
6. main data payload。
准确说，block header 和 data header 位于 record 的 header area。
实际 block data 和 main data 位于后面的 payload area。
通用 header 后的结构不保证自然对齐。
所以 decode 时会复制到 aligned 的 `DecodedXLogRecord` 空间。

`XLogRecord` 固定头包含：

- `xl_tot_len`
- `xl_xid`
- `xl_prev`
- `xl_info`
- `xl_rmid`
- `xl_crc`

`xl_tot_len` 是整条 record 长度。

`xl_xid` 是当前 transaction id，如果有。

`xl_prev` 指向前一条 WAL record。

`xl_info` 是 record 类型和 flag。

`xl_rmid` 是 resource manager id。

`xl_crc` 保护 record 内容。

`XLogRecordAssemble()` 会先计算不包含 fixed header 的 CRC 部分，并把 `xl_prev` 临时置为 invalid。

`XLogInsertRecord()` 真正保留 WAL 位置后才知道前一条 record 的位置。
它填好 `xl_prev` 后，再把 fixed header 纳入 CRC，最终写入 `xl_crc`。

---

## 10. block reference 的物理格式

每个 block reference 以 `XLogRecordBlockHeader` 开始。
它包含：

- `id`
- `fork_flags`
- `data_length`

`id` 是 block id。

`fork_flags` 的低 4 位保存 fork number。
高 4 位保存 block-level flags。
这些 flag 包括：

- `BKPBLOCK_HAS_IMAGE`
- `BKPBLOCK_HAS_DATA`
- `BKPBLOCK_WILL_INIT`
- `BKPBLOCK_SAME_REL`

`data_length` 是该 block 的 rmgr-specific block data 长度。
它不包括 FPI。
它也不包括 block header 本身。
如果 `BKPBLOCK_HAS_IMAGE` 被设置，后面跟 `XLogRecordBlockImageHeader`。
image header 包含：

- page image 长度。
- page hole offset。
- image info flags。
如果 page image 压缩且存在 hole，还会跟 `XLogRecordBlockCompressHeader`，保存 hole length。
如果 `BKPBLOCK_SAME_REL` 没设置，block header 后还会跟 `RelFileLocator`。
然后一定跟 `BlockNumber`。
所以一个 block reference 既描述“去哪找页”，也描述“这条 record 有没有这个页的 image 和 data”。

---

## 11. main data 的物理格式

main data 没有 block tag。
它只需要一个特殊 fragment id 和长度。
如果 main data length 小于 256，使用 `XLR_BLOCK_ID_DATA_SHORT`。
格式相当于：

- id = 255。
- 1 字节长度。
如果 main data length 至少 256，使用 `XLR_BLOCK_ID_DATA_LONG`。
格式相当于：

- id = 254。
- 4 字节长度。
这两个 struct 在 `xlogrecord.h` 中更多是文档化定义。
实际组装时，`xloginsert.c` 直接向 scratch buffer 写 id 和 length。
decode 时，`xlogreader.c` 也按 id 分支解析。
main data header 出现后，decode 按约定认为 header 部分结束。
剩余 payload 先按前面 block headers 记录的长度分配给 block image 和 block data。
最后的 payload 才是 main data。
这就是为什么 `XLogRecordAssemble()` 要先把所有 block header 写完，最后写 main data header。

---

## 12. `XLogRecordAssemble()` 做了什么

`XLogRecordAssemble()` 是本节最值得慢读的函数。
它把之前登记的 buffer、main data、block data 变成最终 `XLogRecData` 链。
第一步，它在 `hdr_scratch` 里预留 `XLogRecord` 固定头。
第二步，它遍历所有已登记 block。
对每个 block，它决定是否需要 FPI。
第三步，它决定是否需要包含该 block 的 block data。
第四步，它把 block header、image header、locator、block number 写入 scratch。
第五步，它把对应的 page image data 或 block data 链接到 `XLogRecData` 链。
第六步，它追加 origin、top-level xid、main data header 和 main data。
最后，它计算 record data 部分 CRC，并填充 `XLogRecord` 的基本字段。
此时 `xl_prev` 还没有最终值。
真正的 WAL byte position 还没有分配。
所以它返回的是“可插入的 rdata 链”，不是已经落入 WAL buffer 的 record。

---

## 13. FPI 决策：最核心的边界

FPI 决策在 `XLogRecordAssemble()` 的 block 循环中完成。
优先级大致如下。
如果 block flags 有 `REGBUF_FORCE_IMAGE`，需要 FPI。
如果有 `REGBUF_NO_IMAGE`，不需要 FPI。
如果当前不需要 page writes，也不需要 FPI。
否则读取 page LSN。
如果 `PageGetLSN(page) <= RedoRecPtr`，需要 FPI。
这个判断的含义是：这页在当前 checkpoint 周期内还没有被 WAL-protected 地修改过。
如果现在只写 delta，崩溃时磁盘上可能留下 torn page。
redo 从 checkpoint redo pointer 开始重放，可能没有一份完整页面可恢复。
所以第一次修改必须带整页镜像。
如果 page LSN 大于 `RedoRecPtr`，说明该页在本 checkpoint 周期已有更早 WAL record 保护。
这次可以只写 delta。

`XLogRecordAssemble()` 会记录这些没有 FPI 的 page 中最小的 page LSN 到 `fpw_lsn`。

`XLogInsertRecord()` 拿 insertion lock 后会复查 `RedoRecPtr` 和 full-page-write 状态。
如果 checkpoint 刚好推进导致某个 delta-only page 现在需要 FPI，它会让上层重试。

---

## 14. FPI 不是“更详细的 delta”

FPI 和 block data 的角色不同。
FPI 是页面镜像。
它用于把某个 block 恢复到 record 生成时的页面状态，或者用于 WAL consistency checking。
block data 是 rmgr-specific delta。
它用于让 redo 逻辑执行某个修改步骤。
默认规则是：如果某个 block 需要 backup image，并且调用者没有 `REGBUF_KEEP_DATA`，那么该 block 的 block data 不进入 record。
原因很直接。
既然 FPI 已经能把这个 block 恢复到修改后的完整状态，再记录同一 block 的 delta 通常是重复的。
但这只是默认。

`REGBUF_KEEP_DATA` 会强制保留 block data。
heap insert 在 logical logging 需要新 tuple 时就可能这么做。
还有一种特殊情况是 `XLR_CHECK_CONSISTENCY`。
如果 WAL consistency checking 对某个 rmgr 开启，record 会带 full-page image 用于 redo 后校验。
但这类 image 不一定设置 `BKPIMAGE_APPLY`。
redo 时只有设置 `BKPIMAGE_APPLY` 的 image 才会被当成页面恢复来源。
校验用 image 则用于比较 redo 后页面是否符合预期。

---

## 15. standard page、hole 和 compression
PostgreSQL page 常有一段 unused hole。
标准 page layout 中，`pd_lower` 和 `pd_upper` 之间是空洞。
如果调用者给 block reference 传 `REGBUF_STANDARD`，FPI 生成时可以跳过这段 hole。

`XLogRecordBlockImageHeader` 记录 hole offset。
未压缩情况下，hole length 可以通过 `BLCKSZ - bimg.length` 推出。
压缩情况下，长度不能这样推，额外的 compress header 会保存 hole length。
如果 `wal_compression` 不是 none，`XLogCompressBackupBlock()` 会尝试用 pglz、lz4 或 zstd 压缩去掉 hole 后的 image。
只有压缩结果加上额外 header 仍然真正省空间时，才采用压缩版本。
否则仍写未压缩 image。
这说明 FPI 不是一定等于 `BLCKSZ` 字节。
它是“足以恢复一页”的 image payload。
可能去掉了标准页 hole。
也可能被 WAL compression 压缩。
redo 侧 `RestoreBlockImage()` 会按 image header 解压、补零 hole，再恢复出完整 `BLCKSZ` page。

---

## 16. `REGBUF_WILL_INIT` 的含义

`REGBUF_WILL_INIT` 表示 redo 会重新初始化这个 page。
它隐含 `REGBUF_NO_IMAGE`。
也就是说，它会抑制 FPI。
这听起来危险，但前提是 redo 例程不能依赖旧 page 内容。
redo 必须用 WAL record 中的信息从零页或新初始化页面重建目标状态。

`xlogutils.c` 在 `XLogReadBufferForRedoExtended()` 中会检查这件事。
如果 block header 标记了 `BKPBLOCK_WILL_INIT`，redo 调用者必须使用 zero mode。
如果 redo 调用了 zero mode，record 也必须标记 `WILL_INIT`。
不一致会 PANIC。
heap insert 的 first tuple on page 可以使用这个模式。
btree split 的 new right page 也使用这个模式。
这类 record 不需要旧页面，也不需要 FPI。
因为 redo 会把页面从头构造出来。
从 correctness 看，重新初始化页面能像 FPI 一样避开 torn page 依赖。

---

## 17. rmgr 是 record 语义的入口

`RmgrId` 在 `rmgr.h` 里是 `uint8`。
内置 rmgr 的数值由 `rmgrlist.h` 的顺序决定。

`rmgrlist.h` 明确提醒：顺序定义了每个 rmgr ID 的数值，而这个数值存储在 WAL record 中。
新增条目应追加到末尾。
修改这张表可能需要 bump `XLOG_PAGE_MAGIC`。

`rmgrlist.h` 同时服务两个用途。
在 `rmgr.h` 中，调用者把 `PG_RMGR` 定义成只展开 symbol name，于是得到 enum。
在 `rmgr.c` 中，调用者把 `PG_RMGR` 定义成函数表 entry，于是得到 `RmgrTable`。

`RmgrData` 在 `xlog_internal.h` 中定义。
它包含：

- `rm_name`
- `rm_redo`
- `rm_desc`
- `rm_identify`
- `rm_startup`
- `rm_cleanup`
- `rm_mask`
- `rm_decode`
这就是 WAL record 的动态分派表。
record header 中的 `xl_rmid` 是表下标。
recovery 过程中，`xlogrecovery.c` 调用 `GetRmgr(record->xl_rmid).rm_redo(xlogreader)`。
之后就进入具体 rmgr 的 redo 逻辑。

---

## 18. `xl_info` 是 rmgr 内部 opcode

`xlogrecord.h` 把 `xl_info` 分成两部分。
低 4 位由 WAL 通用层使用。
高 4 位给 rmgr 自由使用。
宏上体现为：

- `XLR_INFO_MASK`
- `XLR_RMGR_INFO_MASK`

`XLogInsert()` 会检查调用者没有乱设置保留 bit。
redo 侧通常会先清掉通用低位。
heap redo 中：
```c
info = XLogRecGetInfo(record) & ~XLR_INFO_MASK;
```
然后用 `info & XLOG_HEAP_OPMASK` 分支。
btree redo 中也做类似处理，然后直接 switch 具体 btree opcode。
所以一条 record 的语义由两个维度确定。

`xl_rmid = RM_HEAP_ID` 表示交给 heap rmgr。

`xl_info = XLOG_HEAP_INSERT` 表示 heap insert record。

`xl_rmid = RM_BTREE_ID` 表示交给 btree rmgr。

`xl_info = XLOG_BTREE_SPLIT_L` 表示 btree split 且新 item 在左页。
同样的 `info` bit pattern 在不同 rmgr 下可以有不同含义。
通用 WAL 层不尝试解释它。

---

## 19. record decode：从字节流回到结构

这个基线里，decode 逻辑在 `xlogreader.c`。

`DecodeXLogRecord()` 从 fixed header 后开始读 fragment headers。
它循环读取 1 字节 `block_id`。
如果是 `XLR_BLOCK_ID_DATA_SHORT`，读取 1 字节 main data length，然后结束 header 解析。
如果是 `XLR_BLOCK_ID_DATA_LONG`，读取 4 字节 main data length，然后结束 header 解析。
如果是 origin 或 top-level xid，读取对应 metadata。
如果 `block_id <= XLR_MAX_BLOCK_ID`，按 block header 解析。
decode 会检查 block id 递增。
它会检查 `BKPBLOCK_HAS_DATA` 和 `data_length` 是否一致。
它会检查 image header 中 hole、compressed、length 的组合是否合法。
它还会处理 `BKPBLOCK_SAME_REL`，复用上一个 relation locator。
解析完 headers 后，它要求剩余 payload 字节数正好等于所有 block image、block data、main data 长度之和。
之后它把 block data 和 main data 复制到 aligned 内存中。
这样 redo 代码可以把 `XLogRecGetData()` 返回值 cast 成自己的 `xl_*` struct。
这个 decode 过程再次说明：通用 WAL 层只验证通用格式。
它不知道某个 heap tuple payload 的内部字段是否符合 heap 语义。
那是 rmgr redo 的责任。

---

## 20. redo contract 初步

redo 例程的基本契约是：给定一条已经通过 CRC 和通用格式检查的 WAL record，把 data directory 推进到至少包含这条 record 效果的状态。
它必须能处理 page 已经包含该修改的情况。
因为崩溃前，某些 data page 可能已经落盘。
它也必须能处理 page 只写了一半的情况。
这就是 FPI 存在的原因。

`XLogReadBufferForRedo()` 是大多数 rmgr redo 的入口辅助函数。
它根据 record 的 block reference 找到目标 page。
如果 block 带有应当 apply 的 FPI，它会 restore image、设置 page LSN、mark dirty，并返回 `BLK_RESTORED`。
如果没有 apply image，它会读取目标 page 并比较 page LSN。
如果 `record->EndRecPtr <= PageGetLSN(page)`，返回 `BLK_DONE`。
这表示该 record 的修改已经在 page 上。
否则返回 `BLK_NEEDS_REDO`。
redo 例程只有在 `BLK_NEEDS_REDO` 时才真正套用 rmgr-specific delta。
套用后要 `PageSetLSN(page, record->EndRecPtr)`，再 `MarkBufferDirty()`。
这让 redo 幂等。
重复看到同一条 record 时，page LSN 会挡住重复应用。

---

## 21. FPI 与 redo 的关系

FPI 参与 redo 时，有一个常被忽略的细节。

`XLogReadBufferForRedo()` 注释说：如果 record 有带 `BKPIMAGE_APPLY` 的 backup block，就 restore 它，即使数据库页面看起来更新。
原因是防 torn page。
数据库页面可能在崩溃时被部分写坏。
WAL record 已经通过 CRC 检查，可信度更高。
所以有 apply image 时，redo 先相信 WAL 里的完整页面。
这会导致后续该 page 上的 WAL 修改也需要继续重放。
但这是可接受的。
正确性优先于避免重复工作。
如果 record 带的是 consistency-check image，而没有 `BKPIMAGE_APPLY`，redo 不会用它覆盖页面。
它会在 redo 后用于校验。
这就是 image header 里 `BKPIMAGE_APPLY` 单独存在的原因。
不是所有 WAL record 中的 page image 都是 redo 恢复来源。

---

## 22. 为什么 page 修改被描述成 WAL record
page 修改发生在 shared buffers 中。
最终持久化发生在 data file 中。
崩溃可能发生在任意时刻。
如果只依赖 data page 自己落盘，系统无法知道哪些修改已经完整写入，哪些只写了一半，哪些还在 shared buffers 中。
WAL record 给这些 page 修改提供了顺序化、可校验、可重放的描述。
它先于 data page 持久化。
data page 的 page LSN 指向“这页至少包含到哪个 WAL 位置的修改”。
写脏页前，buffer manager 可以要求对应 WAL 已经 flush。
崩溃恢复时，recovery 从 checkpoint redo pointer 开始读 WAL。
每条 record 根据 `xl_rmid/xl_info` 交给 rmgr。
rmgr 再用 block references 找 page，用 main/block data 或 FPI 修复 page。
这就是 page 修改被压成 WAL record 的原因。
WAL record 是 durable order。
data page 是可延迟、可重复构造的物理状态。

---

## 23. record 是 atomic action，不一定是 SQL 语句

一条 WAL record 通常对应一个 storage-level atomic action。
它不一定对应一条 SQL。
一条 SQL 可能修改很多 heap page 和 index page。
它会生成很多 WAL records。
一个复杂 storage action 也可能必须拆成多条 WAL records。

`access/transam/README` 用 btree split 举例。
page split 和 parent insertion 因锁顺序等原因可能分成多个 atomic actions。
中间状态必须自洽。
如果 recovery 在两条 record 之间被中断，系统仍要能继续工作。
btree 的 incomplete split flag 就服务这个目的。
第一条 record 可以完成 child page split，并标记 parent downlink 尚未完成。
第二条 record 插入 parent key 后清除该标志。
这不是 WAL 层通用规则能推出来的。
它是 btree rmgr 和 btree 并发算法共同设计的结果。
通用 WAL record 只是承载这些状态转移。

---

## 24. heap insert：生成侧

heap insert 的生成侧在 `heapam.c`。
普通 insert 在需要 WAL 时会准备 `xl_heap_insert`。
这个结构只包含 inserted tuple 的 offset number 和 flags。
如果插入的是页面上的第一条 tuple，并且页面此前没有其他 tuple，代码会设置 `XLOG_HEAP_INIT_PAGE`。
同时 buffer flags 会加 `REGBUF_WILL_INIT`。
这表示 redo 可以重新初始化整个 page。
之后调用：
```c
XLogBeginInsert();
XLogRegisterData(&xlrec, SizeOfHeapInsert);
XLogRegisterBuffer(0, buffer, REGBUF_STANDARD | bufflags);
XLogRegisterBufData(0, &xlhdr, SizeOfHeapHeader);
XLogRegisterBufData(0, tuple_bytes, tuple_len);
recptr = XLogInsert(RM_HEAP_ID, info);
PageSetLSN(page, recptr);
```
`xl_heap_header` 只保存 tuple header 中必须记录的少数字段。
完整 `HeapTupleHeaderData` 不直接原样进 WAL。
redo 会根据 record 中的 xid、offset、header fields 和 tuple payload 重建。
这就是“描述 page 修改”而不是“复制整个 tuple 内存对象”的一个具体例子。
如果 relation 需要 logical logging，insert 还可能设置 `REGBUF_KEEP_DATA`。
这样即使该 block 被 FPI 覆盖，tuple data 仍保留在 WAL record 中。
logical decoding 需要从 WAL record 读到新 tuple。

---

## 25. heap insert：redo 侧

heap insert 的 redo 侧在 `heapam_xlog.c` 的 `heap_xlog_insert()`。
它先通过 `XLogRecGetData(record)` 取得 `xl_heap_insert`。
然后通过 `XLogRecGetBlockTag(record, 0, ...)` 取得目标 locator 和 block number。
如果 flags 表示 visibility map 需要清理，即使 heap page 已经 up-to-date，也会修 VM。
如果 `XLOG_HEAP_INIT_PAGE` 设置，redo 调用 `XLogInitBufferForRedo(record, 0)`。
然后 `PageInit()` 重新初始化 page。
否则调用 `XLogReadBufferForRedo(record, 0, &buffer)`。
只有返回 `BLK_NEEDS_REDO` 时，才从 block 0 data 读取 tuple payload。
redo 读取 block data 后：

- 先解析 `xl_heap_header`。
- 再复制 tuple data。
- 补回 `xmin`、`cmin`、`ctid` 等字段。
- 调用 `PageAddItem()` 把 tuple 放入页面。
- 设置 page LSN。
- mark buffer dirty。
这个流程和生成侧严格对应。
生成侧把“足以重建 tuple”的信息分散到 main data、block tag、block data、record xid 中。
redo 侧按同样 contract 拼回来。

---

## 26. heap update：同一个 record 里的多个页面

heap update 更能体现 block reference 的价值。
update 可能在同一页生成新版本。
也可能旧 tuple 和新 tuple 在不同页。
生成侧约定 block 0 是 new page。
如果 old page 不同，block 1 是 old page。
main data 使用 `xl_heap_update`。
它保存旧 tuple offset、旧 xmax、infobits、新 tuple offset、新 xmax 和 flags。
block 0 data 保存新 tuple 的可重建 payload。
如果同页更新，且不需要 logical tuple data，且新页不需要 FPI，代码还会尝试 prefix/suffix 优化。
它比较 old tuple 和 new tuple 的 user data 前缀、后缀。
相同的前缀和后缀不写入 WAL。
redo 时从 old tuple 复制回来。
这不是通用压缩。
这是 heap update 自己的 record contract。
它依赖旧 tuple 在同一页，并且 redo 能先看到旧 tuple。
如果 old page 和 new page 不同，或者 logical decoding 需要完整 tuple，或者新页可能有 FPI，这种优化就不适用。

---

## 27. heap update：redo 侧

`heap_xlog_update()` 先解析 main data。
它通过 block 0 得到 new block。
如果有 block 1，就得到 old block。
没有 block 1 时，old block 等于 new block。
redo 先处理旧 tuple。
它调用 `XLogReadBufferForRedo()` 读取 old page。
如果需要 redo，就修改旧 tuple 的 xmax、infomask 和 `t_ctid` forward link。
然后处理新 tuple 所在页。
如果 new page 需要 init，就走 `XLogInitBufferForRedo()`。
否则走 `XLogReadBufferForRedo()`。
当新页需要 redo 时，读取 block 0 data。
如果 flags 表示 prefix/suffix 来自 old tuple，先从 record 里读 prefixlen/suffixlen。
然后读取 `xl_heap_header` 和中间的 tuple data。
最后把前缀和后缀从 old tuple 拼回来。
这里可以看到 WAL record 的信息组织方式很精细。
record 不保存“更新后整页”。
也不保存“SQL 层的 UPDATE 语句”。
它保存 redo 这次 heap page 状态转移需要的最小物理信息。

---

## 28. btree insert：同一容器，不同 rmgr
btree insert 的生成侧在 `nbtinsert.c`。
简单 leaf insert 准备 `xl_btree_insert`。
main data 只需要新 item 的 offset number。
目标 index page 登记为 block 0。
新 index tuple 作为 block 0 data。
调用：
```c
XLogBeginInsert();
XLogRegisterData(&xlrec, SizeOfBtreeInsert);
XLogRegisterBuffer(0, buf, REGBUF_STANDARD);
XLogRegisterBufData(0, itup, IndexTupleSize(itup));
recptr = XLogInsert(RM_BTREE_ID, XLOG_BTREE_INSERT_LEAF);
```
如果是 internal page insert，record 还可能登记 child page 为 block 1。
因为 internal insert 可能完成 child level 的 incomplete split。
如果还要更新 metapage，则登记 metapage 为 block 2，并用 `REGBUF_WILL_INIT | REGBUF_STANDARD`。
可以看到，WAL record 容器相同。
但 block id 含义已经完全不同于 heap。
heap block 0 是 heap page。
btree block 0 是 index page。
block data 的解析也完全由 btree redo 决定。

---

## 29. btree insert：redo 侧

`btree_redo()` 读取 `xl_info`，进入 `btree_xlog_insert()`。
如果不是 leaf insert，它会先清 child page 的 incomplete split flag。
然后对 block 0 调用 `XLogReadBufferForRedo()`。
只有返回 `BLK_NEEDS_REDO`，才读取 block 0 data。
简单 insert 直接 `PageAddItem()` 把 index tuple 放入 page。
posting list split insert 更复杂。
record data 先保存 posting split offset，再保存原始 new item。
redo 用 `_bt_swap_posting()` 重做 posting list split。
它先替换页面中原有 posting tuple，再插入最终 new item。
这个例子再次说明：block data 是 rmgr-defined delta。
通用 WAL 层不知道 `postingoff` 是什么。
只有 btree redo 知道如何解释这些 bytes。

---

## 30. btree split：多 block reference 的代表

btree split 是本节最适合反复阅读的例子。
生成侧在 `nbtinsert.c` 中准备 `xl_btree_split`。
main data 包含：

- page level。
- right page 第一个 item offset。
- new item offset。
- posting split offset。
block 0 是 original page，也就是 split 后的 left page。
block 1 是 new right page。
block 1 用 `REGBUF_WILL_INIT`。
可选 block 2 是原来的 right sibling，因为它的 left link 要改。
可选 block 3 是 child page，用于 internal split 时清 incomplete split。
left page 的 block data 中可能保存 new item 和新的 high key。
right page 的 block data 中保存右页 tuples，redo 会用 `_bt_restore_page()` 重建右页。
生成侧没有把 right page 作为普通 delta 处理。
它选择记录足以从头恢复右页的 tuple payload。
这样通常比让 WAL 层认为新右页需要整页 FPI 更省。
这是一种 rmgr-level 的 WAL 体积优化。

---

## 31. btree split：redo 侧

`btree_xlog_split()` 先取 block 0 和 block 1 的 block number。
如果 block 2 不存在，说明没有右兄弟。
如果是 internal page split，还会先清 block 3 上的 incomplete split。
然后 redo 重建 right page。
它对 block 1 调用 `XLogInitBufferForRedo()`。
这要求 record 中 block 1 有 `WILL_INIT`。
redo 初始化 btree page opaque fields。
再从 block 1 data 读取 tuple payload，调用 `_bt_restore_page()` 恢复页面内容。
随后处理 original/left page。
如果 block 0 需要 redo，代码创建临时 left page。
它按 item number 顺序把 high key、新 item、旧 item 或 posting replacement 加进去。
最后用 `PageRestoreTempPage()` 替换原页。
如果有 block 2，则修复右兄弟的 `btpo_prev`。
这些 buffer 最后一起 release，避免 hot standby 读者观察到不一致状态。
这就是一个 record 同时描述多个 page 修改的例子。

---

## 32. record header 和 rmgr 示例对应关系

以 heap insert 为例。

`xl_rmid` 是 `RM_HEAP_ID`。

`xl_info` 的 rmgr bits 是 `XLOG_HEAP_INSERT`，可能再带 `XLOG_HEAP_INIT_PAGE`。

`xl_xid` 来自当前 transaction id。
block 0 指向目标 heap page。
main data 是 `xl_heap_insert`。
block 0 data 是 `xl_heap_header` 和 tuple payload。
以 btree split 为例。

`xl_rmid` 是 `RM_BTREE_ID`。

`xl_info` 是 `XLOG_BTREE_SPLIT_L` 或 `XLOG_BTREE_SPLIT_R`。
main data 是 `xl_btree_split`。
block 0 指向 left/original page。
block 1 指向 new right page，并标记 `WILL_INIT`。
block 2 和 block 3 视情况存在。
这两条 record 的通用 header 结构完全一样。
差异都藏在 rmgr id、info opcode、main data struct 和 block id 约定中。
这正是 WAL record 设计的扩展点。

---

## 33. `XLogInsertRecord()` 和 WAL 字节位置

`XLogInsert()` 组装完 record 后，会调用 `XLogInsertRecord()`。
这个函数位于 `xlog.c`。
它负责把 record 插入 WAL buffers。
它先分类特殊 record，例如 xlog switch 和 checkpoint。
普通 record 需要拿 WAL insertion lock。
拿锁后，它会复查 `RedoRecPtr` 和 `fullPageWrites`。
如果组装 record 时的 FPI 决策已经过期，就释放锁并要求重试。
如果可以继续，它调用 `ReserveXLogInsertLocation()` 保留 WAL 空间。
这个过程也填充 `rechdr->xl_prev`。
接着它把 header 纳入 CRC 计算，写回 `xl_crc`。
最后调用 `CopyXLogRecordToWAL()` 把 `XLogRecData` 链复制到已保留的 WAL 位置。
所以 `XLogRecordAssemble()` 和 `XLogInsertRecord()` 分工明确。
前者构造 record 内容。
后者把内容放进 WAL 字节流，并补上只有插入时才知道的链式位置和最终 CRC。

---

## 34. buffer delta 与 FPI 的边界总结

block data 是 delta。
FPI 是 image。
main data 是 record-level 参数。
默认情况下：

- main data 总是保留。
- block data 只有在该 block 没有被 FPI 替代时保留。
- `REGBUF_KEEP_DATA` 可以让 block data 即使有 FPI 也保留。
- `REGBUF_FORCE_IMAGE` 强制 FPI。
- `REGBUF_NO_IMAGE` 禁止 FPI。
- `REGBUF_WILL_INIT` 禁止 FPI，但 redo 必须从头初始化 page。
- `REGBUF_STANDARD` 允许 FPI 省略 standard page hole。
FPI 决策由 WAL 层做。
delta 的格式由 rmgr 做。
这条边界非常重要。
调用者不能假设自己登记的 block data 一定会出现在 WAL record 中。
除非使用 `REGBUF_KEEP_DATA`。
redo 例程也不能在 FPI 已经恢复页面的路径上继续无条件读取 block data。
常见模式是先调用 `XLogReadBufferForRedo()`。
只有返回 `BLK_NEEDS_REDO` 时，才读取 block data 并应用 delta。

---

## 35. page LSN 的两个方向

正常执行时，`XLogInsert()` 返回 record end LSN。
调用者把它写入被修改 page。
这表示 page 依赖这条 WAL record。
之后如果要把 dirty page 写出，buffer write 路径必须确保 WAL 至少 flush 到 page LSN。
这是 WAL-before-data。
redo 执行时，`record->EndRecPtr` 也会写入 page LSN。
这表示 recovery 已经把该 record 的效果应用到 page。
下次再遇到相同或更早 record，`XLogReadBufferForRedo()` 能通过比较 page LSN 返回 `BLK_DONE`。
所以 page LSN 同时服务两个阶段。
运行期，它是写脏页前的 WAL flush 依赖。
恢复期，它是 redo 幂等性的进度标记。
FPI 会稍微改变流程。
有 apply image 时，redo 先 restore page，再设置 page LSN。
如果 restored page 是 all-zero new page，不能随便写 LSN，因为会破坏新页语义。
源码里也有对 `PageIsNew()` 的特殊处理。

---

## 36. 成本、资源与常见误区：不要混淆三种“数据”

读 WAL 代码时，最容易混淆三种数据。
从成本和资源角度看，这三种数据也决定了 WAL record 的大小、FPI 是否压过 delta、redo 是否还需要读取原 page。
把它们混在一起，常见结果是高估某个 record 的 WAL 体积，或者误判 redo 为什么没有读取某段 block data。
第一种是 page content。
它是 shared buffer 里真实页面的当前内容。

`XLogRegisterBuffer()` 保存 page pointer，以便组装时可能生成 FPI。
第二种是 block data。
它是调用者通过 `XLogRegisterBufData()` 登记的 rmgr payload。
它绑定某个 block id。
它可能因为 FPI 被省略。
第三种是 main data。
它是调用者通过 `XLogRegisterData()` 登记的 record-level payload。
它不绑定 block。
它不会因为某个 block 有 FPI 被省略。
heap insert 中，page content 是目标 heap page。
block data 是 tuple payload。
main data 是 `xl_heap_insert`。
btree split 中，page content 包括 left page、right page、sibling page。
block data 是 high key、right page tuples 等。
main data 是 split metadata。

---

## 37. 源码跟读练习一：从 insert API 读到 assemble
打开 `src/backend/access/transam/xloginsert.c`。
先读文件头注释。
确认构造 record 的四步：begin、register、assemble、insert。
然后读 `registered_buffer`。
问题：

- 哪些字段来自 buffer tag？
- 哪些字段保存 per-block rdata？
- 为什么有两个 `bkp_rdatas`？
- 压缩页 buffer 放在哪里？

接着读 main data 的全局变量。
问题：

- `mainrdata_head` 和 `mainrdata_last` 怎么维护链表？
- `mainrdata_len` 为什么是 `uint64`？

再读 `XLogBeginInsert()`。
问题：

- 为什么 recovery 中不能普通插 WAL？
- 为什么不能嵌套 begin？

最后读 `XLogResetInsertion()`。
问题：

- reset 时哪些状态会清理？
- 为什么不需要释放登记 data 指针指向的内存？

---

## 38. 源码跟读练习二：buffer 和 data 登记

继续读 `XLogRegisterBuffer()`。
重点看注释中的一句：必须为每个被 WAL-logged operation 修改的 page 调用。
问题：

- assert 版本为什么检查 dirty 和 content lock？
- `REGBUF_NO_CHANGE` 绕过的是什么？
- `BufferGetTag()` 取出的信息后面如何进入 record？
- 为什么同一个 page 不能用两个 block id 重复登记？

然后读 `XLogRegisterData()`。
问题：

- 它如何追加到 main data？
- 多次调用后 redo 侧为什么能看到连续 chunk？

再读 `XLogRegisterBufData()`。
问题：

- 它如何检查 block id 已登记？
- 单 block data 为什么限制在 `UINT16_MAX`？
- 它追加到哪个链表？

最后读 `xloginsert.h` 中的 `REGBUF_*` flags。
把每个 flag 写成一句话。
尤其要区分 `NO_IMAGE`、`WILL_INIT`、`KEEP_DATA`。

---

## 39. 源码跟读练习三：record 格式和 decode
打开 `src/include/access/xlogrecord.h`。
先画出 record layout。
不要跳过开头注释。
问题：

- fixed header 后为什么可以有多个 block header？
- main data header 为什么用特殊 block id 表达？
- record struct 之后为什么没有 padding？

然后读 `XLogRecord`。
问题：

- 哪些字段服务链式读取？
- 哪些字段服务 rmgr 分派？
- 哪些字段服务一致性检查？

再读 `XLogRecordBlockHeader` 和 image header。
问题：

- `data_length` 不包含哪些内容？
- `fork_flags` 的低 4 位和高 4 位分别是什么？
- `BKPBLOCK_SAME_REL` 为什么能省空间？

最后打开 `xlogreader.c` 的 `DecodeXLogRecord()`。
问题：

- 它如何识别 main data short/long？
- 它如何检查 block id 顺序？
- 它为什么把 block data 和 main data 复制到 aligned buffer？

---

## 40. 源码跟读练习四：rmgr 分派

打开 `src/include/access/rmgr.h`。
确认 `RmgrId` 是 `uint8`。
读 enum 的生成方式。
问题：

- 为什么 enum 的实际值由 `rmgrlist.h` 顺序决定？
- 为什么 widening `RmgrId` 会影响 WAL file format？

打开 `src/include/access/rmgrlist.h`。
找到 `RM_HEAP_ID`、`RM_HEAP2_ID`、`RM_BTREE_ID`。
问题：

- 每个 rmgr entry 里有哪些函数？
- `heap_mask`、`btree_mask` 和 consistency checking 有什么关系？

打开 `src/backend/access/transam/rmgr.c`。
看 `PG_RMGR` 宏如何展开成 `RmgrTable`。
再打开 `src/include/access/xlog_internal.h` 的 `RmgrData`。
问题：

- redo、desc、identify 各服务什么场景？
- custom rmgr 为什么只能在指定 id 范围内注册？

最后打开 `xlogrecovery.c`。
找到 `GetRmgr(record->xl_rmid).rm_redo(xlogreader)`。
把“record header 到 rmgr redo”的调用链写下来。

---

## 41. 源码跟读练习五：heap record 示例

打开 `heapam.c` 中普通 insert 的 WAL 生成片段。
问题：

- `xl_heap_insert` 进 main data 还是 block data？
- tuple payload 进 main data 还是 block data？
- 为什么 `xl_heap_header` 只保存 tuple header 的一部分字段？
- 什么情况下设置 `XLOG_HEAP_INIT_PAGE`？
- 什么情况下设置 `REGBUF_KEEP_DATA`？

然后打开 `heapam_xlog.c` 的 `heap_xlog_insert()`。
问题：

- redo 如何取得 block 0 的 block tag？
- 为什么 VM 清理可能在 heap page 已经 up-to-date 时仍要做？
- `XLogInitBufferForRedo()` 和 `XLogReadBufferForRedo()` 分别对应什么生成侧 flags？
- redo 如何补回 tuple 的 `xmin`、`cmin`、`ctid`？

再读 `heap_xlog_update()`。
问题：

- block 0 和 block 1 分别是什么？
- prefix/suffix 优化为什么只适合某些 same-page update？
- redo 为什么要先处理 old tuple，再处理 new tuple？

---

## 42. 源码跟读练习六：btree record 示例

打开 `nbtinsert.c` 的 btree insert WAL 生成片段。
问题：

- simple leaf insert 的 main data 是什么？
- new index tuple 放在哪个 block data？
- internal insert 为什么可能登记 child page？
- meta update 为什么用 `REGBUF_WILL_INIT | REGBUF_STANDARD`？

打开 `nbtxlog.c` 的 `btree_xlog_insert()`。
问题：

- 它如何通过 `XLogReadBufferForRedo()` 避免重复应用？
- posting list split 时，record data 的第一个字段是什么？
- 为什么 redo 要用 `_bt_swap_posting()`？

再打开 btree split 生成和 redo。
问题：

- block 0、1、2、3 分别是什么？
- 为什么 right page 用 `WILL_INIT`？
- right page data 为什么保存 tuples，而不是依赖旧 page？
- redo 为什么要把 left page 复制到 temp page 后按顺序重建？

---

## 43. 实验一：用 `pg_waldump` 看 rmgr 和 info
这个实验观察 record header 层面的 rmgr 分派。
准备一个独立测试实例。
确保能找到 `pg_waldump`。
记录当前 WAL 位置：
```sql
SELECT pg_current_wal_lsn();
```
执行一组很小的写入：
```sql
CREATE TABLE wal_lesson_heap(id int primary key, payload text);
INSERT INTO wal_lesson_heap VALUES (1, 'alpha');
UPDATE wal_lesson_heap SET payload = 'alpha-2' WHERE id = 1;
```
再次记录 WAL 位置：
```sql
SELECT pg_current_wal_lsn();
```
对这段 LSN 范围运行：
```bash
pg_waldump -p "$PGDATA/pg_wal" -s START_LSN -e END_LSN
```
观察输出中的 rmgr。
你应该能看到 Heap、Btree、Transaction 等 rmgr 记录。
重点不是记住每一行。
重点是把输出里的 rmgr 名称和 `rmgrlist.h` 中的 entry 对上。
再观察 description 中的 INSERT、UPDATE、INSERT_LEAF 等词。
这些名字来自各 rmgr 的 identify/desc 逻辑。
它们对应 record header 里的 `xl_info`。

---

## 44. 实验二：观察 checkpoint 后的 FPI
这个实验观察 FPI 边界。
确保 `full_page_writes` 是 on。
```sql
SHOW full_page_writes;
```
执行 checkpoint。
```sql
CHECKPOINT;
SELECT pg_current_wal_lsn();
```
然后修改一个普通表页面。
```sql
INSERT INTO wal_lesson_heap VALUES (2, repeat('x', 100));
```
记录结束 LSN，用 `pg_waldump` 查看这段 WAL。
重点找带有 `FPW` 或 block image 信息的记录。
现象可能因页面状态、tuple 大小、是否新页、索引维护等而不同。
如果 insert 命中新初始化页面，heap 可能走 `WILL_INIT`，不一定需要 FPI。
为了更稳定地观察普通已存在页面的 FPI，可以先插入一些数据让页面存在，再 checkpoint，再 update 其中一行。
对照源码思考：

- 为什么 checkpoint 后首次修改更容易带 FPI？
- 如果 record 使用 `WILL_INIT`，为什么可以不带 FPI？
- 如果 block 带 FPI，为什么同一 block 的 block data 可能消失？

---

## 45. 实验三：对比 heap 和 btree 的 block data
这个实验不是直接解码 block data bytes。
而是通过源码和 WAL dump 输出建立 mental model。
准备表和索引：
```sql
DROP TABLE IF EXISTS wal_lesson_heap;
CREATE TABLE wal_lesson_heap(id int primary key, payload text);
```
插入多行：
```sql
INSERT INTO wal_lesson_heap
SELECT g, md5(g::text)
FROM generate_series(1, 1000) AS g;
```
查看 WAL。
你会看到 heap insert/multi-insert 相关记录，也会看到 btree insert 相关记录。
然后回到源码：

- heap insert 的 block 0 data 是 heap tuple 重建材料。
- btree insert 的 block 0 data 是 index tuple 或 posting split payload。
- 两者都通过 `XLogRegisterBufData(0, ...)` 进入 WAL。
- redo 侧都通过 `XLogRecGetBlockData(record, 0, ...)` 读取。
相同 API，不同语义。
这就是 rmgr contract 的核心。

---

## 46. 讨论题

1. 为什么 WAL record 选择“通用容器 + rmgr payload”，而不是为 heap、btree、GIN 各自定义完全独立的日志格式？
2. `XLogRegisterBuffer()` 只登记 page 身份和指针，为什么不能在调用点立刻决定最终 record 一定带 FPI 或 block data？
3. 如果一个 block 同时登记了 block data，又因为 checkpoint 边界带了 FPI，redo 侧为什么通常不需要再读这段 block data？
4. `REGBUF_KEEP_DATA` 打破“有 FPI 就省略 block data”的默认规则时，服务的是 redo、logical decoding，还是两者都有？
5. heap insert 和 btree split 都使用 block reference，为什么 block 0/1/2 的语义不能由通用 WAL 层解释？
6. 如果 redo routine 忘记在修改 page 后设置 page LSN，会怎样破坏后续幂等判断和 WAL-before-data？
7. `pg_waldump` 能看到 rmgr、info 和 block image，但为什么不能直接证明某个 block data payload 的业务语义？
8. 本节的可迁移规律是什么：什么时候应该把“格式 framing”和“模块语义”拆成两层 contract？

---

## 47. 本节小结

WAL record 是 PostgreSQL 描述持久化 page 修改的基本单位。
它由通用 record 格式和 rmgr-specific payload 共同组成。
通用层负责：

- record header。
- block reference。
- main data 和 block data 的 framing。
- FPI 决策。
- CRC。
- WAL byte position。
- decode 后的访问 API。
rmgr 负责：

- 定义 `xl_info` opcode。
- 定义 main data struct。
- 定义每个 block id 的含义。
- 定义 block data payload 格式。
- 编写 redo 例程。
- 必要时编写 desc、identify、mask、decode。

`XLogBeginInsert()` 开启一次 record 构造。

`XLogRegisterBuffer()` 登记被修改 page。

`XLogRegisterData()` 登记 main data。

`XLogRegisterBufData()` 登记绑定某个 block 的 page delta。

`XLogInsert()` 决定 FPI、组装 record、插入 WAL，并返回可写入 page LSN 的 end LSN。
block data 和 FPI 的边界由 WAL 层控制。
默认有 FPI 时省略同 block 的 block data。

`REGBUF_KEEP_DATA` 可以保留它。

`REGBUF_WILL_INIT` 则要求 redo 从头初始化 page，从而不需要 FPI。
redo 的初步契约是幂等地把页面推进到 record 描述的状态。
它先通过 page LSN 判断是否需要重放。
如果有 apply FPI，先恢复整页。
如果需要 delta redo，再读取 main data 和 block data 执行 rmgr-specific 操作。
heap 和 btree 使用同一套 WAL record API。
但它们的 block id、payload 和 redo 行为完全不同。
这就是 rmgr 的意义。

---

## 48. 下一步衔接

下一节可以继续深入 WAL insertion。
本节把 record 当成逻辑结构读完。
下一节要看它如何进入 WAL buffers。
重点会从 `XLogRecordAssemble()` 转到 `XLogInsertRecord()`、WAL insert lock、WAL buffer mapping、record 跨 WAL page 写入和并发插入位置保留。
到那时，本节的 record layout 会成为理解 WAL byte stream 的前提。
