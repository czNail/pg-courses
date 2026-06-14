# PostgreSQL operator/function lookup execution
## 课程定位
前置知识：已经理解 parse/analyze/rewrite/plan/execute 的大致边界。
前置知识：已经读过 `ExprContext`、`ExprState`、`ExecProcNode` 和 fmgr 调用边界。
本节唯一主问题：
```text
operator、support function 和 function cache 如何把 catalog metadata 转成可调用执行状态？
```
核心矛盾：
```text
SQL 层必须允许重载、隐式类型转换、search_path、扩展函数和 catalog 变更；
执行期 hot path 又必须尽量只看到 OID、函数指针、strict/null 规则和已排好的 ExprState step。
```
学完后应能判断：一个表达式慢、函数调用报错或 catalog 变更后的行为，到底发生在 lookup、coercion、plan invalidation、fmgr 初始化，还是执行期 `FunctionCallInfo` 调用。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e`。
本节只讲 operator/function lookup 到 executor callable state 的链路。
本节不展开函数体语言实现、JIT codegen、完整 selectivity 估算和 index opclass 选择。
这些主题会在表达式解释器、collation/sort support、JIT 和 access method 课程中继续展开。
## 1. 本节在总主线中的位置
04 目录前面已经建立了三条基础线索。
第一条线索是 executor 如何以 `PlanState` tree 和 `ExecProcNode()` 做 tuple-by-tuple 调度。
第二条线索是 `ExprContext` 如何给表达式求值提供当前 tuple、param 和 per-tuple memory。
第三条线索是 fmgr 如何用 `FmgrInfo` 和 `FunctionCallInfo` 调用 PostgreSQL C 函数。
本节插在这些线索之间。
SQL 里的 `a + b`、`lower(x)`、`x = ANY (...)` 或 `ORDER BY x` 看上去是语法。
执行器真正看到的不是名字。
执行器看到的是 `OpExpr`、`FuncExpr`、`ScalarArrayOpExpr`、`SortGroupClause`、`EquivalenceClass` 和若干 support function OID。
这些 OID 再被 `ExecInitExprRec()` 初始化成可执行的 `ExprEvalStep`。
最后 `ExecInterpExpr()` 只在 step 中填参数、检查 strict/null、调用 `fn_addr`。
所以本节要追的不是“函数怎么运行”。
本节要追的是：
```text
名字和 catalog 元数据，什么时候停止作为名字存在，
并变成执行器可以直接调用的状态？
```
这条链路跨越 parser、namespace/syscache、typcache、planner support、plancache invalidation、executor expression init 和 fmgr。
如果只读其中一个文件，很容易得出错误结论。
例如，只读 `fmgr.c` 会以为函数 lookup 发生在执行期。
只读 `parse_oper.c` 会以为 operator cache 就是执行期 cache。
只读 `execExprInterp.c` 会看不到 search_path、ambiguous function 和 coercion 早就被处理掉了。
本节把这些边界串起来。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
parse analysis 用名字、输入类型和 search_path 查 pg_operator/pg_proc，
把重载解析结果保存为 operator OID 和 function OID；
planner 在需要时用 support function 读取额外优化语义；
executor 初始化表达式时把 function OID 转成 FmgrInfo、FunctionCallInfo 和 ExprEvalStep；
执行期反复调用 step 中缓存的 fn_addr，而不是重新做名字解析。
```
这条模型有四个阶段。
第一阶段是 name lookup。
`FuncnameGetCandidates()` 和 `OpernameGetCandidates()` 以 search path、schema qualification、参数个数、named/default/variadic 等规则生成候选。
第二阶段是 overload resolution。
`func_get_detail()` 和 `oper()` 把候选缩成一个 `pg_proc` 或 `pg_operator` OID。
这个阶段会处理 `unknown` literal、domain base type、隐式 coercion、named notation、default arguments 和 ambiguous error。
第三阶段是 executable metadata construction。
`make_op()`、函数调用 transform 和 planner 支撑代码把 OID、输入类型、返回类型、collation、strict/set-returning 等信息塞进表达式树或计划状态。
第四阶段是 call-state initialization。
`ExecInitExprRec()` 调 `ExecInitFunc()`，再调 `fmgr_info()`；其他长期 cache owner 也会按需用 `fmgr_info_cxt()`，生成 `FmgrInfo`、`FunctionCallInfo` 和 `EEOP_FUNCEXPR*` step。
核心 tension 在这里：
```text
越晚解析名字，越能反映最新 catalog；
越早解析并缓存 callable state，执行期每个 tuple 的成本越低。
```
PostgreSQL 的选择不是“永远最晚解析”。
PostgreSQL 也不是“把 C 函数地址永久塞进 plan”。
它把状态分层：
| 层次 | 保存什么 | 失效边界 |
| --- | --- | --- |
| parse tree / analyzed query | `OpExpr.opno`、`opfuncid`、`FuncExpr.funcid`、参数 coercion | catalog/search_path/role/RLS/plan cache invalidation |
| planner metadata | support function 输出、selectivity、cost、index condition、sort support 选择 | 重新规划或相关 cache invalidation |
| executor expression state | `FmgrInfo`、`FunctionCallInfo`、`ExprEvalStep` | executor context 生命周期 |
| fmgr C function cache | 外部 C 函数地址、info record、xmin/tid freshness | backend-local hash 与 syscache 元组身份检查 |
这四层的失效语义不同。
`OpExpr.opfuncid` 是语义结果，不是可调用函数指针。
`FmgrInfo.fn_addr` 是当前 backend 当前 executor 生命周期内的调用入口，不是跨 backend 共享状态。
support function 的返回值服务 planner 或执行前准备，不等价于 SQL 函数执行结果。
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/parser/parse_oper.c` | operator 名字解析、lookaside cache、`OpExpr` 构造、operator OID 到 underlying function OID。 |
| 2 | `src/backend/parser/parse_func.c` | function 候选查找、重载解析、named/default/variadic、coercion 和 `FuncExpr` 构造前的语义判断。 |
| 3 | `src/backend/catalog/namespace.c` | `FuncnameGetCandidates()`、`OpernameGetOprid()`、search_path 和 namespace visibility。 |
| 4 | `src/backend/utils/cache/lsyscache.c` | 从 OID 读取轻量 catalog 属性，例如 `get_opcode()`、`get_func_support()`、`get_func_retset()`。 |
| 5 | `src/backend/utils/cache/typcache.c` | 类型级 operator/procedure cache，服务排序、分组、hash、比较函数和 opclass 相关 lookup。 |
| 6 | `src/backend/optimizer/util/clauses.c` | `SupportRequestSimplify`、inline 相关 support function 调用。 |
| 7 | `src/backend/optimizer/util/plancat.c` | `SupportRequestSelectivity`、`SupportRequestCost`、`SupportRequestRows` 调用。 |
| 8 | `src/backend/optimizer/path/indxpath.c` | `SupportRequestIndexCondition` 如何把函数语义转成 index qual 候选。 |
| 9 | `src/backend/utils/fmgr/fmgr.c` | `fmgr_info()`、builtin fast path、C function cache、security definer wrapper、function stats。 |
| 10 | `src/include/fmgr.h` | `FmgrInfo`、`FunctionCallInfo` 和 `FunctionCallInvoke()` 的 ABI 边界。 |
| 11 | `src/backend/executor/execExpr.c` | `ExecInitExprRec()` 和 `ExecInitFunc()` 如何把 `FuncExpr` / `OpExpr` 编译成 eval step。 |
| 12 | `src/backend/executor/execExprInterp.c` | `EEOP_FUNCEXPR*` case 如何在执行期调用 `fn_addr`。 |
| 13 | `src/include/catalog/pg_operator.h` | operator catalog 字段：`oprleft`、`oprright`、`oprcode`、selectivity hooks。 |
| 14 | `src/include/catalog/pg_proc.h` | function catalog 字段：`prolang`、`proisstrict`、`proretset`、`prosupport`、`prosecdef`、`proconfig`。 |
| 15 | `src/include/nodes/supportnodes.h` | planner support request 节点种类和返回协议。 |
推荐阅读顺序不是按模块层次，而是按状态转化。
先读 parser，因为名字和重载只有在 parse analysis 里还存在。
再读 `lsyscache.c` 和 catalog header，因为后续模块大多只拿 OID 问一个 catalog 属性。
再读 support function，因为它们说明 planner 为什么需要函数作者提供额外语义。
最后读 executor 和 fmgr，因为这里已经没有 SQL 名字解析，只有 callable state。
## 4. 关键数据结构与状态
### 4.1. `pg_operator`
`pg_operator` 把 SQL operator 名字绑定到输入类型、输出类型和 underlying function。
本节最关键的字段是：
| 字段 | 语义 |
| --- | --- |
| `oprname` | operator 名称，例如 `+`、`=`、`<@`。 |
| `oprnamespace` | namespace，参与 search_path 和可见性判断。 |
| `oprleft` / `oprright` | 左右输入类型；prefix operator 的左类型为 invalid。 |
| `oprresult` | operator 返回类型。 |
| `oprcode` | 真正执行 operator 的 `pg_proc` OID。 |
| `oprrest` / `oprjoin` | restriction / join selectivity estimator function。 |
| `oprcanmerge` / `oprcanhash` | operator 是否可用于 merge/hash 相关路径的 catalog hint。 |
`pg_operator` 不是执行函数表。
它是 operator 语义的 catalog 元数据。
真正执行时，`oprcode` 会进入 `OpExpr.opfuncid`，然后被 fmgr 初始化。
所以 operator 的执行 hot path 不是按 operator 名字查 `pg_operator`。
它是按已经保存的 `opfuncid` 调函数。
### 4.2. `pg_proc`
`pg_proc` 描述函数、过程、聚合、窗口函数和 support function。
本节关注这些字段：
| 字段 | 语义 |
| --- | --- |
| `oid` | 函数身份，表达式树和 plan 中保存的主 key。 |
| `proname` / `pronamespace` | SQL 名字和 namespace，只在 lookup 阶段重要。 |
| `proargtypes` | 输入参数类型向量，参与 overload resolution。 |
| `prorettype` | 返回类型。 |
| `proretset` | 是否返回 set，影响 SRF 初始化和执行协议。 |
| `proisstrict` | strict/null 输入规则，影响 executor 是否跳过调用。 |
| `provolatile` | volatility，影响 planner 常量折叠、index qual、并行等判断。 |
| `proparallel` | parallel safety。 |
| `prosupport` | planner support function OID。 |
| `prolang` / `prosrc` / `probin` | fmgr 如何找到实际实现。 |
| `prosecdef` / `proconfig` | 是否需要 security definer / SET wrapper。 |
`prosupport` 的语义容易误读。
它不是“这个函数执行时额外调用的 helper”。
它是 planner 或相关准备阶段主动调用的元函数，用来回答 simplify、cost、rows、selectivity、index condition 等问题。
### 4.3. `OpExpr`
`OpExpr` 是 parse analysis 后的 executable expression node 之一。
关键字段可以压缩成：
| 字段 | 语义 |
| --- | --- |
| `opno` | 选中的 `pg_operator` OID。 |
| `opfuncid` | `pg_operator.oprcode`，真正执行时调用的函数 OID。 |
| `opresulttype` | operator 返回类型。 |
| `opretset` | underlying function 是否 set-returning。 |
| `opcollid` | 表达式结果 collation。 |
| `inputcollid` | 传给 underlying function 的 collation。 |
| `args` | 已插入必要 coercion 后的参数表达式。 |
| `location` | 错误定位。 |
`opno` 和 `opfuncid` 不能混为一谈。
`opno` 用于保持 operator 语义、反查 selectivity、opclass 关系和 deparse。
`opfuncid` 用于执行。
同一个 function 可以被多个 operator 包装。
同一个 operator 的 planner 语义也不只来自 `opfuncid`。
### 4.4. `FuncExpr`
`FuncExpr` 是普通函数调用的表达式节点。
关键字段：
| 字段 | 语义 |
| --- | --- |
| `funcid` | 选中的 `pg_proc` OID。 |
| `funcresulttype` | 返回类型。 |
| `funcretset` | 是否 SRF。 |
| `funcvariadic` | 是否按 variadic 形式调用。 |
| `funcformat` | 调用语法形式，影响 deparse。 |
| `funccollid` | 返回 collation。 |
| `inputcollid` | 函数接收的 collation。 |
| `args` | 已按 declared arg type coerced 的表达式列表。 |
| `location` | 错误定位。 |
`FuncExpr` 已经不保存原始候选集合。
ambiguous function、default argument、named notation 和 coercion 决策都已经折叠进 `funcid` 和 `args`。
因此执行期不能“重新挑一个更合适的 overload”。
### 4.5. `FmgrInfo`
`FmgrInfo` 是 fmgr 对“如何调用某个函数”的 backend-local 描述。
本节关注这些字段组合：
| 字段 | 语义 |
| --- | --- |
| `fn_addr` | 真实调用入口，可能是 builtin C function、external C function、language handler wrapper 或 `fmgr_security_definer`。 |
| `fn_oid` | 对应 `pg_proc` OID。 |
| `fn_nargs` | 参数个数。 |
| `fn_strict` | null 输入时是否跳过调用。 |
| `fn_retset` | 是否返回 set。 |
| `fn_stats` | 是否需要 function usage 统计。 |
| `fn_extra` | 函数或 wrapper 挂载的私有缓存。 |
| `fn_mcxt` | `fn_extra` 等附属数据的内存上下文。 |
| `fn_expr` | 指向表达式节点，供函数体读取调用上下文。 |
`fn_addr` 是 hot path 入口。
但 `fn_oid` 必须最后设置。
源码注释说明，有些代码把 `fn_oid` 有效视为整个 `FmgrInfo` 已经有效。
这就是典型的“field + 初始化顺序 = 语义”。
### 4.6. `FunctionCallInfo`
`FunctionCallInfo` 是一次函数调用的参数帧。
它不是 catalog cache。
它保存：
| 字段 | 语义 |
| --- | --- |
| `flinfo` | 指向 `FmgrInfo`。 |
| `context` | 聚合、触发器、SRF 等特殊调用上下文。 |
| `resultinfo` | SRF、tuplestore、record 返回等扩展结果协议。 |
| `fncollation` | 当前调用使用的 collation。 |
| `isnull` | 返回值是否 NULL。 |
| `nargs` | 参数个数。 |
| `args[i].value` / `args[i].isnull` | 第 i 个参数 Datum 和 null 标记。 |
执行器会在每次 tuple 求值时改写 `args[i]` 和 `isnull`。
`FunctionCallInfo` 的结构本身通常在 executor init 阶段分配好。
这减少了每 tuple malloc。
### 4.7. `ExprEvalStep`
`ExprEvalStep` 是表达式解释器的指令。
function/operator 调用共用 `func` union 分支。
关键字段：
| 字段 | 语义 |
| --- | --- |
| `opcode` | `EEOP_FUNCEXPR`、`EEOP_FUNCEXPR_STRICT`、`EEOP_FUNCEXPR_FUSAGE` 等。 |
| `d.func.finfo` | 指向 `FmgrInfo`。 |
| `d.func.fcinfo_data` | 指向 `FunctionCallInfo` 参数帧。 |
| `d.func.fn_addr` | 复制出来的调用入口，减少一层 indirection。 |
| `d.func.nargs` | 参数个数。 |
这里可以看到 PostgreSQL 的 hot path 取舍。
它保留了通用 fmgr ABI。
但执行器会根据 strict、参数个数和 stats 预先选择不同 opcode。
这样每次调用不用重新判断完整 catalog 元数据。
### 4.8. planner support request
support function 使用 `Node *` request 协议。
`src/include/nodes/supportnodes.h` 中定义了多类 request。
本节关注这些：
| request | 常见调用者 | 作用 |
| --- | --- | --- |
| `SupportRequestSimplify` | `eval_const_expressions()` | 让函数自己提供简化表达式。 |
| `SupportRequestSimplifyAggref` | aggregate simplification | 简化聚合表达式。 |
| `SupportRequestInlineInFrom` | function inlining | 尝试把 FROM 中函数调用 inline。 |
| `SupportRequestSelectivity` | planner selectivity | 估算 restriction / join selectivity。 |
| `SupportRequestCost` | planner cost | 估算函数执行成本。 |
| `SupportRequestRows` | set-returning function rows | 估算 SRF 行数。 |
| `SupportRequestIndexCondition` | index path generation | 把函数条件转成 indexable qual。 |
| `SupportRequestWFuncMonotonic` | window path | 判断 window function 单调性。 |
| `SupportRequestOptimizeWindowClause` | window optimization | 优化 window frame。 |
support function 的调用也是 fmgr 调用。
但它发生在 planner 或初始化相关路径。
它的返回值是 planner metadata，不是 SQL row 的 Datum。
## 5. 主流程源码 walkthrough
### 5.1. 主链路总览
用一条普通查询做主线：
```sql
SELECT *
FROM t
WHERE a + 1 = lower(b)::int;
```
这条 SQL 至少涉及三类 lookup。
`+` 和 `=` 是 operator lookup。
`lower()` 是 function lookup。
`::int` 可能走 cast lookup 或 coercion path。
主链路可以压缩为：
```text
transform expression
  -> operator/function candidate lookup
  -> overload resolution and coercion
  -> analyzed expression tree stores OIDs
  -> planner may call support functions by OID
  -> plan cache records dependencies and invalidation boundary
  -> ExecInitExprRec() builds ExprEvalStep
  -> fmgr_info() / fmgr_info_cxt() fills FmgrInfo
  -> ExecInterpExpr() invokes fn_addr per tuple
```
这一节的关键是“状态在每一步变窄”。
SQL 名字是宽状态。
候选集合是宽状态。
OID 是窄状态。
`FmgrInfo` 是更窄的 backend-local callable state。
`fn_addr` 是执行 hot path 最窄的入口。
### 5.2. operator lookup：从名字到 operator OID
operator lookup 的入口之一是 `make_op()`。
它接收 operator 名字、左右表达式树和上下文。
它先取左右输入类型。
对 prefix operator，只查右输入类型。
对 binary operator，调用 `oper()`。
`oper()` 的主流程是：
```text
make_oper_cache_key()
  -> find_oper_cache_entry()
  -> binary_oper_exact()
  -> OpernameGetCandidates()
  -> oper_select_candidate()
  -> SearchSysCache1(OPEROID)
```
第一步是 parser-local lookaside cache。
cache key 包括 operator name、左右类型和 search_path 相关状态。
这不是 executor cache。
它只避免 parse analysis 中重复解析同一个 operator。
第二步是 exact match。
`binary_oper_exact()` 会特别处理一个常见 SQL 现象：`unknown` literal。
如果一边是 unknown，另一边有明确类型，源码会尝试把 unknown 当成另一边的类型。
如果涉及 domain，还会尝试 base type。
第三步是候选查找。
`OpernameGetCandidates()` 从 namespace 可见的 operator 中找同名、同 operator kind 的候选。
第四步是候选选择。
`oper_select_candidate()` 会先用 `func_match_argtypes()` 删除不能接受输入类型的候选。
如果还剩多个，复用 `func_select_candidate()` 的启发式规则。
第五步是拿到 syscache tuple。
`oper()` 返回的是 syscache entry。
调用者必须 `ReleaseSysCache()`。
这条 ownership 边界很重要：parser 可以短暂读 catalog tuple，但不能把 tuple 指针塞进表达式树。
表达式树只能保存 OID 和必要属性。
### 5.3. operator expression：从 operator OID 到 `OpExpr`
`make_op()` 拿到 operator tuple 后，会读取 `pg_operator` 字段。
关键转化是：
```text
pg_operator.oid
  -> OpExpr.opno
pg_operator.oprcode
  -> OpExpr.opfuncid
pg_operator.oprresult
  -> OpExpr.opresulttype
```
随后它会调用 `make_fn_arguments()`。
这个函数会把实际参数表达式 coercion 到 operator declared arg type。
因此执行期不需要再做 overload 选择。
执行期只按表达式树中的子表达式求值。
`make_op()` 还会处理 collation。
`opcollid` 描述结果 collation。
`inputcollid` 描述传给 underlying function 的 collation。
这解释了一个常见现象：同一个函数 OID 在不同 collation 下可能表现不同。
collation 不在 `FmgrInfo` 里固定。
它在 `FunctionCallInfo.fncollation` 中随调用设置。
### 5.4. function lookup：从名字到 function OID
普通函数调用的核心解析函数是 `func_get_detail()`。
它的主流程是：
```text
FuncnameGetCandidates()
  -> exact argument type match
  -> special case: function-like type coercion
  -> func_match_argtypes()
  -> func_select_candidate()
  -> SearchSysCache1(PROCOID)
  -> read prorettype/proretset/provariadic/prokind/defaults
```
`FuncnameGetCandidates()` 负责 namespace search、schema qualification、variadic、default argument、named notation 等候选生成。
`func_get_detail()` 负责把候选缩成一个语义结果。
它可能返回的不是普通函数。
返回码可能是 `FUNCDETAIL_NORMAL`、`FUNCDETAIL_AGGREGATE`、`FUNCDETAIL_WINDOWFUNC`、`FUNCDETAIL_PROCEDURE`、`FUNCDETAIL_COERCION`、`FUNCDETAIL_MULTIPLE` 或 `FUNCDETAIL_NOTFOUND`。
这就是为什么 parser 不能把 `foo(...)` 直接理解成 `FuncExpr`。
`foo(...)` 可能是聚合。
可能是 window function。
可能是类型 coercion。
可能是 ambiguous。
可能因为 named/default/variadic 展开而改变参数列表。
### 5.5. function expression：从 function OID 到 `FuncExpr`
当 parser 确认这是普通函数调用后，会构造 `FuncExpr`。
`funcid` 是 `pg_proc` OID。
`funcresulttype` 来自 `pg_proc.prorettype` 或多态推导后的结果。
`funcretset` 来自 `pg_proc.proretset`。
`args` 已经按 declared type 处理过 coercion。
`inputcollid` 在 parse collation 阶段确定。
`FuncExpr` 的一个诊断价值是：它把 SQL 重载规则的结果冻结下来。
后续 executor 不会因为参数 Datum 的运行时值改变而重新选择 overload。
例如 `$1` 的类型在 prepared statement 中一旦确定，`funcid` 就跟着确定。
运行时只是给这个 `funcid` 传不同 Datum。
### 5.6. `LookupFuncName()`：DDL 和对象引用路径
并不是所有 function lookup 都来自表达式。
DDL、extension、operator creation、trigger creation、language handler lookup 也会调用 `LookupFuncName()`。
`LookupFuncName()` 的语义更像“按名字和精确参数类型找对象 OID”。
它调用 `LookupFuncNameInternal()`。
这个路径会扫描 `FuncnameGetCandidates()` 的结果，并检查参数类型和 object type。
如果有多个匹配，会报 ambiguous。
如果找不到，在 `missing_ok` 为 false 时会报 undefined function。
这条路径通常不会构造 `FuncExpr`。
例如 `CREATE OPERATOR` 需要找 underlying procedure。
它只需要一个 `pg_proc` OID 写入 `pg_operator.oprcode`。
### 5.7. lsyscache：OID 到轻量属性
parse 和 planner 之后，很多模块不想持有 syscache tuple。
它们只需要一个 catalog 属性。
`lsyscache.c` 提供了这些 helper。
本节相关的典型函数包括：
```text
get_opcode(operator_oid)
get_op_rettype(operator_oid)
get_op_opfamily_properties(...)
get_func_rettype(function_oid)
get_func_retset(function_oid)
get_func_nargs(function_oid)
get_func_signature(function_oid, ...)
get_func_support(function_oid)
```
这些 helper 的价值不是“更短的 API”。
它们把 syscache lookup、missing catalog error、Datum 解码和 tuple release 封装成一次小查询。
调用者拿到的是 OID、bool、类型 OID 或少量属性。
调用者不持有 catalog tuple。
因此这类属性需要在合适的上层生命周期中缓存。
例如 expression init 会把 strict、retset、fn_addr 缓存在 `FmgrInfo`。
typcache 会把类型比较、hash、排序相关函数缓存到 `TypeCacheEntry`。
### 5.8. typcache：类型级 operator/function cache
operator/function lookup 不只发生在 SQL 表达式里。
排序、分组、hash、join、partition pruning、array comparison 等路径都需要类型的 equality、less-than、hash、compare function。
`lookup_type_cache()` 用 flags 表示调用者需要什么。
常见 flags 包括：
```text
TYPECACHE_EQ_OPR
TYPECACHE_LT_OPR
TYPECACHE_GT_OPR
TYPECACHE_CMP_PROC
TYPECACHE_HASH_PROC
TYPECACHE_BTREE_OPFAMILY
TYPECACHE_HASH_OPFAMILY
```
`parse_oper.c` 里的 `get_sort_group_operators()` 就会调用 typcache。
它一次性拿 `<`、`=`、`>` 和 hash proc。
源码注释说明这样做是为了减少 lookup overhead，并保证结果来自匹配的 opclass。
这是一条重要边界：
```text
表达式 operator lookup 解决“这条 SQL 写的 operator 是哪一个”；
typcache 解决“这个类型在某类执行/规划协议下应该用哪组 operator/procedure”。
```
它们都依赖 catalog。
但调用者的语义不同。
### 5.9. support function：从函数 OID 到 planner metadata
`pg_proc.prosupport` 指向 planner support function。
planner 通过 `get_func_support()` 取出 support function OID。
如果存在，就构造对应 `SupportRequest*` node。
然后用 fmgr 调用 support function。
典型链路：
```text
planner sees FuncExpr
  -> get_func_support(funcid)
  -> build SupportRequestSimplify / Cost / Rows / Selectivity / IndexCondition
  -> OidFunctionCall1(support_oid, PointerGetDatum(&request))
  -> support function returns request pointer, result node, list, or NULL
```
support function 的接口故意很通用。
它接收一个 request node 指针。
它可以识别自己支持的 request type。
它不支持时返回 NULL。
这使一个 support function 可以服务多个 planner 问题。
但这也意味着 planner 不能假设 support function 一定可靠或一定返回。
support function 是优化信息，不是 correctness 的唯一来源。
正确性仍由表达式语义、operator/function OID、coercion 和 executor 调用保证。
### 5.10. executor init：从 `FuncExpr` / `OpExpr` 到 `ExprEvalStep`
执行器初始化表达式时进入 `ExecInitExprRec()`。
遇到 `FuncExpr`、`OpExpr`、`DistinctExpr`、`NullIfExpr` 等节点时，会走函数调用初始化逻辑。
operator 的执行最终和函数调用合流。
核心动作是：
```text
choose function oid
  -> object_aclcheck(ProcedureRelationId, funcid, ACL_EXECUTE)
  -> InvokeFunctionExecuteHook(funcid)
  -> allocate FmgrInfo
  -> fmgr_info(funcid, flinfo)
  -> fmgr_info_set_expr(original expression node, finfo)
  -> allocate FunctionCallInfo with nargs
  -> initialize argument result slots
  -> choose EEOP_FUNCEXPR* opcode
  -> push ExprEvalStep
```
`ExecInitFunc()` 使用当前表达式初始化上下文；索引、typcache 或 access method state 这类长期 cache owner 会在自己的上下文中使用 `fmgr_info_cxt()`。
稳定语义是：`FmgrInfo` 和 `FunctionCallInfo` 的生命周期至少覆盖对应 call site。
它们不应该在每 tuple 求值时重新分配。
`ExecInitFunc()` 会根据 `FmgrInfo` 的 `fn_strict`、`fn_stats` 和参数个数选择 opcode。
strict 函数可以在任一参数 NULL 时直接返回 NULL。
参数为 1 或 2 的 strict 函数有专门 opcode。
需要 function usage stats 的函数有 `_FUSAGE` opcode。
这说明 catalog metadata 已经被编译成执行器分支选择。
### 5.11. execution：从 step 到 `fn_addr`
表达式解释器执行到 function step 时，不再查名字。
主动作是：
```text
evaluate argument steps
  -> fill fcinfo->args[i]
  -> reset fcinfo->isnull
  -> if strict and any arg null: result = null
  -> call step->d.func.fn_addr(fcinfo)
  -> copy fcinfo->isnull and Datum result to target slot
```
`FunctionCallInvoke(fcinfo)` 宏本质上调用 `fcinfo->flinfo->fn_addr(fcinfo)`。
解释器为了减少 indirection，还把 `fn_addr` 复制到 step 中。
如果 function stats 需要开启，`EEOP_FUNCEXPR_FUSAGE` 路径会在调用前后处理 `pgstat_init_function_usage()` 和 `pgstat_end_function_usage()`。
如果是 security definer 或带 `proconfig` 的函数，`fn_addr` 可能不是用户函数本体。
它可能是 `fmgr_security_definer`。
这个 wrapper 会切换 userid、设置 local GUC、调用内部 `FmgrInfo`，然后恢复状态。
所以 profile 中看到的 `fmgr_security_definer` 不代表 SQL 函数体不存在。
它代表调用入口被安全 wrapper 接管了。
## 6. 生命周期 / ownership / cleanup
### 6.1. lookup 阶段的 syscache tuple
`oper()` 返回 syscache tuple。
调用者负责 `ReleaseSysCache()`。
`func_get_detail()` 自己在读取 `pg_proc` 字段后释放 tuple。
`lsyscache.c` helper 通常在函数内部完成 search 和 release。
这类 tuple 不跨阶段保存。
表达式树保存的是 OID、类型 OID、bool 和 collation。
### 6.2. parser lookaside cache
`parse_oper.c` 中 operator lookaside cache 是 backend-local 的小 cache。
它用来避免 parse analysis 中重复 operator lookup。
它依赖 syscache invalidation callback 清理。
它不属于 plan。
它也不提供 executor hot path 的调用状态。
如果把它理解成“operator 执行缓存”，诊断方向会错。
### 6.3. analyzed expression tree
`OpExpr`、`FuncExpr` 等节点通常分配在 query parse/analyze 的 memory context 中。
如果进入 plan cache，它们会被拷贝或保存到 `CachedPlanSource` 管理的上下文。
它们保存语义 OID。
它们不保存 `FmgrInfo`。
这是 plan cache 能够在 invalidation 后重新规划的原因之一。
如果表达式树里保存的是裸 C 函数指针，就无法可靠跨 catalog 变更。
### 6.4. planner support 调用状态
support function 调用通常是短生命周期。
planner 构造 request node，调用 support function，然后读取返回结果。
request node 所在内存上下文由 planner 当前上下文管理。
support function 返回的 node 必须符合调用者对内存上下文的预期。
它不应该把指向短生命周期栈对象的指针当作长期 plan 状态返回。
### 6.5. `FmgrInfo` ownership
`fmgr_info()` 使用 `CurrentMemoryContext` 作为 `fn_mcxt`。
`fmgr_info_cxt()` 允许调用者指定上下文。
源码注释强调：如果 `FmgrInfo` 要存进长期对象，就应该使用 `fmgr_info_cxt()`。
执行器表达式初始化就是典型场景。
`FmgrInfo` 应随 `ExprState` 或 executor context 生命周期存在。
`fn_extra` 里的缓存也挂在 `fn_mcxt` 下。
### 6.6. `FunctionCallInfo` ownership
`FunctionCallInfo` 参数帧在 executor init 阶段为固定参数个数分配。
每次 tuple 求值只更新里面的 `args` 和 `isnull`。
这个对象不属于 per-tuple memory。
它的参数 Datum 可能指向 per-tuple context 中的短生命周期值。
所以函数如果要跨调用保存 pass-by-reference 参数，必须复制到自己的长期上下文。
### 6.7. ERROR / abort 收尾
表达式求值中的 ERROR 会跳出当前执行栈。
per-tuple memory 由 `ExprContext` reset 或 executor cleanup 回收。
executor context 最终由 `ExecutorEnd()` 删除。
syscache pin/refcount 由各 lookup 路径的 release 或 ResourceOwner 兜底。
security definer wrapper 中的 userid/GUC 状态在正常路径主动恢复。
错误路径依赖 transaction/subtransaction abort 和 `PG_TRY/PG_CATCH` 中恢复 `fcinfo->flinfo`。
这里要区分：
```text
MemoryContext 管内存；
ResourceOwner 管外部资源和 pins；
invalidation 管语义过期；
fmgr wrapper 管调用期间的 userid/GUC/统计状态。
```
## 7. 正确性机制层次
### 7.1. namespace 和 search_path 正确性
未 schema-qualified 的函数和 operator 必须按当前 search path 查找。
这保证 SQL 名字解析符合用户可见性规则。
但 search_path 只在 lookup 阶段参与。
执行期不会每 tuple 检查 search_path。
prepared statement 或 cached plan 需要通过 plan cache 的 search_path match 和 invalidation 机制决定是否重分析或重规划。
### 7.2. type coercion 正确性
overload resolution 不是只比较 `argtype == declared_type`。
`unknown` literal、domain、binary coercible、implicit cast、variadic/default/named argument 都可能改变候选选择。
一旦选择完成，parser 会把必要 coercion 节点插入表达式树。
执行器按树执行。
执行器不重新推导类型。
### 7.3. syscache 和 catalog visibility
lookup 通过 syscache/catcache 读取 `pg_operator`、`pg_proc`、`pg_type`、`pg_namespace` 等 catalog。
syscache 保证 backend 看到符合当前 snapshot/命令边界的 catalog 元组。
shared invalidation 负责把过期 cache entry 标记掉。
这不是锁。
invalidation 不阻止另一个事务修改 catalog。
它只保证后续 lookup 不继续信任过期 cache。
### 7.4. plan cache invalidation
`OpExpr` 和 `FuncExpr` 保存的 OID 可能因为 DROP/ALTER 相关对象而失效。
prepared statement 的 `CachedPlanSource` 记录依赖。
catalog invalidation 到来时，plan source 或 generic plan 会被标记 invalid。
下一次执行会重新分析、rewrite 或重新规划。
这就是为什么 executor expression state 不需要处理 catalog invalidation。
executor state 属于一次执行。
长期语义缓存属于 plan cache。
### 7.5. fmgr call ABI 正确性
所有 fmgr-compatible C function 都接收 `PG_FUNCTION_ARGS`。
调用者通过 `FunctionCallInfo` 传递参数、null flags、collation、context 和 resultinfo。
strict 函数的 null 输入检查由调用者负责。
`FunctionCallInvoke()` 不会自动检查 strict。
执行器提前选择 strict opcode，就是为了把这个责任放在表达式解释器里。
### 7.6. security definer / proconfig 正确性
带 `prosecdef` 或 `proconfig` 的函数不能直接调用用户函数地址。
`fmgr_info_cxt_security()` 会把 `fn_addr` 设置为 `fmgr_security_definer`。
wrapper 内部再构造真实函数的 `FmgrInfo`。
调用时切换 userid 和 GUC。
错误路径依赖事务 abort 和 wrapper 的 `PG_TRY/PG_CATCH` 保证调用帧恢复。
### 7.7. function stats 正确性
`track_functions` 不是所有函数调用都无条件计数。
`FmgrInfo.fn_stats` 和表达式 opcode 决定是否进入 `_FUSAGE` 路径。
统计是观测机制，不改变函数语义。
但它会改变调用开销和 profile 形态。
### 7.8. support function 正确性边界
support function 只能提供优化信息。
它不能让 planner 产生与原表达式不等价的计划。
`SupportRequestIndexCondition` 返回的 index condition 必须能作为原函数条件的安全推导。
`SupportRequestSimplify` 返回的表达式必须等价。
如果 support function 返回 NULL，系统必须退回通用规划路径。
这是 fallback，也是 correctness 边界。
## 8. 错误路径 / 异常路径 / fallback
### 8.1. operator 不存在
如果 exact match 和候选选择都失败，`oper()` 在 `noError=false` 时调用 `op_error()`。
常见错误是：
```text
operator does not exist: integer = text
```
这类错误发生在 parse analysis。
执行器还没有启动。
因此 EXPLAIN ANALYZE、executor hook、function stats 都看不到它。
### 8.2. ambiguous operator
候选存在但 `func_select_candidate()` 不能选出唯一最佳候选，会报 ambiguous。
典型原因是 unknown literal、多个 schema 中同名 operator、扩展引入重载或参数需要同等级 coercion。
解决方式通常是显式 cast 或 schema qualification。
这不是执行期分支选择失败。
### 8.3. function 不存在或不唯一
`LookupFuncName()` 在找不到时可以按 `missing_ok` 返回 `InvalidOid`。
但 ambiguous function 即使 `missing_ok=true` 也会报错。
因为“不唯一”不是“缺失”。
调用者如果需要容忍对象不存在，仍不能容忍语义不唯一。
### 8.4. function-like cast fallback
`func_get_detail()` 特别处理 `typename(expr)`。
如果函数名恰好是类型名，并且可以按允许的 coercion path 处理，它会返回 `FUNCDETAIL_COERCION`。
这解释了为什么 `int4('1')` 在语义上可能不是普通函数调用。
parser 会把它转成 coercion，而不是 `FuncExpr`。
### 8.5. default / named / variadic 失败
`FuncnameGetCandidates()` 和 `func_get_detail()` 会处理 named/default/variadic。
如果 named argument 不匹配，或者 variadic 用法和 named notation 组合不合法，候选会被过滤或标记失败。
错误消息通常来自 parse 阶段。
执行器不会知道当初有几个 default argument 被补上。
它只看到最终参数表达式列表。
### 8.6. syscache lookup failed
如果表达式树或 plan 中保存的 function OID 在初始化时查不到，`fmgr_info_cxt_security()` 会 `elog(ERROR, "cache lookup failed for function ...")`。
正常情况下，plan cache invalidation 应该避免长期缓存引用已删除对象。
如果走到这里，常见解释是内部状态、扩展、手工 catalog 修改或 invalidation 边界被破坏。
这类错误通常不是用户 SQL overload ambiguity。
### 8.7. C function address lookup 失败
对于 C language function，fmgr 需要找到外部符号。
builtin function 走 `fmgr_isbuiltin()` fast path。
external C function 会通过动态加载和 `CFuncHash` 缓存地址。
如果库不存在、符号不存在或 info record 不匹配，会在首次初始化或调用相关路径报错。
这类错误发生在 fmgr 层，不在 parser 重载解析层。
### 8.8. security wrapper ERROR
security definer 或带 `SET` 配置的函数调用时，wrapper 会切换环境。
如果函数体 ERROR，wrapper 的 `PG_CATCH` 会恢复 `fcinfo->flinfo` 并触发 hook abort。
GUC 和 userid 的彻底恢复依赖事务/subtransaction abort。
这解释了源码注释中为什么有些状态不在 catch 中逐项恢复。
### 8.9. support function 返回 NULL
support function 不支持某个 request，或者判断当前表达式不能优化，应该返回 NULL。
planner 调用者必须把 NULL 当作“没有额外优化信息”。
这不是错误路径。
这是正常 fallback。
例如函数不能转成 index condition 时，planner 继续保留原函数 filter。
### 8.10. strict function null short-circuit
strict 函数遇到任一 NULL 参数，执行器直接返回 NULL。
这不是错误。
也不会调用函数体。
因此在 profile 或自定义函数日志中，看不到这些调用。
如果用户以为函数“应该被调用并处理 NULL”，需要检查 `pg_proc.proisstrict`。
## 9. 成本、资源与跨模块传播
### 9.1. parse-time lookup 成本
operator/function lookup 成本主要随这些变量扩张：
| 变量 | 影响 |
| --- | --- |
| 候选函数/操作符数量 | 更多 overload 需要更多候选过滤和启发式选择。 |
| search_path 长度 | 未限定名字需要按 namespace 可见性查找。 |
| named/default/variadic 使用 | 候选生成和参数重排更复杂。 |
| unknown literal 数量 | 需要更多类型推断和歧义处理。 |
| catalog cache miss | 需要访问系统 catalog，可能触发 syscache/catcache 构建。 |
这部分成本通常发生在 parse analysis。
prepared statement 可以摊薄它。
但 prepared statement 也会引入 invalidation 和 generic/custom plan 判断。
### 9.2. planner support 成本
support function 让 planner 获得更准的 cost、rows、selectivity 或 index condition。
代价是多一次 fmgr 调用和 request 构造。
如果 support function 做大量 catalog lookup 或复杂计算，规划时间会上升。
这个成本随 query 中相关函数表达式数量扩张。
它不随输出 tuple 数直接扩张。
### 9.3. executor init 成本
`ExecInitExprRec()` 会为每个函数/operator 调用初始化 `FmgrInfo` 和 `FunctionCallInfo`。
这个成本随表达式树中的 call site 数量扩张。
它通常是每次 executor start 一次。
prepared generic plan 不能直接复用上一轮 executor 的 `FmgrInfo`。
因为 `FmgrInfo` 是 executor state，不是 plan tree 的一部分。
### 9.4. execution hot path 成本
每 tuple 函数调用成本包括：
```text
argument expression evaluation
  + null/strict checks
  + FunctionCallInfo arg writes
  + indirect call through fn_addr
  + optional function stats
  + function body cost
```
operator 调用也是函数调用。
`a = b` 如果在 filter 中执行一亿次，underlying equality function 的 fmgr overhead 和函数体本身都会进入 CPU profile。
这就是为什么表达式解释器、JIT、strict opcode、内联 builtin、sort support 等优化都很重要。
### 9.5. function stats overhead
开启 `track_functions` 会让可跟踪函数进入 usage 统计路径。
统计粒度是函数累计，不是单个 expression call site。
它能帮助发现热点函数。
但它不能告诉你某个函数是在 join qual、projection、index qual 还是 trigger 中被调用。
此外统计本身会增加调用开销。
对极高频小函数，这个扰动可能可见。
### 9.6. C function cache 成本
external C function 第一次解析需要动态库和符号查找。
`CFuncHash` 用 function OID、`xmin`、`tid` 等信息判断缓存地址是否仍然新鲜。
后续调用通常不再重复动态符号查找。
这个 cache 是 backend-local。
不同 backend 会各自建立。
### 9.7. typcache 成本
排序、hash、group、join 和 partition pruning 依赖 typcache 中的 operator/procedure。
第一次查某个 type 的某类能力会触发 catalog lookup。
后续在同 backend 中复用 `TypeCacheEntry`。
typcache invalidation 会清理或标记过期。
因此 workload 中如果不断触达大量不同类型、opclass 或 partition key，typcache 也可能成为 lookup 成本来源。
### 9.8. 跨模块传播
本节至少连接这些模块：
| 模块 | 传播方式 |
| --- | --- |
| parser/analyzer | 把 SQL 名字、候选、coercion 折叠成 OID 和表达式树。 |
| namespace/syscache | 提供 search_path visibility 和 catalog tuple lookup。 |
| planner | 使用 support function、volatility、strictness、selectivity、cost 和 typcache 信息。 |
| plan cache | 保存 analyzed/rewrite/plan 状态，并根据 catalog invalidation 重建。 |
| executor expression | 把表达式节点编译成 `ExprEvalStep`。 |
| fmgr | 把 function OID 变成 `FmgrInfo.fn_addr` 和调用 ABI。 |
| pg_stat | 可选记录 function usage。 |
| extension | 可创建 operator、function、support function、C symbol 和 fmgr hook。 |
这个链路没有 WAL。
普通函数调用也不涉及 shared memory 状态推进。
catalog 修改本身当然会写 WAL 和发 invalidation。
但本节讨论的是 lookup 和执行状态转换，不是 DDL 的持久化。
## 10. 观测与诊断入口
### 10.1. 看 parse 后选中的 operator/function
最直接的方法是看 `EXPLAIN (VERBOSE)`。
它会展示部分表达式。
但它不总是直接展示 OID。
需要精确确认时，可以查 catalog。
例如：
```sql
SELECT o.oid AS op_oid,
       o.oprname,
       o.oprleft::regtype,
       o.oprright::regtype,
       o.oprcode::regprocedure
FROM pg_operator o
WHERE o.oprname = '='
  AND o.oprleft = 'integer'::regtype
  AND o.oprright = 'integer'::regtype;
```
普通函数：
```sql
SELECT p.oid::regprocedure,
       p.proisstrict,
       p.proretset,
       p.provolatile,
       p.proparallel,
       p.prosupport::regprocedure
FROM pg_proc p
WHERE p.oid = 'lower(text)'::regprocedure;
```
这些查询看到的是 catalog metadata。
它们不是执行期 `FmgrInfo`。
### 10.2. 看函数统计
开启 function stats：
```sql
SET track_functions = 'all';
SELECT lower('ABC');
SELECT funcid::regprocedure, calls, total_time, self_time
FROM pg_stat_user_functions
ORDER BY calls DESC;
```
注意边界：
`pg_stat_user_functions` 只统计 user-defined functions。
许多 internal/builtin 函数不会以你期望的形式出现在这里。
表达式内建 operator 的 underlying function 也可能因为 builtin fast path 或统计设置而不可见。
### 10.3. 用 `pg_stat_statements` 看查询层成本
`pg_stat_statements` 能看 query 层累计时间。
它看不到每个 function call site。
如果一个 query 变慢，而 `pg_stat_user_functions` 中某个函数 total_time 同步上升，可以推断函数体贡献较大。
但不能仅凭这两个视图证明 operator lookup 慢。
operator lookup 多发生在 parse/planning。
需要结合 prepared statement、planning time、perf 或 debug log 判断。
### 10.4. 用 `EXPLAIN (ANALYZE)` 区分 planning 和 execution
`EXPLAIN (ANALYZE, VERBOSE)` 能展示 planning time 和 execution time。
如果函数 lookup 或 support function 很贵，通常体现在 planning time。
如果 underlying function 每 tuple 调用很贵，通常体现在 execution time。
但 `EXPLAIN` 不会把每个 expression function 的耗时拆出来。
需要 profiler 或源码插桩。
### 10.5. perf / flamegraph
CPU 热点可能出现这些符号：
```text
ExecInterpExpr
ExecInterpExpr
ExecEvalFuncExprFusage
FunctionCallInvoke
fmgr_security_definer
pgstat_init_function_usage
text_eq / int4eq / btint4cmp / lower
```
如果热点在 `FuncnameGetCandidates()`、`OpernameGetCandidates()`、`SearchSysCache`，说明 parse/planning/catalog lookup 成本更可疑。
如果热点在 `ExecInterpExpr` 的 `EEOP_FUNCEXPR*` case、`ExecEvalFuncExprFusage` 或具体函数体，说明执行期 call site 更可疑。
如果热点在 `fmgr_security_definer`，要检查 `prosecdef`、`proconfig` 和 fmgr hook。
### 10.6. gdb 断点
源码跟踪建议断点：
```text
make_op
oper
func_get_detail
LookupFuncName
get_func_support
ExecInitFunc
fmgr_info_cxt_security
ExecInterpExpr
ExecEvalFuncExprFusage
fmgr_security_definer
```
观察问题：
```text
parse 阶段选中了哪个 operator/function OID？
OpExpr.opfuncid 是否等于 pg_operator.oprcode？
ExecInitFunc 什么时候调用 fmgr_info？
FmgrInfo.fn_addr 是用户函数还是 wrapper？
strict 函数传 NULL 时是否进入函数体？
```
### 10.7. 能直接观测、只能推断、几乎不可见
能直接观测：
```text
pg_proc / pg_operator catalog metadata
pg_stat_user_functions 累计函数统计
EXPLAIN planning/execution time
perf 栈样本
错误消息中的 function/operator signature
```
只能推断：
```text
某个 SQL 名字解析时曾经有哪些候选
某个 support function 是否改变了某个成本估算
某个 cached plan 是否因为函数 catalog 变化失效
```
几乎不可见：
```text
每个 ExprEvalStep 的 FmgrInfo 具体地址
每 tuple 的 FunctionCallInfo 参数写入
parser lookaside operator cache 命中率
```
这些需要 gdb、debug build、临时日志或源码插桩。
### 10.8. 诊断闭环示例
现象：
```text
同一条 SQL 在未显式 cast 时偶尔报 ambiguous function。
```
解释路径：
```text
unknown literal + overload candidates
  -> FuncnameGetCandidates() 返回多个可见候选
  -> func_match_argtypes() 后仍不唯一
  -> func_select_candidate() 无法选 best
  -> parse analysis ERROR
```
回到源码：
```text
parse_func.c: func_get_detail()
parse_func.c: func_match_argtypes()
parse_func.c: func_select_candidate()
```
验证方式：
```sql
SELECT oid::regprocedure, pronamespace::regnamespace
FROM pg_proc
WHERE proname = 'your_function_name';
```
显式 cast 或 schema qualification 后，候选集合变窄，错误消失。
## 11. 常见误区
误区一：operator 是执行期按名字调用的。
实际不是。
operator 名字在 parse analysis 中解析成 `pg_operator` OID。
执行期调用的是 `pg_operator.oprcode` 对应的 function。
误区二：`opno` 和 `opfuncid` 是重复字段。
实际不是。
`opno` 保留 operator 语义。
`opfuncid` 服务执行调用。
planner、deparse、selectivity 和 opclass 关系可能仍需要 `opno`。
误区三：support function 是 SQL 函数执行的 helper。
实际 support function 服务 planner。
它回答 simplify、cost、rows、selectivity、index condition 等问题。
它不在每个 tuple 执行 SQL 函数时自动调用。
误区四：`FmgrInfo` 是全局函数缓存。
实际 `FmgrInfo` 是 backend-local、调用者生命周期内的 callable state。
它可能挂在 executor context、relation cache、typcache 或 access method state 下。
不能跨 backend 共享裸指针。
误区五：`fn_addr` 一定指向用户函数本体。
实际可能指向 `fmgr_security_definer` 或 language handler wrapper。
需要看 `prosecdef`、`proconfig`、`prolang` 和 hook。
误区六：strict 函数没有出现在日志里说明没被表达式求值。
实际可能是参数为 NULL，被 strict opcode 短路了。
这时函数体没有被调用，但表达式 step 确实被执行。
误区七：invalidation 会阻止并发 catalog 修改。
实际 invalidation 是过期通知。
它不是 lock。
它不会让执行期每 tuple 重新检查 catalog。
误区八：`pg_stat_user_functions` 能解释所有 operator/function 成本。
实际它只提供函数级累计统计，且受 `track_functions` 和函数类型影响。
operator underlying builtin、表达式 call site 和 support function 调用常常需要 perf 或插桩。
## 12. 课堂实验
### 实验 1：从 operator catalog 查到 underlying function
执行：
```sql
SELECT o.oid AS operator_oid,
       o.oprname,
       o.oprleft::regtype AS left_type,
       o.oprright::regtype AS right_type,
       o.oprcode::regprocedure AS call_function
FROM pg_operator o
WHERE o.oprname = '='
  AND o.oprleft = 'integer'::regtype
  AND o.oprright = 'integer'::regtype;
```
再执行：
```sql
EXPLAIN (VERBOSE)
SELECT *
FROM generate_series(1, 10) AS g(i)
WHERE i = 5;
```
目标：把 SQL 中的 `=` 和 `pg_operator.oprcode` 关联起来。
源码回读：
```text
parse_oper.c: make_op()
parse_oper.c: oper()
parse_oper.c: oprfuncid()
execExpr.c: ExecInitExprRec()
```
### 实验 2：观察 ambiguous function 与显式 cast
创建两个重载函数：
```sql
CREATE SCHEMA lookup_lab;
SET search_path = lookup_lab, public;
CREATE FUNCTION f_lookup_lab(x int) RETURNS text
LANGUAGE sql IMMUTABLE
AS $$ SELECT 'int' $$;
CREATE FUNCTION f_lookup_lab(x text) RETURNS text
LANGUAGE sql IMMUTABLE
AS $$ SELECT 'text' $$;
```
尝试：
```sql
SELECT f_lookup_lab('1');
SELECT f_lookup_lab('1'::int);
SELECT f_lookup_lab('1'::text);
```
不同版本和上下文下，unknown literal 的选择可能受候选规则影响。
目标不是记住某个结果。
目标是用源码解释为什么显式 cast 能缩小候选集合。
源码回读：
```text
parse_func.c: func_get_detail()
parse_func.c: func_match_argtypes()
parse_func.c: func_select_candidate()
```
### 实验 3：strict 函数 NULL 短路
创建一个 strict 函数：
```sql
CREATE OR REPLACE FUNCTION lookup_lab.strict_notice(x int)
RETURNS int
LANGUAGE plpgsql
STRICT
AS $$
BEGIN
  RAISE NOTICE 'called with %', x;
  RETURN x;
END;
$$;
```
执行：
```sql
SELECT lookup_lab.strict_notice(1);
SELECT lookup_lab.strict_notice(NULL::int);
```
预期：第二条不会打印 notice。
源码解释：
```text
pg_proc.proisstrict
  -> FmgrInfo.fn_strict
  -> EEOP_FUNCEXPR_STRICT*
  -> EEOP_FUNCEXPR_STRICT* null short-circuit
```
### 实验 4：观察 function stats 粒度
执行：
```sql
SET track_functions = 'all';
SELECT lookup_lab.strict_notice(i)
FROM generate_series(1, 1000) AS g(i);
SELECT funcid::regprocedure, calls, total_time, self_time
FROM pg_stat_user_functions
WHERE funcid = 'lookup_lab.strict_notice(int)'::regprocedure;
```
再执行含 NULL 的版本：
```sql
SELECT lookup_lab.strict_notice(NULLIF(i, i))
FROM generate_series(1, 1000) AS g(i);
```
观察 calls 是否符合 strict 短路预期。
注意：统计刷新和事务边界可能造成短暂延迟。
### 实验 5：源码断点追 `FmgrInfo`
在 debug build 中启动 backend，设置断点：
```text
b ExecInitFunc
b fmgr_info_cxt_security
b ExecInterpExpr
b ExecEvalFuncExprFusage
```
执行：
```sql
SELECT lookup_lab.strict_notice(i)
FROM generate_series(1, 3) AS g(i);
```
观察：
```text
ExecInitFunc 调用次数是否等于表达式 call site，而不是输出行数？
fmgr_info_cxt_security 何时填 fn_addr？
EEOP_FUNCEXPR* 每行是否复用同一个 FmgrInfo？
```
### 实验 6：support function catalog 探查
查询带 support function 的内置函数：
```sql
SELECT oid::regprocedure, prosupport::regprocedure
FROM pg_proc
WHERE prosupport::oid <> 0
ORDER BY 1
LIMIT 20;
```
选择一个函数，查它在 `EXPLAIN` 中是否影响估算或 index condition。
源码回读：
```text
lsyscache.c: get_func_support()
optimizer/util/clauses.c
optimizer/util/plancat.c
optimizer/path/indxpath.c
include/nodes/supportnodes.h
```
目标：确认 support function 是 planner metadata 入口，而不是执行期 helper。
## 13. 讨论题
1. 为什么 `OpExpr` 同时保存 `opno` 和 `opfuncid`，而不是只保存 underlying function OID？
2. 如果 executor 每 tuple 都按函数名和参数类型重新查 `pg_proc`，会破坏哪些性能和正确性边界？
3. `FmgrInfo.fn_oid` 为什么要最后设置？这反映了哪种初始化不变量？
4. support function 返回 NULL 时，planner 应该如何 fallback？为什么这不能影响 SQL correctness？
5. `track_functions` 能证明某个 operator underlying function 被调用了多少次吗？有哪些盲区？
6. search_path 改变后，prepared statement 为什么不能简单继续使用旧的名字解析结果？
7. security definer 函数的 `fn_addr` 为什么可能不是用户函数本体？这对 profiling 有什么影响？
8. strict 函数的 null short-circuit 应该放在函数体里做，还是由调用者做？PostgreSQL 当前设计的成本收益是什么？
## 14. 本节小结
本节主链路是：
```text
SQL name
  -> namespace/syscache candidate lookup
  -> overload resolution and coercion
  -> OpExpr / FuncExpr OID state
  -> planner support metadata
  -> ExecInitExprRec / ExecInitFunc
  -> FmgrInfo / FunctionCallInfo / ExprEvalStep
  -> ExecInterpExpr fn_addr call
```
核心状态边界是：
```text
名字只活在 parse/DDL lookup 阶段；
OID 活在 analyzed query、plan 和 catalog dependency 中；
FmgrInfo 活在 backend-local executor 或 cache owner 生命周期中；
FunctionCallInfo 是每个 call site 复用的调用帧；
fn_addr 是执行 hot path 的最终入口。
```
ownership 和 cleanup 的关键是：
syscache tuple 要及时 release。
analyzed expression tree 由 query/plan cache context 管。
support request 由 planner 当前上下文管。
`FmgrInfo.fn_extra` 挂在 `fn_mcxt`。
executor expression state 随 executor context 清理。
ERROR 路径依赖 MemoryContext、ResourceOwner、transaction abort 和 fmgr wrapper 的局部恢复共同收尾。
正确性不是一个机制保证的。
namespace/search_path 决定名字可见性。
type coercion 决定 overload 语义。
syscache 和 invalidation 决定 catalog 元数据不过期。
plan cache 决定长期语义状态何时重建。
fmgr ABI 决定执行期如何传 Datum、NULL、collation 和 context。
能观测的是 catalog 元数据、函数级统计、planning/execution time、错误消息和 profiler 栈。
不能直接观测的是每个 call site 的候选集合、`ExprEvalStep` 内部地址、parser lookaside cache 命中率和每 tuple `FunctionCallInfo` 写入。
本节可迁移规律：
```text
当一个系统既要支持动态命名、重载和元数据变更，
又要让高频执行路径足够便宜时，
通常会把“语义解析状态”和“可调用执行状态”分层缓存；
前者通过 invalidation 保持正确，
后者通过短生命周期 ownership 和预编译 step 降低 hot path 成本。
```
这些判断依赖 PostgreSQL 版本、函数语言、扩展、search_path、plan cache 状态、`track_functions`、JIT 和 workload。
不要把当前 profile 中的一个符号直接解释成完整因果。
先问它位于哪一层：lookup、support、executor init、fmgr wrapper，还是函数体本身。
