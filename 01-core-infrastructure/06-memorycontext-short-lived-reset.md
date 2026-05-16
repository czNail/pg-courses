# PostgreSQL 短生命周期上下文与 reset 边界

## 课程定位

前置知识：已经理解上一节的 MemoryContext tree 是 backend-local 内存 ownership 的表达方式，知道 `palloc()` 默认分配到 `CurrentMemoryContext`，也知道 context 可以通过 reset/delete 批量释放。

本节唯一主问题：

```text
一条 SQL 执行过程中，per-query、per-tuple、per-expression 等短生命周期内存为什么主要靠 reset 批量回收，哪些指针不能跨过这个边界？
```

核心矛盾：executor 每处理一行 tuple 都可能执行大量表达式、函数、投影、qual 判断和数据类型转换；这些临时分配的数量随 tuple 数线性增长。如果逐个 `pfree()`，hot path 会被释放逻辑和 ownership 传递淹没；但如果不及时释放，长扫描、join、聚合和 DML 会在一次 query 内把 backend-local 内存撑爆。

学完后应能判断：某个 `Datum`、`char *`、`text *`、`TupleTableSlot` 中的指针能不能保存到下一行、下一次 rescan、query 结束之后；什么时候应该依赖 `ResetExprContext()`，什么时候必须 `datumCopy()`、`ExecMaterializeSlot()` 或切到更长生命周期 context 再分配。

本课基于本地 `~/postgres` 源码，分支 `master`，提交 `8eba2edb8010`。

## 1. 本节在总主线中的位置

上一节建立了 MemoryContext tree 的 ownership 模型：

```text
chunk 不应该只由调用者逐个 pfree；
chunk 应该挂到一个能被统一 reset/delete 的生命周期节点。
```

这一节把焦点缩到 executor hot path。这里的生命周期不是 backend 或 transaction，而是更短的层次：

```text
一次 Executor invocation:
  EState / per-query context
    -> PlanState、ExprState、TupleTableSlot、ResultRelInfo 等 query 级工作状态
    -> 每个 ExprContext 自己的 per-tuple memory context
       -> 表达式求值中的临时 varlena、cstring、函数返回值、比较 key 等
```

本节不展开 transaction context、portal context 和 ERROR cleanup 的完整故事。我们只跟一条 SQL 在 executor 中如何一行一行推进，并解释为什么 PostgreSQL 在这个层次选择：

```text
对短生命周期临时结果做 reset；
对需要跨边界的结果显式复制或转移 ownership。
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ExecutorState 是一次 executor run 的 per-query owner；
ExprContext 保存当前 tuple、参数和表达式求值环境；
表达式求值前切到 ecxt_per_tuple_memory；
每个 tuple cycle 开始或失败重试时 ResetExprContext() 批量丢弃上一轮临时结果。
```

这里的 tension 是：

```text
每行都可能产生很多短命 by-reference 结果
  vs
executor hot path 不能为每个临时结果维护精确 free 链
```

如果不 reset，下面这样的语句会持续积累临时字符串：

```sql
SELECT count(*)
FROM generate_series(1, 10000000) g
WHERE length(md5(g::text) || md5((g + 1)::text)) > 0;
```

每一行都会产生若干中间结果。真正需要活到下一层的只是 `WHERE` 判断的布尔结果，或者投影后被目标 slot 正确持有的结果。绝大多数中间 `text *`、`cstring`、函数调用 scratch space 不应该活过当前 tuple。

如果逐个 `pfree()`，表达式执行器、数据类型函数、操作符函数、extension 函数之间要传递大量 ownership 规则：

```text
这个函数返回的 varlena 是 caller free 吗？
这个 cast 的中间 cstring 谁释放？
短路表达式未走到的分支如何释放？
ERROR longjmp 后谁释放？
```

PostgreSQL 把这些问题压缩成更稳定的边界：

```text
在 per-tuple context 中产生的临时结果，只保证活到下一次 ResetExprContext()。
想跨过这个边界，必须复制到更长生命周期或放进有明确 ownership 的容器。
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/nodes/execnodes.h` | 定义 `EState`、`ExprContext`、`PlanState` 等 executor 状态；重点看 `ExprContext` 的两个 memory context 字段。 |
| 2 | `src/include/executor/executor.h` | 定义 `ResetExprContext()`、`GetPerTupleExprContext()`、`ResetPerTupleExprContext()`、`ExecEvalExprSwitchContext()`、`ExecQualAndReset()` 等热路径接口。 |
| 3 | `src/backend/executor/execUtils.c` | 创建和释放 `EState`、`ExprContext`；这里能看到 per-query context 与 per-tuple context 的 parent/child 关系。 |
| 4 | `src/backend/executor/execMain.c` | `ExecutorFinish()` / `ExecutorEnd()` 如何在 per-query context 中结束 plan，并最终 `FreeExecutorState()`。 |
| 5 | `src/include/executor/execScan.h` | scan node 的核心 tuple 循环；这是最适合观察 per-tuple reset 的入口之一。 |
| 6 | `src/backend/executor/execExpr.c` / `execExprInterp.c` | 表达式初始化状态放在 per-query，表达式运行时结果经 `ExprContext` 求值。 |
| 7 | `src/include/executor/tuptable.h` / `src/backend/executor/execTuples.c` | `TupleTableSlot` 如何保存 tuple 值、何时 owned、何时只是引用；理解“slot 活得久”和“slot 里的 by-ref 指针安全”不是同一件事。 |
| 8 | `src/backend/executor/nodeAgg.c`、`nodeHashjoin.c`、`nodeIndexscan.c` | 更复杂节点如何额外创建工作用 `ExprContext`，并在 group、probe、runtime key 等边界 reset。 |

阅读顺序的关键是一直追问：

```text
这个对象是 query 级状态，还是 tuple 级临时结果？
这个指针指向的内存在哪个 context 中？
下一次 ResetExprContext() 后还有效吗？
如果需要保存，是 copy 了数据，还是只 copy 了指针？
```

## 4. 关键数据结构与状态

### `EState`：一次 executor run 的 per-query owner

`CreateExecutorState()` 在 `execUtils.c` 中创建一个名为 `"ExecutorState"` 的 per-query context：

```text
qcontext = AllocSetContextCreate(CurrentMemoryContext,
                                 "ExecutorState",
                                 ALLOCSET_DEFAULT_SIZES);
```

随后切到这个 context 中创建 `EState` 本身：

```text
oldcontext = MemoryContextSwitchTo(qcontext);
estate = makeNode(EState);
estate->es_query_cxt = qcontext;
```

这说明：

```text
EState 本身、PlanState、ExprState、tuple table、projection info 等 executor 工作状态，
通常活到本次 executor run 结束。
```

`FreeExecutorState()` 最后会：

```text
MemoryContextDelete(estate->es_query_cxt);
```

所以 per-query context 是一次 executor invocation 的 owner，不是一行 tuple 的临时垃圾桶。

### `ExprContext`：表达式求值时的当前世界

`ExprContext` 在 `execnodes.h` 中有两类状态：

| 字段 | 语义 |
| --- | --- |
| `ecxt_scantuple` / `ecxt_innertuple` / `ecxt_outertuple` | 当前表达式里的 `Var` 应该从哪个 slot 取值。 |
| `ecxt_param_exec_vals` / `ecxt_param_list_info` | 参数求值入口。 |
| `ecxt_aggvalues` / `ecxt_aggnulls` | 聚合或窗口函数预计算值。 |
| `ecxt_per_query_memory` | query 生命周期内可用的 context，通常就是 `estate->es_query_cxt`。 |
| `ecxt_per_tuple_memory` | 表达式求值的短期 context，通常每个 `ExprContext` 自己一个。 |
| `ecxt_callbacks` | shutdown/rescan 时需要触发的回调，例如取消未完成的 SRF 状态。 |

源码注释给出了最重要的边界：

```text
ecxt_per_query_memory:
  query-lifespan context，可放函数调用 cache 等 query 级状态

ecxt_per_tuple_memory:
  expression results 的短期 context，通常每个 tuple reset 一次
```

注意命名上的一个容易误解点：

```text
per-expression 并不一定表示“每个表达式一个 MemoryContext”。
```

在 executor 中，表达式状态 `ExprState` 通常是 per-query 的；表达式运行时临时结果通常进入所属 `ExprContext` 的 per-tuple context。也就是说，“per-expression 临时内存”更准确地说是：

```text
表达式求值过程中产生、依附于当前 ExprContext tuple cycle 的临时结果。
```

### `ecxt_per_tuple_memory`：reset 边界本身

`executor.h` 把 reset 边界压成一个宏：

```text
#define ResetExprContext(econtext) \
    MemoryContextReset((econtext)->ecxt_per_tuple_memory)
```

这不是语法糖，而是 executor 的核心性能边界。它意味着：

```text
不追踪本轮表达式求值中分配了哪些 chunk；
只在 tuple cycle 边界把整个短期 context 归零。
```

### `TupleTableSlot`：比 per-tuple context 更微妙

`TupleTableSlot` 通常是 per-query 状态。它的 `tts_values` / `tts_isnull` 数组也通常随 slot 生命周期存在。

但这不等于数组中的所有 by-reference `Datum` 都能长期保存：

```text
slot 数组本身活得久；
slot 数组里的指针可能指向：
  heap tuple / minimal tuple / buffer-backed tuple
  slot 拥有的 tuple
  per-tuple context 里的表达式结果
  其它更长生命周期对象
```

因此判断指针能不能跨 reset，不能只看它在不在 `TupleTableSlot` 里，而要看它指向的数据由谁拥有。

`tuptable.h` 里也能看到类似语义：slot 可以 own 物理 tuple，也可以只是引用；`TTS_FLAG_SHOULDFREE` 只说明 slot 是否应在清空时释放它拥有的 tuple，不说明任意 `Datum` 指针都被深拷贝了。

## 5. 主流程源码 walkthrough

这一节用一个普通 scan + qual + projection 的执行周期做主线。

### 5.1 创建 per-query owner

Executor 初始化时会创建 `EState`：

```text
CreateExecutorState()
  -> AllocSetContextCreate(CurrentMemoryContext, "ExecutorState", ...)
  -> MemoryContextSwitchTo(qcontext)
  -> makeNode(EState)
  -> estate->es_query_cxt = qcontext
```

状态变化是：

```text
当前 executor run 有了一个 per-query owner；
之后大部分 plan/expression/slot 工作状态都挂在这个 owner 下。
```

这个 context 的 parent 是调用者当时的 `CurrentMemoryContext`。在普通 SQL 执行中，它会被更外层 query/portal 生命周期包住；但从 executor 角度看，`estate->es_query_cxt` 就是本次执行的工作内存根。

### 5.2 创建 ExprContext 和 per-tuple context

当 plan node 需要 qual 或 projection 时，会创建 `ExprContext`。例如 `ExecAssignExprContext()`：

```text
ExecAssignExprContext(estate, planstate)
  -> planstate->ps_ExprContext = CreateExprContext(estate)
```

`CreateExprContext()` 进入 `CreateExprContextInternal()`：

```text
MemoryContextSwitchTo(estate->es_query_cxt)
  -> econtext = makeNode(ExprContext)
  -> econtext->ecxt_per_query_memory = estate->es_query_cxt
  -> econtext->ecxt_per_tuple_memory =
       AllocSetContextCreate(estate->es_query_cxt, "ExprContext", ...)
  -> estate->es_exprcontexts = lcons(econtext, estate->es_exprcontexts)
```

状态变化是：

```text
ExprContext 节点本身属于 per-query context；
它内部又有一个子 context，专门承接本 ExprContext 的 per-tuple 临时结果；
EState 记录所有 ExprContext，方便 query shutdown 时显式触发 callback。
```

这里的 child 关系很重要：

```text
ExecutorState
  -> ExprContext
  -> ExprContext
  -> ExprContext
```

每个 `"ExprContext"` 是一个可被独立 reset 的短期 context。它们又挂在 per-query context 下，所以 query 结束时即使没有单独 reset，也会被 `MemoryContextDelete(estate->es_query_cxt)` 整体带走。

### 5.3 每轮 tuple cycle 前 reset

`execScan.h` 的 `ExecScanExtended()` 是最清楚的入口之一。它在取下一行之前先做：

```text
ResetExprContext(econtext);
```

源码注释说明这个 reset 是为了释放上一轮 tuple cycle 的 expression evaluation storage。

随后循环取 tuple：

```text
slot = ExecScanFetch(...)
econtext->ecxt_scantuple = slot
if (qual == NULL || ExecQual(qual, econtext))
    return ExecProject(projInfo) 或返回 scan slot
else
    ResetExprContext(econtext)
```

状态随时间推进如下：

```text
tuple N-1 的临时表达式结果
  -> ResetExprContext() 后全部失效

fetch tuple N
  -> econtext->ecxt_scantuple 指向当前 scan slot
  -> ExecQual() 在 per-tuple context 中求值
  -> ExecProject() 形成输出 slot

tuple N 被过滤掉
  -> 立即 ResetExprContext()
  -> 继续 fetch 下一个 tuple
```

这里可以看到 reset 的两个位置：

```text
每个 tuple cycle 开始前 reset:
  清掉上一轮临时结果

tuple 被 qual 过滤后 reset:
  清掉失败尝试中产生的临时结果，避免在同一次 ExecScan 调用内部累积
```

### 5.4 表达式求值时切到 per-tuple context

`executor.h` 中的 `ExecEvalExprSwitchContext()` 显式切换到 `ecxt_per_tuple_memory`：

```text
oldContext = MemoryContextSwitchTo(econtext->ecxt_per_tuple_memory);
retDatum = state->evalfunc(state, econtext, isNull);
MemoryContextSwitchTo(oldContext);
```

`ExecQual()` 会调用它：

```text
ret = ExecEvalExprSwitchContext(state, econtext, &isnull);
```

这意味着表达式求值过程中没有显式 context 参数的 `palloc()`，默认都会进入本轮 tuple 的短期 context。

典型临时对象包括：

```text
函数返回的 by-reference Datum
数据类型输入输出函数生成的 cstring / varlena
表达式拼接、cast、比较时的中间结果
runtime key evaluation 的临时数组或值
投影过程中暂时可见的表达式结果
```

这就是为什么很多函数不需要知道自己被谁调用，也不需要手工 free 每个中间结果。它们只要遵守：

```text
需要短命结果时，分配在当前 per-tuple context；
需要长命状态时，调用者必须切到更长 context 或复制。
```

### 5.5 query 结束时 delete per-query owner

`standard_ExecutorEnd()` 在结束 plan 时会先切到 per-query context：

```text
oldcontext = MemoryContextSwitchTo(estate->es_query_cxt);
ExecEndPlan(queryDesc->planstate, estate);
UnregisterSnapshot(...);
MemoryContextSwitchTo(oldcontext);
FreeExecutorState(estate);
```

`FreeExecutorState()` 先处理仍然存在的 `ExprContext`：

```text
while (estate->es_exprcontexts)
    FreeExprContext(..., true);
```

这样做不是为了释放普通 palloc chunk，因为 delete per-query context 最终也能释放它们；这里更重要的是：

```text
触发 ExprContext shutdown callbacks，
释放那些不是普通 palloc memory 的状态。
```

随后：

```text
MemoryContextDelete(estate->es_query_cxt);
```

一次 executor run 的全部工作内存到这里结束。

## 6. 生命周期 / ownership / cleanup

### 核心生命周期表

| 层次 | 典型 owner | 典型内容 | 释放边界 | 指针能否跨过 |
| --- | --- | --- | --- | --- |
| per-tuple | `ExprContext.ecxt_per_tuple_memory` | 表达式中间结果、函数返回 by-ref 临时值、比较 key、cast scratch | `ResetExprContext()` / `ReScanExprContext()` / `FreeExprContext()` | 默认不能 |
| per-expression runtime | 所属 `ExprContext` 的 per-tuple context | 某次表达式求值中产生的临时对象 | 同 per-tuple | 默认不能 |
| per-query | `estate->es_query_cxt` | `EState`、`PlanState`、`ExprState`、slot、projection info、函数 cache | `FreeExecutorState()` | 可跨 tuple，不能跨 query end |
| per-rescan callback state | `ExprContext.ecxt_callbacks` 指向的状态 | SRF、聚合等需要 shutdown/rescan 的非纯内存状态 | `ReScanExprContext()` / `FreeExprContext()` | 取决于 callback 语义 |
| portal / transaction / cache | 更外层 context | cursor、transaction-local 状态、cache | portal/transaction/backend 边界 | 只能按对应生命周期判断 |

### `pfree`、`reset`、`delete` 在这里的分工

`pfree()` 适合：

```text
少量明确 ownership 的对象；
提前释放能显著降低峰值；
对象生命周期不等于某个现成 context 边界。
```

`MemoryContextReset()` 适合：

```text
context 本身要继续复用；
里面的所有 chunk 都是同一短生命周期；
hot path 上每轮都要清掉大量小对象。
```

`MemoryContextDelete()` 适合：

```text
生命周期 owner 本身结束；
不再需要这个 context 节点；
包括它的子 context 也都应该消失。
```

本节主线中的对应关系是：

```text
per-tuple context:
  重复 reset，保留 context 对象本身，避免每行重新创建 context

per-query context:
  query end delete，连 EState 自己一起释放
```

### 哪些指针不能跨过 reset 边界

默认不能跨过 `ResetExprContext(econtext)` 的有：

```text
在 econtext->ecxt_per_tuple_memory 中 palloc 的任何指针
by-reference Datum，如果它指向 per-tuple context 中的 varlena / expanded value / cstring
表达式函数返回的临时 pass-by-reference 结果
投影或 qual 中临时构造但没有被 materialize/copy 的值
extension 函数在当前 context 中分配并返回、随后又被调用者保存的指针
```

容易误判的有：

```text
Datum 本身只是一个机器字；
pass-by-value 类型可以直接复制 Datum；
pass-by-reference 类型复制 Datum 只是在复制指针。
```

所以这个判断是错误的：

```text
我把 Datum 保存到数组里了，所以它安全。
```

正确问题应该是：

```text
这个 Datum 是 by-value 还是 by-reference？
如果 by-reference，它指向的数据在哪个 context 或哪个 tuple/buffer 中？
下一次 reset/clear/release 后还存在吗？
```

### 跨边界保存的常见做法

如果结果需要跨过 per-tuple reset，常见做法是：

```text
切到 per-query 或更长 context 后重新 palloc
用 datumCopy() 深拷贝 pass-by-reference Datum
把 slot materialize，让 slot 或 tuplestore 拥有 tuple
把结果插入 tuplestore、hash table、sort state、aggregate transition state 等有明确 owner 的结构
用 MemoryContextSwitchTo() 明确改变后续 palloc 的归属
```

但每一种做法都要回答同一个问题：

```text
新的 owner 是谁？
它什么时候释放？
ERROR 或 rescan 时 callback / ResourceOwner 是否也需要参与？
```

例如 Hash Join 或 Agg 不是简单地把 per-tuple 指针存进 hash table。真正要保存到 hash table 或 aggregate state 的数据，必须归入 hash/agg 的工作 context 或 transition context，否则下一轮 tuple reset 后就会悬空。

## 7. 正确性机制层次

短生命周期 reset 不只是性能优化，也是一条正确性边界。

| 机制 | 解决的问题 | 本节中的体现 |
| --- | --- | --- |
| `CurrentMemoryContext` | 让深层函数无需传 context 参数也能分配到当前生命周期 | `ExecEvalExprSwitchContext()` 在表达式求值前切到 per-tuple context。 |
| `MemoryContextReset()` | 批量释放同生命周期对象 | `ResetExprContext()` 每轮 tuple cycle 清空临时表达式结果。 |
| `MemoryContextDelete()` | owner 结束时释放整棵子树 | `FreeExecutorState()` 删除 `estate->es_query_cxt`。 |
| `ExprContext` callback | 处理非纯 palloc memory 或需要 rescan/shutdown 的状态 | `FreeExprContext()` / `ReScanExprContext()` 调用 shutdown callbacks。 |
| `TupleTableSlot` ownership flag | 区分 slot 是否拥有物理 tuple | `TTS_FLAG_SHOULDFREE` 影响 `ExecClearTuple()`，但不替代 memory context 边界。 |
| ResourceOwner / snapshot cleanup | 管理 buffer pin、relation、snapshot 等非内存资源 | `FreeExecutorState()` 注释明确它不负责释放所有非内存资源；完整 plan cleanup 还依赖 executor end 路径。 |

一个关键分层是：

```text
MemoryContext 管 palloc chunk；
ExprContext callback 管表达式生命周期相关的额外状态；
ResourceOwner 管锁、pin、snapshot、文件等非普通内存资源；
TupleTableSlot 管 tuple 引用或 owned tuple 的清理。
```

不要把 `ResetExprContext()` 理解成“当前行所有资源都没了”。它只保证 per-tuple memory context 里的 palloc chunk 被释放。buffer pin、snapshot、relation ref、slot owned tuple 等有自己的 owner 和 cleanup 路径。

## 8. 错误路径 / 异常路径 / fallback

### ERROR 时为什么 reset 仍然有价值

在 PostgreSQL 中，`ERROR` 可能通过 longjmp 跳出普通 C 调用栈。表达式执行中如果某个函数 `ereport(ERROR)`，它不会沿着每层 C 函数正常返回并逐个释放局部对象。

MemoryContext 的价值在于：

```text
本轮 tuple 中分配的普通内存仍然挂在 ecxt_per_tuple_memory 下；
executor 或更外层 ERROR cleanup 结束相关生命周期时，可以统一释放。
```

但这并不表示每个 callback 都会像 commit 路径一样运行。`FreeExprContext()` 的注释区分了 `isCommit`：

```text
isCommit 为 false 时表示 error cleanup，
不应调用某些 shutdown callback，只释放内存。
```

这说明短生命周期内存 cleanup 和非内存资源 cleanup 是分层处理的。不能把“context reset 会释放内存”推广成“所有状态都能安全恢复”。

### `ReScanExprContext()` 为什么不是简单 reset

`ReScanExprContext()` 做两件事：

```text
ShutdownExprContext(econtext, true)
MemoryContextReset(econtext->ecxt_per_tuple_memory)
```

它用于 plan node rescan。rescan 不只是“清空临时内存”，还可能需要取消正在进行的 set-returning function、清理表达式 callback 挂住的状态。

因此：

```text
普通 tuple cycle:
  ResetExprContext()

rescan / shutdown:
  ReScanExprContext() 或 FreeExprContext()
```

如果在需要 shutdown callback 的场景只调用 `MemoryContextReset()`，就可能释放了内存却漏掉了状态机清理。

### fallback：少量显式 `pfree` 仍然存在

PostgreSQL 并不是完全不用 `pfree()`。例如某些输出函数或局部转换会在明确知道对象 ownership 且提前释放有收益时直接 `pfree()`。

但这不是主模型。主模型仍然是：

```text
大批同生命周期临时结果靠 reset；
少量特殊对象按明确 ownership 显式释放；
需要跨边界的结果深拷贝或转移 owner。
```

## 9. 成本、资源与跨模块传播

### hot path 成本随 tuple 数扩张

假设一条 SQL 扫描 `N` 行，每行表达式产生 `K` 个短期分配。

逐个 `pfree()` 的成本近似是：

```text
O(N * K) 次分配追踪 + O(N * K) 次释放调用 + 大量 ownership 分支
```

per-tuple reset 的模型是：

```text
O(N * K) 次分配
O(N) 次 reset
```

更重要的是，reset 降低了接口复杂度。大多数深层函数只需要正常 `palloc()` 返回结果，不需要把“谁释放我返回的临时对象”写进每个调用协议。

### reset 不是零成本

`MemoryContextReset()` 也有成本：

```text
它要遍历 context 中的分配块；
可能保留或释放 allocator block；
会影响 cache locality；
频繁创建/删除 context 比重复 reset 更贵。
```

所以 executor 选择的是：

```text
每个 ExprContext 创建一次 per-tuple context；
每轮 tuple cycle 重复 reset；
query 结束时 delete。
```

这比“每行创建一个 context 再 delete”更适合 hot path。

### 和表达式执行器的边界

`ExecEvalExprSwitchContext()` 把表达式执行器和内存生命周期绑定起来：

```text
表达式实现不需要知道自己处于 scan、join、agg 还是 projection；
只要它通过 palloc 分配临时 by-ref 结果，就自然进入当前 econtext 的 per-tuple context。
```

这使得 executor 可以在多个模块之间复用表达式求值：

```text
SeqScan qual
IndexScan runtime key
Join qual
Projection
Constraint checking
Partition pruning
Trigger / DML RETURNING 相关表达式
ANALYZE / statistics expression evaluation
```

但代价是调用者必须明确：

```text
表达式返回值是否只是本轮 tuple 内可用？
如果要保存到外部结构，是否已经转移到正确 context？
```

### 和聚合、hash、sort 的边界

Agg、Hash Join、Memoize、Sort 等节点常常需要保存跨 tuple 的状态。它们不能把 per-tuple 指针直接长期保存。

典型模式是：

```text
在 per-tuple context 中计算输入表达式
  -> 如果结果只是比较/过滤，reset 后丢弃
  -> 如果结果要进入 hash table / sort / transition state，复制到对应工作 context
```

因此性能问题经常出现在两个方向：

```text
复制太少:
  保存了悬空指针，结果错误或 crash

复制太多:
  把本可短命的对象放进 per-query / hash / agg context，导致峰值内存上升
```

### 和 extension 的边界

扩展函数如果在表达式求值期间使用 `palloc()`，默认分配目标通常就是 per-tuple context。

这带来一个清晰规则：

```text
返回给 SQL 表达式立即消费的临时 by-ref 值，可以在当前 context 分配；
保存到 fn_extra、全局缓存、SPI 后续调用、tuplestore、hash table 的值，必须复制到对应长生命周期 context。
```

扩展代码中常见 bug 是：

```text
把当前调用返回的 text * 或 char * 缓存在 fn_extra；
下一行 ResetExprContext() 后继续使用这个指针。
```

正确做法通常是把 cache 分配到 `fcinfo->flinfo->fn_mcxt` 或其它明确的长生命周期 context，而不是当前 per-tuple context。

## 10. 观测与诊断入口

### SQL 观察：`pg_backend_memory_contexts`

当前 backend 可以查询：

```sql
SELECT name, level, total_bytes, used_bytes
FROM pg_backend_memory_contexts
WHERE name IN ('ExecutorState', 'ExprContext')
ORDER BY level, name;
```

注意这个视图是采样式观察。per-tuple context 可能在每轮 tuple cycle 被快速 reset，所以你经常只能看到：

```text
ExecutorState 存在；
ExprContext 存在；
ExprContext 的 used_bytes 不随扫描行数线性增长。
```

这恰好是本节要解释的现象：大量临时结果没有被长期保留。

### 从另一个 session 打日志

如果一个 query 运行时间足够长，可以从另一个 session 执行：

```sql
SELECT pg_log_backend_memory_contexts(<pid>);
```

日志中通常能看到 `ExecutorState` 和若干 `ExprContext`。如果某个查询的 per-query context 或某个工作 context 持续增长，要判断：

```text
这是合理保存跨 tuple 状态，还是本应 per-tuple 的数据被放错 context？
```

### gdb 观察 reset 边界

可以在调试 backend 上设置断点：

```gdb
break CreateExecutorState
break CreateExprContextInternal
break ExecEvalExprSwitchContext
break MemoryContextReset
break FreeExecutorState
```

建议观察：

```gdb
print qcontext->name
print econtext->ecxt_per_query_memory->name
print econtext->ecxt_per_tuple_memory->name
print CurrentMemoryContext->name
```

在 `ExecEvalExprSwitchContext()` 内部，`CurrentMemoryContext` 应切到 `"ExprContext"`。在 `MemoryContextReset()` 被 `ResetExprContext()` 调用时，`context->name` 也通常是 `"ExprContext"`。

### 源码搜索入口

几个高价值搜索：

```bash
rg -n "ResetExprContext\\(" src/backend src/include
rg -n "ExecEvalExprSwitchContext" src/backend src/include
rg -n "ecxt_per_tuple_memory" src/backend src/include
rg -n "MemoryContextSwitchTo\\(estate->es_query_cxt" src/backend/executor
```

读这些结果时，不要只数调用点。要按边界分类：

```text
tuple cycle reset
qual failed 后 reset
rescan reset + callback
runtime key / partition pruning / index scan 临时 context
agg/hash/sort 工作 context
query end delete
```

## 11. 常见误区

### 误区一：per-tuple context 里的结果可以保存到下一行

不能默认保存。下一次 `ResetExprContext()` 后，指向 per-tuple context 的 by-reference 指针就失效。

正确判断：

```text
pass-by-value Datum 可以复制值本身；
pass-by-reference Datum 必须确认数据 owner；
需要跨 tuple 保存时，复制到 per-query、agg/hash/sort context 或其它正确 owner。
```

### 误区二：TupleTableSlot 活得久，所以 slot 里的所有值都安全

slot 本身通常是 per-query 的，但 slot 里的指针可能引用短命结果。`tts_values` 数组活着，不代表数组元素指向的数据也活着。

正确判断：

```text
slot 是否 materialized？
slot 是否 owns tuple？
Datum 指向的是 tuple 内部、buffer、per-tuple chunk，还是更长 context？
```

### 误区三：reset 等价于 delete

`ResetExprContext()` 会清空 `ecxt_per_tuple_memory`，但 context 节点还在，下一行继续复用。

`FreeExprContext()` 会 delete 这个 per-tuple context，并释放 `ExprContext` 节点。`FreeExecutorState()` 则 delete 整个 per-query context。

### 误区四：MemoryContext 能释放所有资源

MemoryContext 只处理普通内存。表达式 callback、slot owned tuple、snapshot、relation、buffer pin、临时文件、JIT context 等都有各自 cleanup 路径。

因此 `FreeExecutorState()` 里先显式处理 `ExprContext` callback 和 JIT/partition directory，再 delete per-query context。这个顺序不是多余的。

### 误区五：把临时对象放到 per-query 就一定安全

从 dangling pointer 角度看可能安全，但从内存峰值角度看可能错误。

如果每行产生的临时对象都放进 per-query context，一条扫描千万行的查询会把所有中间结果留到 query end。正确做法是：

```text
只把跨 tuple 真正需要的状态放进 per-query 或工作 context；
每行临时 scratch 留在 per-tuple context。
```

## 12. 课堂实验

### 实验一：观察 `ExprContext` 不随行数线性增长

Session A：

```sql
SELECT pg_backend_pid();

SELECT count(*)
FROM generate_series(1, 50000000) g
WHERE length(md5(g::text) || md5((g + 1)::text)) > 0;
```

Session B 在查询运行期间：

```sql
SELECT pg_log_backend_memory_contexts(<pid_from_session_a>);
```

观察日志中的：

```text
ExecutorState
ExprContext
```

讨论：

```text
这条 query 每行都生成字符串，为什么 ExprContext 没有按 50000000 行线性增长？
这些字符串在哪个边界被释放？
```

如果机器太快，可以增大 `generate_series` 上限，或换成更重的表达式。

### 实验二：用 gdb 看表达式求值时的 context 切换

启动一个调试 backend，设置：

```gdb
break ExecEvalExprSwitchContext
commands
  silent
  print econtext->ecxt_per_tuple_memory->name
  print CurrentMemoryContext->name
  continue
end
```

然后执行：

```sql
SELECT length(md5(g::text))
FROM generate_series(1, 10) g;
```

再把断点改成手动单步：

```gdb
break ExecEvalExprSwitchContext
```

进入函数后观察：

```gdb
next
print CurrentMemoryContext->name
next
print CurrentMemoryContext->name
```

预期现象：

```text
进入 evalfunc 前，CurrentMemoryContext 切到 ExprContext；
返回后，CurrentMemoryContext 切回 oldContext。
```

这个实验把“表达式临时结果进入 per-tuple context”从概念变成可见状态。

### 实验三：从源码确认 reset 边界

阅读 `src/include/executor/execScan.h` 的 `ExecScanExtended()`：

```text
ResetExprContext(econtext)
  -> ExecScanFetch()
  -> econtext->ecxt_scantuple = slot
  -> ExecQual()
  -> ExecProject()
  -> qual failed 时再次 ResetExprContext()
```

回答：

```text
如果 qual 中的函数返回一个 palloc 的 text *，
它最多能活到哪里？

如果 projection 需要把这个 text * 返回给上层节点，
上层节点如何避免拿到 reset 后的悬空指针？

如果 Hash Join 需要把这个值存成 hash key，
为什么不能只保存 Datum 指针？
```

### 实验四：扩展函数缓存指针的思想实验

假设一个 C 扩展函数：

```text
第一次调用:
  result = palloc(...)
  flinfo->fn_extra = result
  return result

第二次调用:
  return flinfo->fn_extra
```

如果第一次调用发生在表达式求值中，`palloc()` 的目标很可能是 per-tuple context。

讨论：

```text
下一行 tuple 开始前 ResetExprContext() 后，fn_extra 指向哪里？
如果这个 cache 真要跨行存在，应该分配到哪个 context？
为什么 fn_extra 的生命周期通常应和 flinfo/function cache 绑定，而不是和当前 tuple 绑定？
```

## 13. 讨论题

1. 为什么 `ExprContext` 需要同时有 `ecxt_per_query_memory` 和 `ecxt_per_tuple_memory`，而不是只用一个 context？
2. `ResetExprContext()` 放在每轮 tuple cycle 开始前，而不是每个表达式函数返回后，解决了什么接口复杂度？
3. 一个 pass-by-reference `Datum` 被存进数组后，判断它是否安全需要看哪些信息？
4. 为什么 `ReScanExprContext()` 要调用 shutdown callbacks，而普通 `ResetExprContext()` 不调用？
5. 如果一个扩展函数为了减少重复计算把结果缓存在 `fn_extra`，它应该如何选择 memory context？
6. 为什么 query 级状态放进 `estate->es_query_cxt` 是合理的，但每行临时字符串放进去通常是 bug？
7. 在一次长 query 中看到 `ExecutorState` 持续增长，如何区分“合理保存跨 tuple 状态”和“短命数据放错 context”？

## 14. 本节小结

本节的核心规律是：

```text
短生命周期内存的正确模型不是“谁分配谁 pfree”，而是“同一生命周期的临时结果进入同一个 context，在边界 reset”。
```

在 executor 中，这个规律具体表现为：

```text
EState / es_query_cxt:
  一次 executor run 的 per-query owner

ExprContext / ecxt_per_tuple_memory:
  一轮 tuple expression evaluation 的临时内存 owner

ResetExprContext():
  tuple cycle 边界，把上一轮 expression results 批量释放

FreeExecutorState():
  query end 边界，释放整个 executor working storage
```

指针能否跨过 reset，不取决于变量名、`Datum` 是否被保存、slot 是否还活着，而取决于：

```text
指针指向的数据由哪个 owner 管；
那个 owner 的 reset/delete 边界是否已经过去；
如果需要跨边界，是否已经深拷贝、materialize 或转移到更长生命周期。
```

这也是 PostgreSQL 内核代码里最常见的 memory context 判断模式：

```text
先问生命周期，再问 context；
先问 owner，再决定 copy；
先问 reset 边界，再保存指针。
```

下一节可以沿着这个问题继续推进到 transaction、portal 与 executor context 的交界：一次 SQL、一个 cursor、一个 transaction 的生命周期为什么不完全重合。
