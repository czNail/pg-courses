# PostgreSQL expression ERROR 与 memory cleanup

## 课程定位

前置知识：熟悉 executor 的 `PlanState` / `ExprState` / `ExprContext`，
理解 `Datum` / `isnull`、fmgr 函数调用、SRF、MemoryContext 和
ResourceOwner 的基本语义。

本节唯一主问题：

```text
函数 ERROR、SRF 中断和表达式短路如何保证 per-tuple memory、FunctionCallInfo 和 ResourceOwner 收尾？
```

核心矛盾：表达式求值位于 executor hot path，不能为每个临时对象付出逐个
登记和析构的成本；但任意 C 函数可以 `ERROR`，SRF 可以被上层提前停止，
布尔表达式和 strict 函数又会短路，系统仍然必须让内存、调用现场和外部资源
回到可清理状态。

学完后应能判断：

- 哪些对象只需要 `MemoryContextReset()`。
- 哪些对象需要 `ExprContext` shutdown callback。
- 哪些对象必须进入 `ResourceOwner`。
- 为什么 `FunctionCallInfo` 不是资源 owner。
- 为什么短路不是 cleanup 漏洞，而是未执行路径或延迟到边界批量清理。

本课基于本地 `/home/nail/postgres` 源码，分支 `master`，
提交 `0e1f1ed157e9`。

本节遵守 `00-course-writing-standard.md`，并参考
`01-core-infrastructure/01-shmem-sizing-segment.md` 的边界读法、
`01-core-infrastructure/05-*` 到 `14-*` 的 MemoryContext / ResourceOwner
课程，以及 04 目录中 `10`、`13`、`23`、`66` 节。

## 1. 本节在总主线中的位置

04 目录前面已经把 executor 分成几层。

`ExecutorStart()` 建立 `EState`。

`ExecutorRun()` 通过 `ExecutePlan()` 拉取 tuple。

`PlanState` 通过 `ExecProcNode()` 一次返回一个 `TupleTableSlot`。

表达式通过 `ExprState` 和 `ExprContext` 访问当前 tuple、参数和短命内存。

fmgr 把 SQL 函数、操作符和扩展 C 函数统一成 `PGFunction(fcinfo)`。

ProjectSet 把 targetlist 中的 SRF 展开为多行输出。

本节接在这些边界之后，只追一个问题：

```text
表达式没有按普通 return 路径走完时，资源如何收尾？
```

这里的“没有走完”有三种。

第一，函数内部 `ereport(ERROR)`。

第二，SRF 已经开始返回多行，但上层不再继续消费。

第三，strict、`AND`、`OR`、`CASE` 或 qual 短路，后续表达式没有执行。

这三类场景不能混成一个“异常处理”。

函数 `ERROR` 会 longjmp 出当前 C 调用栈。

SRF 中断通常不是错误，只是上层控制流停止。

短路也不是错误，而是表达式解释器有意跳过 step。

本节的基本模型是：

```text
per-tuple MemoryContext 负责短命 palloc 内存；
ExprContext callback 负责 rescan/delete 时的表达式局部 shutdown；
ResourceOwner 负责 buffer pin、catcache ref、snapshot 等外部资源；
FunctionCallInfo 只负责传参和返回协议，不负责释放资源。
```

这四层分工是本节主线。

如果只说“事务 abort 会清理”，就会漏掉 SRF 正常提前停止。

如果只说“ExprContext callback 会清理”，就会误解 ERROR 路径。

当前源码明确说明：`ExprContext` shutdown callback 在 error abort 时不会被调用。

所以本节要把正常停止、rescan、free 和 `ERROR` 区分开。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
表达式在 ecxt_per_tuple_memory 中产生短命对象；
函数调用复用 ExprState/SetExprState 中的 FunctionCallInfo；
ResetExprContext 只 reset per-tuple memory；
ReScanExprContext/FreeExprContext 在非 error cleanup 时调用 ExprContext callback；
ERROR 依赖 MemoryContext tree 和 ResourceOwner release 兜底。
```

核心 tension 是：

```text
低成本批量 cleanup
  vs
函数、SRF 和短路可以让局部 C return 路径消失或不完整
```

PostgreSQL 的选择不是在每个表达式对象上挂析构函数。

普通短命对象进入 per-tuple context。

需要表达式 shutdown 通知的对象注册 `ExprContext` callback。

需要事务级外部资源 release 的对象登记到 `CurrentResourceOwner`。

`FunctionCallInfo` 不承担这些责任。

它只是一次调用的 envelope。

它可能是表达式状态里的内存。

它可能是 `SetExprState` 跨多次 SRF 调用复用的内存。

它也可能来自 `LOCAL_FCINFO` 的栈空间。

因此它没有统一 release 时机。

本节最重要的不变量是：

```text
raw pointer 不是 owner；
FunctionCallInfo 不是 owner；
ERROR-safe cleanup 只能依赖栈外生命周期对象。
```

短路也服从这个不变量。

被短路跳过的 step 没有执行，也就没有创建资源。

已经执行过的 step 产生的内存等 per-tuple reset。

已经登记到 ResourceOwner 的资源等 owner release。

已经注册的 expression shutdown 动作等 rescan 或 normal free。

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/nodes/execnodes.h` | `ExprContext`、`ReturnSetInfo`、`SetExprState`、`ProjectSetState`。 |
| 2 | `src/include/fmgr.h` | `FunctionCallInfoBaseData`、`InitFunctionCallInfoData`、`FunctionCallInvoke`。 |
| 3 | `src/include/executor/executor.h` | `ResetExprContext`、`ResetPerTupleExprContext`、`ExecQualAndReset` 宏。 |
| 4 | `src/backend/executor/execUtils.c` | `CreateExprContext()`、`FreeExprContext()`、`ReScanExprContext()`、callback 注册和 shutdown。 |
| 5 | `src/backend/executor/execExprInterp.c` | `EEOP_FUNCEXPR*`、strict、boolean step、jump step。 |
| 6 | `src/backend/executor/execMain.c` | `ExecutePlan()` 的 per-output tuple reset 和 `ExecutorEnd()` cleanup。 |
| 7 | `src/backend/executor/nodeProjectSet.c` | `ExecProjectSet()`、`ExecProjectSRF()`、`argcontext` 和 pending SRF。 |
| 8 | `src/backend/executor/execSRF.c` | `ExecMakeFunctionResultSet()`、`ShutdownSetExpr()`、SRF protocol。 |
| 9 | `src/backend/utils/fmgr/funcapi.c` | `InitMaterializedSRF()`、`FuncCallContext`、`shutdown_MultiFuncCall`。 |
| 10 | `src/backend/utils/fmgr/fmgr.c` | function usage wrapper 中的 `PG_TRY` cleanup 例子。 |
| 11 | `src/backend/utils/mmgr/mcxt.c` | `MemoryContextCallback` reset/delete 行为。 |
| 12 | `src/backend/access/transam/xact.c` | abort path 和 `ResourceOwnerRelease()` 阶段。 |
| 13 | `src/include/utils/resowner.h` | ResourceOwner release phase、priority 和 callback API。 |

阅读顺序按 boundary 展开。

先读结构体。

再读 reset/free/abort。

最后读 hot path opcode。

不要从 `execExprInterp.c` 顶部线性读到尾。

本节只讲表达式错误和 cleanup 的交界。

不展开完整事务状态机、语言 handler、tuplestore 内部或所有 resource kind。

## 4. 关键数据结构与状态

### 4.1 `ExprContext`

`ExprContext` 是表达式现场。

源码注释明确说它有两个 memory context。

`ecxt_per_query_memory` 是 query-lifespan context。

`ecxt_per_tuple_memory` 是短命表达式结果 context。

`CurrentMemoryContext` 在 `ExecEvalExpr()` 时通常应切到 per-tuple context。

本节关注这些字段：

| 字段 | 语义 |
| --- | --- |
| `ecxt_scantuple` | 当前 scan tuple slot。 |
| `ecxt_innertuple` | 当前 join inner slot。 |
| `ecxt_outertuple` | 当前 join outer slot。 |
| `ecxt_per_query_memory` | 查询生命周期内存。 |
| `ecxt_per_tuple_memory` | 表达式短命结果内存。 |
| `ecxt_callbacks` | rescan/delete 时的 shutdown callback 链。 |

`ResetExprContext(econtext)` 在当前源码中只是宏：

```c
MemoryContextReset((econtext)->ecxt_per_tuple_memory)
```

它不会调用 `ecxt_callbacks`。

`ReScanExprContext(econtext)` 才会先 `ShutdownExprContext(econtext, true)`，
再 reset per-tuple memory。

`FreeExprContext(econtext, isCommit)` 会调用 `ShutdownExprContext()`，
然后 delete per-tuple context。

如果 `isCommit` 是 false，callback list 会被清掉，但 callback 函数不会执行。

这点非常关键。

`RegisterExprContextCallback()` 的源码注释也说：执行被 error abort 时不会调用 callback。

所以 `ExprContext` callback 适合正常 rescan/delete 的表达式 shutdown。

它不是 ERROR-safe 外部资源 owner。

### 4.2 `FunctionCallInfoBaseData`

`FunctionCallInfo` 是函数调用现场。

核心字段：

| 字段 | 语义 |
| --- | --- |
| `flinfo` | 指向 `FmgrInfo`。 |
| `context` | aggregate、trigger、window 等调用上下文。 |
| `resultinfo` | SRF 等额外结果协议。 |
| `fncollation` | collation。 |
| `isnull` | callee 返回 NULL 的标志。 |
| `nargs` | 参数个数。 |
| `args[]` | `NullableDatum` 参数数组。 |

源码注释强调 caller 必须提供足够 `args[]` 空间。

动态分配用 `SizeForFunctionCallInfo(nargs)`。

栈上分配用 `LOCAL_FCINFO(name, nargs)`。

`InitFunctionCallInfoData()` 不初始化 `args[]`。

这让 caller 可以复用同一块 `FunctionCallInfo`，只更新本行参数。

`FunctionCallInvoke(fcinfo)` 只调用 `fcinfo->flinfo->fn_addr(fcinfo)`。

它不会检查 strict。

它不会切换 context。

它不会登记 cleanup。

它也不会在 ERROR 后被系统拿来析构。

如果 `args[i].value` 是 pass-by-reference，指针的生命周期来自 slot、
per-tuple memory、argcontext 或其它 context。

`FunctionCallInfo` 不拥有这些指针。

### 4.3 `FmgrInfo`

`FmgrInfo` 缓存函数身份和稳定属性。

本节关注：

| 字段 | 语义 |
| --- | --- |
| `fn_addr` | 实际 C 函数入口。 |
| `fn_oid` | SQL 函数 OID。 |
| `fn_strict` | NULL 输入是否由 caller 短路。 |
| `fn_retset` | 是否声明返回 set。 |
| `fn_stats` | 是否需要 function usage 统计 path。 |
| `fn_extra` | 函数或 handler 的跨调用缓存。 |
| `fn_mcxt` | `fn_extra` 附属状态的生命周期。 |
| `fn_expr` | 调用表达式节点。 |

`fn_extra` 是常见 cleanup 风险点。

SQL 函数、SRF helper、类型 I/O 或扩展函数都可能把状态挂到这里。

但 `fn_extra` 只是指针。

真正的 owner 是 `fn_mcxt`、callback 或 ResourceOwner。

### 4.4 `ReturnSetInfo`

SRF 通过 `ReturnSetInfo` 和 caller 协商多行结果。

核心字段：

| 字段 | 语义 |
| --- | --- |
| `econtext` | SRF 所在表达式上下文。 |
| `expectedDesc` | caller 期望的 tuple descriptor。 |
| `allowedModes` | caller 能接受的返回模式。 |
| `returnMode` | callee 实际选择的模式。 |
| `isDone` | value-per-call 状态。 |
| `setResult` | materialize 模式的 tuplestore。 |
| `setDesc` | materialize 模式的 tuple descriptor。 |

`ReturnSetInfo` 不是 owner。

它告诉 caller：本次返回是单值、一个 set 元素、set 结束，还是 tuplestore。

tuplestore 的 cleanup 要靠 `SetExprState`、ExprContext callback、memory context
和 ResourceOwner 的组合。

### 4.5 `SetExprState`

`SetExprState` 保存潜在 SRF 的状态。

本节关注：

| 字段 | 语义 |
| --- | --- |
| `func` | `FmgrInfo`。 |
| `funcResultStore` | materialize SRF 的 tuplestore。 |
| `funcResultSlot` | 从 tuplestore 读出的 slot。 |
| `funcResultDesc` | 返回 tuple 描述符。 |
| `funcReturnsSet` | 初始化期记录是否返回 set。 |
| `setArgsValid` | value-per-call 参数是否需要跨调用复用。 |
| `shutdown_reg` | 是否注册了 shutdown callback。 |
| `fcinfo` | SRF 调用现场。 |

`setArgsValid` 的语义是：SRF 正在 value-per-call 中间，
下一次调用要复用已经求过值的参数。

这就是 `FunctionCallInfo` 会跨多次调用存在的场景。

但它仍不是 owner。

如果未耗尽 SRF 要被取消，`ShutdownSetExpr()` 清理 `funcResultStore`、
清 `setArgsValid`、清 `shutdown_reg`。

### 4.6 `ProjectSetState`

ProjectSet 是 targetlist SRF 的 executor node。

关键字段：

| 字段 | 语义 |
| --- | --- |
| `elems` | 每个 targetlist 元素的表达式状态。 |
| `elemdone` | 每个 SRF 的 done 状态。 |
| `pending_srf_tuples` | 当前输入 tuple 是否还有 SRF 输出。 |
| `argcontext` | 保存 SRF 参数求值结果的 context。 |

`ExecProjectSet()` 每次入口先 `ResetExprContext(econtext)`。

这释放上一条输出 tuple 的表达式短命内存。

如果 `pending_srf_tuples` 为 true，它继续同一个输入 tuple。

只有要取新输入 tuple 时，才 `MemoryContextReset(node->argcontext)`。

这个顺序保证 value-per-call SRF 的参数能活过多条输出行。

### 4.7 `MemoryContextCallback`

`MemoryContextCallback` 和 `ExprContext` callback 不是一回事。

`MemoryContextRegisterResetCallback()` 把 callback 注册到某个 memory context。

reset/delete 时 `MemoryContextCallResetCallbacks()` 会先把 callback 从链表弹出，
再调用它。

如果 callback 内部 ERROR，后续 reset/delete 不会再次调用同一个 callback。

这种 callback 适合和 memory context 生命周期强绑定的状态。

但外部资源仍应优先使用 ResourceOwner。

### 4.8 `ResourceOwner`

`ResourceOwner` 记录内存以外的资源。

`resowner.h` 把 release 分成三阶段：

- `RESOURCE_RELEASE_BEFORE_LOCKS`。
- `RESOURCE_RELEASE_LOCKS`。
- `RESOURCE_RELEASE_AFTER_LOCKS`。

before-locks 阶段包括 buffer pin、relcache ref、DSM、JIT 等对其它 backend
可见或有顺序要求的资源。

after-locks 阶段包括 catcache ref、snapshot ref、file、wait event set 等
backend-local cleanup。

表达式函数如果通过内核 API 获取这些资源，通常会登记到 `CurrentResourceOwner`。

`ERROR` 后局部 C cleanup 不执行，但 abort path 会 release owner。

## 5. 主流程源码 walkthrough

### 5.1 初始化期：从表达式到调用现场

典型链路：

```text
ExecutorStart()
  -> ExecInitNode()
     -> ExecInitExpr()
        -> fmgr_info_cxt()
        -> palloc(SizeForFunctionCallInfo(nargs))
        -> InitFunctionCallInfoData()
        -> 选择 EEOP_FUNCEXPR / STRICT / FUSAGE opcode
```

初始化期把 catalog 信息压进 `FmgrInfo`。

把每次调用会变化的参数槽放进 `FunctionCallInfo`。

把 strict、retset、function usage 统计等分支提前编译成 opcode。

这样 executor hot path 不用每行查 catalog。

初始化期的内存通常在 `es_query_cxt` 或表达式状态 context 中。

如果初始化期 ERROR，清理由查询 context 和事务 abort 处理。

### 5.2 tuple 周期：顶层 reset

`ExecutePlan()` 的主循环每轮开始执行：

```text
ResetPerTupleExprContext(estate)
  -> ResetExprContext(estate->es_per_tuple_exprcontext)
     -> MemoryContextReset(ecxt_per_tuple_memory)
```

这只 reset per-output-tuple context。

各节点自己的 `ExprContext` 也会在节点逻辑中 reset。

例如 ProjectSet 每次入口先 reset 本节点 econtext。

这条路径不会调用 `ExprContext` shutdown callback。

所以普通 per-tuple reset 的语义很窄：

```text
释放短命表达式内存，不取消未完成 SRF，不释放 ResourceOwner 资源。
```

### 5.3 普通函数：填参数、调用、读结果

`EEOP_FUNCEXPR` 的模型很短：

```c
fcinfo->isnull = false;
d = op->d.func.fn_addr(fcinfo);
*op->resvalue = d;
*op->resnull = fcinfo->isnull;
```

参数表达式在前置 step 中写入 `fcinfo->args[]`。

callee 返回 `Datum`，并通过 `fcinfo->isnull` 标记 NULL。

如果 callee 在 per-tuple context 中 palloc，下一次 reset 清理。

如果 callee 获取外部资源，必须正常 release 或登记到 ResourceOwner。

如果 callee `ERROR`，调用后的赋值不会发生。

### 5.4 strict 函数：caller 侧短路

`FunctionCallInvoke()` 不检查 strict。

strict 由表达式解释器或 SRF caller 做。

普通 strict opcode 的语义是：

```text
检查 args[i].isnull；
只要有 NULL，直接返回 NULL；
否则重置 fcinfo->isnull 并调用 fn_addr。
```

因此 SQL 函数标记为 `STRICT` 后，NULL 输入时 C 函数本体不会执行。

函数内部写的 `PG_ARGISNULL()` 分支不会成为 cleanup 点。

已经求值的参数产生的短命内存仍等 per-tuple reset。

未执行的函数本体没有创建函数内部资源。

### 5.5 boolean / qual / CASE 短路

`execExprInterp.c` 中有 `EEOP_BOOL_AND_STEP`、`EEOP_BOOL_OR_STEP`、
`EEOP_QUAL`、`EEOP_JUMP_IF_NOT_TRUE` 等 step。

语义分别是：

- `AND` 遇到 false 可以跳到结束。
- `OR` 遇到 true 可以跳到结束。
- qual 遇到 false 或 NULL 直接返回 false。
- jump step 改变解释器 program counter。

短路不会遍历被跳过的 step 做 cleanup。

这是正确的，因为那些 step 没执行。

已经执行过的 step 按自己的 owner 收尾。

### 5.6 ProjectSet：一个输入 tuple，多条输出

`ExecProjectSet()` 的核心顺序是：

```text
ResetExprContext(econtext)
if pending_srf_tuples:
    ExecProjectSRF(continuing = true)
else:
    MemoryContextReset(argcontext)
    outerTupleSlot = ExecProcNode(outerPlan)
    econtext->ecxt_outertuple = outerTupleSlot
    ExecProjectSRF(continuing = false)
```

`ExecProjectSRF()` 在 per-tuple context 中求 targetlist。

普通表达式直接 `ExecEvalExpr()`。

SRF 调用 `ExecMakeFunctionResultSet()`。

如果某个 SRF 返回 `ExprMultipleResult`，
`pending_srf_tuples` 会被设为 true。

下一次上层拉取时，ProjectSet 不取新输入 tuple，而是继续同一个输入 tuple。

### 5.7 SRF 参数：为什么需要 `argcontext`

`ExecMakeFunctionResultSet()` 的注释说：

```text
argContext needs to live until all rows have been returned.
```

如果 `setArgsValid` 为 false，它会切到 `argContext` 求参数。

如果 value-per-call SRF 返回 `ExprMultipleResult`，
它设置 `setArgsValid = true`，并注册 `ShutdownSetExpr` callback。

下一次继续同一个 SRF 时，不重新求参数。

这避免参数指针被每输出行的 per-tuple reset 释放。

当换到新输入 tuple 时，ProjectSet reset `argcontext`。

### 5.8 materialize SRF：tuplestore 状态

`ReturnSetInfo.allowedModes` 在 ProjectSet SRF 路径中允许：

```text
SFRM_ValuePerCall | SFRM_Materialize
```

如果函数选择 materialize，`rsinfo.setResult` 会被交给
`ExecPrepareTuplestoreResult()`。

`SetExprState.funcResultStore` 保存 tuplestore。

后续调用从 tuplestore 读 row。

读完后 `tuplestore_end()` 并清空 `funcResultStore`。

如果没读完就被上层停止，`ShutdownSetExpr()` 在正常 rescan/free 路径中负责结束它。

### 5.9 rescan/free：表达式 callback 执行点

`ExecReScan()` 在 node-specific rescan 前会：

```text
if (node->ps_ExprContext)
    ReScanExprContext(node->ps_ExprContext);
```

`ReScanExprContext()` 调用 `ShutdownExprContext(econtext, true)`，
然后 reset per-tuple memory。

`FreeExecutorState()` 会逐个 `FreeExprContext(..., true)`，
保证剩余 callback 有机会执行。

源码注释还强调：`FreeExecutorState()` 不负责释放非内存资源，
但会显式 shutdown ExprContext，因为 callback 可能释放不在 per-query context
里的资源。

### 5.10 ERROR：callback 不执行，owner 兜底

当函数 `ERROR` 时，局部调用栈不会普通展开。

`RegisterExprContextCallback()` 的注释明确说：

```text
callback will not be called in the event that execution is aborted by an error
```

`FreeExprContext(econtext, false)` 也只清 callback list，不调用 callback。

因此 ERROR-safe cleanup 不能依赖 ExprContext callback。

ERROR 路径依赖：

- memory context delete/reset 释放 palloc 内存。
- ResourceOwner release 释放外部资源。
- transaction / portal cleanup 删除 executor state。
- 必要时 MemoryContext reset callback 处理与 context 绑定的状态。

这就是本节的核心边界。

## 6. 生命周期 / ownership / cleanup

### 6.1 谁创建

`CreateExecutorState()` 创建 `es_query_cxt`。

`CreateExprContext()` 在 `es_query_cxt` 内创建 `ExprContext`，并创建
独立的 `ecxt_per_tuple_memory`。

表达式初始化期创建 `FmgrInfo` 和 `FunctionCallInfo` storage。

`ExecInitProjectSet()` 创建 `ProjectSetState.argcontext`。

事务开始时 `AtStart_ResourceOwner()` 创建 `TopTransactionResourceOwner`，
并设置 `CurrentResourceOwner`。

子事务开始时 `AtSubStart_ResourceOwner()` 创建子 owner。

### 6.2 谁持有

`ExprContext` 挂在 `EState.es_exprcontexts` 链表。

`PlanState.ps_ExprContext` 指向节点自己的表达式上下文。

普通函数调用现场由 `ExprState` step 持有。

SRF 调用现场由 `SetExprState` 持有。

SRF 参数求值结果由 `ProjectSetState.argcontext` 持有。

外部资源由当前 ResourceOwner tree 持有。

### 6.3 正常释放

普通表达式短命内存由 `ResetExprContext()` 释放。

ProjectSet 换输入 tuple 时 reset `argcontext`。

SRF 自然结束时清 `setArgsValid` 或结束 `funcResultStore`。

rescan 时 `ReScanExprContext()` 调用 expression callback。

`ExecutorEnd()` 最终调用 `FreeExecutorState()`，再 delete `es_query_cxt`。

事务结束时 `ResourceOwnerRelease()` 按阶段释放资源。

### 6.4 ERROR / abort 释放

`AbortTransaction()` 先切到 abort-safe memory context，
调用 `AtAbort_ResourceOwner()` 确保 `CurrentResourceOwner` 有效。

随后在 post-abort cleanup 中，如果存在 `TopTransactionResourceOwner`，
会执行：

```text
ResourceOwnerRelease(..., RESOURCE_RELEASE_BEFORE_LOCKS, false, true)
ResourceOwnerRelease(..., RESOURCE_RELEASE_LOCKS, false, true)
ResourceOwnerRelease(..., RESOURCE_RELEASE_AFTER_LOCKS, false, true)
```

`CleanupTransaction()` 再 `ResourceOwnerDelete()`，并 `AtCleanup_Memory()`。

子事务 abort 有对应的 subtransaction owner release。

因此函数 ERROR 后能否安全收尾，取决于资源是否进入了正确 owner 层级。

### 6.5 长期对象失效

`FmgrInfo.fn_extra` 随 `fn_mcxt` 失效。

`FunctionCallInfo.args[]` 指向的对象随参数来源失效。

`ReturnSetInfo` 只在当前调用有效。

`SetExprState` 随表达式状态失效。

`argcontext` 在新输入 tuple 或 executor cleanup 时失效。

ResourceOwner release 后，登记的资源句柄失效。

裸指针跨这些边界使用，就是 stale pointer 或 stale resource。

## 7. 正确性机制层次

本节正确性由多层机制组合。

| 层次 | 机制 | 保证 | 不保证 |
| --- | --- | --- | --- |
| 短命内存 | per-tuple MemoryContext | palloc 对象批量释放 | 外部资源 release |
| 表达式 shutdown | ExprContext callback | rescan/delete 时取消 SRF 等局部状态 | error abort 时执行 |
| 函数协议 | FunctionCallInfo | 参数、NULL、collation、resultinfo 传递 | ownership |
| SRF 协议 | ReturnSetInfo / SetExprState | value-per-call 和 materialize 状态 | 自动消费完整 set |
| 控制流 | opcode jump / strict | 跳过未执行表达式 | 清理已执行资源 |
| 外部资源 | ResourceOwner | abort/commit/subabort release | malloc/static 状态 |
| 错误传播 | ERROR longjmp | 进入事务错误处理 | C 局部析构自动运行 |

MemoryContext 和 ResourceOwner 是互补关系。

MemoryContext 释放内存。

ResourceOwner 释放资源语义。

ExprContext callback 位于两者之间，处理表达式局部的“正常取消”。

它不是事务 abort 机制。

这三者的边界不能互换。

## 8. 错误路径 / 异常路径 / fallback

### 8.1 普通函数 `ERROR`

函数内部 `ereport(ERROR)` 后，表达式解释器不会继续执行调用后的赋值。

`fcinfo->isnull` 或 result slot 可能处于半更新状态。

这些值不再有 SQL 语义。

cleanup 进入事务/portal 层。

per-tuple context 最终随 executor context 删除。

ResourceOwner release 释放登记资源。

### 8.2 strict NULL 短路

strict 函数遇到 NULL 参数时，函数本体不执行。

这意味着：

- 函数内部不会分配资源。
- 函数内部 cleanup 代码不会运行。
- 参数表达式已产生的内存仍等 per-tuple reset。
- NULL 结果由 caller 构造。

把 SQL 函数标成 `STRICT` 后，不要期望 C 函数处理 NULL 输入。

### 8.3 boolean 短路

`false AND f()` 和 `true OR f()` 可以跳过 `f()`。

`CASE` 也会跳过未选择分支。

被跳过的函数没有调用现场，没有函数内部资源。

已经执行的左侧表达式仍按自己的 owner 清理。

### 8.4 SRF 被 LIMIT 提前停止

例子：

```sql
SELECT generate_series(1, 1000000)
LIMIT 1;
```

上层只需要一行。

SRF 未必自然返回 `ExprEndResult`。

正常 `ExecutorEnd()` 会通过 `FreeExecutorState()` 调用 remaining
ExprContext callback。

如果是 ERROR abort，callback 不执行，必须依赖 memory context 和 ResourceOwner。

### 8.5 rescan 取消未完成 SRF

`ExecReScan()` 先 `ReScanExprContext()`。

这会调用 `ShutdownSetExpr()`。

随后 node-specific rescan 清节点状态。

ProjectSet 的 `ExecReScanProjectSet()` 会把 `pending_srf_tuples` 清为 false。

下一次执行取新输入 tuple 前 reset `argcontext`。

### 8.6 callback 内部 ERROR

MemoryContext reset callback 在调用前会先从链表弹出。

ExprContext callback 也在 `ShutdownExprContext()` 中先从链表取下再调用。

这降低了重复 cleanup 风险。

但 callback 不应做复杂、可能失败的业务逻辑。

资源 release callback 也只能做非关键 cleanup，源码注释要求不应失败。

### 8.7 子事务 abort

PL/pgSQL exception block、SPI 或内部子事务会改变 owner 层级。

资源登记到子 owner，会在 subabort 释放。

资源登记到父 owner，会活过子事务。

诊断 expression ERROR 时，不能只问“有没有 ResourceOwner”。

还要问“登记在哪一层 ResourceOwner”。

## 9. 成本、资源与跨模块传播

### 9.1 hot path 成本

per-tuple reset 的成本随输出 tuple 数增长。

函数调用成本包括参数写入、NULL 检查、`isnull` 重置和函数指针调用。

strict 与 boolean 短路减少无用调用，但增加 opcode 分支。

SRF 增加 `pending_srf_tuples` 状态、`argcontext` 保留和多次调用协议。

function usage 统计会进入 `ExecEvalFuncExprFusage()`，
增加 `pgstat_init_function_usage()` 和 `pgstat_end_function_usage()`。

ResourceOwner 登记过多会增加 remember/forget 和 abort release 成本。

### 9.2 规模变量

| 变量 | 放大点 |
| --- | --- |
| tuple 数 | reset 次数、函数调用次数、参数写入次数。 |
| 表达式 step 数 | opcode dispatch、branch、jump。 |
| SRF 输出倍数 | 同一输入 tuple 保留参数的时间。 |
| materialize SRF 结果量 | tuplestore 内存、临时文件、cleanup 成本。 |
| ERROR 频率 | abort cleanup、ResourceOwner release、portal cleanup。 |
| track_functions | 观测开销进入函数 hot path。 |

### 9.3 跨模块边界

executor expression interpreter 负责短路和函数 opcode。

fmgr 负责函数 ABI。

execSRF 和 ProjectSet 负责 SRF 多行协议。

MemoryContext 负责内存生命周期。

ResourceOwner 负责外部资源生命周期。

transaction manager 负责 abort release 顺序。

portal/executor cleanup 负责 query 状态结束。

pgstat 负责 function usage 观测。

这些模块互相连接，但边界清晰：

```text
MemoryContext 不是 ResourceOwner；
ExprContext callback 不是 ERROR callback；
FunctionCallInfo 不是 cleanup hook；
ReturnSetInfo 不是 tuplestore owner。
```

## 10. 观测与诊断入口

### 10.1 能直接看到

`EXPLAIN (ANALYZE)` 能看到 ProjectSet、Limit、loops、rows。

`pg_stat_user_functions` 在开启 `track_functions` 后能看到函数调用次数和时间。

server log 能看到 `ERROR` 位置。

`pg_log_backend_memory_contexts(pid)` 能打印 memory context tree 快照。

`gdb` 能查看 `ExprContext`、`SetExprState`、`FunctionCallInfo` 和
`CurrentResourceOwner`。

### 10.2 只能推断

每次 `ResetExprContext()` 没有普通 SQL view。

`FunctionCallInfo.args[]` 的瞬时内容没有 view。

boolean 短路是否跳过函数，通常通过函数调用计数、side effect、ERROR 是否发生
或断点推断。

SRF callback 是否执行，通常需要断 `ShutdownSetExpr()` 或 `ShutdownExprContext()`。

ResourceOwner 中具体资源数量也通常需要 debug、断点或有针对性的日志。

### 10.3 推荐断点

```text
ResetExprContext 相关调用点
ReScanExprContext
FreeExprContext
ShutdownExprContext
ExecProjectSet
ExecProjectSRF
ExecMakeFunctionResultSet
ShutdownSetExpr
MemoryContextCallResetCallbacks
AbortTransaction
ResourceOwnerRelease
```

普通函数 opcode 非常热。

更实用的办法是按目标函数 OID、`FmgrInfo.fn_oid` 或 `SetExprState.func.fn_oid`
设置条件断点。

### 10.4 runtime truth

本节锚定的 runtime truth：

```text
短路和 SRF 提前停止不会逐对象析构；
cleanup 成立，是因为状态进入了正确 lifecycle owner，并在 reset/rescan/free/abort 边界统一收尾。
```

看到内存增长时，先区分它是 per-tuple context 峰值、`argcontext`
跨 SRF 输出保留、per-query `fn_extra` cache、ResourceOwner 资源未 release，
还是扩展绕过 PostgreSQL owner 的真实泄漏。

## 11. 常见误区

### 11.1 把 `ResetExprContext` 当 SRF shutdown

当前源码中 `ResetExprContext` 只是 `MemoryContextReset()`；取消未完成 SRF
要靠 `ReScanExprContext()` 或 `FreeExprContext(..., true)`。

### 11.2 认为 ExprContext callback 会在 ERROR 时执行

源码明确说不会；`FreeExprContext(false)` 只清链表，不调用 callback。
ERROR-safe 外部资源必须使用 ResourceOwner 或其它 abort-safe owner。

### 11.3 把 `FunctionCallInfo` 当 owner

`FunctionCallInfo` 只传参数和结果，不释放 `args[]` 指针，不释放
`fn_extra`，也没有统一生命周期。

### 11.4 认为 strict 函数会处理 NULL 输入

strict NULL 由 caller 短路，函数本体通常不执行，C 代码中的 NULL
分支不会成为 cleanup 入口。

### 11.5 认为 SRF 总会自然结束

Limit、cursor close、rescan、portal drop 或 ERROR 都可能让 SRF 不到
`ExprEndResult`。
自然结束路径之外必须有取消路径。

### 11.6 把 MemoryContext 当 ResourceOwner

MemoryContext 释放 palloc 内存，ResourceOwner 释放资源语义；buffer pin、
snapshot、catcache ref 不能只靠释放内存解决。

### 11.7 忽略 owner 层级

子事务会创建子 ResourceOwner；资源登记到错误层级，可能过早释放或释放过晚。
ERROR cleanup 诊断必须看 `CurrentResourceOwner`。

## 12. 课堂实验

### 实验 1：短路是否调用函数

创建函数：

```sql
CREATE OR REPLACE FUNCTION boom() RETURNS bool
LANGUAGE plpgsql AS $$
BEGIN
  RAISE EXCEPTION 'boom called';
END;
$$;
```

执行：

```sql
SELECT false AND boom();
SELECT true OR boom();
SELECT CASE WHEN false THEN boom() ELSE true END;
SELECT boom();
```

前三条不应调用 `boom()`。

最后一条直接报错。

回到源码看 `EEOP_BOOL_*`、`EEOP_JUMP_IF_NOT_TRUE` 和 CASE jump step。

### 实验 2：per-tuple reset 与 callback 区别

在 debug build 中给这些位置下断点：

```text
ResetExprContext 的调用点
ReScanExprContext
ShutdownExprContext
```

执行一个多行表达式查询，再执行一个会 rescan 的计划。

观察：

- 普通每行 reset 不进入 `ShutdownExprContext()`。
- rescan 会进入 `ShutdownExprContext(..., true)`。
- error cleanup 不应期待 callback 执行。

### 实验 3：ProjectSet 的 `argcontext`

执行：

```sql
SELECT generate_series(1, 3), repeat('x', 10)
FROM generate_series(1, 2) g;
```

断点：

```text
ExecProjectSet
ExecProjectSRF
ExecMakeFunctionResultSet
MemoryContextReset(node->argcontext)
```

观察同一输入 tuple 的多条输出之间，per-tuple context 会 reset，
但 `argcontext` 要等换输入 tuple 才 reset。

## 13. 讨论题

1. 为什么 `FunctionCallInfo` 不能负责释放 `args[]` 指向的对象？
2. `ResetExprContext` 和 `ReScanExprContext` 的 cleanup 语义差异是什么？
3. 为什么 `ExprContext` callback 不能作为 ERROR-safe cleanup 机制？
4. strict 函数 NULL 输入时，函数内部 cleanup 代码为什么不会执行？
5. `false AND f()` 跳过 `f()` 后，是否需要为 `f()` 做 cleanup？
6. value-per-call SRF 为什么需要 `setArgsValid`？
7. ProjectSet 为什么需要 `argcontext`？
8. materialize SRF 被 `LIMIT 1` 提前停止时，谁结束 tuplestore？
9. ResourceOwner release 为什么要分 before-locks、locks、after-locks？
10. 子事务 abort 时，资源登记到父 owner 和子 owner 的差异是什么？

## 14. 本节小结

本节主问题是：

```text
函数 ERROR、SRF 中断和表达式短路如何保证 per-tuple memory、FunctionCallInfo 和 ResourceOwner 收尾？
```

答案不是某个统一 cleanup 函数。PostgreSQL 依赖分层 owner：per-tuple
MemoryContext 清理普通表达式短命内存，`ExprContext` callback 清理正常
rescan/free 时的表达式局部状态，`ResourceOwner` 清理事务、子事务、portal
或 executor 路径中的外部资源。`FunctionCallInfo` 只负责调用协议，不负责
ownership；`ResetExprContext` 只 reset memory，不调用 expression callback；
`ReScanExprContext` 和 `FreeExprContext(..., true)` 才执行 callback。error
abort 时 expression callback 不执行，所以函数 `ERROR` 的安全性必须落到
memory context tree 和 ResourceOwner。短路路径没有特殊 cleanup：被跳过的
step 没执行，已执行 step 的资源按原 owner 收尾。

可迁移的系统规律是：

```text
hot path 上用批量生命周期降低成本；
但任何可能跨过局部 return 路径的资源，都必须进入栈外 owner。
```

诊断时始终问五个问题：

- 谁创建？
- 谁持有？
- 正常路径谁释放？
- rescan/free 时谁取消？
- ERROR/abort 时谁兜底？
