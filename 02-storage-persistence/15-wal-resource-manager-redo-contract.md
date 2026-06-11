# PostgreSQL WAL resource manager 与 redo contract

## 课程定位

前置知识：已经看过 WAL record layout、full-page image、page LSN、WAL-before-data 和 crash recovery 需要按 record 顺序重放数据页修改。

本节唯一主问题：

```text
一条已经通过通用 WAL 层读取和校验的 record，怎样交给 rmgr 安全、幂等地改变数据页？
```

核心矛盾：通用 WAL 层必须不理解 heap、btree、GIN 等具体语义；但 crash recovery 又必须让每个具体页面修改严格按模块 contract 生效。

一句话运行模型：

```text
recovery loop 读取并校验 record 后，ApplyWalRecord() 根据 xl_rmid 分派到 RmgrTable；rmgr 通过 xlogreader helper 读取 main/block data，通过 xlogutils 获取或初始化 page，按 FPI/page LSN 判断是否需要 redo，修改后设置 page LSN 并 mark dirty。
```

学完后应能判断：`RmgrId` 为什么是 WAL 格式的一部分；`rmgrlist.h` 顺序为什么不能随便改；有 FPI 和无 FPI 时 redo 判断为什么不同；`BLK_DONE`、`BLK_NEEDS_REDO`、`BLK_RESTORED` 对 redo routine 意味着什么。

源码基线：本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。

## 1. 本节在总主线中的位置

前面课程已经把 WAL record 的生成、插入、flush、FPI 和 segment 生命周期讲完。本节转到 recovery 侧：一条 record 被读出并校验之后，如何变成对具体数据页的一次安全修改。

这节不是重新列举所有 rmgr。它重点讲通用 WAL 层和具体 rmgr redo routine 之间的 contract，这个 contract 决定 crash recovery 是否幂等、是否能防 torn page、是否能在错误 WAL 内容上及时停止。

## 2. 核心矛盾与一句话运行模型

通用 WAL 层负责读取 WAL page、拼接跨页 record、校验 CRC、decode block references 和 FPI；但它不能理解 heap tuple 或 btree split 的业务语义。rmgr 层正好接过这一部分，同时必须遵守 page LSN 和 FPI 的幂等规则。

redo routine 的最短 contract：

```text
只解释自己的 xl_info/payload
  -> 用 xlogreader helper 取 main/block data
  -> 用 xlogutils 获取、初始化或 restore page
  -> 仅在需要 redo 的页面上做物理修改
  -> 修改后设置 page LSN 并 mark dirty
  -> 对 WAL 内容自相矛盾的情况报错
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节读什么 |
| --- | --- | --- |
| 1 | `src/backend/access/transam/rmgr.c`、`src/include/access/rmgr.h`、`src/include/access/rmgrlist.h` | `RmgrId`、`RmgrTable`、rmgr entry 顺序和 WAL 格式边界。 |
| 2 | `src/backend/access/transam/xlogrecovery.c` | recovery 主循环、`ApplyWalRecord()` 如何分派 rmgr redo。 |
| 3 | `src/backend/access/transam/xlogreader.c`、`src/include/access/xlogreader.h` | `XLogReaderState`、decoded record、main/block data helper。 |
| 4 | `src/backend/access/transam/xlogutils.c`、`src/include/access/xlogutils.h` | `XLogReadBufferForRedo()`、FPI restore、page LSN 判断、redo buffer contract。 |
| 5 | `src/backend/access/heap/heapam_xlog.c` | heap redo 示例。 |
| 6 | `src/backend/access/nbtree/nbtxlog.c` | btree redo 与多页 split 示例。 |
| 7 | `src/backend/access/gist/gistxlog.c`、`src/backend/access/gin/ginxlog.c` | GiST/GIN 对 `BLK_RESTORED` 等边界的差异。 |
| 8 | `src/include/access/xlog_internal.h`、`src/include/access/xlogrecord.h` | record header、block reference 和 FPI 格式辅助核对。 |

## 4. 关键结论：rmgr redo contract

PostgreSQL 的 WAL replay 分成两层 contract。
第一层是通用 WAL 层。
它负责读取 WAL page、校验 page header、拼接跨页 record、校验 record CRC、decode record header、decode block reference、decode full-page image 和 block data。
第二层是 rmgr 层。
它负责解释 `xl_rmid` 和 `xl_info` 指向的具体操作。
通用 WAL 层不理解 heap tuple 怎么拼回去。
通用 WAL 层也不理解 btree split 后左页、右页和右邻页的指针应该怎么互相修。
这些语义属于 rmgr。
WAL record header 里的 `xl_rmid` 决定使用哪个 rmgr。
同一个 rmgr 内，`xl_info` 决定具体 record 类型。
`rmgr.h` 把 `RmgrId` 定义成 `uint8`。
这意味着 rmgr id 是 WAL 文件格式的一部分。
随意扩大它不是普通 C 类型调整，而是 WAL 格式变更。
内置 rmgr id 的数值由 `rmgrlist.h` 的 entry 顺序决定。
`rmgr.c` 用同一个 `rmgrlist.h` 展开出 `RmgrTable`。
所以 `RM_HEAP_ID`、`RM_BTREE_ID` 等枚举值和 `RmgrTable[rmid]` 的函数表位置天然同步。
crash recovery 读到一条 record 后，`xlogrecovery.c` 的 `ApplyWalRecord()` 最终调用：

```c
GetRmgr(record->xl_rmid).rm_redo(xlogreader);
```

这行代码是 rmgr redo contract 的入口。
传给 `rm_redo` 的不是裸 WAL bytes。
传入的是 `XLogReaderState *`。
这个 reader 的 `record` 字段已经指向 `DecodedXLogRecord`。
redo routine 可以通过 `XLogRecGetData()` 取 main data。
可以通过 `XLogRecGetBlockData()` 取某个 block 的 rmgr-specific payload。
可以通过 `XLogRecGetBlockTag()` 取 relation fork block identity。
可以通过 `XLogReadBufferForRedo()` 让通用 recovery 工具完成 FPI restore、page LSN 判断、buffer pin/lock。
最重要的幂等 contract 是：
如果 WAL record 没有需要应用的 FPI，就比较 record `EndRecPtr` 和 page header 里的 `pd_lsn`。
如果 `EndRecPtr <= PageGetLSN(page)`，说明这条 record 对这个 page 已经生效，redo routine 应该跳过该页的增量修改。
如果 `EndRecPtr > PageGetLSN(page)`，说明需要 redo。
这就是 `BLK_DONE` 和 `BLK_NEEDS_REDO` 的来源。
但有 FPI 且 `BKPIMAGE_APPLY` 置位时，规则故意不同。
`XLogReadBufferForRedoExtended()` 会先 restore FPI。
它甚至会在磁盘 page 看起来更新时也 restore。
原因是 crash 期间的 database page 可能 torn 或错误写入。
WAL record 已经通过 CRC 校验，FPI 比数据文件里的 page 更可信。
因此 FPI restore 是防 torn page 的优先 contract。
这会让后续同一 page 的 WAL record 重新 replay，而不是误以为 page 已经够新。
redo routine 自己的 contract 可以概括为七条：
1. 只解释属于自己 rmgr 的 `xl_info` 和 payload。
2. 用 xlogreader helper 读取已经 decode 的 main data 和 block data。
3. 对每个要修改的 page 通过 xlogutils 读取、初始化或 restore buffer。
4. 只在 `BLK_NEEDS_REDO` 或特定需要处理 `BLK_RESTORED` 的场景下改 page。
5. 修改 page 后设置 page LSN 为 record end LSN，并 mark dirty。
6. 对 `BLK_DONE` 保持幂等，不重复做物理修改。
7. 对 WAL 内容自相矛盾的情况报错，常见边界是 `PANIC` 或 `ERROR`。

## 5. rmgr id 与 rmgr table

先看 `src/include/access/rmgr.h`。
这个头文件定义：

```c
typedef uint8 RmgrId;
```

它还通过宏展开 `rmgrlist.h` 来生成 `enum RmgrIds`。
`rmgr.h` 的注释明确说，内置 rmgr 的实际数值由 `rmgrlist.h` entry 顺序决定。
它还提醒：`RM_MAX_ID` 必须放得进 `RmgrId`，扩大类型会影响 XLOG 文件格式。
这句话是理解 rmgr id 的关键。
`RmgrId` 不是内存里的临时 enum。
它会写进 `XLogRecord.xl_rmid`。
因此一个旧集群里已经存在的 WAL record，必须在新 binary 下还能把同一个数字解释成同一个 rmgr。
`rmgrlist.h` 的注释更直接。
它说新的 entry 应该加在末尾，避免改变现有 entry 的 id。
还说改这个 list 可能需要 bump `XLOG_PAGE_MAGIC`。
这就是 WAL 兼容性边界。
本基线的 `rmgrlist.h` 包含这些内置 rmgr：
- `RM_XLOG_ID`
- `RM_XACT_ID`
- `RM_SMGR_ID`
- `RM_CLOG_ID`
- `RM_DBASE_ID`
- `RM_TBLSPC_ID`
- `RM_MULTIXACT_ID`
- `RM_RELMAP_ID`
- `RM_STANDBY_ID`
- `RM_HEAP2_ID`
- `RM_HEAP_ID`
- `RM_BTREE_ID`
- `RM_HASH_ID`
- `RM_GIN_ID`
- `RM_GIST_ID`
- `RM_SEQ_ID`
- `RM_SPGIST_ID`
- `RM_BRIN_ID`
- `RM_COMMIT_TS_ID`
- `RM_REPLORIGIN_ID`
- `RM_GENERIC_ID`
- `RM_LOGICALMSG_ID`
- `RM_XLOG2_ID`
这个列表每一行都是 `PG_RMGR(...)`。
宏参数的形式是：

```c
PG_RMGR(symbol, name, redo, desc, identify, startup, cleanup, mask, decode)
```

同一个文件被不同调用方用不同 `PG_RMGR` 宏展开。
`rmgr.h` 用它生成 enum。
`rmgr.c` 用它生成 `RmgrTable`。
`rmgr.c` 中的宏展开是函数表初始化。
`RmgrTable` 的每个 entry 包含 rmgr 名字、redo、desc、identify、startup、cleanup、mask、logical decode 回调。
这个表的定义在 `src/backend/access/transam/rmgr.c:50` 附近。
`RmgrData` 的结构定义在 `src/include/access/xlog_internal.h:348-360` 附近。
字段意义如下：
- `rm_name` 是输出和识别用名字。
- `rm_redo` 是 recovery apply 的核心入口。
- `rm_desc` 生成 `pg_waldump`、debug 和错误上下文中的详细描述。
- `rm_identify` 把 `xl_info` 转成 record 类型名字。
- `rm_startup` 在 redo 阶段开始时初始化 rmgr 私有状态。
- `rm_cleanup` 在 redo 阶段结束时清理 rmgr 私有状态。
- `rm_mask` 给 WAL consistency checking 屏蔽不稳定 page bytes。
- `rm_decode` 给 logical decoding 使用。
`RmgrTable` 的有效 entry 以 `rm_name != NULL` 判断。
`RmgrIdExists()` 和 `GetRmgr()` 在 `xlog_internal.h:372-382` 附近。
`GetRmgr()` 如果发现 id 未注册，会调用 `RmgrNotFound()`。
`RmgrNotFound()` 的错误提示要求把实现该 rmgr 的 extension 放进 `shared_preload_libraries`。
这说明本版本已经支持 custom WAL rmgr。
自定义 rmgr 的 id 范围在 `rmgr.h` 里定义：
- `RM_MIN_CUSTOM_ID` 是 128。
- `RM_MAX_CUSTOM_ID` 是 255。
- `RM_EXPERIMENTAL_ID` 是 128。
`RegisterCustomRmgr()` 在 `rmgr.c:107-144` 附近。
它检查四类边界。
第一，名字不能为空。
第二，id 必须在 custom range。
第三，只能在处理 `shared_preload_libraries` 初始化期间注册。
第四，不能和已有 id 或已有名字冲突。
这个限制和 recovery contract 直接相关。
如果 standby 或 crash recovery 需要 replay 某个 custom rmgr 的 WAL，进程启动时必须已经注册这个 rmgr。
否则 WAL stream 里合法的 `xl_rmid` 数字会找不到实现。
那不是可跳过的 warning。
数据库无法知道这个 record 对哪些文件做了什么语义修改。
因此 recovery 必须停止。

## 6. record header：`xl_rmid` 与 `xl_info`

`src/include/access/xlogrecord.h` 定义 WAL record 的通用格式。
固定头是 `XLogRecord`。
关键字段包括：
- `xl_tot_len`
- `xl_xid`
- `xl_prev`
- `xl_info`
- `xl_rmid`
- `xl_crc`
`xl_rmid` 选择 rmgr。
`xl_info` 的高 4 bits 由 rmgr 使用。
低位有通用 flag。
`XLR_INFO_MASK` 是通用层要屏蔽掉的 bit mask。
很多 redo routine 的第一行都是：

```c
uint8 info = XLogRecGetInfo(record) & ~XLR_INFO_MASK;
```

heap 还会再套一层 `XLOG_HEAP_OPMASK`。
btree、gist、gin 则直接 switch 高位 opcode。
这里有一个很容易混淆的点。
`xl_rmid` 决定“由谁解释”。
`xl_info` 决定“解释成该 rmgr 下的哪种操作”。
`xl_info` 本身不具备全局意义。
`0x10` 在 heap 里可以是 delete。
`0x10` 在 btree 里可以是 upper insert。
所以所有 `rm_identify()`、`rm_desc()` 和 `rm_redo()` 都必须在 rmgr 上下文中解释 `xl_info`。
`xlogreader.c` 在校验 record header 时只做通用合法性检查。
它检查 record 长度。
它检查 `xl_rmid` 是否落在 valid range。
它检查 prev-link。
它不会检查 heap 的 `xl_info` 是否有效。
heap 的未知 opcode 会在 `heap_redo()` 里 `PANIC`。
btree 的未知 opcode 会在 `btree_redo()` 里 `PANIC`。
这就是通用层和 rmgr 层的错误边界。

## 7. crash recovery 主循环如何进入 rmgr redo

主线在 `src/backend/access/transam/xlogrecovery.c`。
`PerformWalRecovery()` 是 WAL recovery 的核心函数。
如果系统干净关闭，就不会进入这个函数。
进入 recovery 后，系统先确定 checkpoint、redo start LSN、timeline 等上下文。
然后用 `XLogPrefetcherBeginRead()` 把 reader 定位到 redo 起点。
再用 `ReadRecord()` 取第一条要 replay 的 record。
如果有 record 需要 replay，`PerformWalRecovery()` 会设置：

```c
InRedo = true;
```

然后调用：

```c
RmgrStartup();
```

`RmgrStartup()` 在 `rmgr.c:58-67` 附近。
它遍历 `0..RM_MAX_ID`。
对每个已存在 rmgr，如果 `rm_startup` 非空，就调用它。
btree、gin、gist、spgist 这类索引 rmgr 会使用 startup/cleanup 管理 redo 工作内存。
随后进入 main redo apply loop。
循环中会处理中断、recovery pause、recovery target、apply delay。
真正 replay 一条 record 的调用是：

```c
ApplyWalRecord(xlogreader, record, &replayTLI);
```

`ApplyWalRecord()` 在 `xlogrecovery.c:1883` 附近。
它先安装一个 error context callback。
这个 callback 是 `rm_redo_error_callback()`。
如果 rmgr redo 内部报错，错误上下文会包含 WAL redo LSN 和 record 描述。
这就是你看到 “WAL redo at ... for Heap/INSERT ...” 这种错误上下文的来源。
`ApplyWalRecord()` 然后推进 `nextXid`。
它处理 timeline switch。
它在 replay 前更新共享内存里的 `replayEndRecPtr` 和 `replayEndTLI`。
注释说明这样 `XLogFlush` 更新 `minRecoveryPoint` 时能看到正确 replay end。
接着，如果 hot standby 已经初始化，并且 record 带 xid，就记录 known assigned xid。
然后有一个特殊分支：
如果 record 的 rmgr 是 `RM_XLOG_ID`，先调用 `xlogrecovery_redo()`。
这个函数处理和 recovery 状态直接相关的 XLOG record。
之后不管是不是 `RM_XLOG_ID`，都会调用通用 rmgr redo 分派：

```c
GetRmgr(record->xl_rmid).rm_redo(xlogreader);
```

这意味着 `RM_XLOG_ID` 也有自己的 `xlog_redo()`。
只是 recovery 状态相关的部分在 `ApplyWalRecord()` 里提前处理。
redo 完成后，如果 record 带 `XLR_CHECK_CONSISTENCY`，会调用 `verifyBackupPageConsistency()`。
最后更新 `lastReplayedReadRecPtr`、`lastReplayedEndRecPtr` 和 timeline。
这三个步骤很重要。
`replayEndRecPtr` 是“正在 replay 或即将 replay 到哪里”。
`lastReplayedEndRecPtr` 是“已经成功 replay 到哪里”。
如果 rmgr redo 报错，后者不会推进。
这就是 crash recovery 的可解释进度边界。
redo loop 结束后，`PerformWalRecovery()` 调用 `RmgrCleanup()`。
注释提醒：达到 recovery target 后，这是最后一个还能带着新 recovery target 重新启动 recovery 的点；之后 resource manager 可以做 end-of-recovery 的永久修正动作。
这说明 rmgr cleanup 不是普通内存释放那么简单。
它也属于 recovery contract 的收尾阶段。

## 8. xlogreader：读出来不等于 redo

`src/include/access/xlogreader.h` 的文件注释给出 xlogreader 的使用模型。
调用方分配 `XLogReaderState`。
用 `XLogBeginRead()` 或 `XLogFindNextRecord()` 定位。
然后反复调用 `XLogReadRecord()`。
读到一条 record 后，xlogreader 会把它拆成 per-block data 和 main data。
访问这些部分要用 `XLogRec*` 宏和函数。
`XLogReadRecord()` 在 `xlogreader.c:391` 附近。
它先释放上一条 record。
再调用 `XLogReadAhead()` 确保 decode queue 里有 record。
然后用 `XLogNextRecord()` 消费队首。
返回值是 record header 指针。
但调用方后续访问的是 `xlogreader->record` 指向的 decoded record。
这就是为什么 redo routine 收到的是 `XLogReaderState *`，不是 `XLogRecord *`。
`XLogDecodeNextRecord()` 是读取和校验的核心。
它处理 WAL page header。
它处理 record header 跨 page 的情况。
它处理 continuation record。
它处理 `XLOG_SWITCH` 的特殊 next pointer。
它在读完 record bytes 后调用 `ValidXLogRecord()` 校验 CRC。
`ValidXLogRecordHeader()` 在 `xlogreader.c:1139` 附近。
它检查：
- `xl_tot_len` 至少包含固定 header。
- `xl_rmid` 在 valid range。
- 随机访问时 `xl_prev < RecPtr`。
- 顺序读取时 `xl_prev == PrevRecPtr`。
顺序 prev-link 检查可以防止 torn WAL page 上出现看起来合法的旧 record。
`ValidXLogRecord()` 在 `xlogreader.c:1205` 附近。
它先对 record payload 计算 CRC。
再把 header 中 `xl_crc` 前面的字段纳入 CRC。
如果 CRC 不匹配，报告 “incorrect resource manager data checksum”。
注意这个错误消息里的 “resource manager data” 指的是 record 中 rmgr-specific data 的校验。
它不代表 rmgr redo 已经开始。
record decode 发生在 `DecodeXLogRecord()`。
这个函数在 `xlogreader.c:1701` 附近。
它把裸 record bytes 解析成 `DecodedXLogRecord`。
`DecodedXLogRecord` 的定义在 `xlogreader.h`。
它包含：
- record 起点 `lsn`
- 下一条 record 起点 `next_lsn`
- 固定 header
- origin
- top-level xid
- main data 指针和长度
- 最大 block id
- flexible array 形式的 decoded block
每个 block 对应一个 `DecodedBkpBlock`。
`DecodedBkpBlock` 包含：
- `in_use`
- `rlocator`
- `forknum`
- `blkno`
- `prefetch_buffer`
- `flags`
- 是否有 full-page image
- full-page image 是否应该 apply
- image 指针、hole offset、hole length、image length、image info
- 是否有 block data
- block data 指针和长度
这个结构名里的 `BkpBlock` 是历史术语。
今天它同时承载 block reference、FPI 和 per-block data。
`DecodeXLogRecord()` 的解析顺序是：
先扫 header fragment。
遇到 block id 就解析 block header。
遇到 `XLR_BLOCK_ID_DATA_SHORT` 或 `XLR_BLOCK_ID_DATA_LONG` 就解析 main data header，并按约定结束 header 扫描。
然后检查剩余长度是否等于所有 data fragment 总和。
最后把 block image、block data、main data 拷贝到 decoded record 后面的连续内存里。
block data 和 main data 会做 alignment。
所以 redo routine 可以把指针 cast 成自己的 WAL struct。
这也是 `XLogRecGetData()` 常被直接 cast 的原因。
decode 阶段会做许多通用一致性检查。
例如：
- block id 必须递增。
- `BKPBLOCK_HAS_DATA` 必须和 data length 一致。
- `BKPBLOCK_HAS_IMAGE` 时 image header 必须自洽。
- `BKPIMAGE_HAS_HOLE` 的 hole offset 和 hole length 必须合理。
- 压缩 image 的长度不能等于完整 `BLCKSZ`。
- `BKPBLOCK_SAME_REL` 不能出现在没有 previous rel 的地方。
这些检查让 rmgr redo 不必自己重新解析 WAL record 外壳。
但它不会验证 rmgr-specific payload 的语义。
例如 heap insert 的 block data 是否真的包含 `SizeOfHeapHeader` 后面的 tuple bytes，是 heap redo 自己检查。

## 9. redo 侧怎样访问 decoded record

`xlogreader.h` 在末尾定义了常用 accessor。
最常用的是：

```c
XLogRecGetRmid(record)
XLogRecGetInfo(record)
XLogRecGetXid(record)
XLogRecGetData(record)
XLogRecGetDataLen(record)
XLogRecHasBlockRef(record, block_id)
XLogRecHasBlockImage(record, block_id)
XLogRecBlockImageApply(record, block_id)
XLogRecHasBlockData(record, block_id)
```

还有几个函数：

```c
XLogRecGetBlockData(record, block_id, &len)
XLogRecGetBlockTag(record, block_id, &rlocator, &forknum, &blkno)
XLogRecGetBlockTagExtended(record, block_id, ..., &prefetch_buffer)
RestoreBlockImage(record, block_id, page)
```

redo routine 的典型形态是：

```c
xl_foo *xlrec = (xl_foo *) XLogRecGetData(record);
XLogRedoAction action = XLogReadBufferForRedo(record, 0, &buffer);
if (action == BLK_NEEDS_REDO)
{
    char *data = XLogRecGetBlockData(record, 0, &len);
    page = BufferGetPage(buffer);
    ...
    PageSetLSN(page, record->EndRecPtr);
    MarkBufferDirty(buffer);
}
if (BufferIsValid(buffer))
    UnlockReleaseBuffer(buffer);
```

这个伪代码里有两个边界。
main data 是整条 record 的 rmgr-specific 参数。
block data 是绑定到某个 block reference 的 rmgr-specific bytes。
full-page image 不是 block data。
如果某个 block 因为 FPI 足以恢复页面，insert 侧可能没有保留 block data。
所以 redo routine 不能假设每个 block reference 都有 block data。
它必须按该 record 类型的约定取。
如果 record 类型要求 block data 存在，但 `XLogRecGetBlockData()` 返回 NULL 或长度不够，redo routine 应该报错。
这里不要把 xlogreader accessor 当成“业务层校验器”。
它只是给 rmgr 提供已经安全拆包后的指针。
业务语义仍归 rmgr。

## 10. full-page image 的应用顺序

FPI 应用的核心在 `xlogutils.c` 和 `xlogreader.c` 两个文件之间。
`xlogreader.c` 负责 decode image。
`xlogutils.c` 负责决定 restore image 还是按 page LSN 做增量 redo。
`RestoreBlockImage()` 在 `xlogreader.c:2095` 附近。
它先检查 block id 是否存在。
再检查该 block 是否真的有 image。
如果 image 压缩过，就按 `BKPIMAGE_COMPRESS_PGLZ`、`BKPIMAGE_COMPRESS_LZ4` 或 `BKPIMAGE_COMPRESS_ZSTD` 解压。
如果构建不支持对应压缩算法，会报告 invalid record。
如果 image 有 hole，就把 hole 前后的 bytes 拷贝到 page，并把 hole 清零。
如果没有 hole，就直接拷贝 `BLCKSZ`。
`RestoreBlockImage()` 只负责把 image bytes 恢复到目标 page buffer。
它不负责读取 buffer。
它也不负责设置 page LSN。
这些在 `XLogReadBufferForRedoExtended()` 里完成。
`XLogReadBufferForRedoExtended()` 在 `xlogutils.c:340` 附近。
它先通过 `XLogRecGetBlockTagExtended()` 取得 relation、fork、block number 和 prefetch buffer。
如果 block id 不存在，直接 `PANIC`。
随后检查 `BKPBLOCK_WILL_INIT` 和读取 mode 是否匹配。
如果 WAL record 标记 `WILL_INIT`，redo routine 必须用 zero mode 初始化。
如果 redo routine 要 zero/init 一个 block，WAL record 也必须标记 `WILL_INIT`。
两者不一致都是 `PANIC`。
这条 contract 很重要。
它防止 redo routine 在没有 WAL 元数据许可的情况下创建或重置一个 page。
也防止 WAL record 声明 page 会被 redo 初始化，但 rmgr 实现却按普通 page 读取。
接下来是 FPI 分支。
如果 `XLogRecBlockImageApply(record, block_id)` 为 true：
1. 用 zero-and-lock 模式读出或扩展目标 buffer。
2. 调用 `RestoreBlockImage()` 把 image 放入 page。
3. 如果 page 不是全新未初始化状态，就把 page LSN 设成 record end LSN。
4. `MarkBufferDirty()`。
5. 如果是 unlogged relation 的 init fork，就 `FlushOneBuffer()`。
6. 返回 `BLK_RESTORED`。
这个分支有一个注释非常关键。
当 backup block 有 `BKPIMAGE_APPLY` 时，即使数据库 page 看起来更新，也要 restore。
因为 crash 时数据库 page 可能部分写入或错误写入。
WAL record 通过了 CRC，数据库 page 未必可信。
这就是 FPI 优先于 page LSN 判断的原因。
因此 `BLK_RESTORED` 的语义不是“record 已经完全处理完”。
它的语义是“这个 block 已经被 full-page image 恢复过”。
对多数 rmgr 操作来说，FPI 就足够了。
但有少数操作还要在 `BLK_RESTORED` 后继续做额外修改。
GiST 的 `gistRedoClearFollowRight()` 就是这种例子。

## 11. 无 FPI 时的 LSN 判断与幂等性

如果 block 没有要应用的 FPI，`XLogReadBufferForRedoExtended()` 进入普通分支。
它调用 `XLogReadBufferExtended()` 读取目标 buffer。
如果 buffer 有效，按需要取得 exclusive lock 或 cleanup lock。
然后比较：

```c
if (lsn <= PageGetLSN(BufferGetPage(*buf)))
    return BLK_DONE;
else
    return BLK_NEEDS_REDO;
```

这里的 `lsn` 是 `record->EndRecPtr`。
它不是 record start LSN。
PostgreSQL 的 page LSN 通常设置为 `XLogInsert()` 返回的 record end LSN。
redo 侧也把 page LSN 设成 `record->EndRecPtr`。
因此比较使用 end LSN。
幂等性来自这个判断。
crash recovery 可能遇到某个 page 已经包含了某条 WAL record 的修改。
原因包括数据页在 crash 前已经落盘。
也可能包括前一次 recovery 尝试已经 replay 到一半后又崩溃。
只要 page LSN 大于等于当前 record end LSN，redo 侧就认为该 page 不需要再应用这个 record。
这条规则也解释了 redo routine 为什么必须在实际修改后 `PageSetLSN(page, lsn)`。
如果忘了设置 LSN，同一条或后续记录的幂等判断就会失效。
如果设置得过早，页面可能在修改不完整时被认为已更新。
所以正常顺序是：
1. 读 buffer。
2. 判断 action。
3. 执行 page bytes 修改。
4. 设置 LSN。
5. mark dirty。
6. unlock/release。
`BLK_DONE` 并不意味着 redo routine 可以完全忽略整条 record。
它只意味着这个 block 的物理修改不需要重复。
有些 record 还有非 page 的副作用。
heap insert/update 里 visibility map 的清理就放在 page action 之外。
源码注释明确说：visibility map 可能需要修正，即使 heap page 已经 up-to-date。
这类逻辑是 rmgr contract 中很容易漏掉的部分。
redo 的幂等性不是简单地“整条 record 执行过就 return”。
它经常是逐 block 判断。
一个 record 修改多个 block 时，某个 block 可能 `BLK_DONE`，另一个 block 可能 `BLK_NEEDS_REDO`。
btree split 就是多 block redo 的典型场景。

## 12. xlogutils 的 buffer 读取 contract

`src/backend/access/transam/xlogutils.c` 开头说明：这个文件包含 XLOG replay 使用的支持函数，正常运行路径不用它。
这里的核心函数是三组。
第一组是：

```c
XLogReadBufferForRedo()
XLogReadBufferForRedoExtended()
```

它们用于读取 WAL record 引用的 block，并决定是否需要 redo。
返回值是 `XLogRedoAction`。
`xlogutils.h` 定义四个值：
- `BLK_NEEDS_REDO`
- `BLK_DONE`
- `BLK_RESTORED`
- `BLK_NOTFOUND`
`BLK_NEEDS_REDO` 表示 WAL record 的增量修改需要应用。
`BLK_DONE` 表示 page LSN 已经足够新。
`BLK_RESTORED` 表示 page 已经从 FPI restore。
`BLK_NOTFOUND` 表示 page 不存在，通常是后续 WAL 中有 drop 或 truncate。
第二组是：

```c
XLogInitBufferForRedo()
```

它用于 redo routine 需要重新初始化一个 page 的场景。
内部调用 `XLogReadBufferForRedoExtended()`，mode 是 `RBM_ZERO_AND_LOCK`。
返回 buffer 后，redo routine 通常会调用 `PageInit()` 或索引页初始化函数。
heap insert 的 `XLOG_HEAP_INIT_PAGE` 分支就是这个模式。
btree split 对新右页使用这个模式。
第三组是：

```c
XLogReadBufferExtended()
```

注释提醒：redo function 通常不应该直接调用它来拿要修改的 page。
因为要修改的 page 应该在 WAL record 中注册。
直接按 relation/block 读取，会绕过 record block reference，外部工具也看不到这个 record 修改了哪些 page。
`XLogReadBufferExtended()` 的行为取决于 `ReadBufferMode`。
在 `RBM_NORMAL` 模式下，如果 page 不存在，或者 page 全零未初始化，返回 `InvalidBuffer`。
同时会把 invalid page 记到 recovery 的 invalid-page table。
如果后续 WAL 确实 drop 或 truncate 了这个 page，表项会被清理。
如果到 consistency point 或 recovery 结束仍然有 unresolved invalid page，就报错或 `PANIC`。
在 `RBM_NORMAL_NO_LOG` 模式下，如果 page 不存在，返回 `InvalidBuffer`，但不记录 invalid page。
在 `RBM_ZERO_*` 模式下，如果 page 不存在，可以扩展 relation 到目标 block。
这只允许在 recovery 中做。
源码里有 `Assert(InRecovery)`。
这也是 redo contract 的一部分。
正常运行时 extend relation 需要 relation extension lock。
recovery 中 startup process 单独执行物理重放，可以使用特殊 flag 跳过普通扩展锁。
`XLogReadBufferExtended()` 还会创建目标 fork 文件。
源码注释说明，这能应对 replay 序列中写入一个后来被删除的 relation。
相比直接忽略写入，先把数据写出来、等后续 drop WAL 再删除，更不容易在文件系统异常后丢掉有价值的数据。
这段逻辑解释了为什么 `BLK_NOTFOUND` 不是随便吞掉错误。
它依赖后续 WAL 证明这个缺页是合法结果。

## 13. heap redo 示例：`heap_redo()`

heap 是最适合入门的 rmgr redo 示例。
它展示了 main data、block data、page LSN、visibility map、副作用和多 page update。
入口在 `src/backend/access/heap/heapam_xlog.c:1199` 附近。
`heap_redo()` 先取：

```c
uint8 info = XLogRecGetInfo(record) & ~XLR_INFO_MASK;
```

然后按 `info & XLOG_HEAP_OPMASK` switch。
主要分支包括：
- `XLOG_HEAP_INSERT`
- `XLOG_HEAP_DELETE`
- `XLOG_HEAP_UPDATE`
- `XLOG_HEAP_TRUNCATE`
- `XLOG_HEAP_HOT_UPDATE`
- `XLOG_HEAP_CONFIRM`
- `XLOG_HEAP_LOCK`
- `XLOG_HEAP_INPLACE`
未知 opcode 会 `PANIC`。
heap 还有 `heap2_redo()`。
`heapam_xlog.h` 注释说 heapam.c 已经用完了一组 opcode，所以 heapam WAL 有第二个 rmgr id。
`RM_HEAP2_ID` 不代表逻辑上另一个访问方法。
它是 WAL opcode 空间的扩展。
这点很适合说明 rmgr id 是 WAL 编码资源，不是面向对象的“模块类名”。

### 10.1 `heap_xlog_insert()`

`heap_xlog_insert()` 在 `heapam_xlog.c:366` 附近。
它先把 main data cast 成：

```c
xl_heap_insert *xlrec = (xl_heap_insert *) XLogRecGetData(record);
```

main data 包含 insert 的 offset、flags 等小结构。
然后通过 block 0 的 tag 取目标 relation 和 block number。
目标 tuple 的 block number 来自 WAL block reference。
目标 tuple 的 offset number 来自 `xlrec->offnum`。
这正好展示了通用 block identity 和 rmgr-specific main data 的组合。
接着有 visibility map 修正。
如果 insert 清掉了 heap page 的 all-visible 标志，redo 需要清 visibility map。
注释说，即使 heap page 已经 up-to-date，也可能需要修 visibility map。
所以这个逻辑不包在 `BLK_NEEDS_REDO` 判断里面。
然后处理目标 heap page。
如果 record 的 `xl_info` 带 `XLOG_HEAP_INIT_PAGE`，说明这是页面上第一个也是唯一 tuple。
redo 使用：

```c
buffer = XLogInitBufferForRedo(record, 0);
PageInit(page, BufferGetPageSize(buffer), 0);
action = BLK_NEEDS_REDO;
```

否则调用：

```c
action = XLogReadBufferForRedo(record, 0, &buffer);
```

如果 `action == BLK_NEEDS_REDO`，才拼 tuple 并插入 page。
tuple 的数据来自 block 0 的 block data：

```c
data = XLogRecGetBlockData(record, 0, &datalen);
```

payload 先包含 `xl_heap_header`。
后面是 tuple header 的 bitmap、padding、oid 和 data。
redo routine 重新构造 `HeapTupleHeader`。
它设置 xmin 为 record xid。
设置 cmin 为 `FirstCommandId`。
设置 `t_ctid` 为目标 tid。
然后用 `PageAddItem()` 把 tuple 放到 page 的指定 offset。
如果 page 最大 offset 和 WAL 里的 offset 不匹配，或 `PageAddItem()` 失败，就 `PANIC`。
这些不是用户 SQL 错误。
它们表示 WAL 内容和 page 状态不满足 redo contract。
修改完成后：

```c
PageSetLSN(page, lsn);
MarkBufferDirty(buffer);
```

最后释放 buffer。
如果 action 是 `BLK_NEEDS_REDO` 且剩余空间低于阈值，还更新 FSM。
源码注释说如果 page 是从 FPI restore 的，不更新 FSM。
FSM 不要求完全精确。
这也是 redo 里“必须恢复 correctness”和“可以接受 approximate metadata”的区别。

### 10.2 `heap_xlog_update()`

`heap_xlog_update()` 在 `heapam_xlog.c:697` 附近。
update 比 insert 更能体现多 block contract。
main data 是 `xl_heap_update`。
block 0 是新 tuple 所在 page。
如果 block 1 存在，它表示 old tuple 所在 page。
如果 block 1 不存在，old 和 new 在同一 page。
redo 先处理 old tuple version。
它调用：

```c
oldaction = XLogReadBufferForRedo(record, (oldblk == newblk) ? 0 : 1, &obuffer);
```

如果 `oldaction == BLK_NEEDS_REDO`，它会：
- 检查 old offset 是否在 page 范围内。
- 检查 line pointer 是否 normal。
- 修改 old tuple 的 xmax、cmax、infomask。
- 根据是否 HOT update 设置或清除 hot updated flag。
- 把 old tuple 的 `t_ctid` 指向 new tid。
- 设置 page prunable。
- 必要时清 all-visible。
- 设置 old page LSN。
- mark dirty。
然后处理 new tuple page。
如果 old 和 new 是同一 block，`nbuffer = obuffer`，`newaction = oldaction`。
如果是新 page 并带 `XLOG_HEAP_INIT_PAGE`，调用 `XLogInitBufferForRedo()` 并 `PageInit()`。
否则调用 `XLogReadBufferForRedo(record, 0, &nbuffer)`。
new tuple 的 block data 来自 block 0。
它可以包含 prefix/suffix 优化。
如果 record 标记 `XLH_UPDATE_PREFIX_FROM_OLD`，redo 从 old tuple 复制 prefix。
如果标记 `XLH_UPDATE_SUFFIX_FROM_OLD`，redo 从 old tuple 复制 suffix。
其余 bytes 来自 WAL block data。
这说明 WAL record 不一定保存完整 tuple image。
它保存的是该 rmgr redo routine 能够重建目标状态所需的最小语义材料。
同页 update 的锁释放也值得注意。
源码注释说正常运行时跨页 update 要按 page number 顺序加锁避免死锁。
但 WAL replay 没有其他 update 并发。
不过不能在新 tuple 写入前释放 old page，否则 Hot Standby 查询可能看到不一致状态。
因此 redo routine 的 buffer 持有顺序也是 contract。
它不是单纯的性能实现细节。

## 14. btree redo 示例：多页 split 的 contract

btree redo 入口在 `src/backend/access/nbtree/nbtxlog.c:1004` 附近。
`btree_redo()` 取 `xl_info` 后 switch。
分支包括 leaf insert、upper insert、meta insert、split、dedup、vacuum、delete、unlink、newroot、reuse page、meta cleanup。
它还切换到 `opCtx` 内存上下文。
`btree_xlog_startup()` 创建这个上下文。
`btree_xlog_cleanup()` 清理它。
这展示了 `rm_startup` 和 `rm_cleanup` 的实际用途。
btree split 在 `btree_xlog_split()`。
这个函数的 contract 比 heap insert 更复杂。
它先从 block reference 取：
- block 0：原 page，也就是 split 后的 left page。
- block 1：新 right sibling page。
- block 2：right sibling 的右邻页，如果存在。
- block 3：internal split 时下一层 child page，用于清 incomplete split。
如果 split 是 internal page，redo 先清 child page 的 incomplete split 标志。
然后重建 right page。
right page 使用：

```c
rbuf = XLogInitBufferForRedo(record, 1);
datapos = XLogRecGetBlockData(record, 1, &datalen);
_bt_pageinit(...);
_bt_restore_page(rpage, datapos, datalen);
PageSetLSN(rpage, lsn);
MarkBufferDirty(rbuf);
```

这是 page initialization contract。
WAL record 里 block 1 必须标记 `WILL_INIT`。
redo routine 必须初始化它。
left page 则按普通 incremental redo：

```c
if (XLogReadBufferForRedo(record, 0, &buf) == BLK_NEEDS_REDO)
```

left page 不是简单追加一个 index tuple。
redo 复制原 page 的 special area，构造 temp page，按 split 后的物理顺序重新插入 left page 项。
这样能保留和 `_bt_split()` 一致的物理顺序，也让 WAL consistency checking 可行。
如果存在 right-of-right sibling，redo 还要修它的 left link。
最后释放 buffers 的注释很重要。
`sbuf`、`rbuf`、`buf` 必须一起释放，避免 readers 观察到不一致状态。
这再次说明 redo contract 不只是“把 bytes 改对”。
它还包括 hot standby 可见性边界。

## 15. GiST 与 GIN：`BLK_RESTORED` 不是总能 return

多数 redo routine 只在 `BLK_NEEDS_REDO` 时修改 page。
因为如果返回 `BLK_RESTORED`，FPI 已经包含目标 page 状态。
但这个规律不是绝对的。
GiST 的 `gistRedoClearFollowRight()` 是反例。
它在 `src/backend/access/gist/gistxlog.c` 开头附近。
源码注释说，即使 WAL record 包含 full-page image，也必须更新 follow-right flag。
原因是该 change 不包含在 full-page image 里。
函数调用：

```c
action = XLogReadBufferForRedo(record, block_id, &buffer);
if (action == BLK_NEEDS_REDO || action == BLK_RESTORED)
```

然后设置 page NSN，清 `F_FOLLOW_RIGHT`，设置 page LSN 并 mark dirty。
这说明 `BLK_RESTORED` 的具体含义要结合 rmgr record 语义。
FPI restore 只表示某个 registered block 的 image 已经恢复。
如果 record 表达的是“先 restore image，再补一个不在 image 里的状态变化”，rmgr 还要继续做。
GIN 则展示另一个边界。
`ginRedoSplit()` 要求 split record 对 left page、right page，必要时 root page，都能通过 FPI restore。
如果 `XLogReadBufferForRedo()` 没有返回 `BLK_RESTORED`，它报 `ERROR`。
`ginRedoVacuumPage()` 也要求 vacuum page record restore page。
这类 record 的 redo contract 就是“这个 record 必须携带可应用 full-page image”。
如果没有，说明 WAL record 不符合 rmgr 语义。

## 16. crash recovery 中的 rmgr contract

现在把前面内容合成 recovery contract。
对通用 recovery 层来说，rmgr redo 是一个函数指针调用。
但对数据库正确性来说，它是 page state machine 的一部分。
通用层在调用 rmgr 前保证：
- WAL page header 已校验。
- record header 基本合法。
- record CRC 已校验。
- `xl_rmid` 落在 valid range。
- record 已 decode 成 `DecodedXLogRecord`。
- block reference 的通用结构自洽。
- `ReadRecPtr` 和 `EndRecPtr` 已设置。
- replay timeline 和 recovery progress 已准备好。
通用层不保证：
- `xl_info` 是该 rmgr 的合法 opcode。
- main data 长度满足该 rmgr 的 struct。
- block data 内容满足该 rmgr 的逻辑格式。
- page 上的 item id 一定在期望状态。
- 多 page 操作的所有页面都需要 redo。
rmgr redo 必须保证：
- 每个被修改 page 都通过 WAL block reference 读取。
- 需要初始化的 page 用 `XLogInitBufferForRedo()`。
- 普通修改用 `XLogReadBufferForRedo()` 或 extended 版本。
- 严格处理 `BLK_NEEDS_REDO`、`BLK_DONE`、`BLK_RESTORED`、`BLK_NOTFOUND`。
- 只在需要时应用物理修改。
- 修改后设置 page LSN 到 record end LSN。
- 修改后 mark dirty。
- 正确释放 buffer lock 和 pin。
- 对 Hot Standby 可见的中间状态保持谨慎。
- 对逻辑上不可能的 WAL 或 page 状态报错。
`BLK_NOTFOUND` 的处理通常是跳过该 block 的物理修改。
但它不是“数据丢了也算了”。
`xlogutils.c` 维护 invalid-page table。
如果后续 WAL 没有 drop/truncate 解释这些缺页，recovery 到一致点会报错。
这允许 recovery 处理“先 replay 到一个后来被删除的 relation/page”的序列。
也防止真的丢页被静默吞掉。
rmgr redo 在 recovery 中还要注意当前进程状态。
`InRecovery` 在 `xlogutils.c` 中定义。
注释说它只在 startup process 中表示“这个进程正在 replay WAL record”。
redo routine 和它调用的底层函数可以用这个变量跳过普通 WAL logging 或普通并发锁路径。
但其他进程判断系统是否处于 recovery 要用 `RecoveryInProgress()`。
不要把 `InRecovery` 当成全局数据库状态变量。
Hot Standby 相关状态在 `standbyState`。
redo routine 有时需要发 recovery conflict。
例如 btree、gist、gin 注释里提到 index 操作通常不需要 conflict processing。
heap2、xact、standby 等 rmgr 则会影响 snapshot、known assigned xids 和 conflict。
这些属于更高层 recovery contract。
本节重点先放在物理 page redo。

## 17. 错误边界

WAL redo 的错误边界比普通 SQL 执行更硬。
原因是 recovery 不能随便跳过一条 record。
一条 record 可能是后续所有 page state 的前提。
如果 skip，数据库可能进入自洽但错误的状态。
通用读取层的错误包括：
- WAL page magic 不匹配。
- WAL page info bit 非法。
- long page header 的 system identifier 不匹配。
- WAL segment size 或 `XLOG_BLCKSZ` 不匹配。
- page address 和预期 LSN 不一致。
- timeline 倒退或不在 expected history。
- record length 太短。
- rmgr id 不在 valid range。
- prev-link 不匹配。
- record CRC 不匹配。
- continuation record 缺失或长度不一致。
- decoded block id 乱序。
- block data flag 和长度不一致。
- FPI hole、compression、length 自相矛盾。
这些错误通常让 `ReadRecord()` 返回 NULL 或按 emode 报错。
crash recovery 初始 checkpoint 附近可能是 `PANIC` 或 `FATAL`。
standby 模式下可能等待更多 WAL 或换 source 重试。
rmgr table 层的错误包括：
- rmgr id 在 valid range 但未注册。
- custom rmgr 未在 `shared_preload_libraries` 初始化阶段注册。
- custom rmgr id/name 冲突。
这类错误不能靠 page LSN 跳过。
因为系统没有函数能解释 record 的语义。
xlogutils 层的错误包括：
- redo routine 指定的 block id 在 record 中不存在。
- `BKPBLOCK_WILL_INIT` 和 zero mode 不一致。
- FPI restore 请求的 block 没有 image。
- image 压缩算法当前 build 不支持。
- image 解压失败。
- invalid page 在达到一致点后仍未被 drop/truncate 解释。
rmgr redo 层的错误包括：
- 未知 opcode。
- offset 越界。
- line pointer 状态不符合预期。
- `PageAddItem()` 失败。
- record 要求 FPI restore 但没有返回 `BLK_RESTORED`。
- main data 或 block data 长度不满足该 record 类型。
哪些地方用 `ERROR`，哪些地方用 `PANIC`，取决于调用栈和错误性质。
但设计原则相同。
如果 WAL 和 page state 不满足 redo contract，不能假装成功。
还有一个反直觉边界。
page checksum 或 page LSN 看起来正常，不一定能阻止 FPI restore。
有 `BKPIMAGE_APPLY` 的 FPI 优先。
这是为了防 torn page 和错误写入。
也就是说，redo 的信任顺序是：
1. 已通过 CRC 的 WAL record。
2. record 中可应用的 FPI。
3. 无 FPI 时才使用数据页 LSN 判断是否已 replay。

## 18. 源码跟读练习

下面 5 个练习只读源码，不改源码。
目标不是枚举所有 rmgr，而是把分派、decode、buffer contract 和具体 redo 差异连成一条线。

### 练习 1：rmgr identity 与函数表

阅读 `src/include/access/rmgr.h`、`src/include/access/rmgrlist.h` 和 `src/backend/access/transam/rmgr.c`。
确认 `RmgrId` 是 `uint8`，内置 rmgr id 来自 `rmgrlist.h` 顺序，`RmgrTable` 也由同一份 list 展开。
回答：为什么新 rmgr 要追加到列表末尾，为什么修改顺序可能需要 bump `XLOG_PAGE_MAGIC`，哪些 rmgr 使用 startup/cleanup、mask 或 logical decode 回调。

### 练习 2：recovery 分派与通用 WAL 校验

阅读 `xlogrecovery.c` 中的 `PerformWalRecovery()`、`RmgrStartup()`、`ApplyWalRecord()`、`rm_redo_error_callback()`，再读 `xlogreader.c` 的 `ValidXLogRecordHeader()`、`ValidXLogRecord()` 和 `DecodeXLogRecord()`。
画出 `ReadRecord -> ApplyWalRecord -> rm_redo -> progress update` 的顺序。
回答：CRC 和 `xl_prev` 为什么在 rmgr redo 前验证，redo 错误上下文为什么要包含 rmgr desc 和 block info，`replayEndRecPtr` 与 `lastReplayedEndRecPtr` 分别在 redo 前后表达什么。

### 练习 3：FPI、page LSN 与 xlogutils contract

阅读 `xlogreader.c` 的 `RestoreBlockImage()`，以及 `xlogutils.c` 的 `XLogReadBufferForRedoExtended()`、`XLogInitBufferForRedo()`、`log_invalid_page()`、`XLogCheckInvalidPages()`。
把返回值分成 `BLK_RESTORED`、`BLK_NEEDS_REDO`、`BLK_DONE` 三类。
回答：为什么有可应用 FPI 时不先比较 page LSN，为什么普通路径比较 record end LSN，`BKPBLOCK_WILL_INIT` 和 `XLogInitBufferForRedo()` 为什么必须互相匹配，invalid page 为什么可以暂存但不能在达到一致状态后继续存在。

### 练习 4：heap redo 的 block contract

阅读 `heapam_xlog.c` 中的 `heap_redo()`、`heap_xlog_insert()` 和 `heap_xlog_update()`。
标出 main data、block tag、block data 分别提供哪些信息。
回答：`xlrec->offnum` 与 block tag 怎样组成目标 TID，visibility map clear 为什么在 page action 判断之外，同页 update 为什么复用 buffer/action，prefix/suffix 优化为什么要求 old tuple 可读。

### 练习 5：index redo 的差异边界

阅读 `nbtxlog.c` 的 `btree_xlog_split()`、`gistxlog.c` 的 `gistRedoClearFollowRight()`、`ginxlog.c` 的 `ginRedoSplit()` 和 `ginRedoVacuumPage()`。
比较三类边界：btree right page 可由 WAL block data 初始化、GiST 在 `BLK_RESTORED` 后仍可能补做状态更新、GIN 某些 record 把 FPI restore 当作硬 contract。
回答：哪些 redo routine 只处理 `BLK_NEEDS_REDO`，哪些还处理 `BLK_RESTORED`，哪些要求必须 restore FPI，释放 buffer 的顺序为什么可能影响 Hot Standby 可见性。

## 19. 实验

这些实验以观察和验证为主。
不要修改 PostgreSQL 源码。

### 实验 1：列出当前基线的 rmgr 顺序

在 `/home/highgo/postgres` 执行：

```bash
nl -ba src/include/access/rmgrlist.h | sed -n '20,80p'
```

观察每个 `PG_RMGR` 的顺序。
然后执行：

```bash
rg -n "PG_RMGR|RM_HEAP_ID|RM_BTREE_ID|RM_GIN_ID|RM_GIST_ID" src/include/access/rmgr.h src/include/access/rmgrlist.h
```

目标：
- 确认 rmgr id 的枚举来自 `rmgrlist.h`。
- 确认具体 redo 函数名来自同一个 list。

### 实验 2：从 recovery loop 找到 redo 入口

执行：

```bash
rg -n "RmgrStartup|ApplyWalRecord|rm_redo|RmgrCleanup" src/backend/access/transam/xlogrecovery.c
```

按输出顺序阅读相关函数。
目标：
- 用一张纸画出 `ReadRecord -> ApplyWalRecord -> rm_redo -> ReadRecord` 循环。
- 标出 redo 前和 redo 后分别更新的 LSN 状态。

### 实验 3：观察 record decode 的访问边界

执行：

```bash
rg -n "DecodeXLogRecord|XLogRecGetData|XLogRecGetBlockData|RestoreBlockImage" src/backend/access/transam/xlogreader.c src/include/access/xlogreader.h
```

目标：
- 确认 decode 函数和 accessor 在同一抽象层。
- 找出 accessor 不做 rmgr payload 语义校验的地方。

## 20. 常见误区

- 把 xlogreader decode 当成 rmgr 语义校验。通用层只校验 WAL 外壳和 block reference，自定义 payload 是否合理仍由 rmgr redo 负责。
- 把 `BLK_DONE` 理解成“record 已经无意义”。它只说明这个 page 对普通 delta redo 不需要再次修改；某些 AM 仍可能在 `BLK_RESTORED` 或特殊状态上做额外收尾。
- 把 FPI 看成性能优化。带 `BKPIMAGE_APPLY` 的 FPI 是 torn page 恢复基底，正确性优先于减少 redo 工作。
- 把 page LSN 当成唯一兜底。`WILL_INIT`、invalid page table、drop/truncate 和 rmgr 私有状态都可能改变判断边界。
- 把 `rmgr_id` 当成运行时模块名。它是 WAL 持久化格式的一部分，数值顺序和 record 解释能力必须长期兼容。

## 21. 讨论题

1. 为什么 `RmgrId` 是 WAL 持久化格式的一部分，而不是普通运行时 enum？
2. 通用 WAL 层已经 decode block reference，为什么仍不能解释 heap tuple 或 btree split 的业务语义？
3. `XLogReadBufferForRedo()` 返回 `BLK_DONE` 时，redo routine 还能不能修改这个 page？哪些特殊场景会挑战这个默认规则？
4. 为什么有 `BKPIMAGE_APPLY` 的 FPI 比数据文件里看起来更新的 page 更可信？
5. `WILL_INIT` contract 如果生成侧和 redo 侧不匹配，为什么不能靠 page LSN 判断兜底？
6. `pg_waldump` 能把 rmgr name 和 record type 展示出来，但为什么不能替代读 `rm_redo`？
7. recovery 中 invalid page table 是 fallback、延迟报错，还是 correctness 逃生门？达到一致状态后为什么边界改变？
8. 本节的可迁移规律是什么：怎样在通用日志框架和模块私有状态机之间划出可验证 contract？

## 22. 本节小结

`RmgrId` 是 WAL record header 的一部分。
它是持久化格式，不是普通运行时 enum。
内置 rmgr id 的数值由 `rmgrlist.h` 顺序决定。
`rmgr.c` 用同一个 list 生成 `RmgrTable`。
recovery 主循环在 `ApplyWalRecord()` 中调用 `GetRmgr(record->xl_rmid).rm_redo(xlogreader)`。
调用前，xlogreader 已经完成 WAL page 读取、record header 校验、CRC 校验和 record decode。
redo routine 通过 xlogreader accessor 读取 main data、block data、block tag 和 image metadata。
它不需要解析 WAL 外壳。
但它必须解释自己的 payload。
FPI 应用由 `XLogReadBufferForRedoExtended()` 和 `RestoreBlockImage()` 负责。
有 `BKPIMAGE_APPLY` 的 FPI 会优先 restore，即使数据页看起来更新。
这是为了防 torn page。
无 FPI 时，幂等判断使用 record `EndRecPtr` 和 page LSN。
`EndRecPtr <= PageGetLSN(page)` 返回 `BLK_DONE`。
否则返回 `BLK_NEEDS_REDO`。
redo routine 修改 page 后必须设置 page LSN 并 mark dirty。
`XLogInitBufferForRedo()` 和 `BKPBLOCK_WILL_INIT` 构成 page 初始化 contract。
WAL record 声明会初始化，redo routine 就必须 zero/init。
redo routine 要 zero/init，WAL record 也必须声明。
heap redo 展示了 tuple 重建、visibility map 修正和逐 block 幂等判断。
btree redo 展示了多页 split、page initialization 和 Hot Standby 可见性边界。
GiST 展示了 `BLK_RESTORED` 后仍需补做状态更新的场景。
GIN 展示了某些 record 对 FPI restore 的硬要求。
错误边界上，通用 WAL 层负责 record 格式和 CRC。
xlogutils 负责 block reference、FPI、buffer、LSN 和 invalid page table。
rmgr 负责 opcode 和 payload 语义。
任何一层发现 contract 破坏，都不能静默跳过。
这就是 WAL resource manager redo contract 的核心。
