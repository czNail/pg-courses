# PostgreSQL fd.c、VFD cache 与文件描述符压力

## 课程定位
本节主题：PostgreSQL backend 如何在 OS file descriptor 很少的现实下，仍然长期记住大量 relation segment、temporary file、配置文件、目录和短期 raw fd。
上一组课程已经围绕 buffer、WAL 和 data page 的持久化边界展开。
这一节进入更低一层的文件句柄管理。
重点不是“fd.c 有哪些 API”。
重点是：为什么 `File` 不能等同于 Unix fd，为什么 PostgreSQL 要在 backend 内维护 VFD cache，并用 LRU 在需要时关闭和重开真实 OS fd。

本节唯一主问题：
一个 backend 可能长期引用很多文件，为什么 PostgreSQL 不长期持有所有真实 OS fd，而是把“文件身份”和“当前是否占用 fd”拆开？

本节围绕的核心矛盾：
内核上层希望文件句柄稳定、可缓存、可跨调用继续使用；操作系统只给每个进程有限数量的 fd，并且还有大量 PostgreSQL 外部或旁路代码会临时消耗 fd。

读完本节，你应该能判断：
- `File` 为什么只是 backend-local VFD 下标，不是 OS fd。
- `VfdCache` 里哪些状态代表逻辑打开，哪些状态代表物理打开。
- VFD LRU 什么时候移动，什么时候关闭真实 fd，什么时候重新 `open()`。
- `PathNameOpenFile`、`OpenTransientFile`、`AllocateFile`、`BasicOpenFile` 应该分别用于什么场景。
- `max_safe_fds` 如何由 OS limit、`max_files_per_process` 和保留 fd 共同决定。
- `nfile`、`numAllocatedDescs`、`numExternalFDs` 为什么要一起参与 fd 压力判断。
- 临时文件、FileSet 文件、BufFile 和 relation segment 在 cleanup 语义上有什么区别。
- `ResourceOwner`、EOXact、subtransaction abort、process exit 各自清理什么。
- 遇到 `EMFILE`、`ENFILE`、close/fsync/write 失败、temp file limit 时，错误边界在哪里。

## 源码基线
源码仓库：`/home/nail/postgres-lab`
基线：`master` = `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`

本节重点阅读：
- `src/backend/storage/file/fd.c`
- `src/backend/storage/file/buffile.c`
- `src/backend/storage/file/fileset.c`
- `src/include/storage/fd.h`

本节为了比较 temporary file 与 relation file，还辅助核对：
- `src/backend/storage/smgr/md.c`
- `src/include/storage/buffile.h`
- `src/include/storage/fileset.h`

核心源码入口位置：
- `fd.h:15-42` 总说明，要求使用 fd.c API，不要直接长期持有裸 fd。
- `fd.c:14-31` 说明 VFD cache 的动机。
- `fd.c:115-139` 预留 fd 和最低可用 fd 常量。
- `fd.c:147-160` `max_files_per_process` 与 `max_safe_fds`。
- `fd.c:188-213` VFD 状态结构。
- `fd.c:220-240` `VfdCache`、`nfile`、临时文件大小统计。
- `fd.c:271-278` `allocatedDescs` 与 `numExternalFDs`。
- `fd.c:297-327` LRU ring 的源码注释。
- `fd.c:895-948` backend 初始化文件访问与临时文件访问。
- `fd.c:950-1083` fd limit 探测与 `set_max_safe_fds()`。
- `fd.c:1090-1156` `BasicOpenFile`。
- `fd.c:1171-1229` external fd 计数。
- `fd.c:1253-1399` LRU 删除、插入和释放。
- `fd.c:1401-1476` VFD slot 分配与回收。
- `fd.c:1478-1510` `FileAccess()`。
- `fd.c:1562-1634` `PathNameOpenFile`。
- `fd.c:1697-1880` anonymous temporary file 与 named temporary file。
- `fd.c:1965-2056` `FileClose()`。
- `fd.c:2148-2491` `FileReadV`、`FileWriteV`、`FileSync`、`FileTruncate`。
- `fd.c:2548-2876` `AllocateFile` 与 `OpenTransientFile`。
- `fd.c:3172-3299` transaction、subtransaction、process exit cleanup。
- `fd.c:3302-3544` postmaster startup 清理遗留临时文件。
- `buffile.c:14-43` BufFile 为什么建在 VFD 之上。
- `buffile.c:71-105` `BufFile` 状态。
- `buffile.c:193-217` `BufFileCreateTemp()`。
- `buffile.c:267-405` FileSet-backed BufFile。
- `buffile.c:413-425` `BufFileClose()`。
- `buffile.c:435-581` buffer load/dump 对底层 `FileRead`、`FileWrite` 的调用。
- `fileset.c:12-18` FileSet 的定位。
- `fileset.c:51-86` `FileSetInit()`。
- `fileset.c:91-145` `FileSetCreate`、`FileSetOpen`、`FileSetDelete`。
- `md.c:45-90` relation segment 的 smgr descriptor pool。
- `md.c:249-274` `mdcreate()` 使用 `PathNameOpenFile()`。
- `md.c:666-708` `mdopenfork()` 打开 relation fork。
- `md.c:722-742` `mdclose()` 调用 `FileClose()`。
- `md.c:1494-1507` `mdfd()` 暴露 raw fd 的危险边界。

---

## 1. 先给结论
`File` 不是 Unix fd。
在本节源码基线里，`src/include/storage/fd.h:51` 把它定义成 `typedef int File`。
但这个 `int` 的语义不是 kernel fd number。
它是当前 backend 私有 `VfdCache` 数组里的下标。

真正的 OS fd 放在 `VfdCache[file].fd`。
这个字段可能是一个真实 fd。
也可能是 `VFD_CLOSED`。
只要 `VfdCache[file].fileName != NULL`，这个 `File` 仍然是逻辑打开的。
它只是当前没有占用真实 kernel fd。

所以 `File` 有两个层次：
- 逻辑层：这个 backend 知道文件名、open flags、mode 和 cleanup 语义。
- 物理层：这个 backend 此刻是否真的持有一个 OS fd。

VFD cache 的本质是把这两个层次拆开。
PostgreSQL 可以长期保留很多 logical file handle。
但只让最近使用的一部分 logical file handle 占用真实 fd。

LRU 负责选择牺牲者。
每次通过 `FileReadV()`、`FileWriteV()`、`FileSync()`、`FileSize()` 等 API 触碰一个 `File` 时，都会先经过 `FileAccess()`。
`FileAccess()` 如果发现真实 fd 已经关闭，就调用 `LruInsert()` 重新打开。
如果已经打开但不是 MRU，就从 LRU ring 删除再插入到头部。

当 fd 压力上来时，`ReleaseLruFiles()` 会反复关闭 LRU tail。
关闭 tail 不等于关闭 logical file。
它只把 `VfdCache[file].fd` 设为 `VFD_CLOSED`，减少 `nfile`，并从 LRU ring 移除。
文件名和 flags 还在。
下次访问还能重新 `open()`。

这就是本节的一句话运行模型：
`File` 记住“我要访问哪个文件”，VFD LRU 决定“此刻谁真的占用 OS fd”。

这套设计不是为了抽象优雅。
它是为了避免 backend 在 relation segment、sort/hash temp file、目录遍历、配置文件读取、pipe、动态库、extension、外部库调用共同存在时把进程 fd limit 打爆。

---

## 2. 为什么不能长期持有所有 OS fd
先看一个直觉错误：
如果 relation file、temporary file、config file 都只是文件，为什么不打开后一直拿着 fd？

因为 PostgreSQL 的 backend 不是只打开一个表文件。
一个 backend 可能访问多个 database object。
每个 relation 可能有多个 fork。
一个 fork 又可能拆成多个 segment。
一个 query 可能同时产生多个 sort、hash、materialize 临时文件。
一个 backend 还可能读配置文件、遍历目录、打开 pipe、载入扩展、操作 WAL 或复制相关文件。

OS 对每个进程能打开多少 fd 有限制。
`fd.c:14-20` 的注释直接把这个问题作为 VFD cache 的动机。
系统限制在很多环境里大约是 1024，也可能更低。
更重要的是，这个限制是 per-process 的，但 PostgreSQL 是多进程系统。
如果每个 backend 都假设自己能打开大量文件，整个系统会更早触碰 OS 级资源瓶颈。

这还不是唯一问题。
并不是所有代码都一定通过 fd.c。
`fd.c:115-130` 说必须给 `system()`、dynamic loader 和其他不咨询 fd.c 的代码留出一些 fd。
这就是 `NUM_RESERVED_FDS`。
本基线里它是 10。

这些保留 fd 不是缓存容量。
它们是安全余量。
fd.c 尽量避免自己把进程 fd limit 用满。
但它无法控制 libc、动态链接器、extension 或第三方库什么时候临时打开文件。

还有一类“非 VFD 但 fd.c 知道”的占用。
比如 `AllocateFile()` 返回 `FILE *`。
比如 `OpenTransientFile()` 返回 raw fd。
比如 `AllocateDir()` 返回 `DIR *`。
这些对象不能像 VFD 那样关闭后透明重开。
它们要么有 stdio buffer 状态，要么是目录迭代状态，要么调用者正持有裸 fd。
所以 fd.c 用 `allocatedDescs[]` 单独跟踪它们。

再有一类是“外部 fd”。
调用者可能真的不得不用旁路 fd。
这时要用 `AcquireExternalFD()` 或 `ReserveExternalFD()` 告诉 fd.c。
否则 fd.c 以为 fd 还很多，实际进程 fd table 已经被旁路代码占掉。

因此长期持有所有 OS fd 会破坏三个边界：
- 资源边界：进程 fd limit 被耗尽，出现 `EMFILE`。
- 协作边界：fd.c 不能给不受它控制的代码留余量。
- cleanup 边界：裸 fd 无法被 `ResourceOwner` 和 EOXact 机制统一回收。

VFD cache 的价值在这里：
它不是为了减少 `open()` 次数。
它是在 `open()` 成本和 fd limit 之间取一个工程上可控的折中。

---

## 3. 核心状态：File、VfdCache、VFD
本节先只理解一个 backend 内的状态。
`VfdCache` 是 backend-local static 变量。
它不是 shared memory。
其他 backend 看不到你的 `File` 值。
同一个整数 `5` 在两个 backend 中可以指向完全不同的 `VfdCache[5]`。

`fd.c:200-213` 定义 `struct vfd`。
课程里不需要背完整结构体。
需要抓住这些字段组合：
- `fd`：当前真实 OS fd，或者 `VFD_CLOSED`。
- `fdstate`：临时文件、事务结束关闭、关闭时删除等语义位。
- `resowner`：这个 `File` 是否由某个 `ResourceOwner` 自动清理。
- `nextFree`：VFD slot 空闲链表。
- `lruMoreRecently` 与 `lruLessRecently`：LRU ring 双向链。
- `fileSize`：临时文件大小统计使用。
- `fileName`：逻辑文件身份。
- `fileFlags` 与 `fileMode`：重新打开时需要的参数。

`fd` 单独不能说明文件是否“打开”。
要结合 `fileName` 看。
源码用两个宏表达这个边界：
- `FileIsValid(file)` 要求下标有效且 `fileName != NULL`。
- `FileIsNotOpen(file)` 只看 `fd == VFD_CLOSED`。

所以有三种状态：
- unused VFD slot：`fileName == NULL`，在 free list 里。
- logical open but physical closed：`fileName != NULL` 且 `fd == VFD_CLOSED`，不在 LRU ring。
- logical open and physical open：`fileName != NULL` 且 `fd >= 0`，在 LRU ring。

第 2 种状态是 VFD cache 的核心。
没有它，`File` 就退化成 OS fd。

`VfdCache[0]` 是特殊 header。
它不代表文件。
`fd.c:216-219` 明确说 `VfdCache[0]` 不是可用 VFD。
LRU ring 以 element zero 为 anchor。
free list 也从 `VfdCache[0].nextFree` 开始。

`nfile` 记录当前由 VFD entries 持有的真实 fd 数量。
注意它不是 logical file 数量。
如果一个 backend 逻辑打开了 1000 个 `File`，但 LRU 只让其中 100 个物理打开，那么 `nfile` 约等于 100。

`allocatedDescs[]` 是另一条线。
它记录 `AllocateFile()`、`AllocateDir()`、`OpenPipeStream()`、`OpenTransientFile()` 得到的非 VFD handle。
这些 handle 不可被 LRU 透明关闭。
所以它们参与 fd pressure 计算，但不在 VFD LRU ring 中。

`numExternalFDs` 是第三条线。
它不是 fd.c 打开的东西。
它只是 fd.c 被告知“外面有人长期占用了一个 fd”。
如果这条线不同步，fd.c 的安全判断就会偏乐观。

本节要记住一个资源不变量：
fd.c 试图维持
`nfile + numAllocatedDescs + numExternalFDs < max_safe_fds`。

这不是强数学保证。
因为 OS 全局 fd table、旁路代码、race 和内核错误仍可能让 `open()` 失败。
但它是 backend 内 fd.c 能维护的主要边界。

---

## 4. VFD LRU 的运行模型
LRU ring 的注释在 `fd.c:297-327`。
这个注释值得认真读。
它说明只有当前真实打开的 VFD 在 ring 里。
逻辑打开但物理关闭的 VFD 不在 ring 里。

ring 的方向容易混淆。
源码里 `VfdCache[0].lruLessRecently` 指向 most recently used。
`VfdCache[0].lruMoreRecently` 指向 least recently used。
`ReleaseLruFile()` 释放时取的是 `VfdCache[0].lruMoreRecently`。

插入逻辑在 `Insert()`。
它把 file 插入到 MRU 位置。
删除逻辑在 `Delete()`。
它只维护链表，不关闭 fd。
关闭并从 LRU 删除是 `LruDelete()`。

正常访问路径是：

```text
FileReadV/FileWriteV/FileSync/FileSize/...
  -> FileAccess(file)
     -> 如果物理关闭：LruInsert(file)
        -> ReleaseLruFiles()
        -> BasicOpenFilePerm(fileName, fileFlags, fileMode)
        -> nfile++
        -> Insert(file)
     -> 如果物理打开但不是 MRU：
        -> Delete(file)
        -> Insert(file)
```

`FileAccess()` 是所有 VFD 操作的闸门。
如果上层绕过 fd.c 直接拿 raw fd 并长期使用，就绕过了这个闸门。

`LruInsert()` 里有两个关键动作。
第一，它先调用 `ReleaseLruFiles()`。
这会在当前 known fd 消耗达到 `max_safe_fds` 时，释放 LRU tail。
第二，它调用 `BasicOpenFilePerm()` 重新打开文件。
这个重新打开使用的是保存在 VFD 里的 `fileName`、`fileFlags`、`fileMode`。

这解释了 `PathNameOpenFilePerm()` 为什么保存 flags 时要去掉 `O_CREAT | O_TRUNC | O_EXCL`。
源码在 `fd.c:1623-1626`。
第一次打开可能要创建或截断。
后续 LRU 重新打开绝不能再次 truncate。
否则一次 cache miss 就可能破坏文件内容。

`ReleaseLruFiles()` 的循环条件是：

```c
nfile + numAllocatedDescs + numExternalFDs >= max_safe_fds
```

它会一直调用 `ReleaseLruFile()`。
如果没有 VFD fd 可释放，就退出。
这意味着 fd.c 不会强行关闭 `AllocateFile()` 或 `OpenTransientFile()` 返回给调用者的 handle。
它只能释放 VFD ring 里的真实 fd。

`ReleaseLruFile()` 如果 `nfile > 0`，就关闭 least recently used VFD。
如果 `nfile == 0`，说明 fd.c 已经没有可释放的 VFD fd。
这时如果还需要打开新 fd，就只能看 OS 是否允许，或者返回失败。

`LruDelete()` 关闭真实 fd 时会调用 `pgaio_closing_fd()`。
这是为了让异步 I/O 层知道 fd 正在关闭。
然后调用 `close()`。
如果 close 失败，源码选择记录日志，而不是试图把内部状态回滚。
注释说，如果 close 失败，宁可泄漏 fd，也不要弄坏内部状态。

这类错误边界很重要。
LRU 的 correctness 不依赖“close 永远成功”。
它依赖内部 VFD 状态不能在异常路径上半更新。

---

## 5. 打开入口的语义边界
`src/include/storage/fd.h:15-42` 是 API 使用规则的压缩版。
它明确说这些不是 Unix routine 的简单改名。
所有文件活动都应该使用它们。
如果必须旁路，至少要用 external fd API 报告长期持有的 fd。

本节必须区分四类入口。

第一类是 `PathNameOpenFile()`。
它返回 `File`。
它用于长期持有的 logical file。
relation segment 是典型调用者。
`md.c:249` 的 `mdcreate()` 用它创建 relation fork 的第一个 segment。
`md.c:688` 的 `mdopenfork()` 用它打开 relation fork。
`md.c:1722` 的 `_mdfd_openseg()` 用它打开后续 segment。

`PathNameOpenFilePerm()` 的流程是：
- 复制文件名到 malloc 内存。
- 分配 VFD slot。
- 释放超额 VFD fd。
- 给 flags 加 `O_CLOEXEC`。
- 用 `BasicOpenFilePerm()` 打开真实 fd。
- 保存 fileName、flags、mode、fileSize、fdstate、resowner。
- 插入 LRU ring。
- 返回 `File` 下标。

这个 API 的重点是“可以长期记住”。
调用者后续必须用 `FileRead`、`FileWrite`、`FileSync`、`FileClose` 等 API。
不能把 `File` 当 fd 传给 `read()`。

第二类是 `OpenTransientFile()`。
它返回 raw OS fd。
但它不是裸 `open()`。
它会走 `reserveAllocatedDesc()`。
它会参与 `numAllocatedDescs` 计数。
它会在 transaction abort 或 cleanup 中被 fd.c 关闭。

它适合短期、非 VFD 的 unbuffered fd。
例如 `fd.c` 自己在 `fsync_fname_ext()` 中用它打开一个路径，然后 `pg_fsync()`，再 `CloseTransientFile()`。
`copydir.c` 复制文件时也使用它。

`OpenTransientFile()` 不适合长期持有。
原因是它的 fd 不能被 VFD LRU 透明关闭和重开。
它占用的是 `allocatedDescs[]` 容量。
如果你长期持有大量 transient fd，就会把 VFD 可用空间挤掉。

第三类是 `AllocateFile()`。
它返回 `FILE *`。
它是 `fopen()` 的 fd.c 包装。
适合短时间用 stdio 读配置文件、解析小文件。

`fd.h:30-33` 特别警告：
如果文件要长期打开，避免 stdio。
因为 `FILE *` 里的状态没法和其他文件共享 kernel fd。
fd.c 只能记录和自动关闭它，不能 LRU reopen 它。

`AllocateFile()` 失败路径和 `OpenTransientFile()` 类似。
它先检查 `reserveAllocatedDesc()`。
然后 `ReleaseLruFiles()`。
如果 `fopen()` 遇到 `EMFILE` 或 `ENFILE`，会尝试 `ReleaseLruFile()` 再重试。

第四类是 `BasicOpenFile()`。
它最接近 `open(2)`。
它返回 raw fd。
它不登记到 `allocatedDescs[]`。
它没有自动 cleanup。
调用者必须自己 `close(2)`。

为什么还需要它？
因为 fd.c 内部需要一个最低层 open wrapper。
`PathNameOpenFilePerm()` 和 `OpenTransientFilePerm()` 都最终调用它。
它的特殊能力是遇到 `EMFILE` 或 `ENFILE` 时，可以释放一个 LRU VFD fd，然后重试。

但 `BasicOpenFile()` 对外部调用者很危险。
源码注释在 `fd.c:1096-1104` 说得很清楚：
拿到 fd 后，caller 必须确保 `ereport(ERROR)` 时不会泄漏。
大多数用户应该使用 VFD abstraction。

这四类入口可以这样记：

```text
PathNameOpenFile   -> 长期 logical file，VFD 管理，LRU 可关闭重开
OpenTransientFile  -> 短期 raw fd，fd.c 记录，abort 可清理
AllocateFile       -> 短期 FILE*，fd.c 记录，abort 可清理
BasicOpenFile      -> 最低层 raw fd，没有自动 cleanup，慎用
```

还有一个相关接口是 `FileGetRawDesc()`。
它会先 `FileAccess()`，确保 VFD 当前物理打开，然后返回 raw fd。
但源码注释在 `fd.c:2507-2514` 明确警告：
这个 fd 只在文件被关闭前有效，而很多事情都可能导致关闭。
所以 caller 不能拿着它做复杂操作。
`md.c:1494-1507` 的 `mdfd()` 就是一个很窄的边界。

---

## 6. fd limit 管理
fd.c 管理 fd limit 的第一步是 `set_max_safe_fds()`。
它在普通 postmaster 启动后期运行，然后 fork 出来的 backend 继承结果。
`fd.c:150-160` 说明了这个变量的生命周期。

`set_max_safe_fds()` 先调用 `count_usable_fds()`。
这个函数通过不断 `dup(2)` 来探测当前进程还能得到多少 fd。
它也估算已经打开的 fd 数。
如果系统有 `getrlimit(RLIMIT_NOFILE)`，它不会越过当前 soft limit。

然后计算：

```text
max_safe_fds = min(usable_fds, max_files_per_process) - NUM_RESERVED_FDS
```

本基线里：
- `max_files_per_process` 默认 1000。
- `NUM_RESERVED_FDS` 是 10。
- `FD_MINFREE` 是 48。

如果扣除保留 fd 后少于 `FD_MINFREE`，backend 启动会 `FATAL`。
错误会报告系统允许多少、server 至少需要多少、已经打开多少。

注意 `max_files_per_process` 不是“每个 backend 永远会打开这么多 fd”。
它是 fd.c 给自己认识的 fd 消耗设置的上限。
真实 OS limit 可能更低。
旁路代码可能再消耗一些。
系统级 file table 也可能因为其他进程耗尽而出现 `ENFILE`。

fd.c 的运行时计数分成三块：
- `nfile`：VFD ring 当前真实打开的 fd。
- `numAllocatedDescs`：`FILE *`、`DIR *`、pipe、transient raw fd。
- `numExternalFDs`：调用者报告的外部 fd。

`ReleaseLruFiles()` 只会通过关闭 VFD fd 来让总数低于 `max_safe_fds`。
它不会关闭 allocated descriptors。
因为它们没有透明 reopen 语义。

`reserveAllocatedDesc()` 还对非 VFD handle 单独设限。
初始容量是 `FD_MINFREE / 3`。
后续最多扩大到 `max_safe_fds / 3`。
注释解释了原因：
allocated descriptors 不能把所有 fd 空间都占住。
VFD 需要剩余空间。
external fd 也允许再吃掉另一部分。

`AcquireExternalFD()` 也使用三分之一原则。
如果 `numExternalFDs < max_safe_fds / 3`，它调用 `ReserveExternalFD()` 并返回 true。
否则设置 `errno = EMFILE` 并返回 false。

`ReserveExternalFD()` 是更强硬的接口。
它不返回失败。
它先 `ReleaseLruFiles()`，然后增加 `numExternalFDs`。
注释建议只在失败等同于会话失败、并且数量可预测很小时直接使用。

fd limit 管理的思想是：
fd.c 不是只管自己的 VFD。
它要为几种 fd 消耗类型划出相互不完全挤占的空间。

---

## 7. temporary file 的特殊语义
临时文件在 fd.c 里不是一类简单的 pathname。
它叠加了三个语义：
- 是否关闭时删除。
- 是否事务结束时自动关闭。
- 是否计入 `temp_file_limit`。

这些语义由 `fdstate` 位表示：
- `FD_DELETE_AT_CLOSE`：`FileClose()` 时 unlink。
- `FD_CLOSE_AT_EOXACT`：事务结束清理。
- `FD_TEMP_FILE_LIMIT`：写入时检查并统计临时文件大小。

`OpenTemporaryFile(bool interXact)` 创建匿名临时文件。
它会选择 temp tablespace。
它会调用 `OpenTemporaryFileInTablespace()` 生成路径。
路径形如 `pgsql_tmp<PID>.<counter>`。
底层仍然调用 `PathNameOpenFile()`。

打开成功后，`OpenTemporaryFile()` 设置：
`FD_DELETE_AT_CLOSE | FD_TEMP_FILE_LIMIT`。
如果 `interXact == false`，它还调用 `RegisterTemporaryFile()`。
这会把 `File` 记到 `CurrentResourceOwner`，设置 `resowner`，并加 `FD_CLOSE_AT_EOXACT`。

所以普通 `OpenTemporaryFile(false)` 的生命周期是：
- 当前 transaction resource owner 持有它。
- 正常 close 会删除底层文件。
- ERROR/abort 时 ResourceOwner 会释放它。
- EOXact 还有 backup cleanup。
- backend exit 会再次兜底清理。

`interXact == true` 则不同。
它允许文件跨事务存在一段时间。
它不登记当前 transaction ResourceOwner。
但只要显式 `FileClose()`，仍会删除。
process exit cleanup 也会处理。

named temporary file 是另一条路径。
`PathNameCreateTemporaryFile()` 创建一个有名字、可被其他参与者打开的临时文件。
它会设置 `FD_TEMP_FILE_LIMIT` 并登记 ResourceOwner。
但它不会设置 `FD_DELETE_AT_CLOSE`。
原因是这种文件可能需要被其他 backend 按名字发现。
删除要由更高层的 FileSet 或 SharedFileSet ownership 处理。

`PathNameOpenTemporaryFile()` 打开已存在的 named temporary file。
它自动 close at end of transaction。
但不计入 caller 的 `temp_file_limit`。
源码注释在 `fd.c:1883-1886` 说得很清楚：
它打开的文件可能是其他 backend 创建的。
caller 不能把它算进自己的临时文件配额。

`PathNameDeleteTemporaryFile()` 按路径删除 named temporary file。
它容忍 `ENOENT`。
这是为了支持 `BufFileDeleteFileSet()`。
因为 BufFile 不知道有多少 segment，只能一直删除直到缺失。

临时文件还有 postmaster startup 清理。
`RemovePgTempFiles()` 会处理 `base/pgsql_tmp` 和 non-default tablespace 下的 temp directories。
它还会调用 `RemovePgTempRelationFiles()` 清理临时 relation 文件。
清理失败通常只 LOG。
原因是删除旧 temp 文件不应该阻止数据库启动。

---

## 8. BufFile：在 VFD 之上减少 thrashing
`buffile.c` 是理解 VFD 压力的好例子。
文件开头注释说，BufFile 是在 fd.c 管理的 virtual Files 之上实现的简化 stdio。
它只提供 buffered I/O，不提供 stdio 格式化。

为什么 BufFile 对 VFD 更有价值？
因为每次触碰 `File` 都可能经过 `FileAccess()`。
如果 fd 压力很大，频繁触碰很多 VFD 会导致反复 close/reopen。
BufFile 把小读写合并到 `BLCKSZ` 大小 buffer 中。
减少底层 `FileRead()` 和 `FileWrite()` 次数。
也就减少 VFD LRU ring 的移动和 reopen 频率。

`BufFile` 状态里有：
- `numFiles`：底层物理文件 segment 数。
- `files`：`File` 数组，每个都是 fd.c 的 logical file。
- `isInterXact`：是否跨事务。
- `dirty`：buffer 是否需要写出。
- `readOnly`：FileSet 共享后的只读状态。
- `fileset` 与 `name`：是否属于 FileSet。
- `resowner`：创建时的 ResourceOwner。
- `curFile`、`curOffset`、`pos`、`nbytes`：logical position 与 buffer 状态。

BufFile 支持超过单个 OS 文件大小限制的临时文件。
本基线里 `MAX_PHYSICAL_FILESIZE` 是 1GB。
底层会拆成多个 fd.c `File`。
这和 relation 的 `RELSEG_SIZE` 分段类似，但它服务的是临时工作文件。

`BufFileCreateTemp(interXact)`：
- 调用 `PrepareTempTablespaces()`。
- 调用 `OpenTemporaryFile(interXact)`。
- 用第一个 `File` 创建 `BufFile`。

`extendBufFile()`：
- 临时切换 `CurrentResourceOwner` 到 BufFile 创建时的 owner。
- 如果不是 FileSet-backed，调用 `OpenTemporaryFile(file->isInterXact)`。
- 如果是 FileSet-backed，调用 `MakeNewFileSetSegment()`。
- 把新 `File` 加到 `files[]`。

这说明 BufFile 的 memory object 和底层 file resource 不是同一种 ownership。
`BufFile` struct 由 palloc 管理。
底层 `File` 由 fd.c 和 ResourceOwner 管理。
ERROR 时即使 `BufFile` struct 只靠 memory context 消失，底层 `OpenTemporaryFile()` 创建的 `File` 仍然能被 ResourceOwner 清理。

`BufFileLoadBuffer()` 调用 `FileRead()`。
读失败时用 `FilePathName(thisfile)` 报错。
`BufFileDumpBuffer()` 调用 `FileWrite()`。
写失败同样报告底层路径。
所以 BufFile 不绕开 fd.c。
它只是把访问粒度变粗。

`BufFileClose()` 会先 flush，再对每个底层 `File` 调用 `FileClose()`。
如果这些底层文件是 anonymous temp file，`FileClose()` 会删除它们。
如果是 FileSet-backed file，是否删除不由 `FileClose()` 自动完成。

---

## 9. FileSet：命名临时文件空间
`fileset.c` 的定位很窄：
提供一个临时 namespace，让文件能按名字被发现。

它适用于两类场景：
- 一个 backend 内，临时文件需要跨事务、反复打开关闭。
- parallel execution 中，多个 backend 需要在同一个 SharedFileSet 下共享临时数据。

`FileSet` 本身记录：
- creator pid。
- per-pid number。
- temp tablespace 列表。

`FileSetInit()` 会 capture 当前 temp tablespace 配置。
如果用户没有配置 temp tablespaces，就使用当前数据库默认 tablespace。
如果列表里有 `InvalidOid`，会替换成当前数据库默认 tablespace。
这么做是为了让使用同一个 FileSet 的参与者对路径选择达成一致。

`FileSetCreate()`：
- 根据 file name hash 选择 tablespace。
- 计算完整路径。
- 调用 `PathNameCreateTemporaryFile(path, false)`。
- 如果目录还不存在，就创建 temp dir 和 fileset dir，再重试。

`FileSetOpen()`：
- 计算路径。
- 调用 `PathNameOpenTemporaryFile(path, mode)`。

`FileSetDelete()`：
- 计算路径。
- 调用 `PathNameDeleteTemporaryFile()`。

`FileSetDeleteAll()`：
- 遍历 FileSet 涉及的 temp tablespaces。
- 删除每个 fileset directory。

关键点是：
FileSet 没有发明新的 fd 管理机制。
它只发明了临时文件命名空间和目录 ownership。
最终创建、打开、关闭、统计、fd pressure 管理都回到 fd.c。

`BufFileCreateFileSet()` 把 BufFile 和 FileSet 接起来。
它用 `MakeNewFileSetSegment()` 创建 segment。
`BufFileOpenFileSet()` 不知道 segment 数量。
它从 0 开始不断 `FileSetOpen()`，直到缺失。
这就是 `PathNameDeleteTemporaryFile()` 必须容忍缺失的原因之一。

FileSet-backed BufFile 的共享边界是只读。
创建者需要调用 `BufFileClose()` 或 `BufFileExportFileSet()`。
`BufFileExportFileSet()` flush buffer，然后标记 `readOnly = true`。
其他 backend 才能安全打开。

---

## 10. relation file 与 temporary file 的差异
relation segment 和 temporary file 都可能通过 `PathNameOpenFile()` 得到 `File`。
但它们的 lifecycle 不一样。
不要因为底层 API 相同就把语义混在一起。

relation file 的上层 owner 是 smgr 和 relcache。
`md.c` 维护 `SMgrRelation` 里的 per-fork `md_seg_fds` 数组。
数组元素是 `MdfdVec`。
其中 `mdfd_vfd` 才是 fd.c 的 `File`。

relation segment 的路径来自 relfilenode、tablespace、database、fork 和 segment number。
它的持久化、fsync、unlink、checkpoint 协议由 storage manager 和 sync request 管理。
fd.c 不知道这个 `File` 代表哪个 relation。
fd.c 只知道路径和 fdstate。

`mdcreate()` 创建 relation fork 后，如果不是 temp relation，会调用 `register_dirty_segment()`。
这会把 fsync request 交给 sync subsystem 或直接本地 fsync。
`FileClose()` 不会删除普通 relation file。
它只关闭 VFD、释放 slot。

temporary file 的 owner 更靠近执行期 resource。
anonymous temp file 由 `FD_DELETE_AT_CLOSE` 决定关闭时删除。
普通 query 的 temp file 还会被 ResourceOwner 和 EOXact 清理。
FileSet temp file 则由 FileSet/SharedFileSet 目录 ownership 删除。

relation 的删除还有更复杂的 crash safety。
`mdunlink()` 对普通 relation main fork 的第一个 segment 不总是立即 unlink。
它可能先 truncate，注册 checkpoint 后 unlink request。
这是为了避免 relfilenumber reuse 和 crash recovery 之间的危险窗口。
临时 relation 不需要同一套 WAL 持久化保护。
`md.c:307-312` 明确说明 temp rel 不做这套 dance。

可以这样对比：

```text
relation file:
  logical owner = SMgrRelation / relcache / storage manager
  fd layer      = File/VFD
  deletion      = smgr/md + checkpoint/sync request
  fsync         = non-temp relation 需要注册或执行
  cleanup       = smgrclose/mdclose/FileClose，不由 fd.c 自动 unlink

anonymous temp file:
  logical owner = executor/query/resource owner
  fd layer      = File/VFD
  deletion      = FD_DELETE_AT_CLOSE
  fsync         = 通常不是持久化对象
  cleanup       = ResourceOwner、EOXact、proc exit

FileSet temp file:
  logical owner = FileSet/SharedFileSet
  fd layer      = File/VFD
  deletion      = FileSetDelete/FileSetDeleteAll
  fsync         = 通常不是 durable relation path
  cleanup       = FileSet ownership 或 DSM detach
```

共同点是：
只要它们使用 `File` API，就共享 VFD LRU 和 fd pressure 管理。

不同点是：
fd.c 只管理 file descriptor 与部分临时文件 cleanup。
它不替 relation layer 决定 crash safety。

---

## 11. resource cleanup 主链路
fd.c 的 cleanup 要分四层看。

第一层是显式 close。
`FileClose(file)` 是 VFD 的正常结束。
它会：
- 如果物理 fd 打开，调用 `pgaio_closing_fd()` 和 `close()`。
- 减少 `nfile`。
- 从 LRU ring 删除。
- 如果计入 `temp_file_limit`，扣减 `temporary_files_size`。
- 如果 `FD_DELETE_AT_CLOSE`，统计大小并 unlink。
- 如果有 `resowner`，从 ResourceOwner forget。
- 释放 `fileName`。
- 把 VFD slot 还给 free list。

第二层是 ResourceOwner release。
`RegisterTemporaryFile()` 会把 `File` 注册到 `CurrentResourceOwner`。
ResourceOwner callback 是 `ResOwnerReleaseFile()`。
它把 `resowner` 清空，然后调用 `FileClose(file)`。
这样 ERROR/abort 时临时文件不会泄漏。

第三层是 transaction/subtransaction cleanup。
`AtEOSubXact_Files()` 只处理 `allocatedDescs[]`。
subtransaction commit 时把 `create_subid` 迁移到 parent。
subtransaction abort 时关闭该 subtransaction 创建的 allocated desc。
注释说明 temporary files 由 ResourceOwner 处理。

`AtEOXact_Files()` 调用 `CleanupTempFiles(isCommit, false)`。
它还清空 temp tablespace list。
如果有事务本地临时文件没被 ResourceOwner 正常释放，`CleanupTempFiles()` 会 warning 并 `FileClose()`。
它也会清理所有 allocated stdio files、dirs、transient fds。

第四层是 backend exit cleanup。
`InitTemporaryFileAccess()` 注册 `before_shmem_exit(BeforeShmemExit_Files, 0)`。
`BeforeShmemExit_Files()` 调用 `CleanupTempFiles(false, true)`。
process exit 时，它会清理所有临时文件，包括 interXact ones。

这四层不是重复设计。
它们对应不同失败边界：
- 正常路径靠 caller 显式 close。
- ERROR/abort 靠 ResourceOwner。
- transaction end 靠 EOXact backup。
- backend exit 靠 before_shmem_exit。

还有 postmaster startup 清理。
这不是当前 backend 的 cleanup。
它处理上一次 postmaster session 或 crash 后遗留在磁盘上的 temp files。
`RemovePgTempFiles()` 和 `RemovePgTempFilesInDir()` 负责这一层。

不要把 memory cleanup 和 fd cleanup 混为一谈。
`BufFile` struct 可以随 memory context 消失。
但底层 fd 和磁盘文件必须通过 ResourceOwner、FileClose 或 FileSet ownership 关闭和删除。

---

## 12. 正确性、错误边界与 fallback
fd.c 最常见的资源错误是 `EMFILE` 和 `ENFILE`。
`EMFILE` 通常表示当前进程 fd 达到限制。
`ENFILE` 通常表示系统级打开文件表耗尽。
对 fd.c 来说，处理策略相似：
记录 LOG，释放一个 LRU VFD fd，重试。

`BasicOpenFilePerm()` 遇到 `EMFILE` 或 `ENFILE` 时：
- 保存 errno。
- LOG “out of file descriptors: release and retry”。
- 调用 `ReleaseLruFile()`。
- 如果释放成功，重新 `open()`。
- 如果没有可释放 VFD，恢复 errno 并返回 -1。

`AllocateFile()`、`OpenPipeStream()`、`AllocateDir()` 也有类似 retry。
`OpenTransientFile()` 直接调用 `BasicOpenFilePerm()`，所以重试逻辑在底层。

这类 fallback 只对 VFD fd 有效。
如果 fd 都被 allocated desc 或 external fd 占满，LRU 可能没有东西可释放。
这就是为什么 `reserveAllocatedDesc()` 和 `AcquireExternalFD()` 要限制三分之一。

第二类错误是重新打开失败。
`LruInsert()` 重新打开 VFD 时调用 `BasicOpenFilePerm()`。
如果失败，返回 -1，`FileAccess()` 再把错误传给上层。
这意味着 `File` logical handle 存在不保证未来每次访问都成功。
底层文件可能被删除、权限可能变化、OS fd table 可能耗尽。

第三类错误是 close 失败。
`LruDelete()` 和 `FileClose()` 对 close failure 主要记录日志。
对非 temporary file，日志级别还会经过 `data_sync_elevel(LOG)`。
如果 `data_sync_retry` 关闭，这类 data sync 相关错误可能升级为 `PANIC`。
源码的原则是：
不要为了处理 close error 把 VFD 内部状态弄坏。

第四类错误是 fsync 失败。
`data_sync_elevel()` 的注释在 `fd.c:3968-3989`。
如果 fsync data file 失败，且 `data_sync_retry` 关闭，PostgreSQL 默认 PANIC。
原因是某些 OS 可能在 write-back failure 后丢弃脏数据。
以后再次 fsync 可能错误地返回成功。
这会破坏 WAL 能恢复数据页的基本假设。

第五类错误是 temp file limit。
`FileWriteV()` 在写入 `FD_TEMP_FILE_LIMIT` 文件前计算写后大小。
如果总临时文件大小超过 `temp_file_limit`，直接 `ereport(ERROR)`。
注释承认这在模块边界上不完美。
理论上更像应该返回 -1 设置 errno。
但调用者需要明确错误消息，且当前调用者通常会立刻 ERROR。

第六类错误是 short read/write。
`FileWriteV()` 对成功 write 会把 errno 设置为 `ENOSPC`。
这是为了让上层发现 short write 后用 `%m` 报错时得到合理提示。
relation layer 的 `mdextend()`、`mdwritev()` 会检查写入字节数，并把 short write 转成 disk full 或 file access error。

第七类错误是 temporary cleanup 失败。
删除 temp file 失败通常 LOG。
startup 清理遗留 temp files 失败也通常 LOG。
原因是 cleanup path 不应该轻易导致数据库无法继续运行。
但 relation fsync、data file write 等持久化路径的错误边界更严格。

第八类是 Windows 特殊边界。
`fd.h` 里 `FILE_POSSIBLY_DELETED(err)` 在 Windows 上把 `EACCES` 也视为可能删除。
原因是 Windows 上 unlink-but-not-yet-gone 的文件可能表现为 `EACCES`。
这在 relation file 路径上影响 `mdopenfork()` 和 `_mdfd_getseg()` 对缺失文件的判断。

---

## 13. 源码 walkthrough：一次 relation block read
这条 walkthrough 用 relation file 说明 VFD 如何隐藏 fd reopen。

上层要读一个 relation block。
storage manager 最终进入 `mdreadv()`。
`mdreadv()` 先通过 `_mdfd_getseg()` 找到 block 所在 segment。
这个 segment 在 `SMgrRelation` 的 `md_seg_fds[forknum]` 里对应一个 `MdfdVec`。
`MdfdVec.mdfd_vfd` 是 fd.c 的 `File`。

然后 `mdreadv()` 调用：

```text
FileReadV(v->mdfd_vfd, iov, iovcnt, seekpos, WAIT_EVENT_DATA_FILE_READ)
```

进入 fd.c 后：
- `FileReadV()` assert `FileIsValid(file)`。
- 调用 `FileAccess(file)`。
- 如果 VFD 物理 fd 已经被 LRU 关闭，`LruInsert()` 重新打开 pathname。
- 如果需要释放 fd，`ReleaseLruFiles()` 先关闭别的 LRU VFD。
- 成功后用 `pg_preadv(vfdP->fd, ...)` 读取。

上层 `mdreadv()` 完全不需要知道刚才是否发生了 reopen。
它只拿 `File`。
这就是 VFD abstraction 的价值。

但这也说明上层不能缓存 raw fd。
如果 `mdreadv()` 曾经调用 `FileGetRawDesc()` 并把 fd 长期存在别处，下一次 VFD LRU 可能已经关闭了它。
raw fd number 甚至可能被 OS 复用给另一个文件。
这就是 `FileGetRawDesc()` 注释警告“不要做太多别的事”的原因。

---

## 14. 源码 walkthrough：一次临时 BufFile 写入
这条 walkthrough 用 executor 临时工作文件说明 temporary cleanup。

某个 sort 或 hash 工作流创建 BufFile：

```text
BufFileCreateTemp(false)
  -> PrepareTempTablespaces()
  -> OpenTemporaryFile(false)
     -> ResourceOwnerEnlarge(CurrentResourceOwner)
     -> OpenTemporaryFileInTablespace(...)
        -> PathNameOpenFile(tempfilepath, O_RDWR | O_CREAT | O_TRUNC | PG_BINARY)
     -> 设置 FD_DELETE_AT_CLOSE | FD_TEMP_FILE_LIMIT
     -> RegisterTemporaryFile(file)
  -> makeBufFile(file)
```

之后上层调用 `BufFileWrite()`。
小写入先落在 `BufFile.buffer`。
buffer 满或 flush 时进入 `BufFileDumpBuffer()`。
它调用 `FileWrite(thisfile, ..., WAIT_EVENT_BUFFILE_WRITE)`。

`FileWrite()` 是 `fd.h` 的 inline wrapper。
它最终调用 `FileWriteV()`。
`FileWriteV()` 先 `FileAccess()`。
如果真实 fd 因 fd pressure 被关了，会重新打开。
然后检查 `temp_file_limit`。
再用 `pg_pwritev()` 写。
成功后更新 `temporary_files_size` 和当前 VFD 的 `fileSize`。

正常结束时：

```text
BufFileClose()
  -> BufFileFlush()
  -> FileClose(each File)
     -> close fd if open
     -> subtract temp size
     -> unlink if FD_DELETE_AT_CLOSE
     -> ResourceOwnerForgetFile
     -> FreeVfd
```

如果中途 ERROR：
- ResourceOwner release 调用 `ResOwnerReleaseFile()`。
- `ResOwnerReleaseFile()` 调用 `FileClose()`。
- anonymous temp file 被 unlink。
- EOXact cleanup 做 backup 检查。

这条链路的关键不是 BufFile 本身。
关键是底层 `File` 的 ResourceOwner ownership 在创建时已经建立。

---

## 15. 观测与诊断入口
fd.c 的内部状态不是普通 SQL view 能直接看到的。
但可以从几个入口间接观察。

第一，OS 层观察 fd 数量。
对某个 backend PID：

```bash
ls /proc/<pid>/fd | wc -l
```

在 fd pressure 实验中，你会看到这个数量不会随 logical `File` 数量线性增长。
如果 workload 产生大量 relation segment 或 temp segments，VFD 可以逻辑记住很多文件，但 OS fd 数会被 `max_safe_fds` 控住。

第二，日志观察 fd pressure。
当 `open()` 遇到 `EMFILE` 或 `ENFILE`，`BasicOpenFilePerm()`、`AllocateFile()` 等会 LOG：

```text
out of file descriptors: ...; release and retry
```

这个日志说明 fd.c 已经开始释放 LRU VFD 并重试。
如果之后仍然失败，说明没有可释放 VFD，或 OS/system-level 问题仍存在。

第三，观察临时文件。
开启 `log_temp_files` 可以看到被删除时的 temp file path 和 size。
`ReportTemporaryFileUsage()` 会向 pgstat 报告 temp file size。
`FileWriteV()` 对 `FD_TEMP_FILE_LIMIT` 文件维护 per-process temp size。

第四，用 wait event 区分路径。
relation read/write 使用 `WAIT_EVENT_DATA_FILE_READ`、`WAIT_EVENT_DATA_FILE_WRITE`。
BufFile 使用 `WAIT_EVENT_BUFFILE_READ`、`WAIT_EVENT_BUFFILE_WRITE`。
这些 wait event 不能直接告诉你是否发生 VFD reopen。
但能帮助判断 I/O 是 relation path 还是 executor temp path。

第五，调试构建可开 `FDDEBUG`。
`fd.c` 里 `DO_DB()` 包了大量 LRU 操作日志。
如果用自定义编译打开 `FDDEBUG`，可以看到 `LruInsert`、`LruDelete`、`ReleaseLruFile`、`FileAccess` 等路径。
这不是生产诊断手段。
它适合源码实验。

第六，gdb 断点。
常用断点：
- `FileAccess`
- `LruInsert`
- `ReleaseLruFile`
- `PathNameOpenFilePerm`
- `FileClose`
- `OpenTemporaryFile`
- `BasicOpenFilePerm`

在断点里看：
- `nfile`
- `numAllocatedDescs`
- `numExternalFDs`
- `max_safe_fds`
- `VfdCache[file].fd`
- `VfdCache[file].fileName`
- `VfdCache[0].lruMoreRecently`
- `VfdCache[0].lruLessRecently`

---

## 16. 常见误区
误区一：`File` 就是 fd。
不是。
`File` 是 VFD index。
只有 `VfdCache[file].fd` 才可能是真实 fd。

误区二：VFD LRU 关闭文件等于 `FileClose()`。
不是。
LRU 关闭只释放真实 fd。
logical file 仍存在。
`FileClose()` 才是释放 VFD slot 和 cleanup 语义的结束点。

误区三：`PathNameOpenFile()` 打开的文件永远物理打开。
不是。
它只是 logical open。
物理 fd 可能被 LRU 关闭，下一次 `FileAccess()` 再打开。

误区四：`OpenTransientFile()` 比 `PathNameOpenFile()` 更轻量，所以适合长期使用。
不对。
它返回 raw fd，不能被 VFD LRU 透明管理。
长期持有会增加 `numAllocatedDescs` 压力。

误区五：`BasicOpenFile()` 是 PostgreSQL 推荐的 `open()` 替代品。
不对。
它只是最低层 escape hatch。
没有自动 cleanup。
长期 fd 还要配合 external fd 计数。

误区六：`temp_file_limit` 限制所有临时名字文件。
不完全。
它限制设置了 `FD_TEMP_FILE_LIMIT` 的文件。
`PathNameOpenTemporaryFile()` 打开别人创建的 named temp file 时，不计入 caller 的 temp limit。

误区七：relation file close 会删除磁盘文件。
不会。
普通 relation file 的删除由 smgr/md 和 checkpoint/sync request 管。
fd.c 的 `FileClose()` 只处理 fd/VFD 和特定 temp flags。

误区八：提高 `max_files_per_process` 总是好事。
不一定。
它只影响 fd.c 可用上限。
太高可能增加每个 backend 同时占用的 fd 数，给系统级 fd table 和多进程 workload 带来压力。

---

## 17. 成本、资源与跨模块传播
VFD 的成本模型不是“打开文件一次有多贵”，而是 fd 预算如何在多类资源之间传播。`nfile` 表示 VFD 当前物理打开数，`numAllocatedDescs` 表示 `AllocateFile`/`OpenTransientFile`/directory 等短期资源，`numExternalFDs` 表示 fd.c 外部长期持有的裸 fd。三者一起逼近 `max_safe_fds` 时，VFD LRU 才有机会释放一部分真实 fd。

成本随这些变量扩张：backend 数增加会把 per-process fd limit 压力复制多份；relation 数和 segment 数增加 logical `File` 数；sort/hash spill 增加 temporary `File` 和 `BufFile` segment；扩展或外部库如果绕过 `AcquireExternalFD()`，会让 fd.c 低估真实压力。VFD 能控制同时打开的 OS fd 数，但不能消除 reopen、LRU ring 操作、路径名保存和 `open()`/`close()` thrashing。

跨模块边界也要分清：smgr/md 用 `PathNameOpenFile()` 把 relation segment 放进 VFD；executor spill 通过 `BufFile` 和 temporary `File` 使用 VFD；FileSet/SharedFileSet 只管理命名空间和 cleanup，不替代 fd LRU；ResourceOwner 兜底释放资源，但不能把 logical handle 自动变成 crash-safe relation storage。

## 18. 课堂实验
实验 1：画 VFD 三态图并用断点校验。

```gdb
break PathNameOpenFilePerm
break FileAccess
break LruInsert
break ReleaseLruFile
break FileClose
```

触发一次 relation read 或 temp spill，观察 `File`、`VfdCache[file].fd`、`VfdCache[file].fileName`。把 unused slot、logical open but physical closed、logical open and physical open 三种状态画出来。

实验 2：制造 fd pressure。
在隔离测试环境调低 `ulimit -n` 和 `max_files_per_process`，运行会产生大量 temp file 或 relation segment 的 workload。观察日志中的 `out of file descriptors: ... release and retry`，同时用 `/proc/<pid>/fd` 看 OS fd 数是否被限制在预算附近。不要在生产环境做这个实验。

实验 3：对照 temporary file 与 relation file cleanup。

```gdb
break BufFileCreateTemp
break FileWriteV
break FileClose
break mdopenfork
break mdclose
```

运行一个会 spill 的查询，再扫描或扩展一张 relation。比较 anonymous temp file close 时自动 unlink，和 relation file close 只释放 VFD、不删除磁盘文件的差异。

## 19. 讨论题
1. 为什么 `File` 必须是 backend-local VFD index，而不能暴露 OS fd？
2. `VfdCache[file].fileName != NULL` 和 `VfdCache[file].fd != VFD_CLOSED` 分别代表什么状态？
3. VFD LRU 关闭一个 fd 后，哪些语义还保留，哪些资源已经释放？
4. 为什么 `OpenTransientFile()` 不适合长期使用？
5. `ResourceOwner` 能保证哪些 cleanup？它不能保证哪些 crash-safety 语义？
6. `temp_file_limit` 为什么在写路径检查，而不是创建临时文件时检查？
7. 如何判断 “too many open files” 是 VFD、allocated desc、external fd 还是系统级 `ENFILE`？
8. 为什么缓存 `FileGetRawDesc()` 返回值是危险的？

## 20. 诊断顺序与可迁移规律
遇到 fd pressure，先看错误来源。如果日志出现 release/retry，说明 fd.c 已尝试释放 LRU VFD；如果仍失败，通常是可释放 VFD 不够、allocated desc/external fd 过多，或系统级 file table 耗尽。

然后区分资源类型：relation segment 和 anonymous temp file 是 VFD；`AllocateFile()`/`OpenTransientFile()` 是 fd.c 登记的短期资源；外部库长期 fd 必须通过 `AcquireExternalFD()` 或 `ReserveExternalFD()` 让 fd.c 计入预算。最后再看 workload：大量 spill、过多 relation segment、多 backend 并发扫描和小 fd limit 都会放大 reopen 和 cleanup 成本。

可迁移规律是：稳定 handle 不一定等于真实资源。`File` 代表 logical file identity，OS fd 是可回收物理资源。把二者拆开，可以让上层长期持有文件语义，同时让底层在资源压力下动态收缩；代价是所有访问必须经过抽象层，raw escape hatch 变危险，cleanup 和重新打开失败都必须被显式处理。

## 21. 本节小结
本节核心链路是：`PathNameOpenFile()` 或 `OpenTemporaryFile()` 创建 backend-local `File`，`FileAccess()` 在每次读写/sync 前确保真实 fd 可用，VFD LRU 在 fd pressure 下关闭冷文件，`FileClose()` 才结束 logical handle 和 cleanup 语义。

核心状态是 `VfdCache[file].fileName`、`VfdCache[file].fd`、LRU ring、`nfile`、`numAllocatedDescs`、`numExternalFDs` 和 `max_safe_fds`。MemoryContext 只能管 C 对象；ResourceOwner、EOXact、process exit 和 FileSet/SharedFileSet 才负责底层文件资源释放。

异常路径包括 `EMFILE`/`ENFILE` 后释放 LRU 并重试、重新打开失败、temp file limit 的写路径 ERROR、cleanup unlink LOG、data file fsync failure 可能 PANIC。观测入口主要是 `/proc/<pid>/fd`、日志、`log_temp_files`、wait event、`FDDEBUG` 和 gdb 断点；SQL 视图不能直接看到 VFD LRU 的内部状态。

最后带走的判断框架是：`File` 是稳定逻辑身份，OS fd 是可回收物理资源。只要某段代码绕过这个边界长期持有 raw fd，它就必须自己承担 fd budget、cleanup 和 reopen/failure 的后果。
