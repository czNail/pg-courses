# PostgreSQL expression deform / detoast cost
## 课程定位
前置知识：已经读过 TupleTableSlot、TupleDesc、ExprContext、per-tuple memory、Datum / varlena、fmgr function call boundary 和 operator/function lookup。

本节唯一主问题：
```text
slot deform、attribute cache、detoast 和 pass-by-value/reference 如何放大表达式成本？
```
核心矛盾：

```text
executor 想用统一的 Datum + isnull 协议低拷贝求值表达式；
但 heap tuple 是按物理顺序打包的字节串，属性 offset、NULL bitmap、varlena、TOAST、byval/byref 和函数消费方式都会把一次看似简单的 Var 读取放大成 CPU、cache、内存和 IO 成本。
```
学完后应能独立判断：
- 一个表达式热点到底是函数本体慢，还是先被 slot deform、varlena detoast、tuple copy 或 collation 比较放大。
- `slot_getattr()` 为什么有时几乎是数组读取，有时会追着前序列扫描 tuple。
- `attcacheoff` 能优化什么，不能优化什么。
- 一个 `Datum` 只是 word copy、指向 tuple 内部、指向 detoast copy，还是指向 expanded object。
- `EXPLAIN`、`pg_stat_*`、perf、gdb 分别能看到哪些成本，哪些只能推断。

本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。
## 1. 本节在总主线中的位置
04 目录前面已经建立了三条线。

第一条线是 executor 用 `ExecProcNode()` 按需拉取 tuple。
第二条线是节点之间通过 `TupleTableSlot` 传递 tuple，而不是直接传 `HeapTuple`。
第三条线是表达式解释器和 fmgr 用 `Datum + isnull` 调用函数、操作符和类型例程。

本节把这三条线合到一个热点问题上：
```text
表达式看到的是一个 Var 或函数参数；
底层可能先要把 slot 中的物理 tuple deform 成 Datum 数组；
函数再可能把 by-reference Datum detoast、decompress、slice、copy 或 flatten。
```
这解释了很多运行时现象。

同一张表、同样的行数，只改列顺序或只改投影列，表达式 CPU 可能明显变化。
同一个 `text` 列，`pg_column_size(col)`、`octet_length(col)`、`length(col)`、`col = constant` 和 `substring(col, 1, 10)` 的成本可能完全不同。
一个 profile 栈里看到 `ExecInterpExpr()`，并不代表真正热点在解释器 dispatch。

栈下面的 `slot_deform_heap_tuple()`、`detoast_attr()`、`toast_fetch_datum()`、`pglz_decompress_datum()`、`memcmp()` 或 collation compare 可能才是成本主体。
本节不重新讲完整 TOAST 写入、不展开所有 fmgr 调用协议，也不讲 JIT codegen 细节。
这些主题分别在相邻课程中处理。

本节只回答一个问题：表达式 hot path 如何被 tuple 物理布局和 varlena 消费方式放大。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：

```text
简单 Var fast path 通过 slot_getattr() 读取属性；
通用解释器通常先用 EEOP_*_FETCHSOME 调 slot_getsomeattrs()，再让 EEOP_*_VAR 直接读 tts_values / tts_isnull；
deform 只产生 Datum，不主动 detoast 普通 varlena；
真正消费 by-reference varlena 的函数或操作符再通过 PG_GETARG_* / DatumGet* 宏决定是否 detoast、copy、slice 或保持 packed。
```
这里有两个延迟策略。
第一个延迟是 attribute deform 延迟。

slot 不在拿到 tuple 时立刻拆出全部列。
它只在 `attnum > tts_nvalid` 时调用 slot ops 的 `getsomeattrs`。
如果表达式只读第 1 列，后面的宽列不需要被 deform。

如果表达式读第 80 列，slot 可能必须扫描前面的物理布局才能知道第 80 列在哪里。
第二个延迟是 detoast 延迟。
deform 对 by-reference 属性通常只是把指针放进 `Datum`。

这个指针可能指向 tuple 内部的 inline varlena、short header、compressed varlena、external TOAST pointer、indirect datum 或 expanded datum。
只有当具体函数需要普通 varlena 内容时，才会通过 `PG_GETARG_TEXT_P()`、`PG_GETARG_TEXT_PP()`、`DatumGetTextPSlice()` 等入口触发 detoast。
这两个延迟策略都在省工作。

但它们也让成本变得不直观。
用户写的是一个表达式。
成本却可能落在 tuple descriptor、slot ops、heap tuple layout、TOAST relation、compression method、per-tuple memory 和函数宏之间。

本节的 mental model 是：
```text
表达式成本 = step dispatch
           + Var fetch / slot deform
           + byval/byref 传递
           + 函数或操作符消费 Datum 的方式
           + detoast / decompress / copy / collation / output
           + per-tuple cleanup
```
只有前一项很少能解释完整慢点。

## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/executor/tuptable.h` | `TupleTableSlot`、`tts_nvalid`、`tts_values`、`slot_getattr()`、slot ops 边界。 |
| 2 | `src/backend/executor/execExprInterp.c` | `EEOP_*_FETCHSOME`、`EEOP_*_VAR`、`ExecJustVarImpl()` 如何把 Var 读取转成 slot deform 或数组读取。 |
| 3 | `src/backend/executor/execTuples.c` | `slot_deform_heap_tuple()`、heap/minimal/buffer slot 的 `getsomeattrs`、materialize、clear。 |
| 4 | `src/include/access/tupdesc.h` | `TupleDescData`、`CompactAttribute`、`attcacheoff`、`firstNonCachedOffsetAttr`、`firstNonGuaranteedAttr`。 |
| 5 | `src/backend/access/common/tupdesc.c` | `TupleDescFinalize()` 如何决定哪些属性 offset 可以缓存。 |
| 6 | `src/include/access/htup_details.h` | `heap_getattr()`、`fastgetattr()`、`fetchatt()` 的单属性路径。 |
| 7 | `src/backend/access/common/heaptuple.c` | `nocachegetattr()`、`heap_deform_tuple()`、by-reference Datum 指向 tuple 内部的规则。 |
| 8 | `src/backend/access/common/detoast.c` | `detoast_attr()`、`detoast_external_attr()`、`detoast_attr_slice()`、`toast_fetch_datum()`。 |
| 9 | `src/backend/utils/adt/varlena.c` | `text_length()`、`textoctetlen()`、`textcat()`、`texteq()`、`text_substring()` 如何选择 detoast 粒度。 |
| 10 | `src/include/fmgr.h` | `PG_GETARG_*`、`PG_DETOAST_DATUM*`、`PG_FREE_IF_COPY` 的调用者协议。 |
| 11 | `src/include/varatt.h` | short、compressed、external、indirect、expanded varlena 的判定宏。 |
阅读顺序不要从 `execTuples.c` 顶部线性读。

先看 `slot_getattr()` 和 `slot_getsomeattrs()`，确认表达式读列如何进入 slot。
再看 `EEOP_*_FETCHSOME` 和 `ExecJustVarImpl()`，确认通用解释器与 fast path 都没有绕过 slot。
然后读 `slot_deform_heap_tuple()`，确认 `tts_nvalid` 和 saved offset 如何推进。

再读 `TupleDescFinalize()`，确认 `attcacheoff` 是怎么来的。
最后读 `detoast.c` 和 `varlena.c`，确认什么时候才真正 fetch TOAST chunks 或 decompress。
本节要保留源码的真实 awkwardness。

同一个“取属性”语义有 slot 增量 deform、`heap_getattr()` 单属性、`heap_deform_tuple()` 全量 deform、minimal tuple 包装、buffer-backed slot 等多个路径。
这些不是理想化架构图中的重复。
它们对应不同 caller 的 ownership、hot path 和物理表示。

## 4. 关键数据结构与状态
### 4.1 `TupleTableSlot`
`TupleTableSlot` 是表达式取值的第一层状态边界。

本节关注这些字段组合：
| 字段 | 语义 |
| --- | --- |
| `tts_ops` | 当前 slot 实现，决定如何 `getsomeattrs`、`clear`、`materialize`、`copy`。 |
| `tts_nvalid` | `tts_values` / `tts_isnull` 中已经有效的属性数量。 |
| `tts_tupleDescriptor` | 属性数量、类型、byval、长度、对齐、缺省 missing value 和 offset cache 的来源。 |
| `tts_values` | deform 后的 `Datum` 数组；by-reference 元素可能只是指向底层 tuple。 |
| `tts_isnull` | 与 `tts_values` 并行的 NULL 语义。 |
| `tts_first_nonguaranteed` | 对 NOT NULL + byval 前缀的优化边界。 |
| `tts_mcxt` | slot 自身和数组所在 context，不等于每个 byref Datum 指向对象的 context。 |
`tts_nvalid` 不是“已经算过表达式”。

它只表示 slot 已经把前多少个物理属性转换成了 `Datum/isnull`。
如果同一行后续表达式再次读取已经 deform 的列，`slot_getattr()` 可以直接读数组。
如果读取更靠后的列，slot 会继续从保存的 offset 往后推进。

这就是增量 deform 的核心。
### 4.2 heap / minimal / buffer slot 的 saved offset
heap slot 和 minimal slot 都有一个 `off` 字段。

源码注释把它写成 `saved state for slot_deform_heap_tuple`。
这个 offset 是本行物理 tuple deform 的进度。
它不是 tuple descriptor 上的全局 offset cache。

它也不能跨 tuple 使用。
当 slot 接收新 tuple 或 clear 时，`tts_nvalid` 和这个 offset 必须回到可重新推进的状态。
buffer-backed slot 还多一个 `buffer` 字段。

如果 `buffer` 不是 `InvalidBuffer`，slot 持有 buffer pin。
这种情况下 by-reference Datum 可能指向 buffer page 内的 tuple 数据。
只复制 `Datum` 不能延长 buffer pin。

要跨越 slot clear、下层节点推进或 buffer pin 生命周期，就必须 materialize 或 copy。
### 4.3 `TupleDescData` 与 `CompactAttribute`
`TupleDescData` 不只是列定义清单。

它还携带 deform hot path 需要的紧凑元数据。
`CompactAttribute` 中本节最关键的字段是：
| 字段 | 语义 |
| --- | --- |
| `attcacheoff` | 如果该属性在 tuple 内 offset 固定，就保存 offset；否则为 `-1`。 |
| `attlen` | 固定长度、varlena 或 cstring 规则。 |
| `attbyval` | `Datum` 里是值还是指针。 |
| `attalignby` | tuple 内属性对齐规则。 |
| `attispackable` | 是否允许 short varlena 等 packed 形态。 |
| `atthasmissing` | tuple 物理列不足时是否可能补 missing value。 |
| `attisdropped` | dropped column 仍占 tuple descriptor 位置。 |
| `attnullability` | NOT NULL 约束状态，影响 guaranteed prefix 优化。 |

`TupleDescFinalize()` 会设置两个重要边界。
`firstNonCachedOffsetAttr` 表示从哪个属性开始没有可用 `attcacheoff`。
遇到 varlena、cstring、virtual generated attribute 或 offset 超过 `int16` 范围后，就不能继续缓存后续 offset。

`firstNonGuaranteedAttr` 表示从哪个属性开始不能保证“存在、非 NULL、byval、固定长度、非 dropped、非 missing”。
这个边界让 deform 可以对前缀走更便宜的路径。
注意：`attcacheoff` 是 TupleDesc 上的物理 offset 缓存。

它不是某个 tuple 的值缓存。
只要前面出现 NULL 或可变长属性，后续属性在不同 tuple 中 offset 就可能不同。
### 4.4 `Datum` by-value 与 by-reference

`Datum` 是统一值容器。
它不携带 type OID。
它不携带 NULL 语义。

它也不携带 ownership。
by-value 类型把值直接放进 `Datum`。
int4、bool、OID 这类值通常就是 word copy。

by-reference 类型把指针放进 `Datum`。
text、bytea、array、jsonb、numeric、record 等大量类型都属于这一类。
deform 一个 by-reference 属性时，常见结果只是：

```text
tts_values[i] = PointerGetDatum(pointer_inside_tuple_or_external_representation)
```
这一步没有自动复制实际 payload。
也没有自动 detoast。

后续函数如果只保存这个 `Datum`，保存的只是指针。
这个指针的有效性取决于 tuple、slot、buffer pin、MemoryContext 或 expanded object 生命周期。
### 4.5 varlena 形态

一个 `text` 逻辑值在表达式层可能有多种物理形态。
| 形态 | 典型判定 | 成本含义 |
| --- | --- | --- |
| 4-byte inline | `!VARATT_IS_EXTENDED(ptr)` | 可以直接用 `VARDATA()` / `VARSIZE()` 类宏。 |
| 1-byte short | `VARATT_IS_SHORT(ptr)` | header 更短，可能未按 4-byte 对齐，适合 `*_PP` / `*_ANY`。 |
| inline compressed | `VARATT_IS_COMPRESSED(ptr)` | 在主 tuple 内，但读取内容前可能要 decompress。 |
| external on-disk | `VARATT_IS_EXTERNAL_ONDISK(ptr)` | 主 tuple 里是 TOAST pointer，读取内容要访问 toast relation。 |
| external indirect | `VARATT_IS_EXTERNAL_INDIRECT(ptr)` | 指向内存中另一个 varlena，寿命由创建者保证。 |
| external expanded | `VARATT_IS_EXTERNAL_EXPANDED(ptr)` | 指向 expanded object，flatten 可能很贵。 |
deform 不会把这些形态统一成普通 varlena。

这个选择很重要。
如果函数只需要原始大小，可能不需要 fetch 全部数据。
如果函数需要按字符遍历，就可能必须 detoast、decompress 或至少取 slice。

因此，表达式成本不是由类型名 `text` 决定，而是由“函数如何消费这个 text Datum”决定。
## 5. 主流程源码 walkthrough
本节主流程从一个简单表达式开始。

SQL 例子可以想成：
```sql
SELECT length(t.c80)
FROM t
WHERE t.c1 = 1;
```
真实主链路可以压缩为：

```text
ExecProcNode()
  -> scan node produces TupleTableSlot
  -> ExprState step reads Var
  -> fast path: ExecJustVarImpl() -> slot_getattr()
  -> interpreter path: EEOP_*_FETCHSOME -> slot_getsomeattrs()
  -> EEOP_*_VAR reads tts_values / tts_isnull
  -> slot_deform_heap_tuple()
  -> tts_values[attnum - 1] contains Datum
  -> fmgr / builtin function consumes Datum
  -> PG_GETARG_* or DatumGet* may detoast
  -> detoast_attr() / detoast_attr_slice()
  -> result allocated in current memory context
  -> per-tuple context reset or slot clear releases short-lived objects
```
### 5.1 表达式 Var 读取不会绕过 slot
`execExprInterp.c` 里简单 Var 的 fast path 会进入 `ExecJustVarImpl()`。

这个函数做的事情很少。
它检查 slot kind，然后计算 `attnum`，最后调用 `slot_getattr()`。
关键含义是：

```text
即使表达式看起来只是读取一列，
真实成本也会落到 slot 当前是否已经 deform 到这列。
```
通用解释器路径稍有不同。
`EEOP_*_FETCHSOME` 先调用 `slot_getsomeattrs(slot, last_var)`，保证后续 referenced columns 已经进入 arrays。
然后 `EEOP_*_VAR` 只从 `tts_values[attnum]` 和 `tts_isnull[attnum]` 取值。
`ExecJustAssignVarImpl()` 也类似于 fast path：它用 `slot_getattr()` 取输入 slot，再写 result slot。
它把输入 slot 的某个属性读出，写到 result slot 的 `tts_values[resultnum]`。

这一步如果读到 by-reference Datum，写入 result slot 的仍可能只是同一个指针。
除非后续 materialize / copy，否则它不是深拷贝。
### 5.2 `slot_getattr()` 的两个分支

`slot_getattr()` 本身是 inline 小函数。
核心逻辑是：
```c
if (attnum > slot->tts_nvalid)
    slot_getsomeattrs(slot, attnum);

*isnull = slot->tts_isnull[attnum - 1];
return slot->tts_values[attnum - 1];
```
如果 `attnum <= tts_nvalid`，这几乎就是数组读取。
如果 `attnum > tts_nvalid`，成本转移给 slot ops。

对 virtual slot，values 本来就是表达式结果数组，通常不需要物理 deform。
对 heap、minimal、buffer heap slot，`getsomeattrs` 会进入 `slot_deform_heap_tuple()`。
因此同一个 `slot_getattr()` 在 profile 中可能代表两种完全不同的成本。

一种是 cheap cache hit。
另一种是触发物理 tuple 解析。
### 5.3 `slot_deform_heap_tuple()` 的时间轴

`slot_deform_heap_tuple()` 是本节最关键函数。
它接收 slot、物理 heap tuple、保存 offset 的指针、请求的属性数和是否支持 cstring。
函数先根据 tuple 是否有 NULL bitmap 确定 data 区起点。

然后用 `slot->tts_nvalid` 作为已经完成的属性数。
接着把 `slot->tts_nvalid` 设为 `reqnatts`。
后续循环负责把从旧 `tts_nvalid` 到 `reqnatts` 的属性填进 arrays。

这一步是增量的。
第一次读第 3 列，只 deform 到第 3 列。
同一行第二次读第 5 列，从第 4 列继续。

同一行第三次再读第 2 列，不再重新解析前两列。
### 5.4 NULL bitmap 如何破坏固定 offset
heap tuple 中 NULL 属性不占 data 区空间。

这意味着只要前面有 NULL，后面属性的 offset 就依赖该 tuple 的 NULL bitmap。
`slot_deform_heap_tuple()` 在有 NULL 时会找 `firstNullAttr`，并填充 `isnull` 数组。
在第一个 NULL 之前，且 tuple descriptor 允许使用 cached offset 时，可以用 `attcacheoff`。

到了第一个 NULL 之后，函数必须按属性顺序推进 offset，并跳过 NULL。
因此，同样是读第 50 列：
- 如果前 49 列都是固定长度、NOT NULL、byval，deform 可能很便宜。
- 如果第 2 列是 nullable 或 varlena，后面 offset 可能要逐列计算。
- 如果请求列很靠后，前序列的布局成本会被放大。

这不是 SQL 层能直接看到的状态。
它来自 heap tuple 物理格式。
### 5.5 `attcacheoff` 的真实作用

`attcacheoff` 优化的是“固定 offset 的前缀”。
`TupleDescFinalize()` 只有在属性固定长度、非 virtual generated 且 offset 能放入 `int16` 时，才继续缓存。
遇到 varlena 或 cstring 后就停止。

所以 `attcacheoff` 的收益最明显出现在：
- 表前部是固定长度列。
- 这些列没有 NULL 干扰。
- 表达式经常读取这些前部列。
- TupleDesc 已经 finalize，compact attrs 可用。
`attcacheoff` 不能优化：

- varlena 本身的长度解析。
- TOAST fetch 或 decompress。
- 第一个 NULL 后面的动态 offset。
- by-reference 对象生命周期。
- 函数对 `Datum` 的 detoast 决策。
把 `attcacheoff` 理解成“列值缓存”是错误的。
它只是帮助从 tuple data 区算地址。

### 5.6 `heap_getattr()` 与 slot deform 的差异
`heap_getattr()` / `fastgetattr()` 是另一条单属性路径。
如果 tuple 无 NULL 且目标属性有 `attcacheoff`，`fastgetattr()` 可以直接 `fetchatt()`。

否则进入 `nocachegetattr()`。
`nocachegetattr()` 会从某个已知 offset 或 tuple 起点向目标属性推进。
源码注释提醒：如果对很多列循环调用 `heap_getattr()`，一旦涉及 noncacheable offset，可能变成 `O(N^2)`。

slot 的增量 deform 正是为了避免上层表达式重复从头走。
但是这个避免只在同一 slot、同一 tuple、同一递增 deform 状态内成立。
如果调用者绕过 slot，或每次都在不同 tuple/slot 上做高位属性读取，仍然会付出前序列扫描成本。

### 5.7 deform 不等于 detoast
`heap_deform_tuple()` 的注释写得很直接：
对 pass-by-reference 类型，放进 `Datum` 的指针会指向给定 tuple。

`slot_deform_heap_tuple()` 也是同一类语义。
它把属性变成 `Datum`。
它不负责把每个 `text` 都变成平坦、未压缩、独立 palloc 的内存块。

这样可以避免大量无用拷贝。
例如一个查询只检查 `id`，宽 `text` 列即使存在，也不会因为 tuple 进入 slot 就被 detoast。
但当表达式函数确实消费该 `text`，成本会在更晚位置爆发。

这就是很多 profile 中 `slot_deform_heap_tuple()` 和 `detoast_attr()` 同时出现的原因。
前者负责找到 `Datum`。
后者负责让函数看到自己需要的 varlena 形态。

### 5.8 varlena 函数如何决定 detoast 粒度
`varlena.c` 中有几个很好的对照。
`textoctetlen()` 取 `PG_GETARG_DATUM(0)` 后调用 `toast_raw_datum_size()`。

它的注释明确说不需要 detoast input。
`text_length()` 在单字节编码下也可以用 raw size。
多字节编码下需要字符数，只能拿 `DatumGetTextPP()` 后遍历 bytes。

`texteq()` 对 deterministic collation 先比较 `toast_raw_datum_size()`。
长度不同可以直接返回 false，避免 detoast 一个或两个值。
长度相同才 `DatumGetTextPP()` 并 `memcmp()`。

`text_substring()` 对单字节编码可以走 `DatumGetTextPSlice()`，尝试只 fetch slice。
多字节编码下，为了字符边界，可能要保守取更大的 slice，再计算字符位置。
因此，同一个 `text` 列：

- 只要原始大小，可能不 fetch toast chunks。
- 要字节前缀，可能只 fetch slice。
- 要字符长度，可能必须看更多内容。
- 要 collation 比较，可能进入 locale 相关路径。
表达式成本被“函数需求”决定，而不是被 `slot_getattr()` 单独决定。
### 5.9 detoast 主路径

`detoast_attr()` 保证返回非 extended varlena。
它会处理几类输入：
- on-disk external：`toast_fetch_datum()` 读取 toast relation chunks；如果 fetched 结果仍 compressed，再 decompress。
- indirect：解引用后递归 detoast；必要时复制，保证结果可 pfree。
- expanded：flatten 成 flat varlena。
- compressed inline：decompress。
- short header：复制成 4-byte header。

`detoast_external_attr()` 的目标略不同。
它把 external source 拉回内存，但结果可能仍 compressed 或 short。
`detoast_attr_slice()` 尝试只取一段。

对非压缩 external datum，可以直接 fetch slice。
对压缩 external datum，如果是 PGLZ prefix，可能根据目标解压长度估算需要的压缩前缀。
对 LZ4，源码注释说明至少当前没有 API 能确定需要多少压缩数据，因此可能要取完整值。

这些差异解释了为什么 “substring 只取前 10 个字符” 不总是等价于 “只读 10 个字节”。
### 5.10 结果回到 per-tuple memory
detoast 结果通常在当前 memory context 下 `palloc`。

表达式求值期间，当前 context 往往是 `ecxt_per_tuple_memory`。
如果函数返回的结果要跨 tuple 保存，必须复制到更长生命周期 context。
如果只是本行过滤或投影临时使用，per-tuple reset 会批量释放。

这就是 detoast 成本的另一半。
它不只是 CPU 或 IO。
它还可能带来 palloc、memcpy、memory context chunk、cache miss 和 reset 时的批量清理成本。

## 6. 状态随时间推进的完整故事
用一行 tuple 的生命周期串起来会更清楚。
### 6.1 scan 节点产生 slot

table AM 或 scan 节点拿到一条可见 tuple。
执行器把它放进 scan slot。
如果是 buffer-backed heap slot，slot 可能持有 buffer pin。

这时 `tts_nvalid` 通常为 0。
物理 tuple 在 slot 里，但还没有把所有属性拆成 `Datum`。
### 6.2 qual 先读前部列

`WHERE id = 1` 读取第 1 列。
表达式 step 调 `slot_getattr(slot, 1, &isnull)`。
slot 只 deform 到第 1 列。

如果第 1 列是固定长度 byval 且 offset cached，成本很低。
`tts_nvalid` 变成 1。
saved `off` 记录本行 deform 进度。

### 6.3 projection 读取后部宽列
targetlist 需要第 80 列 `large_text`。
`slot_getattr(slot, 80, &isnull)` 发现 `80 > tts_nvalid`。

slot 从第 2 列继续 deform 到第 80 列。
如果中间有 nullable、dropped、missing、varlena 或 noncacheable offset，函数必须按 tuple data 顺序推进。
第 80 列进入 `tts_values[79]`。

如果它是 external TOAST pointer，此时 `Datum` 仍可能只是 pointer。
### 6.4 函数消费该 Datum
`length(large_text)` 进入 varlena 函数。

函数看数据库编码和需求决定是否 `DatumGetTextPP()`。
如果需要字符遍历，可能触发 detoast。
如果值 external on-disk，`detoast.c` 打开 toast relation，fetch chunks，必要时 decompress。

返回的 flat 或 packed varlena 在当前 context 中。
### 6.5 本行结束
如果该 detoast copy 只在本行使用，下一轮 `ResetExprContext()` 或 per-tuple context reset 会回收。

如果投影结果写入 virtual slot，并被上层节点马上消费，指针仍然可能有效。
如果结果要进入 tuplestore、hash table、sort 或跨节点保存，需要 copy / materialize 到对应 owner。
否则下一轮 reset 后就会悬空。

### 6.6 slot 被清理或复用
scan 节点推进到下一行时，slot 会 clear 或覆写内容。
buffer-backed slot 需要释放旧 buffer pin。

slot-owned tuple 需要 pfree。
`tts_nvalid` 回到 0。
saved offset 回到新 tuple 的初始状态。

上一次 by-reference Datum 如果仍被外部保存，就必须已经被复制。
否则它只是一个指向旧 tuple 或旧 context 的裸指针。
## 7. 生命周期 / ownership / cleanup

### 7.1 谁创建状态
slot 通常在 `ExecInit*` 阶段创建。
`TupleDesc` 可能来自 relcache、计划 targetlist、匿名 record、projection info 或节点私有结果类型。

`tts_values` 和 `tts_isnull` 数组随 slot descriptor 分配。
heap tuple 内容由 scan/table AM、上游节点或 materialize/copy 路径提供。
TOAST detoast copy 由具体函数在当前 memory context 下创建。

这些对象不是同一个 owner。
### 7.2 谁持有物理 tuple
slot 是否拥有 tuple，要看 slot type 和 flags。

`TTS_FLAG_SHOULDFREE` 表示物理 tuple 由 slot 释放。
buffer-backed slot 可能不设置 `SHOULDFREE`，因为 tuple 指向 buffer page。
这种 slot 的 `buffer` 字段决定 clear 时是否 `ReleaseBuffer()`。

minimal slot 通过 `HeapTupleData` workspace 让 deform 逻辑可以按 heap tuple 方式处理 minimal tuple。
不要把 `Datum` 指针当成 owner。
它只是当前 slot 内容的一种视图。

### 7.3 谁持有 detoast copy
`detoast_attr()`、`toast_fetch_datum()`、`toast_decompress_datum()` 通常返回 palloc 内存。
这块内存属于调用时的 current memory context。

fmgr 不自动切换到某个安全长生命周期 context。
表达式执行通常依赖 `ExprContext` 的 per-tuple context 来承接短命结果。
函数如果要跨调用保存 detoast 结果，应复制到 `fn_mcxt`、aggregate context、tuplestore context 或节点私有 context。

只靠 `PG_FREE_IF_COPY()` 不能解决生命周期错误。
它只帮助函数在本次调用后释放 detoast copy。
### 7.4 ERROR / abort 时谁兜底

slot、TupleDesc、executor state 的大多数内存由 MemoryContext tree 收尾。
buffer pin、relation lock、toast relation open/close 等外部资源依赖相应 cleanup 路径和 ResourceOwner 机制兜底。
`toast_fetch_datum()` 打开 toast relation 后在正常路径关闭。

如果中间 ERROR，PostgreSQL 的错误恢复会释放资源 owner 下的锁、pin 和 relation refs。
表达式 per-tuple allocation 不需要逐个 pfree。
ERROR 进入上层清理后，对应 context 会被 reset 或 delete。

但是 C 函数如果把 by-reference pointer 放进更长生命周期结构，而没有深拷贝，ERROR cleanup 不能修复已经写坏的语义。
### 7.5 长期对象如何失效
slot 本身是 backend-local。

`TupleDesc` 如果来自 relcache/typcache，可能受 invalidation 影响。
表达式解释器中 `get_cached_rowtype()` 的注释提醒：返回的 TupleDesc 不保证 pinned；如果要跨可能发生 cache invalidation 的操作使用，包括 detoasting input tuples，应当 pin 或增加 refcount。
这是一个非常具体的边界。

detoast 可能访问 catalog、打开 relation、触发错误或 cache invalidation。
所以不要把“我已经有一个 TupleDesc 指针”理解成“它在所有后续操作中稳定可用”。
## 8. 正确性机制层次

本节正确性不是靠单一机制保证。
它由几个局部协议叠起来。
| 层次 | 机制 | 保证什么 | 不保证什么 |
| --- | --- | --- | --- |
| tuple visibility | scan/table AM 与 MVCC snapshot | 返回给 executor 的 tuple 是当前执行语义下可见的候选 | 不保证表达式取属性便宜 |
| slot protocol | `tts_nvalid`、slot ops、clear/materialize | 已 deform 值与 slot 内容一致，资源按 slot type 释放 | 不保证 byref Datum 可跨 slot 生命周期 |
| TupleDesc | `CompactAttribute`、`attcacheoff`、natts、missing attrs | 解释 tuple 字节布局和逻辑列边界 | 不保证后续 varlena 已 detoast |
| heap layout | NULL bitmap、alignment、varlena length | 正确找到属性起点和长度 | 不保证高位属性随机访问是 O(1) |
| fmgr / varlena | `PG_GETARG_*`、`PG_FREE_IF_COPY` | callee 按类型协议消费 Datum | 不自动复制长期保存的 byref 参数 |
| MemoryContext | per-tuple reset、executor context delete | 短命 detoast copy 批量释放 | 不释放未注册的外部资源语义 |
| ResourceOwner | buffer pin、relation lock 等 | ERROR 时外部资源收尾 | 不复制 tuple payload |

MVCC 保证的是“读哪个版本”。
slot deform 保证的是“如何从这个版本里取列”。
detoast 保证的是“如何把某个 varlena 表示转换为函数需要的形式”。

这些层次不能互相替代。
把一个查询慢解释成“MVCC 慢”或“表达式慢”，都太粗。
需要继续问：慢在可见性检查、slot deform、TOAST fetch、decompress、函数计算、collation，还是输出格式化。

## 9. 错误路径 / 异常路径 / fallback
### 9.1 请求属性超过 TupleDesc
slot ops 的 `getsomeattrs` 必须在请求属性超过 slot TupleDesc 时报 ERROR。

`slot_deform_heap_tuple()` 末尾如果发现物理 tuple 属性不足，会调用 `slot_getmissingattrs()`。
如果请求超过逻辑 TupleDesc 范围，则不能静默返回垃圾。
这保证 dropped / missing / inheritance 场景仍然按 TupleDesc 语义解释。

### 9.2 物理 tuple 属性不足
heap tuple 物理上可能没有后续新增列。
逻辑 TupleDesc 仍然可能有这些列。

缺失属性通过 missing attr 或 NULL 语义补齐。
这条 fallback 让 `ALTER TABLE ADD COLUMN ... DEFAULT` 这类历史 tuple 可以继续被旧物理行承载。
成本含义是：读取后部新增列不一定只是读 tuple data。

它可能进入 missing attr 补齐路径。
### 9.3 noncacheable offset 的退化
没有 `attcacheoff` 时，系统不会猜 offset。

它从可证明的起点按 alignment 和 length 逐列推进。
`nocachegetattr()` 会尽量从最高的已缓存前序属性或第一个 NULL 前面开始。
slot deform 会利用 saved offset 增量推进。

fallback 是“更慢但正确”。
这正是 attribute cache 的边界。
### 9.4 detoast 外部值

on-disk TOAST pointer 进入 `toast_fetch_datum()`。
函数会打开 toast relation，按 value id fetch chunks。
如果 toast pointer 类型不对，源码直接 `elog(ERROR)`。

如果压缩方法 id 非法，`toast_decompress_datum()` 也会 ERROR。
这些错误通常意味着数据损坏、宏误用或内部不变量被破坏。
正常 SQL 代码不应该把非 on-disk datum 强塞给 toast fetch。

### 9.5 compressed slice 的退化
`detoast_attr_slice()` 对压缩值并不总能只读目标 slice。
PGLZ prefix 可以估算需要的压缩前缀。

LZ4 路径可能需要获取完整压缩数据。
如果 slicelength 溢出，代码会退化为 fetch all。
这类 fallback 对正确性友好，对性能不总友好。

所以 `substring(large_text, 1, 10)` 的实际成本仍然 workload-dependent。
### 9.6 `PG_FREE_IF_COPY` 的边界
很多 varlena 函数在 `DatumGetTextPP()` 后调用 `PG_FREE_IF_COPY(arg, n)`。

如果宏返回的是原始参数指针，不会释放。
如果宏返回的是 detoast copy，会释放。
这解决的是本次函数调用内的临时 copy。

它不能让 caller 保存的 by-reference Datum 变安全。
也不能替代 per-tuple context reset。
### 9.7 表达式短路

strict 函数、boolean expression、CASE、AND/OR 短路会避免部分 step 执行。
如果后续 Var 没被读取，就不会触发对应 slot deform。
如果后续 varlena 函数没被调用，就不会触发 detoast。

这意味着表达式顺序和 selectivity 会影响 CPU。
但是 SQL 语义和 planner 重写可能改变实际执行结构。
不要只按原始 SQL 文本判断 detoast 是否发生。

## 10. 成本、资源与跨模块传播
### 10.1 成本模型
本节成本可以拆成四类。

第一类是 attribute access cost。
它随读取列的最大 attnum、前序可变长列数量、NULL bitmap、missing attrs、dropped columns 和 TupleDesc cacheability 增长。
第二类是 by-reference payload cost。

它随 varlena 大小、压缩方法、TOAST chunks 数量、是否 external、是否 expanded、函数是否需要完整内容增长。
第三类是 memory cost。
它随 detoast copy 次数、palloc chunk 数量、memcpy 字节数、per-tuple reset 频率增长。

第四类是 cache / branch cost。
它来自 tuple descriptor 间接访问、NULL bitmap 分支、varlena header 分支、compression 分支和函数 dispatch。
一个简化公式是：

```text
per tuple cost
  ~= max_requested_attnum_deform_cost
   + number_of_byref_consumptions * detoast_or_pointer_cost
   + function_body_cost
   + copy_or_materialize_cost
```
对行数 N，成本近似乘 N。
对 join，内侧表达式可能随 outer loops 放大。

对 aggregate/hash/sort，by-reference key 还可能被复制到长期结构中。
### 10.2 列顺序放大
PostgreSQL heap tuple 不是列式存储。

读取第 50 列可能需要理解前 49 列的物理布局。
如果高频过滤列放在固定长度、NOT NULL、byval 的前部，deform 更容易走 cheap path。
如果高频过滤列放在多个 nullable varlena 后面，每行都可能付出前序布局扫描。

这不是建议所有 schema 都按 microbenchmark 重排。
列顺序涉及兼容性、DDL 成本、应用接口和存储布局。
但在诊断极端 CPU 热点时，列顺序确实可能是内核层面的放大因子。

### 10.3 投影剪枝与表达式选择性
planner 如果能去掉不需要的列，executor 就不会读取它们。
`SELECT count(*) FROM t` 不需要 deform 宽 text 列。

`SELECT * FROM t WHERE id = 1` 需要输出所有列，最终会触发更多 deform 和可能的 output detoast。
过滤条件的选择性也重要。
如果 cheap qual 先过滤掉大多数行，后续 expensive projection 或 function 不会执行。

如果 expensive function 在过滤前就必须求值，detoast 会按输入行数放大。
### 10.4 TOAST IO 与 buffer 系统
TOAST external value 会访问 toast relation。

这会进入 table AM、buffer manager、relation cache、lock manager 和 IO 路径。
`EXPLAIN (ANALYZE, BUFFERS)` 可能看到额外 shared hit/read。
如果 toast relation 不在缓存里，会有读 IO。

如果值压缩，还会有 decompression CPU。
如果函数只需要 raw size，可能完全避免这些成本。
因此 TOAST 成本既可能表现为 CPU，也可能表现为 buffer read，还可能被 OS cache 掩盖。

### 10.5 fmgr 与函数边界
fmgr 调用本身已经在前一节讲过。
本节只强调一点：callee 决定是否 detoast。

`FunctionCallInvoke()` 不会自动 detoast 参数。
strict NULL 短路也不会自动把 varlena 转成 flat。
C 函数通过 `PG_GETARG_TEXT_P()`、`PG_GETARG_TEXT_PP()`、`PG_GETARG_BYTEA_P()`、`PG_GETARG_DATUM()` 等宏表达自己的消费需求。

如果扩展函数一上来用 `_P` 强制完整 detoast，就可能比核心函数更贵。
如果扩展函数保存 `_PP` 返回的 pointer 到长期状态，又可能制造生命周期 bug。
### 10.6 output boundary

即使表达式本身不 detoast，输出阶段也可能需要格式化。
文本协议要调用类型 output 函数。
binary 协议要调用 send 函数。

某些 output/send 函数会 detoast 或复制。
所以 `SELECT large_text FROM t` 的成本可能不在 qual 或 projection，而在 `DestReceiver` / `printtup` 的输出边界。
本节的分析方法仍然适用：

找 `Datum` 从 slot 到函数的消费点。
不要只看 scan node。
### 10.7 parallel query 边界

并行 worker 不能共享 leader 的 slot 指针、detoast copy 或 `FmgrInfo` 指针。
每个 worker 在本进程内有自己的 executor state、slot、per-tuple context 和 detoast allocation。
TOAST relation 的 IO 也会由 worker 自己发起。

所以并行查询可能减少 wall time，但不会让单行 detoast 成本消失。
如果每个 worker 都触发大量 detoast，IO 和 memory bandwidth 可能成为新的瓶颈。
### 10.8 JIT 边界

JIT 可以降低 expression dispatch 和部分 deform 的解释器成本。
但 JIT 不能消除 heap tuple 的物理布局约束。
它也不能让 external TOAST value 不需要 fetch。

如果 profile 显示主要成本在 `toast_fetch_datum()`、decompression 或 locale compare，JIT 的收益有限。
如果主要成本在大量简单 Var 和算术表达式，JIT 或 deform specialization 才更可能有效。
## 11. 观测与诊断入口

### 11.1 可直接看到的现象
`EXPLAIN (ANALYZE, BUFFERS)` 能看到节点时间、行数、loops 和 buffer 访问。
如果 detoast 引发 toast relation buffer 访问，可能在 shared hit/read 中体现。

`EXPLAIN (ANALYZE, TIMING OFF, BUFFERS)` 可以减少计时扰动，保留 buffer 信息。
`pg_stat_statements` 能看到归一化 query 的总耗时、calls、rows、shared_blks_hit/read。
`pg_stat_io` 可以观察 backend 级 IO 行为，但粒度不是单个表达式。

`pg_statio_user_tables` 和 toast 表相关统计可以辅助判断 toast relation 是否被频繁访问。
`pg_stat_user_functions` 能看到被 track 的 SQL/C 函数累计时间，但核心表达式 opcode 和 slot deform 不作为普通函数暴露。
### 11.2 必须依赖 profiler 的现象

slot deform 和 detoast 主要是 CPU hot path。
它们没有专门的 `pg_stat_slot_deform_count`。
要定位它们，通常需要 perf、gprof、eBPF、flamegraph 或采样 profiler。

典型栈符号包括：
```text
ExecInterpExpr
ExecJustScanVar / ExecJustVarImpl
slot_getsomeattrs
slot_deform_heap_tuple
tts_heap_getsomeattrs
nocachegetattr
detoast_attr
detoast_attr_slice
toast_fetch_datum
table_relation_fetch_toast_slice
pglz_decompress_datum
lz4_decompress_datum
text_length
texteq
varstr_cmp
```
如果火焰图里 `slot_deform_heap_tuple` 占比高，先看列访问位置、TupleDesc、NULL/varlena 分布和投影剪枝。

如果 `detoast_attr` 或 `toast_fetch_datum` 占比高，先看函数是否真的需要完整 varlena、TOAST storage、压缩方法和 output boundary。
如果 `memcmp` 或 collation compare 占比高，问题可能已经越过 detoast，进入比较逻辑。
### 11.3 gdb 断点入口

源码跟读时可以设置这些断点：
```gdb
break slot_deform_heap_tuple
break detoast_attr
break detoast_attr_slice
break toast_fetch_datum
break ExecJustVarImpl
```
断在 `slot_deform_heap_tuple()` 时，优先打印：

```gdb
p reqnatts
p slot->tts_nvalid
p slot->tts_tupleDescriptor->firstNonCachedOffsetAttr
p slot->tts_tupleDescriptor->firstNonGuaranteedAttr
p ((HeapTupleTableSlot *) slot)->off
```
断在 detoast 时，优先判断：
```gdb
p VARATT_IS_EXTERNAL_ONDISK(attr)
p VARATT_IS_COMPRESSED(attr)
p VARATT_IS_SHORT(attr)
p VARATT_IS_EXTERNAL_EXPANDED(attr)
```

这些宏在 gdb 中不一定都能直接展开。
必要时可以在源码里临时加 `elog(LOG, ...)` 或计数器。
临时插桩要放在个人分支，不要带入产品代码。

### 11.4 能看到、只能推断、看不到
能直接看到：
- plan node 行数、loops、时间。
- buffer hit/read，包括可能的 toast relation 访问。
- 函数累计时间，如果开启函数统计且函数在统计范围内。
- perf 栈中的具体 C 符号。

只能推断：
- 某个表达式是否因为列顺序触发大量前序 deform。
- detoast copy 是在 qual、projection、output 还是 aggregate key 中产生。
- per-tuple reset 回收了多少 detoast copy。
- `PG_GETARG_TEXT_PP()` 返回的是原始 pointer 还是 copy。
几乎不可见：

- 每个 slot 的 `tts_nvalid` 变化。
- 每个属性是否命中 `attcacheoff`。
- 每次 `Datum` pointer 的真实 owner。
- 每个 varlena 函数避免了多少 detoast。
因此诊断必须完成闭环：
```text
SQL / EXPLAIN 看到慢
  -> profiler 找到 C 热点
  -> gdb / 源码确认状态边界
  -> 回到 schema、SQL、函数和 TOAST storage 解释
```

## 12. 常见误区
### 12.1 “`slot_getattr()` 只是数组读取”
只在 `attnum <= tts_nvalid` 时近似成立。

如果请求属性超过当前 valid 前缀，它会触发 slot ops。
对 heap-like slot，这可能进入物理 tuple deform。
### 12.2 “`attcacheoff` 会缓存所有列 offset”

不会。
它只缓存固定 offset 的前缀。
varlena、cstring、virtual generated attribute、过大 offset 或 NULL 干扰都会让后续 offset 需要运行时计算。

### 12.3 “deform 就会 detoast”
不会。
deform 通常只把 tuple 属性表示成 `Datum/isnull`。

by-reference Datum 可能仍然是 TOAST pointer 或 compressed varlena。
具体函数消费时才决定是否 detoast。
### 12.4 “复制 `Datum` 就复制了值”

只对 by-value 类型成立。
对 by-reference 类型，复制 `Datum` 只是复制指针。
跨 slot、buffer pin、per-tuple context 或 expanded object 生命周期保存时必须深拷贝或 materialize。

### 12.5 “`PG_GETARG_TEXT_PP()` 总是零拷贝”
不总是。
它允许 packed 结果，通常比 `_P` 少做 short header 规范化。

但遇到 compressed 或 external 仍可能 detoast。
返回 pointer 是否需要 `PG_FREE_IF_COPY()` 由宏协议决定。
### 12.6 “只看 EXPLAIN 就能定位 detoast”

EXPLAIN 不能直接告诉你 detoast 次数。
它只能给节点时间、行数和 buffer。
detoast CPU、decompress 和 memcpy 通常要靠 profiler。

toast relation IO 也可能混在节点 buffers 中，需要结合表结构和栈。
### 12.7 “宽列不出现在 SELECT 列表就没有成本”
大多数情况下，如果表达式确实不读取该列，就不会 deform / detoast 它。

但 `SELECT *`、output、trigger、RETURNING、row_to_json、whole-row Var、FDW、logical decoding 或某些函数可能间接消费宽列。
要看最终执行计划和表达式树，不只看肉眼 SQL。
### 12.8 “JIT 可以解决 detoast 问题”

JIT 可以减少解释器和部分 deform overhead。
它不能消除 TOAST fetch、decompress、copy、locale compare 或 output formatting 的必要工作。
优化前先看 profile 栈。

## 13. 课堂实验
### 实验 1：列顺序与 slot deform
目标：观察高位属性访问如何放大 `slot_deform_heap_tuple()`。

准备两张表。
一张把热过滤列放在前面。
一张把热过滤列放在多个 varlena 后面。

```sql
CREATE TABLE deform_front (
  id int NOT NULL,
  k int NOT NULL,
  pad1 text,
  pad2 text,
  pad3 text
);
CREATE TABLE deform_back (
  pad1 text,
  pad2 text,
  pad3 text,
  id int NOT NULL,
  k int NOT NULL
);
INSERT INTO deform_front
SELECT g, g % 100, repeat('x', 50), repeat('y', 80), repeat('z', 120)
FROM generate_series(1, 1000000) AS g;

INSERT INTO deform_back
SELECT repeat('x', 50), repeat('y', 80), repeat('z', 120), g, g % 100
FROM generate_series(1, 1000000) AS g;
ANALYZE deform_front;
ANALYZE deform_back;
```
对比：

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING OFF)
SELECT count(*) FROM deform_front WHERE k = 42;
EXPLAIN (ANALYZE, BUFFERS, TIMING OFF)
SELECT count(*) FROM deform_back WHERE k = 42;
```
如果环境允许，用 perf 采样 backend。

重点看 `slot_deform_heap_tuple()`、`nocachegetattr()`、`ExecInterpExpr()` 的占比差异。
解释时不要只看总时间。
要把行数、buffers、cache warmup、CPU governor 和并发噪声控制住。

### 实验 2：同一 text 值的不同消费方式
目标：观察函数如何决定 detoast 粒度。
准备数据：

```sql
CREATE TABLE detoast_cost (
  id int,
  payload text
);
INSERT INTO detoast_cost
SELECT g, string_agg(md5((g::bigint * 100000 + s)::text), '')
FROM generate_series(1, 50000) AS g
CROSS JOIN generate_series(1, 200) AS s
GROUP BY g;
ALTER TABLE detoast_cost ALTER COLUMN payload SET STORAGE EXTENDED;
VACUUM FULL detoast_cost;
ANALYZE detoast_cost;
```

对比：
```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING OFF)
SELECT sum(pg_column_size(payload)) FROM detoast_cost;
EXPLAIN (ANALYZE, BUFFERS, TIMING OFF)
SELECT sum(octet_length(payload)) FROM detoast_cost;

EXPLAIN (ANALYZE, BUFFERS, TIMING OFF)
SELECT sum(length(payload)) FROM detoast_cost;
EXPLAIN (ANALYZE, BUFFERS, TIMING OFF)
SELECT count(*) FROM detoast_cost WHERE substring(payload, 1, 4) = 'abcd';
```
预期不是固定数字。

重点是观察哪些查询触发更多 CPU 或 toast buffers。
回到源码解释：
- `pg_column_size` 的语义更接近物理大小。
- `pg_column_size` 对 varlena 走 `toast_datum_size()`，`octet_length` 可用 `toast_raw_datum_size()`。
- `length` 在多字节编码下可能要看字符。
- `substring` 是否 slice 取决于编码、压缩和函数路径。

### 实验 3：gdb 观察 `tts_nvalid`
目标：看到 slot 增量 deform 的状态推进。
在测试 backend 上执行一个只读查询。

设置断点：
```gdb
break slot_deform_heap_tuple
commands
  silent
  printf "reqnatts=%d nvalid=%d firstNonCached=%d firstNonGuaranteed=%d\n", reqnatts, slot->tts_nvalid, slot->tts_tupleDescriptor->firstNonCachedOffsetAttr, slot->tts_tupleDescriptor->firstNonGuaranteedAttr
  continue
end
```
先执行只读前部列的查询。

再执行读后部列的查询。
观察 `reqnatts` 如何变化。
如果输出太多，用条件断点限定 relation 或计数。

不要在生产环境这样做。
### 实验 4：源码临时计数 detoast
目标：建立 detoast 次数与 SQL 形态的关联。

在个人源码树中给 `detoast_attr()`、`detoast_attr_slice()`、`toast_fetch_datum()` 加临时计数或 `elog(DEBUG1, ...)`。
运行实验 2 的 SQL。
记录每种查询触发的次数。

然后删除插桩。
这个实验的价值不是保留代码。
价值是让你确认：

```text
一次 SQL 表达式
  -> 哪个函数宏触发 detoast
  -> detoast 是否 fetch external chunks
  -> 是否 decompress
  -> copy 放在哪个 memory context
```
## 14. 讨论题
1. 为什么 `slot_getattr(slot, 80)` 不能直接跳到第 80 列读取，而经常必须理解前 79 列的布局？

2. `attcacheoff` 为什么只适合固定长度前缀？NULL bitmap 和 varlena 分别如何破坏后续 offset？
3. 如果一个 by-reference `Datum` 来自 buffer-backed slot，把它保存到 hash table 中会有什么生命周期风险？
4. 为什么 deform 阶段不主动 detoast 所有 varlena？这个选择省了什么，又把成本推迟到哪里？

5. `PG_GETARG_TEXT_P()` 和 `PG_GETARG_TEXT_PP()` 在 alignment、short header、copy 和 pfree 协议上有什么不同诊断意义？
6. `EXPLAIN (ANALYZE, BUFFERS)` 看到 shared reads 增加时，如何判断是 heap scan、toast fetch、index access 还是其他节点造成？
7. 如果 perf 显示 `slot_deform_heap_tuple()` 占比很高，你会按什么顺序检查 SQL、schema、TupleDesc、列顺序和表达式树？

8. 如果 perf 显示 `detoast_attr_slice()` 占比很高，为什么不能立刻断言 `substring()` 只读了小 slice？
## 15. 本节小结
本节唯一主问题是：slot deform、attribute cache、detoast 和 pass-by-value/reference 如何放大表达式成本。

核心链路是：
```text
ExprState Var step
  -> slot_getattr()
  -> 或 EEOP_*_FETCHSOME / slot_getsomeattrs()
  -> tts_nvalid miss
  -> slot_deform_heap_tuple()
  -> Datum/isnull arrays
  -> fmgr / varlena function
  -> detoast / slice / decompress / copy
  -> per-tuple cleanup
```
`slot_getattr()` 不是单一成本。

它可能只是 `tts_values` 数组读取，也可能触发物理 tuple 增量 deform。
`attcacheoff` 优化的是固定 offset 前缀，不是列值缓存。
遇到 NULL、varlena、missing、dropped 或 noncacheable offset 后，系统会退回按物理顺序推进。

deform 不等于 detoast。
by-reference `Datum` 经常只是指针。
这个指针可能指向 tuple、buffer page、TOAST pointer、short header varlena、compressed varlena 或 expanded object。

真正 detoast 发生在函数或操作符消费 `Datum` 时。
`detoast_attr()`、`detoast_attr_slice()`、`toast_fetch_datum()` 和 compression 例程会把成本扩散到 CPU、IO、memory allocation 和 cache locality。
ownership 边界必须同时看 slot、buffer pin、TupleDesc refcount、MemoryContext 和 ResourceOwner。

复制 `Datum` 不等于复制 by-reference payload。
跨 tuple、slot、buffer pin 或 per-tuple context 保存时，必须 materialize 或深拷贝。
观测上，EXPLAIN 和 pg_stat 只能给节点级或累计线索。

slot deform 次数、`tts_nvalid` 推进、`attcacheoff` 命中、detoast copy 次数都不是普通 SQL 指标。
CPU 问题需要 profiler 和断点回到源码确认。
本节可迁移规律是：

```text
统一抽象把调用点变简单；
但真实成本会在抽象边界之后按物理表示、缓存命中、ownership 和消费语义重新展开。
```
判断表达式成本时，不要停在“表达式解释器慢”。
要继续问：

- 这个 Var 触发了多少 tuple deform？
- 目标列前面有没有破坏 offset cache 的列？
- `Datum` 是 by-value 还是 by-reference？
- by-reference 指针属于谁？
- 函数是否真的需要 flat detoasted value？
- detoast copy 在哪个 context 下释放？
- 可观测指标能证明哪一层，哪一层只能从栈和源码推断？
这些问题比背函数名更稳定。
