# PostgreSQL JIT expression / deform boundary
## 课程定位
前置知识：已经理解 `PlannedStmt` 到 `EState` 的 executor startup 边界，知道 `ExprState` 是表达式执行态，知道 `TupleTableSlot` 通过 deform 把物理 tuple 展开成 `tts_values` / `tts_isnull`。 本节唯一主问题：
```text
LLVM JIT 何时接管 expression / deform，为什么诊断时必须把 JIT startup cost 与 plan cache 的 generic/custom 状态一起判断？
```
核心矛盾：
```text
JIT 希望把通用解释器、间接跳转和 tuple deform 分支替换成针对本次 plan 的 native code
  vs
生成 native code 本身有启动成本，而且 prepared statement 可能在 custom plan 与 generic plan 之间切换。
```
学完后应能判断：
```text
一个查询没有出现 JIT，是 planner 没设置 jitFlags、executor 没有 parent/EState、provider 没加载，还是 LLVM provider 自己 fallback。
一个 prepared statement 忽快忽慢，是执行代价变化、generic/custom 选择变化、JIT startup cost 重复出现，还是 deform 没有进入可编译边界。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。 本节只讲 expression evaluation 与 tuple deforming 进入 LLVM JIT 的边界。 不讲 LLVM IR 逐条生成细节。 不讲所有表达式 opcode。 不讲 planner cost 模型本身如何估算 CPU。 不讲未来 JIT cache 设计。 这些都可能重要，但会稀释本节唯一主问题。
---
## 1. 本节在总主线中的位置
04 目录前面已经建立了三条线。 第一条线是 executor 生命周期：
```text
ExecutorStart()
  -> InitPlan()
  -> ExecInitNode()
  -> ExecProcNode()
  -> ExecutorEnd()
```
第二条线是表达式执行：
```text
ExecInitExpr()
  -> ExprState.steps
  -> ExecReadyExpr()
  -> ExprState.evalfunc
```
第三条线是 prepared statement 与 plan cache：
```text
CachedPlanSource
  -> GetCachedPlan()
  -> custom CachedPlan / generic CachedPlan
  -> PlannedStmt
  -> Portal
  -> ExecutorStart()
```
本节把三条线接到一起。 JIT 不是一个独立的执行节点。 JIT 也不是 planner 之外的全局加速器。 在当前源码里，JIT 的入口点很具体：
```text
planner 在 PlannedStmt 上写 jitFlags；
ExecutorStart 把 jitFlags 放入 EState；
ExecReadyExpr 尝试 jit_compile_expr；
LLVM provider 把 ExprState.steps 编译成 native evalfunc；
tuple deform 只在 expression 编译的 FETCHSOME step 内被顺手编译。
```
这解释了两个线上常见现象。 第一个现象：`jit_tuple_deforming = on` 不等于所有 tuple deform 都会被 JIT。 它需要 `PGJIT_PERFORM`、`PGJIT_EXPR`、`PGJIT_DEFORM` 同时满足。 还需要表达式里出现可识别的 `EEOP_*_FETCHSOME`。 还需要 slot ops 和 tuple descriptor 在初始化期足够固定。 第二个现象：prepared statement 使用 generic plan 后，plan 复用了，但 JIT 编译成本仍可能每次执行出现。 原因是当前 JITed expression 不是缓存在 `CachedPlan` 里的长期 native code。 `EState` 和 `ExprState` 仍然是每次执行重新创建。 LLVM provider 的 JIT context 也挂在本次 `EState` 下。 所以诊断 JIT startup cost 时，不能只问“是不是 generic plan”。 也要问“这次 executor startup 是否重新构建了 ExprState 并触发 JIT”。 本节后续所有内容都围绕这个判断展开。
---
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
planner 根据 plan total_cost 和 JIT GUC 写 PlannedStmt.jitFlags；
executor startup 把 jitFlags 复制到 EState；
每个 ExprState ready 时先尝试 jit_compile_expr；
provider 成功时把 ExprState.evalfunc 改成 compiled thunk；
第一次执行 compiled thunk 时触发 LLVM emission 并改成真实 native function；
deform JIT 是 expression 编译遇到固定 FETCHSOME 时生成的子函数。
```
这个模型里有四个边界。 第一，规划边界。 `src/backend/optimizer/plan/planner.c` 在生成 `PlannedStmt` 时判断：
```text
top_plan->total_cost > jit_above_cost
```
满足后才设置 `PGJIT_PERFORM`。 随后再按 `jit_optimize_above_cost`、`jit_inline_above_cost`、`jit_expressions`、`jit_tuple_deforming` 设置更多 flags。 这里的判断依据是 planner cost。 它不是实际运行时间。 它也不是 LLVM 编译成本预测。 第二，执行器边界。 `ExecutorStart()` 只是把 `queryDesc->plannedstmt->jitFlags` 复制到 `estate->es_jit_flags`。 它不会马上编译所有表达式。 真正尝试发生在每个 `ExprState` ready 时。 第三，provider 边界。 `jit.c` 不包含 LLVM 细节。 它只负责加载 `jit_provider` 指定的 shared library，并通过 `JitProviderCallbacks.compile_expr` 转发。 provider 可以不可用。 provider 也可以拒绝某个表达式。 调用者必须接受 `jit_compile_expr()` 返回 false。 第四，deform 边界。 tuple deform 并不是独立从 slot 层全局接管。
LLVM expression compiler 在处理 `EEOP_INNER_FETCHSOME`、`EEOP_OUTER_FETCHSOME`、`EEOP_SCAN_FETCHSOME` 等 step 时，才会考虑生成 `slot_compile_deform()`。 如果缺少固定 `TupleDesc` 或固定 slot ops，它就调用 `slot_getsomeattrs_int()`。 系统 tension 可以压缩成：
```text
specialize 越深，每 tuple CPU 越低；
specialize 越深，startup cost、memory 生命周期、schema 防御和 plan cache 状态越难隐藏。
```
因此本节最重要的诊断句是：
```text
JIT 是否值得，不是只看 JIT block；
要同时看本次 plan 是 generic 还是 custom、本次 jitFlags 从哪里来、JIT startup 是否重复发生、表达式/deform 是否真在 hot path 上。
```
---
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/jit/jit.h` | `PGJIT_*` flags、`JitContext`、`JitInstrumentation`、provider callback 边界。 |
| 2 | `src/backend/optimizer/plan/planner.c` | `PlannedStmt.jitFlags` 如何由 `top_plan->total_cost` 与 JIT GUC 决定。 |
| 3 | `src/include/nodes/plannodes.h` | `PlannedStmt.jitFlags` 是 plan cache 交给 executor 的静态合同之一。 |
| 4 | `src/backend/utils/cache/plancache.c` | `GetCachedPlan()` 如何选择 generic/custom，并影响最终 `PlannedStmt` 来源。 |
| 5 | `src/backend/executor/execMain.c` | `ExecutorStart()` 如何把 `jitFlags` 放入 `EState.es_jit_flags`。 |
| 6 | `src/include/nodes/execnodes.h` | `ExprState.evalfunc`、`EState.es_jit_flags`、`EState.es_jit`、worker instrumentation。 |
| 7 | `src/backend/executor/execExpr.c` | `ExecInitExpr()`、`ExecPushExprSetupSteps()`、`ExecComputeSlotInfo()`、`ExecReadyExpr()`。 |
| 8 | `src/backend/executor/execExprInterp.c` | 解释器 fallback、fast path、`CheckExprStillValid()`。 |
| 9 | `src/backend/jit/jit.c` | provider 加载、`jit_compile_expr()` gating、fallback 到解释器。 |
| 10 | `src/backend/jit/llvm/llvmjit.c` | `LLVMJitContext` 创建、ResourceOwner cleanup、optimization/emission。 |
| 11 | `src/backend/jit/llvm/llvmjit_expr.c` | `llvm_compile_expr()` 如何编译 `ExprState.steps`，以及第一次执行时如何取 native function。 |
| 12 | `src/backend/jit/llvm/llvmjit_deform.c` | `slot_compile_deform()` 对 slot ops、tuple descriptor、列属性的特化。 |
| 13 | `src/backend/commands/explain.c` | EXPLAIN JIT block 如何汇总 leader 和 worker instrumentation。 |
| 14 | `src/backend/executor/execParallel.c` | 并行执行下 `es_jit_flags` 传播和 worker JIT instrumentation 合并。 |
推荐阅读顺序不是从 LLVM 文件开始。 先读 `jit.h`。 因为 flags 和 instrumentation 已经告诉你 PostgreSQL 把 JIT 当成什么状态。 再读 `planner.c`。 因为是否 JIT 先由 `PlannedStmt.jitFlags` 决定。 再读 `plancache.c`。 因为 prepared statement 的 generic/custom 选择会影响这次拿到哪一个 `PlannedStmt`。 最后读 `llvmjit_expr.c` 和 `llvmjit_deform.c`。 否则容易把 LLVM 能力误读成 executor 一定会使用的能力。 本节源码基线核对：
```text
cd /home/nail/postgres
git rev-parse --abbrev-ref HEAD
git rev-parse --short=12 HEAD
```
本地输出：
```text
master
0e1f1ed157e9
```
---
## 4. 关键数据结构与状态
本节核心状态都是 backend-local。 JITed function、`ExprState`、`EState`、`JitContext` 都不是跨 backend 共享状态。 并行 worker 也不会直接复用 leader 的 native function 指针。 worker 会拿到 plan 信息，在自己的进程中建立执行态和 JIT 状态。
### 4.1 `PlannedStmt.jitFlags`
`PlannedStmt` 定义在 `src/include/nodes/plannodes.h`。 本节只看一个字段：
| 字段 | 语义 |
| --- | --- |
| `jitFlags` | planner 已决定本计划是否执行 JIT，以及启用 expression、deform、inline、optimize 哪些能力。 |
`jitFlags` 是 plan 的一部分。 它不是 executor 动态评估出来的即时状态。 `planner.c` 中的核心判断是：
```c
result->jitFlags = PGJIT_NONE;
if (jit_enabled && jit_above_cost >= 0 &&
    top_plan->total_cost > jit_above_cost)
{
    result->jitFlags |= PGJIT_PERFORM;
    if (jit_expressions)
        result->jitFlags |= PGJIT_EXPR;
    if (jit_tuple_deforming)
        result->jitFlags |= PGJIT_DEFORM;
}
```
真实源码还会按阈值设置 `PGJIT_OPT3` 和 `PGJIT_INLINE`。 这意味着：
```text
jitFlags = GUC snapshot + planner cost + 本次 plan shape 的结果。
```
不要把 `jit=on` 理解成“执行器一定 JIT”。 `jit=on` 只是 planner 允许考虑 JIT。 真正进入执行器的是 `PlannedStmt.jitFlags`。
### 4.2 `PGJIT_*`
`src/include/jit/jit.h` 定义了本节要用的 flags：
| flag | 语义 |
| --- | --- |
| `PGJIT_PERFORM` | 本 plan 允许执行 JIT。没有它，后续全部无效。 |
| `PGJIT_OPT3` | LLVM 走更重的优化路径。 |
| `PGJIT_INLINE` | 尝试函数/操作符 inlining。 |
| `PGJIT_EXPR` | 尝试编译 expression evaluation。 |
| `PGJIT_DEFORM` | 在 expression JIT 内尝试编译 tuple deforming。 |
`PGJIT_DEFORM` 不是独立执行引擎。 当前 `jit_compile_expr()` 先检查 `PGJIT_EXPR`。 只有 expression 编译进入 provider，deform 才有机会在 `llvm_compile_expr()` 内被生成。 所以一个 plan 有 `PGJIT_DEFORM` 但没有 `PGJIT_EXPR`，对当前 expression 路径没有实际意义。 正常 planner 设置时，`jit_tuple_deforming` 与 `jit_expressions` 可以分别控制 flags。 但执行入口仍以 expression 编译为主线。
### 4.3 `EState.es_jit_flags` 与 `EState.es_jit`
`EState` 定义在 `src/include/nodes/execnodes.h`。 JIT 相关字段是：
| 字段 | 语义 |
| --- | --- |
| `es_jit_flags` | 本次 executor invocation 应执行哪些 JIT 操作。 |
| `es_jit` | 按需创建的 `JitContext`，保存本次 query 的 JIT state 和 instrumentation。 |
| `es_jit_worker_instr` | leader 合并 worker JIT instrumentation 的空间。 |
`ExecutorStart()` 执行：
```c
estate->es_jit_flags = queryDesc->plannedstmt->jitFlags;
```
这一行是 `PlannedStmt` 到 executor 运行态的边界。 注意它只复制 flags。 它不创建 `JitContext`。 `JitContext` 是 expression provider 第一次实际需要时创建的。 `FreeExecutorState()` 会在 `estate->es_jit` 存在时调用 `jit_release_context()`。 如果 ERROR 让 executor 没有走到正常 cleanup，LLVM provider 创建 context 时注册的 `ResourceOwner` 会兜底释放。
### 4.4 `ExprState`
`ExprState` 定义在 `src/include/nodes/execnodes.h`。 本节关注这些字段：
| 字段 | 语义 |
| --- | --- |
| `steps` / `steps_len` | expression tree 被初始化成的 flat opcode 序列。 |
| `evalfunc` | 真正执行表达式的函数指针，可指向解释器、检查 thunk、JIT thunk 或 native function。 |
| `evalfunc_private` | `evalfunc` 私有状态，解释器 fast path 或 compiled expression 都会用。 |
| `parent` | 所属 `PlanState`，JIT 需要通过它找到 `EState`。 |
| `resvalue` / `resnull` | 表达式结果存储位置。 |
| `resultslot` | projection 类表达式写入的结果 slot。 |
`ExecReadyExpr()` 是关键分岔点：
```c
if (jit_compile_expr(state))
    return;

ExecReadyInterpretedExpr(state);
```
成功时，`ExprState.evalfunc` 被 JIT provider 接管。 失败时，解释器接管。 这不是 ERROR 路径。 这是正常 fallback。
### 4.5 `JitContext`
`JitContext` 定义在 `src/include/jit/jit.h`：
| 字段 | 语义 |
| --- | --- |
| `flags` | 本 context 对应的 `PGJIT_*`。 |
| `instr` | code generation、deform、inlining、optimization、emission 的计时与函数数。 |
LLVM provider 使用 `LLVMJitContext` 扩展它。 `llvm_create_context()` 把 context 分配在 `TopMemoryContext`。 同时用 `ResourceOwnerRememberJIT()` 把它登记到当前 resource owner。 这样做的原因是 JITed machine code 和 LLVM object 不是普通 query context 里的小块内存。 它们需要 provider 自己释放。
### 4.6 `JitProviderCallbacks`
provider-independent 层只知道三个 callback：
| callback | 语义 |
| --- | --- |
| `reset_after_error` | ERROR 后恢复 provider 错误处理状态。 |
| `release_context` | 释放 provider-specific JIT context。 |
| `compile_expr` | 尝试编译一个 `ExprState`。 |
`jit_provider` 默认通常指向 `llvmjit`。 `jit.c` 会按需加载 shared library。 加载失败后会缓存失败状态。 因此某个 backend 中 provider 不可用时，不会每个表达式都反复尝试昂贵加载。
### 4.7 `ExprEvalStep` 的 FETCHSOME
`ExecPushExprSetupSteps()` 会根据表达式中的 `Var` 统计需要展开到第几个 attribute。 然后插入：
```text
EEOP_INNER_FETCHSOME
EEOP_OUTER_FETCHSOME
EEOP_SCAN_FETCHSOME
EEOP_OLD_FETCHSOME
EEOP_NEW_FETCHSOME
```
这些 step 的含义是：
```text
在真正读取 Var 之前，确保 slot 至少已经 deform 到 last_var。
```
`ExecComputeSlotInfo()` 会尝试判断 slot 是否 fixed。 如果 fixed，它会把 `TupleDesc` 和 `TupleTableSlotOps` 填入 step。 如果 slot 固定且是 virtual slot，函数直接返回 false。 因为 virtual slot 已经有 values/nulls，不需要 deform。 这一步对 deform JIT 至关重要。 LLVM provider 只有在看到 fixed ops 和 known desc 时，才有足够信息生成针对 tuple descriptor 的 deform 函数。
### 4.8 `TupleDesc` 与 slot ops
`slot_compile_deform()` 的输入是：
```text
LLVMJitContext *context
TupleDesc desc
const TupleTableSlotOps *ops
int natts
```
它只处理几类 slot ops：
```text
TTSOpsHeapTuple
TTSOpsBufferHeapTuple
TTSOpsMinimalTuple
```
遇到 `TTSOpsVirtual` 返回 NULL。 遇到未知 slot ops 也返回 NULL。 这个 NULL 不是错误。 它表示 provider 拒绝生成 specialized deform function。 调用方随后回到 generic `slot_getsomeattrs_int()`。
### 4.9 `CachedPlanSource` 的 plan cache 状态
plan cache 相关状态定义在 `src/include/utils/plancache.h`。 本节只看影响 JIT 判断的字段：
| 字段 | 语义 |
| --- | --- |
| `gplan` | 当前 generic `CachedPlan`，可能为空或失效。 |
| `generic_cost` | generic plan 的估算成本，未知时为 -1。 |
| `total_custom_cost` | 历史 custom plan 成本累计。 |
| `num_custom_plans` | 已构造 custom plan 次数。 |
| `num_generic_plans` | 已使用 generic plan 次数。 |
| `cursor_options` | 可强制 generic 或 custom。 |
这些字段不直接记录 JIT 编译时间。 但它们决定下一次 `GetCachedPlan()` 返回 generic 还是 custom。 而 generic/custom 的 plan shape 和 `top_plan->total_cost` 会决定 `PlannedStmt.jitFlags`。 这就是本节把 JIT startup cost 和 plan cache 状态放在一起讲的原因。
---
## 5. 主流程源码 walkthrough
本节主流程按时间推进。 不要从 LLVM IR 生成函数顶部线性读到尾。 先看 flags 如何生成，再看 executor 如何接收，再看 expression 如何尝试，再看 deform 如何作为子路径进入。
### 5.1 planner 写入 `PlannedStmt.jitFlags`
入口在 `src/backend/optimizer/plan/planner.c`。 `standard_planner()` 生成 `PlannedStmt` 后，会根据 `top_plan->total_cost` 和 JIT GUC 设置 flags。 伪流程是：
```text
top_plan = planner 生成的顶层 Plan
result = PlannedStmt

result->jitFlags = PGJIT_NONE
if jit_enabled
   and jit_above_cost >= 0
   and top_plan->total_cost > jit_above_cost:
       add PGJIT_PERFORM
       if cost > jit_optimize_above_cost:
           add PGJIT_OPT3
       if cost > jit_inline_above_cost:
           add PGJIT_INLINE
       if jit_expressions:
           add PGJIT_EXPR
       if jit_tuple_deforming:
           add PGJIT_DEFORM
```
这里有三个细节。 第一，判断用的是 `top_plan->total_cost`。 不是 estimated rows。 不是 actual time。 不是 expression 数量。 第二，`PGJIT_OPT3` 和 `PGJIT_INLINE` 是更重的启动成本开关。 它们可能降低 per-tuple CPU。 也可能在短查询里把大部分时间花在 startup 上。 第三，`PGJIT_DEFORM` 只是允许 deform JIT。 它不保证所有 deform 都能被编译。
### 5.2 plan cache 选择当前要执行的 plan
prepared statement 执行时，调用链会进入 `GetCachedPlan()`。 核心选择在 `choose_custom_plan()`：
```text
oneshot:
  custom
no boundParams:
  generic
no revalidation needed:
  generic
plan_cache_mode force_generic:
  generic
plan_cache_mode force_custom:
  custom
cursor option force:
  follow cursor option
num_custom_plans < 5:
  custom
generic_cost < avg_custom_cost:
  generic
else:
  custom
```
源码里前 5 次 custom 是一个重要阈值。 这意味着一个 prepared statement 的早期执行和后期执行可能使用不同计划来源。 如果参数分布强烈影响 plan shape，custom plan 的 `top_plan->total_cost` 可能跨过 `jit_above_cost`。 generic plan 的 `top_plan->total_cost` 也可能跨过，但它代表的是参数无关计划。 所以同一个 SQL 文本，甚至同一个 prepared statement 名称，不一定有稳定的 JIT 行为。 `GetCachedPlan()` 返回前还会给 `PlannedStmt->planOrigin` 打上 generic/custom 标签。 这个标签主要服务内部和诊断。 JIT 决策本身已经在对应的 plan 生成时写入 `jitFlags`。
### 5.3 `ExecutorStart()` 接收 JIT 合同
入口在 `src/backend/executor/execMain.c`。 `ExecutorStart()` 创建 `EState` 后，把计划合同复制到执行态：
```c
estate->es_jit_flags = queryDesc->plannedstmt->jitFlags;
```
这个阶段不会调用 LLVM。 也不会加载 provider。 也不会检查表达式是否可编译。 它只是把 planner 的决定带入本次 executor invocation。 这解释了一个诊断边界：
```text
看到 EState.es_jit_flags = PGJIT_NONE，
不要去 llvmjit_expr.c 找原因；
先回到 planner cost、GUC、generic/custom plan 来源。
```
### 5.4 `ExecInitExpr()` 建立表达式 steps
典型入口在 `src/backend/executor/execExpr.c`：
```text
ExecInitExpr()
  -> make ExprState
  -> ExecInitExprRec()
  -> ExprEvalPushStep()
  -> ExecReadyExpr()
```
`ExecInitExprRec()` 把表达式树转成 flat steps。 这些 steps 是解释器和 JIT provider 的共同输入。 JIT 并不重新理解 SQL parse tree。 它编译的是 executor 已经构造好的 `ExprEvalStep` 程序。 因此表达式 JIT 的边界不是 parser。 也不是 optimizer。 它是 executor expression state。
### 5.5 setup steps 决定 deform 边界
对 projection、qual、agg trans 等表达式，初始化期会先统计表达式里用到的 `Var`。 `expr_setup_walker()` 记录：
```text
last_inner
last_outer
last_scan
last_old
last_new
multiexpr_subplans
```
随后 `ExecPushExprSetupSteps()` 插入必要的 `FETCHSOME` step。 如果表达式只访问 `a`、`b` 两列，它不需要 deform 到第 30 列。 它只要求 slot 至少 deform 到 `last_var`。 这让 interpreter 和 JIT 都能避免无关列成本。 `ExecComputeSlotInfo()` 是 `FETCHSOME` 的关键补充。 它尝试判断 slot 类型和 tuple descriptor 是否固定。 固定时，step 里会记录：
```text
op->d.fetch.fixed = true
op->d.fetch.kind = tts_ops
op->d.fetch.known_desc = desc
```
不固定时，JIT provider 后面拿不到足够信息，只能走 generic deform。
### 5.6 `ExecReadyExpr()` 决定 JIT 还是解释器
`ExecReadyExpr()` 非常短。 它的意义很大：
```c
if (jit_compile_expr(state))
    return;

ExecReadyInterpretedExpr(state);
```
这里的 `return` 表示 provider 已经设置好 `ExprState.evalfunc`。 如果 JIT 不可用、不该用、provider 拒绝、表达式没有 parent，都会落到解释器。 解释器路径 `ExecReadyInterpretedExpr()` 还会选择简单 fast path。 例如常量、简单 Var、简单 assign Var 等场景可以绕过完整解释器启动。 因此“没有 JIT”不等于“最慢解释器路径”。 这也解释了为什么低成本表达式强行 JIT 不一定更快。
### 5.7 `jit_compile_expr()` 的 provider-independent gating
入口在 `src/backend/jit/jit.c`。 `jit_compile_expr()` 先做三层检查：
```text
if !state->parent:
  return false
if !(es_jit_flags & PGJIT_PERFORM):
  return false
if !(es_jit_flags & PGJIT_EXPR):
  return false
```
第一层说明 standalone expression 当前不 JIT。 源码注释给出原因：没有关联的 `PlanState` / `EState` 时，生成的函数缺少 executor shutdown callback，可能活到事务结束并带来内存问题。 第二层说明 planner 必须先允许 JIT。 第三层说明 expression JIT 是 deform JIT 的外层入口。 通过这些检查后，`provider_init()` 尝试加载 provider。 如果成功，调用：
```text
provider.compile_expr(state)
```
如果失败，返回 false。 调用者继续解释器路径。
### 5.8 provider 加载不是每次都重试
`provider_init()` 有两个 static 状态：
```text
provider_successfully_loaded
provider_failed_loading
```
当 `jit_enabled` 为 false，直接返回 false。 当 provider shared library 不存在，记录 failed 并返回 false。 当加载依赖出错，函数会先把 failed 标记置 true，再调用 `load_external_function()`。 这样做避免每次表达式 ready 都反复探测 shared library。 对诊断来说：
```text
同一 backend 中，provider 加载失败可能让后续本该 JIT 的表达式都直接 fallback。
```
如果需要确认 provider 可用，可以调用 SQL 函数 `pg_jit_available()`。 它会触发 provider init。
### 5.9 `llvm_compile_expr()` 创建或复用 JIT context
LLVM provider 入口在 `src/backend/jit/llvm/llvmjit_expr.c`。 `llvm_compile_expr()` 先要求 `parent` 存在。 然后通过 parent 找到 `EState`：
```text
parent = state->parent
if parent->state->es_jit:
  context = existing es_jit
else:
  context = llvm_create_context(parent->state->es_jit_flags)
  parent->state->es_jit = &context->base
```
一个 query execution 通常共用一个 `JitContext`。 多个 expression 可以把 IR 放入同一个 mutable module。 这样可以减少重复 emission 成本，也给 LLVM 优化提供更大上下文。 但是 context 仍属于本次 executor invocation。 它不是 `CachedPlan` 的一部分。
### 5.10 expression 编译不立即取 native pointer
`llvm_compile_expr()` 会生成 `evalexpr` 函数的 LLVM IR。 生成结束后，它不立刻 emit 成可执行机器码。 源码中的注释说明：
```text
不要立即 emit；
第一次真正执行表达式时再 emit；
这样可以把多个函数一起 emit，避免反复 remap 和 LLVM overhead。
```
因此 provider 设置的是一个 thunk：
```text
state->evalfunc = ExecRunCompiledExpr
state->evalfunc_private = CompiledExprState(context, funcname)
```
第一次执行时：
```text
ExecRunCompiledExpr()
  -> CheckExprStillValid()
  -> llvm_get_function(context, funcname)
  -> state->evalfunc = native function
  -> native function(state, econtext, isNull)
```
所以 EXPLAIN JIT timing 中的 startup 不都发生在 `ExecInitExpr()`。 generation 主要来自 IR 生成。 emission 可能被第一次执行触发。 这对短查询尤其重要。
### 5.11 JIT expression 内部处理 `FETCHSOME`
`llvm_compile_expr()` 遍历 `ExprState.steps`。 遇到 `EEOP_*_FETCHSOME` 时，它先生成一个检查：
```text
if slot->tts_nvalid >= last_var:
  jump next step
else:
  fetch/deform needed
```
如果 step 里有固定 `tts_ops` 和 `desc`，并且 context flags 包含 `PGJIT_DEFORM`，它会调用：
```text
slot_compile_deform(context, desc, tts_ops, last_var)
```
如果返回非 NULL，就在 compiled expression 里调用这个 specialized deform function。 如果返回 NULL，则生成对 `slot_getsomeattrs_int(slot, last_var)` 的调用。 这就是 deform boundary 的准确位置：
```text
deform JIT 不是替换整个 slot API；
它只替换 expression FETCHSOME slow branch 中某个固定 tuple layout 的 deform 函数。
```
### 5.12 `slot_compile_deform()` 的接受与拒绝
入口在 `src/backend/jit/llvm/llvmjit_deform.c`。 它利用 `TupleDesc` 的编译期知识：
```text
固定列宽
alignment
NOT NULL
dropped column
missing attribute
NULL bitmap
tuple header layout
```
目标是减少 generic deform 中的大量分支。 但它明确拒绝一些情况：
```text
virtual tuple:
  return NULL
unsupported slot ops:
  return NULL
```
当前支持的主要 slot ops 是：
```text
TTSOpsHeapTuple
TTSOpsBufferHeapTuple
TTSOpsMinimalTuple
```
这不是 correctness 降级。 这是 performance fallback。 unsupported 情况仍然使用 generic deform。
### 5.13 执行期仍保留 schema 防御
解释器 first-call 路径会通过 `ExecInterpExprStillValid()` 调用 `CheckExprStillValid()`。 JIT first-call thunk `ExecRunCompiledExpr()` 也先调用 `CheckExprStillValid()`。 检查内容包括：
```text
Var 引用的 attribute 是否仍存在
attribute 是否被 drop
attribute type 是否和 plan 期一致
virtual generated column 是否异常出现
```
理想情况下，plan invalidation 会阻止过期 plan 被复用。 但 executor 仍有防御。 这说明 JIT 不是绕开 executor correctness 的捷径。 JIT 只是换了 expression evalfunc。 它仍遵守 executor 的 slot、tuple descriptor、schema validity 边界。
### 5.14 `ExecutorEnd()` 与 JIT context cleanup
正常 cleanup 走 `FreeExecutorState()`。 如果 `estate->es_jit` 非空：
```text
jit_release_context(estate->es_jit)
estate->es_jit = NULL
```
provider-specific `llvm_release_context()` 会释放 LLVM module、ORC resource tracker、handles，并从 ResourceOwner 中忘记 JIT context。 如果执行中 ERROR，没有走到正常 `FreeExecutorState()`，ResourceOwner callback 会调用：
```text
ResOwnerReleaseJitContext()
  -> jit_release_context(&context->base)
```
这条线是 JITed code lifetime 的关键。 不要把它理解成普通 `MemoryContextReset()` 可以完全处理的资源。
---
## 6. 生命周期 / ownership / cleanup
### 6.1 谁创建 `jitFlags`
`jitFlags` 由 planner 写入 `PlannedStmt`。 对 simple query，它随本次 planning 生成。 对 prepared statement custom plan，它随每次 custom planning 生成。 对 prepared statement generic plan，它随 generic plan 生成并保存在 `CachedPlan` 的 `PlannedStmt` 中。 这意味着 plan cache 状态会影响 JIT 行为。 不是因为 plan cache 保存了 native code。 而是因为 plan cache 决定这次执行拿到哪一个 `PlannedStmt.jitFlags`。
### 6.2 谁持有 `jitFlags`
`PlannedStmt` 持有静态 flags。 `ExecutorStart()` 把它复制到 `EState.es_jit_flags`。 之后 expression JIT 只看 `EState`。 如果你在 gdb 里看一个正在执行的 query，优先看：
```text
queryDesc->plannedstmt->jitFlags
queryDesc->estate->es_jit_flags
queryDesc->estate->es_jit
```
第一项说明计划合同。 第二项说明本次 executor invocation 的启用状态。 第三项说明是否已经实际创建 JIT context。
### 6.3 谁创建 JIT context
`llvm_compile_expr()` 按需创建。 它不是 `ExecutorStart()` 创建。 它也不是 `GetCachedPlan()` 创建。 创建时分配在 `TopMemoryContext`。 同时登记到 `CurrentResourceOwner`。 原因是 provider 需要处理 LLVM 内部资源和 mmap 后的 code object。 这些资源不能只依赖 executor per-query memory context 的 reset。
### 6.4 谁持有 native function
JITed function 属于 `LLVMJitContext` 的 module / handle。 `ExprState.evalfunc` 会在第一次执行后指向 native function。 但这个指针只在当前 backend、当前 JIT context 生命周期内有意义。 它不能放入 `CachedPlan`。 它不能跨 backend。 它不能被并行 worker 直接复用。 它也不能在 `ExecutorEnd()` 后继续使用。
### 6.5 谁释放
正常路径：
```text
ExecutorEnd()
  -> FreeExecutorState()
     -> jit_release_context()
        -> provider.release_context()
        -> pfree(context)
```
ERROR 路径：
```text
ResourceOwnerRelease()
  -> ResOwnerReleaseJitContext()
     -> jit_release_context()
```
backend exit 路径：
```text
proc_exit_inprogress:
  llvm_release_context() 避免重新进入 LLVM cleanup
  让进程退出处理剩余 OS 资源
```
这里的边界很重要：
```text
MemoryContext 管普通内存；
ResourceOwner 管 JIT context 这类需要错误路径兜底的外部资源；
provider.release_context 管 LLVM 内部对象。
```
### 6.6 generic plan 的 lifetime 与 JIT lifetime 不同
generic `CachedPlan` 可能在 backend 内长期存在。 它的 `PlannedStmt` 可以被多个 portal 执行复用。 但每次执行仍会新建 `EState` 和 `PlanState`。 表达式 `ExprState` 也会重新初始化。 JIT context 仍然按本次 execution 创建和释放。 因此：
```text
generic plan 可以省 planner cost；
generic plan 当前不能省 expression native code generation cost。
```
这句话是本节最容易被忽略的 ownership 边界。
---
## 7. 正确性机制层次
JIT 路径的正确性不是 LLVM 单独保证的。 它依赖多层 PostgreSQL 机制共同收口。
| 层次 | 机制 | 保证 | 不要误解为 |
| --- | --- | --- | --- |
| plan validity | plan cache invalidation | 过期语义下次重建 | 正在执行的 native function 会被跨执行复用 |
| executor startup | `PlannedStmt` 到 `EState` | 本次执行拿到 flags、snapshot、params、slots | `PlannedStmt` 本身可直接执行 |
| expression init | `ExprState.steps` | 表达式 tree 被转成执行程序 | JIT 重新做 parse/analyze |
| schema defense | `CheckExprStillValid()` | first-call 检查 Var 与 slot descriptor | 可以替代 plan invalidation |
| slot boundary | `TupleTableSlotOps` 与 `TupleDesc` | deform 的物理布局假设 | 所有 slot 都能 JIT deform |
| cleanup | `FreeExecutorState()` 与 ResourceOwner | 正常和 ERROR 路径释放 JIT context | 普通 MemoryContext reset 足够 |
| provider isolation | `jit.c` callback | LLVM 不进入 core binary hard dependency | provider 失败等于 query 失败 |
### 7.1 plan cache invalidation 与 schema check 是两道防线
正常情况下，schema 变化会让 cached plan 失效。 下一次 `GetCachedPlan()` 会 revalidate 或 replan。 但 executor expression 仍保留 `CheckExprStillValid()`。 这是防御式边界。 JIT first-call 也执行这个检查。 因此 JIT 不应该让过期 attribute type 静默读错。
### 7.2 slot ops 是物理布局合同
`slot_compile_deform()` 对 slot ops 很保守。 它只支持自己知道如何解释的 tuple representation。 对 virtual slot，根本没有 deform 工作。 对未知 slot ops，fallback。 这说明 deform JIT 的 correctness 前提是：
```text
known descriptor + known physical slot representation + fixed target natts
```
缺任何一项，都不能生成 specialized code。
### 7.3 provider failure 是性能降级，不是语义降级
`jit_compile_expr()` 返回 false 后，`ExecReadyExpr()` 调用解释器。 表达式语义不依赖 JIT。 JIT 只改变执行方式和成本结构。 这也是为什么 PostgreSQL 可以按需加载 `llvmjit`。 没有 provider 时，数据库仍能执行查询。
### 7.4 FATAL on LLVM OOM 的边界
`src/backend/jit/README` 解释了 LLVM OOM 处理。 PostgreSQL 不能安全地在任意 LLVM C++ 内部状态中抛普通 ERROR。 因此 LLVM 交互区会通过 `llvm_enter_fatal_on_oom()` / `llvm_leave_fatal_on_oom()` 保护。 当 LLVM 内部 OOM 或 fatal error 发生时，可能走 FATAL。 这不是普通表达式错误。 这是 provider isolation 与 backend safety 的取舍。
---
## 8. 错误路径 / 异常路径 / fallback
### 8.1 `jit_enabled = off`
planner 不设置 `PGJIT_PERFORM`。 `EState.es_jit_flags` 为 `PGJIT_NONE`。 `jit_compile_expr()` 在 flags 检查处返回 false。 解释器执行。 观测上没有 JIT block。
### 8.2 `jit_above_cost` 没跨过
即使 `jit=on`，如果：
```text
top_plan->total_cost <= jit_above_cost
```
planner 仍不设置 `PGJIT_PERFORM`。 这类问题不要去看 provider。 先看 EXPLAIN 中 plan cost 和当前 GUC。 注意阈值使用 `>`，不是 `>=`。
### 8.3 provider shared library 不存在
`provider_init()` 会构造：
```text
pkglib_path / jit_provider . DLSUFFIX
```
如果文件不存在，它记录 debug 信息，标记 `provider_failed_loading`，返回 false。 后续同一 backend 不再反复探测。 执行走解释器。 这通常表现为 planner 可能已经设置了 JIT flags，但 EXPLAIN 没有实际 JIT functions。
### 8.4 表达式没有 parent
`jit_compile_expr()` 对 `!state->parent` 直接返回 false。 源码注释的核心理由是 cleanup。 没有关联 `PlanState` 就没有可挂的 `EState`。 创建 one-off JIT context 会活到事务结束，可能放大内存和调试器成本。 这是一条 ownership 边界，不是 LLVM 不会编译该表达式。
### 8.5 provider 拒绝 deform
`llvm_compile_expr()` 只有在以下条件都满足时尝试 deform JIT：
```text
step 是 FETCHSOME
step 有 tts_ops
step 有 desc
context flags 包含 PGJIT_DEFORM
slot_compile_deform() 返回非 NULL
```
任何条件不满足，都调用 `slot_getsomeattrs_int()`。 查询仍正确。 只是 deform 仍走 generic C 路径。
### 8.6 virtual slot
`ExecComputeSlotInfo()` 如果确认 fixed slot 是 `TTSOpsVirtual`，直接认为不需要 deform step。 `slot_compile_deform()` 如果收到 virtual ops，也返回 NULL。 这不是 JIT 缺失。 virtual slot 已经是 values/nulls 数组。 没有物理 tuple header、null bitmap、alignment walk 需要展开。
### 8.7 first-call validity check 报错
如果 plan 期和执行期的 slot descriptor 不兼容，`CheckExprStillValid()` 可能报错：
```text
attribute has been dropped
attribute has wrong type
unexpected virtual generated column reference
```
JIT 路径不会绕过这些错误。 `ExecRunCompiledExpr()` 在取 native function 前会先检查。
### 8.8 LLVM OOM 或 fatal error
进入 LLVM API 前，provider 使用 fatal-on-OOM 保护。 如果 LLVM 内部无法安全返回，backend 可能 FATAL。 这类错误不是 SQL 表达式语义错误。 诊断时要看 server log 和 backend 重启现象。
### 8.9 parallel worker instrumentation 丢失的误读
并行 worker 中也可能 JIT。 leader 通过 `es_jit_worker_instr` 合并 worker instrumentation。 如果只看 leader 的 `estate->es_jit->instr`，会漏掉 worker 部分。 EXPLAIN 汇总路径在 `ExplainPrintJITSummary()`。 它会合并 leader 和 worker。
---
## 9. 成本、资源与跨模块传播
### 9.1 成本模型的第一层：planner cost gate
JIT 启用首先由 planner cost gate 控制。 默认值来自 `jit.c`：
```text
jit_above_cost = 100000
jit_inline_above_cost = 500000
jit_optimize_above_cost = 500000
```
这些是 cost units。 不是毫秒。 因此“查询实际跑了 2 秒但没有 JIT”可能完全合理。 如果 planner 估算低于阈值，JIT 不会开启。 反过来，“查询实际很快但出现 JIT startup”也可能发生。 如果估算成本很高，planner 会设置 flags。
### 9.2 成本模型的第二层：startup cost
JIT startup 包含多段：
```text
provider load
LLVM session/context 初始化
IR generation
deform function generation
inlining
optimization
emission
first-call native function lookup
```
EXPLAIN JIT block 会把这些分成：
```text
Generation
Deform
Inlining
Optimization
Emission
Total
```
注意 `Deform` 包含在 `Generation` 里。 `ExplainPrintJIT()` 源码明确不把 deform_counter 再加到 total。 否则会重复计算。
### 9.3 成本模型的第三层：per-tuple saving
JIT 的收益来自每 tuple 或每 expression 的重复执行。 expression JIT 可能减少：
```text
opcode dispatch
indirect branch
fmgr function call overhead
strict NULL branch
constant branch
slot fetch branch
```
deform JIT 可能减少：
```text
tuple descriptor lookup
null bitmap branch
alignment branch
variable-length offset walk
attnotnull / dropped / missing checks
unneeded column walk
```
收益随行数、表达式复杂度、访问列数、tuple layout、CPU branch predictor 状态增长。 如果查询只返回几行，startup cost 可能远大于 saving。
### 9.4 plan cache 影响成本判断
prepared statement 让问题多一层。 `GetCachedPlan()` 可能前 5 次生成 custom plan。 之后如果 generic plan 成本低于平均 custom cost，就转 generic。 这会带来三种 JIT 现象。 第一，custom plan 跨过 `jit_above_cost`，generic plan 没跨过。 早期执行有 JIT，后续没有。 第二，custom plan 没跨过，generic plan 跨过。 早期执行没有 JIT，后续转 generic 后有 JIT startup。 第三，二者都有 JIT。 generic plan 省下 planner cost，但每次 executor 仍要重新 JIT expression。 第三种最容易误诊。 看到 generic plan 不能推断 JIT startup 已经摊薄到零。 当前 native expression 不保存在 generic `CachedPlan` 中。
### 9.5 custom plan 的参数敏感性
custom plan 使用 `ParamListInfo` 中的参数值规划。 不同参数可能产生不同 plan shape。 例如：
```text
小范围参数 -> index scan -> estimated cost below jit_above_cost
大范围参数 -> seq scan + aggregate -> estimated cost above jit_above_cost
```
这会让同一个 prepared statement 的 JIT 行为随参数变化。 如果应用端只记录 SQL 模板，不记录参数分布，就很难解释 JIT block 为什么忽有忽无。
### 9.6 generic plan 的稳定性和盲点
generic plan 不使用具体参数值。 它的 `jitFlags` 稳定来自 generic planning 的估算。 这降低了 plan shape 抖动。 但也可能让某些参数执行明显不匹配。 JIT startup 也仍然在每次 execution 发生。 因此 generic plan 的优势是省 planning 和稳定计划。 不是复用 native JIT code。
### 9.7 资源压力传播
JIT 资源不只是一点 CPU。 它可能传播到：
| 资源 | 传播路径 |
| --- | --- |
| backend local memory | `LLVMJitContext`、module、generated IR、compiled object。 |
| executable memory | ORC JIT emission 后的 code section。 |
| CPU frontend | startup 阶段大量编译工作抢占执行时间。 |
| server log | provider load、LLVM fatal、debugging/profiling support 输出。 |
| EXPLAIN output | JIT query-level summary，不是节点级 attribution。 |
| ResourceOwner | ERROR cleanup 时需要释放 context。 |
这些资源大多是 backend-local。 不涉及 shared buffer、WAL 或 MVCC correctness。 但它们会改变 query latency 和 backend RSS。
### 9.8 与 fmgr 的边界
第 66 节讲过 fmgr 调用边界。 JIT expression 可能降低 fmgr 的间接调用成本，甚至在可用 bitcode 时尝试 inlining。 但 JIT 不改变 SQL 函数的语义协议。 `FunctionCallInfo`、strict、collation、NULL、SRF 等边界仍然存在。 当函数本体很重时，JIT 减少调用框架开销的收益可能很小。 当表达式里有大量简单内置操作符时，收益更明显。
### 9.9 与 TupleTableSlot 的边界
deform JIT 不是让 slot 消失。 slot 仍然保存：
```text
tts_values
tts_isnull
tts_nvalid
tts_tupleDescriptor
tts_ops
```
JIT deform 只是更快地把物理 tuple 展开到这些数组。 后续 `EEOP_*_VAR` 仍从 slot values/nulls 读。 如果表达式访问的列已经 valid，就不会重复 deform。 这也是 `tts_nvalid >= last_var` 检查的意义。
### 9.10 与 parallel executor 的边界
并行执行中，JIT flags 会序列化给 worker。 worker 在自己的 executor state 中创建 JIT context。 leader 不能把自己的 native function pointer 传给 worker。 worker instrumentation 最后回传并汇总。 因此并行查询的 JIT startup cost 可能在多个 worker 中重复出现。 这会放大短并行查询的启动成本。 但对长 analytic query，worker 并行执行的 per-tuple saving 也可能更大。
---
## 10. 观测与诊断入口
本节锚定的 runtime truth 是：
```text
同一个 prepared statement 的一次执行是否 JIT，取决于当前拿到的 PlannedStmt.jitFlags；
一次执行中 JIT startup 是否值得，取决于 startup timing 是否被足够多的 expression/deform hot path 摊薄。
```
### 10.1 `EXPLAIN (ANALYZE)` 的 JIT block
典型输出会包含：
```text
JIT:
  Functions: 8
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 2.100 ms (Deform 0.350 ms), Inlining 0.000 ms, Optimization 0.000 ms, Emission 3.800 ms, Total 5.900 ms
```
这能看到：
```text
是否真的创建了 JIT functions
expression/deform flags
startup 各阶段时间
```
它看不到：
```text
每个 expression 的 native code 是否收益为正
每个节点单独消耗多少 JIT startup
generic/custom 选择的完整历史
provider load 是否在本次 timing 中占多少
```
JIT block 是 query-level 汇总。 不要把它强行归因到某个 plan node。
### 10.2 `EXPLAIN EXECUTE` 推断 generic/custom
对 prepared statement：
```sql
PREPARE q(int) AS
SELECT count(*) FROM t WHERE k < $1;

EXPLAIN (ANALYZE, COSTS, SETTINGS) EXECUTE q(10);
```
常见推断方法：
```text
custom plan:
  Filter 或 Index Cond 中可能显示具体常量
generic plan:
  Filter 或 Index Cond 中可能保留 $1
```
这不是所有计划的绝对规则。 但常用于快速判断。 更稳的方式是同时看 `pg_prepared_statements` 中 generic/custom 计数。
### 10.3 `pg_prepared_statements`
`pg_prepared_statements` 能帮助观察 prepared statement 是否转向 generic。 重点看：
```text
name
statement
parameter_types
from_sql
generic_plans
custom_plans
```
这些字段解释 plan cache 状态。 它们不解释 JIT timing。 所以要和 EXPLAIN JIT block 合看。 诊断顺序建议：
```text
先看 custom_plans / generic_plans 是否变化；
再看 EXPLAIN EXECUTE 中 plan shape 和 cost；
再看 JIT block 是否出现；
最后比较 JIT startup 与总执行时间。
```
### 10.4 GUC 观测
本节常用 GUC：
```text
jit
jit_above_cost
jit_inline_above_cost
jit_optimize_above_cost
jit_expressions
jit_tuple_deforming
jit_provider
plan_cache_mode
```
`EXPLAIN (SETTINGS)` 只会显示非默认设置。 如果线上环境靠 session SET 改了 JIT 阈值，必须把 settings 纳入证据。 否则容易把 cost gate 误判成源码行为。
### 10.5 `pg_jit_available()`
SQL 函数 `pg_jit_available()` 会尝试加载 provider 并返回当前 backend 是否可用。 它回答的是：
```text
当前 backend 能否加载 JIT provider？
```
它不回答：
```text
某个 query 是否跨过 jit_above_cost？
某个 expression 是否可编译？
deform 是否会 specialized？
```
### 10.6 gdb 断点
源码跟读时可以断：
```text
standard_planner
GetCachedPlan
ExecutorStart
ExecReadyExpr
jit_compile_expr
llvm_compile_expr
slot_compile_deform
ExecRunCompiledExpr
FreeExecutorState
```
观察点：
```text
((PlannedStmt *) result)->jitFlags
queryDesc->plannedstmt->jitFlags
queryDesc->estate->es_jit_flags
state->parent
state->evalfunc
state->evalfunc_private
op->d.fetch.fixed
op->d.fetch.kind
op->d.fetch.known_desc
```
调试时不要在所有 expression 上无脑断 `llvm_compile_expr()`。 复杂计划会有很多 expression。 可以先用 EXPLAIN JIT Functions 数量估计规模。
### 10.7 perf / flamegraph
如果 JIT block 显示 startup 很小，但查询仍然 CPU 高，EXPLAIN 可能不够。 需要 `perf` 看：
```text
ExecInterpExpr
ExecRunCompiledExpr
slot_getsomeattrs_int
heap_deform_tuple
generated native symbols
fmgr call target
LLVM compilation frames
```
没有 JIT 时，hot path 常在解释器、fmgr、deform。 有 JIT 时，hot path 可能迁移到内置函数本体、hash/agg、memory access、cache miss。 不要因为 JIT 开启就认为 expression overhead 已经不是问题。
### 10.8 server log
`jit.c` 会用 DEBUG1 记录 provider 探测。 LLVM fatal 或 provider load error 会出现在 server log。 如果生产环境没有 DEBUG1，通常只能从 EXPLAIN 没有 JIT block、`pg_jit_available()`、安装包状态和 GUC 推断。
### 10.9 能直接看到、只能推断、不可见
直接看到：
```text
EXPLAIN JIT Functions / Options / Timing
EXPLAIN plan cost
pg_prepared_statements generic/custom counters
current_setting() 中的 JIT GUC
pg_jit_available()
```
只能推断：
```text
某次 prepared execution 是否因 generic/custom 切换改变 jitFlags
deform JIT 是否实际覆盖主要 tuple layout
JIT saving 是否超过 startup
provider load 成本是否污染第一次执行
```
几乎不可见：
```text
每个 ExprState 对 latency 的单独贡献
每个 generated native function 的执行次数
每个 deform specialized function 节省了多少 branch miss
```
这些需要源码插桩、perf 或 gdb。
---
## 11. 常见误区
### 11.1 “`jit=on` 就一定 JIT”
错误。 `jit=on` 只是 planner cost gate 的前提。 还需要 `top_plan->total_cost > jit_above_cost`。 还需要 expression 有 parent 和 `PGJIT_EXPR`。 还需要 provider 可用。
### 11.2 “`jit_tuple_deforming=on` 会 JIT 所有 deform”
错误。 deform JIT 是 expression JIT 的子路径。 只有 fixed slot ops 和 known tuple descriptor 的 `FETCHSOME` 才可能进入 `slot_compile_deform()`。 virtual slot 不需要 deform。 unsupported slot ops 会 fallback。
### 11.3 “generic plan 会复用 JITed native code”
当前实现下错误。 generic plan 复用 `PlannedStmt`。 `EState`、`PlanState`、`ExprState` 和 `JitContext` 仍然按执行创建。 所以 generic plan 可以省 planning cost，但不能直接省本次 expression JIT startup。
### 11.4 “EXPLAIN 的 JIT Total 应该加上 Deform”
错误。 `deform_counter` 已包含在 `generation_counter` 中。 `ExplainPrintJIT()` 计算 total 时明确不重复加 deform。
### 11.5 “没有 JIT 就一定走最慢解释器”
错误。 `ExecReadyInterpretedExpr()` 会为简单 expression 选择 fast path。 例如简单 Var、Const、Assign Var 等不一定走完整 opcode dispatch。
### 11.6 “JIT 是节点级指标”
错误。 JIT instrumentation 挂在 `EState` 的 JIT context 上。 EXPLAIN 输出是 query-level 汇总。 并行 worker 会合并到 summary。 不要把 JIT startup 直接分摊给某个节点，除非你有额外插桩证据。
### 11.7 “provider 加载失败会让 SQL 失败”
通常错误。 provider 不存在时，JIT fallback 到解释器。 但 LLVM 内部 OOM 或 fatal error 可能导致 FATAL。 要区分 provider unavailable 和 provider fatal。
---
## 12. 课堂实验
### 实验 1：观察 cost gate 与 JIT block
目标：确认 `jit_above_cost` 控制的是 plan cost gate，不是实际执行时间。 步骤：
```sql
CREATE TABLE jit_demo AS
SELECT g AS id, g % 1000 AS k, md5(g::text) AS payload
FROM generate_series(1, 2000000) AS g;

ANALYZE jit_demo;

SET jit = on;
SET jit_above_cost = 1000000000;

EXPLAIN (ANALYZE, COSTS, SETTINGS)
SELECT count(*) FROM jit_demo WHERE k + 1 > 500;

SET jit_above_cost = 10;

EXPLAIN (ANALYZE, COSTS, SETTINGS)
SELECT count(*) FROM jit_demo WHERE k + 1 > 500;
```
观察：
```text
第一次通常没有 JIT block。
第二次可能出现 JIT block。
比较 JIT Timing 与 Execution Time。
```
回到源码：
```text
planner.c:
  top_plan->total_cost > jit_above_cost

execMain.c:
  estate->es_jit_flags = plannedstmt->jitFlags

execExpr.c:
  ExecReadyExpr()
```
思考：
```text
如果把 jit_above_cost 设得很低，为什么短查询可能变慢？
```
### 实验 2：prepared statement 的 custom/generic 与 JIT
目标：观察 plan cache 状态如何改变同一 prepared statement 的 JIT 行为。 准备：
```sql
DROP TABLE IF EXISTS jit_skew;
CREATE TABLE jit_skew(id int, k int, payload text);

INSERT INTO jit_skew
SELECT g, CASE WHEN g < 1990000 THEN 1 ELSE g END, md5(g::text)
FROM generate_series(1, 2000000) AS g;

CREATE INDEX ON jit_skew(k);
ANALYZE jit_skew;

SET jit = on;
SET jit_above_cost = 100;
SET plan_cache_mode = auto;

PREPARE q(int) AS
SELECT count(*) FROM jit_skew WHERE k = $1 OR length(payload) > 100;
```
执行：
```sql
EXPLAIN (ANALYZE, COSTS, SETTINGS) EXECUTE q(1);
EXPLAIN (ANALYZE, COSTS, SETTINGS) EXECUTE q(2);
EXPLAIN (ANALYZE, COSTS, SETTINGS) EXECUTE q(3);
EXPLAIN (ANALYZE, COSTS, SETTINGS) EXECUTE q(4);
EXPLAIN (ANALYZE, COSTS, SETTINGS) EXECUTE q(5);
EXPLAIN (ANALYZE, COSTS, SETTINGS) EXECUTE q(6);

SELECT name, generic_plans, custom_plans
FROM pg_prepared_statements
WHERE name = 'q';
```
观察：
```text
前几次通常 custom_plans 增长。
后续是否转 generic 取决于成本。
EXPLAIN 中条件显示常量还是 $1，可辅助判断 generic/custom。
JIT block 是否出现，要和当前 plan cost 一起解释。
```
回到源码：
```text
plancache.c:
  choose_custom_plan()
  num_custom_plans < 5
  generic_cost < avg_custom_cost

planner.c:
  jitFlags per plan
```
思考：
```text
如果第 6 次转 generic，但 JIT Timing 仍出现，说明复用了什么，没有复用什么？
```
### 实验 3：deform JIT 与访问列数
目标：观察访问列数和 tuple layout 对 deform JIT 的影响。 准备：
```sql
DROP TABLE IF EXISTS jit_wide;
CREATE TABLE jit_wide (
  a int,
  b int,
  c int,
  d text,
  e text,
  f text,
  g int,
  h int,
  i text,
  j int
);

INSERT INTO jit_wide
SELECT g, g, g, md5(g::text), md5((g+1)::text), md5((g+2)::text),
       g, g, md5((g+3)::text), g
FROM generate_series(1, 1000000) AS g;

ANALYZE jit_wide;

SET jit = on;
SET jit_above_cost = 10;
SET jit_expressions = on;
SET jit_tuple_deforming = on;
```
对比：
```sql
EXPLAIN (ANALYZE, COSTS)
SELECT sum(a) FROM jit_wide;

EXPLAIN (ANALYZE, COSTS)
SELECT sum(j) FROM jit_wide;

SET jit_tuple_deforming = off;

EXPLAIN (ANALYZE, COSTS)
SELECT sum(j) FROM jit_wide;
```
观察：
```text
访问靠后的列更容易触发更多 deform 工作。
JIT block 中 Deforming true 不等于所有时间都在 deform。
关闭 jit_tuple_deforming 后，expression JIT 仍可能存在，但 deform specialized code 不再生成。
```
回到源码：
```text
execExpr.c:
  expr_setup_walker()
  ExecPushExprSetupSteps()
  ExecComputeSlotInfo()

llvmjit_expr.c:
  EEOP_*_FETCHSOME
  slot_compile_deform()

llvmjit_deform.c:
  TupleDesc specialization
```
---
## 13. 讨论题
1. 为什么 `jitFlags` 放在 `PlannedStmt`，而不是 `EState` 里完全重新计算？
2. 为什么 `jit_compile_expr()` 遇到 `!state->parent` 直接返回 false，而不是创建一个临时 context？
3. `PGJIT_DEFORM` 已经设置时，哪些条件仍会让 `slot_compile_deform()` 返回 NULL？
4. generic plan 复用了 `PlannedStmt`，为什么仍不能推断 JIT startup cost 被复用？
5. 如果一个 prepared statement 前 5 次 custom 没有 JIT，第 6 次 generic 出现 JIT，你会按什么顺序验证原因？
6. EXPLAIN JIT block 中 `Generation`、`Deform`、`Emission` 的边界分别对应哪些源码动作？
7. 为什么 `CheckExprStillValid()` 仍要在 JIT first-call 路径执行？
8. 并行 query 中 leader 和 worker 的 JIT instrumentation 为什么不能简单看 leader 的 `es_jit->instr`？
---
## 14. 本节小结
本节核心链路是：
```text
planner cost gate
  -> PlannedStmt.jitFlags
  -> plan cache generic/custom selection
  -> ExecutorStart copies flags
  -> ExecReadyExpr tries jit_compile_expr
  -> LLVM provider compiles ExprState.steps
  -> FETCHSOME may generate slot_compile_deform
  -> first execution emits native function
  -> ExecutorEnd / ResourceOwner releases JitContext
```
核心状态是三组。 第一组是 plan 级状态：
```text
PlannedStmt.jitFlags
CachedPlanSource generic/custom counters
CachedPlanSource generic_cost / total_custom_cost
```
它决定这次 execution 拿到怎样的 JIT 合同。 第二组是 executor 级状态：
```text
EState.es_jit_flags
EState.es_jit
ExprState.evalfunc
ExprState.steps
```
它决定本次表达式是否实际被 JIT 接管。 第三组是 provider 级状态：
```text
JitContext.instr
LLVMJitContext module/handles
CompiledExprState funcname/context
slot_compile_deform generated function
```
它决定 startup cost、native function lifetime 和 cleanup。 ownership 边界是：
```text
generic CachedPlan 可以长期存在；
JITed expression native code 属于本次 EState 的 JitContext；
ResourceOwner 负责 ERROR 兜底；
MemoryContext 不能单独表达 LLVM resource 生命周期。
```
fallback 边界是：
```text
planner 不设 flags -> interpreter
provider 不可用 -> interpreter
expression 无 parent -> interpreter
deform 信息不足 -> slot_getsomeattrs_int
LLVM fatal/OOM -> 可能 FATAL
```
观测边界是：
```text
EXPLAIN JIT block 能看 query-level startup 和 options；
pg_prepared_statements 能看 generic/custom 计数；
EXPLAIN plan cost 能解释 cost gate；
perf/gdb 才能解释每个 expression 或 deform 函数的实际 hotness。
```
可迁移的系统规律是：
```text
当一个优化把 runtime hot path 特化成 native 或 cached state 时，
诊断不能只看特化后的执行速度；
必须同时看特化的创建时机、生命周期、fallback 条件、以及上游 cache 是否真的复用了那个特化产物。
```
放到本节就是：
```text
JIT startup cost 是 executor invocation 的成本；
plan cache generic/custom 是 PlannedStmt 选择的状态；
两者相交但不等价。
```
哪些判断仍依赖 workload：
```text
JIT 是否收益为正，依赖行数、表达式复杂度、访问列位置、函数成本、CPU、LLVM 版本和参数分布。
```
哪些判断仍依赖版本：
```text
当前课程基于 master 0e1f1ed157e9。
未来如果 PostgreSQL 引入 expression JIT cache、planner-stage JIT 或 background optimization，
generic plan 与 JIT startup 的关系需要重新验证。
```
下一步如果继续追 CPU hot path，可以接着看 expression opcode、fmgr inline、slot deform 与 aggregate transition 的交界。
