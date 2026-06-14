# PostgreSQL 类型转换、操作符与函数查找

## 课程定位
前置知识：已经理解 raw parser 只生成语法树，parse analysis 负责把名字、类型和语义对象绑定到 `Query`。
本节唯一主问题：

```text
类型推断、隐式 coercion、operator/function lookup 如何把语法表达式变成可执行语义？
```
核心矛盾：

```text
SQL 允许 unknown literal、重载函数、重载操作符、domain、多态伪类型和上下文相关写法； executor 却必须拿到确定的类型 OID、函数 OID、操作符 OID、collation 和 coercion 节点。
```
学完后应能判断：

```text
unknown 字符串何时被定型； implicit / assignment / explicit coercion 的边界在哪里； 一个函数或操作符为什么解析到某个 pg_proc / pg_operator 行； 为什么某些表达式在 parse analysis 就报错； 这些语义对象如何传播到 planner 和 executor。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e`。
重点源码：

```text
src/backend/parser/parse_expr.c src/backend/parser/parse_node.c src/backend/parser/parse_coerce.c src/backend/parser/parse_oper.c src/backend/parser/parse_func.c src/backend/parser/parse_type.c src/backend/parser/parse_param.c src/backend/utils/cache/lsyscache.c src/include/parser/parse_coerce.h src/include/parser/parse_func.h src/include/parser/parse_oper.h src/include/catalog/pg_cast.h src/include/catalog/pg_operator.h src/include/catalog/pg_proc.h src/include/catalog/pg_type.h
```
本节不讲 raw grammar，也不讲 planner 如何估算选择率。
本节只讲 parser/analyzer 如何把一个还带模糊性的表达式收束为唯一可执行语义。

## 1. 本节在总主线中的位置
05 目录前面的 planner 课程默认输入已经是语义化的 `Query`。
第 56 到 60 节补齐 planner 之前的路径。
本节处在 name resolution 之后、rewrite 和 planner 之前。
上一节回答：

```text
ColumnRef 指向哪个 range table entry、列、alias 或 whole-row reference？
```
本节回答：

```text
表达式中的 literal、函数、操作符和 cast 最终是什么类型，调用哪个 catalog 对象？
```
后续 planner 看到的不是 SQL token。
planner 看到的是：

```text
FuncExpr.funcid OpExpr.opno OpExpr.opfuncid RelabelType.resulttype CoerceViaIO.resulttype CoerceToDomain.resulttype Const.consttype Param.paramtype
```
这意味着 analyzer 的类型决定会影响后续所有阶段。
例如：

```sql
SELECT '1' + 2;
SELECT '1' || 2;
SELECT int_col = '42' FROM t;
SELECT int_col::text = '42' FROM t;
```
这些 SQL 的 raw tree 只是保存 literal、操作符名和语法位置。
parse analysis 必须决定：

```text
'1' 是 unknown、int4 还是 text？ + 是 int4pl 还是 numeric_add？ || 是 textcat 还是 array cat？ = 是 int4eq 还是 texteq？ cast 插在常量侧还是列侧？
```
这些决定不是 cosmetic。
它们会改变选择率函数、索引匹配、表达式简化、函数执行路径和错误时机。
本节的主线是：

```text
raw expression -> transformExpr() -> actual input type OIDs -> function/operator candidate lookup -> implicit coercion feasibility -> candidate selection -> polymorphic consistency -> coercion node insertion -> typed Expr tree
```

## 2. 核心矛盾与一句话运行模型
一句话运行模型：

```text
parse analysis 先给子表达式生成初步类型，再用实际输入类型查找函数或操作符候选， 通过 context-limited coercion 和重载选择得到唯一 catalog OID， 最后把必要的 coercion 节点插回 expression tree。
```
这里有三个约束。
第一，SQL 必须容易写。
用户可以写：

```sql
SELECT '42' + 1;
SELECT date '2026-01-01';
INSERT INTO t_int VALUES ('42');
```
第二，执行语义必须确定。
executor 不能执行“某个叫 `+` 的东西”。
executor 需要：

```text
operator OID underlying function OID left input type right input type result type collation
```
第三，隐式转换必须保守。
如果所有 explicit cast 都能自动参与重载解析，很多 SQL 会变得歧义或意外改变语义。
PostgreSQL 把转换能力分层：

```text
implicit expression coercion assignment coercion explicit cast PL/pgSQL assignment fallback
```
函数和操作符 lookup 通常只接受 `COERCION_IMPLICIT`。
目标列赋值使用 assignment 边界。
显式 `CAST` 和 `::` 使用 explicit 边界。
这形成本节的核心 tension：

```text
宽松 SQL 表达能力 vs 唯一、稳定、可执行、可诊断的 catalog-bound 语义
```
PostgreSQL 的解决方式是一组顺序化规则，而不是一个万能 cast。
常见决策顺序：

```text
exact match -> binary-compatible match -> pg_cast allowed by current CoercionContext -> type category and preferred type heuristic -> unknown literal special handling -> polymorphic consistency -> ambiguity or undefined function/operator error
```
理解这条顺序，比背每个 catalog 表更重要。

## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `parse_expr.c` | raw expression 转 typed expression；函数、操作符、cast 的入口分派。 |
| 2 | `parse_node.c` | `A_Const` 转 `Const`，字符串 literal 的 `UNKNOWNOID` 起点。 |
| 3 | `parse_func.c` | `ParseFuncOrColumn()`、`func_get_detail()`、函数候选裁剪和 `FuncExpr` 构造。 |
| 4 | `parse_oper.c` | `make_op()`、`oper()`、operator 候选解析和 `OpExpr` 构造。 |
| 5 | `parse_coerce.c` | `can_coerce_type()`、`coerce_type()`、`find_coercion_pathway()`、common type、多态一致性。 |
| 6 | `parse_param.c` | unknown external Param 的类型推断和最终校验。 |
| 7 | `parse_type.c` | 类型名、typmod、`LookupTypeName()` 等类型解析辅助。 |
| 8 | `lsyscache.c` | catalog tuple 上的轻量 accessor。 |
| 9 | `parse_coerce.h` | coercion API 和 `CoercionPathType`。 |
| 10 | `parse_func.h` | `FuncDetailCode` 和函数 lookup 对外接口。 |
| 11 | `parse_oper.h` | operator lookup 对外接口。 |
| 12 | `pg_cast.h` | `castsource`、`casttarget`、`castfunc`、`castcontext`、`castmethod`。 |
| 13 | `pg_operator.h` | `oprleft`、`oprright`、`oprresult`、`oprcode`、selectivity 元数据。 |
| 14 | `pg_proc.h` | `proargtypes`、`prorettype`、`proretset`、`provariadic`、default 参数和函数属性。 |
| 15 | `pg_type.h` / `pg_type.dat` | type category、preferred type、polymorphic pseudo-type、`unknown`。 |
推荐阅读方式：

```text
入口函数 -> 输入 type OID -> 候选集合 -> coercion context -> 候选选择 -> Expr node 输出 -> 错误路径
```
不要从 `pg_cast.dat` 或 `pg_proc.dat` 开始背数据。
catalog row 只是事实。
一次真实解析还依赖 search path、实际参数类型、unknown 状态、coercion context、domain、多态规则和错误位置。

## 4. 关键数据结构与状态

### `UNKNOWNOID`
`unknown` 是 parser/analyzer 使用的 pseudo-type。
它不是普通业务列类型。
字符串 literal 在缺少上下文时通常先成为：

```text
Const consttype = UNKNOWNOID constvalue = literal text
```
这个状态保留了“稍后由上下文决定类型”的机会。
例如：

```sql
SELECT '42'::int4;
SELECT '42' + 1;
INSERT INTO t_int4 VALUES ('42');
SELECT 'abc';
```
前几条会把 unknown 定成目标上下文需要的类型。
最后一条在顶层输出时通常会解析成 `text`。
重要边界：

```text
unknown 可以隐式转向很多目标类型； unknown 不能无限期留给 executor； unknown literal 和 unknown Param 的处理不同； unknown 的存在会参与函数/operator 重载选择。
```

### `CoercionContext`
`CoercionContext` 表示这次转换允许多宽。
`pg_cast.castcontext` 的用户可见编码是：

```text
i  implicit a  assignment e  explicit
```
backend 内部常见值：

```text
COERCION_IMPLICIT COERCION_ASSIGNMENT COERCION_PLPGSQL COERCION_EXPLICIT
```
顺序有语义。
`find_coercion_pathway()` 依赖 enum ordering 判断当前 context 是否足够宽。
典型边界：

```text
函数/operator 参数匹配：通常 COERCION_IMPLICIT INSERT/UPDATE 目标列赋值：COERCION_ASSIGNMENT 显式 CAST 或 ::：COERCION_EXPLICIT PL/pgSQL assignment：没有正常路径时允许 IO fallback
```

### `CoercionPathType`
`parse_coerce.h` 中定义：

```text
COERCION_PATH_NONE COERCION_PATH_FUNC COERCION_PATH_RELABELTYPE COERCION_PATH_ARRAYCOERCE COERCION_PATH_COERCEVIAIO
```
它回答：

```text
source type 到 target type 应该怎样实现？
```
常见结果：

```text
binary compatible -> RelabelType cast function -> FuncExpr array element coercion -> ArrayCoerceExpr input/output conversion -> CoerceViaIO domain target -> CoerceToDomain may be added
```
`COERCION_PATH_RELABELTYPE` 不等于语义上完全无事发生。
它表示底层 Datum 表示可复用，但上层看到的类型、typmod 或 domain 边界变了。
如果 target 是 domain，还可能需要 domain constraint check。

### `FuncExpr`
`FuncExpr` 是普通函数调用的 typed expression 节点。
核心字段：

```text
funcid funcresulttype funcretset funcvariadic funcformat funccollid inputcollid args location
```
`funcid` 绑定 `pg_proc.oid`。
后续 planner 会读取函数 volatility、strictness、parallel safety、cost 和 support function。
executor 通过 fmgr 执行 `funcid` 对应函数。

### `OpExpr`
`OpExpr` 是操作符调用的 typed expression 节点。
核心字段：

```text
opno opfuncid opresulttype opretset opcollid inputcollid args location
```
`opno` 绑定 `pg_operator.oid`。
`opfuncid` 来自 `pg_operator.oprcode`。
这说明 operator 不是普通函数调用的纯语法糖。
执行时需要 underlying function。
优化时需要 operator OID、selectivity 函数、commutator、negator、merge/hash 标记和 operator family 关系。

### `pg_cast`
对本节最重要的列：

```text
castsource casttarget castfunc castcontext castmethod
```
`castmethod` 可以是：

```text
function binary inout
```
`castfunc = 0` 常见于 binary-compatible cast。
但一次实际转换是否可用，还要看：

```text
当前 CoercionContext； source / target 是否 domain； 是否 unknown Const； 是否 Param； 是否 array element coercion； 是否 typmod coercion； 是否 common type 选择； 是否 polymorphic declared argument。
```

### `pg_proc`
函数 lookup 关注：

```text
proname pronamespace proargtypes proallargtypes proargmodes proargnames prorettype proretset provariadic pronargdefaults prokind
```
parse analysis 要决定：

```text
调用哪个 pg_proc 行； 实际参数如何 coerced 到 declared 参数； 返回类型是否要因 polymorphic 输入而改写； 是否允许 SRF 出现在当前位置。
```

### `pg_operator`
operator lookup 关注：

```text
oprname oprnamespace oprkind oprleft oprright oprresult oprcode oprrest oprjoin oprcanmerge oprcanhash
```
parse analysis 选择 `opno` 和 `opfuncid`。
planner 后续可能使用 `oprrest`、`oprjoin`、merge/hash 标记和 operator family 做优化判断。

### polymorphic pseudo-types
常见多态类型包括：

```text
anyelement anyarray anynonarray anyenum anyrange anymultirange anycompatible anycompatiblearray anycompatiblenonarray anycompatiblerange anycompatiblemultirange
```
它们不是运行期动态类型。
调用点必须在 parse analysis 中被约束成 concrete type 或一组一致的 concrete type。
否则后续节点没有稳定返回类型。

## 5. 主流程源码 walkthrough

### 5.1 `transformExpr()` 的分派
表达式入口在 `parse_expr.c`。
核心形态：

```text
transformExpr() -> transformExprRecurse() -> T_A_Const -> T_FuncCall -> T_A_Expr -> T_TypeCast -> T_CollateClause -> other expression nodes
```
每个分支的目标相同：

```text
返回一个已经带类型语义的 Node。
```
literal 分支会生成 `Const`。
函数调用分支会进入 `ParseFuncOrColumn()`。
操作符分支会进入 `make_op()`。
显式 cast 分支会进入类型查找和 `coerce_to_target_type()`。
这一步之后，raw syntax node 不再是主要语义对象。
主要产物变成：

```text
Const Var FuncExpr OpExpr RelabelType CoerceViaIO ArrayCoerceExpr CoerceToDomain BoolExpr CaseExpr ScalarArrayOpExpr
```

### 5.2 unknown literal 的时间线
字符串 literal 起点：

```text
A_Const string -> make_const() -> Const(UNKNOWNOID, original text)
```
有目标类型时：

```text
Const(UNKNOWNOID, "42") -> coerce_type(... target int4 ...) -> call int4 input function during analysis -> Const(INT4OID, 42)
```
这解释了：

```sql
SELECT 'abc'::int4;
```
错误会在 parse/analyze 阶段出现。
因为 unknown Const 可以直接用目标类型 input function 变成目标 Datum。
普通列值 cast 不同：

```sql
SELECT text_col::int4 FROM t;
```
这里输入是运行期 expression，错误可能在 executor 逐行执行时出现。
unknown Param 又不同。
外部参数 `$1` 的类型可能由上下文推断。
如果上下文始终不足，statement 结束前会报不能确定参数类型。

### 5.3 函数调用主链路
普通函数调用主链路：

```text
transformFuncCall() -> transform each argument -> actual_arg_types = exprType(args) -> ParseFuncOrColumn() -> func_get_detail() -> FuncnameGetCandidates() -> func_match_argtypes() -> func_select_candidate() -> enforce_generic_type_consistency() -> make_fn_arguments() -> makeNode(FuncExpr)
```
`ParseFuncOrColumn()` 名字别扭，是因为 SQL 里某些写法可能像函数，也可能像复合类型字段访问。
本节只看函数路径。
`func_get_detail()` 负责查找和分类。
它处理：

```text
schema-qualified function name search_path visibility ordinary function aggregate window function procedure variadic default arguments named arguments type coercion syntax special case
```
`func_match_argtypes()` 用 `can_coerce_type(..., COERCION_IMPLICIT)` 裁剪候选。
因此候选 declared arg 不必精确等于 actual arg。
只要 actual 可以隐式变成 declared，就能保留。
`func_select_candidate()` 在多个候选里继续选择。
典型规则：

```text
优先 exact match 更多的候选； 把 domain 看作 base type 做比较； 优先 preferred type； 对 unknown 参数使用 type category； 如果已知参数类型一致，尝试把 unknown 也当作该类型。
```
选中函数后，`enforce_generic_type_consistency()` 处理多态 declared type。
然后 `make_fn_arguments()` 把实际参数 coercion 到 declared 参数类型。
最终产物：

```text
函数名 + raw args -> selected pg_proc.oid -> coerced args -> concrete return type -> FuncExpr
```

### 5.4 操作符调用主链路
二元操作符主链路：

```text
transformAExprOp() -> transform left / right -> make_op() -> exprType(left), exprType(right) -> oper() -> binary_oper_exact() -> OpernameGetCandidates() -> oper_select_candidate() -> enforce_generic_type_consistency() -> coerce operands -> makeNode(OpExpr)
```
`oper()` 先尝试 exact match。
如果一边是 unknown，`binary_oper_exact()` 会尝试借用另一边的类型。
例如：

```sql
SELECT 1 = '1';
```
常见结果是把右侧 unknown 当成左侧 `int4`，从而选择 `int4eq(int4,int4)`。
如果 exact match 失败，`oper()` 获取同名 operator 候选。
候选再经过 implicit coercion 可行性和 operator-specific 选择规则裁剪。
选中后，`make_op()` 读取：

```text
oprleft oprright oprresult oprcode
```
然后把左右操作数 coercion 到 operator declared input type。
最终产物：

```text
operator token + operands -> selected pg_operator.oid -> selected pg_operator.oprcode -> coerced operands -> OpExpr
```
这里的 `opno` 会传到 planner。
同一个 token `=` 在不同类型上对应不同 operator OID 和 selectivity 语义。

### 5.5 `coerce_to_target_type()` 与 `coerce_type()`
`coerce_to_target_type()` 是常见入口。
它先问：

```text
can_coerce_type()
```
如果不能，返回 `NULL`，由调用者产生上下文化错误消息。
如果能，再调用：

```text
coerce_type() coerce_type_typmod()
```
`coerce_type()` 构造实际表达式节点。
它处理：

```text
source 和 target 相同； target 是 polymorphic pseudo-type； unknown Const； Param type hook； CollateExpr； pg_cast pathway； domain base type； array coercion； IO coercion； typmod； 错误位置。
```
区分两个问题：

```text
can_coerce_type() -> 在当前 context 下是否允许？ coerce_type() -> 如果允许，插入什么节点？
```

### 5.6 `find_coercion_pathway()`
`find_coercion_pathway()` 是 coercion 路径中心。
输入：

```text
targetTypeId sourceTypeId CoercionContext
```
输出：

```text
CoercionPathType funcid when needed
```
它会先把 domain 归约到 base type 做路径判断。
如果 source base type 等于 target base type，返回 `COERCION_PATH_RELABELTYPE`。
随后查 `pg_cast` 的 `(castsource, casttarget)`。
如果有 cast tuple，则比较 `castcontext` 和当前 `ccontext`。
源码中依赖 enum ordering：

```text
if (ccontext >= castcontext) cast is allowed
```
再根据 `castmethod` 返回 function、binary 或 inout 路径。
如果没有 `pg_cast` 行，还会考虑数组 element coercion。
再后面有保守的 IO fallback：

```text
assignment to string type may use CoerceViaIO; explicit cast from string type may use CoerceViaIO; PL/pgSQL assignment has its own fallback.
```
重要结论：

```text
显式 cast 能成功，不代表函数/operator 参数能隐式匹配； assignment cast 能用于 INSERT，不代表能用于 operator lookup； 没有 pg_cast 行时，仍可能存在数组或受限 IO 路径。
```

### 5.7 common type 路径
有些语法不是“匹配某个函数声明”，而是“多个表达式合成一个共同类型”。
典型场景：

```text
CASE UNION / INTERSECT / EXCEPT ARRAY[...] VALUES GREATEST / LEAST COALESCE
```
核心函数：

```text
select_common_type() coerce_to_common_type() verify_common_type() select_common_typmod()
```
大致规则：

```text
所有非 unknown 类型相同 -> 选它； 否则按 type category 和 preferred type 推断； 类别冲突 -> 报错； 全部 unknown -> 通常选 text； 最后把每个输入 coercion 到 common type。
```
例子：

```sql
SELECT ARRAY[1, 2, 3];
SELECT ARRAY[1, 2.5];
SELECT CASE WHEN true THEN 'a' ELSE 'b' END;
SELECT CASE WHEN true THEN 1 ELSE 'x' END;
```
这些不是普通函数重载，但共用同一套 type category、unknown 和 coercion 边界。

### 5.8 polymorphic 一致性
函数或操作符可能声明 polymorphic 参数。
调用点必须把伪类型约束为具体类型。
主入口：

```text
check_generic_type_consistency() enforce_generic_type_consistency()
```
`check_generic_type_consistency()` 常用于候选是否可行。
`enforce_generic_type_consistency()` 会改写 declared argument type 数组，并给出具体返回类型。
例子：

```sql
SELECT array_append(ARRAY[1,2], 3);
```
调用点可推断：

```text
anyarray = int4[] anyelement = int4 return = int4[]
```
但如果输入全是未定类型：

```sql
SELECT array_append(NULL, NULL);
```
就可能无法约束多态关系。
正确行为是报错，而不是让 executor 运行时决定。

## 6. 生命周期 / ownership / cleanup

### 谁创建
当前 backend 的 parse analysis 创建 typed expression tree。
主要创建者：

```text
transformExpr() ParseFuncOrColumn() make_op() coerce_type() select_common_type()
```
catalog 候选来自 syscache / catcache。
表达式节点分配在 parse/analyze 使用的 memory context 中。

### 谁持有
最终 `Query` 持有 expression tree。
关键语义以 OID 和类型字段保存：

```text
FuncExpr.funcid OpExpr.opno OpExpr.opfuncid Const.consttype Param.paramtype RelabelType.resulttype CoerceViaIO.resulttype CoerceToDomain.resulttype
```
parse analysis 不应该把 catcache tuple 指针长期带到 planner。
它把需要的信息复制成 OID、type OID、typmod、collation 和 flag。

### 谁释放
临时候选列表、数组和中间节点跟随当前 memory context 释放。
syscache tuple 通过 `ReleaseSysCache()` 释放引用。
这里不是 buffer pin、lock 或 WAL 生命周期。
主要资源是 backend-local memory 和 catalog cache 引用。

### ERROR / abort 时
parse analysis 中的失败通常是 `ereport(ERROR)`。
ERROR 会回到 statement / transaction 错误边界。
临时 memory context 会清理已经构造的节点。
catcache 引用由错误处理路径兜底。
executor 还没有启动，因此不涉及 executor node cleanup。

### 长期缓存失效
prepared statement 或 cached plan 可能保存分析后的表达式。
这些表达式依赖函数、操作符、类型和 cast。
DDL 修改相关 catalog 对象时，不能原地修改已有 `Expr`。
系统通过 dependency 和 invalidation 让相关缓存不再被无条件复用。
本节只需记住边界：

```text
Expr node 保存 catalog OID； catalog 变化靠 invalidation 保护长期正确性； parse analysis 本身不负责运行期重解析。
```

## 7. 正确性机制层次
第一层：catalog identity。
SQL token 不是执行语义。
最终语义必须绑定到：

```text
pg_type.oid pg_proc.oid pg_operator.oid pg_cast path or cast function oid
```
第二层：namespace visibility。
函数和操作符 lookup 受 schema、search_path、可见性、默认参数和 variadic 影响。
只查 `pg_proc.proname` 或 `pg_operator.oprname` 不足以判断 SQL 会调用谁。
第三层：coercion context。
同一条 cast 在 explicit 场景可用，不代表在 implicit 场景可用。
函数/operator lookup 太宽会制造歧义。
赋值需要更宽松，但仍不能等同 explicit。
第四层：候选选择确定性。
PostgreSQL 会尽量用 exact match、preferred type、type category 和 unknown 规则解析常见表达式。
如果仍不能唯一选择，就必须报歧义。
不能随机挑一个候选。
第五层：polymorphic consistency。
多态伪类型必须在调用点落到 concrete type。
返回 `anyelement` 但输入无法约束时，不能构造稳定 `FuncExpr`。
第六层：collation。
字符串操作的类型确定后，还需要 collation。
`FuncExpr`、`OpExpr`、`RelabelType` 等节点会携带 output collation 和 input collation。
本节不展开 collation 推导，但要知道类型 OID 不是文本语义的全部。
第七层：domain constraint。
domain 不是 alias。
从 base type 到 domain 需要保留 `CoerceToDomain`，因为约束检查是语义的一部分。
第八层：planner contract。
planner 消费已经定型的表达式。
它不会重新解释 `+`、`=` 或函数名。
它读取的是 operator OID、function OID、类型、collation、volatility、selectivity 和 coercion 节点。

## 8. 错误路径 / 异常路径 / fallback

### 操作符不存在
例子：

```sql
SELECT 1 + true;
```
路径：

```text
transformAExprOp() -> make_op() -> oper() -> no exact match -> no coercible candidate -> ereport(ERROR)
```
用户通常看到：

```text
operator does not exist: integer + boolean
```
边界：

```text
parser 不会为了让 operator 成立而使用 explicit-only cast。
```

### 函数不存在
例子：

```sql
SELECT substring(42, 1, 2);
```
如果没有候选能接受实际参数或 implicit coercion 后的参数，`func_get_detail()` 返回 not found。
`ParseFuncOrColumn()` 抛出函数不存在错误。
错误位置来自 raw node location。

### 函数或操作符歧义
例子：

```sql
SELECT f('1');
```
如果 `f(int4)` 和 `f(text)` 都可行，而 unknown 规则无法唯一选择，会报 ambiguous。
正确修复方式是显式定型：

```sql
SELECT f('1'::int4);
SELECT f('1'::text);
```
这体现了一个关键规则：

```text
unknown 帮助解析常见表达式； unknown 不应该掩盖真实重载歧义。
```

### unknown Param 无法定型
例子：

```sql
PREPARE q AS SELECT $1 IS NULL;
```
如果上下文没有要求 `$1` 的具体类型，参数类型可能无法确定。
`parse_param.c` 会在有上下文时更新 unknown Param。
最终仍不能确定时，parse analysis 报错。
executor 不能接收没有类型的 Param。

### unknown Const 输入失败
例子：

```sql
SELECT 'abc'::int4;
```
`coerce_type()` 对 unknown Const 调目标类型 input function。
因此错误发生在 parse/analyze 阶段。
这和运行期 cast 不同：

```sql
SELECT text_col::int4 FROM t;
```
后者要等 executor 看到具体行值。

### explicit cast 成功但 implicit lookup 失败
例子形态：

```sql
SELECT value::target_type;
SELECT some_function(value);
```
第一条使用 explicit context。
第二条函数参数匹配通常使用 implicit context。
如果 `pg_cast.castcontext = 'e'`，显式 cast 成功不代表函数调用能匹配。

### assignment 成功但 operator lookup 失败
目标列赋值可以使用 assignment coercion。
operator lookup 通常不能。
所以某个值能插入某列，不代表它能和该列类型的 operator 自动比较。

### domain 约束失败
例子：

```sql
CREATE DOMAIN positive_int AS int CHECK (VALUE > 0);
SELECT (-1)::positive_int;
```
domain coercion 需要 `CoerceToDomain`。
如果输入是常量，错误可能更早暴露。
如果输入来自行值，检查在执行表达式时发生。
关键点：

```text
domain coercion 不只是改 type OID。
```

### polymorphic 无法解析
例子：

```sql
SELECT array_append(NULL, NULL);
```
如果输入不能约束 `anyarray` 和 `anyelement`，`enforce_generic_type_consistency()` 无法给出 concrete type。
正确结果是报错。
不能把它延后到 executor。

### fallback 到 text
并非所有 unknown 都报错。
顶层 unknown 输出或全 unknown common type 场景常 fallback 到 `text`。
例子：

```sql
SELECT 'abc';
SELECT CASE WHEN true THEN 'a' ELSE 'b' END;
```
边界：

```text
有目标类型时服从目标； 重载歧义时不盲目 text 化； Param 无上下文时不能总默认 text。
```

### CoerceViaIO fallback
`find_coercion_pathway()` 在没有 `pg_cast` 行时，保守允许某些 IO 转换。
典型包括：

```text
assignment to string type explicit cast from string type PL/pgSQL assignment fallback
```
这是兼容性折中，不是“任意类型都能隐式经文本转换”。

## 9. 成本、资源与跨模块传播
parse analysis 通常不是长查询的主要成本。
但以下场景会放大它：

```text
大量短 SQL 且不使用 prepared statement； 表达式中函数和操作符很多； schema 中有大量同名重载函数； search_path 很长； 使用 default / variadic / named arguments； ORM 生成巨大 VALUES、CASE、IN 列表； 频繁 DDL 造成 catcache invalidation。
```
主要成本：

```text
syscache / catcache lookup； candidate list 构造； can_coerce_type() 检查； type category / preferred type 查询； find_coercion_pathway() 查 pg_cast； 表达式节点分配； 错误消息构造。
```
成本随这些变量扩张：

```text
表达式节点数； 候选函数数； 候选操作符数； search_path namespace 数； unknown 参数位置数； catalog cache miss 次数。
```
跨模块传播到 planner：

```text
OpExpr.opno -> selectivity function -> mergejoinable / hashjoinable -> equivalence class -> index operator family matching FuncExpr.funcid -> volatility -> strictness -> leakproof -> function cost -> support function -> parallel safety coercion node -> expression simplification -> index expression matching -> domain constraint preservation
```
跨模块传播到 executor：

```text
FuncExpr.funcid -> fmgr lookup and execution OpExpr.opfuncid -> underlying function execution CoerceViaIO -> output/input functions ArrayCoerceExpr -> element-wise coercion CoerceToDomain -> domain constraint check
```
扩展作者需要特别谨慎：

```text
过宽 implicit cast 会污染重载解析； operator 没有合理 selectivity 会影响 planner； 函数 volatility / strictness / parallel 标记错误会影响优化和执行； search_path 下的同名函数可能改变调用结果。
```

## 10. 观测与诊断入口
最直接入口是 `pg_typeof()`：

```sql
SELECT pg_typeof('1');
SELECT pg_typeof('1'::int4);
SELECT pg_typeof('1' + 2);
SELECT pg_typeof(ARRAY[1, 2.5]);
```
它能看到最终类型。
它看不到候选裁剪过程。
看表达式输出：

```sql
EXPLAIN (VERBOSE, COSTS OFF)
SELECT * FROM t WHERE int_col = '42';
```
`VERBOSE` 有时能看到 filter 中的 cast 后形态。
但 EXPLAIN 仍不会展示所有 parser lookup 中间状态。
查函数候选：

```sql
SELECT p.oid, n.nspname, p.proname, p.proargtypes, p.prorettype,
       p.provariadic, p.pronargdefaults
FROM pg_proc p
JOIN pg_namespace n ON n.oid = p.pronamespace
WHERE p.proname = 'substring';
```
查操作符候选：

```sql
SELECT o.oid, n.nspname, o.oprname,
       o.oprleft::regtype, o.oprright::regtype,
       o.oprresult::regtype, o.oprcode
FROM pg_operator o
JOIN pg_namespace n ON n.oid = o.oprnamespace
WHERE o.oprname = '+';
```
查 cast：

```sql
SELECT castsource::regtype, casttarget::regtype,
       castfunc::regprocedure, castcontext, castmethod
FROM pg_cast
WHERE castsource = 'integer'::regtype
   OR casttarget = 'integer'::regtype;
```
注意：

```text
catalog 查询只能列候选； 不能单独证明 parser 会选择哪个候选。
```
还需要结合：

```text
search_path； 实际参数 type OID； unknown 参数位置； coercion context； variadic/default/named arguments； preferred type 和 type category； polymorphic 约束。
```
错误消息也是诊断入口：

```text
operator does not exist -> 没有可用 operator 或 implicit coercion 不足 function does not exist -> 没有可用 function candidate function is not unique -> 候选无法唯一裁剪 cannot cast type X to Y -> 当前 context 下没有 coercion pathway could not determine polymorphic type -> 多态伪类型无法从输入约束 could not determine data type of parameter -> unknown Param 没有足够上下文
```
源码断点推荐：

```text
transformAExprOp make_op oper binary_oper_exact oper_select_candidate ParseFuncOrColumn func_get_detail func_match_argtypes func_select_candidate can_coerce_type coerce_type find_coercion_pathway enforce_generic_type_consistency select_common_type
```
断点时重点看：

```text
actual_arg_types declared_arg_types input_typeids candidate->args rettype funcid operOid pathtype funcId for cast UNKNOWNOID positions
```
能直接观测：

```text
最终返回类型； 错误消息； 部分 EXPLAIN VERBOSE 表达式； catalog 候选对象； prepared statement 参数类型。
```
通常只能推断：

```text
候选列表每轮剩多少； unknown 参数按哪个 type category 被判断； preferred type 规则淘汰了谁； 某个 cast path 是 function、binary 还是 IO。
```
需要断点或临时 instrumentation：

```text
func_select_candidate() 的每轮过滤； oper_select_candidate() 的候选排序； find_coercion_pathway() 查到的 pg_cast tuple； Param unknown 类型何时被写回。
```

## 11. 常见误区
误区一：字符串 literal 默认就是 `text`。
更准确地说，它经常先是 `unknown`，再由上下文或最终输出规则定型。
误区二：显式 cast 能用，隐式 lookup 就能用。
`pg_cast.castcontext` 区分 implicit、assignment 和 explicit。
函数/operator lookup 通常只用 implicit。
误区三：`RelabelType` 可以完全忽略。
它不做物理转换，但改变暴露类型、typmod、collation 或 domain 边界。
误区四：operator 只是函数语法糖。
operator 有自己的 catalog identity，planner 使用 operator OID 做选择率、join 和索引判断。
误区五：多态函数是运行期动态分派。
PostgreSQL 在 parse analysis 就要把 polymorphic 类型约束到 concrete type。
误区六：`pg_proc` 找到同名函数就知道 SQL 会调用谁。
还要看 namespace、参数类型、默认参数、variadic、implicit coercion 和候选选择。
误区七：cast 放在常量侧和列侧都一样。
`int_col = '42'` 和 `int_col::text = '42'` 可能结果一样，但 planner 看到的是不同表达式。
误区八：parse analysis 错误都是语法错误。
很多错误是语义错误：类型无法确定、函数歧义、operator 不存在、多态不一致、unknown 常量输入失败。

## 12. 课堂实验

### 实验 1：观察 unknown literal 定型
运行：

```sql
SELECT pg_typeof('42');
SELECT pg_typeof('42'::int4);
SELECT pg_typeof('42' + 1);
SELECT pg_typeof('42' || 1);
```
再查候选 operator：

```sql
SELECT o.oid, o.oprname, o.oprleft::regtype, o.oprright::regtype,
       o.oprresult::regtype, o.oprcode
FROM pg_operator o
WHERE o.oprname IN ('+', '||')
ORDER BY 2, 3::text, 4::text;
```
要画出的链路：

```text
unknown Const -> operator candidate -> implicit coercion -> selected operator -> final result type
```

### 实验 2：构造函数重载
准备：

```sql
CREATE SCHEMA coercion_lab;
SET search_path = coercion_lab, public, pg_catalog;

CREATE FUNCTION f(x int4) RETURNS text
LANGUAGE sql IMMUTABLE AS $$ SELECT 'int4' $$;

CREATE FUNCTION f(x text) RETURNS text
LANGUAGE sql IMMUTABLE AS $$ SELECT 'text' $$;
```
执行：

```sql
SELECT f(1);
SELECT f('1');
SELECT f('1'::int4);
SELECT f('1'::text);
```
观察 `f('1')` 是解析到某一边还是报歧义。
用 `func_select_candidate()` 断点确认候选裁剪。
清理：

```sql
DROP SCHEMA coercion_lab CASCADE;
```

### 实验 3：比较 implicit、assignment、explicit
查找 cast：

```sql
SELECT castsource::regtype, casttarget::regtype,
       castcontext, castmethod
FROM pg_cast
ORDER BY castsource::regtype::text, casttarget::regtype::text;
```
选择一个 assignment-only 或 explicit-only cast。
比较三类场景：

```text
函数参数匹配； INSERT / UPDATE 目标列赋值； 显式 value::target_type。
```
目标是确认：

```text
同一个 cast row 在不同 CoercionContext 下可见性不同。
```

### 实验 4：跟踪 `int_col = '42'`
准备：

```sql
CREATE TABLE coercion_t(id int4 PRIMARY KEY, v text);
INSERT INTO coercion_t VALUES (42, 'x');
```
执行：

```sql
EXPLAIN (VERBOSE, COSTS OFF)
SELECT * FROM coercion_t WHERE id = '42';
```
断点：

```text
transformAExprOp make_op oper binary_oper_exact coerce_type
```
记录：

```text
左操作数类型； 右操作数初始类型； 选中的 operator OID； 右侧 unknown Const 何时变成 int4； EXPLAIN filter 形态。
```
对比：

```sql
EXPLAIN (VERBOSE, COSTS OFF)
SELECT * FROM coercion_t WHERE id::text = '42';
```
观察 cast placement 对表达式和索引匹配的影响。

### 实验 5：common type
运行：

```sql
SELECT pg_typeof(ARRAY[1, 2, 3]);
SELECT pg_typeof(ARRAY[1, 2.5]);
SELECT pg_typeof(CASE WHEN true THEN 'a' ELSE 'b' END);
SELECT pg_typeof(CASE WHEN true THEN 1 ELSE 2.5 END);
```
断点：

```text
select_common_type coerce_to_common_type
```
目标是区分：

```text
函数/operator 重载解析； 多个表达式合成 common type。
```

## 13. 讨论题
1. 为什么 PostgreSQL 不在 lexer/parser 阶段就把所有字符串 literal 定成 `text`？
2. 为什么函数/operator lookup 通常不能使用 explicit-only cast？
3. `RelabelType` 不改变 Datum 表示，为什么仍要保留在 expression tree 中？
4. 用户定义 implicit cast 可能怎样改变已有 SQL 的解析结果？
5. 为什么 operator OID 对 planner 比 underlying function OID 更重要？
6. `anyelement` 返回类型无法从输入推断时，为什么不能让 executor 再决定？
7. `int_col = '42'` 和 `int_col::text = '42'` 为什么可能得到不同计划？
8. catalog 里有多个同名函数时，还需要哪些状态才能判断 SQL 会调用哪一个？

## 14. 本节小结
本节主链路：

```text
raw expression -> transformExpr() -> actual type OIDs -> function/operator candidates -> implicit coercion feasibility -> candidate selection -> polymorphic consistency -> coercion insertion -> FuncExpr / OpExpr / typed expression
```
核心状态：

```text
UNKNOWNOID 是临时不确定状态； CoercionContext 决定 cast 可见性； pg_cast 给路径，不单独决定使用场景； FuncExpr 保存函数语义 OID； OpExpr 保存 operator OID 和 underlying function； coercion node 保存类型转换、relabel、IO、array 或 domain 语义。
```
ownership：

```text
typed expression 由当前 backend 在 parse analysis 中创建； Query 持有最终表达式； 临时候选和 syscache tuple 不跨阶段持有； ERROR 依赖 memory context 和 cache 引用清理； 长期缓存依赖 catalog invalidation。
```
错误路径：

```text
无候选 -> function/operator does not exist； 候选无法唯一 -> ambiguous； context 不允许 -> cast 或 lookup 失败； unknown Param 无上下文 -> 参数类型无法确定； polymorphic 无法约束 -> 多态类型错误； unknown Const 输入失败 -> analysis 阶段报输入错误。
```
观测边界：

```text
pg_typeof() 看最终类型； EXPLAIN VERBOSE 看部分表达式； pg_proc / pg_operator / pg_cast 看候选事实； 断点才能看完整候选裁剪过程。
```
可迁移规律：

```text
当语言前端允许重载、未定类型和上下文相关语义时， 分析阶段必须把模糊写法收束成唯一可执行对象； 收束规则既要宽松到支持常用 SQL， 又要保守到拒绝歧义和危险隐式转换。
```
这些判断依赖 catalog 内容、扩展对象、search_path、参数类型、版本和具体 SQL 形态。
不要把某条 SQL 的解析结果当成普遍规则。
遇到类型和 lookup 问题时，按固定顺序回到：

```text
actual input type -> candidate set -> CoercionContext -> unknown handling -> polymorphic consistency -> final Expr node
```
