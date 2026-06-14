# PostgreSQL type input/output、varlena、TOAST 与 detoast

## 课程定位
前置知识：已经理解 executor 的 `EState`、`ExprContext`、`TupleTableSlot`、per-tuple memory context，以及 `DestReceiver` 如何把 executor 输出交给客户端或上层调用者。 本节唯一主问题：

```text
type input/output、typmod、varlena、TOAST/detoast 如何影响 executor 中的 Datum 生命周期？
```
核心矛盾：executor 想用统一的 `Datum` 协议低拷贝传递值；但同一个 `Datum` 可能是按值标量、指向 slot/buffer/tuple 的引用、短 header varlena、压缩 varlena、on-disk TOAST pointer、expanded datum，或者刚由 input/output 函数 palloc 出来的临时对象。 学完后应能判断：一个 `Datum` 能否跨过当前 slot、当前 per-tuple context、当前函数调用或当前 output boundary；什么时候必须 copy、materialize、detoast，什么时候应该保持 toasted 或 packed 形态以避免多余成本。 本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置
前面课程已经讲过 slot 的物理/虚拟 tuple 边界，以及 executor memory context 的生命周期。 本节插在 TupleTableSlot / ExprContext 之后，常见执行节点之前。 原因是：执行节点之间传递的不是“完整 C 对象”，而是 `Datum` 加 `isnull`。 如果不知道 `Datum` 的实际表示、类型 I/O 函数、typmod 和 TOAST/detoast 边界，就很难解释这些现象：
- 为什么 `SELECT length(large_text)` 可能比 `SELECT pg_column_size(large_text)` 多很多 CPU 或 IO。
- 为什么 `textout()` 可以接受 toasted value，而某些 C 函数必须显式 detoast。
- 为什么 `varchar(10)` 的长度约束不是 `Datum` header 的一部分。
- 为什么一个看起来只是指针的 `Datum` 不能随便保存到更长生命周期结构。
- 为什么 `PG_GETARG_TEXT_P()` 和 `PG_GETARG_TEXT_PP()` 的成本与对齐语义不同。
04 目录主线是：

```text
PlanState
  -> TupleTableSlot
  -> ExprContext / per-tuple context
  -> expression / function / type boundary
  -> executor node cost and observability
```
本节只回答一个问题：类型 I/O、typmod、varlena、TOAST/detoast 如何共同改变 executor 中一个 `Datum` 的生命周期。 不会展开完整 parser、COPY、logical replication 或 table AM 设计。 这些模块只在解释同一条 `Datum` 生命周期时出现。

## 2. 核心矛盾与一句话运行模型
一句话运行模型：

```text
executor 用 Datum + isnull 作为统一值协议；
类型 metadata 决定 Datum 是 by-value 还是 by-reference；
varlena / TOAST 让 by-reference Datum 可以延迟成短 header、压缩、外部存储或 expanded 形态；
真正消费值的 input/output/function 边界再按需要 detoast、copy、flatten 或格式化。
```
这里的 tension 可以压缩成：

```text
统一 Datum 协议和低拷贝执行
  vs
类型特定表示、外部存储、typmod 约束、memory context ownership 必须精确
```
PostgreSQL 没有让每个 executor 节点理解所有类型的二进制布局。 它把运行时值分成几层：

| 层次 | 运行时问题 | 主要源码边界 |
| --- | --- | --- |
| `Datum` 协议 | 一个 word 里放值还是指针 | `src/include/postgres.h` |
| 类型 metadata | by-value、长度、对齐、I/O 函数 | `pg_type`、`pg_attribute`、`lsyscache.c` |
| varlena header | 可变长值的物理 header 和 packed 形式 | `src/include/c.h`、`src/include/varatt.h` |
| TOAST 存储 | 大值压缩或移出主 tuple | `heaptoast.c`、`toast_helper.c`、`toast_internals.c` |
| detoast 消费 | 按需 fetch、decompress、slice、flatten | `src/backend/access/common/detoast.c` |
| executor 生命周期 | slot、ExprContext、DestReceiver 的 ownership | `execTuples.c`、`execExprInterp.c`、`printtup.c` |
本节的关键不是记住每个宏。 关键是建立一个判断：

```text
Datum 本身不携带 ownership；
Datum 的生存期由创建者、类型 metadata、slot 形态、MemoryContext 和 detoast 路径共同决定。
```
这句话会贯穿后面所有章节。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/postgres.h` | `Datum` 是 `uint64`；它可以是 by-value value，也可以是 by-reference pointer。 |
| 2 | `src/include/catalog/pg_type.h` | `typlen`、`typbyval`、`typalign`、`typstorage`、`typinput`、`typoutput`、`typreceive`、`typsend`、`typmodin`、`typmodout`。 |
| 3 | `src/backend/catalog/pg_type.c` | 创建或更新类型 catalog 行时如何保存 I/O 函数、typmod 函数和依赖。 |
| 4 | `src/include/catalog/pg_attribute.h` | `attlen`、`attbyval`、`attalign`、`attstorage`、`atttypmod` 是列级拷贝或约束边界。 |
| 5 | `src/include/c.h` | `varlena` 基本定义；不要直接读写 `vl_len_`。 |
| 6 | `src/include/varatt.h` | 4-byte、1-byte、compressed、external、indirect、expanded varlena 的判定和大小宏。 |
| 7 | `src/include/fmgr.h` | `PG_GETARG_*`、`PG_DETOAST_DATUM*`、`PG_FREE_IF_COPY` 的调用者协议。 |
| 8 | `src/backend/utils/fmgr/fmgr.c` | `InputFunctionCall()`、`InputFunctionCallSafe()`、`OutputFunctionCall()`、`ReceiveFunctionCall()`、`SendFunctionCall()`。 |
| 9 | `src/backend/parser/parse_type.c` | typmod 从 SQL 语法进入 `typmodin` 的路径；`stringTypeDatum()` 调用输入函数。 |
| 10 | `src/backend/utils/adt/format_type.c` | `format_type_with_typemod()` 与 `typmodout` 的输出路径。 |
| 11 | `src/backend/utils/adt/varlena.c` | `textin()`、`textout()`、`cstring_to_text()`、`text_to_cstring()`、`text_length()`、`pg_column_size()`。 |
| 12 | `src/backend/access/common/detoast.c` | `detoast_attr()`、`detoast_external_attr()`、`detoast_attr_slice()`、`toast_fetch_datum()`。 |
| 13 | `src/backend/access/heap/heaptoast.c` | `heap_toast_insert_or_update()` 如何压缩和 externalize tuple 中的大 varlena。 |
| 14 | `src/include/access/toast_internals.h`、`src/backend/access/common/toast_internals.c` | `toast_compress_datum()`、`toast_save_datum()` 如何形成 TOAST pointer 和 chunk。 |
| 15 | `src/backend/access/common/printtup.c` | executor 输出阶段如何缓存 output/send function，并把 slot 中 `Datum` 格式化。 |
建议阅读时不要从 `varlena.c` 顶部一路读到底。 更有效的顺序是：

```text
Datum 表示
  -> 类型 metadata
  -> slot_getattr 如何得到 Datum
  -> type input/output 如何构造和格式化 Datum
  -> varlena header 如何表示同一个逻辑值
  -> TOAST 写入如何替换存储形态
  -> detoast 读取如何创建新 copy
  -> MemoryContext / slot / output 边界如何收尾
```

## 4. 关键数据结构与状态

### 4.1. `Datum`
`Datum` 在本地源码中定义为：

```c
typedef uint64_t Datum;
```
这个定义只说明容器大小。 它不说明里面是整数、浮点、OID、指针，还是 varlena pointer。 真正解释 `Datum` 的是类型 metadata：

| metadata | 语义 |
| --- | --- |
| `typbyval` / `attbyval` | `Datum` 是否直接携带值。 |
| `typlen` / `attlen` | 固定长度、varlena、cstring 等存储长度规则。 |
| `typalign` / `attalign` | tuple 内对齐规则。 |
| `typstorage` / `attstorage` | 是否允许 TOAST，以及偏好 inline / compressed / external。 |
| `typinput` / `typoutput` | 文本格式和内部 `Datum` 的边界。 |
| `typreceive` / `typsend` | 二进制协议和内部 `Datum` 的边界。 |
| `typmodin` / `typmodout` | SQL 类型修饰符和内部 `int32 typmod` 的边界。 |
| `atttypmod` | 某一列实际携带的 typmod。 |
raw `Datum` 不是语义。 `Datum + isnull + type OID + typmod + MemoryContext + owner` 才是 executor 能安全解释的运行时值。

### 4.2. nullness 不在 `Datum` 里
executor 和 fmgr 都把 nullness 放在旁边。 常见组合是：

```text
Datum value
bool isnull
```
`NullableDatum` 也是这个思路。 这直接影响函数调用协议：
- `PG_ARGISNULL(n)` 先判断 null。
- 非 strict 函数必须自己检查 null。
- strict 函数的调用者可以跳过执行或返回 null。
- `PG_GETARG_*` 不能用于 null argument。
这也是为什么 `Datum` 生命周期讨论总要带上 `isnull`。 一个 null `Datum` 的 bit pattern 没有业务意义。

### 4.3. by-value 与 by-reference
by-value 类型把值放进 `Datum`。 例如 int4、bool、OID 这类值通常可以直接复制一个 word。 by-reference 类型把指针放进 `Datum`。 这包括 text、bytea、array、jsonb、numeric、record 等大量类型。 by-reference 的危险点是：

```text
复制 Datum 只复制指针；
不会复制指针指向的对象；
也不会延长对象所在 MemoryContext、slot、buffer pin 或 expanded object 的生命周期。
```
因此 executor 中这些操作含义不同：

| 操作 | 对 by-value | 对 by-reference |
| --- | --- | --- |
| `Datum` 赋值 | 完整复制值 | 只复制指针 |
| `datumCopy()` | 可退化为 word copy | 可能 palloc 新对象 |
| `ExecMaterializeSlot()` | 多数情况成本低 | 可能复制 tuple 或 values |
| 保存到长生命周期状态 | 通常安全 | 必须确认 copy / context |

### 4.4. `varlena`
`varlena` 是可变长类型的基本物理抽象。 `src/include/c.h` 中的注释提醒：不要直接碰 `vl_len_`。 应使用这些宏：

```text
VARSIZE_ANY()
VARSIZE_ANY_EXHDR()
VARDATA_ANY()
VARSIZE()
VARDATA()
SET_VARSIZE()
```
原因是同一个逻辑 text 可能不是普通 4-byte header。 它可能是：

| 形态 | 判定入口 | 语义 |
| --- | --- | --- |
| 4-byte uncompressed | `!VARATT_IS_EXTENDED(ptr)` | 普通 inline varlena。 |
| 1-byte short | `VARATT_IS_SHORT(ptr)` | 短小值，可能未按 4-byte 对齐。 |
| inline compressed | `VARATT_IS_COMPRESSED(ptr)` | 仍在主 tuple 内，但 payload 压缩。 |
| external on-disk | `VARATT_IS_EXTERNAL_ONDISK(ptr)` | 主 tuple 里是 TOAST pointer，数据在 toast relation。 |
| external indirect | `VARATT_IS_EXTERNAL_INDIRECT(ptr)` | 指向内存中另一个 varlena，由创建者保证寿命。 |
| external expanded | `VARATT_IS_EXTERNAL_EXPANDED(ptr)` | 指向 type-specific expanded object。 |
这个表解释了为什么函数名里会出现 `P`、`PP`、`PCopy`、`PSlice`。 它们不是命名噪音，而是 ownership 和 alignment 协议。

### 4.5. TOAST pointer
传统 on-disk TOAST pointer 存的是 `varatt_external`。 关键字段是：

| 字段 | 语义 |
| --- | --- |
| `va_rawsize` | 逻辑上完全展开后的 varlena 大小，包括 header。 |
| `va_extinfo` | 外部保存大小和压缩方法信息。 |
| `va_valueid` | toast relation 内某个值的 ID。 |
| `va_toastrelid` | 存放 chunks 的 toast relation OID。 |
注意：这个结构存放在 tuple 内部时可能未对齐。 源码经常先 `memcpy` 到局部变量，再读取字段。 所以不要把 `DatumGetPointer()` 后的地址强转成长期可安全访问的 C struct。

### 4.6. typmod
`typmod` 是一个 `int32`，不是 varlena header 的一部分。 它通常来自：

```text
SQL type decoration
  -> TypeName.typmods
  -> typenameTypeMod()
  -> typmodin(cstring[])
  -> int32 typmod
  -> pg_attribute.atttypmod 或表达式节点
```
输出时再走：

```text
format_type_with_typemod()
  -> printTypmod()
  -> typmodout(typmod)
```
这意味着 `varchar(10)` 的 “10” 不在每个 `Datum` 里。 列定义、表达式 coercion、输入函数和约束检查共同维护这个边界。

### 4.7. input/output function
类型 I/O 函数把外部格式和内部 `Datum` 接起来。 `InputFunctionCall()` 的调用协议是 3 个参数：

```text
cstring input
Oid typioparam
int32 typmod
```
`OutputFunctionCall()` 的调用协议是：

```text
Datum value -> cstring
```
二进制路径对应：

```text
ReceiveFunctionCall(StringInfo, typioparam, typmod)
SendFunctionCall(Datum) -> bytea
```
这些函数不拥有长期语义。 它们只是当前调用上下文中创建或消费 `Datum`。 返回对象通常是当前 memory context 中的 palloc 内存。

### 4.8. MemoryContext 与 `fn_mcxt`
fmgr 调用时常见两个 memory context：

| context | 用途 |
| --- | --- |
| 当前 context | 本次调用产生的普通结果、detoast copy、cstring。 |
| `FmgrInfo.fn_mcxt` | 缓存函数私有信息，例如 output function cache、type cache、转换状态。 |
`fn_extra` 常用于缓存类型相关信息。 例如 `pg_column_size()` 第一次调用时会把 `typlen` 存在 `fcinfo->flinfo->fn_extra`。 但普通 detoast copy 不应该偷偷塞进 `fn_extra`。 否则会把 per-tuple 数据错误延长到函数缓存生命周期。

## 5. 主流程源码 walkthrough
本节主流程从“一个 text 值进入 PostgreSQL、被存储、被 executor 读出、被函数消费、被输出”展开。 这不是唯一入口，但它覆盖了本节最关键的生命周期变化。

### 5.1. SQL 类型声明进入 typmod
例子：

```sql
CREATE TABLE t (
  payload varchar(100)
);
```
parser 侧会把 `varchar(100)` 表示成 `TypeName`。 `parse_type.c` 中 `typenameTypeMod()` 处理 typmod：

```text
typenameTypeMod()
  -> 检查 type 是否允许 typmod
  -> 把 typmods 列表转成 cstring array
  -> OidFunctionCall1(typmodin, arrtypmod)
  -> 返回 int32 typmod
```
状态变化是：

```text
SQL 文本中的 "(100)"
  -> 类型私有编码的 int32 typmod
```
这里没有构造任何用户行的 `Datum`。 它只是建立后续输入、coercion、display 和列描述需要使用的 metadata。 对 executor 来说，typmod 不是值的一部分，而是表达式状态或 tuple descriptor 的一部分。

### 5.2. 输入函数构造内部 `Datum`
当一个字符串 literal 或 COPY 文本字段需要变成内部值时，会进入类型 input function。 核心入口是 `fmgr.c`：

```text
InputFunctionCall(flinfo, str, typioparam, typmod)
  -> InitFunctionCallInfoData()
  -> args[0] = cstring
  -> args[1] = typioparam
  -> args[2] = typmod
  -> FunctionCallInvoke()
  -> 检查 null 协议
```
`textin()` 是最简单的 varlena 输入函数之一：

```text
textin()
  -> PG_GETARG_CSTRING(0)
  -> cstring_to_text(input)
  -> palloc(len + VARHDRSZ)
  -> SET_VARSIZE()
  -> memcpy payload
  -> PG_RETURN_TEXT_P(result)
```
这一步得到的 `Datum` 是 by-reference。 `Datum` 里放的是一个指向 palloc varlena 的指针。 这个对象归当前 memory context 管。 如果当前 context 是 per-tuple context，它可能在下一条 tuple 前被 reset。 如果当前 context 是 parse/analyze/planner 的长期 context，它可能活到 plan 或 query state 结束。 要读源码时始终问：

```text
InputFunctionCall() 当前运行在哪个 MemoryContext？
返回的 Datum 会不会被复制到更长生命周期位置？
如果 input function ERROR，已 palloc 的中间对象由哪个 context reset？
```

### 5.3. typmod 约束不等于 header 长度
`typmod` 作为第三个参数传给 input/receive function。 对于很多类型，它会影响解析、截断、精度或合法性检查。 但它不会改变 `Datum` 协议。 `varchar(10)` 和 `text` 仍然都是 by-reference varlena。 区别在于：
- 列或表达式携带的 typmod 不同。
- 输入或 coercion 时会调用类型特定逻辑检查。
- output display 可以通过 typmodout 还原类型修饰符。
- 存储中的 varlena payload 不会为每个值重复保存这个 `10`。
这解释了一个常见误判：

```text
看到一个 text/varchar Datum 指针，不能从 varlena header 推断列声明长度。
```
要结合 `TupleDescAttr(desc, attno)->atttypmod`。

### 5.4. executor 表达式产生 `Datum`
执行阶段，表达式解释器或 JIT 会把函数、操作符、常量、字段引用求成：

```text
Datum value
bool isnull
```
当表达式调用 C 函数时，会构造 `FunctionCallInfo`。 函数内部用 `PG_GETARG_*` 读取参数。 对于 text 参数，常见宏是：

```text
PG_GETARG_TEXT_PP(n)
PG_GETARG_TEXT_P(n)
PG_GETARG_TEXT_P_COPY(n)
PG_GETARG_TEXT_P_SLICE(n, off, len)
```
它们的生命周期语义不同：

| 宏 | 语义 |
| --- | --- |
| `PG_GETARG_TEXT_PP` | 允许 packed short header；会展开 external/compressed；结果可能未对齐。 |
| `PG_GETARG_TEXT_P` | 返回非 extended 4-byte header；需要对齐访问时使用。 |
| `PG_GETARG_TEXT_P_COPY` | 一定返回可写 palloc copy。 |
| `PG_GETARG_TEXT_P_SLICE` | 只取 slice，可能避免读取完整外部值。 |
函数如果只是 `memcmp`、`memcpy`、按字节扫描 text payload，通常应优先考虑 `PP`。 函数如果要按结构体字段读 int16/int32，必须考虑对齐，不能盲目用 packed pointer。

### 5.5. tuple 形成时仍可能只是 inline varlena
执行 INSERT / UPDATE 时，targetlist 求值产生 values/isnull。 随后 heap tuple 形成路径会根据 tuple descriptor 把这些值写成物理 tuple。 对于 varlena，写入时会看：

```text
attlen == -1
attstorage
value size
tuple size
relation toast target
```
短值可能被转成 short-header。 普通值可能保留 4-byte header。 此时 `Datum` 指向的 palloc 对象可能被复制进新 tuple。 复制进 tuple 后，原来的 palloc 对象不再是存储层长期值。 长期值变成 heap tuple 中的 bytes。

### 5.6. TOAST 写入路径改变存储形态
当 tuple 过大，heap AM 会进入：

```text
heap_toast_insert_or_update()
  -> heap_deform_tuple(newtup)
  -> toast_tuple_init()
  -> 尝试压缩 EXTENDED 属性
  -> 仍过大则 externalize EXTENDED / EXTERNAL 属性
  -> 必要时压缩 MAIN 属性
  -> 最后才 externalize MAIN 属性
  -> 构造新的 heap tuple
```
这里的重要状态变化是：

```text
executor 产生的 inline varlena Datum
  -> heap tuple 中的 inline varlena bytes
  -> 可能被替换为 compressed inline datum
  -> 可能被替换为 on-disk TOAST pointer
```
TOAST 不是 executor 每次访问都主动调用的“压缩函数”。 它是存储 tuple 时由 heap/table AM 路径触发的形态转换。 executor 上层看到的仍然只是 slot 里的 `Datum`。 这个 `Datum` 可能指向 tuple 中的 TOAST pointer。

### 5.7. toast chunk 的持久化
`toast_save_datum()` 负责把一个 varlena 值写入 toast relation。 核心状态是 `varatt_external`：

```text
va_rawsize
va_extinfo
va_valueid
va_toastrelid
```
数据被切成 chunks 写入 toast table。 主 heap tuple 里只留下 external datum。 这一步还引入存储层正确性：
- toast relation 是普通 relation，有自己的 heap/index。
- toast chunks 随主 tuple 的 insert/update/delete 被维护。
- `UPDATE` 可能复用或删除旧 toast value。
- crash safety 由普通 heap/index/WAL 路径承担。
本节不展开 WAL 细节。 但要记住：TOAST pointer 不是进程内指针。 它是一个可持久化的外部值引用。

### 5.8. scan slot 读到的可能是 TOAST pointer
SELECT 阶段，scan node 从 table AM 得到 tuple，放入 `TupleTableSlot`。 当上层调用 `slot_getattr()` 时，heap deform 路径会返回对应属性的 `Datum`。 对于 by-reference varlena，这个 `Datum` 通常指向 tuple 内部的 bytes。 如果该属性 toasted，指向的就是 tuple 内部的 external datum bytes。 此时还没有必然 detoast。 这就是 lazy 的关键。 只要后续表达式不真正需要 payload，PostgreSQL 可以把 TOAST pointer 继续往上传。 `pg_column_size()`、某些比较、某些输出路径会根据需要决定是否读取外部数据。

### 5.9. detoast 由消费方触发
消费方如果调用 `PG_GETARG_TEXT_PP()`，会走：

```text
PG_GETARG_TEXT_PP(n)
  -> PG_GETARG_DATUM(n)
  -> PG_DETOAST_DATUM_PACKED()
  -> pg_detoast_datum_packed()
  -> detoast_attr 或相关路径
```
更底层的核心路径在 `detoast.c`：

```text
detoast_attr()
  -> on-disk external: toast_fetch_datum()
  -> inline compressed: toast_decompress_datum()
  -> short header: palloc 4-byte header copy
  -> indirect: dereference then recurse / copy
  -> expanded: flatten
```
`detoast_external_attr()` 稍弱一些：

```text
external -> fetch or flatten
结果不依赖外部存储
但可能仍是 compressed 或 short header
```
`detoast_attr_slice()` 则试图只取一段。 它是 substring 等函数减少读取大 TOAST value 的关键。

### 5.10. detoast copy 的 owner 是当前 context
`pg_detoast_datum()` 的协议非常重要：

```text
如果输入不需要 detoast，可能返回原指针；
如果需要 fetch/decompress/flatten，会在当前 MemoryContext palloc 新对象。
```
所以调用者不能只看返回类型是 `text *` 就假设它可写或长期有效。 需要问两个问题：

```text
返回值是否等于原始 PG_GETARG_POINTER(n)？
返回值所在 MemoryContext 能活多久？
```
`PG_FREE_IF_COPY(ptr, n)` 正是为这种情况准备的。 它的协议是：

```text
如果 ptr != 原始参数指针，认为它是 palloc 的 detoast copy，可以 pfree。
```
多数普通 SQL 函数可以依赖 per-tuple context reset。 但 index support function、排序比较、哈希比较这类 hot path 支持函数不能随意泄漏 detoast copy。 它们经常需要显式 `PG_FREE_IF_COPY()`。

### 5.11. `text_to_cstring()` 是 output 边界的缩影
`varlena.c` 中 `textout()` 很短：

```text
textout()
  -> PG_GETARG_DATUM(0)
  -> TextDatumGetCString()
  -> text_to_cstring()
```
`text_to_cstring()` 做了几件事：

```text
pg_detoast_datum_packed()
VARSIZE_ANY_EXHDR()
palloc(len + 1)
memcpy(VARDATA_ANY())
补 '\0'
如果 detoast 产生 copy，则 pfree copy
```
这解释了 output 成本：
- client 需要文本格式时，最终必须构造 null-terminated cstring。
- 即使内部 text 没有 `\0` 终止，输出也要产生一个新的 C string。
- toasted/compressed value 会在这里按需 detoast。
- output cstring 属于当前 output memory context。

### 5.12. DestReceiver 输出路径
常规 SELECT 给客户端输出时，`printtup.c` 会准备每列的 output/send function。 输出每个 slot 时大致是：

```text
slot_getattr()
  -> Datum attr + isnull
  -> 如果 null 输出 null
  -> 文本格式 OutputFunctionCall()
  -> 二进制格式 SendFunctionCall()
  -> 写入 frontend/backend protocol buffer
```
这里的状态边界是：

```text
executor 内部 Datum 生命周期
  -> type output/send function 生成外部表示
  -> protocol buffer 持有要发给客户端的 bytes
```
客户端永远不会看到 TOAST pointer。 TOAST pointer 是 server 内部存储表示。 外部协议看到的是类型输出函数定义的文本或二进制格式。

### 5.13. 一个完整时间线
把前面流程压缩成一个对象故事：

```text
01. SQL 声明 varchar(100)，typmodin 把 100 编成 int32 typmod。
02. 输入字符串进入 typinput，text/varchar input palloc 一个 varlena。
03. executor targetlist 持有 Datum + isnull，Datum 对 varlena 是指针。
04. heap tuple formation 复制或引用这些值并形成 tuple bytes。
05. heap_toast_insert_or_update 可能把大 varlena 压缩或 externalize。
06. 主 heap tuple 最终可能只保存一个 TOAST pointer。
07. SELECT scan 把 heap tuple 放入 slot。
08. slot_getattr 返回指向 tuple 内 varlena 或 TOAST pointer 的 Datum。
09. expression function 用 PG_GETARG_TEXT_PP/P/COPY/Slice 决定是否 detoast。
10. detoast 如果需要，会在当前 MemoryContext palloc copy。
11. output function 把内部 Datum 转成 cstring 或 bytea。
12. per-tuple context reset / slot clear / ExecutorEnd 回收短生命周期对象。
```
这条时间线是后面 correctness、异常和诊断的共同骨架。

## 6. 生命周期 / ownership / cleanup

### 6.1. 谁创建 `Datum`
创建者可能是：

| 创建者 | 例子 | 生命周期起点 |
| --- | --- | --- |
| type input function | `textin()`、`numeric_in()` | 当前 memory context 中 palloc。 |
| expression function | `textcat()`、`substring()` | 当前 per-tuple 或函数指定 context。 |
| slot deform | `slot_getattr()` / heap deform | 指向 slot 当前 tuple 内容。 |
| heap tuple formation | `heap_form_tuple()` | tuple bytes 内部。 |
| TOAST write path | `toast_save_datum()` | toast relation chunks + main tuple pointer。 |
| detoast path | `detoast_attr()` | 当前 context 中的新 palloc varlena。 |
| expanded object API | array/jsonb 等类型 | type-specific expanded context。 |
同样都是 `Datum`，owner 完全不同。 这也是本节最重要的边界感。

### 6.2. 谁持有 by-reference `Datum`
常见 owner 有四类：

| owner | 持有内容 | 失效点 |
| --- | --- | --- |
| `TupleTableSlot` | 当前 tuple 或 values array | `ExecClearTuple()`、rescan、slot overwrite、ExecutorEnd。 |
| `ExprContext` per-tuple context | 函数临时结果、detoast copy、output cstring | `ResetExprContext()`。 |
| per-query / node context | 节点状态、缓存、必要的长期表达式结果 | `ExecutorEnd()` 或 node cleanup。 |
| storage relation | heap tuple bytes、toast chunks | MVCC、VACUUM、relation cleanup 规则。 |
`Datum` 指针本身不告诉你 owner 是谁。 必须从创建路径和当前 context 判断。

### 6.3. slot 内容与 detoast copy 不是同一个对象
scan slot 中的 `Datum` 可能指向 buffer-backed heap tuple。 这个指针依赖：
- slot 当前内容没有被 clear。
- 底层 buffer pin 或 tuple copy 仍有效。
- tuple descriptor 仍匹配。
detoast copy 则通常依赖：
- 当前 memory context 没有 reset。
- 调用者没有 `pfree`。
- copy 没有被错误地保存到更短 context 外。
所以“已经 detoast”不等于“生命周期更长”。 只是存储形态变成了当前 context 中的普通 varlena。

### 6.4. `PG_FREE_IF_COPY`
`PG_FREE_IF_COPY(ptr, n)` 的边界是函数调用参数。 典型模式：

```c
text *arg = PG_GETARG_TEXT_PP(0);
/* read-only work */
PG_FREE_IF_COPY(arg, 0);
```
普通函数很多时候省略它，因为 per-tuple context 会 reset。 但以下路径要更谨慎：
- btree/hash comparison support function。
- sort support comparator。
- index AM support function。
- tight loop 中手工调用同一个函数很多次。
- 函数在长生命周期 context 中运行。
如果不清楚上下文，优先确认 `fcinfo->flinfo->fn_mcxt` 和当前 `CurrentMemoryContext`。

### 6.5. ERROR / abort 时谁兜底
大多数 input/detoast/output 产生的内存由 MemoryContext 兜底。 如果函数在 per-tuple context 中 ERROR，中间 palloc 对象会随错误清理和 context reset 被回收。 但不是所有资源都是普通内存：

| 资源 | cleanup 机制 |
| --- | --- |
| toast fetch 打开的 relation | 函数正常 close；ERROR 由 resource owner / error cleanup 兜底。 |
| buffer pin | slot/table AM/ResourceOwner cleanup。 |
| output protocol buffer | 上层 destination / portal / error path cleanup。 |
| toast table writes | 事务 abort 和 WAL/MVCC 规则处理。 |
| expanded object | 所在 memory context 或 expanded datum API 约束。 |
不要把 MemoryContext 当成唯一 owner。 本节和前面 ResourceOwner 课程的连接点就在这里。

### 6.6. 长期缓存只缓存 metadata
可以长期缓存的是：
- `FmgrInfo`。
- `typoutput` / `typsend` function lookup。
- `typlen`。
- typcache / syscache 结果的拷贝或受控引用。
- sort support state。
不应该长期缓存的是：
- 指向 slot 内 tuple 的 varlena pointer。
- per-tuple context 中的 detoast copy。
- output function 返回的 cstring。
- 未确认 owner 的 indirect TOAST pointer 目标。
一个经验规则：

```text
缓存类型知识可以；
缓存某条 tuple 的 by-reference Datum 必须证明 ownership。
```

## 7. 正确性机制层次

### 7.1. 类型 metadata 正确性
`pg_type` 定义类型级属性。 `pg_attribute` 把很多类型属性复制成列级属性。 这有两个原因：
- tuple deform 和 formation 需要非常热的 `attlen`、`attbyval`、`attalign`。
- 列级 `attstorage`、`atttypmod` 可以和类型默认值不同。
正确性不来自单个字段。 它来自组合：

```text
type OID
  + typbyval / typlen / typalign
  + attribute-level attstorage / atttypmod
  + executor expression type metadata
```
如果这些 metadata 错，`Datum` 解释会直接变成内存安全问题。

### 7.2. function strictness 与 null 协议
`InputFunctionCall()` 对 null 有特殊处理。 如果 `str == NULL` 且函数 strict，直接返回 null result。 否则仍会调用 input function，并在返回后检查：
- null input 应该返回 null。
- non-null input 不应该返回 null。
这不是性能细节。 它是类型 I/O 函数和 fmgr 协议的一部分。 普通 SQL 函数也类似：strictness 决定是否跳过调用。 非 strict 函数必须自己用 `PG_ARGISNULL()`。

### 7.3. varlena alignment 正确性
packed varlena 可能未按 4-byte 对齐。 因此 `PG_GETARG_TEXT_PP()` 适合 byte-wise 读取。 但如果类型内部结构需要直接读宽字段，就要使用非 packed detoast 路径或 `memcpy`。 这就是 `fmgr.h` 注释强调：

```text
VARDATA_ANY + memcpy 是 alignment-oblivious；
直接取 int16/int32 字段不是。
```
正确性边界不是“有没有 detoast”。 而是“返回指针是否满足后续访问方式需要的对齐和形态”。

### 7.4. TOAST fetch 的一致性
on-disk TOAST value 由主 tuple 指针引用。 读取时 `toast_fetch_datum()` 会打开 toast relation，并通过 table AM 取 chunks。 它依赖普通 relation、snapshot、lock、buffer 和 index 机制。 本节不用展开 MVCC。 但要知道：
- TOAST chunk 不是独立用户可见行。
- 主 tuple 的可见性决定用户是否应该看到这个 value。
- toast fetch 需要访问 toast relation 和 toast index。
- abort / crash safety 由 heap/index/WAL 机制承担。

### 7.5. typmod 正确性
typmod 的正确性分两段：

```text
parse/DDL 阶段：typmodin 验证和编码类型修饰符。
execution/input/coercion 阶段：类型函数按 typmod 检查或转换具体值。
```
因此不能只在 output display 修饰类型名。 也不能只在 varlena payload 里找长度。 `varchar(10)` 的合法性是类型函数、coercion 和 tuple descriptor metadata 共同维护的。

### 7.6. MemoryContext 正确性
detoast 创建的 copy 用 `palloc`。 `palloc` 进入当前 memory context。 因此 correctness 依赖调用者在进入函数前设置了合适的 current context。 executor 的 per-tuple reset 是这里的主要防泄漏机制。 但是对于 comparator 或 support function，单纯等待 per-tuple reset 可能成本太高。 需要手工释放或避免 full detoast。

## 8. 错误路径 / 异常路径 / fallback

### 8.1. input function soft error
`InputFunctionCallSafe()` 支持 soft error。 它接收 `ErrorSaveContext`，让 input function 把某些错误保存下来并返回 false。 流程是：

```text
InputFunctionCallSafe()
  -> FunctionCallInvoke()
  -> SOFT_ERROR_OCCURRED(escontext)
  -> false
```
这让调用者可以在某些场景下报告更好的错误位置或继续处理。 如果不是 `ErrorSaveContext`，行为仍然是 `ereport(ERROR)`。 这条路径说明：类型输入不是简单转换函数，它也参与错误报告边界。

### 8.2. input/output null 协议违反
`InputFunctionCall()` 会检查输入函数的 null 行为。 非 null 输入返回 null，会报错。 null 输入返回非 null，也会报错。 `OutputFunctionCall()` 明确不应调用在 null datums 上。 输出层必须先看 `isnull`。 这解释了为什么 executor 输出总是先处理 null，再调用 output/send function。

### 8.3. invalid compression method
`toast_decompress_datum()` 根据压缩 method 分派到 pglz 或 lz4。 如果 method id 无效，会 `elog(ERROR)`。 这是存储内容损坏、版本不兼容或 bug 的异常路径。 调用者看到的是查询 ERROR。 中间分配的内存由 context cleanup 回收。

### 8.4. slice fallback
`detoast_attr_slice()` 尝试只取需要的部分。 但 compressed external datum 有限制：slice 通常只能作为前缀处理，必要时会 fetch 更多或完整解压。 流程可以理解为：

```text
需要 slice
  -> 如果 external plain，尽量只 fetch slice
  -> 如果 compressed 且不能直接得到目标 slice，先 fetch/decompress 前缀或完整值
  -> 返回新的 varlena
```
所以 `substring(large_text, 1, 10)` 可能很便宜。 但 `substring(large_text, 1000000, 10)` 是否便宜取决于压缩、slice offset、TOAST chunk 和实现路径。

### 8.5. 没有 toast table
`heap_toast_insert_or_update()` 只有在 relation 有 toast table 时才能 externalize。 如果没有 toast table，压缩可能仍然尝试，但 external storage 不可用。 最终 tuple 仍必须满足 heap tuple size 限制。 超出限制会在更高层报错。 这不是 detoast 的问题，而是存储形态没有可用 fallback。

### 8.6. indirect pointer 的寿命风险
`varatt_indirect` 表示指向内存中另一个 varlena。 源码注释明确：创建者负责保证目标存储活得足够久。 如果错误地把 indirect pointer 存入长期结构，而目标只活在短 context 中，就会变成悬垂指针。 `detoast_attr()` 遇到 indirect 会递归并在必要时 copy。 这是一条安全边界：

```text
不确定 indirect pointer owner 时，detoast/copy 后再跨边界保存。
```

### 8.7. output 期间 ERROR
输出函数可能因为编码、类型格式化、内存分配或 detoast fetch 失败而 ERROR。 此时 executor 已经产生了 tuple，但不代表结果已经安全送达客户端。 上层 portal、destination receiver、protocol 状态会进入错误收尾。 这解释了为什么 output function 也是 executor 可观测边界的一部分。 慢 SQL 可能慢在“执行完成后的输出格式化”，而不在 scan/join 本身。

## 9. 成本、资源与跨模块传播

### 9.1. 成本模型
本节成本主要来自四类变量：

| 成本 | 放大变量 | 典型症状 |
| --- | --- | --- |
| detoast CPU | 行数、调用次数、压缩算法、重复 detoast | CPU profile 中 pglz/lz4 或 varlena 函数突出。 |
| toast IO | 外部值数量、chunk 数、缓存命中率 | `EXPLAIN BUFFERS` 或 toast relation IO 增加。 |
| memory churn | detoast copy 大小、per-tuple reset 频率 | backend memory context 峰值和 allocator 成本上升。 |
| output formatting | 输出列数、输出行数、文本格式成本 | executor 节点时间不高，但客户端收数慢。 |
不要把 “TOAST” 等同于 “慢”。 TOAST 也可能节省主 heap IO。 慢通常出现在：
- 每行多次 detoast 同一大值。
- 比较函数在 sort/hash/index hot path 中重复 detoast。
- 输出大字段到客户端。
- 压缩值需要完整解压才能做小操作。
- toast table 随机访问缓存命中差。

### 9.2. `PG_GETARG_TEXT_P` vs `PG_GETARG_TEXT_PP`
`PG_GETARG_TEXT_P()` 可能把 short-header 转成 4-byte header copy。 `PG_GETARG_TEXT_PP()` 可以保留 packed short-header。 因此在只按 bytes 读取 text 的函数中，`PP` 可避免不必要复制。 但 `PP` 返回可能未对齐。 所以成本和正确性要一起判断：

```text
能用 VARDATA_ANY + VARSIZE_ANY_EXHDR 安全处理，就优先 packed。
需要宽字段对齐访问，就不要冒险。
```

### 9.3. `toast_raw_datum_size()` 与 full detoast
`varlena.c` 中 `textoctetlen()` 可以用 `toast_raw_datum_size()` 得到逻辑长度。 它不必 fetch 全部 payload。 `pg_column_size()` 对 varlena 使用 `toast_datum_size()` 得到物理大小。 这说明观测函数本身也有不同成本模型：

| 函数 | 可能行为 |
| --- | --- |
| `pg_column_size(value)` | 看物理存储大小，通常不需要完整 detoast。 |
| `octet_length(text)` | 可用 raw size 规避 full detoast。 |
| `length(text)` | 多字节编码下可能需要扫描 payload。 |
| `textout(text)` | 输出给客户端时最终需要构造 cstring。 |

### 9.4. 与 parser/analyzer 的边界
typmod 进入系统发生在 parser/analyzer。 但具体值检查可能在输入、coercion 或执行时发生。 跨模块链路是：

```text
parse_type.c
  -> typmodin
  -> expression / target list carries typmod
  -> executor computes Datum
  -> type coercion/input checks typmod
  -> format_type.c / typmodout displays typmod
```
这解释了为什么某些错误在 parse/analyze 阶段报，某些错误在执行阶段报。

### 9.5. 与 storage/table AM 的边界
executor 不直接管理 toast chunks。 写入路径在 heap/table AM 中决定是否 toast。 读取路径在 detoast 时通过 table AM fetch toast chunks。 边界是：

```text
executor: 传递 Datum、调用函数、管理 context
heap/table AM: 形成 tuple、压缩、外部化、维护 toast relation
detoast.c: 把 external/compressed/expanded 还原成调用者要的内存形态
```
未来如果 table AM 有自己的压缩或外部存储策略，也要保持 executor 的 `Datum` 协议。

### 9.6. 与 output/protocol 的边界
`DestReceiver` 决定 output function 是否会成为瓶颈。 例如：
- 普通客户端文本格式使用 `typoutput`。
- 二进制格式使用 `typsend`。
- COPY TO 有自己的 output path，但仍依赖 type output/send。
- logical replication 也可能按文本或二进制调用 type I/O。
同一个 executor plan，换 destination 可能改变最终 CPU 和内存成本。

### 9.7. 与 planner/cost 的边界
planner 通常不会精确建模每次 detoast 的 CPU 或 toast table IO。 它知道列宽统计、函数 cost、表达式 cost，但不知道每一行是否会在某个函数中 full detoast。 因此这类问题常表现为：

```text
estimated rows 接近实际
plan shape 看起来合理
但 CPU 或 BUFFERS 明显高
```
这时诊断要从 executor 表达式函数和 varlena 消费方式入手，而不是只改 join order。

## 10. 观测与诊断入口

### 10.1. 能直接看到的状态
可以直接看列类型、typmod、storage：

```sql
SELECT
  a.attname,
  format_type(a.atttypid, a.atttypmod) AS display_type,
  a.attlen,
  a.attbyval,
  a.attstorage,
  a.attcompression
FROM pg_attribute a
WHERE a.attrelid = 'public.t'::regclass
  AND a.attnum > 0
ORDER BY a.attnum;
```
可以看 relation 的 toast table：

```sql
SELECT
  c.oid::regclass AS table_name,
  c.reltoastrelid::regclass AS toast_table
FROM pg_class c
WHERE c.oid = 'public.t'::regclass;
```
可以看值大小：

```sql
SELECT
  pg_column_size(payload) AS physical_size,
  octet_length(payload) AS logical_octets
FROM public.t
LIMIT 10;
```
如果本版本提供 `pg_column_toast_chunk_id()`，可以辅助判断某个值是否 on-disk toasted：

```sql
SELECT pg_column_toast_chunk_id(payload)
FROM public.t
WHERE payload IS NOT NULL
LIMIT 10;
```

### 10.2. 能间接看到的状态
`EXPLAIN (ANALYZE, BUFFERS)` 可以看到 buffer 访问。 如果查询只访问主表字段 metadata，toast relation 可能没有额外 buffer。 如果表达式触发 full detoast，可能出现更多 buffer hit/read。 但 EXPLAIN 不会标一行：

```text
detoast calls: N
```
这类状态只能通过源码断点、perf、trace 或自定义计数推断。

### 10.3. profiler 入口
CPU 侧常见符号：

```text
detoast_attr
detoast_attr_slice
toast_fetch_datum
toast_decompress_datum
pglz_decompress_datum
lz4_decompress_datum
text_to_cstring
OutputFunctionCall
slot_getattr
```
如果 profile 里这些符号突出，下一步不要立刻怀疑 planner。 先确认：
- 哪个表达式函数触发 detoast。
- 是否每行重复 detoast 同一列。
- 是否 output formatting 在消耗时间。
- 是否 toasted value 太大且缓存命中差。

### 10.4. gdb 断点入口
源码跟读时可以设这些断点：

```text
InputFunctionCall
OutputFunctionCall
textin
textout
text_to_cstring
detoast_attr
detoast_attr_slice
toast_fetch_datum
heap_toast_insert_or_update
toast_save_datum
```
断点停住后建议打印：

```text
CurrentMemoryContext->name
fcinfo->flinfo->fn_oid
fcinfo->args[n].isnull
DatumGetPointer(fcinfo->args[n].value)
VARSIZE_ANY(DatumGetPointer(value))
VARATT_IS_EXTERNAL(...)
VARATT_IS_COMPRESSED(...)
VARATT_IS_SHORT(...)
```
不要只打印 `Datum` 的整数值。 那通常不能解释 ownership。

### 10.5. 观测盲区
本节最大的盲区是：

```text
PostgreSQL 没有内置、通用、低成本的 per-query detoast 计数器。
```
因此你能直接看到：
- 类型 metadata。
- toast table 是否存在。
- 某个值的物理大小。
- query 的 buffer 和 timing。
- profile 栈上的 detoast/decompress 函数。
你看不到或只能推断：
- 每个表达式 detoast 了几次。
- 某个 detoast copy 是否被重复创建。
- 每个 detoast copy 的 memory context 峰值。
- output function 花费是否完全来自 detoast 还是格式化。
诊断时要把这些不确定性写清楚。

## 11. 常见误区
误区一：`Datum` 就是值本身。实际：对 by-reference 类型，`Datum` 是指针；复制 `Datum` 不复制对象，也不延长对象生命周期。
误区二：`text *` 一定是普通 4-byte header text。实际：裸指针可能是 short、compressed、external 或 expanded；很多函数能处理它，是因为内部先 detoast 或使用 packed 宏。
误区三：detoast 一定发生在 scan 阶段。实际：scan/deform 常常只返回指向 tuple 内 bytes 的 `Datum`，detoast 由消费方触发。
误区四：`varchar(10)` 的 10 保存在每个值的 varlena header 里。实际：typmod 在 tuple descriptor、表达式状态或 catalog metadata 中，varlena header 只描述物理长度/形态。
误区五：`PG_GETARG_TEXT_P()` 永远更安全。实际：它可能制造不必要 copy；如果只做 byte-wise 访问，`PG_GETARG_TEXT_PP()` 常常更合适，但 packed pointer 不能用于需要对齐的结构访问。
误区六：`pg_column_size()` 越小，查询越快。实际：压缩或 external storage 能减小物理大小，但读取时可能引入 decompress 或 toast fetch 成本，需要结合 workload 判断。
误区七：只要值已经 detoast，就可以长期保存指针。实际：detoast copy 在当前 memory context 中，context reset 后指针仍会失效。
误区八：output 阶段不属于执行器性能问题。实际：`DestReceiver` 调用 type output/send function，可能触发 detoast、格式化和内存分配；慢查询的尾部成本可能在这里。

## 12. 课堂实验

### 实验 1：观察 typmod 与 storage metadata
准备表：

```sql
DROP TABLE IF EXISTS public.toast_demo;

CREATE TABLE public.toast_demo (
  id int generated always as identity,
  short_text varchar(20),
  payload text
);

ALTER TABLE public.toast_demo
  ALTER COLUMN payload SET STORAGE EXTENDED;
```
查看 metadata：

```sql
SELECT
  a.attname,
  format_type(a.atttypid, a.atttypmod) AS display_type,
  a.attlen,
  a.attbyval,
  a.attstorage,
  a.attcompression
FROM pg_attribute a
WHERE a.attrelid = 'public.toast_demo'::regclass
  AND a.attnum > 0
ORDER BY a.attnum;
```
观察点：
- `varchar(20)` 的 typmod 体现在 `format_type()` 输出。
- `text` / `varchar` 都是 varlena，`attlen` 通常是 `-1`。
- storage policy 是列级 metadata，不在每个值的 `Datum` header 中。

### 实验 2：构造可 TOAST 的值并比较大小
插入数据：

```sql
INSERT INTO public.toast_demo(short_text, payload)
SELECT
  'row-' || g,
  string_agg(md5((g * 100000 + x)::text), '')
FROM generate_series(1, 10) AS g
CROSS JOIN generate_series(1, 4000) AS x
GROUP BY g;
```
查看大小：

```sql
SELECT
  id,
  pg_column_size(payload) AS physical_size,
  octet_length(payload) AS logical_octets
FROM public.toast_demo
ORDER BY id
LIMIT 5;
```
再看 toast table：

```sql
SELECT c.reltoastrelid::regclass
FROM pg_class c
WHERE c.oid = 'public.toast_demo'::regclass;
```
观察点：
- `pg_column_size()` 和 `octet_length()` 回答的问题不同。
- `repeat('x', n)` 很容易被压缩，未必能代表 external IO。
- `md5()` 拼接更接近不可压缩大 text，但仍受版本、压缩算法、toast target 影响。

### 实验 3：比较不同行为的表达式
执行：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT sum(pg_column_size(payload))
FROM public.toast_demo;

EXPLAIN (ANALYZE, BUFFERS)
SELECT sum(octet_length(payload))
FROM public.toast_demo;

EXPLAIN (ANALYZE, BUFFERS)
SELECT sum(length(payload))
FROM public.toast_demo;

EXPLAIN (ANALYZE, BUFFERS)
SELECT string_agg(payload, '')
FROM public.toast_demo;
```
观察点：
- `pg_column_size()` 偏物理大小。
- `octet_length()` 可利用 raw size。
- `length()` 在多字节编码下可能需要扫描 payload。
- `string_agg()` 会产生新的大 varlena，关注内存和输出成本。
不要把这些实验结果绝对化。 它们依赖编码、压缩、数据分布、缓存状态和 PostgreSQL 版本。

### 实验 4：gdb 跟 detoast 生命周期
在调试构建上设断点：

```text
break detoast_attr
break detoast_attr_slice
break toast_fetch_datum
break text_to_cstring
break OutputFunctionCall
```
运行：

```sql
SELECT length(payload)
FROM public.toast_demo
WHERE id = 1;

SELECT payload
FROM public.toast_demo
WHERE id = 1;
```
断点中记录：

```text
当前函数
CurrentMemoryContext->name
传入 Datum 指针
VARATT_IS_EXTERNAL / COMPRESSED / SHORT
返回指针是否等于输入指针
```
目标不是记住调用栈。 目标是画出：

```text
slot Datum
  -> detoast copy
  -> output cstring
  -> context reset / pfree
```

### 实验 5：源码小改动计数 detoast
只在本地实验分支中做，不要作为产品代码提交。 在 `detoast.c` 的 `detoast_attr()` 入口加一个 backend-local 计数器或 `elog(DEBUG1)`。 分别执行：

```sql
SELECT pg_column_size(payload) FROM public.toast_demo;
SELECT length(payload) FROM public.toast_demo;
SELECT substring(payload from 1 for 20) FROM public.toast_demo;
SELECT payload FROM public.toast_demo;
```
观察：
- 哪些 SQL 触发 full detoast。
- 哪些触发 slice。
- 哪些直到 output 才 detoast。
- 一行是否可能触发多次 detoast。
这个实验要回到源码解释，而不是停在 SQL 差异。

## 13. 讨论题
1. 为什么 `Datum` 不携带类型 OID、typmod 和 ownership？如果携带，会给 hot path 带来什么成本？
2. 一个 executor 节点想缓存某个 text 值到 node private context，需要检查哪些条件？
3. `PG_GETARG_TEXT_PP()` 为什么可以更快？它牺牲或要求调用者承担什么边界？
4. 为什么 `varchar(10)` 不能只靠 varlena header 判断合法性？
5. `pg_column_size()`、`octet_length()`、`length()` 观察到的分别是哪一层事实？
6. 如果 profile 中 `toast_fetch_datum()` 很热，下一步应该检查 SQL pattern、schema storage、数据分布还是 planner？为什么？
7. output function 触发 detoast 时，为什么 EXPLAIN 节点级时间可能不容易归因？
8. 如果一个扩展函数返回指向 static buffer 的 varlena pointer，会破坏哪些 executor 假设？

## 14. 本节小结
本节主链路是：

```text
type declaration
  -> typmodin
  -> type input / receive
  -> Datum + isnull
  -> slot / tuple / memory context
  -> heap TOAST write path
  -> slot_getattr
  -> function consumption and detoast
  -> type output / send
  -> cleanup
```
核心状态是 `Datum`，但 `Datum` 本身不是完整语义。 它必须和 `isnull`、type metadata、typmod、slot owner、MemoryContext、varlena shape 一起解释。 `varlena` 不是一种单一内存布局。 它可以是普通 inline、short header、compressed、on-disk external、indirect、expanded。 因此读取 varlena 时要先判断使用场景：

```text
需要 bytes 扫描：优先 packed 宏。
需要宽字段对齐：使用 aligned detoast 或 memcpy。
需要修改：必须 copy。
需要跨生命周期保存：必须证明 owner 或 copy 到合适 context。
```
TOAST 是存储层为了 tuple 大小和 IO locality 引入的形态转换。 detoast 是消费方为了得到可处理内存形态而触发的按需恢复。 两者不是同一个阶段。 写入时的 `heap_toast_insert_or_update()` 可能把 inline varlena 变成 toast pointer。 读取时的 `detoast_attr()` 可能把 toast pointer、compressed datum 或 expanded object 变成当前 context 中的普通 varlena copy。 正确性由多层机制叠加：
- catalog metadata 决定 `Datum` 如何解释。
- fmgr strict/null 协议决定函数能否读取参数。
- varlena alignment 规则决定能否直接访问内部字段。
- MemoryContext 决定 detoast copy 和 output cstring 的寿命。
- table AM / toast relation / WAL / MVCC 决定 external value 的持久化和一致性。
观测上，能直接看到的是类型 metadata、toast table、物理大小、逻辑长度、buffer 和 profile 栈。 看不到的是内置 per-query detoast 次数、每个 copy 的 owner、同一值是否被重复 detoast。 因此诊断这类问题要把 SQL 现象、源码断点、profile 和 memory context 结合起来。 可迁移规律：

```text
当系统用一个统一 word-sized value 协议承载多种物理表示时，
value bits 只能表达“如何到达对象”，不能表达“对象归谁、能活多久、是否已展开”；
真正的语义必须从 metadata、owner、context 和消费边界恢复。
```
这些判断依赖 workload、数据分布、编码、压缩算法、缓存状态和 PostgreSQL 版本。 不要把一次实验里的 TOAST 行为泛化成所有 varlena 查询的成本模型。
