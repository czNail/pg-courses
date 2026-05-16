# PostgreSQL MemoryContext tree 与 backend-local ownership 边界

## 课程定位

前置知识：已经理解 PostgreSQL 是多进程架构，也已经知道 main shared memory 是跨 backend 的固定共享状态。本节开始转向另一类基础设施：每个 backend 自己的私有内存。

本节唯一主问题：

```text
为什么 PostgreSQL 把 backend-local 内存挂到 MemoryContext tree 上，而不是依赖调用者逐个 pfree()？
```

核心矛盾：PostgreSQL 的执行路径大量依赖可中断的 `ERROR` longjmp、复杂的模块回调和深层函数调用；但 C 语言本身没有自动析构、没有所有权类型，也不能要求每个调用者在所有正常和异常路径上都精确 `pfree()` 每个 chunk。

学完后应能判断：一块 backend-local 内存应该挂在哪个 context 下；什么时候应该显式 `pfree()`，什么时候应该依赖 parent context reset/delete；为什么 `TopMemoryContext` 接近 `malloc()`，通常不是“先放着以后再说”的安全位置。

本课基于本地 `~/postgres` 源码，分支 `master`，提交 `8eba2edb8010`。

## 1. 本节在总主线中的位置

前几节讨论的是 shared memory：一块跨进程可见的固定资源如何被 postmaster 创建、由各 backend attach，并通过名字稳定定位。

MemoryContext 解决的是另一侧问题：

```text
shared memory:
  进程之间共享，地址和生命周期受 postmaster / shmem allocator 管理

backend-local memory:
  每个 backend 私有，普通 C 指针只在本进程有效，生命周期由 MemoryContext tree 管理
```

这一节只讲第一层边界：MemoryContext tree 如何表达 ownership。下一节会把焦点缩到短生命周期 context 的 reset 边界，再往后才进入事务、Portal、ERROR cleanup 和 allocator 类型选择。

因此本节不试图讲完所有常见 context 名字，也不展开 `AllocSet`、`Generation`、`Slab`、`Bump` 的内部算法。这里先建立一个更重要的判断：

```text
chunk 的释放责任不应该散落在每个调用者手里；
它应该归属于一个能被系统统一 reset/delete 的生命周期节点。
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
palloc() 把 chunk 分配到 CurrentMemoryContext；
每个 context 挂在一个 parent 下形成 tree；
上层生命周期结束时 reset/delete parent，就批量释放它和子树中的 backend-local 内存。
```

这里的 tension 是：

```text
低调用成本、少传参、适配任意深层函数
  vs
在 ERROR longjmp 和复杂 ownership 下仍然可靠释放内存
```

如果只用 `malloc()` / `free()` 风格，代码会变成这样：

```text
调用者 A 分配 parse tree
  -> 调用 B
     -> B 分配临时 List
        -> 调用 C
           -> C 分配错误上下文字符串
           -> C ereport(ERROR)
```

一旦 `ERROR` 通过 longjmp 跳回顶层，A/B/C 的普通 C 栈清理逻辑不会逐层执行。每个函数都“记得 free 自己的东西”这个模型在 PostgreSQL 里不成立，因为很多路径根本不会按普通返回链路返回。

MemoryContext 把问题换成：

```text
这块内存应该活到哪个系统事件？

当前语句结束？
当前 query executor 结束？
当前 transaction 结束？
当前 portal 关闭？
当前 backend 退出？
```

只要 chunk 挂到了正确的 context，正常路径和 `ERROR` 路径都可以在生命周期边界统一释放。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/utils/palloc.h` | 定义 `palloc()`、`pfree()`、`CurrentMemoryContext`、`MemoryContextSwitchTo()`；这是大多数 backend 代码直接接触的接口。 |
| 2 | `src/include/utils/memutils.h` | 暴露标准顶层 context 全局变量和 context 操作函数，如 `TopMemoryContext`、`MemoryContextReset()`、`MemoryContextDelete()`。 |
| 3 | `src/include/nodes/memnodes.h` | 定义 `MemoryContextData`、parent/child 链、reset callback、allocator method table。 |
| 4 | `src/backend/utils/mmgr/mcxt.c` | 类型无关的 context tree 操作：初始化、reset、delete、set parent、callback、stats、signal logging。 |
| 5 | `src/backend/utils/mmgr/README` | 解释 PostgreSQL 选择 context tree 的设计理由，以及常见全局 context 的语义。 |
| 6 | `src/backend/utils/mmgr/aset.c` | `AllocSet` 实现；本节只需要知道它是最常见的 context implementation。 |
| 7 | `src/backend/tcop/postgres.c` | backend 主循环中 `MessageContext` 的创建、每条消息前 reset，以及顶层 `ERROR` 恢复。 |
| 8 | `src/backend/access/transam/xact.c` | `TopTransactionContext`、`CurTransactionContext` 的创建和事务结束 reset。 |
| 9 | `src/backend/utils/mmgr/portalmem.c` | Portal 自己的 context 如何挂在 backend-local tree 上。 |
| 10 | `src/backend/executor/execUtils.c` | `EState`、per-query context、`ExprContext` 如何挂在 executor 生命周期下。 |
| 11 | `src/backend/utils/adt/mcxtfuncs.c` | `pg_backend_memory_contexts` 和 `pg_log_backend_memory_contexts()` 的观测入口。 |

阅读顺序的关键不是“先背所有 context 名字”，而是一直追问：

```text
这个指针是跨进程的吗？
谁创建这个 context？
它的 parent 是谁？
上层生命周期结束时，谁 reset/delete 它？
ERROR 后能不能靠同一个边界释放？
```

## 4. 关键数据结构与状态

### `MemoryContextData`

`MemoryContext` 在公开 API 里是一个不透明指针：

```text
typedef struct MemoryContextData *MemoryContext;
```

真正的结构在 `memnodes.h` 中。理解本节只需要关注几组字段：

| 字段 | 语义 |
| --- | --- |
| `type` / `methods` | 当前 context 使用哪种 allocator 实现，例如 `AllocSet`、`Generation`、`Slab`、`Bump`。 |
| `parent` / `firstchild` / `prevchild` / `nextchild` | context tree 的 ownership 关系；delete parent 会处理整棵子树。 |
| `name` / `ident` | 诊断用名称；出现在 `pg_backend_memory_contexts` 和 memory context log 中。 |
| `reset_cbs` | reset/delete 前调用的 callback，用来处理“跟内存绑定但不是 palloc chunk”的资源。 |
| `isReset` / `mem_allocated` | reset 快路径和统计信息；不要把它们当成业务语义。 |

最重要的不变量是：

```text
MemoryContext 是 backend-local 对象；
普通 MemoryContext 指针不能跨 backend 传递；
parent/child 关系表达生命周期 ownership，不表达并发可见性。
```

这和 shared memory 正好相反。shared memory 里的对象可以被多个 backend 通过各自 attach 后访问，但不能保存普通 backend-local 指针作为跨进程协议。MemoryContext tree 里的指针则只属于当前进程。

### `CurrentMemoryContext`

`CurrentMemoryContext` 是 `palloc()` 的隐式目标：

```text
palloc(size)
  -> MemoryContextAlloc(CurrentMemoryContext, size)
```

它的存在不是为了偷懒，而是为了避免把 memory context 参数传遍整个 backend。例如 `copyObject()`、表达式执行函数、数据类型输入输出函数、parser/planner 中间函数，很多都需要临时分配 pass-by-reference 对象。如果每层都显式传 context，接口会被生命周期参数淹没。

但这个便利也带来一个危险：

```text
CurrentMemoryContext 指向哪里，决定了没有显式 context 参数的分配能活多久。
```

所以常见模式是：

```text
oldcxt = MemoryContextSwitchTo(target_context);
... palloc / makeNode / copyObject ...
MemoryContextSwitchTo(oldcxt);
```

如果忘记切回，后续无关分配就会被挂到错误生命周期上。这个 bug 不一定立刻 crash，更常见的是 retention、峰值内存异常，或在 context reset 后出现 dangling pointer。

### `TopMemoryContext`

`TopMemoryContext` 是整棵树的根。`mcxt.c` 的 `MemoryContextInit()` 创建它，并暂时把 `CurrentMemoryContext` 指向它。

语义上它接近：

```text
在当前 backend 生命周期内永不 reset/delete 的 malloc 区域
```

这不是说它不能用，而是说它的 ownership 边界非常长。适合放：

```text
backend 生命周期内真正 permanent 的状态
控制模块会自己 delete/reset 的顶层 owner
必须在 ERROR 恢复中仍可用的少量基础设施
```

不适合放：

```text
一条 SQL 的临时结果
一次 executor run 的工作内存
一个 transaction 内才需要的列表
“先放这里以后再清”的临时对象
```

`TopMemoryContext` 下的内存如果没有更短的子 context 接管，通常只能等 backend 退出才回收。对 session pooling、长连接、复杂函数和扩展来说，这就是实际意义上的 leak。

### 标准 context 名字只是入口，不是语义本身

`memutils.h` 暴露了一些全局 context：

```text
TopMemoryContext
ErrorContext
PostmasterContext
CacheMemoryContext
MessageContext
TopTransactionContext
CurTransactionContext
PortalContext
```

不要把这些名字当成清单背诵。它们的语义来自：

```text
创建位置
parent
reset/delete 时机
ERROR 路径是否还能访问
是否允许跨语句、跨事务或跨 portal 存活
```

例如 `CacheMemoryContext` 和 `TopMemoryContext` 都很长寿，但 `CacheMemoryContext` 用来让 relcache、catcache 等缓存状态在诊断上可识别，也方便缓存子对象形成更短的子 context。它不是因为物理上有另一种内存，而是因为 ownership 语义不同。

## 5. 主流程源码 walkthrough

本节主流程按一个 backend 从启动到执行一条 SQL 的时间线展开。

### 5.1 backend 启动：建立根 context

内存管理子系统从 `MemoryContextInit()` 开始：

```text
MemoryContextInit()
  -> AllocSetContextCreate(NULL, "TopMemoryContext", ...)
     -> TopMemoryContext 成为根 context
  -> CurrentMemoryContext = TopMemoryContext
  -> AllocSetContextCreate(TopMemoryContext, "ErrorContext", ...)
     -> ErrorContext 预留 8KB
     -> MemoryContextAllowInCriticalSection(ErrorContext, true)
```

这里有两个很有 PostgreSQL 风格的选择。

第一，`TopMemoryContext` 没有 parent。它是 backend-local context tree 的根。

第二，`ErrorContext` 在启动阶段就分配少量常驻内存。这是为了让 out-of-memory 本身也能通过 `ERROR` 路径报告，而不是因为报告错误还需要内存、结果再次 OOM。

这一步之后，backend 具备了最基本的不变量：

```text
任何 palloc chunk 都应该能追溯到某个 MemoryContext；
任何普通业务 context 都应该直接或间接挂在 TopMemoryContext 下；
ERROR 报告路径至少有 ErrorContext 可用。
```

### 5.2 backend 主循环：每条协议消息一个 `MessageContext`

在 `PostgresMain()` 中，backend 创建 `MessageContext`：

```text
MessageContext = AllocSetContextCreate(TopMemoryContext,
                                       "MessageContext",
                                       ALLOCSET_DEFAULT_SIZES);
```

随后进入主循环。每轮处理前：

```text
MemoryContextSwitchTo(MessageContext);
MemoryContextReset(MessageContext);
initStringInfo(&input_message);
```

这说明 `MessageContext` 的语义不是“当前事务”，也不是“当前 SQL plan”。它更靠近 FE/BE protocol 的一条 command message：

```text
上一条消息留下的 parse buffer / query string / 简单查询临时状态
  -> 下一轮主循环顶部 reset
```

如果某块内存必须跨越这个边界，例如 prepared statement 的 plan、portal 中的 queryDesc、事务提交前要保留的 pending state，就不能只放在 `MessageContext` 里。

### 5.3 事务开始：`TopTransactionContext` 与 `CurTransactionContext`

事务启动路径在 `xact.c` 的 `AtStart_Memory()`：

```text
AtStart_Memory()
  -> s->priorContext = CurrentMemoryContext
  -> 如有必要创建 TransactionAbortContext
  -> 如有必要创建 TopTransactionContext
  -> CurTransactionContext = TopTransactionContext
  -> MemoryContextSwitchTo(CurTransactionContext)
```

这里先只看 ownership：

```text
TopTransactionContext 是一个可复用的顶层事务 context；
CurTransactionContext 指向当前事务层级该使用的 context；
事务结束时 TopTransactionContext reset，子 context 一并消失。
```

子事务会创建新的 `CurTransactionContext`，并挂在父事务 context 下。提交子事务时，如果子 context 里有需要顶层 commit 使用的状态，它可能保留到顶层事务结束；回滚子事务时则会被丢弃。这个细节会在后续事务生命周期课里展开。

本节只抓住一点：

```text
MemoryContext tree 不是简单“按模块分目录”；
它表达的是时间上谁覆盖谁、谁结束时能带走谁。
```

### 5.4 Portal 与 Executor：把执行对象挂到可关闭的 owner 下

Portal 管理在 `portalmem.c`。创建 portal 时：

```text
CreatePortal()
  -> portal = MemoryContextAllocZero(TopPortalContext, sizeof *portal)
  -> portal->portalContext =
       AllocSetContextCreate(TopPortalContext, "PortalContext", ...)
  -> portal->resowner = ResourceOwnerCreate(CurTransactionResourceOwner, "Portal")
```

两个 owner 被同时创建：

```text
portalContext:
  管 portal 生命周期内的 backend-local memory

portal->resowner:
  管非内存资源，例如 pin、lock、snapshot 等
```

随后 executor 创建自己的 per-query context。`execUtils.c` 的 `CreateExecutorState()` 会创建 `ExecutorState` context，并把 `EState` 放进去：

```text
CreateExecutorState()
  -> qcontext = AllocSetContextCreate(CurrentMemoryContext,
                                      "ExecutorState",
                                      ALLOCSET_DEFAULT_SIZES)
  -> MemoryContextSwitchTo(qcontext)
  -> estate = makeNode(EState)
  -> estate->es_query_cxt = qcontext
```

释放时：

```text
FreeExecutorState(estate)
  -> FreeExprContext(...) 处理表达式 shutdown callbacks
  -> jit_release_context(...)
  -> DestroyPartitionDirectory(...)
  -> MemoryContextDelete(estate->es_query_cxt)
```

这里正好体现为什么不是逐个 `pfree()`：

```text
executor 初始化会创建 PlanState tree、ExprContext、JIT state、partition directory 等多层对象；
其中一些对象还会注册 shutdown callback；
ExecutorEnd / FreeExecutorState 只需要删除 per-query context，就能释放大量普通内存。
```

注意 `FreeExecutorState()` 注释也明确说：它不负责释放所有非内存资源。MemoryContext 能管理 palloc chunk 和 reset callback，不能替代 ResourceOwner、pin、lock、snapshot cleanup。

## 6. 生命周期 / ownership / cleanup

本节的 ownership 规则可以压缩成一张表：

| 生命周期问题 | 应放在哪里 | cleanup 边界 |
| --- | --- | --- |
| backend 启动后一直存在，直到进程退出 | `TopMemoryContext` 或其长期子 context | backend exit |
| cache entry / relcache 辅助状态 | `CacheMemoryContext` 或 cache entry 私有子 context | cache invalidation / backend exit |
| 一条客户端 command message 内有效 | `MessageContext` | 下一轮 `PostgresMain()` message loop 顶部 reset |
| 当前事务内有效 | `CurTransactionContext` / `TopTransactionContext` | commit / abort 时 reset 或子事务清理 |
| 当前 portal 有效 | `portal->portalContext` | `PortalDrop()` |
| 当前 executor run 有效 | `estate->es_query_cxt` | `ExecutorEnd()` / `FreeExecutorState()` |
| 表达式一次或若干 tuple cycle 内有效 | `ExprContext` 的 per-tuple context | plan node rescan / tuple cycle reset |

本节先看前五类的共同规律：

```text
创建者选择 parent；
parent 决定最长存活边界；
上层模块只需要 delete/reset 自己拥有的 context；
子树内的 chunk 不要求每个调用者逐个 pfree。
```

### `pfree()` 仍然存在，但不是默认 ownership 模型

`pfree(pointer)` 可以释放任意 context 里的 chunk，因为 chunk header 能找到 owning context；它不依赖 `CurrentMemoryContext`。

但在 PostgreSQL 内核里，显式 `pfree()` 通常服务这些场景：

```text
一个循环内的大 chunk 会造成明显峰值，不能等到 context reset
同一 context 内对象生命周期差异很大，需要提前还给 allocator
数据结构本身提供了明确的 remove/free 操作
```

它不适合作为默认资源策略，因为一旦代码经历 `ERROR` longjmp，普通函数返回路径上的 `pfree()` 不一定执行。

### reset 与 delete 的差异

`MemoryContextReset(context)`：

```text
删除所有子 context；
释放当前 context 中的 chunk；
保留 context 节点本身，供下一轮复用。
```

`MemoryContextDelete(context)`：

```text
自底向上删除整棵子树；
从 parent 链上 delink；
释放 context 节点和其所有内存。
```

所以 `MessageContext` 和 `TopTransactionContext` 常见做法是 reset：它们作为稳定容器反复使用。executor 的 `estate->es_query_cxt` 常见做法是 delete：一次 executor run 结束后整个 context 节点也没有复用价值。

### `MemoryContextSetParent()` 是生命周期升级，不是随手移动

`MemoryContextSetParent(context, new_parent)` 允许把一个 context 改挂到新 parent 下。典型用途是：

```text
先在短生命周期 context 下构建对象；
构建成功后再 reparent 到 CacheMemoryContext 等长期 context；
如果构建中 ERROR，短生命周期 parent 会自动兜底清理。
```

这个模式避免了“先放到长期 context，失败时手动清一半”的风险。它体现了一个非常实用的规律：

```text
对象在完全构造成功以前，应该先归属于失败时能自动清理的短生命周期 owner。
```

## 7. 正确性机制层次

MemoryContext 主要保证的是：

```text
当前 backend 内 palloc chunk 的生命周期和批量释放。
```

它不保证：

```text
跨 backend 可见性
shared memory 并发互斥
事务提交/回滚语义
buffer pin / relation refcount
文件描述符、DSM handle、lock、snapshot 等外部资源释放
```

因此在 PostgreSQL 里，正确性通常由多层机制组合：

| 机制 | 管什么 | 不要误解为 |
| --- | --- | --- |
| MemoryContext | backend-local palloc chunk 的生命周期 | 通用资源 owner |
| MemoryContext reset callback | 与 context 绑定的少量外部 cleanup hook | ResourceOwner 的替代品 |
| ResourceOwner | buffer pin、relation refcount、DSM segment、snapshot 等资源 | 普通 palloc chunk 的所有权树 |
| invalidation | cache 语义过期通知 | 内存立即释放 |
| lock / LWLock / pin | 并发访问和正在使用 | 生命周期自动清理 |
| ERROR longjmp | 跳回受保护的恢复点 | 自动执行 C 栈上每个函数的清理逻辑 |

这也是为什么 `FreeExecutorState()` 会先显式处理 ExprContext shutdown callback、JIT context、partition directory，再 delete per-query memory context。内存树能兜住大多数 palloc chunk，但非内存资源需要自己的机制。

## 8. 错误路径 / 异常路径 / fallback

MemoryContext 的价值在 `ERROR` 路径中最明显。

`PostgresMain()` 顶层使用 `sigsetjmp()` 建立恢复点。发生 `ereport(ERROR)` 后，控制流不是沿着普通调用栈逐层返回，而是跳回主循环的错误处理分支：

```text
ERROR longjmp
  -> PostgresMain() 顶层恢复点
     -> error_context_stack = NULL
     -> HOLD_INTERRUPTS()
     -> EmitErrorReport()
     -> debug_query_string = NULL
     -> AbortCurrentTransaction()
     -> PortalErrorCleanup()
     -> jit_reset_after_error()
     -> MemoryContextSwitchTo(MessageContext)
     -> FlushErrorState()
     -> 回到协议同步/ReadyForQuery 流程
```

如果内存释放依赖普通调用者逐个 `pfree()`，这条路径会跳过大量局部 cleanup。MemoryContext tree 把问题变成：

```text
事务 abort 会 reset/delete 事务相关 context；
PortalErrorCleanup 会关闭仍活跃的 portal；
下一轮 message loop 会 reset MessageContext；
ErrorContext 在错误报告完成后由 FlushErrorState 清理错误状态。
```

这不意味着任何错误后都不会有 leak。常见风险仍然存在：

```text
把短命对象误放进 TopMemoryContext 或 CacheMemoryContext
把外部 malloc 内存藏在 palloc 对象里却没有 reset callback
把子事务 context 中的对象指针挂到父事务 list，子事务 abort 后留下 dangling pointer
忘记在 ERROR-safe 边界释放 ResourceOwner 管的非内存资源
```

MemoryContext 解决的是“普通 palloc chunk 能否被生命周期边界兜住”。它不能修复错误的 owner 选择。

## 9. 成本、资源与跨模块传播

### 成本模型

MemoryContext 的成本来自三部分：

```text
每个 context 节点的元数据
allocator 内部 block/chunk 管理成本
reset/delete 时遍历子树和调用 callbacks 的成本
```

它省掉的是更大的系统成本：

```text
每个调用点维护复杂 free 列表
每条 ERROR 路径手写清理
每个函数接口都传递 context 参数
每个临时对象都做精确 free 的 CPU 和代码复杂度
```

PostgreSQL 的选择不是“不要 free”，而是：

```text
把大量小对象的释放，从 scattered pfree 转移到生命周期边界上的 batched reset/delete。
```

这特别适合 parser、planner、executor 这样的工作负载：对象很多、大小不一、生命周期高度相关，且中途可能被 ERROR 打断。

### 跨模块传播

MemoryContext ownership 往往沿调用链向下传播：

```text
PortalContext
  -> ExecutorState
     -> ExprContext
        -> 表达式求值临时对象
```

调用者通过 `MemoryContextSwitchTo()` 临时改变默认分配目标，然后调用大量不带 context 参数的通用函数。被调用者不需要知道完整系统生命周期，只需要遵守当前 context 的默认约定。

这就是 `CurrentMemoryContext` 的工程价值：它把“谁拥有这批对象”的信息放在动态执行上下文里，而不是塞进每个函数签名。

### 与 shared memory 的边界

本节对象都是 backend-local。它们可以保存普通 C 指针、函数指针、palloc chunk 地址，因为只在当前进程解释。

不能做的是：

```text
把 MemoryContext 指针写进 shared memory，期望另一个 backend 使用；
把 palloc chunk 地址作为跨进程 handle；
用 MemoryContext cleanup 管 shared memory 中对象的全局生命周期。
```

跨进程对象需要 shmem/DSM 的名字、offset、handle、refcount、pin、lock 等机制。MemoryContext tree 管的是本 backend 私有内存的 ownership。

## 10. 观测与诊断入口

### `pg_backend_memory_contexts`

当前 backend 的 context tree 可以通过 SQL 查看：

```sql
SELECT name, ident, type, level, total_bytes, free_bytes, used_bytes
FROM pg_backend_memory_contexts
ORDER BY total_bytes DESC
LIMIT 20;
```

这个视图来自 `pg_get_backend_memory_contexts()`，核心实现位于 `src/backend/utils/adt/mcxtfuncs.c`。它从 `TopMemoryContext` 做 breadth-first 遍历，并输出：

```text
name
ident
type
level
path
total_bytes
total_nblocks
free_bytes
free_chunks
used_bytes
```

诊断时要注意：

```text
total_bytes 是 allocator 从底层拿到的空间，不等于当前活对象大小；
free_bytes 可能只是 context 内部可复用空间，不一定已经还给 OS；
used_bytes 增长可能是 leak，也可能是合理缓存、prepared statement、portal 或 executor 峰值。
```

所以不要看到 `TopMemoryContext` 或 `CacheMemoryContext` 大就直接下结论。要结合 context 名字、parent path、SQL 行为和生命周期边界判断。

### `pg_log_backend_memory_contexts(pid)`

如果目标 backend 正在跑，当前会话无法直接读它的 `pg_backend_memory_contexts`。可以用：

```sql
SELECT pg_log_backend_memory_contexts(<pid>);
```

它通过 `PROCSIG_LOG_MEMORY_CONTEXT` 给目标进程发信号。目标进程在安全的 interrupt 检查点调用：

```text
HandleLogMemoryContextInterrupt()
  -> 设置 LogMemoryContextPending

ProcessLogMemoryContextInterrupt()
  -> MemoryContextStatsDetail(TopMemoryContext, ...)
  -> 写 server log
```

这个入口适合定位“某个活跃 backend 为什么占用很多内存”。它不是低成本指标采样接口，频繁调用会放大日志和遍历成本。

### gdb 断点

源码调试时常用断点：

```gdb
break MemoryContextCreate
break MemoryContextReset
break MemoryContextDelete
break MemoryContextSetParent
break AtStart_Memory
break AtCommit_Memory
break AtAbort_Memory
break CreateExecutorState
break FreeExecutorState
```

观察重点不是单个 chunk 地址，而是：

```text
新 context 的 parent 是谁
CurrentMemoryContext 切到了哪里
出错或结束时是否回到预期 context
reset/delete 是否覆盖了预期子树
```

## 11. 常见误区

误区一：`palloc()` 就是带错误处理的 `malloc()`。

更准确地说，`palloc()` 是“分配到当前 lifecycle owner 的 malloc”。真正重要的是 `CurrentMemoryContext` 当时指向哪里。

误区二：`pfree()` 少写就是 leak。

在 PostgreSQL 里，大量对象本来就不应该逐个 `pfree()`，而是跟随 context reset/delete。真正的 leak 是对象挂错了更长生命周期，或外部资源没有对应 cleanup。

误区三：`TopMemoryContext` 安全，因为总会在 backend 退出时释放。

长连接里 backend 退出可能很久以后才发生。把短命对象放到 `TopMemoryContext`，对用户可见就是 session 级内存增长。

误区四：MemoryContext 可以替代 ResourceOwner。

MemoryContext 管 palloc memory；ResourceOwner 管 pin、refcount、snapshot、DSM attachment 等资源。两者经常并行存在，例如 Portal 同时有 `portalContext` 和 `resowner`。

误区五：context 名字就是生命周期。

名字只是诊断标签。生命周期由 parent、reset/delete 时机、ERROR 路径共同决定。

## 12. 课堂实验

### 实验 1：查看当前 backend 的 context tree

在任意 psql 会话中执行：

```sql
SELECT name, ident, type, level, total_bytes, free_bytes, used_bytes
FROM pg_backend_memory_contexts
ORDER BY level, total_bytes DESC
LIMIT 40;
```

观察：

```text
TopMemoryContext 是否是 level 1
MessageContext、TopTransactionContext、CacheMemoryContext 等是否出现在树中
某些 context 是否有 ident，例如 unnamed portal 或 query text
```

再执行：

```sql
SELECT name, type, total_bytes, used_bytes
FROM pg_backend_memory_contexts
WHERE name IN ('TopMemoryContext', 'MessageContext', 'TopTransactionContext', 'ExecutorState', 'ExprContext')
ORDER BY name;
```

这一步的目标不是记住数值，而是建立直觉：这些 context 是当前 backend 的活状态，不是全局共享指标。

### 实验 2：让另一个 backend 打印 memory context log

会话 A：

```sql
SELECT pg_backend_pid();
```

会话 B：

```sql
SELECT pg_log_backend_memory_contexts(<会话A的pid>);
```

然后看 PostgreSQL server log，找到：

```text
logging memory contexts of PID ...
```

观察 log 中的树形缩进、context 名字和 used/total/free 数字。对比 `pg_backend_memory_contexts`，理解一个是当前 backend 的 SQL view，一个是给目标 backend 发信号后写 log。

### 实验 3：用 gdb 看 context parent

在 debug build 上启动 backend 后设置：

```gdb
break CreateExecutorState
break FreeExecutorState
break MemoryContextDelete
continue
```

执行一条普通 `SELECT`，在 `CreateExecutorState()` 停下后单步到 `qcontext` 创建完成，查看：

```gdb
print CurrentMemoryContext->name
print qcontext->name
print qcontext->parent->name
```

在 `FreeExecutorState()` 停下时观察将被释放的 executor context：

```gdb
print estate->es_query_cxt->name
```

在 `MemoryContextDelete()` 停下时观察通用 delete 入口收到的 context：

```gdb
print context->name
```

把这条链路和源码中的：

```text
ExecutorStart()
  -> CreateExecutorState()
  -> MemoryContextSwitchTo(estate->es_query_cxt)
  -> ExecInitNode(...)

ExecutorEnd()
  -> FreeExecutorState()
  -> MemoryContextDelete(estate->es_query_cxt)
```

对应起来。

## 13. 讨论题

1. 一个扩展在 `_PG_init()` 中把 per-query 临时数组分配到 `TopMemoryContext`，功能测试都通过。为什么这仍然是内核级 bug？

2. 如果一个函数创建临时 context，填充缓存对象，最后 `MemoryContextSetParent()` 到 `CacheMemoryContext`，这个模式相比一开始就分配在 `CacheMemoryContext` 有什么错误路径优势？

3. 为什么 `MemoryContextReset(MessageContext)` 不是在每条 SQL 结束的所有资源清理？哪些资源不能靠它处理？

4. `pg_backend_memory_contexts.used_bytes` 持续增长时，如何区分 leak、cache retention 和正常执行峰值？

5. 为什么普通 backend-local `MemoryContext` 指针不能写入 shared memory 给其它 backend 使用？

## 14. 本节小结

MemoryContext tree 的核心不是 allocator 技巧，而是 ownership 模型。

PostgreSQL 用 `CurrentMemoryContext` 降低深层调用的分配成本，用 parent/child tree 表达生命周期覆盖关系，用 reset/delete 在语句、事务、portal、executor 等边界批量释放 backend-local 内存。这样即使 `ERROR` 通过 longjmp 跳过普通返回路径，系统仍然有机会在上层生命周期边界收口。

本节可迁移规律是：

```text
当系统存在大量短命对象、复杂调用链和非局部异常跳转时，
逐对象 free 不是可靠的 ownership 模型；
把对象挂到可验证的生命周期 owner 上，才是可维护的 cleanup 边界。
```

下一节会沿着这个模型继续缩小范围：为什么 executor 和表达式求值需要短生命周期 context，以及 reset 边界如何决定指针能不能跨 tuple、跨 expression、跨 query 保存。
