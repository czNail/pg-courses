# PostgreSQL smgr/md segment create、extend、truncate

## 课程定位
本节主题：PostgreSQL 的 storage manager 如何把一个 relation fork 映射成一组磁盘 segment 文件。 重点是 `smgr` API 到 `md` 实现之间的边界。 也就是从 `smgrcreate()`、`smgrextend()`、`smgrzeroextend()`、`smgrread()`、`smgrwrite()`、`smgrtruncate()`、`smgrdounlinkall()` 这些入口，走到 `md.c` 里真正的文件创建、扩展、读取、写入、截断和删除。
本节只讲 heap/index 等普通 relation 文件的 storage manager 侧生命周期。 不展开 buffer replacement。 不展开 WAL record 格式。
不展开 heap tuple、btree page split 等访问方法语义。 但会解释这些上层模块为什么依赖 `smgr/md` 提供清晰的错误边界和持久化边界。

本节唯一主问题：
一个 relation fork 可以持续扩展、被截断、被删除、被 redo 重放，为什么 PostgreSQL 不让上层直接操作文件，而是在 `smgr` 和 `md.c` 之间维护 segment 生命周期协议？

本节围绕的核心矛盾：
上层模块希望把 I/O 表达成“读写某个 fork 的 block”；底层文件系统只能处理有限大小的 OS 文件、fd 生命周期、短读短写、fsync、unlink race 和 crash 后重放。`smgr/md` 的职责就是把 block 语义映射成 segment 文件操作，同时把错误和持久化边界停在正确的位置。

读完本节，你应该能回答：
- `src/backend/storage/smgr/smgr.c` 为什么是 switch 层，而不是直接操作文件的层。
- 当前源码基线中 `smgr` 只有哪个 concrete storage manager。
- `smgr.h` 里的 `smgrread()` 和 `smgrwrite()` 为什么只是 inline wrapper。
- 当前基线中 `md.h` 是否存在，以及它暴露的是 `mdreadv/mdwritev` 还是 `mdread/mdwrite`。
- `SMgrRelation` 在 backend 本地缓存了什么。
- 为什么 `SMgrRelation` 的 `smgr_cached_nblocks` 当前只在 recovery 中可靠使用。
- relation fork 为什么会被拆成多个 segment 文件。
- 默认 1GB segment 从哪里来。
- block number 怎样换算成 segment number 和 segment 内偏移。
- segment 文件名怎样从 base relpath 扩展出 `.1`、`.2`。
- `mdcreate()` 创建了哪个 segment。
- `mdextend()` 怎样创建后续 segment。
- `mdzeroextend()` 为什么比循环 `mdextend()` 更适合批量扩展。
- `mdreadv()` 怎样处理短读、EOF 和 recovery 边界。
- `mdwritev()` 怎样处理短写和 fsync 注册。
- `mdtruncate()` 为什么把 inactive segment 截成 0，而不是直接 unlink。
- `mdunlink()` 为什么普通 main fork 的第一个 segment 常常延迟删除。
- `register_dirty_segment()`、`mdregistersync()`、`mdimmedsync()` 分别解决什么 durability 问题。
- `bulk_write.c` 为什么可以直接调用 `smgrextend()`，以及它怎样补齐 checkpoint 竞态。
- 哪些错误是 `ERROR`，哪些路径必须降级成 `WARNING`，哪些路径会 `FATAL` 或 `PANIC`。

## 源码基线
源码仓库：`/home/nail/postgres-lab` 基线：`master` = `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`

本节重点阅读：
- `src/backend/storage/smgr/smgr.c`
- `src/backend/storage/smgr/md.c`
- `src/backend/storage/smgr/bulk_write.c`
- `src/include/storage/smgr.h`
- `src/include/storage/md.h`

本节辅助核对：
- `src/include/storage/bulk_write.h`
- `src/include/storage/block.h`
- `src/include/pg_config.h.in`
- `configure.ac`
- `meson_options.txt`
- `meson.build`
- `src/backend/catalog/storage.c`
- `src/include/catalog/storage_xlog.h`
- `src/backend/access/transam/xlogutils.c`

本基线中的重要校正：
- `src/include/storage/md.h` 存在。
- 当前基线的 `md` 读写接口是 `mdreadv()` 和 `mdwritev()`。
- 当前基线没有导出 `mdread()` 和 `mdwrite()` 函数。
- `src/include/storage/smgr.h` 里有 `static inline smgrread()` 和 `static inline smgrwrite()`。
- 这两个 inline wrapper 分别调用 `smgrreadv(..., nblocks = 1)` 和 `smgrwritev(..., nblocks = 1)`。
- 所以本节讲“`mdread/mdwrite`”时，按当前源码校正为“`smgrread/smgrwrite` 的单页 wrapper，实际落到 `mdreadv/mdwritev`”。
- `smgr.c` 当前只有一个 `smgrsw[]` entry，也就是 `md.c`。
- `md.c` 的文件名历史上叫 magnetic disk，但注释已经说明它实际是 Unix-like filesystem storage manager。

---

## 1. 先给结论
`smgr` 是 relation file I/O 的公共入口。 它不直接拼文件名。 它不直接 `open()`、`write()`、`fsync()`。
它维护每个 backend 的 `SMgrRelation` 缓存，然后把操作分派到具体 storage manager。 当前源码基线里具体实现只有 `md.c`。 所以大多数调用路径长这样：

```text
buffer manager / catalog storage / redo / bulk writer
  -> smgr API
    -> smgrsw[reln->smgr_which]
      -> md.c
        -> fd.c / filesystem
```

`smgr.c` 的核心职责是四个。 第一，提供稳定的 storage manager API。 第二，用 backend-local hash table 缓存 `SMgrRelation`。
第三，统一处理中断保护、pin 生命周期、invalidation、cached size 更新。 第四，把操作分派到 `md.c`。

`md.c` 的核心职责也是四个。 第一，把 relation fork 映射成一个或多个 segment 文件。 第二，维护每个 fork 已打开 segment 的 `MdfdVec` 数组。
第三，对 create、extend、read、write、truncate、unlink 做真实文件操作。 第四，为普通 relation 的 dirty segment 注册 checkpoint fsync 或 unlink 请求。

segment 拆分的基本公式是：

```text
segment_size_bytes = RELSEG_SIZE * BLCKSZ
segno              = blocknum / RELSEG_SIZE
segoff_blocks      = blocknum % RELSEG_SIZE
seekpos_bytes      = BLCKSZ * segoff_blocks
```

默认构建中 `BLCKSZ = 8192`，`segsize = 1GB`，所以 `RELSEG_SIZE = 131072`。 `131072 * 8192 = 1073741824`，也就是 1GB。 源码不是在 `md.c` 里写死 1GB。
`RELSEG_SIZE` 是构建期配置值。 `pg_config.h.in` 说明 `RELSEG_SIZE * BLCKSZ` 是单个 relation segment 文件的最大大小。 `configure.ac` 和 `meson.build` 负责把 `--with-segsize` 或 `segsize` 选项换算成 blocks。
改变 `BLCKSZ` 或 `RELSEG_SIZE` 都要求 initdb，因为它们影响磁盘布局和 control file。

磁盘上的 segment 文件命名规则很简单。 第 0 个 segment 使用 fork 的基础路径。 第 1 个 segment 追加 `.1`。
第 2 个 segment 追加 `.2`。 以此类推。 `md.c` 中 `_mdfd_segpath()` 实现这个规则。

`md.c` 维护一个非常重要的不变量：
- 0 个或多个 full segment，每个正好 `RELSEG_SIZE` blocks。
- 接着 1 个 partial segment，大小范围是 `0 <= size < RELSEG_SIZE` blocks。
- partial segment 后面可以有任意多个 inactive segment。
- inactive segment 的大小是 0 blocks。
- active segment 是 full segment 加 partial segment。
- inactive segment 是曾经有数据、后来被 `mdtruncate()` 截成 0 的旧 segment。

这个不变量解释了很多看起来奇怪的行为。 `mdtruncate()` 不急着删除 inactive segment。 `mdunlink()` 删除 relation 时会先 truncate 再 unlink 后续 segment。
`mdregistersync()` 和 `mdimmedsync()` 会关心 active 和 inactive segment。 这些设计都是为了处理其他 backend、checkpointer 或 kernel 仍持有旧 fd 的情况。

---

## 2. 从 `smgr.h` 看公开 API
先读 `src/include/storage/smgr.h`。 `SMgrRelationData` 是 `SMgrRelation` 的实体。 它的第一个字段是 `smgr_rlocator`。
注释说明这个字段必须放在第一位，因为它是 hash table lookup key。

`smgr_rlocator` 是 `RelFileLocatorBackend`。 它同时包含 relation 的物理 identity 和 backend identity。 普通 permanent/unlogged relation 的 backend 是 `INVALID_PROC_NUMBER`。
temp relation 使用 backend-specific proc number。 这就是为什么 `SmgrIsTemp(smgr)` 只要检查 `smgr_rlocator`。

`SMgrRelationData` 中和本节最相关的字段有：
- `smgr_targblock`
- `smgr_cached_nblocks[MAX_FORKNUM + 1]`
- `smgr_which`
- `md_num_open_segs[MAX_FORKNUM + 1]`
- `md_seg_fds[MAX_FORKNUM + 1]`
- `pincount`
- `node`

`smgr_targblock` 是当前 insertion target block 的缓存。 它不是本节主线，但 truncate 和 release 时会被清掉。

`smgr_cached_nblocks` 是每个 fork 的最后已知大小。 `smgr.h` 的注释说这个值在 cache flush event 后重置为 `InvalidBlockNumber`。 它还特别强调：当前这个信息只在 recovery 中可靠。
原因是普通运行时没有针对 fork extension 的共享失效机制。 一个 backend 扩展 relation 后，另一个 backend 的 cached size 不会自动被所有正常路径更新。

`smgr_which` 是 storage manager selector。 当前 `smgr.c` 写着“we only have md.c at present”，新建 `SMgrRelation` 时设置为 0。 所以所有普通 relation I/O 都走 `smgrsw[0]`。

`md_num_open_segs` 和 `md_seg_fds` 是 `md.c` 私有字段。 它们按 fork 分组。 每个 fork 有一个当前已打开 segment 数量。
每个 fork 有一个 `MdfdVec` 数组。 这个数组不是一定包含磁盘上的全部 active segment。 它只包含本 backend 已经打开过、或 `mdnblocks()` 为了计算大小而打开过的 segment。

`pincount` 和 `node` 是 `smgr` 生命周期管理。 relcache 持有的 `SMgrRelation` 会 pin。 未 pin 的 entry 会挂在 `unpinned_relns` 链表上。
事务结束时，`AtEOXact_SMgr()` 会销毁所有未 pin 的 entry。 这避免 transient smgr entry 永远持有 fd。

`smgr.h` 中公开的本节核心 API 包括：
- `smgropen()`
- `smgrexists()`
- `smgrcreate()`
- `smgrdounlinkall()`
- `smgrextend()`
- `smgrzeroextend()`
- `smgrreadv()`
- `smgrwritev()`
- `smgrnblocks()`
- `smgrnblocks_cached()`
- `smgrtruncate()`
- `smgrimmedsync()`
- `smgrregistersync()`

还有两个 inline wrapper：

```c
smgrread(...)  -> smgrreadv(..., &buffer, 1)
smgrwrite(...) -> smgrwritev(..., &buffer, 1, skipFsync)
```

这说明当前源码已经把底层读写 API 向 vector I/O 迁移。 上层仍然可以用单页 `smgrread()` 和 `smgrwrite()`。 但真正的 dispatch 函数是 `smgrreadv()` 和 `smgrwritev()`。

---

## 3. `smgr.c` 是 switch 层
读 `src/backend/storage/smgr/smgr.c:78-126`。 这里定义 `struct f_smgr`。 它是一组函数指针。
这个结构就是 `smgr.c` 和某个 storage manager module 之间的 ABI。

`f_smgr` 中和本节直接相关的函数指针有：
- `smgr_create`
- `smgr_exists`
- `smgr_unlink`
- `smgr_extend`
- `smgr_zeroextend`
- `smgr_readv`
- `smgr_writev`
- `smgr_nblocks`
- `smgr_truncate`
- `smgr_immedsync`
- `smgr_registersync`
- `smgr_fd`

读 `src/backend/storage/smgr/smgr.c:128-152`。 `smgrsw[]` 只有一个 entry。 这个 entry 的注释是 `magnetic disk`。
所有函数指针都指向 `md.c`：
- `mdinit`
- `mdopen`
- `mdclose`
- `mdcreate`
- `mdexists`
- `mdunlink`
- `mdextend`
- `mdzeroextend`
- `mdprefetch`
- `mdmaxcombine`
- `mdreadv`
- `mdstartreadv`
- `mdwritev`
- `mdwriteback`
- `mdnblocks`
- `mdtruncate`
- `mdimmedsync`
- `mdregistersync`
- `mdfd`

这就是 API 到实现的最短路径。 `smgrcreate()` 本身只有 HOLD_INTERRUPTS、dispatch、RESUME_INTERRUPTS。 真正创建文件的是 `mdcreate()`。
`smgrextend()` 本身也是 dispatch。 真正选择 segment、计算偏移、写入 block 的是 `mdextend()`。 `smgrtruncate()` 本身负责 buffer drop、smgr invalidation 和 cached size 更新。
真正执行 `FileTruncate()` 的是 `mdtruncate()`。

`smgr.c` 文件头还有一个容易忽略的点。 它说大多数函数会 `HOLD_INTERRUPTS()`。 原因是异步 interrupt processing 可能触发 procsignal 处理。
procsignal 又可能触发 `smgrreleaseall()`。 如果这发生在 `smgr.c` 或 `md.c` 操作同一个 `SMgrRelation`、同一个 fd array 的中间，就会破坏不可重入假设。 所以 switch 层把中断保护集中起来。

`smgr.c` 的错误约定也写在 `f_smgr` 注释里。 一般 storage manager subfunction 遇到问题通过 `elog(ERROR)` 或 `ereport(ERROR)` 报告。 例外是 `smgr_unlink`。
unlink 通常发生在 transaction commit/abort 后的 cleanup。 那时已经太晚，不能再把事务变成失败。 所以 unlink 失败应该报 `WARNING`，而不是 `ERROR`。
`mdunlink()` 正是按这个约定实现的。

---

## 4. `smgropen()` 与 `SMgrRelation` 缓存
读 `src/backend/storage/smgr/smgr.c:225-289`。 `smgropen()` 的注释很关键。 它返回一个 `SMgrRelation` 对象。
如果 hash table 中已经有这个 relation，就返回同一个 entry。 如果没有，就创建一个新的 entry。 这个函数不打开底层文件。
它只创建或查找 backend-local 的 smgr cache entry。

`SMgrRelationHash` 的 key 是 `RelFileLocatorBackend`。 第一次调用 `smgropen()` 时会创建 hash table。 新 entry 初始化时：
- `smgr_targblock = InvalidBlockNumber`
- 每个 fork 的 `smgr_cached_nblocks = InvalidBlockNumber`
- `smgr_which = 0`
- `pincount = 0`
- 加入 `unpinned_relns`
- 调用 `smgrsw[0].smgr_open(reln)`

当前 `smgr_open` 是 `mdopen()`。 读 `src/backend/storage/smgr/md.c:710-719`。 `mdopen()` 只做一件事：把每个 fork 的 `md_num_open_segs[forknum]` 设为 0。
它不打开任何 segment 文件。 这就是 `smgropen()` 不做 I/O 的实际证明。

文件真正打开通常发生在这些路径：
- `mdcreate()` 创建第 0 个 segment 后把 fd 放入 `md_seg_fds`。
- `mdopenfork()` 为某个 fork 打开第 0 个 segment。
- `_mdfd_getseg()` 为目标 block 打开或创建目标 segment。
- `mdnblocks()` 逐段打开 active segment 来计算 relation size。

`smgrpin()` 和 `smgrunpin()` 控制 entry 是否在事务结束时被销毁。 relcache 持有的 smgr reference 会 pin。 直接访问 smgr 的辅助进程、checkpointer、redo 或 buffer flush 路径可能拿 unpinned entry。
事务结束时 `AtEOXact_SMgr()` 调用 `smgrdestroyall()`。 它会销毁所有 unpinned entry。

`smgrdestroy()` 的实际行为是：
- assert `pincount == 0`
- 对每个 fork 调用 `smgr_close`
- 从 `unpinned_relns` 链表删除
- 从 hash table 删除

当前 `smgr_close` 是 `mdclose()`。 `mdclose()` 会从最后一个 opened segment 开始关闭 fd。 每关闭一个 segment，就用 `_fdvec_resize()` 把 open segment 数量减一。

`smgrrelease()` 和 `smgrclose()` 当前关系也值得注意。 `smgrclose()` 只是 `smgrrelease()` 的 synonym。 `smgrrelease()` 不销毁 `SMgrRelation` 对象。
它释放底层资源，也就是关闭每个 fork 的 opened segment。 然后把每个 fork 的 cached size 设为 `InvalidBlockNumber`。 它还清掉 `smgr_targblock`。

这和 procsignal barrier 相关。 `PROCSIGNAL_BARRIER_SMGRRELEASE` 要求 backend 立即关闭 open files。 `ProcessBarrierSmgrRelease()` 调用 `smgrreleaseall()`。
这不会销毁 `SMgrRelation` 对象。 它只关闭 fd、清 cached size。 因为 barrier 可能发生在事务中间，不能让仍被调用栈引用的 `SMgrRelation *` 失效。

---

## 5. segment 拆分与路径规则
`md.c` 文件头是本节最重要的注释之一。 它说明 PostgreSQL 把大 relation 拆成多个 segment 文件，是为了避开操作系统单文件大小限制。 segment size 由 `RELSEG_SIZE` 决定。

`RELSEG_SIZE` 的单位不是 byte。 它的单位是 PostgreSQL block。 单个 segment 文件最大大小是：

```text
RELSEG_SIZE * BLCKSZ
```

默认值来自构建配置。 `meson_options.txt` 中：
- `blocksize` 默认是 `8`，单位 kB。
- `segsize` 默认是 `1`，单位 GB。
- `segsize_blocks` 默认是 `0`，表示不直接用 block 数覆盖。

`meson.build` 中：
- `blocksize = get_option('blocksize').to_int() * 1024`
- 如果 `segsize_blocks != 0`，直接使用 block 数。
- 否则 `segsize = (segsize GB bytes) / blocksize`
- 最后 `cdata.set('RELSEG_SIZE', segsize)`

`configure.ac` 中也是同样的逻辑。 默认 `blocksize=8`。 默认 `segsize=1`。
`RELSEG_SIZE = (1024 / blocksize) * segsize * 1024`。 代入默认值：

```text
RELSEG_SIZE = (1024 / 8) * 1 * 1024 = 131072 blocks
BLCKSZ      = 8 * 1024 = 8192 bytes
segment     = 131072 * 8192 = 1073741824 bytes
```

所以“1GB segment”是默认构建结果。 如果开发者用 `--with-segsize-blocks` 或 Meson 的 `-Dsegsize_blocks=`，可以做小 segment 测试。 源码还提醒 `segsize_blocks` 主要用于开发者测试 segment 相关代码。

`src/include/storage/block.h` 定义 `BlockNumber`。 它是 `uint32`。 有效 block number 范围是 `0` 到 `0xFFFFFFFE`。
`0xFFFFFFFF` 是 `InvalidBlockNumber`。 这就是为什么 `mdextend()` 拒绝扩展到 `InvalidBlockNumber`。 这也是为什么 `mdzeroextend()` 检查 `blocknum + nblocks >= InvalidBlockNumber`。

segment number 和 offset 的计算散布在多个函数中。 `mdextend()` 使用：

```text
seekpos = BLCKSZ * (blocknum % RELSEG_SIZE)
```

`_mdfd_getseg()` 使用：

```text
targetseg = blkno / RELSEG_SIZE
```

`mdmaxcombine()` 返回：

```text
RELSEG_SIZE - (blocknum % RELSEG_SIZE)
```

这说明一次 vector I/O 不能跨 segment。 因为跨 segment 就跨了不同文件。 当前 `mdreadv()` 和 `mdwritev()` 遇到请求跨 segment 会 `elog(ERROR, "read crosses segment boundary")` 或 `elog(ERROR, "write crosses segment boundary")`。
上层如果想合并多个 block 的 I/O，需要先调用 `smgrmaxcombine()`。

路径规则在 `_mdfd_segpath()`。 `segno == 0` 时返回 `relpath(...)`。 `segno > 0` 时返回 `relpath + "." + segno`。
例如 main fork 第 0 段可能像：

```text
base/5/16384
```

第 1 段就是：

```text
base/5/16384.1
```

第 2 段就是：

```text
base/5/16384.2
```

fork 后缀和 segment 后缀是两个不同概念。 FSM fork、VM fork、INIT fork 的基础路径由 `relpath()` 处理。 segment 后缀由 `md.c` 在基础路径后继续追加 `.segno`。

---

## 6. `MdfdVec` 与 open segment array
`md.c` 中 `MdfdVec` 很小：
- `mdfd_vfd` 是 fd.c descriptor pool 中的 `File`。
- `mdfd_segno` 是 segment number。

每个 `SMgrRelation` 的每个 fork 都有：
- `md_num_open_segs[forknum]`
- `md_seg_fds[forknum]`

这个数组分配在 `MdCxt`。 `mdinit()` 在 backend 启动时创建 `MdSmgr` memory context。 `_fdvec_resize()` 负责扩容、缩小和释放数组。

这里有一个设计细节。 当 `nseg` 变小但不为 0 时，`_fdvec_resize()` 不会 `repalloc()` 到更小。 注释解释原因：`mdtruncate()` 保证不分配内存，因此可以在 critical section 中使用。
如果缩小数组时真的释放或重新分配，后续在 critical section 中重新扩张就可能需要分配内存。 所以它只更新 `md_num_open_segs`。 数组中多余空间暂时浪费。
这是为了换取 truncate 路径的可靠性。

`md.c` 的注释还说明： 一个 fork 的 `md_num_open_segs` 有某个值，并不意味着 relation 没有更多 segment。 它只表示当前 backend 已经打开到哪里。
其他 backend 可能已经扩展了 relation。 如果当前 backend 没有访问后续 segment，就不会有 array entry。

但对于 inactive segment，`md.c` 不保留 array entry。 只要发现 partial segment，就认为后面的 segment 都 inactive。 这和 segment layout 不变量一致。

`mdnblocks()` 有一个副作用。 它会打开所有 active segment，并把它们加入 `md_seg_fds`。 这就是为什么 `mdtruncate()` 的注释要求 caller 必须先调用 `smgrnblocks()` 得到当前大小。
`mdtruncate()` 不自己探索磁盘上的 active segments。 它依赖 `mdnblocks()` 已经把 active segments 都打开。 这样 truncate loop 才能从最后一个 active segment 往前处理。

---

## 7. `mdcreate()`：创建第 0 个 segment
`smgrcreate()` 在 `smgr.c` 中只是 dispatch。 真正实现是 `mdcreate()`。 读 `src/backend/storage/smgr/md.c:217-274`。

`mdcreate()` 的语义是创建一个 relation fork 的第 0 个 segment。 它不创建 `.1`、`.2`。 后续 segment 由 `_mdfd_getseg()` 在 extend 路径中按需创建。

`mdcreate()` 的参数包括：
- `SMgrRelation reln`
- `ForkNumber forknum`
- `bool isRedo`

`isRedo` 是 crash recovery 和 WAL replay 的幂等边界。 如果 `isRedo == true` 且这个 fork 已经有 opened segment，`mdcreate()` 直接返回。 如果 `O_CREAT | O_EXCL` 创建失败，且 `isRedo == true`，它会尝试普通 open。
这允许 replay create record 时发现文件已经存在。 这不是普通 create 路径的行为。 普通路径文件已存在仍是错误。

`mdcreate()` 的步骤：
1. 如有需要，调用 `TablespaceCreateDbspace()` 创建 tablespace 的 per-database 子目录。
2. 调用 `relpath()` 得到 fork 的 base path。
3. 使用 `_mdfd_open_flags() | O_CREAT | O_EXCL` 创建文件。
4. redo 路径创建失败时尝试 open existing file。
5. `_fdvec_resize(reln, forknum, 1)`。
6. 把 fd 放进 `md_seg_fds[forknum][0]`。
7. 设置 `mdfd_segno = 0`。
8. 非 temp relation 调用 `register_dirty_segment()`。

第 8 步说明 create 本身也要进入 fsync 体系。 文件创建和元数据更新需要 checkpoint 侧知道。 否则 crash 后可能出现 WAL 说 relation 已创建，但文件或目录项没有稳定落盘的问题。

`RelationCreateStorage()` 是上层常见入口。 读 `src/backend/catalog/storage.c:121-180`。 它根据 relation persistence 决定 backend proc number 和是否需要 WAL。
然后：
- `smgropen()`
- `smgrcreate(... MAIN_FORKNUM, false)`
- permanent relation 需要 WAL 时调用 `log_smgrcreate()`
- 如果 permanent 但 `XLogIsNeeded()` 为 false，则 `AddPendingSync()`

`log_smgrcreate()` 写 `RM_SMGR_ID` 的 `XLOG_SMGR_CREATE` record。 `storage_xlog.h` 说明 smgr WAL record 当前有两种：
- `XLOG_SMGR_CREATE`
- `XLOG_SMGR_TRUNCATE`

注意 deletion 不在这里作为普通 `XLOG_SMGR_DELETE`。 `storage_xlog.h` 注释说 deletion 由 `xact.c` 处理，因为它是 transaction commit 的一部分。

---

## 8. `_mdfd_getseg()`：segment lifecycle 的枢纽
`_mdfd_getseg()` 是理解 `md.c` 的核心函数。 读 `src/backend/storage/smgr/md.c:1746-1878`。 它根据 block number 找到目标 segment。
如果目标 segment 不存在，根据 behavior 决定报错、返回 NULL、创建 segment，或 recovery 下创建 segment。

behavior flags 在 `md.c:112-122`：
- `EXTENSION_FAIL`
- `EXTENSION_RETURN_NULL`
- `EXTENSION_CREATE`
- `EXTENSION_CREATE_RECOVERY`
- `EXTENSION_DONT_OPEN`

这些 flag 描述调用者对缺失 segment 的容忍度。 `mdextend()` 使用 `EXTENSION_CREATE`。 它是在明确扩展 relation，所以可以创建 segment。
`mdreadv()` 和 `mdwritev()` 使用 `EXTENSION_FAIL | EXTENSION_CREATE_RECOVERY`。 普通运行时读写缺失 segment 是错误。 recovery 时可以创建缺失 segment，以便继续 replay。
`mdprefetch()` 在 recovery 中使用 `EXTENSION_RETURN_NULL`，因为预取失败可以返回 false。 `mdwriteback()` 使用 `EXTENSION_DONT_OPEN`，因为它不想为了 writeback 重新打开 segment。

`_mdfd_getseg()` 先计算：

```text
targetseg = blkno / RELSEG_SIZE
```

如果 `targetseg < md_num_open_segs[forknum]`，说明目标 segment 已经打开，直接返回。 如果 caller 传了 `EXTENSION_DONT_OPEN`，目标 segment 没打开就返回 NULL。 否则，它从当前最后一个 opened segment 开始，逐段走到 target segment。

这一段循环做了关键校验。 每走到下一个 segment 前，先用 `_mdnblocks()` 看当前 segment 大小。 如果当前 segment 大于 `RELSEG_SIZE`，直接 `elog(FATAL, "segment too big")`。
这不是普通 SQL 错误。 这是磁盘布局不变量损坏。

如果 caller 允许创建，或者当前在 recovery 且 caller 允许 `EXTENSION_CREATE_RECOVERY`，函数会创建后续 segment。 但创建前还有一个重要动作。 如果当前 segment 小于 `RELSEG_SIZE`，它会先把当前 segment pad 到 full。
具体做法是分配一个 zero buffer，然后调用 `mdextend()` 写入当前 segment 的最后一个 block：

```text
block = nextsegno * RELSEG_SIZE - 1
```

这样可以维护“target segment 前面的 segment 必须 full”的不变量。 这对 recovery 和某些非连续扩展场景都重要。 注释中特别提到 hash indexes 可能 discontiguously extend。

如果 caller 不允许创建，且当前 segment 小于 `RELSEG_SIZE`，说明目标 block 已经落在 relation EOF 之后。 如果 caller 允许 `EXTENSION_RETURN_NULL`，返回 NULL 并把 `errno` 设为 `ENOENT`。 否则报错，错误信息会指出 previous segment 只有多少 blocks。

最后 `_mdfd_openseg()` 打开或创建下一个 segment。 `_mdfd_openseg()` 要求 segment 按顺序打开。 它 assert `segno == md_num_open_segs[forknum]`。
这保证 array index 和 segment number 对齐。

---

## 9. `mdextend()`：写一个 block 并可能创建 segment
`smgrextend()` 是公共 API。 读 `src/backend/storage/smgr/smgr.c:610-639`。 它的注释说语义接近 `smgrwrite()`，但用于 relation extension。
也就是 `blocknum` 在当前 EOF 或 EOF 之后。 它还假设在当前 EOF 之后写一个 block 时，中间空洞会被文件系统读作 zero。

`smgrextend()` dispatch 到 `mdextend()` 后，会维护 `smgr_cached_nblocks`。 如果 cached size 正好等于 `blocknum`，说明这次扩展是顺序扩展一个 block。 它把 cached size 更新为 `blocknum + 1`。
否则把 cached size 设为 `InvalidBlockNumber`。 这体现了 cached size 的保守策略。 如果不能证明缓存仍正确，就让下一次 `smgrnblocks()` 问 kernel。

`mdextend()` 读 `src/backend/storage/smgr/md.c:478-544`。 它的流程是：
1. direct I/O build 下 assert buffer 对齐。
2. debug 宏 `CHECK_WRITE_VS_EXTEND` 下校验 `blocknum >= mdnblocks()`。
3. 如果 `blocknum == InvalidBlockNumber`，报 program limit exceeded。
4. 调用 `_mdfd_getseg(... EXTENSION_CREATE)` 找到或创建目标 segment。
5. 计算 segment 内 byte offset。
6. `FileWrite()` 写入一个 `BLCKSZ` block。
7. 写入失败或短写时报错。
8. `skipFsync == false` 且不是 temp relation 时注册 dirty segment。
9. assert 当前 segment block 数不超过 `RELSEG_SIZE`。

短写错误信息会包含实际写了多少 bytes。 如果 `FileWrite()` 返回负数，错误 hint 是检查 free disk space。 如果返回非负但少于 `BLCKSZ`，errcode 是 `ERRCODE_DISK_FULL`。
这是磁盘满的典型边界。

`skipFsync` 不是“允许丢数据”。 它表示调用者会用其他机制补上 durability。 普通 buffer flush 传 false。
bulk write 会传 true，然后在 finish 阶段整体注册或立即 fsync。 temp relation 无论如何都不需要 fsync。

---

## 10. `mdzeroextend()`：批量扩展 zero blocks
`smgrzeroextend()` 用于一次扩展多个 zero blocks。 当前 buffer manager 的 relation extension 路径会用它。 读 `src/backend/storage/buffer/bufmgr.c:3008-3018`。
共享 buffer extension 先分配 buffer descriptor 和 tag，然后调用 `smgrzeroextend()` 真正扩展磁盘文件。 注释还说如果 `smgrzeroextend()` 失败，会留下 allocated 但非 valid 的 buffer。 下一次 relation extension 会选择同一个 block number，再尝试扩展。

`smgrzeroextend()` dispatch 后也维护 cached size。 如果 cached size 正好等于 `blocknum`，它更新为 `blocknum + nblocks`。 否则设为 `InvalidBlockNumber`。

`mdzeroextend()` 读 `src/backend/storage/smgr/md.c:546-663`。 它和 `mdextend()` 的主要差异：
- 可以一次扩展多个 block。
- 会按 segment boundary 拆分。
- 对大于 8 blocks 的扩展，优先用 `FileFallocate()`，除非配置要求 write zeros。
- 对较小扩展或 write-zero 方法，使用 `FileZero()`。

循环变量：
- `curblocknum` 是当前待扩展的 block。
- `remblocks` 是剩余 block 数。
- `segstartblock = curblocknum % RELSEG_SIZE`
- `numblocks` 是本 segment 内本次能处理的 block 数。

如果 `segstartblock + remblocks > RELSEG_SIZE`，本轮只能扩到 segment 末尾。 下一轮再处理下一个 segment。 这就是批量扩展必须理解 segment boundary 的原因。

`mdzeroextend()` 也有上限检查。 如果 `blocknum + nblocks >= InvalidBlockNumber`，直接报错。 这里用 `uint64` 做加法，避免 32 位溢出。

`FileFallocate()` 的优点是可能不用把 zero pages 放进 page cache。 对大扩展通常更高效。 但注释说小扩展使用 fallocate 可能破坏某些文件系统的 delayed allocation。
当前阈值是大于 8 blocks 才考虑 fallocate。

---

## 11. `mdreadv()`：读 block、短读和 EOF
当前基线的 read path 是 vector I/O。 `smgrread()` 是单页 wrapper。 `smgrreadv()` dispatch 到 `mdreadv()`。

`mdreadv()` 读 `src/backend/storage/smgr/md.c:855-991`。 它循环处理 `nblocks`，但当前实现要求一次调用不能跨 segment。 它先计算当前 segment 能读多少 blocks。
又用 `lengthof(iov)` 限制一次 vector I/O 的 iovec 数量。 然后如果 `nblocks_this_segment != nblocks`，直接 `elog(ERROR, "read crosses segment boundary")`。 所以 callers 必须用 `smgrmaxcombine()` 控制合并范围。

真正读之前，`mdreadv()` 调用：

```text
_mdfd_getseg(reln, forknum, blocknum, false,
             EXTENSION_FAIL | EXTENSION_CREATE_RECOVERY)
```

普通运行时缺失 segment 是错误。 recovery 中允许创建缺失 segment。 这和 WAL replay 的容错策略有关。
如果 WAL 中先有对高 block 的写，后来又有 drop/truncate，replay 不能因为中间文件缺失就停在错误位置。 它会尽量创建文件并重放，直到后续 drop/truncate record 清理。

`buffers_to_iovec()` 把多个 buffer pointer 合并成 iovec。 如果 buffers 在内存中连续，就合并成更少的 iovec。 这减少系统调用开销。
direct I/O build 下还会 assert 每个 buffer 对齐。

短读处理是 `mdreadv()` 的重点。 它调用 `FileReadV()` 后，不假设一次 short read 就等于 EOF。 它维护：
- `transferred_this_segment`
- `size_this_segment`
- `seekpos`
- `iov`

如果 `nbytes > 0` 但还没读满，就更新 offset 和 remaining iovec，然后继续读。 这处理了内核或文件系统返回短读但仍可继续读取的情况。

如果 `nbytes < 0`，报 `ERROR`。 错误信息包含 blocks 范围和文件名。

如果 `nbytes == 0`，说明到达 EOF，或者 EOF 处存在 partial block。 正常情况下，上层不应该读不存在的 block。 所以默认报 `ERRCODE_DATA_CORRUPTED`。

但这里有两个历史/恢复边界：
- `zero_damaged_pages` 为 on 时，可以返回 zero page。
- `InRecovery` 为 true 时，也可以返回 zero page。

当前源码对此加了强烈注释。 它说这条 codepath 在 recovery 中被认为不可达。 原因是 missing segment 不会被直接读成不存在 block，而可能已由 recovery create path 处理。
把超过 EOF 的 block 放进 buffer pool 也有问题，因为 `smgrnblocks()` 不会把它们算进去。 源码在这条分支里放了 `Assert(false)`，计划后续移除这段逻辑。 文档里要保留这个事实，不能把它描述成推荐路径。

AIO 版本是 `mdstartreadv()`。 它没有复制 `zero_damaged_pages` 的逻辑。 注释解释原因：AIO 的 definer 和 completor 之间设置可能不同，而且这段逻辑本来就有问题。
`md_readv_complete()` 把 byte count 转成 block count。 0 blocks read 被视为失败。 partial read 标记为 `PGAIO_RS_PARTIAL`，由上层重试未读部分。

---

## 12. `mdwritev()`：写 existing blocks
`smgrwrite()` 是单页 wrapper。 `smgrwritev()` dispatch 到 `mdwritev()`。

`smgrwritev()` 的注释非常重要。 它只用于更新已存在 blocks。 扩展 relation 必须用 `smgrextend()`。
这不是性能建议，而是语义边界。 extend 需要维护 segment 创建、cached nblocks 和 relation extension 竞态。 write existing block 不应该悄悄改变 relation size。

`smgrwritev()` 的注释还说明它不是 synchronous write。 返回时 block 只是交给 kernel。 但系统会安排在下一个 checkpoint 前 fsync。
这个安排依赖 `register_dirty_segment()`。

`mdwritev()` 读 `src/backend/storage/smgr/md.c:1063-1166`。 它和 `mdreadv()` 类似：
- debug 宏下检查写入范围不超过 `mdnblocks()`。
- 每次调用不能跨 segment。
- 使用 `_mdfd_getseg(... EXTENSION_FAIL | EXTENSION_CREATE_RECOVERY)`。
- 使用 iovec 合并 buffers。
- 循环处理短写。
- 写完后按需注册 dirty segment。

短写处理和短读类似。 如果 `FileWriteV()` 返回负数，报 `ERROR`。 如果 `errno == ENOSPC`，加 hint 检查 free disk space。
如果返回正数但没写满，更新 offset 和 iovec 继续写。 注释说如果短写原因是磁盘空间不足，后续尝试应该从 kernel 收到 `ENOSPC`。

`mdwritev()` 写完后：

```text
if (!skipFsync && !SmgrIsTemp(reln))
    register_dirty_segment(reln, forknum, v);
```

这和 `mdextend()`、`mdzeroextend()` 一致。 普通 relation 的数据文件写入必须进入 sync request 体系。 temp relation 不需要。
调用者传 `skipFsync = true` 时，必须有后续补偿机制。

共享 buffer flush 路径在 `src/backend/storage/buffer/bufmgr.c:4570-4590`。 它先按 WAL-first rule flush WAL。 然后设置 page checksum。
最后调用 `smgrwrite(..., false)`。

local buffer flush 路径在 `src/backend/storage/buffer/localbuf.c:203-212`。 它也设置 checksum，然后调用 `smgrwrite(..., false)`。 temp relation 虽然传 false，但 `mdwritev()` 因 `SmgrIsTemp(reln)` 不注册 fsync。

---

## 13. `mdnblocks()` 与 cached size
`smgrnblocks()` 是 size 查询入口。 读 `src/backend/storage/smgr/smgr.c:814-837`。 它先调用 `smgrnblocks_cached()`。
如果得到有效值，就直接返回。 否则 dispatch 到 `mdnblocks()`。 然后把结果写回 `smgr_cached_nblocks[forknum]`。

但 `smgrnblocks_cached()` 读 `src/backend/storage/smgr/smgr.c:839-858`。 当前它只在 `InRecovery` 且 cached value 有效时返回缓存。 否则返回 `InvalidBlockNumber`。
注释解释原因：目前没有 fork extension 的 shared invalidation 机制。

这解释了一个看似矛盾的地方。 `smgrextend()`、`smgrzeroextend()`、`smgrtruncate()` 都维护 cached size。 但普通运行时 `smgrnblocks_cached()` 不会信任它。
这些维护仍然有价值，因为 recovery 和某些内部路径会读取缓存。 例如 buffer drop 优化会在 recovery 中依赖 cached fork size。

`mdnblocks()` 读 `src/backend/storage/smgr/md.c:1226-1286`。 它先 `mdopenfork(... EXTENSION_FAIL)` 打开第 0 个 segment。 然后从当前最后一个 opened segment 开始。
如果 segment 小于 `RELSEG_SIZE`，返回：

```text
segno * RELSEG_SIZE + nblocks
```

如果 segment 正好 `RELSEG_SIZE`，尝试打开下一个 segment。 如果下一个 segment 不存在，返回：

```text
segno * RELSEG_SIZE
```

如果发现 segment 大于 `RELSEG_SIZE`，报 `FATAL`。 这是磁盘 segment 不变量损坏。

`_mdnblocks()` 用 `FileSize()` 得到单个 segment 的 byte length。 然后返回 `len / BLCKSZ`。 注释明确说 partial block at EOF 会被忽略。
也就是说，如果文件尾部有不是整 block 的残片，`mdnblocks()` 不会把它算成一个 PostgreSQL block。

---

## 14. `smgrtruncate()` 与 `mdtruncate()`
`smgrtruncate()` 是上层 truncate API。 读 `src/backend/storage/smgr/smgr.c:860-925`。 它一次可以 truncate 多个 fork。
caller 传入：
- `forknum[]`
- `nforks`
- `old_nblocks[]`
- `nblocks[]`

注释列出几个 contract。 truncate 是立即执行，不能 rollback。 caller 必须持有 relation 的 `AccessExclusiveLock`。
caller 应该先在 critical section 外检查当前大小。 在检查大小和调用 `smgrtruncate()` 之间，不应该处理中断或调用同 relation 的 smgr 函数。

`smgrtruncate()` 先调用 `DropRelationBuffers()`。 这会从 buffer pool 丢弃将被删除的 blocks。 不写回它们。
因为 truncate 之后这些 blocks 不应该再存在。

然后调用 `CacheInvalidateSmgr(reln->smgr_rlocator)`。 这个 shared invalidation 要求其他 backend 关闭 dangling smgr references。 原因有两个。
第一，它们可能持有即将被 truncate/remove 的 segment fd。 第二，它们的 `smgr_targblock` 可能指向新 EOF 之后。 `CacheInvalidateSmgr()` 的注释还强调，某些 backend 可能只有 smgr entry，没有 relcache entry。
比如它只是在写 dirty shared buffer。 所以 relcache invalidation 不够，需要 smgr-level invalidation。

之后 `smgrtruncate()` 对每个 fork：
1. 先把 local cached size 设为 `InvalidBlockNumber`。
2. 调用 `mdtruncate(reln, fork, old_blocks, nblocks)`。
3. 成功后更新 local cached size。

cached size 更新有一个 recovery 特例。 如果 `nblocks > old_nblocks`，说明 replay 早期 truncate record 时磁盘已经处在更小的新状态。 `mdtruncate()` 在 recovery 下会忽略这个请求。
所以 cached size 设为 old size，而不是请求的 nblocks。

`mdtruncate()` 读 `src/backend/storage/smgr/md.c:1288-1385`。 它保证不分配内存，因此可以在 critical section 中使用。 这依赖 caller 已经通过 `smgrnblocks()` 打开 active segments。

如果 `nblocks > curnblk`：
- recovery 中直接 return。
- 非 recovery 报 `ERROR`。

如果 `nblocks == curnblk`，直接 return。

真正 truncate 从最后一个 opened segment 往前走。 设：

```text
priorblocks = (curopensegs - 1) * RELSEG_SIZE
```

如果 `priorblocks > nblocks`，说明整个当前 segment 已经在新 EOF 之后。 这是 inactive segment。 `mdtruncate()` 会把它 `FileTruncate(..., 0)`。
非 temp relation 注册 dirty segment。 然后关闭 fd，并把 open segment count 减一。 它 assert 这个 segment 不是第 0 个 segment。
也就是永远不在这里 drop 第一个 segment。

如果 `priorblocks + RELSEG_SIZE > nblocks`，说明当前 segment 是最后一个保留 segment。 它会截到：

```text
lastsegblocks = nblocks - priorblocks
lastsegblocks * BLCKSZ
```

这个 branch 包含一个非常关键的注释。 如果 `nblocks` 正好是 `RELSEG_SIZE` 的倍数，比如 K 个完整 segment，`mdtruncate()` 会把第 K+1 个 segment 截成 0 并保留。 这保持文件布局不变量：
- 0 个或多个 full segment。
- 一个 partial segment。
- 后面可以有 0 长度 inactive segment。
当 relation size 正好落在 segment boundary，0 长度的下一个 segment 就是 partial segment。

如果当前 segment 完全仍被需要，就 break。 更早的 segment 都不需要处理。

---

## 15. `mdunlink()`：删除 relation 文件
`smgrdounlinkall()` 是 smgr 层的删除入口。 读 `src/backend/storage/smgr/smgr.c:527-607`。 它用于立即 unlink 给定 relations 的所有 forks。
注释明确说这不应该用于 transactional operations，因为不能 undo。 事务性 relation drop 通过 pending delete 在 commit/abort 后调用它。

`smgrdounlinkall()` 做几件事：
1. `DropRelationsAllBuffers()` 丢弃 relation 的所有 buffers。
2. 为每个 relation 保存 `RelFileLocatorBackend`。
3. 关闭本 backend smgr 层所有 fork fd。
4. 对每个 relation 发送 `CacheInvalidateSmgr()`。
5. 对每个 relation 的每个 fork 调用 `smgr_unlink`。

当前 `smgr_unlink` 是 `mdunlink()`。 `mdunlink()` 参数是 `RelFileLocatorBackend`，不是 `SMgrRelation`。 注释说调用到这里时，通常已经没有 `SMgrRelation` hash entry。

`mdunlink()` 可以删除单个 fork，也可以在 `forknum == InvalidForkNumber` 时删除所有 forks。 实际工作在 `mdunlinkfork()`。 读 `src/backend/storage/smgr/md.c:276-476`。

普通 permanent relation 的 main fork 第 0 个 segment 有特殊处理。 非 redo、非 binary upgrade、非 temp、main fork 的情况下：
- 不立即 unlink 第 0 segment。
- 先 truncate 到 0。
- 注册 `SYNC_UNLINK_REQUEST`，等下一个 checkpoint 后再 unlink。

这样做是为了避免 relfilenumber 复用导致 crash recovery 丢失新 relation 内容。 源码注释给了完整场景。 简化说：
1. 旧 relation 删除并 commit。
2. 新 relation 恰好复用同一个 relfilenumber。
3. crash 发生在 checkpoint 前。
4. recovery replay 旧 relation 删除 record。
5. 如果旧文件已完全消失，而新 relation 某些内容依赖 fsync 而不是 WAL 重放，可能被错误删除或无法恢复。

保留一个 0 长度 base file 可以阻止 relfilenumber 在安全 checkpoint 前被复用。 relfilenumber 分配会跳过已存在文件。

以下情况不走延迟 unlink：
- redo。
- binary upgrade。
- 非 main fork。
- temp relation。

这些情况下第 0 segment 会被 truncate 后 unlink，或 temp relation 直接 unlink。 redo 中没有创建冲突 relation 的可能，所以可以立即删除。 其他 forks 不承担防止 relfilenumber 复用的职责。
temp relation 不写 WAL，文件名模式也不同。

后续 segment，也就是 `.1`、`.2` 等，会循环删除。 删除前非 temp relation 先 truncate。 这样即使其他 backend 或 checkpointer 仍持有 open fd，磁盘空间也能尽快释放。
然后注册 forget request，忘掉 pending fsync。 最后 unlink。 循环直到遇到 `ENOENT`。

`mdunlink()` 的错误级别是 `WARNING`。 这符合 `smgr.c` 的约定。 删除通常发生在事务结尾后的 cleanup。
这时不能因为 unlink 失败把已经决定 commit/abort 的事务改回失败。

---

## 16. fsync 注册：dirty、sync、unlink、forget
`md.c` 不只负责读写。 它还负责把 relation segment 纳入 checkpoint fsync 体系。

核心 helper 是 `register_dirty_segment()`。 读 `src/backend/storage/smgr/md.c:1509-1557`。 它用 `INIT_MD_FILETAG()` 构造 `FileTag`。
tag 中包含：
- handler = `SYNC_HANDLER_MD`
- rel locator
- fork number
- segment number

然后调用：

```text
RegisterSyncRequest(&tag, SYNC_REQUEST, false)
```

如果无法把请求交给 checkpointer 队列，就在当前 backend 里直接 `FileSync()`。 这是一条后备路径。 源码还会把这次 fsync 统计到 IO stats。

哪些路径调用 `register_dirty_segment()`：
- `mdcreate()`
- `mdextend()`，如果 `skipFsync == false` 且非 temp。
- `mdzeroextend()`，如果 `skipFsync == false` 且非 temp。
- `mdwritev()`，如果 `skipFsync == false` 且非 temp。
- `mdtruncate()`，如果非 temp。
- `mdregistersync()` 为 active/inactive segments 注册。

`register_unlink_segment()` 注册 `SYNC_UNLINK_REQUEST`。 普通 main fork 第 0 segment 延迟 unlink 用它。 请求设置 `retryOnError = true`。

`register_forget_request()` 注册 `SYNC_FORGET_REQUEST`。 unlink 前会忘掉某个 segment 的 pending sync request。 如果文件马上要删，继续 fsync 它没有意义。

`ForgetDatabaseSyncRequests()` 注册 `SYNC_FILTER_REQUEST`。 它用于 drop database 时过滤掉相关数据库的 fsync/unlink requests。

`mdregistersync()` 的语义是“把整个 relation fork 标记为下次 checkpoint 需要 fsync”。 它先调用 `mdnblocks()`，确保 active segments 都打开。 然后临时打开 inactive segments。
再对所有 active 和 inactive segments 调用 `register_dirty_segment()`。 打开 inactive segment 后会立刻关闭。

为什么 inactive segment 也要 sync？ 因为 `mdtruncate()` 会把 segment 截成 0。 如果这个 0 长度状态没有在 checkpoint 前稳定落盘，crash 后旧数据可能重新出现在 table 中。
`mdimmedsync()` 的注释把这个场景说得更直白。

`mdimmedsync()` 是立即 fsync。 它也会 sync active 和 inactive segments。 它用于 caller 明确需要在返回前保证文件稳定落盘的场景。
例如不逐页 WAL-log 的新 index build，需要在 commit 前 fsync 完整 relation。

---

## 17. `bulk_write.c`：绕过 buffer manager 后怎样补偿
`bulk_write.c` 主题是高效且可靠地填充一个新 relation。 文件头注释说它绕过 buffer manager，直接调用 `smgrextend()`。 好处是避免 buffer manager 锁和管理开销。
代价是 build 完成后第一次访问仍要重新读入 shared buffers。

`BulkWriteState` 保存：
- 目标 `SMgrRelation`
- fork number
- 是否使用 WAL
- pending writes 数组
- 当前 relation size
- bulk 操作开始时的 `RedoRecPtr`
- memory context

`smgr_bulk_start_smgr()` 初始化 state。 它调用 `smgrnblocks()` 记录当前 relation size。 它调用 `GetRedoRecPtr()` 记录开始时的 checkpoint redo pointer。

`smgr_bulk_write()` 只把 page 放进 pending array。 pending 数量达到 `XLR_MAX_BLOCK_ID` 时 flush。

`smgr_bulk_flush()` 做三件事。 第一，按 block number 排序 pending writes。 第二，如果需要 WAL，调用 `log_newpages()` 一次记录多个 pages。
第三，对每个 page 设置 checksum，然后写文件。

写文件时：
- 如果 `blkno >= relsize`，说明要扩展。
- 如果 `blkno > relsize`，先用 all-zero page 填空洞。
- 填空洞调用 `smgrextend(..., skipFsync = true)`。
- 写目标 page 也调用 `smgrextend(..., skipFsync = true)`。
- 如果 `blkno < relsize`，调用 `smgrwrite(..., skipFsync = true)`。

这里所有 smgr write/extend 都传 `skipFsync = true`。 所以 bulk writer 必须在 finish 阶段补偿 durability。

`smgr_bulk_finish()` 先 flush pending pages。 然后根据 relation 类型处理 fsync。

temp relation 不需要 fsync。

`use_wal == false` 时，可能是 unlogged relation，也可能是 permanent relation 在 `wal_level=minimal` 下跳过逐页 WAL。 bulk writer 不能在这里区分两者。 它保守调用 `smgrregistersync()`。
unlogged relation clean shutdown 需要 checkpoint fsync。 permanent relation minimal WAL 的 commit-time pending sync 逻辑会负责最终选择 fsync 或 WAL-log whole relation。

`use_wal == true` 时，它已经 WAL-log 了 pages。 但它写文件时跳过了逐 segment fsync registration。 所以 finish 阶段要么 `smgrregistersync()`，要么 `smgrimmedsync()`。

关键竞态是 concurrent checkpoint。 如果 checkpoint 在 bulk write 过程中发生，它可能已经错过 bulk writer 早期写出的 pages。 crash 后 recovery 从 checkpoint redo pointer 开始，早期 WAL record 可能不会重放。
如果那些 pages 又没有 fsync，就会丢。

bulk writer 的处理方式：
1. 设置 `DELAY_CHKPT_START`，防止新的 checkpoint 从检查和注册之间开始。
2. 比较 `start_RedoRecPtr` 和当前 `GetRedoRecPtr()`。
3. 如果不相等，说明 checkpoint 已经发生过，调用 `smgrimmedsync()`。
4. 如果相等，调用 `smgrregistersync()`。
5. 清掉 `DELAY_CHKPT_START`。

这解释了为什么 `skipFsync` 是可控优化，而不是绕过持久性。

---

## 18. WAL redo 中的 smgr 边界
本节主文件是 `smgr.c`、`md.c`、`bulk_write.c`。 但 create/truncate 的 redo 入口在 `src/backend/catalog/storage.c`。 这是因为 `RM_SMGR_ID` 的 redo routine 是 `smgr_redo()`。

`storage_xlog.h` 说明：
- file creation 和 truncation 在这里记录。
- deletion actions 由 `xact.c` 处理，因为 deletion 是 transaction commit 的一部分。

`log_smgrcreate()` 写 `XLOG_SMGR_CREATE`。 redo 时 `smgr_redo()`：
- 解析 `xl_smgr_create`
- `smgropen()`
- `smgrcreate(..., true)`

这里的 `true` 传到 `mdcreate()`。 这使 create redo 幂等。 文件已经存在时，redo 可以打开已有文件。

`XLOG_SMGR_TRUNCATE` 的 redo 更复杂。 `smgr_redo()` 会：
- `smgropen()`
- `smgrcreate(... MAIN_FORKNUM, true)`，必要时强制创建 relation。
- 在真正 truncate 前 `XLogFlush(lsn)`。
- 根据 flags 准备 main、FSM、VM fork 的 truncate。
- 调用 `XLogTruncateRelation()` 清理 xlogutils 的 invalid-page 状态。
- 在 critical section 中调用 `smgrtruncate()`。
- 如有需要，更新 FSM 上层页面。

为什么 redo truncate 前要 `XLogFlush(lsn)`？ 注释说 truncate 是不可逆的。 一旦数据文件被截短，就不能让 minimum recovery point 落在这条 WAL record 之前。
正常 buffer write 由 buffer manager 保证 WAL-first rule。 truncate 绕过普通 dirty buffer 写回，所以必须手动 flush。

`XLogReadBufferExtended()` 也和本节相关。 恢复中读某个 block 时，它会：
- `smgropen()`
- `smgrcreate(..., true)`
- `smgrnblocks()`
- 如果 block 已存在就读 buffer。
- 如果 block 不存在且 mode 允许 zero/extend，就调用 `ExtendBufferedRelTo(... EB_PERFORMING_RECOVERY | EB_SKIP_EXTENSION_LOCK)`。

注释明确说 recovery 中不需要 relation extension lock。 这是因为 recovery 进程按 WAL 顺序重放，普通并发修改者不存在。 这和 normal execution 下通过 buffer manager extension lock 控制扩展竞态形成对比。

---

## 19. EOF、短读和扩展竞态
这一节把几个容易混淆的边界放在一起。

第一，短读不等于 EOF。 `mdreadv()` 如果读到正数但不足请求大小，会调整 offset 和 iovec 继续读。 只有 `nbytes == 0` 才进入 EOF/partial block at EOF 分支。

第二，EOF 通常是错误。 正常读路径不应该读不存在的 block。 如果读到 EOF，默认报 data corrupted。
`zero_damaged_pages` 和 `InRecovery` 的 zero-fill 分支是容错/历史边界。 当前源码还在这条路径放了 `Assert(false)`，说明开发者认为它不应该在 recovery 中被依赖。

第三，短写要继续写。 `mdwritev()` 对正数短写继续写剩余 iovec。 如果 kernel 最终返回 `ENOSPC`，再报磁盘空间错误。

第四，普通扩展由上层控制并发。 `smgr` 和 `md` 不自己拿 relation extension lock。 共享 buffer extension 路径在 buffer manager 中安排 buffer tag、IO in progress 和 extension lock。
`mdextend()` 只假设 caller 正在做合法 extension。

第五，recovery 扩展不需要 extension lock。 `XLogReadBufferExtended()` 调用 `ExtendBufferedRelTo()` 时使用 `EB_SKIP_EXTENSION_LOCK`。 原因是 recovery 顺序重放，没有普通并发 extension。

第六，segment 创建要维护前序 segment full。 `_mdfd_getseg()` 在创建 target segment 前，如果前一个 segment 小于 `RELSEG_SIZE`，会用 zero block pad 到最后一个 block。 否则会出现 `.1` 存在但第 0 segment 不满的非法布局。

第七，size cache 只在能证明正确时更新。 `smgrextend()` 只有在 cached size 等于扩展 blocknum 时才递增。 `smgrzeroextend()` 只有在 cached size 等于起始 blocknum 时才加 nblocks。
否则 invalid cache。 这避免非连续扩展或 stale cache 导致错误大小。

第八，truncate 通过 smgr invalidation 处理其他 backend 的旧 fd。 `smgrtruncate()` 先发 `CacheInvalidateSmgr()`，再改磁盘。 其他 backend 收到 invalidation 后会 close smgr fd。
这减少对已截断或已删除 segment 的悬挂引用。

第九，`mdwriteback()` 特意不重新打开 segment。 它使用 `EXTENSION_DONT_OPEN`。 如果 segment 没打开，直接 return。
注释解释这是为了避免和 `PROCSIGNAL_BARRIER_SMGRRELEASE` 竞态。 writeback 只是 hint，不值得为了 hint 打开一个可能即将被释放或删除的 fd。

---

## 20. 错误边界速查
`smgr.c` 层的通用约定：
- 大多数 smgr subfunction 失败是 `ERROR`。
- unlink 失败应该是 `WARNING`。
- bootstrap 和 WAL recovery 中允许一些普通运行时不允许的情况。

`mdcreate()`：
- 普通 create 发现文件已存在是错误。
- redo create 发现文件已存在可以 open existing。
- 非 temp create 后注册 dirty segment。

`mdextend()`：
- `blocknum == InvalidBlockNumber` 是 program limit exceeded。
- `FileWrite()` 负数是 file access error。
- short write 是 disk full 类错误。
- 写完 segment 不能超过 `RELSEG_SIZE`。

`mdzeroextend()`：
- `blocknum + nblocks >= InvalidBlockNumber` 是 program limit exceeded。
- `FileFallocate()` 或 `FileZero()` 失败报 file access error。
- 每轮不能跨过 segment boundary。

`mdreadv()`：
- read crosses segment boundary 是 `ERROR`。
- `FileReadV()` 负数是 `ERROR`。
- EOF 默认是 data corrupted。
- `zero_damaged_pages` 或 recovery 的 zero-fill 分支存在，但当前源码认为不可依赖。

`mdwritev()`：
- write crosses segment boundary 是 `ERROR`。
- `FileWriteV()` 负数是 `ERROR`。
- `ENOSPC` 添加 free disk space hint。
- short write 通过循环继续尝试。

`mdnblocks()` 和 `_mdfd_getseg()`：
- segment 大于 `RELSEG_SIZE` 是 `FATAL`。
- previous segment 不满而 caller 不允许创建后续 segment，是 `ERROR` 或 NULL。
- `EXTENSION_RETURN_NULL` 场景会返回 NULL 并可能设置 `errno = ENOENT`。

`mdtruncate()`：
- 非 recovery 下请求 truncate 到大于当前大小，是 `ERROR`。
- recovery 下同一情况直接忽略。
- `FileTruncate()` 失败是 `ERROR`。

`mdunlink()`：
- unlink 和 truncate 删除路径上的失败大多是 `WARNING`。
- `ENOENT` 通常被视为可接受。
- redo 下 relation 已经不存在也不奇怪。

`smgr_redo()`：
- unknown op code 是 `PANIC`。
- truncate redo 在真正截断前 flush WAL record LSN。
- truncate 的物理操作放在 critical section 中。

---

## 21. 主流程源码 walkthrough
按一条对象生命周期读，不要按文件名平铺。

```text
RelationCreateStorage()
  -> smgropen()
     -> 建立 backend-local SMgrRelation，不打开 segment
  -> smgrcreate()
     -> smgrsw[] 分派到 mdcreate()
  -> mdcreate()
     -> 创建 fork 的第 0 segment，非 temp 登记 dirty segment
  -> log_smgrcreate()
     -> 写 XLOG_SMGR_CREATE，让 redo 能重建 fork 存在性
```

扩展和写入继续沿同一条链：

```text
buffer manager / bulk writer / redo
  -> smgrextend() / smgrwrite()
  -> mdextend() / mdwritev()
  -> _mdfd_getseg()
     -> 根据 blocknum 定位或创建 segment
  -> FileWriteV()
  -> register_dirty_segment()
```

截断和删除是反向生命周期：

```text
RelationTruncate()
  -> XLogFlush(truncate LSN)
  -> smgrtruncate()
  -> mdtruncate()
     -> truncate active tail，留下 0 长度 inactive segment
     -> register_dirty_segment()

smgrDoPendingDeletes()
  -> smgrdounlinkall()
  -> mdunlink()
     -> main fork 首段延迟 unlink
     -> 额外 segment forget 后尽量立即 unlink
```

源码阅读顺序保持为：`smgr.h` 的 `SMgrRelationData`，`smgr.c` 的 `smgrsw[]` 和 `smgropen()`，`md.c` 的 segment invariant、`mdcreate()`、`_mdfd_getseg()`、`mdextend()`/`mdreadv()`/`mdwritev()`、`mdtruncate()`/`mdunlink()`，最后接 `bulk_write.c` 和 `storage.c` redo。

## 22. 生命周期 / ownership / cleanup
`SMgrRelation` 是 backend-local handle，owner 是当前 backend 的 smgr hash table。relcache 可以 pin 它；未 pin 的 entry 可被释放。它保存 locator、每 fork cached size、每 fork 已打开 segment 数组和 smgr switch index。

segment 文件的 owner 是 relation fork 生命周期。`mdcreate()` 创建第 0 segment，`mdextend()`/`mdzeroextend()` 创建后续 segment，`mdclose()` 只关闭当前 backend 打开的 VFD，不删除 relation 文件。删除语义来自 `mdunlink()`、transaction pending delete、checkpoint unlink request 和 crash recovery。

cleanup 分两类：正常结束时 caller 显式 close 或 unlink；ERROR/abort 后由 transaction pending delete、ResourceOwner、smgr release、checkpoint sync/unlink request 或 redo/recovery 路径收尾。unlink 失败通常降成 `WARNING`，因为很多删除发生在事务结尾 cleanup，不能再把已经决定的事务结果改成失败。

`bulk_write.c` 是特殊 ownership：它直接写 relation 文件并传 `skipFsync = true`，但 `smgr_bulk_finish()` 必须用 `DELAY_CHKPT_START`、redo pointer 检查、`smgrregistersync()` 或 `smgrimmedsync()` 补齐 checkpoint 竞态。

## 23. 成本、资源与观测诊断
成本随四个变量扩张：relation fork 的 block 数决定 segment 数；同时打开的 relation/fork 数决定 `SMgrRelation` 和 `MdfdVec` 数组压力；写入量决定 fsync request 数；bulk write、truncate 和 redo 会把成本传播到 checkpointer、WAL flush、fd.c VFD LRU 和 storage invalidation。

能直接观察的是文件系统里的 `.1`、`.2` segment、`pg_relation_size()`、`pg_relation_filepath()`、checkpoint 日志和 `pg_stat_io` 中的数据文件 I/O。只能推断的是当前 backend 打开了多少 `MdfdVec`、size cache 是否可信、writeback hint 是否被跳过。需要断点的状态包括 `_mdfd_getseg()` behavior、`register_dirty_segment()`、`mdtruncate()` 中 inactive segment 的形成。

诊断 relation storage 问题时先问：是 caller 读写了不存在的 block，还是 segment layout 被破坏；是普通运行时路径，还是 recovery 允许创建缺失 relation；是 fd pressure，还是 fsync/checkpoint 压力；是 `skipFsync` 优化被正确补偿，还是漏了 durability contract。

## 24. 常见误区
- `smgropen()` 不打开文件；它只创建或查找 backend-local `SMgrRelation`。
- `md_num_open_segs == N` 只表示当前 backend 打开了 N 个 segment，不表示磁盘上只有 N 个 active segment。
- `smgr_cached_nblocks` 不是普通运行时的共享大小真相；当前主要在 recovery 能可靠使用。
- 1GB segment 不是 `md.c` 写死的 magic number，而是默认 `RELSEG_SIZE * BLCKSZ`。
- truncate 不等于立即 unlink；`mdtruncate()` 会制造 0 长度 inactive segment，后续 sync 和 unlink 还要处理。
- `skipFsync = true` 不是放弃持久化，而是 caller 承诺用另一条路径补偿。
- 当前基线导出的是 `mdreadv()`/`mdwritev()`；单页 `smgrread()`/`smgrwrite()` 是 `smgr.h` 的 inline wrapper。

## 25. 课堂实验
实验 1：手算 segment 并回源码校验。

```text
BLCKSZ = 8192
RELSEG_SIZE = 131072
```

计算 block `0`、`131071`、`131072`、`262144` 的 segment 和 offset。再读 `_mdfd_segpath()`、`mdreadv()`、`mdwritev()`，确认 segment 0 无 `.0` 后缀，segment 1 才是 `.1`。

实验 2：小 segment build 观察 `.1`、`.2`。

```bash
cd /home/nail/postgres-lab
meson setup build-segtest -Dsegsize_blocks=16 -Dcassert=true
ninja -C build-segtest
```

用单独数据目录 initdb，创建大表并插入足够数据，观察 `base/dbOid/relfilenode*`。把看到的文件和 `_mdfd_getseg()`、`mdnblocks()` 对齐。不要复用生产数据目录。

实验 3：断点观察 truncate 和 fsync 注册。

```text
break mdtruncate
break FileTruncate
break register_dirty_segment
break mdunlink
```

触发表重写或 truncate，观察哪些 segment 变成 0 长度、哪些请求进入 dirty sync 或 delayed unlink。

## 26. 讨论题
1. 为什么 `smgr.c` 仍保留 switch 层，即使当前基线只有 `md.c` 一个实现？
2. `_mdfd_getseg()` 为什么要在创建后续 segment 前 pad 前序 segment？
3. 为什么 `mdreadv()` 在 recovery 中允许创建缺失 relation，而普通运行时不允许？
4. truncate 为什么要先 flush WAL，再改文件长度？
5. 为什么普通 main fork 首段需要 delayed unlink，而额外 segment 可以 forget 后立即 unlink？
6. `skipFsync` 的正确性责任从 md 层转移到了哪个 caller？
7. 哪些错误应该是 `ERROR`，哪些 cleanup 错误只能是 `WARNING`？

## 27. 本节小结
`smgr.c` 是 relation file I/O 的稳定入口和 backend-local handle cache；`md.c` 是当前基线唯一 concrete storage manager，把 fork/block 映射成 segment 文件操作。

核心状态是 `SMgrRelation`、每 fork 的 open segment 数组、segment layout invariant 和 dirty sync request。ownership 分层：smgr handle 属于 backend；segment 文件属于 relation fork；删除和持久化由 transaction cleanup、md unlink、sync request、checkpoint 和 redo 共同收尾。

主流程是 create 第 0 segment、按 block 扩展/读写目标 segment、写后登记 fsync、truncate 后 fsync 文件长度、删除时区分 delayed unlink 和 immediate unlink。异常路径包括 redo create 幂等、recovery 创建缺失 relation、短读短写、disk full、inactive segment 和 bulk write checkpoint 竞态。

可观测状态主要在文件系统、SQL relation path/size、checkpoint 日志和 gdb 断点中。可迁移规律是：上层只应该表达“哪个 fork 的哪个 block 应该存在或被修改”；底层必须独立维护“这个 block 落在哪个 segment、怎样打开、怎样扩展、怎样截断、怎样登记 fsync，以及错误停在哪里”。
