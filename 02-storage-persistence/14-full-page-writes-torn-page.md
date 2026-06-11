# PostgreSQL Full page image、torn page 与 hint bit WAL 边界

## 课程定位

前置知识：已理解 WAL record block reference、page LSN、checkpoint redo pointer、`FlushBuffer()` 的 WAL-before-data gate，以及 checksum 只是检测机制不是恢复机制。

本节唯一主问题：

```text
为什么增量 redo 已经存在，PostgreSQL 仍要在某些 page 修改中写入 full-page image？
```

核心矛盾：增量 WAL 可以控制体积；但 crash 时磁盘 page 可能是 torn state，增量 redo 不能假设它仍是合法旧 page。

一句话运行模型：

```text
checkpoint 推进 RedoRecPtr 后，page 第一次 WAL-logged 修改若发现 page_lsn <= RedoRecPtr，就在 XLogRecordAssemble() 中写入 full-page image；recovery 侧优先 restore 可信 FPI，再处理 rmgr redo 边界，hint bit 在 checksum/wal_log_hints 场景下复用同一物理安全边界。
```

学完后应能判断：FPI 为什么防 torn page；checkpoint 后第一次修改为什么特殊；`XLogRegisterBuffer()` flags 如何影响 FPI；hint bit 为什么可能写 `XLOG_FPI_FOR_HINT`；FPI 的 WAL 体积成本在哪里。

源码基线：本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。

## 1. 本节在总主线中的位置

上一组 WAL 课程已经讲了 record、insertion、flush 和 segment。本节回到 page 层，解释为什么有了 page LSN 和增量 redo 之后，PostgreSQL 仍然需要在一些修改中把整页镜像写进 WAL。

这节是后续 redo contract 的关键前置：redo routine 不能总是假设磁盘 page 是合法旧版本。FPI 提供的是一个比数据文件当前内容更可信的页面基底。

## 2. 核心矛盾与一句话运行模型

如果 crash 发生在 8KB page 写到一半时，磁盘上的 page 可能既不是旧版本也不是新版本。增量 redo 通常假设 page 至少结构合法；torn page 破坏了这个前提。

因此本节模型是：

```text
checkpoint redo pointer 之后
  -> page 第一次 WAL 修改
  -> XLogRecordAssemble() 判断需要 FPI
  -> WAL record 携带 block image
  -> recovery 侧优先 restore FPI
  -> 后续增量 redo 在可信基底上继续
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节读什么 |
| --- | --- | --- |
| 1 | `src/backend/access/transam/xloginsert.c` | `XLogRegisterBuffer()` flags、`XLogRecordAssemble()`、FPI 判定和 block image 生成。 |
| 2 | `src/backend/access/transam/xlog.c` | `RedoRecPtr`、checkpoint 后 FPI 边界、`full_page_writes` 变化。 |
| 3 | `src/backend/storage/buffer/bufmgr.c` | `MarkBufferDirtyHint()`、hint bit 与 FPI-for-hint。 |
| 4 | `src/backend/access/heap/heapam_visibility.c` | committed hint bit 设置和 commit LSN 边界。 |
| 5 | `src/include/storage/bufpage.h` | page header、`pd_lsn`、checksum、标准 page layout。 |
| 6 | `src/include/access/xlogrecord.h` | WAL block image 的 record 表达。 |
| 7 | `src/backend/access/transam/xlogutils.c`、`src/backend/access/transam/xlogreader.c` | recovery 侧如何读取并应用 FPI。 |
| 8 | `src/backend/storage/page/bufpage.c`、`src/backend/storage/page/README` | checksum 与 page 物理格式辅助核对。 |

## 4. 关键结论：FPI 与 torn page

`full_page_writes` 的核心目标不是让每一次修改都更容易 redo。

它的核心目标是防 torn page。

torn page 指一个数据库 page 写入磁盘时只落下了部分新内容。

PostgreSQL 常见 page 大小是 8KB，但底层设备、文件系统、缓存层或虚拟化存储可能按更小粒度完成写入。

如果系统在 8KB page 写到一半时崩溃，磁盘上的 page 可能是旧半页加新半页。

这个 page 既不是修改前版本，也不是修改后版本。

它可能有 page header、line pointer 和 tuple bytes，但这些结构彼此不再一致。

普通 WAL redo 通常是增量 redo。

它假设磁盘 page 至少是某个合法旧版本。

如果 page 已经 torn，redo routine 往往不知道如何从这个混合状态修起。

Full page image，简称 FPI，就是 WAL record 中附带的整页快照。

恢复时如果 WAL block reference 带有可应用的 FPI，recovery 直接用它覆盖数据库文件里的 page。

这样即使磁盘上的 page 是 torn 的，恢复也不用信任它。

PostgreSQL 不会每次修改都写 FPI。

默认策略是：checkpoint 后，一个 page 的第一次 WAL-logged 修改需要 FPI。

后续同一 checkpoint 周期内，该 page 的修改通常不再需要 FPI。

源码中的核心条件在 `xloginsert.c:692-695`：

```c
needs_backup = (page_lsn <= RedoRecPtr);
```

`page_lsn` 来自 page header 的 `pd_lsn`。

`RedoRecPtr` 是当前 checkpoint 的 redo pointer。

如果 page LSN 小于等于 redo pointer，说明这页在当前 redo point 之后还没有被 WAL 保护过。

这次修改要带 FPI。

如果 page LSN 大于 redo pointer，说明这页已经在当前 checkpoint 周期内被 WAL 修改过。

这次通常只需要记录增量 WAL。

这就是本节最重要的心智模型：

FPI 不是“每次写整页”。

FPI 是“每个 checkpoint 周期里，每个 page 的首次 WAL 修改提供可信页面基底”。

## 5. page header、LSN 与 checksum

`bufpage.h` 定义了 PostgreSQL 标准 page header。

`PageHeaderData` 位于 `src/include/storage/bufpage.h:184-197`。

其中最前面的字段是 `pd_lsn`。

注释 `bufpage.h:143-154` 说明：`pd_lsn` 标识最近一次修改该 page 的 WAL record，dirty buffer 写出前必须先把 WAL flush 至少推进到 page LSN。

这就是 WAL-before-data。

`PageGetLSN()` 和 `PageSetLSN()` 在 `bufpage.h:410-420`。

普通 WAL 修改完成后，调用方通常把 `XLogInsert()` 返回的 record end LSN 写入 page header。

`xloginsert.c:475-479` 也强调：`XLogInsert()` 返回值可以作为受影响数据页的 LSN，它表示数据页写出前 WAL 必须 flush 到哪里。

同一个 page header 里还有 `pd_checksum`。

checksum 不是 WAL LSN。

checksum 是数据页写出 shared buffers 前计算的 page-level 校验。

`bufpage.h:156-165` 特别说明：page 自身没有 flag 表示 checksum 是否有效，是否验证取决于集群级 checksum 状态。

`PageSetChecksum()` 在 `bufpage.c:1518-1529`。

它只在 page 已初始化，并且 `DataChecksumsNeedWrite()` 为 true 时计算 checksum。

`DataChecksumsNeedWrite()` 在 `xlog.c:4674-4679`。

它在 checksum 开启、正在开启、正在关闭但仍需写 checksum 的状态下返回 true。

这里要分清两个校验：

- WAL record 有自己的 CRC。
- 数据 page 有自己的 checksum。

FPI 存在 WAL record 里，由 WAL CRC 保护。

FPI replay 时不依赖 page checksum。

`src/backend/storage/page/README:26-29` 说明：WAL-logged changes 不更新 page checksum，full page images 有 WAL CRC 保护，WAL replay 不应该检查 FPI 的 page checksum。

checksum 可以检测 page 损坏。

checksum 自己不能修复 torn page。

修复需要 WAL 中有足够内容。

对普通数据页修改来说，这个足够内容通常来自 checkpoint 后第一次修改时的 FPI。

## 6. checkpoint 后第一次修改

checkpoint 让恢复可以从某个 redo point 开始，而不是从数据库创建时开始 replay。

在线 checkpoint 不是瞬间完成的。

`CreateCheckPoint()` 的注释在 `xlog.c:7378-7387` 说明：在线 checkpoint 会先插入 `XLOG_CHECKPOINT_REDO`，这个 record 的位置成为 redo point，随后系统继续写 WAL，checkpointer 慢慢把 buffers 刷到磁盘，最后写 `XLOG_CHECKPOINT_ONLINE`。

这意味着 redo point 确定后，checkpoint 仍在进行。

其他 backend 仍可继续修改页面并写 WAL。

这些后续修改不能假设自己已经包含在 checkpoint 中。

所以 RedoRecPtr 必须及时推进。

shutdown checkpoint 的注释在 `xlog.c:7548-7558` 说得很直接：如果 checkpoint 失败，RedoRecPtr 指得太靠后只会导致多写一些 FPI；但不能延后推进，因为 checkpoint dump buffers 期间发生的 XLogInsert 必须假设自己的 buffer changes 不在 checkpoint 中。

在线 checkpoint 中，`XLOG_CHECKPOINT_REDO` 走 `XLogInsertRecord()` 的特殊分支。

`xlog.c:929-940` 在持有全部 WAL insertion locks 时，把 `RedoRecPtr` 和 `Insert->RedoRecPtr` 设置为 record start position。

`xlog.c:7570-7605` 随后把这个值复制到 checkpoint record 和 `XLogCtl->RedoRecPtr`。

一旦 RedoRecPtr 前进，后续 WAL insert 就会重新以这个 redo point 判断 FPI。

如果某个 page 的 `pd_lsn <= RedoRecPtr`，说明它在这个 redo point 之后还没有新的 WAL 修改。

这次修改要带 FPI。

如果没有 FPI，crash 后从这个 redo point 开始恢复时，recovery 可能先遇到一个增量 record，却只能在磁盘上的 torn page 上重做。

这就是 checkpoint 后第一次修改特殊的原因。

注意这里的“第一次”不是从上次 page 落盘以来第一次。

它是从当前 checkpoint redo point 以来第一次 WAL-logged 修改。

## 7. `full_page_writes`、`doPageWrites` 与重试

用户看到的 GUC 是 `full_page_writes`。

源码里的全局变量是 `fullPageWrites`。

`xlog.c:129` 默认设置为 true。

WAL insert 路径实际使用 `doPageWrites`。

`xlog.c:283-293` 注释说明：`doPageWrites` 是 backend-local copy，表示 `fullPageWrites || runningBackups > 0`。

也就是说，即使用户 GUC 是 off，某些 backup 场景仍可能强制 full-page writes。

`GetFullPageWriteInfo()` 在 `xlog.c:6967-6970`。

它把本 backend 缓存的 `RedoRecPtr` 和 `doPageWrites` 返回给 `xloginsert.c`。

这个缓存可能过期。

所以 `XLogRecordAssemble()` 先用它组装一次 record。

真正插入前，`XLogInsertRecord()` 会在拿到 WAL insertion lock 后复核。

`xlog.c:846-848` 注释说明：持有 insertion lock 时，`RedoRecPtr` 和 `fullPageWrites` 不会在插入完成前变化。

复核逻辑在 `xlog.c:878-887`。

如果共享 RedoRecPtr 更新了，刷新 backend-local copy。

重新计算 `doPageWrites = Insert->fullPageWrites || Insert->runningBackups > 0`。

如果现在需要 full-page writes，而刚才组装 record 时漏掉了应该备份的页，就释放 lock，退出 critical section，并返回 `InvalidXLogRecPtr`。

`XLogInsert()` 在 `xloginsert.c:512-535` 外层是 do-while。

收到 invalid 后，它重新调用 `XLogRecordAssemble()`。

这个重试机制保护 checkpoint 并发。

它允许无锁阶段先做一次便宜判断，同时确保最终插入的 WAL record 符合最新 RedoRecPtr 和 full-page-writes 状态。

## 8. `XLogRegisterBuffer()` 的职责

WAL record 构造从 `XLogBeginInsert()` 开始。

调用方注册 main data、block data 和 buffer references。

最后调用 `XLogInsert()`。

block 级别最重要的入口是 `XLogRegisterBuffer()`。

源码位置：`xloginsert.c:246-310`。

注释 `xloginsert.c:241-244` 说：每个被 WAL-logged operation 修改的 page 都必须注册。

这里容易误解。

`XLogRegisterBuffer()` 不等于“这个 buffer 一定写 FPI”。

它只是把 buffer 注册为当前 WAL record 的 block reference。

是否带 FPI，由后面的 `XLogRecordAssemble()` 根据 flags、`doPageWrites`、RedoRecPtr 和 page LSN 判断。

`XLogRegisterBuffer()` 的断言也很重要。

`xloginsert.c:254-269` 要求：普通情况下，buffer 应该已经 dirty，并且当前 backend 持有 exclusive 或 share-exclusive content lock。

这对应标准修改顺序：

1. 修改 page。
2. `MarkBufferDirty()`。
3. 注册 buffer。
4. 插入 WAL。
5. 用返回 LSN 设置 page LSN。

hint bit 路径允许 share-exclusive lock。

因为 hint bit 可以在读路径发生，且走 `MarkBufferDirtyHint()` 的特殊协议。

## 9. `XLogRegisterBuffer` flags

flags 定义在 `src/include/access/xloginsert.h:31-41`。

这是调用方传给 WAL 构造器的 intent。

`xlogrecord.h` 里定义的是 WAL record 物理格式中的 block flags，不是调用方 flags。

`REGBUF_FORCE_IMAGE`：强制记录 full-page image。

`log_newpage()` 在 `xloginsert.c:1197-1203` 使用它，并插入 `RM_XLOG_ID, XLOG_FPI`。

`REGBUF_NO_IMAGE`：禁止记录 full-page image。

即使 `full_page_writes` 开启，也不为该 block 写 FPI。

调用方必须能证明 redo 不需要旧 page image。

`REGBUF_WILL_INIT`：redo 时该 page 会被重新初始化。

它隐含 `REGBUF_NO_IMAGE`，定义是 `(0x04 | 0x02)`。

`XLogRecordAssemble()` 在 `xloginsert.c:714-715` 把它翻译为 `BKPBLOCK_WILL_INIT`。

redo 侧 `XLogReadBufferForRedoExtended()` 在 `xlogutils.c:362-371` 检查这个标志，要求 WILL_INIT 与 zero mode 初始化严格匹配。

`REGBUF_STANDARD`：page 遵循标准 slotted page layout。

这允许 FPI 省略 `pd_lower` 和 `pd_upper` 之间的 hole。

相关逻辑在 `xloginsert.c:732-750`。

如果 `pd_lower/pd_upper` 不合法，就不省略 hole。

`REGBUF_KEEP_DATA`：即使已经写 FPI，也保留 block-specific data。

默认情况下，如果 FPI 会被应用，该 block 的增量 data 可以省略。

某些 record 仍需要保留 data 供 redo、decoding 或其他逻辑使用。

`REGBUF_NO_CHANGE`：故意注册 clean buffer，并且不会更新该 page 的 LSN。

它主要绕过 `XLogRegisterBuffer()` 的 dirty 和 lock 断言。

普通修改不要用它。

一个简化表：

| flag | 作用 |
| --- | --- |
| `REGBUF_FORCE_IMAGE` | 强制 FPI |
| `REGBUF_NO_IMAGE` | 禁止 FPI |
| `REGBUF_WILL_INIT` | redo 会初始化 page，隐含 no image |
| `REGBUF_STANDARD` | 允许省略标准 page hole |
| `REGBUF_KEEP_DATA` | 有 FPI 时仍保留 block data |
| `REGBUF_NO_CHANGE` | 注册 clean buffer 且不更新 LSN |

## 10. `XLogRecordAssemble()` 的 FPI 判定

`XLogRecordAssemble()` 在 `xloginsert.c:620-940`。

它遍历所有已注册 block。

对每个 block 先判断 `needs_backup`。

核心逻辑在 `xloginsert.c:678-700`：

```text
if FORCE_IMAGE:
    needs_backup = true
else if NO_IMAGE:
    needs_backup = false
else if !doPageWrites:
    needs_backup = false
else:
    needs_backup = PageGetLSN(page) <= RedoRecPtr
```

如果不需要 backup，它还会用这个 page LSN 更新 `fpw_lsn` 的最小值。

`fpw_lsn` 是重试机制的一部分。

如果 record 中有 page 没带 FPI，`fpw_lsn` 记录这些 page 中最老的 page LSN。

`XLogInsertRecord()` 后面如果发现 RedoRecPtr 已经推进到覆盖这个 `fpw_lsn`，就说明刚才组装 record 时漏了 FPI。

它返回 invalid，让 `XLogInsert()` 重新组装。

接着判断 `needs_data`。

逻辑在 `xloginsert.c:702-708`：

- 没有 block data，就不需要 data。
- 如果 `REGBUF_KEEP_DATA`，保留 data。
- 否则只有不需要 backup 时才保留 data。

这解释了为什么普通 record 带可应用 FPI 时，常常可以省略该 block 的增量 data。

FPI 已经是该 WAL record 执行后的 page image。

如果不带 FPI，redo 才需要 rmgr-specific block data 在旧 page 上做增量修改。

`include_image` 的判断在 `xloginsert.c:717-721`：

```c
include_image = needs_backup || (info & XLR_CHECK_CONSISTENCY) != 0;
```

这行把两类 image 区分开：

- `needs_backup` 是 crash safety 的 FPI。
- `XLR_CHECK_CONSISTENCY` 是 replay consistency checking 的 image。

只有 `needs_backup` 为 true 时，`xloginsert.c:794-795` 才设置 `BKPIMAGE_APPLY`。

这意味着有 block image 不等于 redo 一定应用它。

## 11. WAL record 中的 block image

`src/include/access/xlogrecord.h` 定义 WAL record 格式。

固定头 `XLogRecord` 位于 `xlogrecord.h:41-53`。

它包含 `xl_tot_len`、`xl_xid`、`xl_prev`、`xl_info`、`xl_rmid` 和 `xl_crc`。

block reference 头 `XLogRecordBlockHeader` 位于 `xlogrecord.h:103-113`。

它包含 block id、fork number 与 block flags、block data length。

如果 block 带 full-page image，就跟一个 `XLogRecordBlockImageHeader`。

源码位置：`xlogrecord.h:141-151`。

`bimg_info` 的关键 bits 位于 `xlogrecord.h:156-163`：

- `BKPIMAGE_HAS_HOLE`
- `BKPIMAGE_APPLY`
- `BKPIMAGE_COMPRESS_PGLZ`
- `BKPIMAGE_COMPRESS_LZ4`
- `BKPIMAGE_COMPRESS_ZSTD`

最关键的是 `BKPIMAGE_APPLY`。

redo 只把带 apply 的 image 当作恢复基底。

标准 page 的 FPI 可以省略 hole。

`REGBUF_STANDARD` 允许 `XLogRecordAssemble()` 用 `pd_lower/pd_upper` 识别 hole。

如果 `wal_compression` 开启，`xloginsert.c:759-768` 还会尝试压缩去掉 hole 后的 image。

压缩方法写入 `bimg_info`。

`xlogrecord.h:127-139` 说明：压缩能降低 WAL volume，但会增加 WAL logging 时的 CPU。

`XLogCompressBackupBlock()` 尾部在 `xloginsert.c:1082-1093` 会检查压缩是否真的节省空间。

如果压缩后加额外 header 不划算，就存原始 image。

## 12. FPI 与 redo 的关系

redo 侧的通用入口是 `XLogReadBufferForRedo()`。

源码位置：`xlogutils.c:266-308`。

注释说明：它读取 WAL record 引用的 block，判断是否需要 redo；如果 WAL record 包含 full-page image，就恢复它。

实际逻辑在 `XLogReadBufferForRedoExtended()`。

源码位置：`xlogutils.c:339-428`。

如果 `XLogRecBlockImageApply(record, block_id)` 为 true，`xlogutils.c:373-406` 会：

1. 读取或扩展目标 buffer。
2. 调用 `RestoreBlockImage()`。
3. 如果 page 不是 new page，设置 page LSN 为 record end LSN。
4. `MarkBufferDirty()`。
5. 返回 `BLK_RESTORED`。

如果没有可应用 image，`xlogutils.c:410-423` 会读取并锁住 page，然后比较 WAL record end LSN 与 page LSN。

如果 record end LSN `<= PageGetLSN(page)`，返回 `BLK_DONE`。

否则返回 `BLK_NEEDS_REDO`。

大多数 rmgr redo routine 只在 `BLK_NEEDS_REDO` 时做增量修改。

如果返回 `BLK_RESTORED`，说明该 WAL record 对这个 page 的结果已经由 FPI 恢复完成。

后续 WAL record 再继续增量 replay。

`RestoreBlockImage()` 在 `xlogreader.c:2095-2197`。

它负责解压 image，复制 page bytes，并在有 hole 时 zero-fill hole。

因此 FPI replay 不是把 WAL bytes 简单附加到 page。

它是按 block image metadata 重建完整 page。

这一点解释了 FPI 和 redo 的关系：

FPI 提供页面基底或本 record 的页面结果。

rmgr redo 提供 operation-level 的增量逻辑。

两者不是重复劳动。

## 13. `XLOG_FPI`、`XLOG_FPI_FOR_HINT` 与普通 record

FPI 可以作为普通 rmgr record 的 block image 出现。

例如 heap update 的 WAL record 既可能有 heap rmgr data，也可能给被修改 block 带 FPI。

这种 FPI 不是独立 record type。

它是 block reference 的一部分。

源码里也有专门的 `XLOG_FPI` record。

`log_newpage()` 在 `xloginsert.c:1190-1215`。

它用 `REGBUF_FORCE_IMAGE` 注册 block，然后插入 `RM_XLOG_ID, XLOG_FPI`。

`xlog_redo()` 在 `xlog.c:9060-9093` 处理 `XLOG_FPI` 和 `XLOG_FPI_FOR_HINT`。

`XLOG_FPI` record 只包含 block references。

每个 block reference 都必须包含 full-page image。

如果 `XLOG_FPI` 没有 image，redo 报错。

`XLOG_FPI_FOR_HINT` 是 hint bit 路径生成的 record。

`xlog.c:9068-9072` 注释说明：它只在 checksums 或 `wal_log_hints` enabled 时生成。

如果生成时 `full_page_writes` disabled，它可能没有 full-page image。

这种情况下 redo 没什么可做。

这说明：checksum 或 `wal_log_hints` 不会在 `full_page_writes=off` 时神奇提供 torn-page 修复能力。

## 14. `full_page_writes` 动态变化

`full_page_writes` 可以通过 SIGHUP 改变。

源码用 `UpdateFullPageWrites()` 更新共享状态和 WAL 状态。

位置：`xlog.c:8753-8811`。

这里的顺序很谨慎。

`xlog.c:8778-8783` 注释说明：多写 full-page images 总是安全的，少写不安全。

如果把 `full_page_writes` 设为 true：

先把共享 `Insert->fullPageWrites` 设为 true。

再写 `XLOG_FPW_CHANGE`。

如果把它设为 false：

先写 `XLOG_FPW_CHANGE`。

再把共享 flag 设为 false。

这样 archive recovery 能正确跟踪状态变化。

`xlog_redo()` 在 `xlog.c:9146-9167` replay `XLOG_FPW_CHANGE`。

recovery 中的 `lastFullPageWrites` 在 `xlog.c:219-224` 定义。

它从 starting checkpoint record 初始化，随后由 `XLOG_FPW_CHANGE` 更新。

所以 `full_page_writes` 不只是本地 GUC。

它是 WAL 流中 recovery 端需要理解的状态。

## 15. hint bit 的来源

heap tuple 可见性判断集中在 `heapam_visibility.c`。

文件开头注释 `heapam_visibility.c:6-11` 说明：`HeapTupleSatisfies*` 例程在看到插入或删除事务已经 commit 或 abort，并且安全时，会更新 tuple hint bits；如果 hint bits 改了，就调用 `MarkBufferDirtyHint()`。

设置 hint bit 的核心 helper 是 `SetHintBitsExt()`。

源码位置：`heapam_visibility.c:142-192`。

它写 tuple header 的 `t_infomask`。

典型 hint bits 包括：

- `HEAP_XMIN_COMMITTED`
- `HEAP_XMIN_INVALID`
- `HEAP_XMAX_COMMITTED`
- `HEAP_XMAX_INVALID`

这些 bits 缓存事务状态。

它们让后续可见性判断少查 `pg_xact`。

例如 `HeapTupleSatisfiesSelf()` 中，xmin commit 后会设置 `HEAP_XMIN_COMMITTED`，位置在 `heapam_visibility.c:347-349`。

如果判断 xmin aborted 或 crashed，会设置 `HEAP_XMIN_INVALID`，位置在 `heapam_visibility.c:352-354`。

这些修改可能发生在 SELECT 读取 tuple 时。

所以读路径也可能脏页。

这就是 hint bit 与普通数据修改的第一个差别。

## 16. committed hint bit 的 WAL flush 边界

hint bit 逻辑上可重建，但 committed hint bit 不能破坏 WAL-before-data。

`SetHintBitsExt()` 的注释在 `heapam_visibility.c:115-122` 说明：只有当事务 commit record 保证会先于 buffer 写出被 flushed，或者 table 是 crash 后会丢弃的 temporary/unlogged table 时，才可以设置 committed hint bit。

源码在 `heapam_visibility.c:152-165`。

如果 `xid` 有效，并且 buffer 是 permanent：

先取得 `commitLSN = TransactionIdGetCommitLSN(xid)`。

然后检查：

```c
XLogNeedsFlush(commitLSN) &&
BufferGetLSNAtomic(buffer) < commitLSN
```

如果条件成立，直接 return，不设置 hint bit。

含义是：

commit WAL 还没 flush。

并且这个 page 自己的 LSN 还不够新，无法在 page flush 时顺便逼迫 commit WAL flush。

在这种情况下，设置 committed hint bit 会让数据页可能先于 commit record 落盘。

所以源码选择暂时不设置 hint。

如果 page LSN 已经大于等于 commit LSN，未来写出 page 时会先 flush WAL 到 page LSN，自然包含 commit record。

这就是 page LSN interlock。

aborted/invalid hint bit 不需要这个 commit LSN 检查。

`heapam_visibility.c:124-125` 注释说明：marking a transaction aborted 时总是可以设置 hint bits。

因为 crash recovery 本来就会把 crash 时仍 in-progress 的事务视为 aborted。

## 17. 设置 hint bits 的 buffer 锁协议

`SetHintBitsExt()` 有单个 tuple 模式和批量模式。

如果 `state == NULL`，它调用 `BufferSetHintBits16()`。

位置：`heapam_visibility.c:168-178`。

`BufferSetHintBits16()` 在 `bufmgr.c:7102-7135`。

它在 shared buffer 上调用 `SharedBufferBeginSetHintBits()`，成功后写 16-bit infomask，再调用 `MarkSharedBufferDirtyHint()`。

批量模式会先调用 `BufferBeginSetHintBits()`。

成功后多个 tuple hint bits 可以共用一次锁升级成本。

最后 `BufferFinishSetHintBits()` 视需要调用 `MarkBufferDirtyHint()`。

`SharedBufferBeginSetHintBits()` 在 `bufmgr.c:6960-7019`。

如果当前已经持有 exclusive 或 share-exclusive lock，就允许设置。

如果只持有 share lock，就尝试升级到 share-exclusive。

`bufmgr.c:7035-7038` 注释说明：要求 share-exclusive lock 是为了防止 setting hint bits 与 page flush 并发，否则可能破坏 page checksum。

flush buffers 也需要 share-exclusive lock。

这意味着当前基线通过 content lock 等级让 hint setting 与写出互斥。

这个边界也解释了为什么本基线不需要旧资料中的 `BM_JUST_DIRTIED` 来处理 hint dirty 与 flush race。

## 18. `MarkBufferDirtyHint()` 的语义

`MarkBufferDirtyHint()` 在 `bufmgr.c:5815-5848`。

它和 `MarkBufferDirty()` 相似，但语义更弱。

注释列出三个差异：

- 调用方没有写普通 WAL。
- 如果 checksums enabled，可能需要写 `XLOG_FPI_FOR_HINT` 防 torn page。
- 调用方可能只有 share-exclusive lock。
- 这个函数不保证总能把 buffer 标成 dirty。

最后一点是关键。

普通数据修改不能悄悄失败。

hint bit 是非关键修改。

在 recovery 中或 relation skipping WAL 时，如果不能安全写 WAL，可以只在内存里设置 hint，但不 dirty page。

这样 hint 可能丢失。

但后续仍可重新查事务状态推导。

shared buffer 的实际逻辑在 `MarkSharedBufferDirtyHint()`。

位置：`bufmgr.c:5704-5812`。

它先要求 buffer pinned，并且持有 exclusive 或 share-exclusive lock。

如果 buffer 已经 dirty，就快速返回。

这是 `bufmgr.c:5716-5725` 的性能优化。

同一页设置多个 hint bits 很常见。

如果 page 已经 dirty，说明它在当前 checkpoint 周期已经走过普通 WAL 或 hint FPI 的保护路径。

不需要每个 hint bit 都再写 WAL。

## 19. hint bit 什么时候写 WAL

`XLogHintBitIsNeeded()` 定义在 `src/include/access/xlog.h:115-123`。

它返回：

```c
wal_log_hints || DataChecksumsNeedWrite()
```

注释说明：通常不 WAL-log hint bit updates；但 checksums enabled 时，必须保护它们免受 torn page writes；如果 `wal_log_hints=on`，也要 WAL-log。

这不是说每次 hint bit update 都写 WAL。

真正写 WAL 需要同时满足更多条件。

`MarkSharedBufferDirtyHint()` 在 `bufmgr.c:5740` 先判断：

```c
XLogHintBitIsNeeded() && (lockstate & BM_PERMANENT)
```

也就是 checksums 或 `wal_log_hints` 要求保护，并且 buffer 属于 permanent relation。

如果当前在 recovery，或者 relation 正在 skipping WAL，`bufmgr.c:5750-5752` 直接 return。

注释说明：可以设置 hint，但不能因为 hint dirty page。

如果可以写 WAL，就先把 buffer 标成 dirty。

位置：`bufmgr.c:5757-5776`。

顺序必须是先 dirty，再写 WAL。

这样 checkpoint 不会在写 WAL 和标 dirty 之间漏掉这个 buffer。

随后调用 `XLogSaveBufferForHint()`。

位置：`bufmgr.c:5785-5786`。

`XLogSaveBufferForHint()` 在 `xloginsert.c:1133-1176`。

它先调用 `GetRedoRecPtr()`，再读 page LSN。

如果 `lsn <= RedoRecPtr`，它插入 `RM_XLOG_ID, XLOG_FPI_FOR_HINT`。

如果 page LSN 已经大于 RedoRecPtr，就不写 WAL。

因为该 page 在本 checkpoint 周期已经被 WAL 保护过。

如果 `XLogSaveBufferForHint()` 返回 valid LSN，`MarkSharedBufferDirtyHint()` 在 `bufmgr.c:5788-5805` 设置 page LSN。

设置时持有 buffer header lock，以便 `BufferGetLSNAtomic()` 能 tear-free 地读取。

合并成判定表：

| 场景 | 是否可能写 hint WAL |
| --- | --- |
| checksums off 且 `wal_log_hints=off` | 通常不写 |
| page 已经 dirty | 不额外写 |
| temporary/local buffer | 不走 permanent WAL 保护 |
| permanent clean page 且 checksums on | 可能写 |
| permanent clean page 且 `wal_log_hints=on` | 可能写 |
| recovery 中不能写 WAL | 不 dirty page |
| page LSN 已经大于 RedoRecPtr | 不写 FPI-for-hint |

如果 `full_page_writes` effective on，`XLOG_FPI_FOR_HINT` 通常携带可应用 FPI。

如果 `full_page_writes` effective off，它可能没有 image。

`xlog.c:9068-9072` 对此有明确注释。

## 20. checksum、hint bit 与 torn page

没有 checksum 时，hint bit 的 torn write 通常只是性能问题。

某些 hint bits 落盘、某些没有，后续仍可查 `pg_xact`。

但 checksums enabled 时，hint bit 改变 page bytes。

如果 page 写出时发生 torn page，磁盘上可能出现“部分新 hint bits + 旧 checksum”或“新 checksum + 部分旧内容”。

下次读入时 checksum 不匹配。

一个逻辑上可丢的 hint 就变成 page-level corruption。

所以 checksum 会让 hint bit dirty clean permanent page 时需要 torn-page 保护。

这就是 `XLogHintBitIsNeeded()` 把 `DataChecksumsNeedWrite()` 纳入条件的原因。

`wal_log_hints` 则是人为要求 hint bit changes 也进入 WAL 保护边界。

它常用于没有 data checksums 但需要 `pg_rewind` 的集群。

成本是额外 WAL。

尤其 checkpoint 后第一次大范围扫描旧 tuple 时，可能产生明显 `XLOG_FPI_FOR_HINT` 和 FPI bytes。

## 21. 性能与体积权衡

FPI 的直接成本是 WAL 体积。

默认 8KB page 上，一次普通 heap update 的增量 WAL 可能只有几十到几百字节。

如果它是 checkpoint 后该 page 的第一次修改，WAL record 可能额外带接近 8KB 的 page image。

`REGBUF_STANDARD` 可以省略标准 page 的 hole。

`wal_compression` 可以压缩 FPI。

但高写入压力下，FPI 仍可能是 WAL volume 的主要来源。

FPI 体积与 checkpoint 频率强相关。

checkpoint 越频繁，RedoRecPtr 越频繁推进。

同一批 hot pages 在每个 checkpoint 周期后都会再次迎来“第一次修改”。

这会造成 checkpoint 后的 FPI burst。

checkpoint 间隔更长，单个周期内同一 page 多次修改通常只需要一次 FPI。

但 checkpoint 间隔太长会增加 crash recovery 需要 replay 的 WAL，也会增加 WAL 保留压力。

`wal_compression` 的权衡是用 CPU 换 WAL bytes。

page 内容越可压缩，收益越明显。

page 内容随机度高时，收益变小。

`wal_log_hints` 和 data checksums 会让 hint bit 路径产生 WAL。

这可能让看似只读的查询增加 WAL records、WAL bytes 和 dirty buffers。

关闭 `full_page_writes` 可以降低 WAL 体积。

但它是在放弃 PostgreSQL 对 torn page 的通用保护。

checksum 不能替代它。

checksum 可以让损坏更早暴露，但没有 FPI 时不一定能修复。

可靠性默认选择仍然是保持 `full_page_writes=on`。

优化优先级通常应放在 checkpoint 参数、`wal_compression`、批量写入方式和 workload 形态上。

## 22. 源码跟读练习

练习一：FPI 判定链路。

阅读 `src/include/access/xloginsert.h:31-41`、`xloginsert.c:246-310`、`xloginsert.c:512-535`、`xloginsert.c:620-708`、`xlog.c:783-896`。

确认 `REGBUF_FORCE_IMAGE` 与 `REGBUF_NO_IMAGE` 的优先级，确认 `doPageWrites=false` 时 page LSN 不会触发 FPI，确认 `fpw_lsn` 为什么记录未备份 page 的最小 LSN。

把核心逻辑写成伪代码：

```text
if FORCE_IMAGE: backup
else if NO_IMAGE: no backup
else if !doPageWrites: no backup
else if PageGetLSN(page) <= RedoRecPtr: backup
```

继续追 `include_image = needs_backup || XLR_CHECK_CONSISTENCY`，再看只有 `needs_backup` 才设置 `BKPIMAGE_APPLY`。

练习二：checkpoint 与 redo。

checkpoint 侧读 `xlog.c:265-293`、`xlog.c:7378-7387`、`xlog.c:7513-7525`、`xlog.c:7570-7605`、`xlog.c:7656-7714`、`xlog.c:7744-7753`。

确认在线 checkpoint 先写 `XLOG_CHECKPOINT_REDO`，redo point 确定后才继续刷 buffers，刷 buffers 期间其他 backend 仍可写 WAL，因此 RedoRecPtr 推进会触发新 checkpoint 周期的第一批 FPI。

redo 侧读 `xlogrecord.h:156-163`、`xlogreader.c:1765-1833`、`xlogreader.h:423-426`、`xlogutils.c:266-301`、`xlogutils.c:373-406`、`xlogutils.c:410-423`、`xlogreader.c:2095-2197`。

确认 `BKPIMAGE_APPLY` 转成 `apply_image`，有 image 但没有 apply 不作为恢复基底，restore image 后返回 `BLK_RESTORED`，没有 image 时才比较 record end LSN 与 page LSN。

练习三：hint bit WAL 边界。

阅读 `heapam_visibility.c:1-11`、`heapam_visibility.c:102-140`、`heapam_visibility.c:142-192`、`bufmgr.c:6960-7019`、`bufmgr.c:7030-7048`、`bufmgr.c:7102-7135`、`bufmgr.c:5704-5812`、`xloginsert.c:1133-1176`、`xlog.c:9060-9093`。

确认 committed hint bit 为什么检查 commit LSN，aborted hint bit 为什么不需要有效 xid，share lock 为什么要升级到 share-exclusive，page 已经 dirty 时为什么不写 hint WAL，recovery 中为什么可以 set hint 但不 dirty page。

结论：hint bit 的逻辑正确性依赖事务状态可重查，物理安全性仍受 page write、checksum、torn page 约束。

## 23. 实验

实验一：观察 checkpoint 后 FPI。

使用默认 `full_page_writes=on`，先看 `pg_stat_wal`：

```sql
SELECT wal_records, wal_fpi, wal_bytes FROM pg_stat_wal;
CREATE TABLE fpi_demo(id int primary key, payload text);
INSERT INTO fpi_demo SELECT g, repeat('x', 100) FROM generate_series(1, 1000) g;
CHECKPOINT;
SELECT wal_records, wal_fpi, wal_bytes FROM pg_stat_wal;
UPDATE fpi_demo SET payload = repeat('y', 100) WHERE id BETWEEN 1 AND 10;
SELECT wal_records, wal_fpi, wal_bytes FROM pg_stat_wal;
UPDATE fpi_demo SET payload = repeat('z', 100) WHERE id BETWEEN 1 AND 10;
SELECT wal_records, wal_fpi, wal_bytes FROM pg_stat_wal;
```

预期：checkpoint 后第一次 UPDATE 更可能增加 `wal_fpi`；不做 checkpoint 的第二次 UPDATE 如果落在同一批已修改 page 上，通常不会再次为同一 page 增加 FPI。

具体数字受行分布、索引页、HOT update、autovacuum 和后台活动影响，看趋势即可。

实验二：比较 `wal_compression`。

分别设置 `wal_compression=off` 和 `wal_compression=on`，每次确认 `SHOW wal_compression; SHOW full_page_writes;`，执行同样 workload：

```sql
SELECT pg_stat_reset_shared('wal');
CHECKPOINT;
UPDATE fpi_demo SET payload = repeat('q', 100) WHERE id BETWEEN 1 AND 500;
SELECT wal_records, wal_fpi, wal_bytes FROM pg_stat_wal;
```

预期：`wal_fpi` 数量可能接近，但 compression on 时 `wal_bytes` 可能更小，CPU 时间可能略高。

实验三：hint bit 让 SELECT 产生 WAL。

在独立测试实例上做：方式一是初始化带 checksums 的集群，方式二是不开 checksums 但设置 `wal_log_hints=on` 并重启。

```sql
CREATE TABLE hint_demo(id int, payload text);
INSERT INTO hint_demo SELECT g, repeat('h', 100) FROM generate_series(1, 20000) g;
CHECKPOINT;
SELECT pg_stat_reset_shared('wal');
SELECT count(*) FROM hint_demo;
SELECT wal_records, wal_fpi, wal_bytes FROM pg_stat_wal;
```

预期：如果扫描设置了 committed hint bits，并且 page 原本 clean，可能看到 WAL 增加；马上再读一次通常新增 WAL 少很多。

这个实验受 tuple 是否已有 hint、autovacuum 是否先访问、page 是否已经 dirty 等因素影响，验证的是边界：SELECT 也可能因为 hint bit 产生 WAL。

## 24. 讨论题

1. 如果普通增量 redo 已经能重放 heap update，为什么 torn page 仍然需要 FPI？
2. `PageGetLSN(page) <= RedoRecPtr` 判断的是 page 内容、checkpoint 周期，还是 WAL flush 位置？
3. `REGBUF_WILL_INIT` 为什么可以绕开 FPI？它把正确性责任转移给了谁？
4. `BKPIMAGE_APPLY` 和 `BKPBLOCK_HAS_IMAGE` 为什么不能混为一谈？
5. checksum 能发现 page 损坏，为什么它不能替代 `full_page_writes`？
6. hint bit 逻辑上可重建，为什么在 checksum 或 `wal_log_hints` 下仍可能产生 WAL？
7. 用 `pg_stat_wal.wal_fpi` 和 `pg_waldump` 观察 FPI 时，哪些结论受 workload、page 分布和后台活动影响？
8. 本节的可迁移规律是什么：什么时候增量日志必须补一个可信 base image？

## 25. 常见误区与误解

误解一：FPI 是每次 UPDATE 都写整页。

不对。

默认逻辑是 checkpoint 后每个 page 第一次 WAL 修改需要。

误解二：checksum 可以替代 `full_page_writes`。

不对。

checksum 主要检测 page 损坏，FPI 提供恢复时可用的完整 page image。

误解三：hint bit 不重要，所以永远不写 WAL。

不对。

hint bit 逻辑上可丢，但它仍然修改 page bytes。

在 checksum 或 `wal_log_hints` 场景下，clean permanent page 的 hint dirty 可能写 WAL。

误解四：有 block image 时 redo 一定应用它。

不对。

只有带 `BKPIMAGE_APPLY` 的 image 才作为恢复基底。

`XLR_CHECK_CONSISTENCY` 也可能让 record 带 image，但那是用于一致性检查。

误解五：`REGBUF_STANDARD` 表示 page 一定安全。

不对。

它只是允许 WAL 构造器按标准 page layout 省略 hole。

如果 `pd_lower/pd_upper` 不合法，源码会放弃 hole 优化。

误解六：`wal_log_hints=on` 只影响 standby。

不对。

它影响 primary 上 hint bit dirty 的 WAL 行为。

## 26. 本节小结

`full_page_writes` 是 PostgreSQL 针对 torn page 的默认保护。

它把 checkpoint 后某个 page 的第一次 WAL-logged 修改扩展成带 full-page image 的 WAL record。

FPI 判定核心是：

```text
doPageWrites && PageGetLSN(page) <= RedoRecPtr
```

RedoRecPtr 随 checkpoint 推进。

page LSN 随 page 修改对应的 WAL record 推进。

这个条件表达的是：这页在当前 checkpoint redo point 之后还没有被 WAL 保护过。

`XLogRegisterBuffer()` 只注册 block reference。

是否写 FPI 由 `XLogRecordAssemble()` 根据 flags、`doPageWrites`、RedoRecPtr 和 page LSN 决定。

`REGBUF_FORCE_IMAGE` 强制 FPI。

`REGBUF_NO_IMAGE` 禁止 FPI。

`REGBUF_WILL_INIT` 表示 redo 会重新初始化 page。

`REGBUF_STANDARD` 允许省略标准 page hole。

`REGBUF_KEEP_DATA` 让 block data 即使有 FPI 也保留。

redo 侧只应用带 `BKPIMAGE_APPLY` 的 image。

checksum 负责检测数据页损坏。

FPI 负责在恢复时提供完整 page image。

两者相关，但不能互相替代。

hint bit 是逻辑上可重建的缓存。

committed hint bit 需要 commit WAL flush 或 page LSN interlock。

checksum 或 `wal_log_hints` 打开时，hint dirty clean permanent page 可能写 `XLOG_FPI_FOR_HINT`。

`MarkBufferDirtyHint()` 的边界是：可以为了安全不 dirty page；page 已经 dirty 时不额外写 WAL；需要保护 clean permanent page 时先 dirty，再写 hint FPI，再设置 page LSN。

性能上，FPI 的主要成本是 WAL 体积和 checkpoint 后的 FPI burst。

`wal_compression` 可以换 CPU 降低 WAL bytes。

checkpoint 参数会改变 FPI 频率和 recovery 窗口。

`wal_log_hints` 和 checksums 会让 hint bit 路径产生额外 WAL。

可靠性默认选择是保持 `full_page_writes=on`。
