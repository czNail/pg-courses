# PostgreSQL 临时文件、BufFile 与 spill I/O 边界

## 课程定位

上一组课程已经看过 relation fork、md segment 和 VFD cache。
这一节继续留在 `storage/file` 和执行器之间。
重点不是“临时文件放在哪个目录”这么单点的问题。
重点是：一个查询的中间结果为什么可以写磁盘，却不进入 WAL 恢复语义。

本节唯一主问题：
执行器把 sort run、hash batch 和 tuplestore 写成真实磁盘文件后，为什么这些文件仍然只是可丢弃的运行时中间状态，而不是需要 WAL、checkpoint 和 redo 保护的持久状态？

本节围绕的核心矛盾：
执行器需要把超过内存预算的中间结果可靠地读回，否则当前查询会错；但这些中间结果没有跨 crash 保留的语义，不能把 relation storage 的 WAL-before-data、page LSN 和 checkpoint contract 搬过来。`BufFile`、fd.c、FileSet 和 ResourceOwner 要解决的是运行时正确性与 cleanup，而不是持久化恢复。

读完本节，你应该能把三条线串起来：
一是 `BufFile` / fd.c 如何管理分段、tablespace、VFD、ResourceOwner 和 cleanup。
二是 `FileSet` / `SharedFileSet` 如何让并行执行共享临时文件。
三是 sort、Hash Join、tuplestore 如何 spill，以及为什么这些 I/O 不进入 WAL redo。

源码基线：本课使用当前实际源码路径 `/home/highgo/postgres`，branch `master`，commit `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；核心源码分工见第 3 节。

## 1. 本节在总主线中的位置

上一节讲的是 fd.c 如何管理大量逻辑文件。本节把这个能力接到执行器：sort、Hash Join、Materialize 等节点超过内存预算后，会把中间状态写成真实磁盘文件，但这些文件不属于 relation storage，也不进入 WAL 恢复合同。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
执行器节点超过内存预算后创建 BufFile 或 FileSet-backed 文件，经 fd.c 的 File 分段读写临时文件；ResourceOwner、FileSet、SharedFileSet 和进程退出清理负责删除这些运行时中间状态，crash 后则由启动清理丢弃残留。
```

核心矛盾是：当前查询必须可靠读回 spill 数据，但 crash 后不应恢复这些中间状态；临时文件路径解决的是运行时容量、fd、共享和 cleanup，不是 WAL/redo 持久化。

## 3. 核心文件分工与阅读顺序

本节把源码基线、重点入口和辅助核对路径集中放在这里，避免课程定位之后再漂一个未编号大节。

源码仓库：`/home/highgo/postgres`

基线：`master` = `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`

本节重点阅读：
- `src/backend/storage/file/buffile.c`
- `src/backend/storage/file/fd.c`
- `src/backend/storage/file/fileset.c`
- `src/backend/storage/file/sharedfileset.c`
- `src/include/storage/buffile.h`
- `src/include/storage/fileset.h`
- `src/include/storage/sharedfileset.h`
执行器和 sort 相关路径：
- `src/backend/executor/nodeSort.c`
- `src/backend/executor/nodeHash.c`
- `src/backend/executor/nodeHashjoin.c`
- `src/backend/executor/nodeMaterial.c`
- `src/backend/utils/sort/tuplesort.c`
- `src/backend/utils/sort/tuplesortvariants.c`
- `src/backend/utils/sort/logtape.c`
- `src/backend/utils/sort/tuplestore.c`
- `src/backend/utils/sort/sharedtuplestore.c`
辅助核对路径：
`src/backend/commands/tablespace.c`、`src/backend/postmaster/postmaster.c`、`src/include/common/file_utils.h`、`src/include/utils/tuplesort.h`、`src/include/utils/tuplestore.h`、`src/include/utils/sharedtuplestore.h`、`src/include/executor/hashjoin.h`。

## 4. 运行模型：临时但真实的 spill I/O

PostgreSQL 的执行器 spill 文件是“临时但真实的磁盘 I/O”。
它不是 shared buffer page，不是 relation fork，也不经过 storage manager 的 md read/write 语义。
它只是一段为了完成当前查询或并行查询而写出的中间状态。
这种中间状态需要承受大数据量、OS fd 限制和错误 cleanup。
排序会写 run，Hash Join 会写 batch，Materialize 会缓存 tuple。
同时，临时文件不能长期占满真实 OS fd，也不能在错误、事务结束、进程退出或崩溃后留下可见语义。
本节的核心对象是 `BufFile`。
它是 fd.c virtual `File` 之上的 BLCKSZ 缓冲接口，提供读写、seek、tell、block seek、size、append 等能力。
底层 `File` 是 VFD 索引，不是 OS fd；fd.c 可以在 fd 压力下关闭真实 fd，需要时再按路径打开。
`BufFile` 的第一个重要边界是分段。
本基线中 `buffile.c` 定义：
- `MAX_PHYSICAL_FILESIZE` 是 `0x40000000`。
- 这是 1GB。
- `BUFFILE_SEG_SIZE` 是 `MAX_PHYSICAL_FILESIZE / BLCKSZ`。
也就是说，一个 `BufFile` 的逻辑地址空间可以超过单个 OS 文件大小。
超过 1GB 后，它会再创建一个 fd.c 临时文件。
这些物理文件合起来构成一个逻辑 `BufFile`。
分段还有一个本节特别重要的目的。
`buffile.c` 注释说它不使用 relation 的 `RELSEG_SIZE`。
它固定按 1GB 分段，是为了让大型 `BufFile` 在有多个临时 tablespace 时可以分布到不同 tablespace。
因为每次 `extendBufFile()` 为普通 `BufFile` 创建下一个 segment，都会再次调用 `OpenTemporaryFile()`。
而 `OpenTemporaryFile()` 会在配置的 `temp_tablespaces` 中轮转选择。
所以一个超大 sort spill 不一定落在一个目录里。
第二个重要边界是 cleanup。
普通 `BufFileCreateTemp(false)` 创建出来的是匿名临时文件。
它的底层 `File` 由 `OpenTemporaryFile(false)` 创建。
fd.c 会给它打上 `FD_DELETE_AT_CLOSE` 和 `FD_TEMP_FILE_LIMIT`。
如果不是 `interXact`，还会注册到当前 `ResourceOwner`。
因此正常 `BufFileClose()` 会一路调用 `FileClose()`。
`FileClose()` 会关闭 OS fd，更新临时文件大小统计，然后 `unlink()`。
如果查询中途 `ERROR`，ResourceOwner release 也会调用 `FileClose()`。
如果进程退出，`before_shmem_exit` 注册的 fd.c cleanup 会兜底关闭临时文件。
第三个边界是“临时文件不靠 WAL 恢复”。
这不是因为它们写入了某种特殊 WAL record。
恰恰相反，普通 spill 路径只经过 `BufFileWrite()`、`FileWrite()`、`pg_pwritev()` 等文件 I/O。
它没有进入 `XLogBeginInsert()` / `XLogRegisterBuffer()` / `XLogInsert()`。
崩溃后也没有 redo routine 重建 sort run、hash batch 或 tuplestore。
postmaster 启动和崩溃重启周期会调用 `RemovePgTempFiles()` 清理残留。
`remove_temp_files_after_crash` 在本基线默认是 `true`。
这说明临时文件的恢复策略是“丢弃”，不是“重放”。
第四个边界是共享。
普通匿名临时文件 close 时删除。
这不适合并行查询。
并行 worker 写出来的文件必须能被 leader 或其他 worker 找到。
因此 fd.c 还提供 `PathNameCreateTemporaryFile()` / `PathNameOpenTemporaryFile()` / `PathNameDeleteTemporaryFile()`。
这类文件有名字，自动 close，但不会在 close 时自动 unlink。
`FileSet` 给这些命名临时文件提供一个临时 namespace。
`SharedFileSet` 在 DSM detach 上加引用计数，最后一个参与者 detach 时清理整个 fileset。
执行器层面三条主线也都落到这个边界。
sort 从 `nodeSort.c` 进入 `tuplesort.c`，再用 `logtape.c` 的一个 `BufFile` 模拟多个 logical tape。
私有 Hash Join 在 `nodeHash.c` / `nodeHashjoin.c` 写 inner/outer batch `BufFile`。
Materialize 通过 `tuplestore.c` 从内存数组切到 `BufFile`。
所以本节的主线是：spill 文件是执行器中间状态的临时磁盘表达，由 `BufFile` 和 fd.c 管理空间、fd、错误和 cleanup；它可以跨 segment、跨临时 tablespace、跨并行进程共享，但不进入 WAL 持久化和 redo 合约。

## 5. fd.c：临时文件先是一个 virtual File

先看 `src/backend/storage/file/fd.c` 文件头。
fd.c 的目标是管理 virtual file descriptor。
PostgreSQL 进程可能同时接触很多文件：
- relation 文件；
- sort/hash spill 文件；
- 配置文件；
- pipe；
- 扩展或工具临时打开的普通文件。
如果每个逻辑文件都长期持有 OS fd，很容易超过进程 fd 限制。
fd.c 的 `File` 不是 OS fd。
它是 `VfdCache` 数组里的一个索引。
真正 OS fd 存在 `Vfd` 结构中。
当 fd 紧张时，fd.c 可以把不活跃的 VFD 从 LRU ring 中关闭。
之后调用 `FileRead()`、`FileWrite()`、`FileSize()` 等接口时，再通过路径重新打开。
所以所有长期文件访问都应该走 fd.c 接口。
文件头还区分了几类接口。
`PathNameOpenFile()` 用于普通路径文件。
它不会自动删除。
比如 relation 文件这种长期文件，调用者必须自己负责 close。
`OpenTemporaryFile()` 用于匿名临时文件。
这种文件在 `FileClose()` 时自动删除。
显式 close、事务结束隐式 close、进程退出隐式 close 都会触发删除。
fd.c 还有一组命名临时文件接口。
名字是：
- `PathNameCreateTemporaryFile()`
- `PathNameOpenTemporaryFile()`
- `PathNameDeleteTemporaryFile()`
- `PathNameCreateTemporaryDir()`
- `PathNameDeleteTemporaryDir()`
它们服务于“临时文件需要被其他 backend 通过名字找到”的场景。
这些文件会自动 close。
它们也计入创建者的 `temp_file_limit`。
但 unlike anonymous temp file，它们 close 时不自动删除。
删除由 `FileSet` / `SharedFileSet` 这样的上层 ownership 机制处理。
普通 spill 的入口是 `OpenTemporaryFile(bool interXact)`。
它先断言临时文件访问已经初始化。
如果 `interXact == false`，它先调用 `ResourceOwnerEnlarge(CurrentResourceOwner)`。
这样可以在真正打开文件前保证 ResourceOwner 有空间记录这个 `File`。
这是 PostgreSQL 常见的错误安全模式。
先预留 owner slot，再创建外部资源。
然后 `OpenTemporaryFile()` 处理临时 tablespace。
如果当前事务已经设置了 `temp_tablespaces`，并且这个临时文件不需要跨事务存在，它调用 `GetNextTempTableSpace()`。
拿到有效 OID 后，调用 `OpenTemporaryFileInTablespace()`。
如果没有可用临时 tablespace，或者指定 tablespace 不可用，则回退到数据库默认 tablespace。
如果数据库默认 tablespace 没设好，再回退到 `DEFAULTTABLESPACE_OID`。
这里有一个经常被忽略的限制。
如果 `interXact == true`，`OpenTemporaryFile()` 会强制使用数据库默认 tablespace。
源码注释给出的理由是避免阻碍 tablespace drop。
跨事务临时文件不是普通查询 spill 的常态。
但这个分支提醒我们：临时文件是否跨事务，会影响 tablespace 选择。
`OpenTemporaryFileInTablespace()` 先调用 `TempTablespacePath()` 拼出目录。
默认 tablespace、无效 tablespace、`pg_global` 都映射到：
- `base/pgsql_tmp`
非默认 tablespace 映射到：
- `pg_tblspc/<tablespace_oid>/<TABLESPACE_VERSION_DIRECTORY>/pgsql_tmp`
临时文件名形如：
- `pgsql_tmp<MyProcPid>.<tempFileCounter>`
这是 `PG_TEMP_FILE_PREFIX` 加 backend pid 和本进程递增计数器。
创建时不使用 `O_EXCL`。
源码注释说明原因是可能存在 orphaned temp file 可以复用。
这和 crash cleanup 边界有关。
临时文件不是 WAL 恢复对象。
如果上次崩溃留下同名文件，重新创建时可以 `O_TRUNC`。
系统不需要保留旧内容。
`OpenTemporaryFile()` 创建成功后设置两个 flag：
- `FD_DELETE_AT_CLOSE`
- `FD_TEMP_FILE_LIMIT`
第一个表示 `FileClose()` 时要 unlink。
第二个表示这个文件的写入要计入 `temp_file_limit`。
如果 `interXact == false`，它再调用 `RegisterTemporaryFile()`。
`RegisterTemporaryFile()` 会：
- `ResourceOwnerRememberFile(CurrentResourceOwner, file)`
- 把 `VfdCache[file].resowner` 设为当前 owner
- 设置 `FD_CLOSE_AT_EOXACT`
- 设置 `have_xact_temporary_files = true`
所以普通临时文件有三层 cleanup。
第一层是调用者显式 `BufFileClose()`。
第二层是 ResourceOwner 在 abort 或 owner release 时通过 `ResOwnerReleaseFile()` 调用 `FileClose()`。
第三层是 `AtEOXact_Files()` 和进程退出 cleanup 扫描 VFD。
第三层更多是兜底和交叉检查。
源码注释说事务结束时，ResourceOwner 机制应该已经关闭了这些文件。
如果还有未关闭的 transaction-local 临时文件，commit 时会打 warning 并 close。
`FileClose()` 的删除顺序很重要。
它先关闭 OS fd。
如果 `FD_TEMP_FILE_LIMIT` 置位，就把该文件大小从 `temporary_files_size` 中扣掉。
如果 `FD_DELETE_AT_CLOSE` 置位，就清掉这个 flag，避免错误路径里递归 close 导致无限循环。
然后 stat 文件大小，尝试 unlink。
最后调用 `ReportTemporaryFileUsage()`。
这个函数负责 `pgstat_report_tempfile(size)` 和 `log_temp_files` 日志。
删除失败怎么处理？
普通 `FileClose()` 的 unlink 失败只 `LOG`。
临时文件删除失败不应该导致 transaction abort 再次递归触发 cleanup。
而且 spill 文件不是数据库一致性的一部分。
不能因为一个临时文件 unlink 失败，让恢复或 shutdown 变成需要 WAL 修复的问题。
写入路径还有 `temp_file_limit`。
`FileWriteV()` 在真正 `pg_pwritev()` 前检查：
- `temp_file_limit >= 0`
- 当前 VFD 有 `FD_TEMP_FILE_LIMIT`
如果本次写入会让当前进程临时文件总大小超过 `temp_file_limit * 1024`，直接 `ereport(ERROR)`。
这意味着 sort/hash/materialize spill 失败时，错误会从 fd.c 写路径向上传播。
上层不需要每个调用点自己检查临时文件 quota。
`FileWriteV()` 成功后会维护两个大小：
- 当前 VFD 的 `fileSize`
- backend 的 `temporary_files_size`
`FileTruncate()` 也会在 truncate 成功后扣回对应大小。
所以临时文件大小统计和 limit 不是单纯按写入字节累加。
它追踪文件当前逻辑大小。

## 6. temp_tablespaces：从 GUC 到路径

临时 tablespace 的入口在 `src/backend/commands/tablespace.c`。
GUC 名字是 `temp_tablespaces`。
它的 check hook 是 `check_temp_tablespaces()`。
assign hook 是 `assign_temp_tablespaces()`。
真正低层 fd.c 使用的是一组 OID。
`check_temp_tablespaces()` 做三件事。
第一，把 GUC 字符串按 identifier list 解析。
第二，如果当前在事务中且已经连接数据库，就查 catalog 验证 tablespace 名字。
第三，检查用户对 tablespace 是否有 `ACL_CREATE` 权限。
空字符串被允许，用 `InvalidOid` 表示当前数据库默认 tablespace。
显式指定当前数据库默认 tablespace 也转成 `InvalidOid`，避免不必要的权限检查。
`assign_temp_tablespaces()` 如果拿到了 check hook 生成的 extra，就调用：
- `SetTempTablespaces(myextra->tblSpcs, myextra->numSpcs)`
否则调用：
- `SetTempTablespaces(NULL, 0)`
但低层 fd.c 不一定马上有可用列表。
因为 GUC assign 可能发生在事务外，不能查 catalog。
因此 `PrepareTempTablespaces()` 是真正“准备当前事务临时 tablespace”的函数。
`PrepareTempTablespaces()` 的注释说：
如果当前事务还没准备过，就解析 `temp_tablespaces`，并告诉 fd.c 哪些 tablespace 用于 temp file。
如果不在事务中，它直接返回。
fd.c 会回退到当前数据库默认 tablespace。
这解释了为什么 `BufFileCreateTemp()`、`tuplesort.c` 的 `inittapestate()`、Hash Join 初始化等位置会主动调用 `PrepareTempTablespaces()`。
底层想避免一个隐蔽 bug：忘记准备临时 tablespace，导致所有 spill 都落到默认 tablespace。
fd.c 的 `SetTempTablespaces()` 保存数组和数量。
它还随机选择一个起始下标。
如果一个事务里创建多个临时文件，会循环推进下标。
源码注释特别提到，这能让大型临时 sort 文件比较好地分散到所有可用临时 tablespace。
这和 `BufFile` 的 1GB 分段直接配合。
每个新 segment 都可能是一次新的 `OpenTemporaryFile()`。
因此一个单独 `BufFile` 的多个 segment 也可能轮转落到不同 tablespace。
`FileSet` 的 tablespace 选择稍有不同。
`FileSetInit()` 会先调用 `PrepareTempTablespaces()`。
然后调用 `GetTempTablespaces()` 把当前事务的临时 tablespace OID 拷贝到 `FileSet` 自己的 `tablespaces[8]` 中。
如果没有配置，它保存 `MyDatabaseTableSpace`。
如果列表里有 `InvalidOid`，它替换成 `MyDatabaseTableSpace`。
这样做是为了让所有使用同一个 `FileSet` 的 backend 对 tablespace 映射达成一致。
`FileSet` 不是每创建一个 segment 就轮转。
它用 `ChooseTablespace()`。
这个函数对文件名做 `hash_bytes()`，再对 `ntablespaces` 取模。
所以同一个 fileset 里同名文件的不同访问者会选择同一 tablespace。
这对共享临时文件很关键。
leader 和 worker 不能各自按本地轮转状态猜路径。
它们必须从同一个 fileset metadata 和同一个 name 得到同一个路径。

## 7. BufFile 的结构：一个逻辑文件，多段物理文件

`src/backend/storage/file/buffile.c` 文件头给出三层定位。
第一，`BufFile` 是 fd.c virtual `File` 之上的缓冲 I/O。
第二，`BufFile` 会自动清理底层临时文件。
第三，`BufFile` 支持超过 OS 文件大小限制的逻辑临时文件。
这三点都和 spill 直接相关。
`struct BufFile` 是 opaque。
外部只能在 `src/include/storage/buffile.h` 看到 typedef 和函数声明。
内部字段包括：
- `numFiles`
- `files`
- `isInterXact`
- `dirty`
- `readOnly`
- `fileset`
- `name`
- `resowner`
- `curFile`
- `curOffset`
- `pos`
- `nbytes`
- `buffer`
`numFiles` 是物理 segment 数量。
`files` 是 `File *` 数组。
除了最后一个 segment 外，每个 segment 长度都应该是 `MAX_PHYSICAL_FILESIZE`。
`curFile` 和 `curOffset` 表示缓冲区起点所在的物理文件和偏移。
`pos` 表示当前读写位置在缓冲区内部的偏移。
`nbytes` 表示缓冲区中有效字节数。
用户看到的位置是：
- `(curFile, curOffset + pos)`
`buffer` 是一个 `PGAlignedBlock`。
大小是 `BLCKSZ`。
这意味着 `BufFile` 的缓冲粒度是一个 PostgreSQL block 大小。
写入小 tuple 时，执行器不需要每个 tuple 触发系统调用。
`BufFileWrite()` 先拷贝到内存 buffer。
buffer 满后才 `BufFileDumpBuffer()`。
读取时，`BufFileReadCommon()` 先从 buffer 拷贝。
buffer 空了才 `BufFileLoadBuffer()`。
`makeBufFileCommon()` 初始化 common 字段。
其中 `file->resowner = CurrentResourceOwner` 是关键。
后续 `extendBufFile()` 创建新 segment 时，会临时把 `CurrentResourceOwner` 切换回这个 owner。
这样一个 `BufFile` 的所有底层 `File` 都属于创建 `BufFile` 时的 ResourceOwner。
即使调用者后续切换了 memory context 或 ResourceOwner，segment ownership 也不会漂移。
普通临时文件由 `BufFileCreateTemp(bool interXact)` 创建。
它先调用 `PrepareTempTablespaces()`。
再调用 `OpenTemporaryFile(interXact)`。
然后用 `makeBufFile()` 包装第一个 `File`。
如果 `interXact == true`，它设置 `file->isInterXact = true`。
普通查询 spill 几乎总是 `false`。
当写入跨越 1GB 边界时，`BufFileDumpBuffer()` 会发现 `curOffset >= MAX_PHYSICAL_FILESIZE`。
如果还没有下一个物理文件，它调用 `extendBufFile()`。
`extendBufFile()` 对普通 `BufFile` 调用 `OpenTemporaryFile(file->isInterXact)`。
对 fileset-based `BufFile` 调用 `MakeNewFileSetSegment()`。
然后扩展 `files` 数组，把新 `File` 放进去。
这正是 BufFile 分段的实际触发点。
不是创建时预分配多个文件。
也不是按 relation segment 规则。
只有写到边界时才创建新 segment。
`BufFileLoadBuffer()` 读取时也知道 segment 边界。
如果 `curOffset >= MAX_PHYSICAL_FILESIZE` 且还有下一个 segment，它推进 `curFile` 并把 `curOffset` 归零。
然后对当前 segment 调用 `FileRead()`。
如果读取失败，报 `ERROR`。
如果读取到数据，会更新 `pgBufferUsage.temp_blks_read`。
如果启用了 `track_io_timing`，还会累计 `pgBufferUsage.temp_blk_read_time`。
`BufFileDumpBuffer()` 写入时必须处理一个 buffer 跨 segment 的情况。
它循环写。
每次计算当前 segment 剩余空间。
如果一个 BLCKSZ buffer 的剩余部分跨过 1GB 边界，先写当前 segment 尾部，再切到下一个 segment。
写失败或短写为 0/负数时，报 `ERROR`。
成功写入会推进 `curOffset`，更新 `pgBufferUsage.temp_blks_written` 和 timing。
这里有个细节。
`BufFileDumpBuffer()` 写完整个 buffer 后，`curOffset` 已经推进到 buffer 末尾。
但用户逻辑位置可能在 buffer 中间。
例如写了一些数据后在 dirty buffer 内做了小范围后退 seek。
所以函数最后会把 `curOffset` 回退 `nbytes - pos`。
必要时还会跨回前一个 segment。
这就是为什么 `BufFile` 的位置状态不能只靠 OS fd 当前 offset。
`BufFileReadCommon()` 进入时先 `BufFileFlush(file)`。
也就是说从读接口进入时，脏写 buffer 必须先落到底层 `File`。
然后循环从 buffer 读，不足时 load。
如果 `exact == true`，但读到的字节数不是请求大小，就 `ERROR`。
`BufFileReadExact()` 是 `exact=true, eofOK=false`。
`BufFileReadMaybeEOF()` 是 `exact=true`，但可以选择 EOF 时返回 0。
Hash Join 和 tuplestore 都依赖这个接口区分“文件自然结束”和“中间短读损坏/错误”。
`BufFileWrite()` 断言 `!file->readOnly`。
这对 fileset sharing 很重要。
导出后、只读打开后的 `BufFile` 不允许写。
写接口只把数据拷贝到 `buffer`。
当 `pos >= BLCKSZ` 时才 dump。
因此 `BufFileClose()`、`BufFileExportFileSet()`、从读路径进入时都必须 flush。
`BufFileSeek()` 的参数不是一个 64 位 byte offset。
它是 `(fileno, offset, whence)`。
这是因为逻辑文件可能超过 `pgoff_t` 能表示的范围。
`SEEK_SET` 使用传入 fileno 和 offset。
`SEEK_CUR` 只看 signed offset。
`SEEK_END` 用最后一个物理文件的 `FileSize()` 计算末尾。
如果 seek 落在当前 buffer 内，只调整 `pos`，不 flush。
否则先 flush，再检查目标 segment 是否存在。
如果目标不可达，返回 `EOF`，不会移动逻辑位置。
`BufFileSeekBlock()` 是 block-oriented seek。
它把 `blknum` 转成：
- `fileno = blknum / BUFFILE_SEG_SIZE`
- `offset = (blknum % BUFFILE_SEG_SIZE) * BLCKSZ`
`logtape.c` 和 `sharedtuplestore.c` 都用这个接口。
它们把底层临时文件看成 BLCKSZ block 数组。
`BufFileSize()` 只 stat 最后一个物理文件。
逻辑 size 是：
- `(numFiles - 1) * MAX_PHYSICAL_FILESIZE + lastFileSize`
这也说明除最后一个 segment 之外，BufFile 认为前面 segment 都是满的。
`BufFileAppend()` 是并行 sort 的关键。
它把 source 的 segment array 追加到 target 的 segment array。
它不复制文件内容。
它直接转移底层 `File` 所有权。
调用后 caller 不能再 `BufFileClose(source)`。
source 和 target 的 `resowner` 必须一致，否则 `ERROR`。
返回值是 source 内容在 target 逻辑空间中的起始 block number。
因为追加发生在 `MAX_PHYSICAL_FILESIZE` 对齐边界，可能留下 holes。
这些 hole 不包含有效数据，调用者不能读取。
`BufFileTruncateFileSet()` 只用于 fileset-based BufFile。
它能删除超出目标 segment 的文件，也能对保留 segment 调用 `FileTruncate()`。
这不是普通匿名 BufFile 的接口。
它存在是因为 FileSet 文件有名字，并且某些上层需要主动截断或删除共享临时数据。

## 8. FileSet：命名临时文件的 namespace

`src/backend/storage/file/fileset.c` 解决的问题是命名。
普通 `OpenTemporaryFile()` 返回一个 `File`。
别的 backend 不知道它的名字。
close 时它还会自动删除。
这不适合并行执行。
`FileSet` 提供一个临时 namespace。
文件可以用 name 发现。
底层是一组即将被删除的目录。
`FileSet` 的头文件字段很少：
- `creator_pid`
- `number`
- `ntablespaces`
- `tablespaces[8]`
`creator_pid` 是创建者 pid。
`number` 是每个 backend 内递增的 fileset 编号。
tablespace 数组是创建 fileset 时捕获的临时 tablespace 列表。
`FileSetInit()` 做四件事。
第一，保存 `MyProcPid`。
第二，分配一个 per-process counter。
第三，调用 `PrepareTempTablespaces()`。
第四，调用 `GetTempTablespaces()` 拷贝 tablespace OID。
如果没有临时 tablespace，就使用 `MyDatabaseTableSpace`。
如果有 `InvalidOid`，替换成 `MyDatabaseTableSpace`。
目录路径由 `FileSetPath()` 生成。
格式近似：
- `<tempdir>/<PG_TEMP_FILE_PREFIX><creator_pid>.<number>.fileset`
默认 tablespace 的 tempdir 是 `base/pgsql_tmp`。
非默认 tablespace 是 `pg_tblspc/.../pgsql_tmp`。
这个目录名以 `PG_TEMP_FILE_PREFIX` 开头。
因此启动清理临时文件时能识别它。
`FilePath()` 先用 `ChooseTablespace()` 选 tablespace，再把文件名拼到 fileset 目录下。
`ChooseTablespace()` 对 name 做 hash。
所以相同 name 在所有参与者中选择同一个 tablespace。
这和普通 `OpenTemporaryFile()` 的轮转策略不同。
FileSet 强调“按名字稳定定位”。
`FileSetCreate()` 用 `FilePath()` 拼路径。
然后调用 `PathNameCreateTemporaryFile(path, false)`。
如果失败，它会创建 tempdir 和 fileset directory，再用 `error_on_failure=true` 重试。
注意 `PathNameCreateTemporaryFile()` 创建的是有名字的临时文件。
它自动 close，但不自动 delete。
`FileSetDelete()` 通过 `PathNameDeleteTemporaryFile()` 删除。
`FileSetDeleteAll()` 对每个 tablespace 的 fileset 目录调用 `PathNameDeleteTemporaryDir()`。
所以 FileSet 的 ownership 是显式的。
调用者应该显式 `FileSetDelete()` 或 `FileSetDeleteAll()`。
如果忘了，启动或崩溃 cleanup 也会清理 `pgsql_tmp` 目录里符合临时前缀的内容。
但正常路径不应该依赖重启清理。

## 9. SharedFileSet：让最后一个并行参与者清理

`src/backend/storage/file/sharedfileset.c` 在 `FileSet` 上加了一层共享 ownership。
它的头文件结构是：
- `FileSet fs`
- `slock_t mutex`
- `int refcnt`
它不是共享文件系统锁。
它只是 DSM 中的一小块共享状态，用来管理引用计数。
`SharedFileSetInit(SharedFileSet *fileset, dsm_segment *seg)` 做三件事。
第一，初始化 spinlock。
第二，把 `refcnt` 设为 1。
第三，调用 `FileSetInit(&fileset->fs)`。
如果传入 DSM segment，还会注册 `on_dsm_detach()` callback。
callback 是 `SharedFileSetOnDetach()`。
worker 通过 `SharedFileSetAttach()` 进入。
它加锁检查 `refcnt`。
如果已经是 0，说明 fileset 已经 destroyed，报 `ERROR`。
否则递增 `refcnt`，再为当前 DSM segment 注册 detach callback。
这意味着每个 attach 的 backend 都会在 detach 时递减引用计数。
`SharedFileSetOnDetach()` 在错误 cleanup 路径也可能执行。
因此它不能抛出会破坏 cleanup 的错误。
它拿 spinlock，把 `refcnt` 减 1。
如果减到 0，就调用 `FileSetDeleteAll(&fileset->fs)`。
`FileSetDeleteAll()` 下层删除目录时也倾向于记录 LOG 而不是 ERROR。
这符合临时文件的删除边界。
`SharedFileSetDeleteAll()` 是主动删除入口。
它直接调用 `FileSetDeleteAll()`。
但常见并行执行路径依赖 detach callback。
尤其 leader 或 worker 出错时，DSM detach 仍然能触发目录清理。
这套机制解决的是生命周期。
它不解决并发写同一个文件的问题。
上层仍然需要约定：
- 哪个 backend 写哪个 name；
- 什么时候 writer 已经 flush/close/export；
- reader 什么时候可以 open；
- 文件是否只读。
`BufFileCreateFileSet()`、`BufFileExportFileSet()`、`BufFileOpenFileSet()` 就是上层配套。

## 10. BufFile + FileSet：共享不是随便读写

`BufFileCreateFileSet(FileSet *fileset, const char *name)` 创建一个 fileset-based BufFile。
它不会调用 `OpenTemporaryFile()`。
它调用 `MakeNewFileSetSegment(file, 0)`。
`MakeNewFileSetSegment()` 使用 name 和 segment number 生成文件名：
- `<buffile_name>.<segment>`
例如 parallel sort worker 0 的第一个 segment 可能叫：
- `0.0`
`MakeNewFileSetSegment()` 创建当前 segment 前，会先删除下一个 segment number。
也就是创建 segment N 时，先删 `N+1`。
源码注释说，这是为了处理 crash restart 前留下的同名文件。
如果旧文件残留，`BufFileOpenFileSet()` 是通过从 segment 0 开始探测连续 segment 数来判断文件长度。
旧的 `N+1` 残留会让它误以为新 BufFile 有更多 segment。
因此创建当前 segment 时先清掉下一个 segment，是一个 crash residue 防御。
`BufFileOpenFileSet()` 不知道有多少 segment。
它从 segment 0 开始循环调用 `FileSetOpen()`。
直到某个 segment 打不开为止。
如果一个 segment 都没找到：
- `missing_ok == true` 时返回 NULL；
- 否则 `ERROR`。
打开成功后，它把 `readOnly` 设为 `mode == O_RDONLY`。
共享读通常用 `O_RDONLY`。
writer 必须在 reader 打开前让文件内容稳定。
`BufFileOpenFileSet()` 的注释明确说，创建者必须先调用：
- `BufFileClose()`
或：
- `BufFileExportFileSet()`
来确保文件准备好被其他 backend 打开，并且把它变成只读。
`BufFileExportFileSet()` 只是 flush 并设置 `readOnly=true`。
它不 close 底层文件。
`BufFileClose()` 则 flush 并 close 底层文件。
为什么 export 后要 read-only？
因为共享 fileset 的 reader 通过 segment 探测和固定路径读取。
如果 writer 继续写，reader 不知道何时结束，也无法通过 WAL 或 page LSN 判断一致性。
所以并行查询层要用同步屏障保证写完再读。
这是临时文件共享自己的 contract。
不是 storage manager 或 WAL 提供的 contract。
`BufFileDeleteFileSet()` 是主动删除一个 named BufFile。
它从 segment 0 开始逐个 `FileSetDelete()`，直到遇到不存在。
如果一个都没删到且 `missing_ok == false`，报 `ERROR`。
源码注释说只有一个 backend 应该尝试删除某个 name。
如果不确定文件存在或是否已关闭，应传 `missing_ok=true`。
这说明 shared temp deletion 的并发协议交给调用者。

## 11. sort spill：nodeSort 到 tuplesort 到 logtape 到 BufFile

排序执行节点入口在 `src/backend/executor/nodeSort.c`。
`ExecSort()` 第一次被调用时，会把 outer plan 全部读完。
它根据 plan 形状选择 datum sort 或 heap tuple sort。
单列 datum 走：
- `tuplesort_begin_datum()`
普通 tuple 走：
- `tuplesort_begin_heap()`
两者都传入 `work_mem`。
然后循环 `ExecProcNode(outerNode)`，把每个 slot 喂给 tuplesort。
最后调用：
- `tuplesort_performsort()`
`nodeSort.c` 自己不懂临时文件。
它只知道 tuplesort state。
真正 spill 判断在 `src/backend/utils/sort/tuplesort.c`。
文件头把算法讲得很清楚：
- 小数据量保存在内存数组；
- 超过 `workMem` 后，写出 sorted runs 到 temporary tapes；
- run 之间用 balanced k-way merge；
- logical tape 由 `logtape.c` 实现；
- `logtape.c` 再依赖 `BufFile`。
`tuplesort_begin_common()` 创建 memory context。
它把 `workMem` 转成字节：
- `allowedMem = Max(workMem, 64) * 1024`
这里最小强制为 64KB。
注释说这是防御 parallel sort 调用者把内存切得太小。
然后初始化 batch state。
如果是 serial sort，`shared = NULL`，`worker = -1`。
如果是 parallel worker，记录 shared state 并分配 worker id。
如果是 parallel leader，记录参与者数量。
`tuplesort_puttuple_common()` 是输入 tuple 的核心状态机。
初始状态是 `TSS_INITIAL`。
它先把 tuple 放到 `memtuples` 数组。
如果 `memtupcount < memtupsize` 且 `!LACKMEM(state)`，继续留在内存。
如果超出预算，就调用：
- `inittapes(state, true)`
然后：
- `dumptuples(state, false)`
从此进入 tape-based external sort。
`inittapes()` 会计算 merge order。
然后调用：
- `inittapestate(state, state->maxTapes)`
- `LogicalTapeSetCreate(false, state->shared ? &state->shared->fileset : NULL, state->worker)`
如果是 serial sort，fileset 为 NULL。
`LogicalTapeSetCreate()` 会调用 `BufFileCreateTemp(false)`。
如果是 parallel worker，fileset 非 NULL，worker >= 0。
它会用 worker number 作为文件名，调用 `BufFileCreateFileSet(&fileset->fs, filename)`。
如果是 parallel leader，fileset 非 NULL 且 worker == -1。
它不会创建 BufFile，等待后续 import worker tape。
`inittapestate()` 还会调用 `PrepareTempTablespaces()`。
注释说 parallel sort 中可能已经调用过，但再次调用无害。
这保证 external sort 的 temp file 能使用合适临时 tablespace。
`dumptuples()` 把当前内存里的 tuple 排序成一个 run。
然后循环调用：
- `WRITETUP(state, state->destTape, stup)`
`WRITETUP` 是函数指针。
heap tuple 的具体写法在 `src/backend/utils/sort/tuplesortvariants.c`。
`writetup_heap()` 会把长度和 MinimalTuple body 写到 `LogicalTape`。
`LogicalTapeWrite()` 再写到 `logtape.c` 的 tape buffer。
最终满一个 tape block 时通过 `ltsWriteBlock()` 写到底层 `BufFile`。
`logtape.c` 的设计值得单独看。
它不是每个 logical tape 一个文件。
它用一个 `BufFile` 表示整个 tape set。
每个 logical tape 是底层文件中的一串 BLCKSZ block。
block trailer 保存前后 block pointer。
读过且不再需要的 block 可以释放到 free list。
这样 sort merge 阶段不需要同时保存输入 tape 和输出 tape 的完整两份数据。
空间使用接近真实数据量加少量 bookkeeping。
`LogicalTapeSetCreate()` 负责创建底层 `BufFile`。
serial sort：
- `lts->pfile = BufFileCreateTemp(false)`
parallel worker：
- `lts->pfile = BufFileCreateFileSet(&fileset->fs, filename)`
parallel leader：
- `lts->pfile = NULL`
因为 leader 会导入 worker 的 shared BufFile。
`ltsWriteBlock()` 接受 block number。
如果目标 block 超过当前文件末尾，它会递归写 zero block 填洞。
源码注释说 BufFile 不支持 holes。
因此 logtape 的预分配块号或 parallel append 后的 hole 需要由 logtape 层理解。
真正写入时：
- `BufFileSeekBlock(lts->pfile, blocknum)`
- `BufFileWrite(lts->pfile, buffer, BLCKSZ)`
读取时：
- `BufFileSeekBlock()`
- `BufFileReadExact()`
`LogicalTapeRewindForRead()` 在写转读时 flush 最后一个 partial block。
它设置 block trailer 中的有效字节数。
然后切到读状态。
如果 tape 不是 frozen，读取时可以释放读过的 block。
如果 tape 是 frozen，内容会保留，用于 random access 或 parallel worker 的最终输出。
`tuplesort_performsort()` 根据状态收尾。
如果还在 `TSS_INITIAL` 且是 serial sort，直接内存排序。
如果是 parallel worker，即使没有超过内存，也必须 dump 到 tape。
因为 worker 输出要交给 leader。
这就是 parallel sort 的一个关键区别。
worker 的结果必须 materialize 成共享临时文件。
它不能把本地内存数组交给 leader。
parallel worker 结束时调用 `worker_freeze_result_tape()`。
它调用 `LogicalTapeFreeze()` 生成 `TapeShare` metadata。
然后把 metadata 写入 shared memory 的 `shared->tapes[state->worker]`。
同时递增 `workersFinished`。
leader 的 `leader_takeover_tapes()` 等所有 worker 完成后调用：
- `LogicalTapeSetCreate(false, &shared->fileset, -1)`
- 对每个 worker 调用 `LogicalTapeImport()`
`LogicalTapeImport()` 用 worker number 生成文件名。
然后：
- `BufFileOpenFileSet(&lts->fileset->fs, filename, O_RDONLY, false)`
- `BufFileSize(file)`
第一个 worker 的 BufFile 被 leader 直接作为 `lts->pfile`。
后续 worker 的 BufFile 用 `BufFileAppend(lts->pfile, file)` 追加。
返回的 offset block number 存入 logical tape。
这就是 parallel sort 临时文件共享的实际机制。
注意这里不是多个进程同时写一个文件。
每个 worker 写自己的 fileset-based BufFile。
写完 freeze/export。
leader 只读打开并拼接 logical view。
共享协议靠 DSM metadata、SharedFileSet lifetime 和 worker completion counter。
WAL 不参与。
排序结束 cleanup 在 `tuplesort_free()`。
如果 `state->tapeset` 存在，调用：
- `LogicalTapeSetClose(state->tapeset)`
`LogicalTapeSetClose()` 调用：
- `BufFileClose(lts->pfile)`
这会关闭底层 `File`。
serial sort 的匿名临时文件因此被删除。
parallel leader import 的 fileset files close 后不会自动 unlink，但 SharedFileSet detach 最终会删除目录。

## 12. Hash Join spill：私有 batch 文件

Hash Join 的构建端在 `src/backend/executor/nodeHash.c`。
probe 和切换 batch 的逻辑在 `src/backend/executor/nodeHashjoin.c`。
这两个文件必须一起读。
单看 `nodeHash.c` 会看不完整 outer batch 读写。
`ExecHashTableCreate()` 先调用 `ExecChooseHashTableSize()`。
它估算：
- bucket 数量；
- batch 数量；
- hash table memory limit；
- skew bucket 数量。
内存限制来自 `get_hash_memory_limit()`，也就是 `work_mem * hash_mem_multiplier` 的语义。
如果估算 inner relation 放不下，就选择多个 batch。
`nbatch` 和 `nbuckets` 都要求是 2 的幂。
这是为了 `ExecHashGetBucketAndBatch()` 快速计算。
本基线里 `ExecChooseHashTableSize()` 还有一个很重要的优化。
它会考虑 batch 文件 buffer 自身的内存开销。
每个 batch 可能有两个文件：
- inner batch file；
- outer batch file。
每个 `BufFile` 有 BLCKSZ buffer。
如果 `nbatch` 很大，`2 * nbatch * BLCKSZ` 会变得显著。
源码用一个 U-shaped memory model 尝试减少 batch 数，让 hash table 和 file buffer 的总内存更合理。
私有 Hash Join 中，如果初始 `nbatch > 1`，`ExecHashTableCreate()` 会在 `spillCxt` 中分配：
- `hashtable->innerBatchFile = palloc0_array(BufFile *, nbatch)`
- `hashtable->outerBatchFile = palloc0_array(BufFile *, nbatch)`
并调用 `PrepareTempTablespaces()`。
但文件不会立刻打开。
每个 batch 的 `BufFile` 是 lazy create。
构建 inner 时，`MultiExecPrivateHash()` 从 inner child 读 tuple。
每个 tuple 计算 hash value。
如果 skew hash 不接收它，就调用：
- `ExecHashTableInsert(hashtable, slot, hashvalue)`
`ExecHashTableInsert()` 再调用 `ExecHashGetBucketAndBatch()`。
如果 batch 是当前 batch，就把 tuple 放入内存 hash table。
否则调用：
- `ExecHashJoinSaveTuple(tuple, hashvalue, &hashtable->innerBatchFile[batchno], hashtable)`
`ExecHashJoinSaveTuple()` 实现在 `nodeHashjoin.c`。
这也是一个跨文件调用点。
如果 `*fileptr == NULL`，它切换到 `hashtable->spillCxt`。
然后创建：
- `BufFileCreateTemp(false)`
再把 `BufFile *` 存回数组。
之后写入：
- `hashvalue`
- `MinimalTuple`
顺序是先 4 字节 hash value，再 tuple 本身。
为什么 batch file 的 buffer 要放在 `spillCxt`？
源码注释说 inner batch file 在某个 batch 被加载入内存后会关闭。
但它的 lifetime 不等于当前 `batchCxt`。
`batchCxt` 会在每个 batch 切换时 reset。
而 spill 文件数组和 buffer 需要活过 batch reset。
同时单独 `spillCxt` 有助于统计 spilling memory consumption。
如果当前内存 hash table 超过 `spaceAllowed`，`ExecHashTableInsert()` 会调用：
- `ExecHashIncreaseNumBatches()`
这个函数可能扩展 `innerBatchFile` / `outerBatchFile` 数组。
然后扫描当前内存 hash table 中已有 tuple。
不再属于当前 batch 的 tuple 会写出到新的 inner batch file。
写法同样是 `ExecHashJoinSaveTuple()`。
这说明 Hash Join 的 spill 不只发生在初始 batch 选择。
运行时发现内存压力，也可能增加 batch 并把已在内存的 tuple 重新分区写出。
outer 侧在 `nodeHashjoin.c`。
`ExecHashJoinOuterGetTuple()` 处理私有 Hash Join。
当前 batch 为 0 时，它从 outer child 取 tuple 并计算 hash。
如果 outer tuple 不属于当前 batch，就调用：
- `ExecHashJoinSaveTuple(mintuple, hashvalue, &hashtable->outerBatchFile[batchno], hashtable)`
然后继续取下一个 outer tuple。
也就是说 inner 和 outer 都可能写 batch 文件。
inner batch file 用于后续重建 hash table。
outer batch file 用于后续 probe。
当一个 batch 处理完，`ExecHashJoinNewBatch()` 切换到下一个 batch。
它先关闭上一个 outer batch file，释放磁盘空间。
如果刚处理完 batch 0，还会清理 skew hash 状态。
然后跳过可以完全忽略的空 batch。
如果 batch 两边都空，直接跳过。
外连接场景有例外，因为即使一边空，也可能需要输出 null-extended tuple。
如果 `nbatch` 运行中增加，也可能必须读取某些 inner/outer batch 以再次分配到更晚 batch。
切到新 batch 后，`ExecHashJoinNewBatch()` 调用：
- `ExecHashTableReset(hashtable)`
然后如果有 inner batch file：
- `BufFileSeek(innerFile, 0, 0, SEEK_SET)`
- 循环 `ExecHashJoinGetSavedTuple()`
- 对每个 tuple 调 `ExecHashTableInsert()`
注意这里重新 insert 时，tuple 可能再次被送到更晚 batch。
因为 `nbatch` 可能已经增加。
读完 inner batch 后，它立刻 `BufFileClose(innerFile)` 并置 NULL。
outer batch file 也会在 probe 前 rewind：
- `BufFileSeek(hashtable->outerBatchFile[curbatch], 0, 0, SEEK_SET)`
之后 `ExecHashJoinOuterGetTuple()` 会用 `ExecHashJoinGetSavedTuple()` 顺序读取。
`ExecHashJoinGetSavedTuple()` 读 tuple 的格式非常直接。
它一次读两个 `uint32`：
- hash value；
- MinimalTuple 的长度 word。
这里依赖 MinimalTuple 开头就是长度。
如果 `BufFileReadMaybeEOF()` 返回 0，表示 EOF。
否则分配 tuple 大小，设置 `tuple->t_len`，再用 `BufFileReadExact()` 读 tuple body。
短读会 `ERROR`。
读取成功后 `ExecForceStoreMinimalTuple()` 放入 slot。
私有 Hash Join cleanup 在 `ExecHashTableDestroy()`。
它跳过 batch 0。
因为 batch 0 不应该有临时文件。
如果 `innerBatchFile` 不为空，它遍历 `1..nbatch-1`：
- 关闭每个 inner batch file；
- 关闭每个 outer batch file；
然后删除 `hashCxt`。
因为 `spillCxt` 是 `hashCxt` 子 context，内存一起释放。
底层匿名临时文件在 `BufFileClose()` / `FileClose()` 时删除。
Hash Join 的错误边界也很清楚。
写 batch 文件时 `BufFileWrite()` 可能因为 `temp_file_limit`、磁盘满、权限、I/O error 抛 `ERROR`。
读 batch 文件时短读或 seek 失败也是 `ERROR`。
这会 abort 当前查询。
但不会触发 WAL redo。
因为这些 batch 文件只是该查询的执行中间状态。

## 13. Parallel Hash：SharedTuplestore 也是 SharedFileSet 的使用者

本基线的 Parallel Hash 不使用私有 `innerBatchFile` / `outerBatchFile` 数组。
`ExecHashTableCreate()` 注释直接说 parallel case uses shared tuplestores instead of raw files。
实际共享批数据走：
- `src/backend/utils/sort/sharedtuplestore.c`
这个文件名在 `utils/sort` 下，但它不是排序专用。
Parallel Hash 使用它来共享 tuple batches。
`SharedTuplestore` 是 `tuplestore.c` 的 parallel-aware subset。
文件头说多个 backend 可以写入一个 `SharedTuplestore`，然后多个 backend 可以扫描已存 tuple。
当前支持的是 parallel scan。
每个 backend 读取任意子集。
`SharedTuplestoreAccessor` 内部仍然有：
- `SharedFileSet *fileset`
- `BufFile *read_file`
- `BufFile *write_file`
也就是说 Parallel Hash 的 spill 最终仍然落到 `SharedFileSet` 下的 `BufFile`。
只是上层抽象不是 `HashJoinTableData` 里的 `BufFile **` 数组，而是 shared tuplestore。
`sts_initialize()` 需要调用者传入 `SharedFileSet` 和唯一 name。
注释说 fileset 本质上是一个会自动清理的 directory。
name 必须在同一个 SharedFileSet 内唯一。
Parallel Hash 在 `ExecParallelHashJoinSetUpBatches()` 中为每个 batch 初始化 inner/outer shared tuplestore accessor。
具体函数名包括：
- `sts_initialize()`
- `sts_attach()`
- `sts_puttuple()`
- `sts_end_write()`
- `sts_begin_parallel_scan()`
- `sts_parallel_scan_next()`
- `sts_end_parallel_scan()`
`sts_puttuple()` 第一次写时创建当前 participant 的文件。
文件名由：
- `<shared tuplestore name>.p<participant>`
生成。
创建调用是：
- `BufFileCreateFileSet(&accessor->fileset->fs, name)`
所以每个 participant 写自己的 fileset-based BufFile。
不是所有 worker 争用一个文件。
写入按 chunk 组织。
chunk 大小是 `STS_CHUNK_PAGES * BLCKSZ`。
本基线 `STS_CHUNK_PAGES` 是 4。
`sts_end_write()` 会 flush 最后一个 chunk。
然后 `BufFileClose(accessor->write_file)`。
这里 close 不会删除文件。
因为 fileset-based named temporary file 不带 `FD_DELETE_AT_CLOSE`。
它只是关闭 VFD。
文件继续存在，供其他 backend 按 name 打开读取。
`sts_begin_parallel_scan()` 断言所有 participant 都不在 writing。
然后设置扫描状态。
`sts_parallel_scan_next()` 通过 per-participant `LWLock` claim chunk。
如果当前 participant 的 file 尚未打开，就调用：
- `BufFileOpenFileSet(&accessor->fileset->fs, name, O_RDONLY, false)`
然后：
- `BufFileSeekBlock(accessor->read_file, read_page)`
- `BufFileReadExact()` 读 chunk header 和 tuple。
读完当前文件后 `BufFileClose(read_file)`。
这和 parallel sort 的共享方式类似：
- 写入者各写各的 named BufFile；
- 写完 close/export；
- 读取者按 name 只读打开；
- cleanup 由 SharedFileSet 生命周期负责。
差异是 sort leader import worker tapes 并拼接 logical tape。
Parallel Hash 则让多个 backend 对 chunk 做 parallel scan。
`nodeHashjoin.c` 的 parallel hash 初始化也显示了 SharedFileSet lifecycle。
leader 初始化 DSM 时调用：
- `SharedFileSetInit(&pstate->fileset, pcxt->seg)`
worker 初始化时调用：
- `SharedFileSetAttach(&pstate->fileset, pwcxt->seg)`
结束或清理时可能主动：
- `SharedFileSetDeleteAll(&pstate->fileset)`
DSM detach callback 仍然是兜底。

## 14. Materialize 与 tuplestore：顺序缓存的 spill

Materialize 节点入口在 `src/backend/executor/nodeMaterial.c`。
`ExecMaterial()` 第一次需要缓存时创建 tuplestore：
- `tuplestore_begin_heap(true, false, work_mem)`
然后按 eflags 设置能力。
如果需要 mark/restore，还会分配第二个 read pointer。
从 subplan 取到 tuple 后，如果 tuplestore 存在：
- `tuplestore_puttupleslot(tuplestorestate, outerslot)`
结束时：
- `tuplestore_end(node->tuplestorestate)`
Materialize 并不是唯一的 tuplestore 使用者。
同一源码里能看到：
- `nodeCtescan.c` 使用 `tuplestore_begin_heap(true, false, work_mem)`。
- `nodeWindowAgg.c` 使用 `tuplestore_begin_heap(false, false, work_mem)`。
本节聚焦 Materialize，但应把 tuplestore 视为通用顺序临时 tuple store。
`src/backend/utils/sort/tuplestore.c` 文件头说它是 dumbed-down `tuplesort.c`。
它不排序。
只存储和吐出 tuple sequence。
它可以在写完全部数据之前读取已经扫描过的部分。
这对 cursor、Materialize、WindowAgg 都重要。
tuplestore 状态只有三种：
- `TSS_INMEM`
- `TSS_WRITEFILE`
- `TSS_READFILE`
内存阶段，tuple 存在 `memtuples` 数组。
超过 `maxKBytes` 后，创建 `BufFile`。
写文件阶段，文件位置是当前写位置。
读文件阶段，文件位置是 active read pointer 的位置。
在读写状态切换时，它用 `BufFileTell()` 保存位置，用 `BufFileSeek()` 恢复位置。
`tuplestore_begin_common()` 初始化：
- `status = TSS_INMEM`
- `allowedMem = maxKBytes * 1024`
- `availMem = allowedMem`
- `myfile = NULL`
- `context = GenerationContextCreate(...)`
- `resowner = CurrentResourceOwner`
这里也保存了创建时的 ResourceOwner。
后续真正创建 BufFile 时会切回这个 owner。
`tuplestore_puttuple_common()` 在 `TSS_INMEM` 中先把 tuple 放入 `memtuples`。
如果仍然 fit，就返回。
如果 `LACKMEM(state)` 或数组压力导致不能继续，就要切到文件。
切换时：
- `PrepareTempTablespaces()`
- 保存 old owner；
- `CurrentResourceOwner = state->resowner`
- 切出 generation context，到 parent context；
- `state->myfile = BufFileCreateTemp(state->interXact)`
- 恢复 memory context 和 ResourceOwner；
- 冻结 `state->backward`；
- `state->status = TSS_WRITEFILE`
- `dumptuples(state)`
这段和 `BufFile` 的 ResourceOwner 逻辑呼应。
tuplestore 创建时在某个 owner 下。
真正 spill 可能发生在稍后的 executor 调用中。
为了让临时文件仍由 tuplestore 的 owner 管理，源码显式切回 `state->resowner`。
tuplestore 的 on-disk tuple 格式在文件头有说明。
每个 tuple 前面有一个 `unsigned int` 表示 on-tape size。
如果需要 backward scan，tuple 后面再写一份同样长度。
heap tuple 的具体写法在 `writetup_heap()`：
- 取 MinimalTuple body；
- 计算 `tuplen`；
- `BufFileWrite(state->myfile, &tuplen, sizeof(tuplen))`
- `BufFileWrite(state->myfile, tupbody, tupbodylen)`
- 如果 `state->backward`，再写 trailing length。
读取用 `getlen()` 和 `readtup_heap()`。
短读由 `BufFileReadExact()` 抛 `ERROR`。
`tuplestore_clear()` 会在有 `myfile` 时 `BufFileClose()`。
然后 reset tuple memory context，回到 `TSS_INMEM`。
`tuplestore_end()` 也会关闭 `myfile`。
所以 Materialize 正常结束时，spill 文件会被关闭并删除。
如果查询 ERROR，ResourceOwner 和 fd.c cleanup 也兜底。
统计边界也在 tuplestore 中。
`tuplestore_updatemax()` 如果还在内存，就用 `allowedMem - availMem`。
如果已经落盘，就用 `BufFileSize(state->myfile)`。
一旦 `usedDisk` 变 true，即使后续 `tuplestore_clear()` 又回到内存，它也不会变回 false。
这让 EXPLAIN 或 instrumentation 能报告曾经使用过 Disk。

## 15. WAL 不恢复边界

把临时 spill 写到磁盘，容易让人误以为它也需要 WAL。
但 PostgreSQL 的持久化边界不是“只要写磁盘就 WAL”。
WAL 保护的是数据库持久状态的改变。
sort run、hash batch、tuplestore 文件只是正在执行的查询的中间状态。
查询完成后它们应该被删除。
查询失败后它们应该被丢弃。
服务器崩溃后，产生它们的执行上下文已经不存在。
恢复这些文件也没有意义。
从调用链看，spill 路径完全不进入 WAL insert。
sort：
- `ExecSort()`
- `tuplesort_puttupleslot()`
- `tuplesort_puttuple_common()`
- `inittapes()`
- `LogicalTapeSetCreate()`
- `BufFileCreateTemp()` 或 `BufFileCreateFileSet()`
- `BufFileWrite()`
- `FileWrite()`
Hash Join：
- `ExecHashTableInsert()` 或 outer probe 保存 batch；
- `ExecHashJoinSaveTuple()`
- `BufFileCreateTemp(false)`
- `BufFileWrite()`
Materialize：
- `tuplestore_puttuple_common()`
- `BufFileCreateTemp(false)`
- `BufFileWrite()`
这些路径不调用 `XLogBeginInsert()`。
也不注册 buffer。
也不设置 page LSN。
对比 relation page 修改：
relation page 修改进入 buffer manager。
修改前或修改中会构造 WAL record。
dirty page 有 page LSN。
checkpoint 和 writeback 要遵守 WAL-before-data。
crash recovery 通过 redo 重放 page change。
spill file 完全不走这条路。
它没有 page LSN。
没有 rmgr。
没有 redo。
没有 checkpoint correctness contract。
那么崩溃后 leftover 怎么办？
fd.c 有 `RemovePgTempFiles()`。
postmaster 正常启动时，在没有其他 PostgreSQL 进程运行的阶段调用它。
它清理：
- `base/pgsql_tmp`
- 非默认 tablespace 下的 `.../pgsql_tmp`
- temporary relation files
如果发生 backend crash 后 postmaster 重启子进程周期，`postmaster.c` 在所有 server processes terminated 后，如果 `remove_temp_files_after_crash` 为 true，也调用 `RemovePgTempFiles()`。
本基线中这个 GUC 默认 true。
`RemovePgTempFiles()` 的注释很关键。
它说删除临时文件失败一般只 `LOG` 并继续。
清理临时文件没有重要到应该阻止数据库启动。
这和 WAL redo 的语义正好相反。
如果 redo 必须修改某个 relation page 却失败，那是持久化一致性问题。
如果删除某个 old `pgsql_tmp` 文件失败，最多是磁盘垃圾和后续运维问题。
FileSet-based 文件也遵守这个边界。
它们是 named temporary files。
正常 cleanup 由 `FileSetDeleteAll()` 或 `SharedFileSetOnDetach()` 完成。
崩溃残留则靠临时目录清理。
`BufFile` 的 `MakeNewFileSetSegment()` 甚至主动删除下一个 segment 来避免 crash residue 干扰 segment 探测。
这些都是“丢弃残留”的策略。
不是“恢复残留”的策略。
因此本节要避免一个错误说法：
“临时文件不重要，所以可以随便写。”
不是。
临时文件不参与 WAL 恢复，但运行时仍然必须正确。
写错 batch 会让当前查询结果错误。
读短 tuple 要报错。
共享文件要等 writer 完成再读。
ResourceOwner 要兜底删除。
临时 tablespace 权限要检查。
只是 crash 后，系统选择 abort 当前执行并删除中间状态，而不是恢复它。

## 16. ResourceOwner 和 cleanup 边界

spill 文件 cleanup 的核心不是 C 内存释放。
`BufFile` 自身用 `palloc()`。
内存 context reset 可以回收 `BufFile` struct。
但底层 OS 文件、VFD、临时路径不会因为 palloc 内存没了自动消失。
所以 fd.c 把 `File` 注册进 ResourceOwner。
普通 `OpenTemporaryFile(false)` 会调用 `RegisterTemporaryFile()`。
`RegisterTemporaryFile()` 把 file 记录到当前 ResourceOwner。
ResourceOwner release 时，fd.c 的 callback `ResOwnerReleaseFile()` 会被调用。
它把 `vfdP->resowner = NULL`，然后 `FileClose(file)`。
这样 query ERROR、subtransaction abort、executor 提前退出时，临时文件仍然可以关闭删除。
`BufFile` 自己也记住创建时的 `ResourceOwner`。
这是为了后续 segment creation。
一个 `BufFile` 创建时只有第一个 `File`。
当它长大到 1GB 后，`extendBufFile()` 要创建第二个 `File`。
如果这时 `CurrentResourceOwner` 已经变成别的 owner，直接打开会把新 segment 注册到错误 owner。
所以 `extendBufFile()` 暂时切换到 `file->resowner`。
tuplestore spill 时也用同样思路，把 `CurrentResourceOwner` 切回 `state->resowner` 后再 `BufFileCreateTemp()`。
`interXact` 是一个特殊参数。
如果 `OpenTemporaryFile(true)`，它不会注册到事务 ResourceOwner。
注释说 caller 必须在能跨事务存活的 memory context 和 resource owner 中创建，并负责关闭。
`tuplestore_begin_heap(randomAccess, interXact, maxKBytes)` 也把责任写在注释里。
普通 executor spill 不应该随意用 interXact。
否则事务结束不会自动删文件，容易造成生命周期泄漏。
fd.c 还有事务结束兜底。
`AtEOXact_Files()` 调 `CleanupTempFiles(isCommit, false)`，并清掉 transaction-local temp tablespace list。
`CleanupTempFiles()` 如果发现 `FD_CLOSE_AT_EOXACT` 仍然存在，会在 commit 时 warning，然后 `FileClose()`。
这不是正常路径。
正常路径是 ResourceOwner 先清。
但这个扫描能在 bug 或异常路径下减少泄漏。
进程退出时，`BeforeShmemExit_Files()` 调 `CleanupTempFiles(false, true)`。
这里会清理所有 temporary files，包括 interXact。
然后在 assert build 下禁止进一步创建 temp files。
`InitTemporaryFileAccess()` 在 backend startup 时注册这个 before-shmem-exit hook。
注释说这是因为临时文件清理可能要报告 pgstat，而 pgstat shutdown 也在 before_shmem_exit 阶段。
顺序必须让临时文件统计还能上报。
SharedFileSet cleanup 不靠每个 file 的 `FD_DELETE_AT_CLOSE`。
fileset-based named files不会 close 即删。
它们靠：
- `FileSetDelete()`
- `FileSetDeleteAll()`
- `SharedFileSetOnDetach()`
- postmaster 临时目录清理
这就是为什么 parallel sort leader close imported BufFile 不等于删除每个 worker 文件。
最终目录删除由 SharedFileSet lifetime 处理。

## 17. 错误与删除边界

临时文件错误处理分成两类。
第一类是当前查询不能继续的 I/O 错误。
第二类是 cleanup 阶段的删除失败。
前者通常 `ERROR`。
后者多数 `LOG`。
运行时读写错误必须 `ERROR`。
例如：
- `BufFileLoadBuffer()` 中 `FileRead()` 返回负数；
- `BufFileDumpBuffer()` 中 `FileWrite()` 返回 0 或负数；
- `BufFileReadExact()` 读不到请求字节数；
- `BufFileSeek()` 中 `FileSize()` 失败；
- `logtape.c` seek 指定 block 失败；
- Hash Join rewind batch file 失败；
- tuplestore seek 失败。
这些都表示当前执行状态已经不可靠。
继续执行可能返回错误结果。
所以必须 abort 当前 query。
临时文件空间限制也是运行时错误。
fd.c 的 `FileWriteV()` 如果发现会超过 `temp_file_limit`，直接 `ereport(ERROR)`。
上层看到的是写临时文件失败。
这不是可降级成内存算法的问题。
排序已经决定 external sort，Hash Join 已经需要 batch，Materialize 已经超过内存。
没有更多内存或磁盘预算，就只能失败。
删除失败通常不应升级为数据库一致性错误。
`FileClose()` unlink anonymous temp file 失败时 `LOG`。
`PathNameDeleteTemporaryDir()` 用于 cleanup path，失败也只是 log。
`RemovePgTempFiles()` 注释明确说删除失败一般 LOG 并继续。
这是因为临时文件不是 relation 持久状态。
删除失败可能消耗磁盘空间，但不说明 WAL 和 data files 不一致。
但主动删除 named BufFile 时可能 `ERROR`。
`BufFileDeleteFileSet(fileset, name, missing_ok=false)` 如果 segment 0 都没找到，会报错。
这是调用者协议错误。
它以为某个 fileset BufFile 存在，实际上不存在。
如果调用者不确定，应传 `missing_ok=true`。
同样，`FileSetDelete()` 的 `error_on_failure` 会决定 unlink 失败是 ERROR 还是 LOG。
共享文件还有协议错误。
`SharedFileSetAttach()` 发现 refcnt 已经是 0，会 `ERROR`。
`LogicalTapeCreate()` 如果 leader 试图在 imported shared fileset 上创建新 tape，也会 `ERROR`。
`BufFileAppend()` 如果 source 和 target resource owner 不同，会 `ERROR`。
这些不是磁盘坏了。
而是上层状态机违反了共享临时文件 contract。
还有一种“残留防御”不是错误。
`OpenTemporaryFileInTablespace()` 不用 `O_EXCL`。
`PathNameCreateTemporaryFile()` 也不用 `O_EXCL`。
同名 orphan 可以被 `O_TRUNC` 复用。
`MakeNewFileSetSegment()` 主动删除下一 segment。
这都是因为 crash residue 不具有语义价值。
遇到残留，优先让新执行建立干净临时状态。

## 18. 成本、资源与跨模块传播

spill 的成本来自三类放大。第一是数据量：行数、行宽、sort run 数、hash batch 数和 tuplestore rewind 次数会决定临时文件读写量。第二是资源预算：`work_mem`、`hash_mem_multiplier`、`temp_file_limit`、临时 tablespace 空间和 fd budget 决定何时从内存切到磁盘，以及能否继续写。第三是并行共享：parallel sort、Parallel Hash 和 SharedTuplestore 会把 FileSet/SharedFileSet 目录、DSM detach、worker/leader 协议和临时文件 cleanup 纳入同一条链路。

资源压力的传播路径是：executor 节点超过内存预算后创建 `BufFile`；`BufFile` 增长后创建更多 fd.c `File` segment；fd.c 受 VFD LRU、临时 tablespace 和 temp file limit 约束；临时 I/O 和 relation/checkpoint I/O 共享底层磁盘队列。提高 `work_mem` 可能减少 spill，但会增加 backend-local 内存峰值；扩大临时 tablespace 只能缓解空间和 I/O 分布，不能修正错误的 cardinality、行宽或 join strategy。

## 19. 观测与诊断入口

spill 的 runtime truth 是：当前查询已经把中间状态从内存搬到临时文件，并且这些文件会在执行结束、ERROR、进程退出或 crash cleanup 中被丢弃。

能直接观察的是 `EXPLAIN (ANALYZE, BUFFERS)` 里的 external sort、Hash 节点的 Batches、Materialize/tuplestore instrumentation、`log_temp_files` 日志、`temp_file_limit` ERROR 和 wait event `BUFFILE_READ`/`BUFFILE_WRITE`。能间接推断的是 `BufFile` segment 数、FileSet 目录 ownership、SharedFileSet 最后一个 detach 的 cleanup。几乎不可见的是当前 `BufFile.curFile`、`curOffset`、`resowner`、`files[]` 与 VFD LRU 状态，需要断点。

诊断顺序：先确认是 sort/hash/tuplestore 哪条路径 spill；再看 `work_mem`、`hash_mem_multiplier`、行宽、行数和并行度；再看 `temp_tablespaces`、磁盘空间、fd pressure 和 `temp_file_limit`；最后回到 `BufFileCreateTemp()`、`BufFileDumpBuffer()`、`FileWriteV()`、`BufFileClose()` 或 FileSet/SharedFileSet cleanup 验证。

## 20. 常见误区

- `work_mem` 不是临时文件大小上限；spill 文件可以远大于单个内存预算。
- `BufFile` segment 不是 relation segment；本基线固定 1GB，且不使用 `RELSEG_SIZE`。
- anonymous temp file close 时自动 unlink；FileSet 文件 close 后不自动 unlink。
- FileSet/SharedFileSet 的删除靠显式 delete、DSM detach 或启动清理，不靠每个 `FileClose()`。
- 并行查询不是多个进程随便写同一个临时文件；parallel sort worker 各写自己的 fileset BufFile，shared tuplestore 也按参与者和 chunk 协议共享。
- 临时文件不写 WAL，不表示运行时错误可以忽略；读写、seek、batch protocol 错误必须 abort 当前 query。

## 21. 课堂实验

实验 1：观察 sort spill。

```sql
SET work_mem = '1MB';
SET log_temp_files = 0;
CREATE TEMP TABLE spill_sort AS
SELECT g, md5(g::text) AS v
FROM generate_series(1, 2000000) AS g;
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM spill_sort ORDER BY v;
```

观察 external sort / external merge 和 temporary file 日志。回到 `ExecSort()`、`tuplesort_puttuple_common()`、`inittapes()`、`LogicalTapeSetCreate()`、`BufFileCreateTemp()`。

实验 2：观察 Hash Join batch spill。

```sql
SET work_mem = '1MB';
SET hash_mem_multiplier = 1;
SET log_temp_files = 0;
CREATE TEMP TABLE hj_a AS SELECT g AS id, repeat(md5(g::text), 4) AS payload FROM generate_series(1, 1000000) AS g;
CREATE TEMP TABLE hj_b AS SELECT g AS id, repeat(md5((g * 17)::text), 4) AS payload FROM generate_series(1, 1000000) AS g;
ANALYZE hj_a;
ANALYZE hj_b;
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM hj_a JOIN hj_b USING (id);
```

如果 Batches 大于 1，读 `ExecHashJoinSaveTuple()` 和 `ExecHashJoinNewBatch()`，解释 inner/outer batch 文件何时写、何时读、何时 close。

实验 3：触发 `temp_file_limit` 并回源码。

```sql
SET work_mem = '1MB';
SET temp_file_limit = '10MB';
SELECT * FROM generate_series(1, 5000000) AS g ORDER BY md5(g::text);
```

预期报 `temporary file size exceeds "temp_file_limit"`。回到 `BufFileDumpBuffer()`、`FileWrite()`、`FileWriteV()`，确认限制在 fd.c 写路径统一拦截。

## 22. 讨论题

1. 为什么 spill 文件是真实磁盘 I/O，却不进入 WAL redo contract？
2. `BufFile` 为什么不用 relation `RELSEG_SIZE`，而是固定按 1GB 分段？
3. `OpenTemporaryFile(false)` 为什么要先让 ResourceOwner 有登记空间，再创建外部文件资源？
4. FileSet 文件为什么 close 后不能自动 unlink？
5. Parallel sort worker 为什么即使结果能放进内存，也可能需要写出 shared tape？
6. `usedDisk` 一旦为 true 为什么不因为后续 clear 回到 false？
7. 删除临时残留失败为什么通常 LOG，而读写临时文件失败必须 ERROR？
8. `log_temp_files` 能解释哪些问题，哪些仍需要 EXPLAIN 或断点？

## 23. 本节小结

本节核心链路是：执行器发现内存预算不足后，通过 sort、Hash Join 或 tuplestore 状态机创建 `BufFile`；`BufFile` 把逻辑临时文件映射到多个 fd.c `File`；fd.c 负责 VFD、temp tablespace、temp file limit 和 ResourceOwner cleanup。

核心状态边界是：anonymous temp file 属于当前查询/事务，带 `FD_DELETE_AT_CLOSE`；FileSet/SharedFileSet 文件有名字，靠 namespace owner 或 DSM detach 清理；postmaster crash cleanup 负责丢弃 `pgsql_tmp` 残留。

异常路径分层：运行时读写/seek/protocol 错误必须 `ERROR`，因为当前查询状态已经不可信；cleanup 删除失败通常只 LOG，因为它不是持久 relation 状态；`temp_file_limit` 是 fd.c 写路径上的资源边界，不是执行节点各自实现的算法 fallback。

可观测入口包括 EXPLAIN 的 external sort/Hash Batches、`log_temp_files`、wait event、`temp_file_limit` ERROR、文件系统临时目录和 gdb 断点。可迁移规律是：不是所有写到磁盘的状态都值得恢复；只有跨 crash 仍有语义价值的状态才应该进入 WAL、page LSN、checkpoint 和 redo contract。
