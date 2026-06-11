# PostgreSQL relation fork 与 block addressing

## 课程定位

前置知识：已经看过 WAL record、redo contract、full-page image 和 page LSN；也知道 buffer manager 最终读写的是某个 relation fork 的 block。

本节唯一主问题：

```text
一个 SQL relation 在 catalog 里有名称、OID、relfilenode、tablespace 和 persistence，storage manager 为什么仍然需要一套独立的 RelFileLocatorBackend + ForkNumber + BlockNumber 坐标？
```

核心矛盾：上层希望用稳定的 SQL 对象身份表达表、索引和临时对象；底层必须用 crash-safe、backend-aware、fork-aware、segment-aware 的物理坐标定位文件，并且不能让 catalog 简写、临时对象、本地 fork 和 WAL/recovery 边界互相污染。

一句话运行模型：

```text
storage manager 用 RelFileLocatorBackend 定位物理 relation，用 ForkNumber 选择 main/fsm/vm/init fork，用 BlockNumber 定位 fork 内 page，再用 RELSEG_SIZE 和 BLCKSZ 换算成 segment 文件名与文件内 offset。
```

学完后应能判断：`pg_class.relfilenode` 与 `oid` 为什么不同；temp relation 为什么要进入 `RelFileLocatorBackend.backend`；main/fsm/vm/init fork 各自服务什么目的；`BlockNumber` 如何换算成 segment 和 offset；permanent、unlogged、temp 的持久化边界在哪里。

源码基线：本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。

## 1. 本节在总主线中的位置

前面 WAL/recovery 课程说明了如何描述和重放 page 修改。本节把注意力放回 storage 地址本身：WAL、buffer manager、smgr 和 md 层最终都需要一个能稳定定位物理文件和 block 的坐标。

这节是后续 smgr/md segment lifecycle、fd cache、fsync queue、heap/index physical layout 的基础。不要从 SQL 表名开始读，要从 storage manager 需要什么物理地址开始读。

## 2. 核心矛盾与一句话运行模型

catalog 层为了用户和元数据管理需要 OID、relfilenode、tablespace、persistence 等概念；storage 层为了实际读写文件，需要的是没有 catalog 简写、能区分 temp backend、能区分 fork、能跨 segment 换算 block offset 的物理坐标。

地址层次如下：

```text
RelFileLocator
  -> RelFileLocatorBackend
  -> SMgrRelation
  -> ForkNumber
  -> BlockNumber
  -> RELSEG_SIZE / BLCKSZ
  -> segment filename + byte offset
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节读什么 |
| --- | --- | --- |
| 1 | `src/include/storage/relfilelocator.h` | `RelFileLocator`、`RelFileLocatorBackend`、backend-local temp relation 身份。 |
| 2 | `src/include/common/relpath.h`、`src/common/relpath.c` | `ForkNumber`、fork name、relation path 和文件名生成。 |
| 3 | `src/include/storage/block.h` | `BlockNumber`、`InvalidBlockNumber`、block 地址边界。 |
| 4 | `src/include/storage/smgr.h`、`src/backend/storage/smgr/smgr.c` | `SMgrRelation`、`smgropen()`、fork size cache、storage manager 抽象。 |
| 5 | `src/backend/storage/smgr/md.c`、`src/include/storage/md.h` | segment 文件打开、block 到 segment/offset 的换算、md 层 invariant。 |
| 6 | `src/backend/catalog/storage.c`、`src/include/catalog/storage_xlog.h` | relation create/truncate/unlink 的 storage WAL 边界。 |
| 7 | `src/include/catalog/pg_class.h`、`src/include/utils/rel.h` | catalog identity、persistence 与 storage identity 的衔接。 |
| 8 | `src/backend/storage/file/reinit.c`、`src/backend/access/heap/heapam_handler.c`、`src/backend/catalog/index.c` | unlogged/temp 初始化和访问方法调用点辅助核对。 |

## 4. 关键结论：relation 物理地址

PostgreSQL 的 relation 文件地址不是一个单字段。
它是一个分层坐标。
最上层是 relation 的物理身份。
这个身份由 `RelFileLocator` 表示。
`RelFileLocator` 有三个字段：
- tablespace OID
- database OID
- relation file number

这三个字段还不够定位所有文件。
因为临时 relation 是 backend-local 的。
因此 storage manager 使用 `RelFileLocatorBackend` 再加一个 `backend`。
`backend == INVALID_PROC_NUMBER` 表示 regular relation。
`backend != INVALID_PROC_NUMBER` 表示 backend-local temp relation。

在内存中，`smgropen()` 返回 `SMgrRelation`。
`SMgrRelation` 是 storage manager 的物理句柄缓存。
它的 hash key 就是 `RelFileLocatorBackend`。
它还缓存每个 fork 的已知大小和 md.c 打开的 segment file descriptor。

一个 relation 不是一个文件。
一个 relation 由多个 fork 组成。
这个基线的合法 fork 是：
- `MAIN_FORKNUM`
- `FSM_FORKNUM`
- `VISIBILITYMAP_FORKNUM`
- `INIT_FORKNUM`

fork 再被分成 segment。
一个 segment 最多有 `RELSEG_SIZE` 个 block。
一个 block 的大小是 `BLCKSZ` byte。
因此 block address 到文件位置的基本换算是：

```c
segno = blocknum / RELSEG_SIZE;
segoff = blocknum % RELSEG_SIZE;
seekpos = (pgoff_t) BLCKSZ * segoff;
```

默认构建中，`BLCKSZ = 8192`。
默认 relation segment size 是 1GB。
所以默认 `RELSEG_SIZE = 1024 * 1024 * 1024 / 8192 = 131072`。
这意味着一个 fork 的 block 0 到 block 131071 位于第 0 个 segment。
block 131072 位于第 1 个 segment，文件名追加 `.1`。

文件名由 relation path、fork 后缀和 segment 后缀三层拼出来。
main fork 没有 fork 后缀。
fsm fork 追加 `_fsm`。
vm fork 追加 `_vm`。
init fork 追加 `_init`。
segment 0 没有 segment 后缀。
segment 1 追加 `.1`。
segment 2 追加 `.2`。
以此类推。

所以默认表空间中的普通 relation 可能长这样：

```text
base/16384/24576
base/16384/24576_fsm
base/16384/24576_vm
base/16384/24576.1
base/16384/24576_vm.1
```

临时 relation 会把 backend proc number 放进文件名：

```text
base/16384/t7_24576
base/16384/t7_24576_fsm
```

unlogged relation 不使用 temp 文件名。
它使用普通 relation 的路径形态。
它的 main fork 修改通常不 WAL-log。
但它有一个 `init` fork，用于 crash 后把 main fork 重置回初始内容。
这个 `init` fork 的创建和内容需要 WAL 或同步保护。
这就是 unlogged relation 和 temp relation 最容易混淆的边界。

## 5. 从 SQL 名字到物理文件号

SQL 层说的是 relation name。
catalog 层有 `pg_class.oid`。
storage 层需要的是 `pg_class.relfilenode` 对应的物理文件号。
在这个基线的 `relfilelocator.h` 注释中，字段名叫 `relNumber`。
它的类型是 `RelFileNumber`。
`RelFileNumber` 在 `relpath.h` 中 typedef 为 `Oid`。

这并不意味着 relation OID 和 relfilenode 是同一个身份。
`pg_class.oid` 是 catalog tuple 的对象身份。
`pg_class.relfilenode` 是这个 relation 当前使用的物理文件号。
很多操作可以给同一个 logical relation 换一组新物理文件。
例如 rewrite、cluster、某些 alter table、reindex 或 table access method 的 copy path。
换文件时，catalog OID 可以保持不变。
物理 relfilenode 可以改变。

`RelFileLocator` 的注释直接强调：
`relNumber` 对应 `pg_class.relfilenode`，不是 `pg_class.oid`。
它还强调 `relNumber` 只在一个 database 和一个 tablespace 内唯一。
所以不能只拿一个 relfilenode 数字就说全局定位到某个文件。
你至少还需要 tablespace 和 database。

`RelFileLocator` 是：

```c
typedef struct RelFileLocator
{
    Oid spcOid;
    Oid dbOid;
    RelFileNumber relNumber;
} RelFileLocator;
```

这三个字段都是物理访问需要的字段。
`spcOid` 对应 `pg_tablespace.oid`。
`dbOid` 对应 `pg_database.oid`。
共享 relation 的 `dbOid` 是 0。
共享 relation 只能在 global tablespace。
`relNumber` 是 relation 的物理文件号。

这里有两个 catalog 简写不能进入 `RelFileLocator`。
第一个是 `pg_class.reltablespace = 0`。
catalog 中 0 表示使用 database 默认表空间。
但是 `RelFileLocator.spcOid` 必须填真实表空间 OID。
第二个是 `pg_class.relfilenode = 0`。
catalog 中 0 表示 mapped relation，真实 filenode 从 relmapper 取得。
但是 `RelFileLocator.relNumber` 不能是这个简写。
它必须是实际可用于文件名的 relfilenode。

这就是 storage 层的纪律：
进入 storage manager 的 identity 必须已经消解完 catalog shorthand。
storage manager 不负责知道“默认表空间是谁”。
storage manager 也不负责把 mapped catalog relation 重新映射成真实 filenode。
这些必须在更高层完成。

## 6. `RelFileLocatorBackend`

`RelFileLocator` 不含 backend。
对普通 relation，这正好。
普通 relation 可以被多个 backend 访问。
文件名也不带 backend 标识。
但 temporary relation 不一样。
临时 relation 的物理文件属于某个 backend 或某个并行 query leader。
它的 storage lifetime 是 backend-local。

`relfilelocator.h` 因此定义：

```c
typedef struct RelFileLocatorBackend
{
    RelFileLocator locator;
    ProcNumber backend;
} RelFileLocatorBackend;
```

`backend == INVALID_PROC_NUMBER` 表示 regular relation。
`backend != INVALID_PROC_NUMBER` 表示 backend-local relation。
宏 `RelFileLocatorBackendIsTemp()` 也只是检查这个字段不是 invalid。
这个判断被 `SmgrIsTemp()` 包装给 smgr 使用。

源码注释给了很强的边界：
backend-local relations are always transient。
它们在数据库 crash 后移除。
它们 never WAL-logged。
它们 never fsync'd。
这句话是理解 temp storage 的核心。

`ProcNumber` 定义在 `src/include/storage/procnumber.h`。
`INVALID_PROC_NUMBER` 是 -1。
`ProcNumberForTempRelations()` 通常返回 `MyProcNumber`。
在并行 worker 中，它返回 parallel leader 的 proc number。
这保证同一 session 下的 parallel worker 使用同一组 temp relation 文件。

默认表空间下，普通 relation 路径可能是：

```text
base/16384/24576
```

同一个 relNumber 如果是 temp relation，路径可能是：

```text
base/16384/t7_24576
```

这里的 `t7_` 来自 `GetRelationPath()`。
7 是 proc number，不是 PID。
proc number 是 backend 在 proc array 中的编号。
它可以在 backend 退出后被重用。
这没有问题，因为 temp relation 文件本来就是 backend-local 的 transient 文件。

不要把 temp 的 `backend` 字段理解为 SQL schema 名字。
SQL 层有 `pg_temp_N` namespace。
storage 层只关心文件名里的 proc number。
这个 proc number 进入 `RelFileLocatorBackend`，再进入 `relpathbackend()`。

## 7. `SMgrRelation` 是什么

`SMgrRelation` 不是 `Relation`。
`Relation` 是 relcache 里的 catalog-aware 结构。
`SMgrRelation` 是 storage manager 的物理文件句柄缓存。

`smgr.h` 的注释说，`smgr.c` 维护一张 `SMgrRelation` 表。
这个表缓存的是 file handles。
调用 `smgropen()` 会创建或查找一个 entry。
调用 `smgrdestroy()` 会销毁 entry。
这两个动作本身不代表一定发生 I/O。
真正打开底层文件通常延迟到 md.c 的 fork 或 segment 访问路径中。

`SMgrRelationData` 的第一个字段是：

```c
RelFileLocatorBackend smgr_rlocator;
```

注释强调它是 hash table lookup key，所以必须放在结构体第一位。
这说明 `SMgrRelation` 的身份就是 `RelFileLocatorBackend`。
不是 SQL 名字。
不是 catalog OID。
也不是某个 `Relation` 指针。

`SMgrRelationData` 还有这些公开字段：
- `smgr_targblock`
- `smgr_cached_nblocks[MAX_FORKNUM + 1]`

`smgr_targblock` 是当前插入目标 block。
`smgr_cached_nblocks` 是每个 fork 的 last known size。
源码注释说，当前这个大小缓存只在 recovery 期间可靠。
原因是普通运行中没有 fork extension 的 cache invalidation。
所以普通路径不要把它当作强一致大小来源。

`SMgrRelationData` 的 md.c 私有字段包括：
- `md_num_open_segs[MAX_FORKNUM + 1]`
- `md_seg_fds[MAX_FORKNUM + 1]`

这两组数组按 fork 分开。
每个 fork 都可能有自己的 segment 文件列表。
main fork 的 segment 0 和 vm fork 的 segment 0 是不同文件。
`SMgrRelation` 需要分别管理它们。

`smgr.c` 里只有一个 storage manager implementation：
`smgrsw[0]` 指向 md.c。
字段名还叫 magnetic disk。
但 md.c 注释已经说明，今天它实际上是 Unix-like filesystem storage manager。
它把 smgr API 映射到常规文件系统 API。

`smgropen(RelFileLocator rlocator, ProcNumber backend)` 的流程很直接：
先构造 `RelFileLocatorBackend`。
再用它查本 backend 的 hash table。
找不到就创建 entry。
新 entry 初始化每个 fork 的 cached nblocks 为 `InvalidBlockNumber`。
新 entry 的 `smgr_which` 设置为 0。
然后调用 md.c 的 `mdopen()` 初始化 per-fork open segment 数组。

这也解释了为什么 buffer manager、checkpointer、WAL redo 都可以绕过 relcache 直接使用 smgr。
只要它们有 `RelFileLocator`、backend number、fork number 和 block number，就能定位文件。
它们不需要 SQL 名字。

## 8. `ForkNumber`

fork 是 relation 物理文件集合中的一个分支。
`ForkNumber` 定义在 `relpath.h`：

```c
typedef enum ForkNumber
{
    InvalidForkNumber = -1,
    MAIN_FORKNUM = 0,
    FSM_FORKNUM,
    VISIBILITYMAP_FORKNUM,
    INIT_FORKNUM,
} ForkNumber;
```

`MAX_FORKNUM` 定义为 `INIT_FORKNUM`。
很多数组都写成 `MAX_FORKNUM + 1`。
例如 `SMgrRelationData.smgr_cached_nblocks`。
再例如 `SMgrRelationData.md_num_open_segs`。
这说明 fork number 被当作小整数数组下标使用。
新增 fork 不是只加 enum 就结束。
`relpath.h` 注释要求更新 `MAX_FORKNUM`、`FORKNAMECHARS` 和 `src/common/relpath.c` 里的 `forkNames`。

`forkNames` 在 `src/common/relpath.c` 中：

```c
[MAIN_FORKNUM] = "main"
[FSM_FORKNUM] = "fsm"
[VISIBILITYMAP_FORKNUM] = "vm"
[INIT_FORKNUM] = "init"
```

但是文件名规则故意不把 `main` 写出来。
`GetRelationPath()` 对 `MAIN_FORKNUM` 使用裸 relNumber。
只有 non-main fork 才追加 `_forkname`。
所以 main fork 是 `24576`，不是 `24576_main`。

`forkname_chars()` 也从 fork number 1 开始检查。
它故意不把 `main` 当成文件名后缀识别。
这和文件名规则完全一致。
`main` 是逻辑 fork 名。
磁盘文件名中没有 `_main`。

四个 fork 的含义如下。

`main` fork 存储 relation 的主要数据。
对 heap table，它存储 heap page。
对 btree index，它存储 index page。
对 sequence，它也有自己的 page 格式。
storage manager 不解释 page 内容。
它只知道这是 fork 0。

`fsm` fork 是 free space map。
它记录 relation 页面上的可用空间摘要。
heap insert、vacuum 等路径会用它寻找可插入页面或维护空闲空间信息。
storage manager 只负责把它作为独立 fork 文件读写。
`storage.c` 的 truncate 路径会在 heap truncate 时准备并截断 FSM。

`vm` fork 是 visibility map。
它记录 heap block 的 all-visible 和 all-frozen 等状态。
vacuum、index-only scan 和 freeze 相关逻辑会使用它。
和 FSM 一样，storage manager 不解释具体 bit。
但 physical truncate 需要把 VM 一起收缩到与 heap block 范围匹配。

`init` fork 主要服务 unlogged relation。
crash 后 unlogged relation 的 main fork 不能被 WAL 还原。
PostgreSQL 用 init fork 保存一个可恢复的初始镜像。
启动恢复过程中，`reinit.c` 会清理带 init fork 的 relation 的其它 fork，再把 init fork 拷贝成 main fork。
所以 init fork 是 unlogged 持久性边界的一部分。

不要把 fork 和 segment 混为一谈。
fork 是逻辑分支。
segment 是单个 fork 太大时切开的文件片。
`24576_vm.1` 的含义是：
relation file number 24576。
fork 是 vm。
segment number 是 1。
它不是“relation 24576 的第 1 个 vm block”。

## 9. 文件名生成

文件名生成的入口是 `GetRelationPath()`。
它接受：
- `dbOid`
- `spcOid`
- `relNumber`
- `procNumber`
- `forkNumber`

`relpath.h` 提供三个包装宏：
- `relpathbackend(rlocator, backend, forknum)`
- `relpathperm(rlocator, forknum)`
- `relpath(rlocatorBackend, forknum)`

`relpathperm()` 固定传 `INVALID_PROC_NUMBER`。
`relpath()` 从 `RelFileLocatorBackend` 同时取 locator 和 backend。
md.c 通常用 `relpath(reln->smgr_rlocator, forknum)`。

`GetRelationPath()` 分三类 tablespace。
第一类是 global tablespace。
共享系统 relation 位于：

```text
global/relNumber
global/relNumber_fsm
global/relNumber_vm
```

global 路径要求 `dbOid == 0`。
它也要求 `procNumber == INVALID_PROC_NUMBER`。
共享 relation 不是 temp relation。

第二类是 default tablespace。
路径位于：

```text
base/dbOid/relNumber
base/dbOid/relNumber_fsm
base/dbOid/relNumber_vm
base/dbOid/relNumber_init
```

如果是 temp relation，则路径变成：

```text
base/dbOid/tprocNumber_relNumber
base/dbOid/tprocNumber_relNumber_fsm
base/dbOid/tprocNumber_relNumber_vm
```

第三类是非默认 tablespace。
路径位于：

```text
pg_tblspc/spcOid/TABLESPACE_VERSION_DIRECTORY/dbOid/relNumber
```

非默认 tablespace 的 temp relation 同样带 `tprocNumber_` 前缀。
差异只是目录前缀换成 `pg_tblspc/...`。

`TABLESPACE_VERSION_DIRECTORY` 定义在 `relpath.h`。
它包含 PostgreSQL major version 和 catalog version。
这保证不同 major 或 catalog layout 的数据目录不会共用同一个 tablespace 子目录。

md.c 再在 `GetRelationPath()` 结果上追加 segment 后缀。
`_mdfd_segpath()` 的规则很简单：
如果 `segno > 0`，追加 `.%u`。
否则直接使用 relation path。

因此 segment 0 没有 `.0` 后缀。
segment 1 才是 `.1`。
这一点和 main fork 没有 `_main` 后缀类似。
默认形态总是最短文件名。

文件名拆解可以按这个顺序做：
先看目录，判断 tablespace 和 database。
再看 basename 是否以 `tN_` 开头，判断 temp backend。
再看是否有 `_fsm`、`_vm`、`_init`，判断 fork。
没有 fork 后缀就是 main fork。
最后看是否有 `.N`，判断 segment。
没有 segment 后缀就是 segment 0。

例如：

```text
base/16384/24576_vm.3
```

它表示：
- default tablespace
- database OID 16384
- relNumber 24576
- fork `VISIBILITYMAP_FORKNUM`
- segment 3
- regular relation

再例如：

```text
base/16384/t12_24576_fsm.1
```

它表示：
- default tablespace
- database OID 16384
- proc number 12 的 temp relation
- relNumber 24576
- fork `FSM_FORKNUM`
- segment 1

再例如：

```text
global/1262
```

它表示：
- shared relation
- global tablespace
- `dbOid = 0`
- main fork
- segment 0
- regular relation

`reinit.c` 的 `parse_filename_for_nontemp_relation()` 可以作为反向解析的练习材料。
它只解析 non-temp relation 文件名。
它拒绝前导零。
它用 `_forkname` 识别非 main fork。
它用 `.segno` 识别 segment。
它不接受 temp 的 `tN_` 前缀。
这正好说明 unlogged reset 处理的是普通命名空间下的 unlogged relation，不是 backend-local temp relation。

## 10. `BlockNumber`

`BlockNumber` 定义在 `block.h`：

```c
typedef uint32 BlockNumber;
```

有效 block 编号从 0 到 `0xFFFFFFFE`。
`0xFFFFFFFF` 是 `InvalidBlockNumber`。
`MaxBlockNumber` 是 `0xFFFFFFFE`。
注释还说 `InvalidBlockNumber` 和 buffer manager 里的 `P_NEW` 是同一个值。

这意味着 block number 是 fork 内编号。
不是 relation 全局所有 fork 的编号。
也不是 segment 内编号。
`BlockNumber 0` 在 main fork、fsm fork 和 vm fork 中可以同时存在。
它们对应不同文件。

`BlockIdData` 是 on-disk storage type。
它把一个 block number 拆成两个 `uint16`。
`BlockNumber` 才是计算用的类型。
`BlockIdSet()` 把高 16 bit 放进 `bi_hi`，低 16 bit 放进 `bi_lo`。
`BlockIdGetBlockNumber()` 再组合回来。
这一层常见于 item pointer 等 on-disk 结构。

本节更关心 `BlockNumber` 到文件位置的换算。
md.c 在多个路径里使用同一套公式：

```c
targetseg = blkno / ((BlockNumber) RELSEG_SIZE);
seekpos = (pgoff_t) BLCKSZ * (blkno % ((BlockNumber) RELSEG_SIZE));
```

`targetseg` 决定打开哪个 segment 文件。
`seekpos` 决定在该 segment 文件内从哪个 byte 开始读写。
一次读写一个 block 时，长度就是 `BLCKSZ`。

默认构建中：
- `BLCKSZ = 8192`
- `RELSEG_SIZE = 131072`
- 每个 segment 最大 `8192 * 131072 = 1073741824` byte
- 也就是 1GB

几个例子：

```text
block 0:
  segno = 0
  segoff = 0
  seekpos = 0

block 1:
  segno = 0
  segoff = 1
  seekpos = 8192

block 131071:
  segno = 0
  segoff = 131071
  seekpos = 1073733632

block 131072:
  segno = 1
  segoff = 0
  seekpos = 0

block 131073:
  segno = 1
  segoff = 1
  seekpos = 8192
```

如果 relation path 是 `base/16384/24576`，那么 block 131072 位于：

```text
base/16384/24576.1
```

如果 fork 是 vm，block 131072 位于：

```text
base/16384/24576_vm.1
```

如果是 temp relation，block 131072 位于：

```text
base/16384/t7_24576.1
```

md.c 对跨 segment I/O 很谨慎。
`mdmaxcombine()` 返回从当前 block 开始最多还能合并多少 block 而不跨 segment。
它的返回值是：

```c
RELSEG_SIZE - (blocknum % RELSEG_SIZE)
```

`smgrreadv()` 注释要求，调用者要用 `smgrmaxcombine()` 检查多个 block 是否能合并成一次 I/O。
`mdreadv()` 和 `mdwritev()` 内部如果发现本次请求跨 segment，会报错。
原因不是 PostgreSQL 不能访问跨 segment 范围。
原因是这一层单次 readv/writev 的 fd 只能指向一个 segment 文件。
上层需要拆成多个 I/O。

`mdzeroextend()` 是例外之一。
它自己有 while loop。
它可以在扩展 relation 时跨 segment 拆分。
每轮计算当前 segment 内最多能扩展多少 block。
然后移动到下一个 segment。

## 11. `BLCKSZ` 与 `RELSEG_SIZE`

`BLCKSZ` 是 relation data block size。
它不是 WAL segment size。
它也不是 OS page size。
buffer manager 的一个 PostgreSQL buffer 正好容纳一个 disk block。
因此 `BLCKSZ` 直接影响 heap page、index page、visibility map page、FSM page 等 page 的大小。

`configure.ac` 允许的 table block size 是 1、2、4、8、16、32 KB。
默认值是 8KB。
`meson_options.txt` 也把 `blocksize` 默认设为 8。
`pg_config.h.in` 注释说明 `BLCKSZ` 必须是 2 的幂。
最大值当前是 32768。
更改 `BLCKSZ` 需要 `initdb`。

`RELSEG_SIZE` 是一个 relation fork 的单个 segment 文件最多容纳多少 block。
它的单位不是 byte。
它的单位是 block。
单个 segment 文件的最大 byte size 是：

```text
RELSEG_SIZE * BLCKSZ
```

默认 segment size 是 1GB。
在默认 `BLCKSZ = 8192` 时，`RELSEG_SIZE = 131072`。
如果 `BLCKSZ` 改成 16KB，同时仍用 1GB segment size，`RELSEG_SIZE` 会变成 65536。
所以 `RELSEG_SIZE` 和 `BLCKSZ` 是一组共同决定磁盘切片的编译期值。

`configure.ac` 中还有 `--with-segsize-blocks`。
它主要给开发者测试 segment 相关代码。
如果指定非零 blocks，它会覆盖 `--with-segsize`。
这可以用于构造很小的 segment，方便观察 `.1`、`.2` 文件。

md.c 顶部有一个静态断言：
`RELSEG_SIZE > 0 && RELSEG_SIZE <= INT_MAX`。
注释还说 `RELSEG_SIZE` 必须 fit into `BlockNumber`。
但因为它也作为整数 GUC 暴露，所以还需要 fit into signed int。

`RELSEG_SIZE` 的改变也需要 `initdb`。
原因很直观：
同一个 block number 到 segment 文件名的映射会改变。
如果一个已有 data directory 按旧 segment size 切文件，新 server 按新 segment size 寻址，就会读错文件和偏移。

不要把 relation segment 和 WAL segment 混在一起。
WAL segment 是 `pg_wal` 里的文件切片。
relation segment 是 `base/.../relfilenode.N` 这种数据文件切片。
两者名字都叫 segment，但属于不同系统。
本节只讨论 relation fork segment。

## 12. md.c 的 segment invariant

md.c 顶部注释定义了一个 relation fork 在磁盘上的 segment invariant。
一个 fork 由连续编号的 segment 文件组成。
前面可以有零个或多个 full segment。
每个 full segment 正好有 `RELSEG_SIZE` 个 block。
然后有恰好一个 partial segment。
partial segment 的大小满足 `0 <= size < RELSEG_SIZE`。
后面可以有任意数量 size 为 0 的 inactive segment。

active segment 是 full segment 加 partial segment。
inactive segment 是曾经需要过、后来被 truncate 到 0 的 segment。
md.c 不一定马上删除 inactive segment。
原因是其它 backend 或 checkpointer 可能还持有打开的 file descriptor。
如果直接 unlink，旧 fd 仍可能写到已经 unlink 的旧文件。
如果 relation 又扩展回来，新文件和旧 fd 会分裂。
把 segment 截断到 0 并保留，可以让旧 fd 仍指向同一个 inode。

这个 invariant 对理解 truncate 很重要。
`mdtruncate()` 从最后一个 open segment 往前处理。
不再 active 的 segment 被 truncate 到 0。
最后一个保留 segment 被截断到目标 block 数对应的 byte 长度。
如果目标 block 数恰好是 `K * RELSEG_SIZE`，第 `K+1` 个 segment 会被截断到 0 并保留。
这符合“一个 partial segment 可以是 0 block”的 invariant。

`mdnblocks()` 计算 fork 大小时，也依赖这个 invariant。
它从已打开的最后 segment 开始。
如果当前 segment 小于 `RELSEG_SIZE`，就返回：

```c
segno * RELSEG_SIZE + nblocks_in_segment
```

如果当前 segment 正好等于 `RELSEG_SIZE`，它尝试打开下一个 segment。
如果下一个 segment 不存在，就说明 relation size 是：

```c
segno * RELSEG_SIZE
```

如果某个 segment 大于 `RELSEG_SIZE`，md.c 报 `FATAL`。
这不是普通大小差异。
这是 segment invariant 被破坏。

`_mdfd_getseg()` 是 block number 到 open segment 的核心函数。
它先计算：

```c
targetseg = blkno / RELSEG_SIZE;
```

如果目标 segment 已经在 `md_seg_fds[forknum]` 数组中，就直接返回。
如果没打开，它从已有的最后 segment 往目标 segment 顺序打开。
它不会跳着打开。
因为它必须确认中间 segment 满足前面的 invariant。

在 recovery 中，`_mdfd_getseg()` 可以在特定行为下创建缺失 segment。
注释解释了原因：
WAL replay 可能遇到对高编号 segment 的写入。
即使这个 relation 后面会被 drop，redo 也要尽量把 WAL 顺序重放到 drop 记录。
因此 recovery 可以创建 segment，并把中间缺口补零。

正常运行中，创建新 segment 通常只应由 extension path 授权。
读一个不存在的 block 不应该悄悄造文件。
这就是 md.c 的 `EXTENSION_*` behavior flags 的意义。

## 13. 主流程 walkthrough：storage.c 的正确性创建、截断和 WAL

`storage.c` 是 catalog 层和 smgr 层之间的重要桥。
`RelationCreateStorage()` 创建物理 storage。
它只创建 main fork。
其它 fork 通常由需要它们的模块懒创建。

`RelationCreateStorage()` 先根据 `relpersistence` 决定两件事：
- 用哪个 `procNumber`
- 是否需要 smgr create WAL

三种分支是：

```text
RELPERSISTENCE_TEMP:
  procNumber = ProcNumberForTempRelations()
  needs_wal = false

RELPERSISTENCE_UNLOGGED:
  procNumber = INVALID_PROC_NUMBER
  needs_wal = false

RELPERSISTENCE_PERMANENT:
  procNumber = INVALID_PROC_NUMBER
  needs_wal = true
```

然后它调用：

```c
srel = smgropen(rlocator, procNumber);
smgrcreate(srel, MAIN_FORKNUM, false);
```

如果 `needs_wal` 为 true，它再调用 `log_smgrcreate()`。
`log_smgrcreate()` 生成 `RM_SMGR_ID` 的 `XLOG_SMGR_CREATE` record。
payload 是 `xl_smgr_create`：
- `RelFileLocator rlocator`
- `ForkNumber forkNum`

`storage_xlog.h` 中还定义了 `XLOG_SMGR_TRUNCATE`。
truncate payload 是 `xl_smgr_truncate`：
- `BlockNumber blkno`
- `RelFileLocator rlocator`
- `int flags`

flags 可以表示：
- truncate heap main fork
- truncate VM fork
- truncate FSM fork
- truncate all

注意 create 和 truncate 是 smgr 自己的 WAL record。
relation 文件删除不在这里记录。
`storage_xlog.h` 注释明确说，file creation 和 truncation 在这里 log。
deletion action 由 `xact.c` 处理。
因为删除 relation 文件是事务 commit/abort 语义的一部分。

`smgr_redo()` 在 `storage.c` 末尾。
遇到 `XLOG_SMGR_CREATE` 时，它用 `INVALID_PROC_NUMBER` 打开 relation，然后调用 `smgrcreate(..., true)`。
`isRedo = true` 允许文件已经存在。
redo 必须幂等。

遇到 `XLOG_SMGR_TRUNCATE` 时，它先打开 relation。
然后必要时强制创建 main fork。
接着 `XLogFlush(record->EndRecPtr)`。
注释说明，truncate 是不可回头的物理改变。
必须手工维护 WAL-first rule。
然后它准备 main、fsm、vm 三个 fork 的 truncation 目标。
最后调用 `smgrtruncate()`。

普通 truncate 路径在 `RelationTruncate()`。
它先把 `smgr_targblock` 和每个 fork 的 cached nblocks 置为 invalid。
然后准备 main fork。
如果 FSM 存在，就调用 `FreeSpaceMapPrepareTruncateRel()` 取得 FSM 目标大小。
如果 VM 存在，就调用 `visibilitymap_prepare_truncate()` 取得 VM 目标大小。
如果 relation needs WAL，它先写 `XLOG_SMGR_TRUNCATE`。
并且立刻 `XLogFlush(lsn)`。
然后才进入 `smgrtruncate()`。

这个顺序很重要。
truncate 会直接丢弃 buffer 并截断磁盘文件。
如果 WAL record 没有先稳定下来，crash 后 standby 或本地 recovery 可能无法知道主 fork、FSM、VM 应该一起收缩。
这会破坏 relation size 和 visibility/free-space metadata 的一致性。

## 14. permanent、unlogged、temp

三种 persistence 的常量在 `pg_class.h`：
- `RELPERSISTENCE_PERMANENT = 'p'`
- `RELPERSISTENCE_UNLOGGED = 'u'`
- `RELPERSISTENCE_TEMP = 't'`

这些是 catalog 层 relation 属性。
但它们会影响 storage manager 使用哪个 backend number、是否 WAL、是否 fsync、是否用 local buffer。

permanent relation 是普通持久 relation。
它使用 `backend = INVALID_PROC_NUMBER`。
它的文件名不带 `tN_`。
通常需要 WAL。
`RelationNeedsWAL()` 对 permanent relation 还考虑 `wal_level=minimal` 下新 relfilenode skip WAL 的特殊情况。
但它的 relation 文件是 regular shared storage。
它需要 checkpointer 或本 backend 的 sync 机制把脏数据持久化。

unlogged relation 也是 regular shared storage。
它同样使用 `backend = INVALID_PROC_NUMBER`。
它的文件名也不带 `tN_`。
所以从路径形态看，unlogged 更像 permanent，而不是 temp。
区别在 WAL。
unlogged relation 的 main fork 修改不进入普通 WAL redo。
crash 后 main fork 内容不能相信。

unlogged relation 靠 init fork 恢复。
heap access method 在创建 unlogged table 时，会创建 `INIT_FORKNUM` 并 log smgr create。
index build 对 unlogged index 也会在需要时创建 init fork。
sequence 初始化对 unlogged sequence 会同时写 main 和 init fork。
copy path 中，`RelationCopyStorage()` 对 unlogged 的 init fork 会按类似 permanent relation 的方式处理。

`RelationCopyStorage()` 里有一个关键布尔值：

```c
copying_initfork =
    relpersistence == RELPERSISTENCE_UNLOGGED &&
    forkNum == INIT_FORKNUM;
```

它决定 init fork 需要像 normal relation 一样 WAL logged 并 sync 到磁盘。
这是因为 init fork 是 crash 后重建 main fork 的来源。
如果 init fork 本身不可靠，unlogged reset 就没有基准。

`reinit.c` 说明 crash 后 reset 的过程。
cleanup 阶段先扫描 database directory，找出所有有 init fork 的 relNumber。
然后删除这些 relation 的其它 fork 文件，但保留 init fork。
init 阶段再把每个 init fork 拷贝成对应 main fork。
如果 init fork 有 segment，目标 main fork 也会带对应 segment 后缀。
最后 fsync 新 main 文件和 directory。

temp relation 是 backend-local storage。
它使用 `backend = ProcNumberForTempRelations()`。
它的文件名带 `tprocNumber_`。
它不 WAL-log。
它不 fsync。
它使用 local buffers。
其它 backend 不能正确访问它的数据。
crash 后它会被清理。

不要把 unlogged 和 temp 都简单归为“不写 WAL”。
它们的边界不同：
- temp 是 backend-local，文件名带 backend proc number，生命周期绑定 session 或 crash cleanup。
- unlogged 是 regular relation，catalog 中长期存在，文件名不带 backend proc number，crash 后用 init fork reset。
- permanent 是 regular relation，catalog 和数据都依赖 WAL、fsync、checkpoint 维持持久性。

也不要把 `RelFileLocatorBackendIsTemp()` 用来判断 unlogged。
它只看 `backend != INVALID_PROC_NUMBER`。
unlogged 的 backend 是 invalid。
所以 smgr 层从 locator backend 身份上只能区分 temp 和 non-temp。
unlogged 与 permanent 的差异来自 catalog `relpersistence` 和上层 WAL policy。

## 15. 生命周期 / ownership / cleanup

这节的对象不是 SQL relation，而是 relation 文件身份。生命周期要按三层看。

第一层是 catalog relation。`pg_class` 里的 OID、`relfilenode`、`relpersistence` 和 tablespace 由 DDL、rewrite、drop、truncate 等上层路径维护。storage 层不能把 `reltablespace = 0` 或 mapped relation 的 `relfilenode = 0` 当作物理地址，它必须拿到已经解析后的 `RelFileLocator`。

第二层是 backend-local `SMgrRelation`。`smgropen()` 用 `RelFileLocatorBackend` 作为 hash key，创建或复用当前 backend 的 smgr handle。它只建立内存身份，不保证打开任何 segment 文件。relcache 可以 pin 这个 handle；释放、失效和关闭底层 segment fd 由 smgr/md 路径处理。

第三层是磁盘文件。`mdcreate()` 创建第 0 segment，`mdextend()`/`mdzeroextend()` 按 block number 创建后续 segment，`mdtruncate()` 改变 fork 长度，`mdunlink()` 删除文件或注册延迟删除。普通 relation 的删除和截断还会通过 WAL、fsync request 和 checkpoint 收尾；temp relation 的文件名带 backend proc number，生命周期绑定 backend/session 和 crash cleanup。

unlogged relation 的 ownership 介于两者之间：catalog 长期存在，locator 是 regular relation；但 crash 后 main fork 不靠 WAL redo 恢复，而是由 `ResetUnloggedRelations()` 删除非 init fork，再把 init fork 拷贝成 main fork。

## 16. 错误路径 / 异常路径 / fallback

`RelFileLocatorBackend` 最重要的异常边界是 temp 和 non-temp。`RelFileLocatorBackendIsTemp()` 只看 `backend != INVALID_PROC_NUMBER`，不能用来判断 unlogged；unlogged 的物理 locator 仍是 regular relation。

文件创建的 fallback 主要出现在 redo。普通 `mdcreate()` 使用 `O_CREAT | O_EXCL`，文件已存在是错误；redo create 传 `isRedo = true`，文件已存在时可以打开已有文件，让 WAL replay 幂等。

truncate 是更强的持久化边界。`RelationTruncate()` 和 truncate redo 都必须先把对应 WAL flush 到 truncation LSN，再截短文件；因为截短不可逆，不能让 recovery 起点落在描述该截短的 WAL 之前。

unlogged reset 的 fallback 是丢弃 main data，而不是 redo。cleanup pass 找到有 init fork 的 relation，删除其他 fork，再由 init pass 重建 main fork。temp 文件则由临时文件清理路径丢弃残留。

## 17. 成本、资源与观测诊断

这个地址模型的 hot path 成本来自换算和缓存 miss，而不是复杂算法。每次 relation I/O 都要先确定 locator、fork、block，再换算 segment number 和 segment 内 offset。大 relation、多 fork、多 backend 会把成本传播到 `SMgrRelation` 缓存、md segment 打开数组、fd.c VFD LRU 和 checkpointer fsync request。

能直接观察的状态包括 `pg_relation_filepath()`、`pg_relation_filenode()`、`pg_class.relfilenode`、`pg_class.relpersistence`、`pg_relation_size()` 和 `$PGDATA` 下的 fork/segment 文件。能间接推断的是 `SMgrRelation` 是否缓存了某个 fork、某个 segment 是否已经被当前 backend 打开。几乎不可见的是其他 backend 的 smgr handle 和 pending invalidation 细节，需要 gdb 或源码断点。

诊断顺序应该是：先确认 SQL relation 映射到哪个 physical locator；再看是否 temp；再看 fork；再按 `blocknum / RELSEG_SIZE` 找 segment；最后才讨论 WAL、fsync、checkpoint 或 unlogged reset。不要从文件名反推完整语义后直接下结论。

## 18. 常见误区

- 把 `pg_class.oid` 当作文件号。普通场景里 OID 和 relfilenode 可能相同，但 rewrite、mapped relation 和系统 catalog 会打破这个直觉。
- 把 `reltablespace = 0` 当作物理 tablespace OID。`RelFileLocator` 必须使用解析后的真实 tablespace。
- 把 main fork 理解成 `_main` 后缀文件。main fork segment 0 没有 fork 后缀，也没有 `.0` segment 后缀。
- 把 temp 和 unlogged 都归为“不写 WAL”。temp 是 backend-local 文件；unlogged 是 regular relation 加 init fork reset。
- 把 `BlockNumber` 当作单个 OS 文件内 offset。它是 fork 内 block 序号，必须先映射到 segment。

## 19. 课堂实验

实验 1：观察普通、temp、unlogged 三种路径。

```sql
create table fork_demo(a int);
create temp table temp_fork_demo(a int);
create unlogged table unlogged_fork_demo(a int);
select pg_relation_filepath('fork_demo');
select pg_relation_filepath('temp_fork_demo');
select pg_relation_filepath('unlogged_fork_demo');
```

对照 `$PGDATA` 下的文件名：普通 relation 是 regular path；temp 文件带 `tN_`；unlogged relation 不带 `tN_`，但应能看到 `_init` fork。

实验 2：观察按需 fork。

```sql
insert into fork_demo select generate_series(1, 10000);
vacuum fork_demo;
select pg_relation_filepath('fork_demo');
```

到对应目录观察 `RELNUMBER`、`RELNUMBER_fsm`、`RELNUMBER_vm`。如果某个 fork 没出现，只能说明当前路径尚未创建它，不能说明 fork 类型不存在。

实验 3：手算 block 到 segment。

```text
BLCKSZ = 8192
RELSEG_SIZE = 131072
```

计算 block `0`、`131071`、`131072`、`262144` 的 segment number、segment 内 block offset 和 byte offset。再回到 `md.c` 的 `_mdfd_segpath()`、`mdreadv()`、`mdwritev()` 校验公式。

## 20. 讨论题

1. 为什么 `RelFileLocator` 不能直接复用 catalog 里的 `reltablespace = 0` 和 `relfilenode = 0`？
2. `RelFileLocatorBackend.backend` 能区分哪些语义？哪些语义必须回到 catalog `relpersistence`？
3. 为什么 main fork segment 0 选择最短文件名，而不是显式写成 `_main.0`？
4. redo create 为什么可以接受文件已经存在，而普通 create 不行？
5. 如果 truncate 没有先 flush WAL，会破坏哪条 crash recovery 不变量？
6. 哪些状态能用 SQL 和文件系统看到，哪些只能通过 smgr/md 断点推断？

## 21. 本节小结

本节的核心链路是：SQL relation 的 catalog 身份先被解析成 `RelFileLocator`，再由 `RelFileLocatorBackend` 加上 backend-local 维度，最后用 `ForkNumber + BlockNumber` 映射到具体 segment 文件和 byte offset。

核心状态边界是：catalog 保存逻辑对象和 persistence；`SMgrRelation` 是 backend-local 物理句柄缓存；md/fd.c 管真实文件和 segment；WAL、fsync request、checkpoint 或临时文件 cleanup 负责 crash 后的收尾。

异常路径决定了这个抽象不能简化成一个文件名。redo create 要幂等，truncate 要先满足 WAL-before-data，unlogged reset 要用 init fork 重建 main fork，temp 文件要按 backend-local 生命周期清理。

可观测入口主要是 `pg_relation_filepath()`、`pg_relation_filenode()`、`pg_class` 和文件系统；smgr cache、open segment 数组和跨 backend invalidation 多数只能用断点推断。

可迁移规律是：持久系统里的“对象身份”通常不是单字段。只有把逻辑身份、物理 locator、局部 owner、子资源类型和 offset 生命周期分开，才能同时支持 DDL rewrite、temp isolation、unlogged reset、segment 拆分和 crash recovery。
