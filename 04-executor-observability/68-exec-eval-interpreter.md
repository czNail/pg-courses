# PostgreSQL ExecEval expression interpreter

## 课程定位

前置知识：已经理解 executor 的 `EState`、`PlanState`、`ExprContext`、`TupleTableSlot`、`Datum` / `isnull` 表示法、fmgr 函数调用边界，以及 per-tuple memory context 的 reset 语义。

本节唯一主问题：

```text
表达式解释器如何把 ExprState steps 执行成 Datum / null，短生命周期内存在哪里 reset？
```

核心矛盾：表达式执行位于每 tuple hot path。它必须表达完整 SQL 语义，包括 `Var`、函数、操作符、`CASE`、布尔短路、参数、投影、聚合和 subplan；同时又不能在每行递归遍历 AST、重复 catalog lookup、逐个释放临时对象，或者让 NULL 协议散落到每个 caller 里。

PostgreSQL 的运行模型是：初始化期把表达式树编译成平坦的 `ExprEvalStep[]`，执行期通过 `ExprState.evalfunc` 解释或 JIT 执行这些 steps，每个 step 把中间或最终结果写入 `Datum *` / `bool *` 目标；表达式求值中的短命 by-reference 结果进入 `ExprContext.ecxt_per_tuple_memory`，由外层 tuple cycle 的 `ResetExprContext()` 批量释放。

学完后应能判断：一个 `ExprState` 活到什么时候；一个 step 的 `resvalue` / `resnull` 指向哪里；`ExecQual()` 为什么把 `NULL` 当作 false；strict 函数和布尔表达式如何短路；JIT 成功时解释器边界如何被替换，失败时为什么 fallback 到解释器；一个 by-reference `Datum` 能不能跨过下一次 per-tuple reset。

本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e`。用户建议核对的 `src/backend/executor/execQual.c` 在这个 checkout 中不存在；当前版本的 qual 初始化在 `execExpr.c`，`ExecQual()` / `ExecQualAndReset()` 是 `executor.h` 中的 inline helper。

## 1. 本节在总主线中的位置

04 目录前面的课程已经走过 executor 生命周期、`PlanState` runtime boundary、`TupleTableSlot`、`ExprContext`、per-tuple memory reset、type I/O 和 fmgr 函数调用边界。

本节把焦点压到最热的一层：

```text
ExecProcNode()
  -> 产出或读取 TupleTableSlot
     -> qual / projection / targetlist expression
        -> ExprState.evalfunc
           -> Datum + isnull
```

第 10 节回答 `ExprContext` 为什么携带当前 tuple、参数和短生命周期内存。第 13 节回答为什么 executor 要在 tuple 周期 reset per-tuple context。第 66 节回答 fmgr 如何把 SQL 函数、操作符和 C 函数 ABI 统一起来。本节回答这些状态进入表达式解释器后，`ExprState.steps` 如何真正被执行。

本节不展开 planner 如何生成表达式树，不覆盖全部 `EEOP_*`，也不把 JIT LLVM IR 作为第二主题。JIT 只作为 `ExecReadyExpr()` 选择执行方法时的 replacement / fallback 边界出现。

本节锚定的 runtime truth 是：

```text
同一个 ExprState 会被很多 tuple 反复执行；
每次执行只返回 Datum/isnull 或把结果写入 projection slot；
本轮求值产生的短命内存靠外层 ExprContext reset 收尾。
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ExecInitExpr() / ExecInitQual()
  -> ExecInitExprRec() 把表达式树编译成 ExprEvalStep[]
  -> ExecReadyExpr() 选择 JIT 或 interpreted execution
  -> ExecEvalExpr*() 调用 state->evalfunc
  -> ExecInterpExpr() 按 opcode dispatch
  -> step 写 Datum/resnull 或跳转
  -> DONE_RETURN 返回 Datum/isnull
  -> 外层 ResetExprContext() 回收短命内存
```

这条链的 tension 是：

```text
通用 SQL 表达式语义、NULL 传播、短路和扩展函数
  vs
每 tuple hot path 的 dispatch、cache miss、函数调用和内存释放成本
```

如果 executor 每次都递归解释 AST，形态会接近：

```text
eval(BooleanExpr)
  -> eval(OpExpr)
     -> eval(Var)
     -> eval(Const)
     -> call operator
  -> eval(FuncExpr)
     -> eval(args...)
```

这个模型容易理解，但每行都要付出递归调用、node tag 分派、参数临时结构和 ownership 判断成本。布尔短路、`CASE`、strict 函数又会让“哪些子树执行过、哪些临时值存在”成为动态问题。

PostgreSQL 把复杂性前移到初始化期：

```text
tree:
  OpExpr(left=Var, right=Const)

steps:
  FETCHSOME
  VAR
  CONST
  FUNCEXPR_STRICT_2
  DONE_RETURN
```

执行期解释器不再递归走表达式树。它只沿平坦 step 数组前进、跳转、调用 helper 或函数指针。需要短路时，step 中已经有 jump target。需要函数时，初始化期已经准备好 `FmgrInfo` 和 `FunctionCallInfo` storage。需要 `Var` 时，setup step 已经知道从 scan、outer、inner、old、new 哪个 slot 读。

代价是 `ExprState` 不再像源 SQL 表达式那样直观。读源码必须同时看 step 顺序、每个 step 的输出目标、jump target、`ExprContext` 当前 slot、当前 memory context，以及 `state->evalfunc` 当前是解释器 wrapper、fast path 还是 JIT 函数。

本节 mental model：

```text
ExprState 是 backend-local 的表达式小程序；
ExprEvalStep 是解释器指令；
Datum/isnull 是指令间传值协议；
ExprContext 是当前 tuple、参数和短命内存现场；
ResetExprContext() 是短命表达式结果的释放边界。
```

## 3. 核心文件分工与阅读顺序

本节按 mental model 读文件，不按文件名排序。

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/nodes/execnodes.h` | `ExprState`、`ExprContext` 的公开状态；先看 result storage、`evalfunc` 和两个 memory context 注释。 |
| 2 | `src/include/executor/execExpr.h` | `ExprEvalOp`、`ExprEvalStep`、解释器 private flags、helper 原型；这是指令集。 |
| 3 | `src/backend/executor/execExpr.c` | `ExecInitExpr()`、`ExecInitQual()`、`ExprEvalPushStep()`、`ExecReadyExpr()`；树如何变成 steps。 |
| 4 | `src/backend/executor/execExprInterp.c` | `ExecReadyInterpretedExpr()`、`ExecInterpExpr()`、`EEOP_*` dispatch；解释器 hot path。 |
| 5 | `src/include/executor/executor.h` | `ExecEvalExpr()`、`ExecEvalExprSwitchContext()`、`ExecQual()`、`ExecQualAndReset()`、`ResetExprContext()` inline 边界。 |
| 6 | `src/backend/executor/execUtils.c` | `CreateExecutorState()`、`CreateExprContext()`、`FreeExprContext()`、`FreeExecutorState()`；owner、callback、JIT cleanup。 |
| 7 | `src/include/executor/execScan.h` | `ExecScanExtended()` 展示每 tuple reset、qual、projection 的调用顺序。 |
| 8 | `src/backend/executor/execTuples.c` | `ExecMaterializeSlot()`、slot copy/materialize 如何把需要跨 reset 的数据转到 slot context。 |
| 9 | `src/include/jit/jit.h`、`src/backend/jit/jit.c` | `jit_compile_expr()` 何时替换解释器，何时返回 false fallback。 |
| 10 | `src/backend/jit/llvm/llvmjit_expr.c` | LLVM provider 消费同一套 step 语义；本节只作为 JIT 边界参考。 |

两个主要入口：

```text
初始化:
  ExecInitExpr()
  ExecInitQual()
  ExecBuildProjectionInfo()

执行:
  ExecEvalExpr()
  ExecEvalExprSwitchContext()
  ExecQual()
  ExecProject()
```

两个收尾边界：

```text
短命内存:
  ResetExprContext(econtext)

长期状态:
  FreeExecutorState(estate)
    -> FreeExprContext(...)
    -> jit_release_context(...)
    -> MemoryContextDelete(estate->es_query_cxt)
```

读 `execExprInterp.c` 时不要从顶部把全部 opcode 背完。选一条 SQL，比如：

```sql
SELECT a + 1
FROM t
WHERE b IS NOT NULL AND f(b) > 10;
```

然后追问：`Var b` 从哪个 slot 来；`b IS NOT NULL` 的结果写到哪里；`AND` 如何短路；`f(b)` 的 strict NULL 检查由哪个 step 做；最终 qual 的 `NULL` 为什么返回 false；本行临时函数结果何时释放。

## 4. 关键数据结构与状态

### 4.1 `ExprState`

`ExprState` 在 `execnodes.h` 中代表一整棵表达式树的执行状态。源码注释说它包含用于求值的 instructions。

核心字段：

| 字段 | 语义 |
| --- | --- |
| `flags` | public 与 private 标志，如 `EEO_FLAG_IS_QUAL`、`EEO_FLAG_DIRECT_THREADED`。 |
| `resvalue` / `resnull` | 标量表达式默认最终结果槽，也常被子表达式临时复用。 |
| `resultslot` | projection 输出 slot；普通标量表达式通常为 `NULL`。 |
| `steps` | `ExprEvalStep` 数组，解释器执行的指令流。 |
| `evalfunc` | 真实执行入口，可能是 still-valid wrapper、fast path、解释器或 JIT 函数。 |
| `expr` | 原始表达式树，主要用于调试。 |
| `evalfunc_private` | evalfunc 私有入口，例如 `ExecInterpExpr` 或 `ExecJust*`。 |
| `steps_len` / `steps_alloc` | 初始化期构建 step 数组；运行时不要当业务语义。 |
| `parent` | 所属 `PlanState`，JIT 和 subplan/agg/window 绑定会依赖它。 |
| `ext_params` | standalone 表达式编译 `PARAM_EXTERN` 时使用。 |
| `innermost_caseval` / `innermost_casenull` | `CASE` 初始化期绑定内部 test value。 |
| `innermost_domainval` / `innermost_domainnull` | domain check 初始化期绑定内部 test value。 |
| `escontext` | soft error 场景的错误保存上下文。 |

不变量：

```text
ExprState 是 backend-local、per-query 级执行状态；
它在 runtime 会 mutate；
不能被多个并发执行共享；
释放通常依赖包含它的 MemoryContext；
没有 ExecEndExpr()。
```

`ExecInitExpr()` 注释明确要求调用 context 必须覆盖表达式重复执行期，通常是相关 `ExprContext` 的 per-query context。原始 `Expr` tree 可以作为只读计划结构存在；`ExprState` 不应当被当成只读计划树。

### 4.2 `ExprEvalStep`

`ExprEvalStep` 是解释器指令。核心结构可以压缩成：

```c
typedef struct ExprEvalStep
{
    intptr_t opcode;
    Datum   *resvalue;
    bool    *resnull;
    union { ... } d;
} ExprEvalStep;
```

`opcode` 初始是 `ExprEvalOp` 枚举值。如果启用 computed goto，`ExecReadyInterpretedExpr()` 会把它改成 label address，所以类型是 `intptr_t`。

`resvalue` / `resnull` 是这条 step 的输出目标。它们可能指向：

- `ExprState.resvalue` / `ExprState.resnull`。
- `FunctionCallInfo.args[i].value` / `args[i].isnull`。
- projection `resultslot->tts_values[i]` / `tts_isnull[i]`。
- `CASE` 或 domain 的临时 value。
- 其它单独 palloc 出来的 runtime storage。

这就是平坦 step 表示能避免递归返回协议的关键：子表达式不是把结果返回给父表达式，而是直接写到后续 step 已知的位置。

构建期有一个重要约束：

```text
在 ExprState 构建完成前，不允许长期保存 steps[] 内部元素地址。
```

`ExprEvalPushStep()` 容量不够时会 `repalloc` 整个数组。指向旧数组内部的指针会失效。因此函数调用的 `FunctionCallInfoBaseData` 等 runtime storage 通常外部分配，再由 step 引用。

`ExprEvalStep` 还有 cache locality 约束。源码有 `sizeof(ExprEvalStep) <= 64` 的 static assert。向 union 塞大对象会扩大所有 step 的 footprint，影响解释器顺序扫描的 cache 行行为。

### 4.3 `ExprEvalOp`

`ExprEvalOp` 不是 AST node type，而是执行动作。

| opcode 族 | 语义 |
| --- | --- |
| `EEOP_*_FETCHSOME` | 确保某个 slot 已 deform 到需要的 attribute。 |
| `EEOP_*_VAR` | 从 scan / inner / outer / old / new slot 取非系统列。 |
| `EEOP_*_SYSVAR` | 取系统列。 |
| `EEOP_CONST` | 写入常量 `Datum` 和 null flag。 |
| `EEOP_FUNCEXPR*` | 调用函数、操作符或其它 fmgr-backed routine。 |
| `EEOP_BOOL_AND_STEP*` / `EEOP_BOOL_OR_STEP*` | 普通 SQL 三值逻辑布尔表达式，带短路。 |
| `EEOP_QUAL` | WHERE / join qual 专用，`NULL` 和 false 都提前返回 false。 |
| `EEOP_JUMP*` | 控制流跳转。 |
| `EEOP_PARAM_*` | 读取或设置 `PARAM_EXEC` / external param。 |
| `EEOP_CASE_TESTVAL*` | `CASE` test value 传递。 |
| `EEOP_ASSIGN_*` | projection 中把结果写入 output slot。 |
| `EEOP_AGG_*` | 聚合 transition 相关专用热路径。 |
| `EEOP_DONE_RETURN` / `EEOP_DONE_NO_RETURN` | 表达式结束并返回，或结束但不返回值。 |

不要把 `EEOP_FUNCEXPR` 理解成只执行 SQL 函数。操作符、类型相关函数、support routine、聚合 transition 等也可能通过相关 step 或 helper 执行。

不要把 `EEOP_BOOL_AND_STEP` 和 `EEOP_QUAL` 混为一谈。普通 `AND` 保留三值逻辑；qual 的语义是过滤 tuple，WHERE 中 `NULL` 不选中，所以 `EEOP_QUAL` 会把 null 合并成 false。

### 4.4 `ExprContext`

`ExprContext` 是解释器的当前执行环境，不是指令本身。

关键字段：

| 字段 | 语义 |
| --- | --- |
| `ecxt_scantuple` | scan Var 当前 slot。 |
| `ecxt_innertuple` | inner Var 当前 slot。 |
| `ecxt_outertuple` | outer Var 当前 slot。 |
| `ecxt_per_query_memory` | query 生命周期 context，常用于函数调用 cache。 |
| `ecxt_per_tuple_memory` | 表达式结果短期 context，通常每 tuple reset。 |
| `ecxt_param_exec_vals` | `PARAM_EXEC` 值数组。 |
| `ecxt_param_list_info` | external param。 |
| `ecxt_aggvalues` / `ecxt_aggnulls` | Agg / WindowAgg 预计算值。 |
| `ecxt_callbacks` | rescan 或 shutdown 时要回调的资源清理。 |

`execnodes.h` 的注释强调：调用 `ExecEvalExpr()` 前，`CurrentMemoryContext` 应该是 `ecxt_per_tuple_memory`。这说明 `ExecInterpExpr()` 不负责切 memory context。

两类入口：

```text
caller 已经在外层切好:
  ExecEvalExpr(state, econtext, &isnull)

单次求值时自动切换:
  ExecEvalExprSwitchContext(state, econtext, &isnull)
```

`ExecEvalExprSwitchContext()` 只做 `MemoryContextSwitchTo(econtext->ecxt_per_tuple_memory)`、调用 `state->evalfunc`、切回旧 context。它不 reset。reset 仍然由 tuple cycle 或 rescan/shutdown 边界负责。

### 4.5 `Datum` / `isnull`

表达式解释器的值语义是：

```text
Datum value + bool isnull
```

如果 `isnull == true`，`Datum` 内容没有 SQL 语义。调试时不能只看 `state->resvalue`，必须同时看 `state->resnull`。

strict 函数短路能说明这一点。`EEOP_FUNCEXPR_STRICT_1` 遇到 NULL 参数时只需要设置：

```text
*op->resnull = true
```

它不调用函数，也不需要给 `*op->resvalue` 写有意义的 SQL 值。

### 4.6 `ProjectionInfo`

投影也是表达式程序。`ProjectionInfo` 内嵌 `ExprState pi_state`，`ExecBuildProjectionInfo()` 会把 targetlist 编译成直接写 output slot 的 steps。

常见 fast path：

```text
Var 直接来自 input slot:
  EEOP_ASSIGN_*_VAR

复杂表达式:
  先写 ExprState.resvalue/resnull
  再由 EEOP_ASSIGN_TMP 写 resultslot
```

slot 的 `tts_values` 数组活得久，不代表数组里的 by-reference 指针也能跨 reset。能否保存要看指针目标由谁拥有。

## 5. 主流程源码 walkthrough
本节用一个 scan + qual + projection 场景做主线：
```sql
SELECT a + 1 AS x
FROM t
WHERE b IS NOT NULL AND f(b) > 10;
```
目标不是列出每个 opcode，而是看状态如何从初始化期推进到每 tuple 执行期。
### 5.1 创建 per-query owner
`CreateExecutorState()` 在 `execUtils.c` 创建 `"ExecutorState"` memory context，并把 `EState` 本身分配在其中：
```text
CurrentMemoryContext
  -> AllocSetContextCreate("ExecutorState")
     -> estate->es_query_cxt
     -> estate
```
这一步建立长期边界：`ExprState.steps` 活到本次 executor run 结束，不是每 tuple 创建，也不是每 tuple reset。
### 5.2 创建 `ExprContext`
`CreateExprContextInternal()` 做三件关键事：
```text
切到 estate->es_query_cxt 分配 ExprContext node
设置 ecxt_per_query_memory = estate->es_query_cxt
在 estate->es_query_cxt 下创建 "ExprContext" per-tuple memory
```
随后把 `ExprContext` 链到 `estate->es_exprcontexts`，让 `FreeExecutorState()` 能逐个 `FreeExprContext()`，先跑 shutdown callbacks，再 delete per-tuple context。
这里有两个不同生命周期：
```text
ExprContext node:
  per-query
ecxt_per_tuple_memory:
  per-tuple reset
  FreeExprContext 时 delete
```
### 5.3 `ExecInitExpr()` 编译普通表达式
`ExecInitExpr()` 的骨架是：
```text
if node == NULL:
  return NULL
state = makeNode(ExprState)
state->expr = node
state->parent = parent
ExecCreateExprSetupSteps(state, node)
ExecInitExprRec(node, state, &state->resvalue, &state->resnull)
push EEOP_DONE_RETURN
ExecReadyExpr(state)
return state
```
`ExecInitExprRec()` 为每个子表达式指定输出位置。函数参数可以直接写 `fcinfo->args[i]`，投影中间值可以写 `ExprState.resvalue`，最后由 assign step 写入 slot。
这把运行时父子返回协议改成了初始化期的 target binding。结果是执行期少一次层层返回，但初始化期必须保证目标不会在仍被需要时被覆盖。
### 5.4 `ExprEvalPushStep()` 追加指令
`ExprEvalPushStep()` 初始分配 16 个 step，容量不够时翻倍并 `repalloc`。
状态变化：
```text
steps_alloc: 0 -> 16 -> 32 -> ...
steps_len:   0 -> n
```
因为 `repalloc` 会移动数组，构建期不能保存 `steps[]` 内部地址。构建完成后，解释器才可以稳定顺序读取。
### 5.5 `ExecInitQual()` 编译 WHERE / join qual
`ExecInitQual()` 对 implicit AND list 做专门编译。
空 qual 直接返回：
```text
if qual == NIL:
  return NULL
```
所以 `ExprState *` 为 `NULL` 在 `ExecQual()` 中表示 true，这是 hot path 优化。
非空 qual 会设置：
```text
state->flags = EEO_FLAG_IS_QUAL
```
然后为每个子表达式追加 `EEOP_QUAL`。`EEOP_QUAL` 的语义是：
```text
if current result is NULL or false:
  write false
  jump to done
else:
  continue
```
这和普通 SQL `AND` 不同。WHERE 中 unknown 不选中 tuple，所以 qual 可以把 NULL 合并成 false。
### 5.6 `ExecReadyExpr()` 选择 JIT 或解释器
构建完成后必须调用 `ExecReadyExpr()`：
```text
if (jit_compile_expr(state))
  return;
ExecReadyInterpretedExpr(state);
```
`jit_compile_expr()` 可能因为这些条件返回 false：
- `state->parent == NULL`。
- `estate->es_jit_flags` 没有 `PGJIT_PERFORM`。
- `estate->es_jit_flags` 没有 `PGJIT_EXPR`。
- `jit_enabled` 为 false。
- provider 不存在或加载失败。
JIT 失败不是 ERROR。它只是 fallback 到解释器。JIT 成功也不是另一套 SQL 语义，而是同一套 `ExprState` / `ExprEvalStep` 的另一种执行方法。
### 5.7 `ExecReadyInterpretedExpr()` 准备解释执行
解释器准备阶段：
```text
ExecInitInterpreter()
Assert 最后一个 step 是 DONE
如果已初始化则返回
state->evalfunc = ExecInterpExprStillValid
设置 EEO_FLAG_INTERPRETER_INITIALIZED
尝试选择 ExecJust* fast path
可用 computed goto 时把 opcode 改为 label address
state->evalfunc_private = ExecInterpExpr
```
第一次执行前 `evalfunc` 可能是 `ExecInterpExprStillValid`。它做一次 still-valid 检查，通过后再把 `evalfunc` 切到真正执行入口。极简单表达式可能走 `ExecJust*`，例如 const、单 Var、简单 assign var。
### 5.8 `ExecScanExtended()` 展示 reset 位置
`execScan.h` 的 `ExecScanExtended()` 是观察 per-tuple reset 的好入口。
有 qual 或 projection 时，循环前先 reset：
```text
ResetExprContext(econtext)
```
抽象流程：
```text
for (;;)
{
  slot = ExecScanFetch(...)
  if TupIsNull(slot):
    return empty slot
  econtext->ecxt_scantuple = slot
  if ExecQual(qual, econtext):
    return ExecProject(projInfo)
  InstrCountFiltered1(...)
  ResetExprContext(econtext)
}
```
这说明解释器本身不 reset。候选 tuple 开始前 reset；qual 失败后再 reset；下一候选 tuple 重新使用同一个 `ExprState`。
### 5.9 `ExecEvalExpr*()` 进入 evalfunc
`ExecEvalExpr()` 很薄：
```text
return state->evalfunc(state, econtext, isNull);
```
它假设 caller 已经在 `econtext->ecxt_per_tuple_memory`。如果没有，caller 用 `ExecEvalExprSwitchContext()` 单次切换。
`ExecQual()` 会调用表达式求值，并断言 `EEOP_QUAL should never return NULL`。因为 `EEOP_QUAL` 已经把 null 合并成 false。
### 5.10 `ExecInterpExpr()` 设置本地执行现场
`ExecInterpExpr()` 开头抓取当前状态：
```text
op = state->steps
resultslot = state->resultslot
innerslot = econtext->ecxt_innertuple
outerslot = econtext->ecxt_outertuple
scanslot = econtext->ecxt_scantuple
oldslot = econtext->ecxt_oldtuple
newslot = econtext->ecxt_newtuple
```
这些是当前调用快照。下一 tuple 更新 `econtext->ecxt_scantuple` 后，下一次执行会重新读取。
dispatch 有两种实现：
```text
computed goto direct threading
standard C switch threading
```
语义相同，差别是跳转预测和分支成本。
### 5.11 `FETCHSOME` 与 `VAR`
`EEOP_*_FETCHSOME` 确保对应 slot deform 到需要的最高 attribute。`EEOP_*_VAR` 从 slot 的 `tts_values` / `tts_isnull` 取值并写到 step 目标。
抽象成：
```text
slot_getsomeattrs(slot, last_var)
*op->resvalue = slot->tts_values[attnum]
*op->resnull = slot->tts_isnull[attnum]
```
表达式解释器不直接解析 heap tuple 物理格式。它依赖 `TupleTableSlot` abstraction。
### 5.12 `CONST`
`EEOP_CONST` 只把常量 value/null 写到目标：
```text
*op->resvalue = const value
*op->resnull = const isnull
```
这个最小例子说明 step 之间不是通过 C 递归返回传值，而是通过预先绑定的 target storage 传值。
### 5.13 `FUNCEXPR`
普通函数 step：
```text
fcinfo->isnull = false
d = op->d.func.fn_addr(fcinfo)
*op->resvalue = d
*op->resnull = fcinfo->isnull
```
strict 函数 step 会先检查参数 null。一参数、二参数 strict 函数有专用 opcode，多参数 strict 函数走循环。
strict NULL 语义：
```text
任何参数为 NULL:
  不调用 fn_addr
  结果 isnull = true
```
因此 strict 是 caller-side step 协议，不是 `FunctionCallInvoke()` 的隐式魔法。
### 5.14 `AND` / `OR` 短路
普通 SQL `AND` 保留三值逻辑：
```text
false AND anything = false
true AND null = null
true AND true = true
```
解释器用 `EEOP_BOOL_AND_STEP_FIRST` 重置 `anynull`，中间 step 遇 null 记录，遇 false 就 jump 到 done，最后 step 再根据 `anynull` 决定 null 或 true。
`OR` 类似：
```text
true OR anything = true
false OR null = null
false OR false = false
```
短路的本质是修改下一条 step，不是递归返回。被跳过的 steps 不执行，所以相关函数调用、ERROR、统计和临时内存都不会发生。
### 5.15 `QUAL`
`EEOP_QUAL` 是 qual 专用的简化 AND step。
```text
if resnull or !DatumGetBool(resvalue):
  resnull = false
  resvalue = false
  jump done
```
这让 `ExecQual()` 返回普通 `bool`，调用者不需要再处理 SQL 三值逻辑。
### 5.16 `JUMP`
`CASE`、`COALESCE`、`NULLIF`、`BoolExpr`、JSON expression、domain check 都可能生成 jump step。
常见形式：
```text
EEOP_JUMP
EEOP_JUMP_IF_NULL
EEOP_JUMP_IF_NOT_NULL
EEOP_JUMP_IF_NOT_TRUE
```
初始化期经常先 push 一个 jump step，把 target 设成占位值；递归生成可能被跳过的子表达式后，再回填真实 target。这就是 `adjust_jumps` 列表存在的原因。
### 5.17 `DONE_RETURN` 与 `DONE_NO_RETURN`
`EEOP_DONE_RETURN`：
```text
*isnull = state->resnull
return state->resvalue
```
`EEOP_DONE_NO_RETURN` 用于 projection 或聚合 transition 这类“副作用就是结果”的表达式程序。它断言 `isnull == NULL` 并返回 0。
### 5.18 `ExecProject()`
投影表达式通常把结果直接写进 `resultslot`：
```text
ExecProject(projInfo)
  -> clear result slot
  -> ExecEvalExprNoReturnSwitchContext(&projInfo->pi_state, econtext)
  -> mark result slot as virtual tuple
```
step 可能是：
```text
FETCHSOME
ASSIGN_SCAN_VAR
CONST
FUNCEXPR
ASSIGN_TMP
DONE_NO_RETURN
```
短命内存规则不变。如果结果列是 by-reference `Datum` 且指向 per-tuple context，下一次 reset 后不能被长期引用。需要跨边界保存时，slot materialize/copy 或 tuplestore 必须接管 ownership。
## 6. 生命周期 / ownership / cleanup
### 6.1 谁创建
`ExprState` 通常由 plan node 初始化阶段创建：
```text
ExecInitNode()
  -> ExecInitExpr()
  -> ExecInitQual()
  -> ExecBuildProjectionInfo()
```
这些调用通常发生在 `estate->es_query_cxt` 或与 plan node 生命周期一致的 context 中。standalone 表达式的 `ExecPrepareExpr()` 会先切到 `estate->es_query_cxt`，再运行 `expression_planner()` 和 `ExecInitExpr()`。
### 6.2 谁持有
语义 owner 通常是 `PlanState`、`ProjectionInfo`、constraint state、hash key state 或 executor 辅助结构。内存 owner 是 memory context。
```text
语义 owner:
  谁会调用这个 ExprState
内存 owner:
  ExprState 分配在哪个 MemoryContext
```
两者通常都落在 per-query 生命周期，但不能偷换。
### 6.3 谁释放
普通 executor run 结束：
```text
ExecutorEnd()
  -> FreeExecutorState(estate)
     -> FreeExprContext(...) for remaining ExprContexts
     -> jit_release_context(estate->es_jit)
     -> MemoryContextDelete(estate->es_query_cxt)
```
`MemoryContextDelete(estate->es_query_cxt)` 释放 `ExprState`、`ExprEvalStep` 数组、函数调用 storage、PlanState 工作状态等。`FreeExecutorState()` 先关闭 remaining `ExprContext`，是为了让 shutdown callback 有机会释放非内存资源。
### 6.4 per-tuple context 谁创建和 reset
`CreateExprContextInternal()` 创建：
```text
ecxt_per_tuple_memory =
  AllocSetContextCreate(estate->es_query_cxt, "ExprContext", ...)
```
它是 per-query context 的 child。query 结束时会最终释放，但长扫描不能等到 query 结束，所以外层每 tuple reset。
主要 reset 点：
| 调用者 | reset 语义 |
| --- | --- |
| `ExecScanExtended()` | 每候选 tuple 前 reset，qual 失败后再 reset。 |
| `ExecQualAndReset()` | 求值 qual 后立即 reset。 |
| `ResetPerTupleExprContext(estate)` | per-output-tuple context 已创建时 reset。 |
| join / agg / project set node | 在各自 tuple、group、probe 或 SRF 边界 reset。 |
| `ReScanExprContext()` | rescan 时先 shutdown callbacks，再 reset。 |
核心答案：
```text
分配目标由 CurrentMemoryContext 指向 ecxt_per_tuple_memory 决定；
释放由 tuple/rescan/shutdown 边界的 ResetExprContext() 或 FreeExprContext() 决定；
ExecInterpExpr() 不逐个释放表达式临时对象。
```
### 6.5 ERROR / abort 兜底
表达式中函数、类型 I/O、subplan、domain check、JSON path 都可能 `ereport(ERROR)`。ERROR longjmp 不会按普通 C 返回链执行后续 cleanup。
本节对象依赖外层机制兜底：
- per-tuple context 中的临时对象随上层 reset/delete 清理。
- per-query context 中的 `ExprState` 随 `FreeExecutorState()` 或外层 query context delete 清理。
- callback 管理的非内存资源由 `FreeExprContext()` / `ReScanExprContext()` 的 shutdown callback 处理。
- JIT provider 的错误状态由 JIT infrastructure reset-after-error 边界处理，JIT context 在 executor cleanup 时 release。
MemoryContext 不是 ResourceOwner。它能释放 palloc 内存，不能释放 buffer pin、relation refcount、file descriptor 或扩展私有外部资源。
### 6.6 by-reference `Datum` 能否跨 reset
判断规则：
```text
by-value Datum:
  可以复制数值本身
by-reference Datum:
  必须知道指向对象由哪个 context 或 slot 拥有
```
如果函数在 `ecxt_per_tuple_memory` 中返回 `text *`，指针只保证活到下一次 reset。要跨边界保存，需要 `datumCopy()` 到更长 context、`ExecMaterializeSlot()`、`ExecCopySlot()`、tuplestore，或在正确 context 中重建结果。
`execTuples.c` 的 materialize / copy 路径会切到 `slot->tts_mcxt`，并从头 deform 或 copy tuple，避免 `tts_values[]` 指向即将失效的非 materialized tuple。
## 7. 正确性机制层次
### 7.1 `Datum` / `isnull` 是最小语义单元
表达式值不是裸 `Datum`，而是：
```text
Datum value + bool isnull
```
所有 step 必须维护这对状态。NULL 时 `Datum` 不解释；非 NULL 时 `Datum` 才按类型解释。
这影响 strict 函数、`IS NULL`、布尔三值逻辑、`EEOP_QUAL` false 合并、projection slot 的 `tts_isnull[]`。
### 7.2 slot deforming 边界
`EEOP_*_FETCHSOME` 把 tuple 物理格式和表达式求值隔开。
```text
TupleTableSlot ops
  -> slot_getsomeattrs()
  -> tts_values / tts_isnull 有效
  -> EEOP_*_VAR 读取
```
表达式解释器不直接保证 heap tuple lifetime。它依赖 slot abstraction、slot owner 和相关 pin / materialize 边界。
### 7.3 first-call validity check
`ExecReadyInterpretedExpr()` 先把 `state->evalfunc` 设为 `ExecInterpExprStillValid`。第一次执行会检查表达式中记录的 slot / attribute 假设是否仍有效，通过后再切到真正执行入口。
这把检查移出每条 step hot path。调试时看到 `evalfunc` 不是 `ExecInterpExpr` 不一定表示没有走解释器；它可能是 still-valid wrapper、`ExecJust*` fast path，或 JIT 函数。
### 7.4 qual、check、普通 boolean 不同
`ExecInitQual()` 编译的 expression 带 `EEO_FLAG_IS_QUAL`。`ExecCheck()` 会断言它不是 qual 编译结果。
```text
WHERE / join qual:
  NULL -> false
CHECK constraint:
  NULL -> true
普通 boolean expression:
  保留 SQL 三值逻辑
```
这是 SQL 语义差异，不只是 executor 实现细节。
### 7.5 MemoryContext 只管内存
`ecxt_per_tuple_memory` 只保证短命 palloc chunk 被 reset。它不保证函数没有副作用，不保证 slot 指向的 buffer 仍 pin 住，不保证 syscache tuple 仍有效，也不保证 SRF 或 subplan 资源已经关闭。
非内存资源必须通过 `ResourceOwner`、slot ownership、callback 或模块自己的 cleanup 协议处理。
### 7.6 JIT 与解释器共享语义
JIT 不是新的 SQL 语义实现。它必须共享：
- `Datum` / `isnull` 协议。
- strict 函数协议。
- slot deforming 边界。
- per-tuple memory context。
- helper 函数。
- `ExprEvalStep` 控制流。
`execExprInterp.c` 注释说明复杂或少见指令会调 helper，这些 helper 被解释器和 JIT 共享。helper 不能自己 dispatch 子表达式，因为 dispatch 方法由 caller 决定。
## 8. 错误路径 / 异常路径 / fallback
### 8.1 JIT fallback
`ExecReadyExpr()` 的 JIT 失败不是 ERROR：
```text
if (!jit_compile_expr(state))
  ExecReadyInterpretedExpr(state)
```
诊断“为什么没有 JIT 表达式”时，先看 `jit` GUC、provider、plan cost、`es_jit_flags`、表达式是否有 parent `PlanState`，再看解释器。
### 8.2 direct-threaded fallback
解释器支持 computed goto direct threading，也支持标准 C switch threading。构建环境支持 computed goto 时可以把 opcode 改成 label address，否则走 switch。
两者语义相同。差别是分支预测和跳转成本。跨平台 CPU profile 差异可能来自构建选项，而不是 SQL 本身。
### 8.3 strict 函数短路
strict 函数 NULL 参数路径：
```text
args[i].isnull == true
  -> *op->resnull = true
  -> skip fn_addr(fcinfo)
```
因此函数体内副作用、ERROR、分配、函数统计都不会发生。调试“为什么函数没被调用”时，先看 `fn_strict` 和参数 null。
### 8.4 表达式短路
`AND`、`OR`、`CASE`、`COALESCE`、`NULLIF` 等都可能跳过后续 step。这会影响函数是否调用、ERROR 是否触发、临时内存是否分配、`pg_stat_user_functions` 是否计数、profile 中是否出现某个函数栈。
短路是 step 流中显式 jump target 的结果，不是解释器随机行为。
### 8.5 ERROR longjmp
表达式中的函数、类型 I/O、subplan、JSON path、domain check 都可能 ERROR。ERROR 后不会执行普通 C 返回链上的后续 cleanup。
本节依赖更外层：
```text
MemoryContext:
  批量释放 palloc 内存
ResourceOwner:
  释放 pin、refcount、外部资源
ExprContext callback:
  关闭 SRF 或 expression shutdown state
ExecutorEnd / FreeExecutorState:
  关闭 executor 工作状态
```
这就是解释器不尝试逐个 `pfree()` 临时对象的原因：它不能要求所有错误路径都普通返回。
### 8.6 rescan callback
`ReScanExprContext()` 先 `ShutdownExprContext(econtext, true)`，再 reset memory。原因是某些表达式状态不是纯内存，例如未完成 SRF、table function 或其它 callback state。
把 rescan 简化成单纯 `ResetExprContext()` 可能漏掉非内存 cleanup。
## 9. 成本、资源与跨模块传播
### 9.1 CPU dispatch 成本
表达式解释器成本近似随：
```text
tuple 数 * 每 tuple 实际执行 step 数
```
扩张。每个 step 都有 dispatch、target 读写、可能的 helper/function 调用。computed goto、`ExecJust*` fast path、strict 特化、projection assign-var fast path 都是在压这部分成本。
### 9.2 slot deforming 成本
`FETCHSOME` 和 `VAR` 的成本来自 slot deforming。访问高编号列可能需要 deform 更多 attribute；varlena 多、tuple 宽、slot 类型不同都会改变成本。
如果 profile 里 `slot_getsomeattrs` / `heap_deform_tuple` 很热，优化方向可能是列访问、投影列、列顺序、slot materialization 或 JIT deforming，而不是函数本身。
### 9.3 fmgr 调用成本
函数调用 step 已缓存 `FmgrInfo` 和 `FunctionCallInfo` storage，执行期不重新查 catalog。但每行仍要写参数、检查 strict null、设置 `fcinfo->isnull`、调用函数指针、处理 usage stats。
如果函数很轻，fmgr 边界成本明显；如果函数很重，瓶颈通常在函数内部。
### 9.4 per-tuple reset 成本
`ResetExprContext()` 批量释放短命对象，避免逐个 `pfree()`。但 reset 本身也有 allocator 成本；如果每行分配大量大对象，会造成更多 block churn。
它仍比让每个表达式节点、函数、类型转换精确传递 free 责任更适合 ERROR-heavy 的 C executor。
### 9.5 cache locality
`ExprEvalStep` 限制在一个常见 cache line 尺寸内。给少见 opcode 增加大 union 字段会扩大所有 step 的 footprint，影响每个表达式的顺序扫描。
新增复杂状态时，优先外部分配并在 step 中保存指针。
### 9.6 跨模块边界
| 模块 | 与本节边界 |
| --- | --- |
| `TupleTableSlot` | `FETCHSOME` / `VAR` 依赖 slot deforming 和 values/nulls 数组。 |
| fmgr | `FUNCEXPR` 复用 `FmgrInfo` / `FunctionCallInfo` 调用 ABI。 |
| MemoryContext | per-tuple 分配和 reset 是表达式短命结果的生命周期边界。 |
| JIT | `ExecReadyExpr()` 可把同一套 steps 交给 provider 编译。 |
| instrumentation | `EXPLAIN` 能看到 JIT、函数统计、节点 timing，但看不到每个 step。 |
| subplan / param | `PARAM_EXEC`、`SUBPLAN` step 连接表达式求值和 executor 子计划。 |
| aggregate/window | `AGGREF`、`WINDOW_FUNC` 读取预计算或专用 runtime state。 |
### 9.7 并行执行边界
`ExprState`、`ExprContext`、`ExprEvalStep *` 都是 backend-local。parallel worker 不能复用 leader 的普通指针。worker 通过序列化后的 plan 在自己进程中重建 `PlanState`、`ExprState` 和 `ExprContext`。
诊断 parallel query 时要区分 leader 的表达式状态、worker 的表达式状态、DSM 中汇总的 instrumentation。
## 10. 观测与诊断入口
### 10.1 能直接看到什么
SQL 层能直接看到：
- `EXPLAIN (ANALYZE)` 的节点实际行数、loops、timing。
- `EXPLAIN` 的 JIT block 中 functions 和 generation time。
- `EXPLAIN (ANALYZE, BUFFERS)` 的 IO 与 buffer 行为。
- `pg_stat_user_functions` 在 `track_functions` 开启时的函数调用统计。
- `pg_backend_memory_contexts` 中当前 backend 的 memory context 名称和大小。
- `pg_log_backend_memory_contexts(pid)` 输出的 memory context tree。
这些都不是 step-level tracer。它们只能从外部现象推断解释器成本。
### 10.2 只能间接推断什么
通常只能间接推断：
- 某个 `EEOP_*` 执行了多少次。
- 某个 jump 是否短路了后续子表达式。
- strict 函数是否因为 NULL 参数被跳过。
- 每 tuple reset 回收了多少表达式临时对象。
- 某个 by-reference `Datum` 是否指向 per-tuple context。
这些需要 profiler、断点、临时日志或源码计数器。
### 10.3 `EXPLAIN` 读法
表达式 CPU 热点常见现象：
```text
actual rows 很大
buffers / IO 不高
wait event 不明显
CPU 时间集中在 executor
```
示例：
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*)
FROM generate_series(1, 1000000) g
WHERE length(md5(g::text) || md5((g + 1)::text)) > 10;
```
如果 buffer 命中高、IO 低，但耗时随 rows 线性增长，就要怀疑表达式 CPU。`pg_stat_*` 不能告诉你是 dispatch、deforming、fmgr 边界还是函数内部，需要 profile。
### 10.4 JIT 诊断
强制观察 JIT：
```sql
SET jit = on;
SET jit_above_cost = 0;
EXPLAIN (ANALYZE, COSTS, VERBOSE)
SELECT sum(a + 1)
FROM generate_series(1, 1000000) a;
```
看 `EXPLAIN` 的 JIT block。若没有 JIT，检查是否编译 JIT、`jit` 是否启用、provider 是否可用、plan cost 是否超过阈值、表达式是否有 parent `PlanState`、`es_jit_flags` 是否含 `PGJIT_EXPR`。
JIT generation time 也是成本。短查询强制 JIT 可能变慢。
### 10.5 memory context 诊断
查看当前 backend 的表达式 context：
```sql
SELECT name, level, total_bytes, used_bytes
FROM pg_backend_memory_contexts
WHERE name LIKE '%ExprContext%'
ORDER BY total_bytes DESC;
```
粒度限制：
- 只看当前 backend 某一时刻。
- 不能直接看到每个 tuple reset 前的瞬间峰值。
- 一个 executor run 可能有多个 `ExprContext`。
- reset 后 allocated block 形态受 allocator 行为影响。
想精确观察 reset，最稳定方式是在源码加临时计数或断点。
### 10.6 `perf` / gdb 入口
CPU 栈常见形态：
```text
ExecProcNode
  -> ExecScanExtended
     -> ExecQual
        -> ExecInterpExpr
           -> fmgr function / helper
```
也可能看到 `slot_getsomeattrs`、`heap_deform_tuple`、`FunctionCallInvoke`、text/numeric/json function。
gdb 断点入口：
```text
ExecInitExpr
ExecInitQual
ExprEvalPushStep
ExecReadyExpr
ExecReadyInterpretedExpr
ExecInterpExpr
ExecScanExtended
MemoryContextReset
```
`ResetExprContext` 是宏，断宏不方便时断 `MemoryContextReset`，再按 context name 过滤 `"ExprContext"`。
调 `ExecInterpExpr` 时不要每行单步整个 switch。更有效的是打印 `state->steps_len`、前几个 `ExecEvalStepOp(state, &state->steps[i])`，或条件断在特定 helper / opcode 分支。
## 11. 常见误区
### 误区 1：把 `ExprState` 当只读计划树
`ExprState` 会在 runtime mutate。它有 flags、`evalfunc`、`evalfunc_private`、结果槽、函数调用缓存和可能的 direct-threaded opcode。不能把它当跨执行共享的只读计划树。
### 误区 2：以为解释器会 reset 临时内存
`ExecInterpExpr()` 执行 steps，不负责每轮 reset。reset 在 tuple cycle、`ExecQualAndReset()`、rescan 或 cleanup 边界。
### 误区 3：只看 `Datum` 不看 `isnull`
NULL 时 `Datum` 没有 SQL 语义。strict 函数短路、qual false 合并、projection slot 都必须同时看 null flag。
### 误区 4：把 `EEOP_QUAL` 当普通 `AND`
`EEOP_QUAL` 是过滤语义，NULL -> false。普通 `AND` / `OR` 保留 SQL 三值逻辑。
### 误区 5：以为 JIT 改变表达式语义
JIT 和解释器共享 `ExprState` / `ExprEvalStep` 语义。JIT 是执行方法替换；失败就 fallback 到解释器。
### 误区 6：把 slot 生命周期等同于 by-reference `Datum` 生命周期
slot 数组可能活到 query 结束，数组里的指针可能指向 per-tuple context、外部 tuple、buffer-backed tuple 或 slot-owned tuple。能不能保存取决于指针目标的 owner。
### 误区 7：随意扩大 `ExprEvalStep`
step 在 hot path 顺序扫描。扩大 union 会影响所有表达式。少见复杂状态应外部分配并以指针引用。
### 误区 8：按旧资料寻找 `execQual.c`
本地 `0e1f1ed157e` checkout 没有 `src/backend/executor/execQual.c`。当前 qual 初始化在 `execExpr.c`，执行 helper 在 `executor.h` inline。
## 12. 课堂实验
### 实验 1：跟踪 qual 如何变成 steps
目标：看到 SQL WHERE 表达式如何变成 `ExprEvalStep[]`。
断点：
```text
ExecInitQual
ExecInitExprRec
ExprEvalPushStep
ExecReadyExpr
ExecReadyInterpretedExpr
```
SQL：
```sql
CREATE TEMP TABLE t(a int, b int);
INSERT INTO t
SELECT g, CASE WHEN g % 3 = 0 THEN NULL ELSE g END
FROM generate_series(1, 10) g;
EXPLAIN (ANALYZE, VERBOSE)
SELECT a
FROM t
WHERE b IS NOT NULL AND b + 1 > 5;
```
记录：`ExecInitQual()` 是否返回 `NULL`；`state->flags` 是否含 `EEO_FLAG_IS_QUAL`；`state->steps_len`；`EEOP_QUAL` 的 jump target；最后一个 step 是否是 `EEOP_DONE_RETURN`。
思考：为什么 WHERE 中 NULL 不需要返回 SQL NULL，而是可以直接变成 false？
### 实验 2：观察 strict 函数短路
目标：确认 NULL 参数时 strict 函数不调用 `fn_addr`。
SQL：
```sql
SET track_functions = 'all';
SELECT count(*)
FROM generate_series(1, 100000) g
WHERE md5(NULL::text) IS NULL;
SELECT count(*)
FROM generate_series(1, 100000) g
WHERE md5(g::text) IS NOT NULL;
```
断点：`execExprInterp.c` 的 `EEOP_FUNCEXPR_STRICT_1` 或 `EEOP_FUNCEXPR_STRICT` 分支。
记录：NULL 参数时是否调用 `fn_addr`；`*op->resnull` 如何设置；`fcinfo->isnull` 何时表达真实函数返回 NULL。
### 实验 3：验证 per-tuple reset 边界
目标：确认表达式短命分配不是在 `ExecInterpExpr()` 中逐个释放。
在源码里临时给 `MemoryContextReset()` 或 `ResetExprContext()` 调用点加日志，只记录 context name 为 `"ExprContext"` 的情况。
SQL：
```sql
CREATE TEMP TABLE t AS
SELECT g AS id, repeat(md5(g::text), 4) AS payload
FROM generate_series(1, 10000) g;
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*)
FROM t
WHERE length(payload || md5(id::text)) > 0;
```
记录：scan 节点每候选 tuple 前是否 reset；qual 失败路径是否额外 reset；`ExecInterpExpr()` 是否直接调用 reset；`pg_backend_memory_contexts` 中 `ExprContext` 大小如何变化。
### 实验 4：JIT fallback 与解释器对照
SQL：
```sql
SET jit = off;
EXPLAIN (ANALYZE, VERBOSE)
SELECT sum((a + 1) * (a + 2))
FROM generate_series(1, 1000000) a;
SET jit = on;
SET jit_above_cost = 0;
EXPLAIN (ANALYZE, VERBOSE)
SELECT sum((a + 1) * (a + 2))
FROM generate_series(1, 1000000) a;
```
断点：
```text
jit_compile_expr
ExecReadyInterpretedExpr
```
记录：`jit_compile_expr()` 返回 true 还是 false；`state->evalfunc` 最终指向哪里；`EXPLAIN` 中 JIT generation time 是否超过收益。
## 13. 讨论题
1. 为什么 PostgreSQL 不在每个 tuple 上递归解释表达式树，而是先编译成 `ExprEvalStep[]`？
2. `ExprEvalStep.resvalue` / `resnull` 指向 `FunctionCallInfo.args[i]` 有什么好处？有什么生命周期风险？
3. 为什么 `ExecEvalExprSwitchContext()` 只切 memory context，不顺手 reset？
4. `EEOP_QUAL` 和 `EEOP_BOOL_AND_STEP` 的 NULL 语义为什么不同？
5. JIT 编译失败为什么不是 ERROR？fallback 到解释器保留哪些语义？
6. 如果 extension 函数把返回的 `text *` 指针保存在 `fn_extra` 中，应该检查哪些 context 边界？
7. 为什么 `ExprEvalStep` 的 union 不能随意变大？
8. profile 中看到 `ExecInterpExpr` 很热时，如何区分 dispatch 成本、slot deforming 成本和函数内部成本？
## 14. 本节小结
本节唯一主问题是：
```text
表达式解释器如何把 ExprState steps 执行成 Datum / null，短生命周期内存在哪里 reset？
```
答案是一条生命周期链：
```text
ExecInitExpr / ExecInitQual
  -> ExecInitExprRec 编译表达式树
  -> ExprEvalPushStep 追加平坦 steps
  -> ExecReadyExpr 选择 JIT 或解释器
  -> ExecEvalExpr* 调 state->evalfunc
  -> ExecInterpExpr 按 opcode dispatch
  -> step 写 Datum/isnull 或跳转
  -> DONE_RETURN 返回最终 Datum/isnull
  -> 外层 ResetExprContext() 回收短命内存
```
核心状态和边界：
- `ExprState` 是 per-query、backend-local、会 mutate 的表达式程序。
- `ExprEvalStep` 是解释器指令，包含 opcode、输出目标和小型 inline 参数。
- `ExprContext` 携带当前 tuple、参数和 memory context。
- `Datum/isnull` 是所有表达式值和中间值的成对语义。
- `ecxt_per_tuple_memory` 是短命表达式结果的分配目标。
- `ResetExprContext()`、`ReScanExprContext()`、`FreeExprContext()` 是释放和 callback 边界。
正确性来自多层组合：`Datum` 必须和 `isnull` 成对解释；`EEOP_QUAL` 把 WHERE 的 NULL 合并成 false；slot deforming 通过 `FETCHSOME` / `VAR` 隔开物理 tuple；strict 函数由 caller step 先做 NULL 短路；JIT 和解释器共享同一套 step 语义；MemoryContext 管内存，非内存资源仍需 callback、ResourceOwner 或模块 cleanup。
诊断时要分清粒度：`EXPLAIN` 能看到节点时间、loops、rows 和 JIT summary；`pg_backend_memory_contexts` 能看到 context 级内存，不是每个 step；`pg_stat_user_functions` 能看到函数统计，但不知道短路跳过了哪些 step；`perf`、gdb 或临时计数器才能接近 opcode dispatch 和 reset 细节。
可迁移规律：
```text
hot path 中的通用语义通常不会靠运行期反复发现；
它会在初始化期被压成更低层的状态机。
状态机提高执行效率的代价，是 ownership、NULL 协议、jump target 和 cleanup 边界必须更精确。
```
对表达式解释器，长期有用的检查框架是：
```text
这段状态活到 query 结束，还是只活到当前 tuple？
这对 Datum/isnull 写到哪里，谁会读取？
这个 step 会不会因为 strict、qual 或 jump 被跳过？
如果返回 by-reference Datum，它指向哪个 context？
当前执行方法是解释器、fast path，还是 JIT？
外层是否会在正确边界 ResetExprContext()？
```
这些问题回答清楚后，表达式解释器就不是一大串 `EEOP_*`，而是一条可调试、可 profile、可推断生命周期的 executor hot path。
