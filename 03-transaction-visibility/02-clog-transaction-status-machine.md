# PostgreSQL pg_xact / CLOG 状态机

## 课程定位

前置知识：已经知道 tuple header 中的 `xmin` / `xmax` 保存的是 XID，也知道可见性判断不能只看 tuple 自己，必须询问事务系统。

本节唯一主问题：

```text
pg_xact 只用两位记录事务终态时，可见性代码如何区分“事务仍在运行”、“事务已经提交”、“事务已经回滚”和“子事务已提交但父事务未定”？
```

核心矛盾：

```text
事务状态查询必须足够便宜，才能出现在 heap tuple visibility 的高频路径；
但事务结束、子事务树、异步提交、崩溃恢复和 CLOG 截断又要求状态发布有严格顺序。
```

一句话运行模型：

```text
运行中事务由 ProcArray 证明；
完成事务由 pg_xact 的两位状态证明；
提交路径先让 WAL 达到可恢复边界，再写 pg_xact，最后从 ProcArray 消失；
读路径先问“是否仍在运行”，再把 pg_xact 的状态解释成 committed / aborted / sub-committed。
```

学完后应能独立判断：

```text
为什么 pg_xact 中的 00 不能直接读成“事务正在运行”；
为什么 TransactionIdDidAbort() 不是普通 visibility 判断的首选入口；
为什么提交必须先写 pg_xact 再 ProcArrayEndTransaction()；
为什么跨 CLOG page 的子事务提交需要 SUB_COMMITTED 过渡状态；
为什么 async commit 要在 CLOG page 上记 group_lsn；
为什么 truncation 前要推进 oldestClogXid 并写 CLOG_TRUNCATE WAL；
哪些状态可以用 pg_stat_slru、wait event、pg_waldump、gdb 观察，哪些只能推断。
```

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

本节不展开 heap visibility 的完整分支。后续课程会专门讲 `HeapTupleSatisfiesMVCC()`、hint bit、`pg_subtrans` 父链和 SubXID overflow。

这里先把事务结果状态机本身讲清楚。

## 1. 本节在总主线中的位置

上一节讨论的是 XID 分配与事务身份边界：

```text
backend 可以先只有 virtual transaction id；
只有真正需要写入或进入 MVCC 语义时才分配 top-level XID；
XID 一旦写入 tuple header，就必须能被其它 backend 解释。
```

本节接着回答：

```text
当另一个 backend 在 tuple header 中看到一个 XID 时，
它到底从哪里知道这个 XID 的事务命运？
```

这个问题不能只靠 `pg_xact` 回答。原因是 `pg_xact` 的初始值就是 `TRANSACTION_STATUS_IN_PROGRESS`。

但是 `pg_xact` 文件并不知道一个 backend 此刻是否还活着。一个 XID 对应的两位仍是 00，可能意味着：

```text
1. 事务确实仍在 ProcArray 中运行；
2. 事务已经崩溃或未写出 abort record，系统按未提交处理；
3. 状态页刚初始化，那个 XID 还没被分配过；
4. 读者不该再查这个 XID，因为对应 CLOG segment 已被安全截断。
```

所以普通可见性判断的 mental model 不是：

```text
读 pg_xact -> 得到状态
```

而是：

```text
先用 ProcArray 判定 running；
running 则当前不可见或需要等待；
not running 再读 pg_xact；
pg_xact committed 则提交；
pg_xact aborted 或仍 00 则未提交；
pg_xact sub-committed 则追父事务。
```

本节的核心对象是 `pg_xact`，但核心边界是：

```text
ProcArray running-state publication
  + pg_xact final-state storage
  + WAL / SLRU crash-safety protocol
  + pg_subtrans parent-chain fallback
```

下一节会继续讲提交顺序、WAL 与 CLOG 持久化边界。本节只保留必要的 WAL 规则，用来解释为什么 CLOG 状态不能随意提前发布。


## 2. 核心矛盾与一句话运行模型

`pg_xact` 的设计极端紧凑。`src/backend/access/transam/clog.c` 开头就给出事实：

```text
每个事务只占 2 bits；
四个事务状态放在一个 byte；
一个 CLOG page 是 BLCKSZ 大小；
默认 8KB page 上可保存 32768 个事务状态。
```

这个紧凑设计服务 hot path：

```text
heap tuple visibility 可能为每个 tuple 的 xmin / xmax 查询事务结果；
事务状态不能是一个大对象、hash 表或 per-transaction 文件。
```

代价也很明显：

```text
两位只能表达很少状态；
它不能保存“谁正在运行”；
不能保存子事务完整树；
不能保存精确 commit LSN；
不能独立表达 crash recovery 的全部顺序。
```

PostgreSQL 的做法是把事务状态拆成几层：

| 层级 | 保存位置 | 回答的问题 |
| --- | --- | --- |
| running state | `ProcArray` / `PGPROC` / `KnownAssignedXids` | 这个 XID 或它的父事务是否仍在运行？ |
| final status bits | `pg_xact` / CLOG SLRU | 事务是否 committed、aborted、sub-committed？ |
| parent chain | `pg_subtrans` | sub-committed 子事务的 top-level 父事务是谁？ |
| durability ordering | WAL commit / abort record、CLOG page `group_lsn` | CLOG 的 committed 状态何时允许落盘或被恢复重放？ |
| retention horizon | `oldestClogXid`、datfrozenxid、VACUUM / freeze | 哪些旧 CLOG segment 已不再需要查询？ |

一句话运行模型可以写成：

```text
ProcArray 负责“现在还活着吗”，pg_xact 负责“结束后是什么结果”，pg_subtrans 负责“子事务该归谁”，WAL/SLRU 负责“这些结果能否安全持久化和恢复”。
```

注意这里的“状态机”不是单个 enum。它是多个状态源按固定顺序组合后的语义状态机。

如果只读 `clog.h` 的四个宏，就会漏掉最重要的部分：

```c
#define TRANSACTION_STATUS_IN_PROGRESS    0x00
#define TRANSACTION_STATUS_COMMITTED      0x01
#define TRANSACTION_STATUS_ABORTED        0x02
#define TRANSACTION_STATUS_SUB_COMMITTED  0x03
```

`IN_PROGRESS` 这个名字容易误导。它是 `pg_xact` page 初始化后的 all-zero 状态。

它不等于“这个事务一定正在某个 backend 里运行”。只有结合 `TransactionIdIsInProgress()` 的 ProcArray 结果，才能把 00 解释成真正的运行中。


## 3. 核心文件分工与阅读顺序

按 mental model 阅读，而不是按文件名排序。

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/access/clog.h` | 四个 `XidStatus` 值、`xl_clog_truncate`、CLOG 对外入口。 |
| 2 | `src/backend/access/transam/transam.c` | `TransactionIdDidCommit()` / `TransactionIdDidAbort()` 如何把 CLOG 状态解释成语义结果。 |
| 3 | `src/backend/storage/ipc/procarray.c` | `TransactionIdIsInProgress()` 为什么要先判断 running，慢路径为什么会回查 `pg_subtrans` 和 `pg_xact`。 |
| 4 | `src/backend/access/transam/xact.c` | commit / abort 何时写 WAL、何时写 CLOG、何时从 ProcArray 消失。 |
| 5 | `src/backend/access/transam/clog.c` | 两位状态如何定位、写入、分 page、处理子事务树和 group update。 |
| 6 | `src/include/access/slru.h` | `SlruSharedData` 中 slot、dirty、page status、`group_lsn` 的语义。 |
| 7 | `src/backend/access/transam/slru.c` | CLOG page 如何读入、写出、等待 IO、checkpoint flush、truncate。 |
| 8 | `src/backend/access/transam/varsup.c` | `GetNewTransactionId()` 分配新 XID 前为什么必须 `ExtendCLOG()`。 |
| 9 | `src/include/access/transam.h` | `TransamVariables` 中 `latestCompletedXid`、`xactCompletionCount`、`oldestClogXid` 的边界。 |

推荐具体跟读路径：

```text
clog.h 的 XidStatus
  -> transam.c::TransactionLogFetch()
  -> transam.c::TransactionIdDidCommit()
  -> transam.c::TransactionIdDidAbort()
  -> procarray.c::TransactionIdIsInProgress()
  -> xact.c::RecordTransactionCommit()
  -> xact.c::RecordTransactionAbort()
  -> clog.c::TransactionIdSetTreeStatus()
  -> clog.c::TransactionIdSetStatusBit()
  -> clog.c::TransactionIdGetStatus()
  -> slru.c::SimpleLruReadPage_ReadOnly()
  -> slru.c::SlruPhysicalWritePage()
  -> clog.c::ExtendCLOG() / TruncateCLOG()
```

不要一开始就从 `clog.c` 顶部顺序往下背函数。本节关注的是状态转换，不是 API 列表。

先看读者如何解释状态，再看写者如何发布状态，最后看底层如何保存状态。


## 4. 关键数据结构与状态

### 4.1 `XidStatus`: 两位不是完整事务对象

`src/include/access/clog.h` 定义：

```c
typedef int XidStatus;
#define TRANSACTION_STATUS_IN_PROGRESS    0x00
#define TRANSACTION_STATUS_COMMITTED      0x01
#define TRANSACTION_STATUS_ABORTED        0x02
#define TRANSACTION_STATUS_SUB_COMMITTED  0x03
```

这四个值占用两个 bits。本节建议把它们翻译成下面的语义：

| CLOG 位值 | 名称 | 可见性语义 |
| --- | --- | --- |
| `00` | `IN_PROGRESS` | CLOG 没有记录完成结果；是否真的 running 要问 ProcArray。 |
| `01` | `COMMITTED` | top-level 事务或已归属的子事务最终提交。 |
| `10` | `ABORTED` | 事务显式回滚，或 abort WAL 重放后被标记。 |
| `11` | `SUB_COMMITTED` | 子事务自己已提交，但父事务最终结果还没确定或需要追溯。 |

`SUB_COMMITTED` 是本节最重要的中间态。它只对子事务有意义。

如果一个子事务写过 tuple，tuple header 里可能保存的是 subxid。子事务 commit 不等于 tuple 对外可见。

只有 top-level transaction commit 后，这个子事务的写入才真正对外提交。因此 `TransactionIdDidCommit()` 看到 `SUB_COMMITTED` 时不会直接返回 true。

它会去 `pg_subtrans` 找父 XID，再递归判断父事务是否提交。

### 4.2 CLOG page 定位：XID 到 page / byte / bit

`clog.c` 的关键宏：

```text
CLOG_BITS_PER_XACT = 2
CLOG_XACTS_PER_BYTE = 4
CLOG_XACTS_PER_PAGE = BLCKSZ * 4
```

定位关系：

```text
pageno = xid / CLOG_XACTS_PER_PAGE
pgindex = xid % CLOG_XACTS_PER_PAGE
byteno = pgindex / 4
bindex = xid % 4
bshift = bindex * 2
```

写入状态时，`TransactionIdSetStatusBit()` 做的事情非常小：

```text
找到 page slot；
找到 byte；
清掉对应两个 bits；
写入新 status；
必要时更新 group_lsn；
把 page 标脏。
```

这个小操作是高频路径友好的关键。但是它能成立，是因为复杂性被推到外围：

```text
page 是否在内存由 SLRU 管；
page 是否可并发修改由 SLRU bank lock 管；
WAL-before-CLOG 由 xact.c 和 group_lsn 管；
跨 page 原子性由 SUB_COMMITTED 过渡状态管；
运行中状态由 ProcArray 管。
```

### 4.3 `XactCtl` 与 SLRU shared state

`clog.c` 中：

```c
static SlruDesc XactSlruDesc;
#define XactCtl (&XactSlruDesc)
```

`XactCtl` 是 CLOG 使用的 SLRU descriptor。`CLOGShmemRequest()` 通过 `SimpleLruRequest()` 注册：

```text
name = "transaction"
Dir = "pg_xact"
nslots = CLOGShmemBuffers()
nlsns = CLOG_LSNS_PER_PAGE
sync_handler = SYNC_HANDLER_CLOG
PagePrecedes = CLOGPagePrecedes
```

`src/include/access/slru.h` 中 `SlruSharedData` 的关键字段：

| 字段 | CLOG 语义 |
| --- | --- |
| `page_buffer[]` | 每个 slot 的 8KB CLOG page 内容。 |
| `page_status[]` | slot 是 empty、read in progress、valid、write in progress。 |
| `page_dirty[]` | 该 CLOG page 是否需要写回 `pg_xact` 文件。 |
| `page_number[]` | slot 当前保存哪个 CLOG page。 |
| `bank_locks[]` | 保护同一 bank 内 slot 元数据和 page 内容访问。 |
| `buffer_locks[]` | 保护单个 slot 的物理 IO。 |
| `group_lsn[]` | async commit 时，每组 XID 对应的 WAL flush 下限。 |
| `latest_page_number` | 当前末尾 page，用于避免错误淘汰和 truncation backstop。 |

`SlruDesc` 是 backend-private handle，里面指向 shared memory。不要把 `SlruDesc *` 当成可以跨进程共享的对象。

真正 shared 的是 `SlruSharedData`。

### 4.4 `TransamVariables`: 全局边界状态

`src/include/access/transam.h` 中 `TransamVariablesData` 对本节重要的字段：

| 字段 | 语义 |
| --- | --- |
| `nextXid` | 下一个可分配 FullTransactionId，受 `XidGenLock` 保护。 |
| `latestCompletedXid` | 最新完成的 XID，用于 snapshot / in-progress shortcut。 |
| `xactCompletionCount` | 完成计数，用于 snapshot reuse 等判断。 |
| `oldestClogXid` | 最老仍允许查 CLOG 的 XID，受 `XactTruncationLock` 保护。 |

`oldestClogXid` 不是 CLOG 文件里的一位状态。它是查询边界。

CLOG truncation 前要先推进它，让并发任意 XID 状态查询知道哪些 XID 已不能再访问 `pg_xact` 文件。

### 4.5 `PGPROC` group update 状态

CLOG 写入提交状态可能成为 LWLock contention 点。`clog.c` 的 group update 使用 `PGPROC` 中的一组字段：

```text
clogGroupMember
clogGroupMemberXid
clogGroupMemberXidStatus
clogGroupMemberPage
clogGroupMemberLsn
clogGroupNext
```

这些字段不是事务语义的一部分。它们是 commit hot path 的并发优化。

当多个 backend 同时要更新同一个 CLOG page，leader 可以一次拿 SLRU bank lock，替 follower 写多个 XID 的 status bits，然后唤醒 follower。优化边界很窄：

```text
只有所有 XID 在同一 CLOG page；
子事务数量不超过 THRESHOLD_SUBTRANS_CLOG_OPT；
MyProc 中的 xid/subxids 与要写入的树一致；
拿不到 bank lock 时才尝试 group update。
```

这说明 CLOG 状态机不是只为 correctness 服务。它也被 commit latency 和 lock handoff 成本塑形。


## 5. 主流程源码 walkthrough

### 5.1 读路径：从 tuple XID 到事务结果

普通 visibility 代码最终会需要回答：

```text
tuple.xmin 对当前 snapshot 是否已提交？
tuple.xmax 对当前 snapshot 是否代表删除/更新已提交？
```

本节不展开 heap 分支，只看事务状态查询的核心路径：

```text
TransactionIdIsInProgress(xid)
  -> 若 true：事务仍运行，不能把 CLOG 00 当终态
  -> 若 false：进入 TransactionIdDidCommit(xid) 或 TransactionIdDidAbort(xid)
      -> TransactionLogFetch(xid)
          -> TransactionIdGetStatus(xid, &lsn)
              -> SimpleLruReadPage_ReadOnly(XactCtl, pageno, &xid)
              -> 取两位 status 和 group_lsn
      -> 根据 COMMITTED / ABORTED / SUB_COMMITTED / IN_PROGRESS 解释
```

`transam.c::TransactionLogFetch()` 有一个 single-item cache：

```text
cachedFetchXid
cachedFetchXidStatus
cachedCommitLSN
```

它只缓存不会再变化的结果：

```text
不缓存 IN_PROGRESS；
不缓存 SUB_COMMITTED；
只缓存 COMMITTED 或 ABORTED。
```

这体现了一个不变量：

```text
final status 可以缓存；
intermediate / unfinished status 不能缓存。
```

`TransactionIdDidCommit()` 的解释逻辑：

```text
status == COMMITTED
  -> true
status == SUB_COMMITTED
  -> 如果 transactionId < TransactionXmin，不能再查 pg_subtrans，按未提交处理
  -> 否则 SubTransGetParent(transactionId)
  -> parent 无效则 WARNING，并按未提交处理
  -> 递归 TransactionIdDidCommit(parentXid)
其它状态
  -> false
```

`TransactionIdDidAbort()` 的解释逻辑相似，但语义要谨慎：

```text
status == ABORTED
  -> true
status == SUB_COMMITTED
  -> 父事务无法追溯或父事务 abort，则 true
其它状态
  -> false
```

源码注释特别提醒：

```text
TransactionIdDidAbort() 只对显式 abort 返回 true；
崩溃导致的隐式 abort 通常仍会在 CLOG 中显示为 IN_PROGRESS；
多数场景应先 TransactionIdIsInProgress()，再 TransactionIdDidCommit()。
```

所以普通 visibility 的稳定模式是：

```text
running?
  yes -> 运行中
  no  -> committed?
          yes -> 已提交
          no  -> 未提交，按 aborted/failed 处理
```

而不是：

```text
did abort?
  no -> committed
```

这就是本节主问题的答案核心。`pg_xact` 自己无法区分 still-running 和 crash-aborted。

这个区分来自：

```text
ProcArray 的 running state
  + CLOG 的 final status
  + “not running and not committed => not visible” 的可见性规则
```

### 5.2 写路径：同步提交

`xact.c::CommitTransaction()` 的关键顺序：

```text
PreCommit callbacks / checks
  -> HOLD_INTERRUPTS()
  -> s->state = TRANS_COMMIT
  -> RecordTransactionCommit()
  -> ProcArrayEndTransaction(MyProc, latestXid)
  -> post-commit cleanup
  -> locks/resource owner/memory cleanup
```

`RecordTransactionCommit()` 中，已有 XID 的事务会：

```text
START_CRIT_SECTION()
MyProc->delayChkptFlags |= DELAY_CHKPT_IN_COMMIT
XactLogCommitRecord(...)
TransactionTreeSetCommitTsData(...)
```

然后根据同步提交策略分支。同步提交或必须强制同步的场景：

```text
XLogFlush(XactLastRecEnd)
TransactionIdCommitTree(xid, nchildren, children)
```

此时 `TransactionIdCommitTree()` 进入 `transam.c`：

```text
TransactionIdCommitTree()
  -> TransactionIdSetTreeStatus(xid, nxids, xids,
                                TRANSACTION_STATUS_COMMITTED,
                                InvalidXLogRecPtr)
```

`InvalidXLogRecPtr` 的含义是：

```text
调用方已经保证 commit WAL record 持久化；
CLOG page 写出时不需要再从 group_lsn 追 WAL flush。
```

CLOG 写完后，`RecordTransactionCommit()` 退出 critical section：

```text
MyProc->delayChkptFlags &= ~DELAY_CHKPT_IN_COMMIT
END_CRIT_SECTION()
```

然后才执行：

```text
ProcArrayEndTransaction(MyProc, latestXid)
```

这条顺序极其重要：

```text
WAL commit record durable
  -> pg_xact shows committed
  -> ProcArray no longer shows running
```

如果顺序反过来，一个读者可能看到：

```text
ProcArray 中事务已经不 running；
pg_xact 还不是 committed；
于是把已提交事务误判为未提交。
```

PostgreSQL 不允许这个窗口暴露给可见性读者。

### 5.3 写路径：异步提交

`synchronous_commit = off` 或某些不需要立即 flush 的场景进入 async commit。`RecordTransactionCommit()` 中：

```text
XLogSetAsyncXactLSN(XactLastRecEnd)
TransactionIdAsyncCommitTree(xid, nchildren, children, XactLastRecEnd)
```

`TransactionIdAsyncCommitTree()` 仍然把 CLOG 标成 committed。区别是它传入 commit record 的 LSN。

`clog.c::TransactionIdSetStatusBit()` 在 `lsn` 有效时更新：

```text
XactCtl->shared->group_lsn[lsnindex] = max(existing, lsn)
```

后续 `slru.c::SlruPhysicalWritePage()` 写出 CLOG page 时会扫描该 page 的 `group_lsn[]`：

```text
max_lsn = page 上所有 lsn group 的最大值
if max_lsn valid:
    XLogFlush(max_lsn)
```

这回答了一个常见疑问：

```text
async commit 既然没有立即 flush WAL，为什么可以马上写 pg_xact committed？
```

答案是：

```text
CLOG 可以在内存中发布 committed；
但 CLOG page 不能在对应 commit WAL record 之前落盘；
group_lsn 把这个 WAL-before-CLOG-data 约束推迟到 SLRU write path。
```

如果 postmaster crash 发生在 WAL buffer 未持久化前，异步提交事务可能丢失。这正是 async commit 允许的语义。

如果 CLOG page 已经要落盘，`group_lsn` 会强制先 flush WAL，避免 crash 后出现：

```text
pg_xact 说 committed；
WAL 中没有对应 commit record；
recovery 无法解释这个 committed。
```

### 5.4 写路径：abort

`xact.c::AbortTransaction()` 的关键顺序：

```text
AfterTriggerEndXact(false)
AtAbort_* cleanup
RecordTransactionAbort(false)
ProcArrayEndTransaction(MyProc, latestXid)
post-abort cleanup
CleanupTransaction()
```

`RecordTransactionAbort()` 中：

```text
如果没有 XID，直接返回；
如果已有 XID，先检查它没有已经 committed；
写 ABORT WAL record；
XLogSetAsyncXactLSN(XactLastRecEnd)
TransactionIdAbortTree(xid, nchildren, children)
```

Abort 的 WAL 不需要像 commit 一样先 flush。源码注释给出的理由是：

```text
crash 后默认假设未完成事务 abort；
丢失 abort record 不会让事务错误地变 committed。
```

但是 PostgreSQL 仍会尽量标 CLOG aborted。这有两个好处：

```text
后续状态查询可以快速看到 ABORTED；
子事务 abort 时，XactLockTableWait 等路径可以避免等待已经 aborted 的 subtransaction。
```

abort 路径同样遵守：

```text
先写 pg_xact aborted；
再 ProcArrayEndTransaction() 宣告不 running。
```

### 5.5 子事务树：跨 CLOG page 的提交原子性

`clog.c::TransactionIdSetTreeStatus()` 是本节最值得细读的函数。它要更新：

```text
top-level xid
若干 committed subxids
```

如果所有 XID 都在同一个 CLOG page：

```text
拿一次 SLRU bank lock；
先把 subxids 标 SUB_COMMITTED；
再把 top-level xid 标 COMMITTED；
最后把 subxids 标 COMMITTED；
同一 page 上对并发读者基本表现为一个受锁保护的更新。
```

如果 subxids 分散在多个 CLOG page，就不能一次锁住所有 page。源码采用三阶段：

```text
1. 对不在 top-level xid page 的 subxids：
   先按 page 分组标 SUB_COMMITTED。
2. 对 top-level xid 所在 page：
   把 top-level xid 和同页 subxids 标 COMMITTED。
3. 回到其它 page：
   把那些 subxids 从 SUB_COMMITTED 改成 COMMITTED。
```

为什么要这样？因为并发读者看到某个远端 page 上的 subxid 时，不能因为它还没被最终改成 COMMITTED 就误判为 abort。

`SUB_COMMITTED` 告诉读者：

```text
这个 subxid 自己已经成功结束；
请通过 pg_subtrans 查父事务；
父事务 commit 才代表它对外 commit。
```

当 top-level xid 已经标 COMMITTED 后，即使某些 subxids 仍暂时停留在 `SUB_COMMITTED`，读者追到父事务也会得到 committed。这就是跨 page “看起来原子”的实现方式。

严格说，它不是物理上一次写完所有 bits。它是通过中间态和父链，让并发观察者得到一致语义。

### 5.6 CLOG bit 写入：`TransactionIdSetStatusBit()`

`TransactionIdSetStatusBit()` 的关键断言：

```text
当前 page slot 必须就是目标 XID 所在 page；
调用者必须持有对应 SLRU bank lock 的 exclusive 模式。
```

状态转换约束：

```text
curval == 0
  或 curval == SUB_COMMITTED 且目标不是 IN_PROGRESS
  或 curval == 目标状态
```

这意味着正常路径不允许：

```text
COMMITTED -> ABORTED
ABORTED -> COMMITTED
COMMITTED -> IN_PROGRESS
```

recovery 中有一个特殊容忍：

```text
如果正在 replay SUB_COMMITTED，
但当前状态已经是 COMMITTED，
则当作 no-op。
```

这是 redo 幂等性的一部分。写入两个 bits 后，如果 `lsn` 有效，就更新对应 `group_lsn`。

最后由上层把 page 标 dirty：

```text
XactCtl->shared->page_dirty[slotno] = true
```

### 5.7 recovery 路径：redo commit / abort

`xact.c::xact_redo_commit()` 在 redo commit record 时：

```text
AdvanceNextFullTransactionIdPastXid(max_xid)
TransactionTreeSetCommitTsData(...)
TransactionIdCommitTree(...) 或 TransactionIdAsyncCommitTree(..., lsn)
Hot Standby 下先 RecordKnownAssignedTransactionIds(max_xid)
标 CLOG 后 ExpireTreeKnownAssignedTransactionIds(...)
处理 invalidation 和 standby locks
```

Hot Standby 下顺序仍然是：

```text
先让 pg_xact 表达 committed；
再让 KnownAssignedXids / ProcArray 视角不再 running。
```

`xact_redo_abort()` 对 abort 也保持：

```text
RecordKnownAssignedTransactionIds(max_xid)
TransactionIdAbortTree(...)
ExpireTreeKnownAssignedTransactionIds(...)
```

recovery 中的 CLOG 更新同样遵守状态机。它不是简单“回放一个文件写”。

事务 WAL record 是 source of truth，redo 会重新设置 CLOG bits。


## 6. 生命周期 / ownership / cleanup

### 6.1 创建：initdb、startup、XID 分配

`BootStrapCLOG()` 在系统安装时创建初始 CLOG page：

```text
SimpleLruZeroAndWritePage(XactCtl, 0)
```

`StartupCLOG()` 在启动时根据 `TransamVariables->nextXid` 初始化 `latest_page_number`。`GetNewTransactionId()` 分配 XID 前会调用：

```text
ExtendCLOG(xid)
ExtendCommitTs(xid)
ExtendSUBTRANS(xid)
```

`ExtendCLOG()` 只在新 CLOG page 的第一个 XID 上工作。它在 `XidGenLock` 下执行，原因非常直接：

```text
不能先把 XID 发给事务，再发现对应 CLOG page 还没初始化；
也不能让后续更大的 XID 先 commit，然后被零页覆盖。
```

`ExtendCLOG()` 的动作：

```text
如果 newestXact 不是 page 第一个 XID，返回；
拿对应 SLRU bank lock；
SimpleLruZeroPage(XactCtl, pageno)；
写 CLOG_ZEROPAGE WAL record；
释放 lock。
```

新 page 初始化成全零。这也再次说明 `IN_PROGRESS = 0` 是物理初始值，不是完整 running 事实。

### 6.2 持有：shared SLRU，不属于单个 backend

CLOG page 的内存 owner 是共享 SLRU。单个 backend 只是在读写某个 slot 时持有锁。

锁语义：

```text
SLRU bank lock
  -> 保护同一 bank 的 slot 元数据和 page 内容访问。
SLRU buffer lock
  -> 保护某个 slot 的物理 IO。
ProcArrayLock
  -> 保护 running-state publication 和 transaction completion ordering。
XidGenLock
  -> 保护 XID 分配与 CLOG/SUBTRANS/CommitTs 扩展顺序。
XactTruncationLock
  -> 保护 oldestClogXid 推进与任意 XID 状态查询的截断边界。
```

没有 per-XID 对象需要释放。XID 的状态是 CLOG page 中两个 bits。

cleanup 的单位不是事务对象，而是：

```text
transaction end 时写 bits；
checkpoint 时写 dirty CLOG pages；
freeze / truncation 后删除旧 CLOG segments。
```

### 6.3 释放：transaction end cleanup

事务结束后，CLOG bits 不会马上释放。它们必须保留到所有可能引用这些 XID 的 tuple 都已经被处理：

```text
旧 tuple 被 VACUUM 回收；
老 XID 被 freeze；
datfrozenxid 推进；
CLOG segment 早于保留边界；
TruncateCLOG() 删除旧 segment。
```

commit 后 cleanup 顺序的核心是：

```text
RecordTransactionCommit()
  -> CLOG 已经可回答 committed
ProcArrayEndTransaction()
  -> 其它 backend 不再把我看作 running
ResourceOwnerRelease(...)
  -> 释放 pin、锁、外部资源
AtEOXact_*()
  -> 清理 cache、snapshot、stats、memory
```

abort 后类似：

```text
RecordTransactionAbort()
  -> CLOG 尽量可回答 aborted
ProcArrayEndTransaction()
  -> 其它 backend 不再把我看作 running
Abort cleanup / CleanupTransaction()
  -> 释放资源和内存上下文
```

如果 ERROR 发生在事务内部，PostgreSQL 不是靠 CLOG 自动清理内存。CLOG 只记录事务结果。

资源释放靠：

```text
ResourceOwnerRelease()
AtEOXact_*()
AtCleanup_Memory()
TopTransactionContext reset
```

不要把 CLOG 当成 ResourceOwner。

### 6.4 长期清理：checkpoint 与 truncation

`CheckPointCLOG()`：

```text
SimpleLruWriteAll(XactCtl, true)
```

它写出 dirty CLOG pages，并把 sync request 交给 checkpointer / sync.c 体系。`TruncateCLOG(oldestXact, oldestxid_datoid)`：

```text
cutoffPage = TransactionIdToPage(oldestXact)
SlruScanDirectory(...ReportPresence...)
AdvanceOldestClogXid(oldestXact)
WriteTruncateXlogRec(cutoffPage, oldestXact, oldestxid_datoid)
SimpleLruTruncate(XactCtl, cutoffPage)
```

这里顺序也不能随便改。先推进 `oldestClogXid`，是为了让并发状态查询不要继续访问即将删除的 CLOG 数据。

写 `CLOG_TRUNCATE` WAL 并 flush，是为了 crash recovery / standby 知道保留边界。真正删除 segment 由 `SimpleLruTruncate()` 执行。


## 7. 正确性机制层次

### 7.1 Visibility 不是 lock

`TransactionIdDidCommit()` 回答：

```text
这个 XID 是否最终提交？
```

它不保护 tuple 被并发修改。行锁、tuple lock、MultiXact、heavyweight lock 是另一层。

同样，`TransactionIdIsInProgress()` 回答：

```text
某个 XID 是否仍处于 running set？
```

它不等于等待事务结束。等待通常走 `XactLockTableWait()` 等机制。

### 7.2 ProcArray ordering 保证 running-state 边界

提交路径中：

```text
RecordTransactionCommit()
  -> pg_xact committed
ProcArrayEndTransaction()
  -> 清掉 MyProc->xid / ProcGlobal->xids[]
```

`ProcArrayEndTransaction()` 注释明确：

```text
transaction commit/abort must already be reported to WAL and pg_xact.
```

这保证读者不会在“not running 但 CLOG 未发布终态”的窗口误判。

### 7.3 WAL-before-CLOG-data

普通同步提交：

```text
XLogFlush(commit_lsn)
TransactionIdCommitTree(...)
```

async commit：

```text
TransactionIdAsyncCommitTree(..., commit_lsn)
group_lsn 记录 commit_lsn
SlruPhysicalWritePage() 写 CLOG page 前 XLogFlush(max_lsn)
```

CLOG page 的写出遵守 WAL-before-data。这里的 data 不是 heap page，而是 `pg_xact` 文件本身。

### 7.4 SLRU bank lock 保证 page 内 bit 修改一致

`TransactionIdSetStatusBit()` 只在持有对应 SLRU bank lock exclusive 时执行。`TransactionIdGetStatus()` 使用 `SimpleLruReadPage_ReadOnly()`，读到 slot 后再解析 bits。

SLRU 负责：

```text
page 不在内存则读入；
read/write in progress 时等待或按 write_ok 规则处理；
dirty page eviction 前写出；
IO 错误时恢复 slot 状态后 ereport。
```

这保证 CLOG bits 的内存访问不会被并发 IO 和 slot reuse 破坏。

### 7.5 子事务正确性来自 CLOG + SUBTRANS

`SUB_COMMITTED` 不是终态。它需要 `pg_subtrans` 父链解释。

当 `TransactionIdDidCommit()` 看到 `SUB_COMMITTED`：

```text
transactionId < TransactionXmin
  -> 不能再信任 pg_subtrans，按未提交处理
SubTransGetParent(transactionId) 无效
  -> WARNING，按未提交处理
parent 有效
  -> 递归判断 parent
```

这说明 CLOG 和 SUBTRANS 的 retention horizon 必须配合。后续课程会专门讲 `pg_subtrans`。

本节只需要记住：

```text
CLOG 的 SUB_COMMITTED 位本身不够；
field + parent-chain + TransactionXmin 边界才是语义。
```

### 7.6 Crash recovery 不是读取旧 CLOG 即可

commit / abort WAL record 才是恢复过程中的事务完成记录。redo 时：

```text
xact_redo_commit()
  -> TransactionIdCommitTree() 或 TransactionIdAsyncCommitTree()
xact_redo_abort()
  -> TransactionIdAbortTree()
```

CLOG 的 zero page / truncate 有自己的 rmgr record：

```text
CLOG_ZEROPAGE
CLOG_TRUNCATE
```

但是普通 commit/abort 的 CLOG bit 更新依赖 xact rmgr redo。不要误以为每次 CLOG bit 改动都有一条 CLOG WAL record。


## 8. 错误路径 / 异常路径 / fallback

### 8.1 CLOG 仍是 00，但事务不在 ProcArray

这是最常见也最容易误解的边界。情况可能是：

```text
backend crash；
postmaster crash；
事务未写 commit record；
abort record 丢失或没必要持久化；
CLOG 没被标 ABORTED。
```

普通 visibility 不能说：

```text
CLOG 是 IN_PROGRESS，所以事务还活着。
```

它应该先问 ProcArray。如果不 running，并且 `TransactionIdDidCommit()` false，则按未提交处理。

这就是 PostgreSQL 能容忍 abort record 不 flush 的原因。

### 8.2 跨 page 子事务提交时的中间状态

并发读者可能看到：

```text
subxid = SUB_COMMITTED
top xid = COMMITTED
```

也可能在短窗口看到：

```text
某些远端 page 的 subxid = SUB_COMMITTED
top xid 正在同页更新
```

正确处理方式不是等待所有 page 都变 COMMITTED。而是：

```text
看到 SUB_COMMITTED -> 查 pg_subtrans 父链 -> 判断父事务。
```

如果父事务还 running，则当前子事务对普通 snapshot 不可见。如果父事务 committed，则可见。

如果父事务 abort 或无法追溯，则不可见。

### 8.3 group update 失败或不适用

`TransactionIdSetPageStatus()` 尝试 group update 的前提很严格。不满足时直接走普通路径：

```text
LWLockAcquire(bank_lock, LW_EXCLUSIVE)
TransactionIdSetPageStatusInternal(...)
LWLockRelease(bank_lock)
```

如果尝试加入 group 时发现当前 leader 对应 page 不同：

```text
proc->clogGroupMember = false
clogGroupNext = INVALID_PROC_NUMBER
return false
```

调用者 fallback 到自己等待 bank lock。所以 group update 是性能优化，不是 correctness requirement。

### 8.4 SLRU IO 失败

`SimpleLruReadPage()` 读失败时，会先把 slot 状态恢复：

```text
READ_IN_PROGRESS -> EMPTY
释放 buffer lock
SlruReportIOError()
```

写失败时：

```text
WRITE_IN_PROGRESS -> VALID
page_dirty = true
释放 buffer lock
SlruReportIOError()
```

这避免 shared SLRU slot 永久卡在 IO in progress。错误会作为 ERROR 或更高等级报告。

对于 `pg_xact` 查询，`clog_errdetail_for_io_error()` 会补充：

```text
Could not access commit status of transaction %u.
```

### 8.5 recovery 中读不到已截断 SLRU 文件

`SlruPhysicalReadPage()` 有一个 recovery-only fallback：

```text
如果文件不存在且 InRecovery，
记录 LOG，
把 page 当 zeroes 返回。
```

原因是 redo 可能从较早 checkpoint 开始，遇到一些已经被后续生命周期截断的 CLOG segment。在 recovery 场景下，把缺失 page 读成 zeroes 可以让后续 WAL redo 重新设置必要状态。

正常运行中缺文件则是错误。

### 8.6 truncation 与任意 XID 查询的边界

`TruncateCLOG()` 删除旧 segment 前会：

```text
AdvanceOldestClogXid(oldestXact)
```

`varsup.c` 注释说明，查询任意 XID 状态的代码需要在测试 `oldestClogXid` 到完成 CLOG lookup 期间持有 `XactTruncationLock`。否则可能出现：

```text
查询者判断某 XID 可查；
truncation 删除 segment；
查询者再去读 pg_xact 文件；
读到不存在或错误 page。
```

本节不展开所有调用点。关键是知道 `oldestClogXid` 是“允许访问 CLOG 的边界”，不是 visibility 终态。


## 9. 成本、资源与跨模块传播

### 9.1 每 tuple visibility 的 CPU / cache 成本

heap scan 中大量 tuple 可能反复查询相同 XID。`transam.c` 的 single-item cache 只缓存最近一个稳定结果。

它对下面场景很有效：

```text
bulk insert 后扫描；
同一事务更新了大量 tuple；
page 上多个 tuple 共享同一个 xmin/xmax。
```

但它不是全局事务状态 cache。如果 workload 的 tuple XID 分布很散，查询会频繁进入 SLRU read-only 路径。

成本变量包括：

```text
tuple 数；
不同 XID 数；
CLOG page cache hit ratio；
SubXID / SUB_COMMITTED 追父链比例；
hint bit 是否已经写回 tuple header。
```

### 9.2 commit hot path 的 LWLock contention

提交时写 CLOG page 需要 SLRU bank lock exclusive。高并发短事务下，大量相邻 XID 往往落在同一个 CLOG page。

这会导致：

```text
多个 backend 在 transaction end 争同一个 XACT_SLRU bank lock；
group update 尝试把多个 status bit 写入合并到一个 leader；
follower wait event 可能表现为 XACT_GROUP_UPDATE。
```

成本随并发 commit backend 数扩张。但它不随 tuple 数直接扩张。

tuple 数主要影响读路径状态查询和 hint bit 机会。

### 9.3 SLRU buffer 与 IO 成本

`transaction_buffers` 控制 CLOG SLRU buffer 数。本基线中：

```text
transaction_buffers = 0 时按 shared_buffers 自动调优；
CLOGShmemBuffers() 使用 SimpleLruAutotuneBuffers(512, 1024)；
显式设置时限制在 16 到 CLOG_MAX_ALLOWED_BUFFERS 之间。
```

SLRU 成本表现为：

```text
pg_stat_slru 中 transaction 行的 blks_hit / blks_read / blks_written / flushes / truncates；
wait_event = SLRU_READ / SLRU_WRITE / SLRU_SYNC / SLRU_FLUSH_SYNC；
checkpoint 阶段 CheckPointCLOG() 写出 dirty pages；
CLOG page eviction 时可能触发 WAL flush。
```

如果 `pg_xact` 工作集超过 SLRU buffers，会看到更多 SLRU read。典型来源：

```text
长事务阻止 vacuum/freeze 推进；
大量历史 tuple 仍引用老 XID；
读路径需要频繁查很旧的 CLOG pages；
SubXID overflow 触发 pg_subtrans slow path，间接增加事务状态判断成本。
```

### 9.4 WAL 与 checkpoint 传播

CLOG 状态更新本身通常不为每个 commit 写 CLOG WAL record。传播路径是：

```text
commit/abort WAL record
  -> xact redo 重放时重新设置 CLOG bits
dirty CLOG page
  -> checkpoint / eviction 写 pg_xact file
async commit group_lsn
  -> CLOG page write 前强制 XLogFlush(max_lsn)
CLOG_ZEROPAGE / CLOG_TRUNCATE
  -> CLOG rmgr record
```

因此 CLOG 压力可能反过来影响 WAL flush latency：

```text
async commit 很多；
CLOG page 要被写出；
SlruPhysicalWritePage() 扫到 max_lsn；
前台或 checkpointer 路径触发 XLogFlush(max_lsn)。
```

这不是每个 async commit 都同步 flush。它是 CLOG page 落盘时的 deferred correctness cost。

### 9.5 与相邻模块的边界

| 模块 | 边界 |
| --- | --- |
| `ProcArray` | 负责 running / not running，不保存最终 commit bits。 |
| `pg_subtrans` | 负责 subxid 到 parent 的追溯，不决定父事务是否 committed。 |
| WAL / xlog | 负责 commit/abort record 的 crash recovery source of truth。 |
| heap visibility | 消费 running + CLOG + snapshot，不直接维护事务结果。 |
| VACUUM / freeze | 推进不再需要 CLOG 查询的 horizon，最终允许 truncation。 |
| checkpointer / sync.c | 写出和 fsync SLRU 文件，处理 queued sync request。 |
| hot standby | 用 KnownAssignedXids 模拟 primary running set，redo 后再过期。 |

涉及后台进程：

```text
checkpointer：checkpoint 时写出 CLOG dirty pages，处理 sync request。
walwriter：async commit LSN 推进后可异步 flush WAL。
startup process：recovery 中 redo xact 和 CLOG records。
autovacuum：通过 vacuum/freeze 推进 datfrozenxid，间接允许 CLOG truncation。
walsender / standby：复制和 Hot Standby 下影响可见性 horizon 与 KnownAssignedXids。
```


## 10. 观测与诊断入口

### 10.1 能直接观测的状态

`pg_stat_slru`：

```sql
SELECT name, blks_zeroed, blks_hit, blks_read, blks_written,
       blks_exists, flushes, truncates
FROM pg_stat_slru
WHERE name = 'transaction';
```

它能看到 CLOG SLRU 的累计行为。粒度是 instance-level 累计统计，不是单个事务。

可用于判断：

```text
CLOG 是否频繁 miss；
checkpoint / eviction 是否写了很多 transaction SLRU page；
是否发生 truncation；
zeroed 是否随着 XID 分配推进。
```

`pg_stat_activity` wait event：

```sql
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event IN ('XactGroupUpdate',
                     'SLRURead', 'SLRUWrite', 'SLRUSync', 'SLRUFlushSync');
```

实际显示名称随版本映射而定。源码 wait event 常量包括：

```text
WAIT_EVENT_XACT_GROUP_UPDATE
WAIT_EVENT_SLRU_READ
WAIT_EVENT_SLRU_WRITE
WAIT_EVENT_SLRU_SYNC
WAIT_EVENT_SLRU_FLUSH_SYNC
```

这些能说明 backend 正在等什么。它们不能单独证明根因。

例如 `SLRU_READ` 可能来自 transaction SLRU，也可能来自其它 SLRU，需要结合 `pg_stat_slru` 和 workload 判断。

### 10.2 文件系统层面的 `pg_xact`

可以观察目录：

```bash
ls -lh "$PGDATA/pg_xact"
```

这只能看到 segment 文件数量和大小。它看不到单个 XID 状态语义。

文件多通常说明：

```text
需要保留的 XID horizon 很老；
VACUUM / freeze 推进不足；
长事务、prepared transaction、replication slot、hot standby feedback 可能拖住 cleanup；
或者只是当前系统 XID 范围还没到下一次可截断机会。
```

不要把 `pg_xact` 文件数量直接解释成“有很多运行中事务”。它更多反映历史事务结果保留范围。

### 10.3 WAL 侧观察

可以用 `pg_waldump` 看事务 commit / abort record 和 CLOG record：

```bash
pg_waldump "$PGDATA/pg_wal/..." | rg 'XACT|CLOG'
```

预期能看到：

```text
XACT COMMIT / ABORT records；
CLOG ZEROPAGE；
CLOG TRUNCATE。
```

不要期待每个 commit 都有 CLOG WAL record。普通 commit 的 CLOG bit 更新由 xact redo 重新执行。

### 10.4 gdb / 断点观察

源码跟读时可以打断点：

```gdb
break TransactionIdSetTreeStatus
break TransactionIdSetStatusBit
break TransactionIdGetStatus
break TransactionIdIsInProgress
break ProcArrayEndTransaction
```

建议观察变量：

```text
xid
status
lsn
pageno
slotno
XactCtl->shared->page_dirty[slotno]
MyProc->xid
MyProc->subxidStatus.count
```

要验证提交顺序，可在单事务提交时观察：

```text
RecordTransactionCommit()
  -> TransactionIdCommitTree()
  -> ProcArrayEndTransaction()
```

在 `TransactionIdSetStatusBit()` 中可以打印：

```text
curval -> status
```

用子事务跨 page 重现很难，因为一个 CLOG page 可容纳大量 XID。实验中可先观察同 page 子事务，再通过源码临时调小 page 容量做教学验证。

### 10.5 能看到、只能推断、几乎不可见

| 类别 | 例子 |
| --- | --- |
| 能直接观测 | `pg_stat_slru` transaction 统计、wait event、`pg_xact` 文件数量、WAL record。 |
| 只能推断 | 某次 visibility check 是否命中 `cachedFetchXid`、某个 tuple 是否刚查过 CLOG 后写了 hint bit。 |
| 需要断点/patch | 单个 XID 两位状态、`group_lsn` 数组值、group update leader/follower 链。 |
| 几乎不可见 | 某个并发读者短暂看到 `SUB_COMMITTED` 后追父链的窗口，除非人为打断点放大。 |

真实诊断中，不要把 `pg_stat_slru` 当完整因果。它只是累计计数。

需要结合：

```text
事务提交速率；
tuple age 分布；
autovacuum/freeze 进度；
replication slot xmin；
long-running query；
checkpoint 周期；
存储 IO latency；
是否大量子事务。
```


## 11. 常见误区

### 误区 1：`TRANSACTION_STATUS_IN_PROGRESS` 等于事务正在运行

不对。它只是 CLOG 两位的 00 初始状态。

真正 running 要看 `TransactionIdIsInProgress()`。not running 且 not committed 的事务，对 visibility 来说就是未提交。

### 误区 2：`TransactionIdDidAbort()` 可以替代 `TransactionIdDidCommit()`

不对。源码注释明确说，崩溃隐式 abort 往往仍显示为 CLOG in-progress。

普通路径应先判 running，再问 did commit。`DidAbort()` 更适合需要识别显式 aborted 的路径。

### 误区 3：子事务 commit 以后就对外可见

不对。子事务 commit 只说明 subtransaction 自己成功结束。

对外可见性取决于 top-level transaction。`SUB_COMMITTED` 必须追父链。

### 误区 4：CLOG 是完整事务日志

不对。它是 commit status log，不是 WAL。

普通 commit / abort 的恢复来源是 xact WAL record。CLOG 只保存事务结果 bits，并由 redo 重建。

### 误区 5：async commit 是不写 CLOG

不对。async commit 也会把 CLOG 标 committed。

区别是传入 commit LSN，并在 CLOG page 写出前用 `group_lsn` 强制 WAL flush。

### 误区 6：`pg_xact` 文件变大说明当前事务很多

不准确。`pg_xact` 保留的是历史事务状态范围。

文件数量更多受 datfrozenxid、VACUUM/freeze、长事务、replication slot、prepared transaction 影响。当前 running transaction 数要看 ProcArray / `pg_stat_activity` / snapshots，而不是只看 `pg_xact` 目录。

### 误区 7：CLOG truncation 只是删文件

不对。它必须先确认有可删 segment，再推进 `oldestClogXid`，再写并 flush `CLOG_TRUNCATE` WAL record，最后让 SLRU 删除 segment。

这是 crash recovery 和并发查询边界的一部分。


## 12. 课堂实验

实验 1：观察 transaction SLRU。执行 `pg_stat_reset_slru('transaction')`，跑一批会分配 XID 的 INSERT，再 `CHECKPOINT`，比较 `pg_stat_slru where name = 'transaction'` 的 `blks_zeroed`、`blks_hit`、`blks_written`、`flushes`。

回源码看 `GetNewTransactionId() -> ExtendCLOG()`、`RecordTransactionCommit() -> TransactionIdCommitTree()`、`CheckPointCLOG() -> SimpleLruWriteAll()`。重点不是把某个统计值对应到单条 SQL，而是确认 `pg_xact` 是 SLRU-backed 状态。

实验 2：用两个会话观察 running 与 committed 的切换。会话 A `BEGIN; INSERT ...; SELECT txid_current();` 后暂停。

会话 B 查询目标表和 `pg_stat_activity.backend_xid`。会话 A `COMMIT` 后，会话 B 再查。

解释时把现象压回两步：提交前 running 由 ProcArray 证明；提交后结果由 CLOG 证明。实验 3：断点观察 bit 更新。

在 debug 环境断 `TransactionIdSetTreeStatus` 和 `TransactionIdSetStatusBit`，执行普通事务和带 savepoint 的事务。观察 top-level commit、sub-commit、跨 page 子事务三阶段的可能差异。

如果只看到简单路径，不说明复杂路径不存在，只说明当前 XID 分布没有触发它。


## 13. 讨论题

1. 如果 `ProcArrayEndTransaction()` 先于 `TransactionIdCommitTree()` 执行，并发 visibility check 可能看到什么错误窗口？
2. 为什么 `TRANSACTION_STATUS_IN_PROGRESS` 不能被命名理解为“backend 一定仍活着”？它的物理初始值属性带来了什么 fallback 语义？
3. 跨 CLOG page 的子事务提交为什么不能直接逐个 page 写 `COMMITTED`？`SUB_COMMITTED` 具体修补了哪个观察窗口？
4. 为什么 abort 路径可以不 flush abort WAL record 就写 CLOG aborted，而 commit 路径不能不管 WAL flush 就让 CLOG page 落盘？
5. `TransactionLogFetch()` 为什么只缓存 committed / aborted，而不缓存 in-progress / sub-committed？
6. 当 `pg_stat_slru` 显示 transaction SLRU read 很高时，可能有哪些 workload / horizon / cache 原因？哪些结论不能仅凭这个视图下？
7. `oldestClogXid` 是 visibility 状态、文件保留边界，还是并发查询保护边界？为什么 truncation 要先推进它？
8. group CLOG update 为什么是性能优化而不是正确性机制？如果它完全禁用，哪些行为应保持不变，哪些成本会变化？


## 14. 本节小结

本节唯一主问题是：

```text
pg_xact 只保存两位终态时，系统如何区分 running、committed、aborted 和 sub-committed？
```

核心链路：

```text
visibility 看到 tuple XID
  -> 先问 ProcArray 是否 running
  -> not running 后问 TransactionIdDidCommit()
  -> TransactionLogFetch()
  -> TransactionIdGetStatus()
  -> CLOG SLRU 取两位
  -> SUB_COMMITTED 时追 pg_subtrans 父链
```

写入链路：

```text
GetNewTransactionId()
  -> ExtendCLOG()
RecordTransactionCommit()
  -> WAL commit record
  -> WAL flush 或 async group_lsn
  -> TransactionIdCommitTree()
  -> ProcArrayEndTransaction()
RecordTransactionAbort()
  -> abort WAL record
  -> TransactionIdAbortTree()
  -> ProcArrayEndTransaction()
```

核心状态：

```text
ProcArray 表达 running；
pg_xact 表达 final / intermediate status bits；
pg_subtrans 表达子事务父链；
SLRU 表达 page cache、dirty、IO 和 group_lsn；
TransamVariables 表达 nextXid、latestCompletedXid、oldestClogXid 等全局边界。
```

正确性层次：

```text
running-state ordering 由 ProcArrayLock 和 end-transaction 顺序保证；
commit durability 由 WAL flush 和 group_lsn 保证；
CLOG page 并发访问由 SLRU bank lock 保证；
子事务跨 page 观察由 SUB_COMMITTED + pg_subtrans 保证；
旧状态删除由 freeze horizon、oldestClogXid、CLOG_TRUNCATE WAL 和 SLRU truncation 保证。
```

错误路径的核心规则：

```text
CLOG 00 不足以证明 running；
not running 且 not committed 才是 visibility 上的未提交；
abort record 丢失通常不会破坏 correctness；
commit record 与 CLOG committed 的发布顺序不能丢。
```

可观测入口：

```text
pg_stat_slru 的 transaction 行；
pg_stat_activity wait_event；
pg_xact 目录；
pg_waldump 中 XACT / CLOG records；
gdb 断点观察 TransactionIdSetStatusBit()。
```

可迁移 mental model：

```text
高频状态查询系统常把“当前活跃集合”和“历史终态存储”拆开。
当前活跃集合服务并发判断，历史终态存储服务重复查询和恢复。
正确性来自两者之间的发布顺序，而不是任一字段本身。
```

哪些判断仍然依赖上下文：

```text
SLRU miss 是否是瓶颈依赖 workload 的 XID 分布和 tuple age；
group update wait 是否显著依赖并发短事务提交速率；
pg_xact 文件保留大小依赖 vacuum/freeze、长事务、prepared transaction、replication slot；
async commit 的丢失窗口依赖 WAL flush 时机和 crash timing；
子事务 slow path 成本依赖 SubXID 数量和 overflow 情况。
```

下一节可以继续追问：

```text
为什么提交必须把 WAL、CLOG、ProcArray 和 checkpoint interlock 排成现在这个顺序？
```
