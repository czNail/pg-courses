# PostgreSQL fmgr function call boundary
## 课程定位
前置知识：熟悉 executor 表达式求值、`ExprContext` per-tuple memory、 `Datum` / `isnull` 表示法，以及 C 函数通过
`PG_FUNCTION_ARGS` 接收参数的基本形式。 本节唯一主问题：
```text
fmgr 如何用 FmgrInfo、FunctionCallInfo、strict、set-returning function 和 memory context 调用 C 函数？
```
核心矛盾：PostgreSQL 要把 SQL 函数、操作符、类型 I/O、聚合 transition、 索引 support function 和扩展 C
函数都统一成一个可调用 ABI；但 executor hot path 又不能为每一行重复 catalog
lookup、动态链接、参数封装和内存清理。 学完后应能判断：一次函数调用的 catalog
元数据在哪里缓存，实际参数在哪里传递， strict NULL 短路由谁执行，SRF
多次返回如何跨调用保存状态，函数申请的内存应该随 tuple、query、`FmgrInfo` 还是 SRF multi-call
生命周期释放。 本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。
## 1. 本节在总主线中的位置
04 目录已经从 executor 生命周期、`PlanState`、`TupleTableSlot`、 `ExprContext`、per-tuple memory、ProjectSet
和观测入口一路走到表达式执行。 第 65 节计划讲 type input/output、varlena 和 detoast。 本节接在它后面，关注
executor 调用“一个函数”的最小边界。 这里的函数不只是 `SELECT myfunc(a)`。 操作符本质上也是函数。
类型输入输出函数也是函数。 排序比较、哈希、索引 support routine 也通过相似的 fmgr 边界被调用。 聚合
transition、combine、serialize、deserialize 也复用同一套调用约定。 因此本节不是扩展开发教程。
本节只回答一个 runtime 问题：executor 已经知道要调用某个函数时，如何把 catalog
元数据、参数、NULL、collation、resultinfo、统计和内存边界组织成一次安全而低成本的 C 调用。 后续第 67
节会继续追 operator / support function lookup。 第 68 节会把视角转到表达式解释器 step 如何把 `ExprState` 执行成
`Datum`。 第 72 节会专门讲函数 ERROR、SRF 中断和表达式短路后的 cleanup。
本节会提前触碰这些主题，但只保留服务 fmgr 调用边界的部分。 读本节时要把函数调用看成 executor hot path
的一个状态转换：
```text
catalog metadata
  -> FmgrInfo
  -> FunctionCallInfo
  -> PGFunction(fcinfo)
  -> Datum + isnull / ReturnSetInfo
```
这条链路的每一层都在减少下一层需要重新发现的信息。 也正因为层次多，很多 bug
并不是函数本体写错，而是 caller 把生命周期、NULL 协议或 SRF resultinfo 放错了位置。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
初始化期用 fmgr_info_cxt() 把 pg_proc 元数据和可调用地址缓存到 FmgrInfo；
执行期把本行参数写入 FunctionCallInfo.args[]；
caller 先处理 strict NULL 短路，再调用 fn_addr(fcinfo)；
普通函数通过 Datum/isnull 返回，SRF 通过 ReturnSetInfo 协商多行协议；
跨调用缓存放在 flinfo->fn_extra，并由 flinfo->fn_mcxt 决定生命周期。
```
本节的系统 tension 是：
```text
统一、可扩展、可被 catalog 驱动的函数 ABI
  vs
每 tuple 表达式求值必须足够轻，且内存和 NULL 语义不能泄漏给错误生命周期
```
PostgreSQL 的选择不是为每类函数写一套直接调用路径。 它把函数调用拆成两类状态：
第一类是相对稳定的函数元数据，放在 `FmgrInfo`。 第二类是每次调用变化的参数和结果通道，放在
`FunctionCallInfoBaseData`。 这种拆分让 catalog lookup、动态链接和函数属性解析可以在初始化期完成。
执行期只需要填参数、检查 NULL、调用函数指针、读取 `fcinfo->isnull`。 但是这个模型也带来边界要求。
`FmgrInfo` 不能放在比 `fn_mcxt` 更长寿的容器里。 `FunctionCallInfo` 的 `args[]` 不能被 callee
当成可长期保存的数组。 strict 函数的 NULL 短路是 caller 责任，不是 `FunctionCallInvoke()` 自动完成。 SRF
不能只返回一个 `Datum`，还必须通过 `ReturnSetInfo` 把“还有没有下一行”告诉 caller。
函数内部申请的内存如果要跨调用保存，不能随手放在 per-tuple context。 因此本节的 mental model 是：
```text
FmgrInfo 是“我是谁、怎么调用、可缓存什么”。
FunctionCallInfo 是“这一次调用给了什么、返回什么、调用现场是什么”。
MemoryContext 是“这些信息能活多久”。
strict 和 SRF 是 caller 与 callee 之间的协议，不是函数指针调用本身的魔法。
```
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/fmgr.h` | `FmgrInfo`、`FunctionCallInfoBaseData`、`LOCAL_FCINFO`、`InitFunctionCallInfoData`、`FunctionCallInvoke`。 |
| 2 | `src/backend/utils/fmgr/fmgr.c` | `fmgr_info()` / `fmgr_info_cxt()`、builtin fast path、`pg_proc` lookup、C symbol / handler、`FunctionCallN` wrappers、统计边界。 |
| 3 | `src/backend/executor/execExprInterp.c` | `EEOP_FUNCEXPR`、`EEOP_FUNCEXPR_STRICT`、`EEOP_FUNCEXPR_FUSAGE` 等表达式解释器 hot path。 |
| 4 | `src/backend/executor/execExpr.c` | 表达式初始化期如何准备 `FmgrInfo` 和 `FunctionCallInfo` storage。 |
| 5 | `src/backend/executor/execSRF.c` | `ExecMakeTableFunctionResult()`、`ExecMakeFunctionResultSet()`、`ReturnSetInfo` 与 SRF 多次调用协议。 |
| 6 | `src/include/nodes/execnodes.h` | `ReturnSetInfo`、`ExprDoneCond`、`SetFunctionReturnMode`。 |
| 7 | `src/include/funcapi.h` 与 `src/backend/utils/fmgr/funcapi.c` | `FuncCallContext`、`SRF_FIRSTCALL_INIT()`、materialize mode helper。 |
| 8 | `src/backend/executor/functions.c` | SQL-language function 的 `fn_extra` cache、`fn_mcxt` 和 sub-`QueryDesc` 生命周期。 |
| 9 | `src/backend/utils/adt/ruleutils.c` 以及 operator/function lookup 邻近文件 | 观察 catalog 元数据如何最终变成 OID、签名和可调用函数，但本节不展开 lookup 主题。 |
阅读顺序不要从 `fmgr.c` 顶部一路背到结尾。 先读 `fmgr.h`，因为边界都写在结构体和宏的注释里。 再读
`fmgr_info_cxt_security()`，确认 `FmgrInfo` 怎么从 OID 变成可调用地址。 然后读 `execExprInterp.c` 的 `EEOP_FUNCEXPR*`
分支，确认 hot path 不会重新 lookup。 最后读 `execSRF.c` 和 `funcapi.c`，理解为什么 SRF
不是普通函数调用的简单循环。 本节不把 `fmgr_builtins` 生成、动态库加载、语言 handler、SQL 函数 planner cache
全部展开。 这些都是相邻主题。 本节只拿它们说明同一个边界：`FmgrInfo`
把函数身份和调用地址稳定下来， `FunctionCallInfo` 把一次调用现场交给 callee。
## 4. 关键数据结构与状态
### 4.1 `PGFunction`
fmgr 眼里的 C 函数类型非常小：
```c
typedef Datum (*PGFunction) (FunctionCallInfo fcinfo);
```
这说明 executor 并不按 C 语言原型给函数传 `int32`、`text *` 或 `bool` 参数。 所有参数都先被编码成
`NullableDatum args[]`。 callee 再用 `PG_GETARG_*` 宏按自己的 SQL 类型解释这些 `Datum`。 这个边界让 SQL
函数签名可以由 catalog 描述。 同一个 C ABI 可以承载不同 SQL 类型、不同参数个数、不同 collation、不同
resultinfo。 代价是 caller 和 callee 必须共同遵守协议。 callee 如果绕过 `PG_ARGISNULL()` 直接读取 NULL
参数，就会把 SQL NULL 当成随机 `Datum`。 caller 如果忘记设置 `fcinfo->isnull = false`，上一轮调用的 NULL
状态可能污染当前结果。
### 4.2 `FmgrInfo`
`FmgrInfo` 是函数身份和稳定属性的缓存。 核心字段语义如下：
| 字段 | 本节语义 |
| --- | --- |
| `fn_addr` | 实际要调用的 C 函数指针，可能是 builtin 函数，也可能来自动态库或语言 handler。 |
| `fn_oid` | SQL 函数 OID，不一定等于 handler 的 OID。 |
| `fn_nargs` | catalog 中记录的输入参数数。 |
| `fn_strict` | `proisstrict`，表示只要输入有 NULL，caller 可以直接返回 NULL 或空 set。 |
| `fn_retset` | `proretset`，提示函数返回 set，但真正 SRF 协议还要看调用位置和 `ReturnSetInfo`。 |
| `fn_stats` | `track_functions` 相关阈值，决定是否走 function usage 统计路径。 |
| `fn_extra` | callee 或 handler 可使用的跨调用私有缓存。 |
| `fn_mcxt` | `fn_extra` 以及 subsidiary data 应该分配到的 memory context。 |
| `fn_expr` | 表达式树中的函数调用节点，给需要 parse-time 信息的函数使用。 |
`FmgrInfo` 是 backend-local 状态。 它里面的指针不能跨进程使用。 并行 worker 不能直接复用 leader 的 `FmgrInfo`
指针。 worker 需要通过计划序列化后在本进程重新初始化对应状态。 `fn_oid`
要最后填入，这不是风格问题。 `fmgr_info_cxt_security()` 的注释说明，有代码把 `fn_oid`
有效视为整个结构已有效。 因此如果初始化过程中 ERROR，半初始化的 `FmgrInfo` 不应该被误判为可调用。
`fn_extra` 是很多复杂行为的入口。 SQL 函数会把 `SQLFunctionCache` 挂在这里。 SRF helper 会把 `FuncCallContext`
挂在这里。 类型 I/O 或扩展函数也可能缓存解析后的辅助结构。 但 `fn_extra` 的生命周期完全取决于
`fn_mcxt`。 所以读到 `fn_extra` 时必须同时问：这个 `FmgrInfo` 是挂在表达式状态、per-query context、flinfo
cache，还是临时 stack wrapper？
### 4.3 `FunctionCallInfoBaseData`
`FunctionCallInfoBaseData` 是一次调用的现场。 核心字段语义如下：
| 字段 | 本节语义 |
| --- | --- |
| `flinfo` | 指向本次调用使用的 `FmgrInfo`。 |
| `context` | 调用上下文，例如 trigger、aggregate、window 或其它调用场景。 |
| `resultinfo` | 返回额外信息的通道，SRF 主要通过它传 `ReturnSetInfo`。 |
| `fncollation` | 函数使用的 collation。 |
| `isnull` | callee 必须设置的结果 NULL 标志；caller 必须在调用前重置。 |
| `nargs` | 本次实际参数个数。 |
| `args[]` | flexible array，保存每个参数的 `Datum` 和 `isnull`。 |
这个结构体叫 `BaseData` 是有意的。 v12 之后参数数组是 flexible array，caller 必须用 `SizeForFunctionCallInfo(nargs)`
或 `LOCAL_FCINFO(name, nargs)` 提供足够空间。 老代码如果只分配固定大小 `FunctionCallInfoData`，会因为没有 `args[]`
空间而破坏内存。 `InitFunctionCallInfoData()` 不初始化 `args[]`。 这让 caller 可以复用同一块
`FunctionCallInfo`，只更新本行参数。 但 caller 必须在每次实际调用前重置 `fcinfo->isnull`。
源码注释明确说，callee 可以假设 `isnull` 初始为 false。 `FunctionCallInvoke(fcinfo)` 只做一件事：
```c
(*fcinfo->flinfo->fn_addr)(fcinfo)
```
它不会检查 strict。 它不会填参数。 它不会切换 memory context。 它不会记录统计。 这些都由 caller 或更外层
wrapper 决定。 这正是 fmgr boundary 最容易误读的地方。
### 4.4 `NullableDatum`
`args[]` 的元素是 `NullableDatum`。 它把 `Datum value` 和 `bool isnull` 放在一起。 这意味着 raw `Datum` 本身没有 NULL
语义。 NULL 语义来自 `value + isnull + 函数 strict 属性 + 调用位置` 的组合。 对 pass-by-reference 类型，`Datum`
里通常是指针。 这个指针指向的对象属于某个 memory context、tuple slot、expanded object 或 detoast 结果。 fmgr
不拥有这些对象。 fmgr 只把指针传给函数。 因此函数如果要把参数保存到跨 tuple
状态，必须复制到合适的上下文。 只保存 `PG_GETARG_TEXT_PP()`
返回的指针通常是不安全的，除非你能证明它所在的 context 覆盖后续使用期。
### 4.5 `ReturnSetInfo`
SRF 调用的额外协议放在 `ReturnSetInfo`。 核心字段语义如下：
| 字段 | 本节语义 |
| --- | --- |
| `econtext` | SRF 所在表达式上下文。 |
| `expectedDesc` | caller 期望的 tuple descriptor。 |
| `allowedModes` | caller 能接受 `ValuePerCall`、`Materialize`、random access 等模式。 |
| `returnMode` | callee 实际选择的返回模式。 |
| `isDone` | `ValuePerCall` 模式下本次返回是单值、多值之一，还是结束。 |
| `setResult` | `Materialize` 模式下 callee 填入的 tuplestore。 |
| `setDesc` | `Materialize` 模式下实际返回 tuple descriptor。 |
`ReturnSetInfo` 不是函数返回值本身。 普通 C 返回值仍然是 `Datum`。 它是 caller 和 callee 之间关于“这个 Datum
是否还有后续”的 side channel。 如果函数声明 `RETURNS SETOF`，但调用位置没有传
`ReturnSetInfo`，正确行为通常是 ERROR。 这避免 callee 在不知道 caller 能否消费多行的情况下返回 set。
### 4.6 `FuncCallContext`
`FuncCallContext` 是写 C SRF 时常见的 helper 状态。 `SRF_FIRSTCALL_INIT()` 会在 `flinfo->fn_mcxt` 下建立或绑定 multi-call
context。 `SRF_PERCALL_SETUP()` 取回同一个 context。 `SRF_RETURN_NEXT()` 设置 `ReturnSetInfo.isDone = ExprMultipleResult`
并返回一个值。 `SRF_RETURN_DONE()` 设置结束状态并清理 multi-call 状态。 它的关键字段包括：
| 字段 | 本节语义 |
| --- | --- |
| `call_cntr` | 已返回次数。 |
| `max_calls` | 可选上限。 |
| `user_fctx` | 扩展作者自己的跨调用状态。 |
| `multi_call_memory_ctx` | 多次调用之间保留状态的 context。 |
| `tuple_desc` | 返回 composite 时使用的 descriptor。 |
这里最重要的不是宏名，而是生命周期。 SRF 每行调用之间会 reset per-tuple context。 如果跨调用状态放在
per-tuple context，下一次调用前就可能失效。 所以 SRF helper 把多次调用状态挂到 `fn_extra` 和 `fn_mcxt` 这条线。
### 4.7 `fn_stats` 与 function usage
`FmgrInfo.fn_stats` 决定函数调用是否需要统计。 表达式解释器会在普通 fast path 和统计 path 之间区分 opcode。
例如 `EEOP_FUNCEXPR` 直接调用函数。 `EEOP_FUNCEXPR_FUSAGE` 会进入 `ExecEvalFuncExprFusage()`，在调用前后执行
`pgstat_init_function_usage()` 和 `pgstat_end_function_usage()`。 这说明 `track_functions` 不是纯粹的 view 层开关。
它会改变 hot path 的 opcode 选择和每次调用成本。
因此诊断高频小函数时，必须把函数统计开销算进观测扰动。
## 5. 主流程源码 walkthrough
主流程分两段看。 第一段是初始化期，把函数 OID 变成 `FmgrInfo`。 第二段是执行期，把一行参数写入
`FunctionCallInfo` 并调用 `fn_addr`。
### 5.1 初始化期：从 OID 到 `FmgrInfo`
典型入口是：
```text
fmgr_info(functionId, finfo)
  -> fmgr_info_cxt(functionId, finfo, CurrentMemoryContext)
     -> fmgr_info_cxt_security(functionId, finfo, mcxt, false)
```
表达式初始化、操作符初始化、聚合函数初始化和类型 I/O 初始化都会走类似路径。 `fmgr_info()` 使用 caller
当前 memory context 作为 `fn_mcxt`。 如果 `FmgrInfo` 本身会活得更久，应使用 `fmgr_info_cxt()` 指定更长寿的
context。 `fmgr_info_cxt_security()` 先把 `fn_oid` 设成 `InvalidOid`。 然后把 `fn_extra` 设为 `NULL`。 再设置 `fn_mcxt`。
它这样做是为了让半初始化对象在 ERROR 后不被误用。 随后有一条 builtin fast path。 如果函数 OID 在
`fmgr_builtins` 里，`FmgrInfo` 可以直接拿到 `fn_addr`、 `fn_nargs`、`fn_strict`、`fn_retset`，不需要查 `pg_proc`。 非
builtin 函数则查 `pg_proc` syscache。 从 `Form_pg_proc` 读取 `pronargs`、`proisstrict`、`proretset` 等字段。
再根据语言、动态库 symbol、handler 或 validator 路径设置 `fn_addr`。 本节只关心最终结果：
```text
FmgrInfo.fn_addr   可调用入口
FmgrInfo.fn_strict NULL 输入是否可短路
FmgrInfo.fn_retset 是否声明返回 set
FmgrInfo.fn_mcxt   子缓存生命周期
```
离开初始化期后，executor hot path 不应该再查 `pg_proc` 来判断 strict。 这就是 `FmgrInfo` 的价值。
### 5.2 表达式初始化期：准备 `FunctionCallInfo` storage
表达式执行不是每次现造 `FunctionCallInfo`。 `execExpr.c` 的 `ExecInitFunc()` 会先检查 `ACL_EXECUTE` 并调用
`InvokeFunctionExecuteHook(funcid)`，再分配 `FmgrInfo` 和 `SizeForFunctionCallInfo(nargs)` 大小的参数工作区。 它调用
`fmgr_info()`、设置 `fn_expr`，再用 `InitFunctionCallInfoData()` 绑定 collation 和参数个数。 常量参数直接预填到
`fcinfo->args[]`，非常量参数追加表达式 step，让执行期把结果写入同一数组。 对普通函数，解释器可以直接跳到
`EEOP_FUNCEXPR`。 对 strict 函数，会选择 `EEOP_FUNCEXPR_STRICT` 或一、二参数特化 opcode。 对需要统计的函数，会选择
`EEOP_FUNCEXPR_FUSAGE` 或 strict 统计 path。 这些 opcode 是成本模型的一部分。
它们把“每次调用前该做哪些工作”提前在初始化期决定。 这样执行期就不需要反复判断所有可能性。
调用链可以简化成：
```text
ExecInitExpr()
  -> 初始化函数表达式 step
  -> fmgr_info_cxt()
  -> InitFunctionCallInfoData()
  -> 选择 EEOP_FUNCEXPR / STRICT / FUSAGE opcode
```
`ExecInitFunc()` 还会拒绝 `flinfo->fn_retset`，因为普通 function expression step 不负责消费 set。 SRF 要进入
`SetExprState`、`ProjectSet` 或 table function 路径。 具体函数名会随表达式种类变化。 阅读源码时不要被 helper
名字分散。 只要看到某个 step 最终携带 `op->d.func.finfo` 和 `op->d.func.fcinfo_data`， 它就进入了本节这条 fmgr 边界。
### 5.3 执行期：非 strict 普通函数
`execExprInterp.c` 的 `EEOP_FUNCEXPR` 展示了最短 hot path。 伪代码是：
```c
fcinfo->isnull = false;
d = op->d.func.fn_addr(fcinfo);
*op->resvalue = d;
*op->resnull = fcinfo->isnull;
```
注意这里没有 `FunctionCallInvoke(fcinfo)` 宏。 解释器直接调用 `op->d.func.fn_addr(fcinfo)`。
这不是语义差异，只是减少一层访问。 在进入这个分支前，参数表达式已经把值写入 `fcinfo->args[]`。
这个分支做的事情很少。 它重置结果 NULL 标志。 调用 C 函数。 把 `Datum` 和 `isnull` 写回 expression step
的结果槽。 如果 callee 在 per-tuple context 分配临时对象，后续 `ResetExprContext()` 会批量清理。 如果 callee 返回
pass-by-reference 指针，caller 必须知道这个指针能活多久。 fmgr 不会自动复制返回值。
### 5.4 执行期：strict NULL 短路
strict 函数的关键点是：caller 负责不调用函数。 `EEOP_FUNCEXPR_STRICT` 对多参数函数逐个检查 `args[i].isnull`。
只要发现 NULL，就把结果设置为 NULL，然后跳过函数调用。 一参数和二参数有特化 opcode：
```text
EEOP_FUNCEXPR_STRICT_1
EEOP_FUNCEXPR_STRICT_2
```
这是 hot path 优化。 很多操作符和函数是一到两个参数。 避免循环可以减少分支和指令数。 strict
的语义压缩成：
```text
if any argument is NULL:
    result is NULL
    do not call fn_addr
else:
    call fn_addr
```
这个语义对普通 scalar 函数很直观。 对 SRF 要更小心。 如果 strict set-returning function 的输入为 NULL，caller
通常表现为空 set 或 NULL 结果，具体取决于调用路径和函数是否真的处在 set 协议中。 `execSRF.c` 的 table
function 路径会在 strict NULL 时跳到 `no_function_result`。 这不是 callee 自己决定的。 如果你在 C 函数里写：
```c
if (PG_ARGISNULL(0))
    ...
```
但 SQL catalog 把函数标成 `STRICT`，那么 NULL 输入时这段代码根本不会执行。
这是扩展函数调试中非常常见的误判。
### 5.5 执行期：function usage 统计 path
当 `track_functions` 要统计某类函数时，表达式 step 不走最短 path。 `ExecInitFunc()` 的判断是
`pgstat_track_functions <= flinfo->fn_stats` 时走 fast path，否则走 FUSAGE path。 builtin / internal 函数通常把
`fn_stats` 设成 `TRACK_FUNC_ALL`，C 和 SQL 函数通常设成 `TRACK_FUNC_PL`，其它语言函数通常设成 `TRACK_FUNC_OFF`。
因此 `track_functions = all` 才会统计 C/SQL 函数，而 PL 类函数在 `pl` 或 `all` 下都可能被统计。 FUSAGE path 走
`EEOP_FUNCEXPR_FUSAGE` 或 `EEOP_FUNCEXPR_STRICT_FUSAGE`，该 path 会：
```text
pgstat_init_function_usage(fcinfo, &fcusage)
  -> 调用函数
pgstat_end_function_usage(&fcusage, finalize)
```
统计对象不是 executor node。 它按函数聚合。 这解释了为什么 `pg_stat_user_functions`
能看到函数级调用次数和耗时， 而 EXPLAIN 的节点耗时看到的是节点包含的整体表达式成本。
这也解释了观测扰动。 高频、低成本函数在开启 `track_functions` 后，每次调用都会多做计时和统计更新。
对慢函数，这个成本可以忽略。 对简单比较函数、类型输入函数或小扩展函数，这个成本可能改变 profile
形状。
### 5.6 直接 fmgr wrapper：`FunctionCall1` 到 `FunctionCall9`
`fmgr.c` 还提供一组便利 wrapper。 例如：
```text
FunctionCall1Coll(flinfo, collation, arg1)
FunctionCall2Coll(flinfo, collation, arg1, arg2)
OidFunctionCall3Coll(functionId, collation, arg1, arg2, arg3)
```
这些 wrapper 通常使用 `LOCAL_FCINFO()` 在栈上准备 `FunctionCallInfo`，再调用 `InitFunctionCallInfoData()`，填入参数并调用
`FunctionCallInvoke(fcinfo)`。 但要注意两组约束：`DirectFunctionCall*` 没有 `FmgrInfo`，callee 不能依赖 `flinfo`；
`FunctionCall*` / `DirectFunctionCall*` 这类便利 wrapper 明确假设参数和结果都不是 NULL，函数返回 NULL 会
`elog(ERROR)`。 它们适合非表达式解释器路径，例如类型 I/O、catalog helper、索引 support routine 或 utility
代码中偶发调用函数。 这些 wrapper 的边界和表达式 hot path 相同。 区别是它们不是按 executor step
预编译出来的。 因此它们可能每次都要准备 `LOCAL_FCINFO`。 如果传入的是 function OID wrapper，还可能临时调用
`fmgr_info()`。 热路径中应尽量复用 `FmgrInfo`，需要 NULL 语义时不要随手选这些 non-null convenience wrapper。
### 5.7 `fn_expr`：函数需要知道自己被如何调用
`FmgrInfo.fn_expr` 保存表达式树中的函数调用节点。 注释说它更像 parse-time argument
information，而不是函数自身属性。 但放在 `FmgrInfo` 里对 callee 更方便。 函数可以通过 helper
读取调用表达式中的 typmod、collation、常量参数或上下文信息。 这对多态函数、record 返回类型、类型 I/O
和一些扩展函数很重要。 但 `fn_expr` 不是 planner 或 executor 任意状态的后门。 它只在 caller 设置时有效。
直接用 `FunctionCallN` wrapper 调函数时，`fn_expr` 可能是 NULL。 扩展函数不能假设它总存在。
### 5.8 `fn_mcxt` 与 `fn_extra`：跨调用缓存
很多函数第一次调用时需要初始化额外状态。 例如解析配置、准备 tuple descriptor、初始化 SQL function cache。
这些状态不应每行重复建立。 fmgr 给 callee 的长期槽位是：
```text
fcinfo->flinfo->fn_extra
fcinfo->flinfo->fn_mcxt
```
典型模式是：
```c
if (fcinfo->flinfo->fn_extra == NULL)
{
    MemoryContext oldcxt;
    oldcxt = MemoryContextSwitchTo(fcinfo->flinfo->fn_mcxt);
    fcinfo->flinfo->fn_extra = build_cache();
    MemoryContextSwitchTo(oldcxt);
}
```
这里的关键不是代码模板，而是 owner。 `fn_extra` 的 owner 是这个 `FmgrInfo`。 `fn_mcxt` 的 owner 是创建 `FmgrInfo`
的上层对象。 如果 `FmgrInfo` 挂在表达式状态下，cache 通常随 query / plan 生命周期释放。 如果 `FmgrInfo`
挂在更长寿的全局 cache 下，cache 也会更长寿，必须处理 invalidation 或版本变化。
### 5.9 SQL-language function：`functions.c` 的 `SQLFunctionCache`
SQL 函数本身也通过 fmgr 被调用。 它的 handler 会在 `FmgrInfo.fn_extra` 下挂 `SQLFunctionCache`。 `functions.c`
的注释明确说：
```text
SQLFunctionCache is subsidiary data for a single FmgrInfo struct.
It is pointed to by fn_extra and allocated in fn_mcxt.
```
这给本节一个很好的 ownership 例子。 SQL 函数内部可能需要解析、rewrite、plan 或缓存多个 query。
这些不是普通 per-tuple 临时对象。 它们属于这个 SQL 函数调用点的 `FmgrInfo`。 如果函数第一次执行时初始化
cache，后续同一 `FmgrInfo` 的调用可以复用。 如果相关 catalog 或 plan 失效，cache 需要通过 dependency / callback /
invalidation 路径失效或重建。 本节不展开 SQL 函数执行器子查询生命周期。 这里只强调：`fn_extra + fn_mcxt` 是
handler 把复杂函数实现接入 fmgr 的扩展槽。
### 5.10 SRF in FROM：`ExecMakeTableFunctionResult()`
`FROM generate_series(...)` 这类 table function 走 `execSRF.c`。 这条路径会建立 `ReturnSetInfo`。 它设置：
```text
allowedModes = ValuePerCall | Materialize | Materialize_Preferred
randomAccess 时增加 Materialize_Random
returnMode = ValuePerCall
econtext = 当前 ExprContext
expectedDesc = caller 期望的 TupleDesc
```
它还会创建或 reset 一个单独的 `argContext`。 源码注释解释得很直接： `FunctionCallInfo` 需要跨 `ValuePerCall`
的多次调用存活。 函数参数也需要比 per-tuple context 更长寿。 但是 caller 的 `CurrentMemoryContext` 往往是
query-lifespan context，不能让每次 table function 都往里面泄漏。 所以 executor 要求 caller 传入可 reset 的
`argContext`。 主流程可以压缩成：
```text
MemoryContextReset(argContext)
MemoryContextSwitchTo(argContext)
构造 ReturnSetInfo
分配 FunctionCallInfo
计算函数参数到 fcinfo->args[]
strict NULL 检查
MemoryContextSwitchTo(econtext->ecxt_per_tuple_memory)
循环调用函数
  ResetExprContext(econtext)
  fcinfo->isnull = false
  rsinfo.isDone = ExprSingleResult
  result = FunctionCallInvoke(fcinfo)
  根据 rsinfo.returnMode / isDone 消费结果
```
这里有两个内存层次。 `argContext` 保证参数和 `FunctionCallInfo` 能跨 SRF 多次返回。 `ecxt_per_tuple_memory`
保证每一次调用内部泄漏的短命内存能被 reset。 这正是 SRF 比普通函数复杂的主要原因。
### 5.11 SRF in targetlist：`ProjectSet`
targetlist 中的 SRF 现在由 `ProjectSet` 节点处理。 第 23 节已经讲过 `pending_srf_tuples`。 本节只补 fmgr 角度。
`ExecMakeFunctionResultSet()` 负责单个 set 表达式。 它同样使用 `ReturnSetInfo`。 它同样要处理 strict NULL。
它同样区分 `ValuePerCall` 和 `Materialize`，但 targetlist 路径只设置 `SFRM_ValuePerCall | SFRM_Materialize`，不设置
`SFRM_Materialize_Random` 或 `_Preferred`。 它嵌在 executor 的 tuple-by-tuple 协议里。 一个输入 tuple
可能产生多行输出。 因此函数调用状态不能简单随本次 `ExecProcNode()` 返回后丢掉。 这就是为什么
`SetExprState` 里有可跨调用保存的 `fcinfo`、`funcResultStore`、 `funcResultSlot` 等状态。 读 `ProjectSet`
时不要只看节点状态。 还要追到 `ReturnSetInfo.isDone`。 它才是 callee 告诉 caller“这个输入 tuple 的 set
是否结束”的协议字段。
### 5.12 `Materialize` SRF
SRF 不一定每次返回一行。 如果 caller 允许 `SFRM_Materialize`，callee 可以一次构造 tuplestore。 `funcapi.c` 的
`InitMaterializedSRF()` 会检查：
```text
resultinfo 必须是 ReturnSetInfo
allowedModes 必须包含 SFRM_Materialize
```
然后切换到 `rsinfo->econtext->ecxt_per_query_memory`，创建 tuple descriptor 和 tuplestore，填入 `rsinfo->setResult` 和
`rsinfo->setDesc`， 把 `returnMode` 设置为 `SFRM_Materialize`。 这条路径把函数内部的多行生成从“多次调用”变成“一次调用生成完整集合”。 对 caller
来说仍然是同一个 `ReturnSetInfo` 协议。 不同的是结果 ownership 变成 tuplestore。 tuplestore 可能使用
work_mem，可能 spill，可能需要 random access。 因此 SRF 调用边界会把资源压力传播到 tuplesort / tuplestore / temp
file 子系统。
## 6. 生命周期 / ownership / cleanup
### 6.1 谁创建 `FmgrInfo`
`FmgrInfo` 由 caller 创建。 表达式执行中通常由 `ExecInitExpr()` 或相关初始化 helper 创建。 聚合、索引、类型
I/O、SQL 函数 handler 也会在自己的初始化边界创建或复制它。 `fmgr_info()` 只填内容，不拥有 `FmgrInfo`
结构体本身。 结构体可以在 stack 上、palloc 内存里，或嵌在更大的 executor state 中。 但它的 subsidiary data
要放到 `fn_mcxt`。 因此 owner 不是 `fmgr.c`。 owner 是创建 `FmgrInfo` 的上层状态。
### 6.2 谁持有 `FmgrInfo`
表达式路径中，`FmgrInfo` 通常被表达式 step 或 `SetExprState` 持有。 它随 `ExprState` 和 `EState` 生命周期存在。
SQL 函数 cache 的 `FmgrInfo` 可能跟随外层函数调用点存在。 系统 cache 或 long-lived helper 也可能持有更长寿的
`FmgrInfo`。 诊断时不能只看 `fn_oid`。 还要确认这个 `FmgrInfo` 挂在哪个 memory context 下。 相同函数 OID
在同一个 query 中可能有多个 `FmgrInfo`，因为不同调用点的 `fn_expr`、collation 或 cache 不同。
### 6.3 谁释放 `FmgrInfo`
`FmgrInfo` 本身随其所在 memory context 释放。 fmgr 不提供通用 destructor。 如果 `fn_extra`
里挂的状态需要额外释放资源，通常要把资源绑定到 memory context callback、ResourceOwner、tuplestore owner 或 caller
的 cleanup 路径。 只靠 `pfree(fn_extra)` 不是完整模型。 PostgreSQL 中很多状态不是内存本身，而是 buffer
pin、tuple store file、SPI context、 snapshot、portal 或 catcache ref。 这些必须由对应 owner 收尾。
### 6.4 谁创建 `FunctionCallInfo`
`FunctionCallInfo` 由每个 caller 准备。 表达式 step 初始化期可能预分配。 SRF 路径可能在 `argContext` 中分配。
便利 wrapper 可能用 `LOCAL_FCINFO()` 在栈上分配。 必须按参数个数分配足够空间。 `SizeForFunctionCallInfo(nargs)`
是动态分配的边界。 `LOCAL_FCINFO(name, nargs)` 是栈上分配的边界。 不允许假设 `FunctionCallInfoBaseData`
固定大小。
### 6.5 谁释放 `FunctionCallInfo`
stack 上的 `LOCAL_FCINFO` 随函数返回失效。 palloc 的 `FunctionCallInfo` 随所在 memory context reset 或 delete 释放。
表达式里的 `FunctionCallInfo` 通常随 `ExprState` 释放。 SRF table function 的 `FunctionCallInfo` 放在可 reset 的
`argContext` 中。 `ValuePerCall` 尚未结束时不能 reset 掉它。 否则下一次调用会丢失参数数组或 resultinfo。
### 6.6 参数和返回值的 ownership
fmgr 不复制参数。 它只传 `Datum`。 pass-by-value 类型直接在 `Datum` 中。 pass-by-reference 类型是指针。
指针指向谁，取决于上游表达式、slot、detoast、类型输入函数或函数内部构造。 返回值同理。 callee
返回一个 pointer `Datum` 时，必须让这个对象至少活到 caller 消费完。 普通表达式结果通常只需要活到当前
tuple 投影或 qual 判断结束。 如果结果要进入 tuple slot、tuplestore、hash table、aggregate state 或 sort tuple， caller
或 callee 必须确保 materialize / copy 到更长寿的 context。
### 6.7 ERROR / abort 时谁兜底
普通函数调用发生 ERROR 时，PostgreSQL 通过 `longjmp` 离开当前调用栈。 C 函数不会收到普通的 finally 回调。
内存由当前 memory context 层次在上层 cleanup 时释放。 外部资源由 ResourceOwner、SPI cleanup、portal cleanup、executor
cleanup 或函数自己 注册的 callback 收尾。 因此函数内部不能把外部资源只藏在 C 局部变量里，指望正常
return 后释放。 一旦 ERROR，它的正常 return path 不会执行。 本节涉及的最小结论是：
```text
MemoryContext 管内存生命周期。
ResourceOwner / context callback 管非内存资源收尾。
fmgr 只定义调用 ABI，不替 callee 做异常安全。
```
### 6.8 invalidation
`FmgrInfo` 中的 `fn_addr` 指向某个函数实现。 对 builtin 函数，这通常稳定。 对 catalog、SQL 函数
cache、动态库、extension handler 或计划 cache，语义可能受 invalidation 影响。 `FmgrInfo` 本身不是 invalidation
机制。 它只是一个 cache 容器。 如果 `fn_extra` 里缓存了依赖 catalog
的语义，创建者必须安排失效、重建或保证生命周期 足够短。 这也是为什么 executor 表达式里的 `FmgrInfo`
通常挂在 query-lifespan context。 query 结束后整体释放，比长期维护精细 invalidation 更便宜。
## 7. 正确性机制层次
### 7.1 catalog truth 与 runtime cache
`pg_proc` 是函数属性的 catalog truth。 `FmgrInfo` 是 runtime cache。 `fn_strict`、`fn_retset`、`fn_nargs` 来自 catalog 或
builtin table。 执行期不应把 `FmgrInfo` 当成独立 truth。 如果 catalog 变化导致 cached plan
或表达式状态失效，上层 plan cache / relcache / syscache invalidation 应让旧执行状态不再被错误复用。
### 7.2 strict 正确性
strict 的正确性依赖 caller。 `FunctionCallInvoke()` 不检查 NULL。 表达式解释器、SRF caller、聚合 transition caller 或
`FunctionCallN` wrapper 必须在 需要时检查。 这让 hot path 可以为一参数、二参数 strict 函数做特化。
代价是新调用点必须显式遵守协议。 漏掉 strict 检查不会一定崩溃。 它可能只在 NULL 输入、pass-by-reference
参数或扩展函数假设非 NULL 时暴露。
### 7.3 NULL result 正确性
callee 通过 `fcinfo->isnull` 返回 NULL 状态。 caller 通过函数返回的 `Datum` 接收值。 两个通道必须一起看。
`Datum` 值本身不能说明结果是否 NULL。 如果 caller 忘记读取 `fcinfo->isnull`，NULL 结果会被当成非 NULL。 如果
caller 忘记在调用前设置 `fcinfo->isnull = false`，上一轮结果可能污染下一轮。
### 7.4 collation 正确性
`fncollation` 是 `FunctionCallInfo` 的字段。 同一个函数 OID 在不同 collation 下可能返回不同结果或比较顺序。
`FmgrInfo` 不把 collation 固化为函数身份的一部分。 这让同一个 `FmgrInfo` 可以被不同 collation 调用点复用，但
caller 必须正确设置 `fcinfo->fncollation`。 排序比较、文本函数、正则、大小写转换等路径尤其依赖这一点。
### 7.5 SRF protocol 正确性
SRF 的正确性不是 `fn_retset` 一个 bit 保证的。 它依赖：
```text
FmgrInfo.fn_retset
FunctionCallInfo.resultinfo
ReturnSetInfo.allowedModes
ReturnSetInfo.returnMode
ReturnSetInfo.isDone
caller 的消费循环
```
任何一个字段错位，都会出现协议错误。 例如 caller 不支持 materialize，而 callee 强行 materialize，应 ERROR。
callee 返回 `ExprMultipleResult` 后，caller 必须继续调用或消费结果。 callee 返回 `ExprEndResult` 后，caller
必须停止本 set。
### 7.6 MemoryContext 正确性
函数调用内存正确性靠 context 边界，而不是 fmgr 自动复制。 常见层次是：
| context | 适合放什么 |
| --- | --- |
| `ecxt_per_tuple_memory` | 本次 tuple 的临时计算、短命 detoast、临时返回值。 |
| `argContext` | SRF 一组 ValuePerCall 调用期间要保留的参数和 fcinfo。 |
| `flinfo->fn_mcxt` | `fn_extra` 和函数调用点级 cache。 |
| query / executor context | 表达式状态、`FmgrInfo`、slot、ProjectionInfo 等 query 生命周期对象。 |
| TopMemoryContext 或专用 cache context | 真正全局缓存，但必须有 invalidation 和大小控制。 |
把状态放错 context 不一定马上崩溃。 它更常表现为两类问题。 一类是 use-after-reset，下一行或下一次 SRF
调用读到已释放对象。 另一类是 query-lifespan bloat，高频调用不断把短命对象塞进长寿 context。
### 7.7 security boundary
`fmgr_info_cxt_security()` 带 `ignore_security` 参数。 当函数有 `prosecdef`、`proconfig`，或 fmgr hook 需要介入时，
`FmgrInfo.fn_addr` 会被设置成 `fmgr_security_definer`，真正的目标函数 `FmgrInfo` 缓存在外层 `flinfo->fn_extra` 里。
调用时 `fmgr_security_definer` 会切换 user id / GUC，触发 hook，把 `fcinfo->flinfo` 临时替换成私有 flinfo，并用
`PG_TRY` 保证 ERROR 时恢复 `fcinfo->flinfo` 和 hook 状态。 因此 fmgr 是权限和安全属性穿过 executor 的边界之一。
函数是否可调用、是否需要 security definer 上下文、是否由语言 handler 解释，不能只看 `fn_addr`。 `fn_addr`
是已经解析后的调用入口。 安全语义往往在设置它、进入 handler 或执行 SQL 函数时发生。 因此诊断权限相关问题时，
不要只在 `EEOP_FUNCEXPR` 断点看 `fn_addr`。 还要回到 function lookup、handler 和 SQL function setup。
### 7.8 并发边界
本节主要状态是 backend-local。 `FmgrInfo`、`FunctionCallInfo`、`ReturnSetInfo`、`FuncCallContext` 都不是 shared memory
状态。 函数内部当然可能访问 shared buffers、catalog、locks、WAL 或 SPI。 但 fmgr 调用协议本身不提供跨 backend
synchronization。 如果扩展函数把指针放进 shared memory，不能把 backend-local `Datum`、`FmgrInfo` 或 memory context
指针也放进去。 这些指针对其他进程没有意义。
### 7.9 统计正确性
`pg_stat_user_functions` 的调用次数和耗时依赖 function usage path。 它不是 executor node instrumentation。 一个 SQL
查询中的函数成本可能同时体现在：
```text
EXPLAIN ANALYZE 节点时间
pg_stat_user_functions 函数累计时间
pg_stat_statements 查询累计时间
perf 栈样本
```
这些口径不同。 不能把一个指标当成完整因果。
### 7.10 本节不涉及的机制
本节几乎不涉及 WAL。 函数内部可以产生 WAL，例如执行 DML、调用 SPI 或修改数据。 但 fmgr
调用边界本身不保证 WAL-before-data。 本节也不直接涉及 MVCC visibility。 函数内部读表时会进入 executor / SPI /
snapshot 机制。 fmgr 只负责把函数调用起来。
## 8. 错误路径 / 异常路径 / fallback
### 8.1 `pg_proc` cache lookup 失败
非 builtin 函数初始化时需要查 `PROCOID` syscache。 找不到函数会 ERROR。 这通常说明 plan cache、catalog OID
或扩展对象状态已经不一致。 正常 DDL invalidation 应避免已删除函数继续被新执行状态调用。 如果你在 gdb
中看到 `fmgr_info()` 因 cache lookup 失败报错，下一步不应只看 `fmgr.c`。 还要追 cached plan
是否失效、表达式状态何时创建、函数对象是否被并发 DDL 修改。
### 8.2 symbol 或动态库加载失败
对于 C 扩展函数，`FmgrInfo.fn_addr` 需要最终解析到动态库 symbol。 如果 `.so` 不存在、symbol 名不匹配、ABI
不兼容或加载被安全策略阻止，初始化阶段会报错。
这类错误发生在函数第一次需要被解析时，不一定发生在 `CREATE FUNCTION` 时。 诊断时要区分：
```text
catalog 中有函数定义
动态库可以被加载
symbol 可以被找到
函数可以按 fmgr ABI 正确执行
```
这四层不是一回事。
### 8.3 strict NULL 跳过 callee
strict NULL 不是错误，但它是重要 fallback path。 函数没有被调用。
所以函数内部日志、计数、断点、`PG_ARGISNULL()` 分支都不会发生。 如果你想观察 NULL 输入，需要临时去掉
SQL `STRICT` 属性，或在 caller 的 strict 分支打断点。
### 8.4 callee 忘记设置 `isnull`
C 函数如果要返回 NULL，必须设置 `fcinfo->isnull = true`。 通常通过 `PG_RETURN_NULL()` 完成。
如果函数直接返回一个任意 `Datum`，caller 会把它解释为非 NULL。 反过来，如果函数设置了 `isnull`
但返回值未定义，caller 仍应以 `isnull` 为准。 这解释了为什么调试函数返回值时不能只打印 `Datum`。
### 8.5 caller 忘记 reset `isnull`
复用 `FunctionCallInfo` 时，caller 必须每次调用前设置 `fcinfo->isnull = false`。 表达式解释器和 `execSRF.c`
都这么做。 新写调用点如果漏掉，会出现上一轮 NULL 污染当前调用的错误。 这类 bug 在非 NULL 和 NULL
交替输入时更容易复现。
### 8.6 SRF 没有 `ReturnSetInfo`
返回 set 的函数如果发现 `fcinfo->resultinfo` 不是 `ReturnSetInfo`，应报错。 `funcapi.c` 的 helper 会检查这一点。
这类错误通常说明函数被放在不支持 SRF 的调用位置，或 caller 使用了普通 `FunctionCallN` wrapper 调用了
set-returning 函数。
### 8.7 SRF mode 不被 caller 支持
callee 不能随意选择返回模式。 它必须检查 `allowedModes`。 caller 可能只支持 `ValuePerCall`。 也可能支持
`Materialize` 但不支持 random access。 如果 callee 忽略这个 bitmask，可能让 caller 面对不能消费的 tuplestore
或错误的顺序语义。
### 8.8 per-tuple context 被 reset
普通表达式每行都会 reset per-tuple context。 SRF loop 中甚至每次调用前都可能 reset。
如果函数把跨调用状态放在当前 context，却没有切换到 `fn_mcxt` 或
`multi_call_memory_ctx`，下一次调用就可能读到无效内存。 这类错误常常只在多行输入、SRF、多次 rescan 或
long-running query 中出现。 单次 `SELECT myfunc(1)` 可能完全看不出来。
### 8.9 SQL 函数 cache 失效
SQL-language function 的 `SQLFunctionCache` 挂在 `fn_extra`。 它可能包含查询树、计划、类型信息和执行状态。
当依赖对象变化时，需要能检测并重建。 如果错误地把依赖 catalog 的状态放进过长寿 context，又没有
invalidation，就会出现旧 schema、旧类型或旧函数语义被复用。
### 8.10 ERROR 中断 SRF
SRF 在多次调用之间可能持有 tuplestore、multi-call context 或外部资源。 如果中途 ERROR，正常的 `SRF_RETURN_DONE()`
不会执行。 内存会由 context reset 兜底。 非内存资源必须有更强 owner。 这就是为什么第 72
节还要单独讲函数 ERROR 和 SRF 中断 cleanup。
## 9. 成本、资源与跨模块传播
### 9.1 CPU hot path 成本
函数调用出现在表达式求值 hot path。 一行可能调用很多函数。 一个函数可能在 filter、projection、join
key、hash key、sort key、aggregate transition 中被反复调用。 每次调用至少包括：
```text
参数表达式求值
写 fcinfo->args[]
strict NULL 检查
设置 fcinfo->isnull
间接函数指针调用
读取结果 isnull
可能的统计计时
```
这些成本随输入行数线性放大。 如果函数在 nested loop 内侧被调用，成本还会随 outer rows * inner probes 放大。
如果函数用于 sort comparison，成本可能随 `N log N` 次比较放大。 如果函数用于 hash equality 或 hash
function，成本会随 build / probe 次数放大。
### 9.2 indirect call 与 branch 成本
`fn_addr(fcinfo)` 是间接调用。 CPU 不能像普通内联函数那样轻易优化。 strict 分支、NULL 分布、function usage path
也会影响 branch prediction。 PostgreSQL 用 opcode 选择和一、二参数 strict 特化减少部分成本。
但它没有把所有函数都 JIT 成直接调用。 JIT 表达式路径可能改变函数调用周边成本，但 fmgr ABI 仍然存在。
诊断 CPU hotspot 时，perf 栈里看到 `ExecInterpExpr`、`ExecEvalFuncExpr*`、 具体 C 函数名或 fmgr
wrapper，要结合调用频率解释。
### 9.3 catalog lookup 成本
`fmgr_info()` 应在初始化期调用。 如果在 per-tuple hot path 里反复用 function OID 调 `fmgr_info()`，会引入 syscache
lookup、 动态库解析检查和 memory context churn。 正确设计通常是：
```text
初始化期 lookup 一次
执行期复用 FmgrInfo
```
这也是为什么 `FmgrInfo` 不是临时参数包，而是独立结构。
### 9.4 memory context 压力
错误 context 会让成本变形。 短命对象放进 query context，会让单 query 内存不断增长。 跨调用对象放进
per-tuple context，会导致 use-after-reset。 SRF materialize mode 会把所有返回行放进 tuplestore。 tuplestore 受 work_mem
影响，可能 spill 到 temp file。 这时一个函数调用问题会传播成 IO 问题。 `EXPLAIN (ANALYZE, BUFFERS)` 可能看不到
temp file 细节，日志和 `log_temp_files` 更直接。
### 9.5 `track_functions` 观测成本
开启 `track_functions` 后，函数调用会进入 usage 统计 path。 成本包括计时、计数和统计状态维护。
对低频重函数，这通常值得。 对每行调用百万次的小函数，它可能明显改变 runtime。
因此实验时要分别测：
```sql
SET track_functions = none;
SET track_functions = pl;
SET track_functions = all;
```
并解释差异。
### 9.6 SRF `ValuePerCall` 成本
`ValuePerCall` 模式下，caller 和 callee 要多次往返。 每次往返都要处理 `ReturnSetInfo.isDone`、per-tuple
reset、可能的统计和 result 消费。
如果每次只返回一个很小值，而总行数很大，函数边界本身就可能成为热点。 `generate_series` 这类 builtin
会尽量走轻路径。 复杂扩展 SRF 如果每行还做 catalog lookup 或 palloc 长寿对象，成本会迅速放大。
### 9.7 SRF `Materialize` 成本
`Materialize` 模式减少多次函数调用往返，但一次性构造完整集合。 它可能增加峰值内存。 它可能使用
tuplestore。 它可能 spill。 它也可能让 caller 可以 random access 或重复扫描。 选择哪种模式不是绝对优劣。
要看 caller 的 `allowedModes`、结果大小、是否需要 random access、是否能流式消费。
### 9.8 SQL 函数成本传播
SQL-language function 通过 fmgr 入口进来，但内部可能执行一条或多条查询。 这会传播到
planner、executor、snapshot、SPI、portal、ResourceOwner 和 plan cache。 从外层看，它仍然是一次函数调用。
从内部看，它可能包含完整 SQL 执行。 因此 `pg_stat_user_functions` 的函数耗时可能包含内部 SQL。
`pg_stat_statements` 是否把内部 SQL 分开统计，取决于调用路径和配置。 诊断时要把 fmgr 边界和内部 executor
边界分开。
### 9.9 类型 I/O 与 detoast
类型输入输出函数也通过 fmgr。 例如 text、numeric、json、timestamp 等类型转换可能在
COPY、cast、表达式和输出中高频出现。 这些函数可能 detoast、分配新 varlena、解析字符串或格式化输出。
本节的结论迁移到类型 I/O：`Datum` 指针的生命周期和 memory context 决定是否能跨边界保存。 第 65
节会更集中讨论 varlena / TOAST。
### 9.10 跨模块边界
本节至少连接这些模块：
| 模块 | 与 fmgr 的边界 |
| --- | --- |
| parser / analyzer | 解析函数名、参数类型和多态类型，最终给 planner/executor 函数 OID。 |
| planner / expression init | 把函数表达式编译成 `ExprState` step，并决定 strict / stats opcode。 |
| executor | 每行填参数、调用函数、处理结果和 per-tuple memory reset。 |
| pg_proc / syscache | 提供 strict、retset、language、prosrc 等 catalog truth。 |
| dynamic loader / language handler | 把非 builtin 函数解析成可调用 C entry。 |
| pg_stat | 可选统计函数调用次数和耗时。 |
| MemoryContext / ResourceOwner | 管理函数内部状态、临时内存和异常 cleanup。 |
| tuplestore / temp file | SRF materialize mode 的结果容器和 spill 路径。 |
fmgr 位于这些模块之间。 它不是最高层语义，也不是最低层 C 调用。 它是把 catalog 驱动的 SQL
函数世界压缩成 executor 可调用 ABI 的边界。
## 10. 观测与诊断入口
### 10.1 可直接观测的 runtime truth
本节锚定的 runtime truth 是：
```text
同一个 SQL 函数调用点，在初始化期只解析成 FmgrInfo；
执行期每行复用 FunctionCallInfo storage，strict NULL 会跳过 callee；
开启 track_functions 会改变调用 path 并暴露函数累计统计。
```
你可以通过 SQL、`pg_stat_user_functions`、`EXPLAIN ANALYZE`、perf 和 gdb 交叉验证。
### 10.2 SQL 层观察 strict 跳过
创建一个非 strict 函数和一个 strict 函数。 让函数内部 `RAISE NOTICE` 或写入临时计数。 再传入 NULL。
预期现象：
```text
非 strict 函数被调用，可以在函数内部看到 NULL。
strict 函数不被调用，结果直接为 NULL。
```
对 C 函数，断点应打在 caller strict 分支和函数入口两处。 只打函数入口会误以为“没有执行到这里”是
lookup 失败。
### 10.3 `pg_stat_user_functions`
开启函数统计：
```sql
SET track_functions = all;
SELECT pg_stat_reset();
SELECT myfunc(i) FROM generate_series(1, 100000) AS g(i);
SELECT funcname, calls, total_time, self_time
FROM pg_stat_user_functions
ORDER BY calls DESC;
```
能看到：
```text
函数级 calls
函数累计 total_time
函数 self_time
```
看不到：
```text
每一次 fcinfo 参数
strict NULL 被跳过的未调用次数
fn_extra 内部 cache 大小
per-tuple context 中临时分配细节
```
如果 strict NULL 跳过函数，`calls` 不应增加。 这可以用来验证 strict 是 caller 端短路。
### 10.4 EXPLAIN 与函数成本
`EXPLAIN (ANALYZE)` 看到的是 plan node 时间。 例如 filter 中函数很慢，通常表现为 scan node 时间增加。 projection
中函数很慢，可能表现为上层 projection 或 targetlist 所在节点时间增加。 EXPLAIN
不直接告诉你哪个函数最慢。 要结合：
```text
pg_stat_user_functions
auto_explain nested statement
perf / flamegraph
gdb 条件断点
```
### 10.5 perf / flamegraph
CPU 热点诊断时，常见栈形态包括：
```text
ExecInterpExpr
  -> ExecEvalFuncExprFusage
     -> pgstat_init_function_usage
     -> my_c_function

ExecInterpExpr
  -> my_c_function

ExecMakeFunctionResultSet
  -> FunctionCallInvoke
     -> generate_series_step
```
如果看到 `ExecInterpExpr` 很热，但具体函数不明显，可能是表达式解释器、参数求值、 strict
检查或小函数调用成本分散。 如果具体 C 函数名很热，再进入函数内部看算法和内存分配。
### 10.6 gdb 断点
常用断点：
```gdb
break fmgr_info_cxt_security
break ExecEvalFuncExprFusage
break ExecEvalFuncExprStrictFusage
break ExecMakeFunctionResultSet
break ExecMakeTableFunctionResult
break init_MultiFuncCall
```
对普通表达式 fast path，`EEOP_FUNCEXPR` 是解释器 label，不是普通函数。 直接在 label 上断点不总是方便。
可以断在具体 C 函数，或者断在 `ExecInterpExpr` 后用条件观察 `op->opcode`。 也可以临时在源码里加 `elog(DEBUG1,
...)`，但只用于实验分支。
### 10.7 MemoryContext 诊断
如果怀疑函数泄漏短命内存，可以观察：
```sql
SELECT name, level, total_bytes, free_bytes, used_bytes
FROM pg_backend_memory_contexts
WHERE name ILIKE '%ExprContext%'
   OR name ILIKE '%Executor%'
ORDER BY total_bytes DESC;
```
但这个视图是 backend 当前 context 粒度。 它不能告诉你某一次 fmgr 调用分配了哪个对象。 更细粒度需要：
```text
MemoryContextStats()
临时加日志
gdb 观察 CurrentMemoryContext
malloc / palloc profiler
```
SRF 相关问题还要观察 `argContext`、`multi_call_memory_ctx` 和 tuplestore context。
### 10.8 日志入口
相关日志或配置包括：
```text
log_min_error_statement
log_error_verbosity
log_temp_files
track_functions
auto_explain.log_nested_statements
auto_explain.log_analyze
```
动态库加载错误、函数 ERROR、SRF protocol ERROR 通常会出现在 server log。 temp file 日志能帮助确认 materialize SRF
或 SQL 函数内部查询是否产生 spill。
### 10.9 能看到、只能推断、看不到
| 类型 | 例子 |
| --- | --- |
| 能直接看到 | `pg_stat_user_functions.calls`、函数 ERROR、temp file、perf 栈中具体函数。 |
| 只能推断 | strict NULL 被跳过次数、`fn_extra` cache 是否命中、per-tuple reset 是否足够频繁。 |
| 基本看不到 | 每次 `fcinfo->args[]` 的生命周期、callee 是否保存了短命指针、`fn_mcxt` 内部对象语义。 |
不要把 `pg_stat_user_functions` 当完整 tracing。 它是函数级累计统计，不是每次调用事件流。
## 11. 常见误区
### 11.1 把 `FunctionCallInvoke()` 当成完整调用框架
它只是调用函数指针。 strict、参数填充、统计、memory context、SRF resultinfo 都在外层。
### 11.2 以为 strict 检查在 callee 内部发生
strict NULL 时 callee 不会被调用。 函数内部的 NULL 分支不会执行。
### 11.3 只看 `Datum` 不看 `isnull`
`Datum` 不携带 NULL 语义。 结果是否 NULL 由 `fcinfo->isnull` 或表达式结果的 `resnull` 决定。
### 11.4 把 `fn_extra` 当全局缓存
`fn_extra` 属于一个 `FmgrInfo`。 它能活多久取决于 `fn_mcxt`。 相同函数 OID 可能有多个 `FmgrInfo` 和多个
`fn_extra`。
### 11.5 把 per-tuple context 当 SRF 多次调用状态
per-tuple context 会被 reset。 SRF 跨调用状态应该放在 `multi_call_memory_ctx` 或 `fn_mcxt` 关联的生命周期。
### 11.6 以为 `fn_retset` 足以驱动 SRF
`fn_retset` 只是函数声明属性。 真正协议还需要 `ReturnSetInfo` 和 caller consumption loop。
### 11.7 忘记 collation 是每次调用现场
`fncollation` 在 `FunctionCallInfo`，不是 `FmgrInfo` 的固定属性。 同一个函数不同 collation 下可能有不同行为。
### 11.8 在 hot path 反复 `fmgr_info()`
这会把初始化成本放进每行执行。 正确路径通常是初始化期一次 lookup，执行期复用 `FmgrInfo`。
### 11.9 把 function stats 当无扰动观测
开启 `track_functions` 会改变调用 path。 它对小而高频的函数可能不是免费观测。
### 11.10 忽略 ERROR cleanup
C 函数 ERROR 后不会走普通 return cleanup。 非内存资源必须有 ResourceOwner、callback 或上层 cleanup。
## 12. 课堂实验
### 实验 1：验证 strict 是 caller 短路
目标：看到 NULL 输入时 strict 函数没有进入 callee。 步骤：
1. 用 PL/pgSQL 创建两个函数，一个 `STRICT`，一个非 strict。

2. 函数内部 `RAISE NOTICE` 打印参数是否 NULL。

3. 分别执行 `SELECT f(NULL)`。

4. 开启 `track_functions = all` 后重复执行。

5. 查询 `pg_stat_user_functions` 的 calls。
示例：
```sql
CREATE OR REPLACE FUNCTION f_not_strict(i int)
RETURNS int
LANGUAGE plpgsql
AS $$
BEGIN
  RAISE NOTICE 'called f_not_strict, null=%', i IS NULL;
  RETURN i;
END;
$$;

CREATE OR REPLACE FUNCTION f_strict(i int)
RETURNS int
LANGUAGE plpgsql
STRICT
AS $$
BEGIN
  RAISE NOTICE 'called f_strict';
  RETURN i;
END;
$$;

SET track_functions = all;
SELECT pg_stat_reset();
SELECT f_not_strict(NULL);
SELECT f_strict(NULL);
SELECT funcname, calls
FROM pg_stat_user_functions
WHERE funcname LIKE 'f_%'
ORDER BY funcname;
```
预期解释：
```text
f_not_strict 被调用，NOTICE 出现，calls 增加。
f_strict 不被调用，NOTICE 不出现，calls 不应因 NULL 调用增加。
```
回到源码：
```text
execExprInterp.c
  EEOP_FUNCEXPR_STRICT_1
    if args[0].isnull:
      *resnull = true
      不调用 fn_addr
```
### 实验 2：观察 `track_functions` 改变调用路径
目标：区分函数级统计和 executor node 时间。 步骤：
1. 准备一个简单 SQL 或 PL/pgSQL 函数。

2. 对 `generate_series(1, N)` 每行调用。

3. 分别设置 `track_functions = none` 和 `all`。

4. 用 `EXPLAIN (ANALYZE)` 观察节点时间。

5. 用 `pg_stat_user_functions` 观察函数 calls。

6. 用 perf 或 gdb 观察是否进入 `ExecEvalFuncExprFusage()`。
示例：
```sql
CREATE OR REPLACE FUNCTION f_add1(i int)
RETURNS int
LANGUAGE sql
AS $$ SELECT i + 1 $$;

SET track_functions = none;
EXPLAIN (ANALYZE, TIMING OFF)
SELECT sum(f_add1(i)) FROM generate_series(1, 1000000) AS g(i);

SET track_functions = all;
SELECT pg_stat_reset();
EXPLAIN (ANALYZE, TIMING OFF)
SELECT sum(f_add1(i)) FROM generate_series(1, 1000000) AS g(i);
SELECT funcname, calls, total_time
FROM pg_stat_user_functions
WHERE funcname = 'f_add1';
```
观察重点：
```text
EXPLAIN 节点时间是 executor 口径。
pg_stat_user_functions 是函数累计口径。
track_functions 可能让表达式 step 走 FUSAGE path。
```
### 实验 3：跟读 `FmgrInfo` 生命周期
目标：确认 lookup 在初始化期，调用在执行期。 建议断点：
```gdb
break fmgr_info_cxt_security
break ExecEvalFuncExprFusage
break ExecMakeTableFunctionResult
```
步骤：
1. 启动单 backend，连接数据库。

2. 对一个简单函数执行 `SELECT f_add1(i) FROM generate_series(1, 3) g(i);`。

3. 在 `fmgr_info_cxt_security` 记录 `functionId`、`mcxt`、`fn_strict`、`fn_retset`。

4. 继续执行到函数调用处，观察 `fcinfo->flinfo` 是否复用同一个 `FmgrInfo`。

5. 打印 `fcinfo->nargs`、`fcinfo->args[0].isnull`、`fcinfo->fncollation`。
思考：
```text
为什么 fmgr_info 不应该在每一行执行？
为什么 FunctionCallInfo 可以复用，但每次要重置 isnull？
```
### 实验 4：SRF `ValuePerCall` 与 `ReturnSetInfo`
目标：看到 set-returning function 通过 `ReturnSetInfo.isDone` 推进。 步骤：
1. 使用 `generate_series(1, 3)` 作为 table function。

2. 在 `ExecMakeTableFunctionResult()` 断点。

3. 打印 `rsinfo.allowedModes`、`rsinfo.returnMode`、`rsinfo.isDone`。

4. 每次 `FunctionCallInvoke(fcinfo)` 后再次打印。

5. 观察 `ExprMultipleResult` 和 `ExprEndResult`。
示例 SQL：
```sql
SELECT * FROM generate_series(1, 3);
```
回到源码：
```text
execSRF.c
  构造 ReturnSetInfo
  循环 FunctionCallInvoke
  如果 returnMode == SFRM_ValuePerCall:
      isDone == ExprMultipleResult -> 消费一行
      isDone == ExprEndResult -> 结束
```
### 实验 5：内存上下文放错位置的最小复现思路
目标：理解为什么 SRF 不能把跨调用状态放在 per-tuple context。 不要求在产品分支提交代码。 可以在临时 C
扩展中做两个版本：
```text
版本 A：第一次调用时在 CurrentMemoryContext 分配 user_fctx。
版本 B：第一次调用时切换到 multi_call_memory_ctx 分配 user_fctx。
```
然后在 SRF 中多次返回。 观察：
```text
per-tuple reset 后版本 A 是否出现异常、随机值或崩溃。
版本 B 是否稳定。
```
回到源码：
```text
funcapi.c
  SRF_FIRSTCALL_INIT()
  multi_call_memory_ctx
execSRF.c
  ResetExprContext(econtext)
```
实验结论应写成：
```text
跨调用状态的正确 owner 是 multi-call context 或 fn_mcxt 相关生命周期；
不是当前看起来可用的 CurrentMemoryContext。
```
## 13. 讨论题
1. 为什么 `FunctionCallInvoke()` 不内置 strict NULL 检查？
2. `FmgrInfo.fn_oid` 和 `FmgrInfo.fn_addr` 分别代表什么？为什么 handler 场景下不能把

   二者混为一谈？
3. `fn_extra` 适合保存什么？哪些状态不应该只放在 `fn_extra` 里？
4. 为什么 `FunctionCallInfoBaseData` 需要 flexible array，而不能固定放

   `FUNC_MAX_ARGS` 个参数？
5. 一个 strict C 函数内部写了 `if (PG_ARGISNULL(0))`，为什么这段代码可能永远不会执行？
6. SRF 的 `fn_retset`、`ReturnSetInfo.returnMode` 和 `ReturnSetInfo.isDone`

   分别回答什么问题？
7. 如果 `pg_stat_user_functions.calls` 没有增加，能否断言函数 lookup 没发生？
8. 为什么同一个函数 OID 在同一个 query 中可能对应多个 `FmgrInfo`？
9. `track_functions = all` 为什么可能改变小函数 benchmark 的结果？
10. C 函数把 pass-by-reference 参数指针保存到全局变量里，会同时违反哪些边界？
## 14. 本节小结
本节主链路是：
```text
pg_proc / builtin metadata
  -> fmgr_info_cxt()
  -> FmgrInfo
  -> InitFunctionCallInfoData()
  -> fcinfo->args[]
  -> strict / stats / SRF protocol
  -> fn_addr(fcinfo)
  -> Datum + isnull / ReturnSetInfo
```
`FmgrInfo` 保存稳定函数属性和可调用入口。 `FunctionCallInfo` 保存一次调用现场。 `fn_extra` 是函数调用点级
cache。 `fn_mcxt` 决定这个 cache 能活多久。 strict NULL 短路由 caller 执行。 `FunctionCallInvoke()` 只调用函数指针。
SRF 通过 `ReturnSetInfo` 把普通 `Datum` 返回扩展成多行协议。 `ValuePerCall` 依赖多次调用和 `isDone`。 `Materialize`
依赖 tuplestore 和 caller 支持的 mode。 内存正确性来自 context 层次。 per-tuple context 适合短命对象。 `argContext`
适合 SRF 一组调用期间的参数和 fcinfo。 `flinfo->fn_mcxt` 适合 `fn_extra`。 ERROR 时，memory context
能释放内存，但非内存资源必须有 ResourceOwner、callback 或上层 cleanup。 能直接观测的是函数级
calls、函数耗时、错误日志、temp file 和 profiler 栈。 只能推断的是 strict 跳过次数、`fn_extra` cache
命中和具体参数指针生命周期。 本节可迁移规律是：
```text
一个低成本 runtime ABI 往往把稳定元数据和单次调用现场拆开；
性能来自初始化期缓存和执行期复用；
正确性来自 caller/callee 共同遵守 NULL、resultinfo 和 memory context 协议。
```
判断函数调用问题时，不要先问“这个函数做了什么”。 先问四个边界：
```text
FmgrInfo 是谁创建和持有的？
FunctionCallInfo 的参数从哪里来、能活多久？
strict / SRF / stats path 由谁选择？
callee 返回的 Datum 指向的对象由哪个 context 拥有？
```
这些边界清楚后，才能把慢 SQL、函数统计、SRF 行数、内存膨胀或 NULL 行为回到源码解释。
