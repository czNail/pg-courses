# PostgreSQL Collation / Sort Support / Comparison Function
## 课程定位
前置知识：已经读过本目录的 `TupleTableSlot`、`ExprContext`、FMGR、operator lookup、Sort、Merge Join 和
btree scan 相关课程，知道 planner 会把 operator、opfamily、collation 和 null ordering 写进计划节点。

本节唯一主问题：
```text
collation、sort support 和 comparison function 如何影响 sort、merge join、btree scan 和 hash equality？
```
核心矛盾：SQL 希望“相等、排序、索引查找、hash 分组”给出一致语义；内核 hot path 又不能每比较一次都重新查 catalog、重新解释 operator、总是走慢速 locale API。PostgreSQL 因此把同一个比较语义拆成 collation OID、opclass support function、sort support callback、btree comparator、equality operator 和 hash support 多条路径，并要求它们在边界上保持相容。
学完后应能判断：一个慢排序、错误的 merge join 假设、btree 索引结果异常、hash 聚合/连接不符合预期，究竟更可能来自 collation 选择、opclass support、abbreviated key fallback、comparison function、nondeterministic collation，还是 hash equality 与 ordering equality 的边界误读。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置
04 目录已经从 executor 生命周期、slot、表达式、FMGR 和 operator lookup 走到具体执行节点。
第 20 节解释 Merge Join 如何消费有序流。
第 22 节解释 Sort 为什么是阻塞节点，以及 tuplesort 如何在 `work_mem` 和临时文件之间切换。
第 65 到 67 节解释 Datum、varlena、FMGR 和 operator/support function 如何进入执行期。
本节把这些线连起来：一个 `ORDER BY text_col COLLATE ...`、一个 merge join clause、一个 btree index scan、一个 hash aggregate 的 group key，看上去都在“比较字符串”，但它们不是同一条调用链。
本节只回答一个问题：collation、sort support 和 comparison function 如何影响 sort、merge join、btree scan 和 hash equality。
不展开 planner pathkey 推导，不展开 CREATE OPERATOR CLASS 语法，不展开所有数据类型的比较实现；这些内容只在影响本节主问题时出现。
后续第 71 节可以继续看 JIT 如何接管表达式和 deform；第 72 节可以继续看表达式 ERROR 时的 cleanup。

## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
planner 把每个比较位置需要的 operator/opfamily/collation/null ordering 写进 Plan；
executor 初始化时把这些元数据编译成 SortSupportData、FmgrInfo 或 scan key；
运行时 sort/merge/btree/hash 分别调用自己的 hot path；
这些 hot path 必须共享同一组语义边界：collation OID、opclass contract、NULL 规则、deterministic equality 和 provider version。
```
这里最容易误解的是“比较函数”这个词。

在 PostgreSQL 内核里，它至少可能指五件事：

| 名称 | 典型入口 | 返回语义 | 本节关注点 |
| --- | --- | --- | --- |
| SQL equality operator | `texteq()`、`=` operator | boolean | hash equality、qual、join qual 使用它或它的编译表达式。 |
| btree comparison support | `BTORDER_PROC`、`bttextcmp()` | int32: `<0/0/>0` | btree opclass ordering contract，old-style comparator。 |
| sort support callback | `BTSORTSUPPORT_PROC`、`bttextsortsupport()` | 初始化 `SortSupportData` | 给 sort/merge/gather merge/index build 提供更快 comparator。 |
| sort comparator pointer | `SortSupportData.comparator` | int: `<0/0/>0` | hot path 直接调用，避免每次走 FMGR。 |
| hash support function | `hashtext()` / extended hash proc | hash code | hash join/agg/grouping 的 bucket 分配，不提供全序。 |

本节的 tension 可以压缩成：
```text
一套 SQL 比较语义必须跨 sort、merge、btree、hash 相容
  vs
每条执行路径为了 CPU、cache 和 I/O 成本都需要不同的低层表示和 fallback。
```
所以课程不会把 `varlena.c` 写成字符串函数清单。

我们沿同一个问题看四条 runtime 链路：
```text
Sort / GatherMerge / SetOp
  -> SortSupportData + ApplySortComparator()

Merge Join
  -> MergeJoinClause.ssup + ApplySortComparator()

Btree scan / index build
  -> ScanKey + FunctionCall2Coll(BTORDER_PROC or strategy proc)
  -> nbtsort uses SortSupportData

Hash equality
  -> hash function decides bucket
  -> equality operator with collation confirms match
```
这四条链路不是互相替代。
它们是同一语义在不同性能边界上的实现。

## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/catalog/pg_collation.h` | `pg_collation` 字段：provider、deterministic、encoding、locale/rules、version。 |
| 2 | `src/include/utils/sortsupport.h` | `SortSupportData`、`ApplySortComparator()`、abbreviated key contract。 |
| 3 | `src/backend/utils/sort/sortsupport.c` | `PrepareSortSupportFromOrderingOp()`、`PrepareSortSupportFromIndexRel()`、shim fallback。 |
| 4 | `src/backend/utils/adt/varlena.c` | `varstr_cmp()`、`texteq()`、`bttextcmp()`、`bttextsortsupport()`、abbreviation。 |
| 5 | `src/backend/utils/adt/pg_locale.c` | collation OID 到 provider locale 的 cache、version check、`pg_strcoll()` / `pg_strxfrm()`。 |
| 6 | `src/backend/utils/sort/tuplesort.c` | abbreviation abort、sort 状态切换、trace_sort 观测。 |
| 7 | `src/backend/utils/sort/tuplesortvariants.c` | heap/datum/index sort 如何准备 sort keys 并比较 tuple。 |
| 8 | `src/backend/executor/nodeSort.c` | Sort plan 的 `sortOperators/collations/nullsFirst` 如何进入 tuplesort。 |
| 9 | `src/backend/executor/nodeMergejoin.c` | merge clause 如何构造 `SortSupportData`，为什么禁用 abbreviation。 |
| 10 | `src/backend/executor/nodeGatherMerge.c` / `nodeMergeAppend.c` | 多路有序流合并如何复用 sort support。 |
| 11 | `src/backend/access/nbtree/nbtcompare.c` | btree 内置比较函数，特别是非 collation 类型的 baseline。 |
| 12 | `src/backend/access/nbtree/nbtsort.c` | btree index build sort 如何使用 `ApplySortComparator()`。 |
| 13 | `src/backend/access/nbtree/nbtsearch.c` / `nbtreadpage.c` | btree scan key 如何用 collation 调用比较/strategy function。 |
| 14 | `src/backend/utils/cache/typcache.c` | 类型 cache 如何缓存 equality、hash、btree opfamily 信息。 |
| 15 | `src/backend/executor/execGrouping.c` | hash table 如何保存 collations、hash functions 和 equality functions。 |
| 16 | `src/backend/executor/execExprInterp.c` | hash aggregate / DISTINCT 等路径如何调用 equality function。 |
| 17 | `src/include/access/nbtree.h` | `BTORDER_PROC`、`BTSORTSUPPORT_PROC` support number contract。 |

阅读顺序按语义流展开，不按文件名排序。
先看 collation 是什么状态。
再看 sort support 如何把 catalog 元数据变成 hot path 函数指针。
然后看文本比较如何依据 collation 选择 `memcmp()`、`strcoll()`、`strxfrm()` 或 ICU/libc 路径。
最后对照 sort、merge join、btree scan、hash equality 四条调用链。
不要从 `varlena.c` 顶部线性读到尾。
这会把核心矛盾淹没在字符串函数里。

## 4. 关键数据结构与状态
本节的状态大多是 backend-local cache 和 plan/executor-local 状态，不是 shared memory。
但它们依赖 system catalog，并且受 syscache/typcache/relcache invalidation 影响。

### `pg_collation`
`pg_collation` 不是一个简单名字表。
它定义比较语义中最危险的一部分：同一串 bytes 在哪个 provider 和哪组 locale/rules 下排序、相等和转换。

关键字段：

| 字段 | 语义 |
| --- | --- |
| `oid` | executor、FMGR、scan key、sort support 传递的 collation identity。 |
| `collprovider` | `builtin`、`icu`、`libc` 或 default provider。 |
| `collisdeterministic` | 是否允许 provider 层面把 byte 不同的字符串视为相等。 |
| `collencoding` | collation 适用编码，`-1` 表示 all encoding。 |
| `collcollate` / `collctype` | libc 传统 locale 字段。 |
| `colllocale` / `collicurules` | ICU/builtin 新路径使用的 locale/rules。 |
| `collversion` | provider-dependent version，用于发现系统 collation 数据变化。 |

raw OID 不是语义。
`collation OID + provider + deterministic + encoding + actual provider library version` 才是比较语义。

### `pg_locale_t`
`pg_newlocale_from_collation(collid)` 把 `pg_collation` row 解析成进程内 locale 对象。

这个对象携带：
```text
collate_is_c
deterministic
provider-specific callbacks
strcoll / strxfrm ability
```
它是 backend-local cache。
多个 backend 不共享这个指针。
源码在 `pg_locale.c` 中还会检查 catalog 中记录的 `collversion` 和 provider 当前实际版本。
版本不匹配通常不是让当前比较立刻失败，而是发出 warning，让使用者重建依赖对象。

### `SortSupportData`
`SortSupportData` 是本节的核心执行期状态。
它不是 catalog row，也不是 SQL 函数调用信息。
它是把“某个 key 怎么排序”编译成函数指针和少量私有状态后的结果。

关键字段：

| 字段 | 语义 |
| --- | --- |
| `ssup_cxt` | sort support 私有状态的内存上下文。 |
| `ssup_collation` | comparator 使用的 collation OID。 |
| `ssup_reverse` | 是否反向排序。 |
| `ssup_nulls_first` | NULL 排在前还是后。 |
| `ssup_attno` | index sort 等路径中对应 index attribute。 |
| `ssup_extra` | opclass 私有状态，例如 varlena sort scratch buffer。 |
| `comparator` | hot path 3-way comparator。 |
| `abbreviate` | caller 告诉 opclass 当前 key 是否原则上可 abbreviation。 |
| `abbrev_converter` | 原始 Datum 到 abbreviated key 的转换函数。 |
| `abbrev_abort` | 根据早期样本决定是否放弃 abbreviation。 |
| `abbrev_full_comparator` | abbreviated key tie-break 的权威 comparator。 |

字段组合才是语义。
`comparator` 单独看不够。
同一个 comparator 指针在不同 `ssup_collation`、`ssup_reverse`、`ssup_nulls_first` 下给上层的排序行为不同。

### `SortShimExtra`
`sortsupport.c` 中的 `SortShimExtra` 是 fallback 状态。
当 opclass 没有提供 `BTSORTSUPPORT_PROC`，或者 sort support 函数没有填 `ssup->comparator` 时，系统用 `BTORDER_PROC` 包一层 shim。
它缓存 `FmgrInfo` 和可复用 `FunctionCallInfoBaseData`。
这比每次重新构造 FMGR 调用便宜，但仍然比原生 comparator 指针慢。

### `VarStringSortSupport`
`varlena.c` 的 `VarStringSortSupport` 是 text/varchar/bpchar/name sort support 私有状态。

它记录：
```text
scratch buffer
last comparison inputs
cached strcoll result
strxfrm blob cache
HyperLogLog cardinality estimate
typid and collation flags
```
它服务两个目标：

1. 避免每次字符串比较都重新分配临时 buffer。
2. 让 abbreviation 在收益不足时尽早 abort。

这个状态挂在 `ssup_extra`，生命周期由 `ssup_cxt` 管。

### `MergeJoinClause.ssup`
`nodeMergejoin.c` 给每个 merge clause 准备一份 `SortSupportData`。
与 Sort 不同，Merge Join 把 `abbreviate` 设为 false。
原因很直接：merge join 每次比较的是当前 outer/inner key，没有一个方便阶段把输入流全部转换成 abbreviated representation。
所以 Merge Join 可以使用 sort support 提供的快速 comparator，但不能使用 tuplesort 那种 abbreviated key 排序状态。

### `ScanKey.sk_collation`
btree scan key 不是只携带 operator。
它也携带 `sk_collation`。
btree scan 在 `_bt_compare()`、page read、array key preprocessing 等路径中会通过 `FunctionCall2Coll()` 把这个 collation 传给 strategy function 或 order proc。
这解释了为什么同一个 btree opclass、同一个 text operator，在不同 collation index 上不能随便互换。

### hash table 中的 collation / equality / hash arrays
`execGrouping.c` 的 tuple hash table 保存：
```text
eqfunctions
hashfunctions
collations
keyColIdx
```
hash path 先用 hash function 把 key 放入 bucket，再用 equality function 确认是否同组。
collation 对 hash function 和 equality function 都重要；`hashtext()` deterministic 时 hash bytes，nondeterministic 时先用 `pg_strnxfrm()` 生成 collation key。
但 hash function 本身不能提供全序，也不能替代 btree comparator。

## 5. 主流程源码 walkthrough
本节主流程按同一个 SQL 语义在四个执行边界的传播展开。

### 5.1 planner/executor 交接：collation 不是运行时猜出来的
计划节点里已经带有 collation。

典型字段：
```text
Sort.collations[]
MergeJoin.mergeCollations[]
GatherMerge.collations[]
IndexScan index quals / scankeys
Agg/Hash grouping key collations
```
executor 初始化时不会重新推导“这条表达式应该用哪个 collation”。
它消费 planner/analyzer 已经解析好的 OID。

这个边界很重要：
```text
SQL expression collation resolution
  -> planned metadata
  -> executor-local callable state
```
如果 collation OID 已经错了，后面 sort support 再快也只是在错误语义上加速。

### 5.2 Sort：从 `nodeSort.c` 进入 tuplesort
`ExecSort()` 首次执行时根据计划字段创建 tuplesort。

核心形状：
```text
ExecSort()
  -> tuplesort_begin_heap(... sortOperators, collations, nullsFirst ...)
  -> tuplesort_puttupleslot()
  -> tuplesort_performsort()
  -> tuplesort_gettupleslot()
```
单列 datum sort 走：
```text
tuplesort_begin_datum(type, sortOperator, collation, nullsFirst, ...)
```
多列 heap tuple sort 走：
```text
tuplesort_begin_heap(tupDesc, nkeys, sortColIdx,
                     sortOperators, collations, nullsFirst, ...)
```

`tuplesortvariants.c` 为每个 sort key 准备 `SortSupportData`：
```text
sortKey->ssup_cxt = CurrentMemoryContext
sortKey->ssup_collation = collations[i]
sortKey->ssup_nulls_first = nullsFirst[i]
sortKey->abbreviate = (i == 0 && allowed)
PrepareSortSupportFromOrderingOp(sortOperators[i], sortKey)
```
后续 comparator hot path 不是反复查 `pg_operator`。

它直接调用：
```text
ApplySortComparator(datum1, isnull1, datum2, isnull2, sortKey)
```

`ApplySortComparator()` 先处理 NULL，再调用 `ssup->comparator`，最后根据 `ssup_reverse` 反转结果。
这就是 `ORDER BY a DESC NULLS FIRST COLLATE "x"` 能被压进一个函数指针调用边界的原因。

### 5.3 sort support 查找：优先 `BTSORTSUPPORT_PROC`，否则 shim
`PrepareSortSupportFromOrderingOp()` 的工作是从 ordering operator 找到 opfamily 和 opcintype。

然后进入 `FinishSortSupportFunction()`：
```text
get_opfamily_proc(opfamily, opcintype, opcintype, BTSORTSUPPORT_PROC)
  -> OidFunctionCall1(sortSupportFunction, PointerGetDatum(ssup))
  -> 如果 comparator 仍为空：
       get_opfamily_proc(..., BTORDER_PROC)
       PrepareSortSupportComparisonShim()
```
这说明 sort support 是加速协议，不是 correctness 的唯一来源。
如果 opclass 没有 sort support，`BTORDER_PROC` 仍然可以保证排序语义。
代价是每次比较会走 shim 中的 FMGR 调用。
`PrepareSortSupportComparisonShim()` 会把 `ssup_collation` 写进 reusable `FunctionCallInfo`，之后 `comparison_shim()` 用 `FunctionCallInvoke()` 调 old-style comparator。
这条 fallback 对扩展 opclass 特别重要：没写 sort support 的 opclass 仍然能用 Sort、Merge Join、btree index build，只是慢。

### 5.4 text 的权威比较：`varstr_cmp()`
text 的 old-style btree comparator 是 `bttextcmp()`。

它最终调用：
```text
text_cmp(arg1, arg2, PG_GET_COLLATION())
  -> varstr_cmp(bytes1, len1, bytes2, len2, collid)
```

`varstr_cmp()` 的关键分支：
```text
if C collation:
  memcmp(bytes)
else:
  if exact same bytes:
    return 0
  result = pg_strncoll(... locale ...)
  if result == 0 and deterministic:
    tie-break by bytewise memcmp/length
```
最后一行是 deterministic collation 的关键。
deterministic collation 下，两个 byte 不同但 locale 认为等价的字符串，需要用 bytewise tie-break 得到稳定全序。
nondeterministic collation 下，`pg_strncoll()` 返回 0 就可以保持 0。

这直接影响：
```text
ORDER BY 的稳定 ordering boundary
btree equal key boundary
dedup/equalimage 能否安全
hash equality 是否要走 collation-aware comparison
```

### 5.5 text equality：deterministic 走 bytewise，nondeterministic 走 collation
`texteq()` 和 `textne()` 不是简单调用 `text_cmp()`。

对 deterministic collation：
```text
toast_raw_datum_size()
length 不同直接 false
length 相同再 detoast + memcmp()
```
对 nondeterministic collation：
```text
text_cmp(arg1, arg2, collid) == 0
```
这是本节最重要的 hash equality 边界之一。
deterministic collation 下，text equality 可以是 byte equality。
nondeterministic collation 下，equality 必须尊重 collation 语义，例如大小写、重音或 ICU rules 造成的等价。
因此不能把 text hash path 理解成“hash bytes 就完事”。
hash table 还必须用 equality function 确认 match，并且 hash support 必须和 equality operator 保持相容。

### 5.6 text sort support：`bttextsortsupport()` 到 `varstr_sortsupport()`
text opclass 的 sort support 函数是 `bttextsortsupport()`。

它拿到 `SortSupportData *` 后进入：
```text
varstr_sortsupport(ssup, TEXTOID, collid)
```

`varstr_sortsupport()` 先拿 locale：
```text
check_collation_set(collid)
locale = pg_newlocale_from_collation(collid)
```
然后按 locale 类型选择 comparator：
```text
C collation:
  varstrfastcmp_c / bpcharfastcmp_c / namefastcmp_c

non-C collation:
  varlenafastcmp_locale / namefastcmp_locale
```
C collation 的 comparator 用 `memcmp()`。
locale comparator 用 `pg_strcoll()`，并在 deterministic collation 下用 bytewise tie-break。
这样 Sort hot path 不需要每次通过 `bttextcmp()` 走 FMGR。

### 5.7 abbreviated key：把昂贵字符串比较改成 cheap key 比较
tuplesort 允许第一排序 key 使用 abbreviated key。

约束很强：
```text
abbreviated comparison 非 0 结果必须和权威 comparator 同方向
abbreviated comparison 返回 0 不表示相等，只表示无法判定
需要 abbrev_full_comparator 做 tie-break
需要 abbrev_abort 判断收益不足时回退
```
text abbreviation 的大致流程：
```text
varstr_abbrev_convert(original)
  -> C collation: copy first bytes into Datum
  -> non-C collation: pg_strxfrm()/ICU analog -> first bytes into Datum
  -> update HyperLogLog cardinality estimates

ssup->comparator = ssup_datum_unsigned_cmp
ssup->abbrev_full_comparator = original comparator
```

`tuplesort.c` 会在输入早期调用 `abbrev_abort()`。
如果发现 abbreviated key 区分度太差，就把 `abbrev_converter` 清空，并把 `comparator` 切回权威 comparator。

这条路径解释了一个常见现象：
```text
同样是 ORDER BY text，C collation、高基数短字符串可能很快；
复杂 ICU/libc collation、低区分度长前缀字符串可能触发 abbreviation abort 或大量 full comparator tie-break。
```

### 5.8 Merge Join：复用 sort support，但禁用 abbreviation
`ExecInitMergeJoin()` 初始化 merge clauses 时会做：
```text
clause->ssup.ssup_cxt = CurrentMemoryContext
clause->ssup.ssup_collation = collation
clause->ssup.ssup_reverse = reversed
clause->ssup.ssup_nulls_first = nulls_first
clause->ssup.abbreviate = false
```
然后它尝试从 opfamily 拿 `BTSORTSUPPORT_PROC`。
如果 sort support 没有提供 comparator，就 fallback 到 `BTORDER_PROC` shim。

运行时 `MJCompare()` 对每个 merge clause 调：
```text
ApplySortComparator(clause->ldatum, clause->lisnull,
                    clause->rdatum, clause->risnull,
                    &clause->ssup)
```
Merge Join 的正确性依赖一个事实：outer 和 inner 输入已经按同一个 opfamily + collation + null ordering 排好序。
如果两边排序语义不同，merge join 的“当前 key 小的一侧前进”假设就不成立。
planner pathkey 与 equivalence class 会负责推导这种相容性；executor 只执行计划。

### 5.9 Gather Merge / Merge Append：多路合并也是 sort comparator
`nodeGatherMerge.c` 和 `nodeMergeAppend.c` 也为每个排序 key 构建 `SortSupportData`。
它们的运行时不是重新排序全部输入，而是在多个已经有序的子流之间做堆/选择。

比较仍然走：
```text
ApplySortComparator()
```
这说明 sort support 不只服务 Sort 节点。
它是“执行期有序流比较”的公共低成本表示。

### 5.10 Btree index build：nbtsort 走 SortSupport
btree index build 需要把 index tuples 排成 opclass 定义的顺序。

`nbtsort.c` 的比较路径会使用 `SortSupportData`，并在多列 key 上逐列调用：
```text
ApplySortComparator(attrDatum1, isNull1,
                    attrDatum2, isNull2,
                    sortKey)
```
这条路径和普通 query Sort 很像，但 caller 不同。
普通 Sort 产出 executor slot。
btree build 产出 index tuple order。
两者必须用同一 opclass/collation 语义，否则 index scan 后续用 scan key 查找时会破坏 ordering invariant。

### 5.11 Btree scan：scan key 调比较函数，不是 sort support 的 abbreviated key
btree scan 的核心不是把所有 key 排序，而是在 page/search boundary 判断：
```text
当前 index key 与 scankey 是否满足策略
该页后续 tuple 是否还可能满足
array key 的 high/low 边界如何比较
```
源码中大量路径通过：
```text
FunctionCall2Coll(&scankey->sk_func,
                  scankey->sk_collation,
                  ...)
```
或者通过 opfamily 的 `BTORDER_PROC` 做三路比较。
这里 `sk_collation` 是关键。
btree index 的 ordering 是写入 index metadata 的 opclass + collation 共同定义的。
scan key 必须按同样 collation 解释查询常量。
如果把 collation 当作显示层格式，就无法解释为什么 changing collation version 后需要重建索引。

### 5.12 Hash equality：hash function 只负责候选集合，equality 决定语义
hash join、hash aggregate、hashed SubPlan、tuple hash table 的基本模型：
```text
hashfunctions(values, collations)
  -> bucket
eqfunctions(values, collations)
  -> confirm equal
```

`execGrouping.c` 的 `BuildTupleHashTable()` 保存 `eqFunctions`、`hashFunctions` 和 `collations`。
`ExecBuildHash32FromAttrs()` 初始化 hash function call info 时把对应 collation 写成 `inputcollid`。
运行时先算 hash code，再在 bucket 内调用 equality 表达式。
对 deterministic text collation，`hashtext()` hash 原始 bytes，`texteq()` 常走 size + memcmp 快路径。
对 nondeterministic collation，`hashtext()` 用 `pg_strnxfrm()` 后的 collation key 参与 hash，`texteq()` 走 `text_cmp()` 确认 collation-aware equality。

这就是 hash equality 与 btree comparison 的边界：
```text
hash 不需要全序；
hash equality 必须和 SQL equality 相容；
如果 equality 在 nondeterministic collation 下更宽，hash support 必须用相容的 collation key，不能把 equal 值分到永远不会相遇的 bucket。
```
不能用“btree compare 返回 0”简单替代 hash equality。
也不能用“hash code 一样”推断 SQL equal。

### 5.13 Ordered-set aggregate：排序语义可被 aggregate 私有状态携带
`orderedsetaggs.c` 不是本节主链路，但它是一个好对照。
ordered-set aggregate 内部也会为 `ORDER BY` 输入构造 sort state。
它说明 sort support 不是只存在于 `Sort` plan node。
只要某个 executor 子系统需要 SQL ordering，就可能把 operator + collation 编译成 `SortSupportData` 或 tuplesort state。

## 6. 生命周期 / ownership / cleanup
### 谁创建 collation runtime 状态
collation OID 来自 parse/analyze/planner 阶段的表达式和计划节点。
运行期真正的 locale 对象由 `pg_newlocale_from_collation()` 按需创建或从 backend-local cache 取出。
owner 是当前 backend。
其他 backend 不能共享该指针。

### 谁持有 `SortSupportData`
不同调用者持有不同生命周期：

| 调用者 | owner | 生命周期 |
| --- | --- | --- |
| `tuplesort_begin_heap/datum` | tuplesort state | 从 begin 到 `tuplesort_end()`。 |
| `ExecInitMergeJoin()` | `MergeJoinState` | 从 node init 到 `ExecEndMergeJoin()`。 |
| `nodeGatherMerge` / `nodeMergeAppend` | plan node state | 从 node init 到 node end。 |
| `nbtsort.c` | index build local state | index build sort 生命周期。 |
| statistics / analyze | analyze local context | stats build 生命周期。 |

`ssup_cxt` 决定 `ssup_extra` 的内存归属。
sort support callback 必须把私有状态分配在 `ssup_cxt`，否则 sort state cleanup 可能漏掉或过早释放。

### 谁释放
`SortSupportData` 本体通常嵌在更大的 state 中，不单独 free。
`ssup_extra` 和 scratch buffer 由对应 MemoryContext reset/delete 释放。
tuplesort 自己还会释放临时文件、memtuples、tapes 等状态。
executor 节点的 state 随 `ExecEndNode()` 和 executor context 释放。
index build 路径随 index build context 释放。

### ERROR / abort 时谁兜底
本节大多数状态是 MemoryContext 管理。
如果 comparator、locale provider、detoast、临时文件或 OOM 抛 ERROR，普通 executor query 会通过 `ExecutorEnd()`、ResourceOwner、MemoryContext reset 和临时文件 cleanup 收尾。
tuplesort 涉及临时文件时，文件资源不只靠 `pfree()`。
它依赖 buffile/logtape/ResourceOwner 等路径在 ERROR 中释放。
sort support 私有 scratch buffer 则由 memory context 释放。

### 长期对象如何失效
operator、opclass、function、typcache、relcache 可能被 DDL 改变，相关 backend-local cache 通过 syscache/typcache/relcache invalidation 失效。
collation OID 本身来自计划；`pg_locale_t` cache 是 backend-local，并按需创建。
已经在执行中的 plan node 拿到的函数指针和 collation OID，不会在比较中途重新规划；plan cache 失效通常影响下一次使用 prepared statement 或 cached plan。
外部 ICU/libc 版本变化不是普通 sinval 消息，主要靠 `collversion` 检查暴露风险。

### collation version 不是普通 invalidation
系统库的 collation 数据变化，例如 libc/ICU 升级，未必对应 PostgreSQL catalog row 变化。
`pg_locale.c` 会在加载 collation 时比较 catalog 中记录的 `collversion` 与 provider 当前版本。
不匹配会 warning。
正确处理通常是刷新 collation version 并重建依赖索引。
这不是一个普通事务内 ERROR cleanup 问题，而是外部 provider 数据改变后的持久一致性问题。

## 7. 正确性机制层次
### 层次一：collation resolution
analyzer/planner 决定表达式输入 collation。
executor 不重新推导。
正确性要求是：同一个比较位置必须传入正确 collation OID。

### 层次二：opclass contract
btree opclass 要保证 ordering operator、`BTORDER_PROC`、`BTSORTSUPPORT_PROC` 语义相容。
hash opclass 要保证 hash function 和 equality operator 相容。
跨类型比较还要保证 left/right type、opfamily 和 support function number 匹配。

### 层次三：NULL 规则
SQL operator 通常 strict，不比较 NULL。
排序需要定义 NULL 的位置。

`ApplySortComparator()` 把 NULL 处理放在 comparator 外面：
```text
NULL vs NULL -> 0
NULL vs NOT NULL -> 由 ssup_nulls_first 决定
NOT NULL vs NOT NULL -> 调 comparator
```
Merge Join 还要额外处理 NULL 不匹配的问题。
`MJCompare()` 中 NULL-vs-NULL 不应让 join 误以为普通 equality match。

### 层次四：deterministic collation
deterministic collation 下，ordering equality 最终要用 bytewise tie-break 保证全序。
equality operator 对 text 可以用 bytewise fast path。
nondeterministic collation 下，locale provider 可以让 byte 不同的字符串比较为 equal。

这会影响：
```text
btree dedup/equalimage
hash equality
substring search support
index rebuild requirement
排序结果的 tie 行为
```

### 层次五：abbreviated key 不能改变语义
abbreviation 是性能优化。
它的非 0 结果必须可靠。
0 只代表“不确定”，需要 full comparator tie-break。
如果成本模型判断收益不足，tuplesort 可以 abort abbreviation。
正确性不依赖 abbreviation 是否继续。

### 层次六：provider version
libc/ICU/builtin collation provider 的排序规则可能随系统升级变化。
PostgreSQL 记录 `collversion`，加载时检查实际版本。
这层机制不能自动重排已有 btree index。
它只能让用户看到风险并执行 rebuild。

### 不涉及的正确性机制
本节基本不依赖 MVCC visibility、WAL redo、buffer pin 或 heavyweight lock 来保证比较语义。
这些机制保证 tuple 和 index page 的存储/并发正确性。
但“两个 Datum 在这个 collation 下谁小谁大”主要由 collation、opclass、FMGR 和 cache invalidation 保证。

## 8. 错误路径 / 异常路径 / fallback
### 8.1 collation 未设置
`varstr_cmp()` 和 `varstr_sortsupport()` 都会调用 `check_collation_set(collid)`。
如果一个 collation-sensitive 操作拿到 `InvalidOid`，会报错。
这通常表示 planner/analyzer 或调用者没有正确传递 collation。
不要在 comparator 里用 database default collation 静默补救。
静默补救会让同一 SQL 在不同上下文下改变语义。

### 8.2 没有 sort support
正常 fallback：
```text
BTSORTSUPPORT_PROC missing
  -> BTORDER_PROC
  -> PrepareSortSupportComparisonShim()
```
如果连 `BTORDER_PROC` 都找不到，说明 opfamily contract 不完整，源码会 ERROR。
这类错误更像 catalog/opclass 定义错误，而不是 executor runtime 偶发错误。

### 8.3 sort support 不设置 comparator
有些 sort support 函数可能根据 collation 或平台选择不启用特殊 comparator。
`FinishSortSupportFunction()` 会检查 `ssup->comparator == NULL` 并 fallback 到 shim。
这允许 opclass 写保守实现：能加速时加速，不能安全加速时退回权威 comparator。

### 8.4 abbreviation abort
tuplesort 早期输入阶段可能发现 abbreviated key 区分度太差。

此时它会：
```text
清空 abbrev_converter
清空 abbrev_abort
把 comparator 切回 full comparator
必要时把已有 memtuple 的 abbreviated datum 还原或按原始 datum 重比
```
用户侧通常只看到排序时间变化。
开启 `trace_sort` 才可能在日志中看到 abbreviation abort 相关信息。

### 8.5 `strxfrm()` 不安全或 provider 不支持
`varstr_sortsupport()` 对非 C collation 会检查 `pg_strxfrm_enabled(locale)`。
如果平台的 `strxfrm()` 或 provider transform 不能安全用于 abbreviation，就禁用 abbreviation。
这不是功能缺失。
这是用性能让步换 correctness。

### 8.6 nondeterministic collation 禁用部分优化
`btvarstrequalimage()` 对 nondeterministic collation 返回 false。
含义是：不能假设相等 key 的二进制 image 完全可替代。
btree dedup 等依赖 equal image 的优化必须保守。
`text_starts_with()` 这类 substring search 在 nondeterministic collation 下会 ERROR。

### 8.7 collation version mismatch
加载 collation 时发现 `collversion` 与 provider actual version 不一致，会 warning。
这时 query 仍可能执行。
但已有 index 的物理顺序可能和当前 provider ordering 不一致。
正确动作通常不是调大 work_mem，也不是重跑 ANALYZE，而是确认依赖对象并重建。

### 8.8 comparator 返回 NULL
`comparison_shim()` 调 old-style comparator 后会检查 `fcinfo.isnull`。
如果 comparison function 返回 NULL，会 ERROR。
sort comparator 的契约不允许 NULL 结果。
NULL 输入由 `ApplySortComparator()` 外层处理。

### 8.9 detoast / OOM / temp file ERROR
text comparator 可能 detoast。
sort 可能 spill 到临时文件。
这些路径 ERROR 时由 executor memory context、ResourceOwner 和临时文件 cleanup 兜底。
这说明比较 hot path 虽然看起来是纯 CPU，但实际会触发内存分配、detoast 和 I/O。

## 9. 成本、资源与跨模块传播
### CPU 成本：locale 比较远比 integer compare 贵
`memcmp()`、integer comparator、`strcoll()`、ICU collation compare 的成本不是一个量级。
对 `ORDER BY text COLLATE "C"`，C collation 常能走 bytewise comparator。
对复杂 libc/ICU collation，比较可能进入 provider callback、转换 buffer、宽字符或 ICU state。
排序的比较次数大约随 `N log N` 增长。
每次 comparator 变贵都会被放大。

### cache 成本：abbreviated key 是 cache 优化
sort support 注释明确强调 CPU cache miss 成本。
abbreviated key 把 pass-by-reference 字符串比较变成 pass-by-value Datum 比较。

收益来自：
```text
减少 detoast/varlena pointer chasing
减少 provider collation 调用
提升 memtuples 中比较 key 的 cache locality
减少 full comparator tie-break
```
但如果 abbreviated key 区分度低，收益会消失。

### memory 成本：sort support 私有 buffer 与 tuplesort memtuples
`VarStringSortSupport` 会持有 scratch buffer。
tuplesort 还持有 memtuples、abbreviated datum、tape state。
当输入超过 `work_mem`，排序写临时文件。
collation 本身不决定是否 spill，但昂贵 comparator 会让同样 spill 量的排序 CPU 更重。

### I/O 成本：排序 spill 和 index build
Sort 节点 spill 观测在 EXPLAIN 的 Sort Method、Disk 和 temp read/write 上。
btree index build 也要排序 index tuples，可能受 `maintenance_work_mem` 和 temp file 影响。
如果 collation 比较慢，index build 的瓶颈可能不是 I/O，而是 CPU comparator。

### 跨模块传播一：parser/analyzer 到 plan
collation resolution 发生在执行前。
表达式节点的 `inputcollid`、plan node 的 `collations[]`、merge clause 的 `mergeCollations` 都来自前期阶段。
executor 只消费这些 OID。

### 跨模块传播二：catalog cache 到 executable function
operator/opclass/support function 查找通过 syscache/typcache/lsyscache。
执行期把 OID 转成 `FmgrInfo` 或 comparator pointer。
DDL invalidation 会影响后续 plan/cache 使用，但不会让正在比较的单个 comparator 调用中途换语义。

### 跨模块传播三：btree index 与 collation provider
btree index 物理顺序依赖 build 时的 collation provider behavior。
系统升级 ICU/libc 后，即使 PostgreSQL catalog 没变，provider ordering 也可能变。
这会从操作系统包升级传播到 SQL 查询正确性。

### 跨模块传播四：hash aggregate / join 与 equality
hash path 不关心全序，但关心 equality/hash 相容。
nondeterministic collation 会让 text equality 走 collation-aware path。
这会把 collation 成本传播到 hash table bucket 内比较。

### 跨模块传播五：parallel / Gather Merge
每个 worker 可能先本地排序。
leader 的 Gather Merge 再按同一 sort key 合并。
如果 worker sort 和 leader merge 使用不同 collation 或 null ordering，结果会破坏全局有序性。
计划节点传递同一组 `collations[]` 是这个边界的运行时保障。

## 10. 观测与诊断入口
本节锚定的 runtime truth：
```text
同一条 SQL 中，比较语义由 collation/opclass 定义；
执行成本由具体 hot path 决定；
EXPLAIN 只能看到节点和 spill，不能直接告诉你 comparator 走了哪条分支。
```

### SQL 入口：确认 collation 与 provider
先看 catalog：
```sql
SELECT oid, collname, collprovider, collisdeterministic,
       collencoding, collcollate, collctype, colllocale, collicurules,
       collversion
FROM pg_collation
WHERE collname IN ('C', 'C.UTF-8', 'und-x-icu');
```
不同系统上的 collation 名称不完全一致。
ICU 是否可用也取决于 build。

### EXPLAIN 入口：看 Sort Method 和 temp I/O
```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT *
FROM t
ORDER BY txt COLLATE "C";
```
再对照：
```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT *
FROM t
ORDER BY txt COLLATE "und-x-icu";
```
能看到：
```text
Sort Method
Memory / Disk
temp read/write
startup time / total time
rows
```
看不到：
```text
每次 comparator 调用次数
是否使用 varstrfastcmp_c
abbreviation 是否 abort
strcoll/strxfrm 调用次数
```
这些要靠 `trace_sort`、perf、gdb 或临时插桩。

### `trace_sort`
开启：
```sql
SET trace_sort = on;
SET client_min_messages = log;
```
然后执行排序 SQL。
日志可以看到 tuplesort begin、switch to external sort、merge pass、abbreviation 相关信息。
注意它是 backend 日志，不是 EXPLAIN 字段。

### perf / flamegraph
如果排序 CPU 很高但没有明显 I/O wait，采样栈通常更有用。

你可能看到：
```text
varstrfastcmp_c
varlenafastcmp_locale
pg_strcoll
ucol_strcollUTF8
strcoll_l
comparison_shim
FunctionCallInvoke
```
这些符号能把问题归因到：
```text
C collation fast path
locale comparator
ICU/libc provider
old-style comparator shim
extension comparator
```

### gdb 断点
适合源码学习的断点：
```text
PrepareSortSupportFromOrderingOp
FinishSortSupportFunction
bttextsortsupport
varstr_sortsupport
varstr_abbrev_abort
MJCompare
ApplySortComparator
texteq
bttextcmp
pg_newlocale_from_collation
```
不要在大表排序上无条件断 `ApplySortComparator()`。
比较次数太多，会让调试不可用。
更好的做法是在小数据集上断一次初始化路径，确认 `ssup_collation`、`comparator`、`abbrev_converter`。

### 系统视图能看到什么
| 入口 | 能看到 | 看不到 |
| --- | --- | --- |
| `pg_collation` | catalog 中的 provider/deterministic/version | provider 当前算法细节。 |
| `EXPLAIN ANALYZE` | sort spill、节点耗时、rows | comparator 分支和调用次数。 |
| `pg_stat_statements` | 多次执行累计时间 | 单次 sort comparator 细节。 |
| `pg_stat_io` / temp stats | 临时文件 I/O | CPU collation 成本。 |
| `pg_stat_activity` wait event | 是否在 I/O/lock/client wait | CPU active 具体函数。 |
| `perf` | CPU 热点函数 | SQL-level collation intent。 |
| `trace_sort` | tuplesort 阶段和部分 abbreviation 信息 | btree scan equality 细节。 |

### 版本诊断
检查 collation version mismatch：
```sql
SELECT pg_collation_actual_version(oid), collversion, collname
FROM pg_collation
WHERE collversion IS NOT NULL
  AND pg_collation_actual_version(oid) IS DISTINCT FROM collversion;
```
这能发现 provider 版本变化。
它不能自动证明哪个索引已经错误。
需要结合依赖关系和 `REINDEX` 计划处理。

## 11. 常见误区
### 误区一：把 collation 当成显示格式
collation 不是输出格式。
它直接参与 ordering、equality、index order、merge join order 和 hash equality。

### 误区二：认为 `=` 和 btree compare 返回 0 总是同一件事
对 deterministic text collation，它们通常相容且接近 byte equality。
对 nondeterministic collation，`texteq()` 会走 collation-aware comparison。
btree compare 的 0、equality operator、hash support 必须按 opclass contract 保持相容，但它们不是同一个函数调用。

### 误区三：以为 Sort、Merge Join、btree scan 使用同一个 comparator 对象
它们共享 opclass/collation 语义。
但 Sort 有 tuplesort state 和 abbreviation。
Merge Join 有 `MergeJoinClause.ssup` 且禁用 abbreviation。
btree scan 使用 scan key 和 support functions。
hash 使用 hash/equality functions。

### 误区四：看到 EXPLAIN 没有 Disk 就排除排序问题
内存排序也可能很慢。
复杂 collation 下，CPU comparator 可以成为主成本。
`EXPLAIN` 不显示 comparator 类型。

### 误区五：认为 abbreviation 一定启用
只有 caller 允许、opclass 支持、platform/provider 安全、成本模型未 abort 时，abbreviation 才存在。
Merge Join 不用 abbreviation。
多列排序通常只对 leading key 考虑 abbreviation。

### 误区六：忽视 collation version
ICU/libc 升级后，老索引的物理顺序可能不再匹配当前比较规则。
warning 不是噪声。
它可能要求 `REINDEX`。

### 误区七：把 hash code 当 equality
hash code 只决定候选 bucket。
SQL equality 仍由 equality function 确认。
collation-aware equality 的成本可能出现在 bucket 内比较，而不是排序节点。

## 12. 课堂实验
### 实验一：观察 C collation 与 ICU/libc collation 的排序成本
准备数据：
```sql
CREATE TABLE coll_sort_demo(txt text);
INSERT INTO coll_sort_demo
SELECT repeat(chr(97 + (g % 26)), 4) || '-' || md5(g::text)
FROM generate_series(1, 500000) AS g;
ANALYZE coll_sort_demo;
```
对比：
```sql
SET work_mem = '64MB';
SET trace_sort = on;
SET client_min_messages = log;
EXPLAIN (ANALYZE, BUFFERS)
SELECT txt FROM coll_sort_demo ORDER BY txt COLLATE "C";
EXPLAIN (ANALYZE, BUFFERS)
SELECT txt FROM coll_sort_demo ORDER BY txt COLLATE "und-x-icu";
```
如果本地没有 `und-x-icu`，用 `SELECT collname FROM pg_collation WHERE collprovider='i' LIMIT 5;` 找一个 ICU collation。

观察：
```text
Sort Method 是否相同
是否 spill
total time 差异
trace_sort 是否提示 abbreviation 或 external sort
perf 栈是否出现 provider collation 函数
```
回到源码：
```text
nodeSort.c -> tuplesort_begin_heap()
tuplesortvariants.c -> PrepareSortSupportFromOrderingOp()
varlena.c -> varstr_sortsupport()
```

### 实验二：gdb 确认 SortSupportData
用小表避免断点爆炸：
```sql
CREATE TABLE coll_debug_demo(txt text);
INSERT INTO coll_debug_demo VALUES ('a'), ('A'), ('á'), ('b');
SELECT * FROM coll_debug_demo ORDER BY txt COLLATE "C";
```
断点：
```text
break PrepareSortSupportFromOrderingOp
break bttextsortsupport
break varstr_sortsupport
```
检查：
```text
ssup->ssup_collation
ssup->ssup_nulls_first
ssup->ssup_reverse
ssup->comparator
ssup->abbrev_converter
```
再换 ICU collation 运行一次，比较 comparator 和 abbreviation 字段。

### 实验三：Merge Join 使用 sort support 但没有 abbreviation
构造两个已排序输入：
```sql
CREATE TABLE mj_a(txt text);
CREATE TABLE mj_b(txt text);
INSERT INTO mj_a SELECT g::text FROM generate_series(1, 1000) g;
INSERT INTO mj_b SELECT g::text FROM generate_series(1, 1000) g;
ANALYZE mj_a;
ANALYZE mj_b;
SET enable_hashjoin = off;
SET enable_nestloop = off;
EXPLAIN (ANALYZE, VERBOSE)
SELECT *
FROM mj_a
JOIN mj_b ON mj_a.txt COLLATE "C" = mj_b.txt COLLATE "C";
```
断点：
```text
break ExecInitMergeJoin
break MJCompare
```
确认：
```text
MergeJoinClause.ssup.ssup_collation
MergeJoinClause.ssup.abbreviate == false
ApplySortComparator() 被用于 key 比较
```
回到源码问题：为什么 Merge Join 不能直接复用 Sort 的 abbreviated key？

### 实验四：nondeterministic collation 与 equality
如果本地支持 ICU，可创建：
```sql
CREATE COLLATION demo_nondet (
  provider = icu,
  locale = 'und-u-ks-level1',
  deterministic = false
);
SELECT 'a' COLLATE demo_nondet = 'A' COLLATE demo_nondet;
SELECT 'a' COLLATE "C" = 'A' COLLATE "C";
```
再观察 hash aggregate：
```sql
CREATE TABLE coll_eq_demo(txt text COLLATE demo_nondet);
INSERT INTO coll_eq_demo VALUES ('a'), ('A'), ('á');
EXPLAIN (ANALYZE, VERBOSE)
SELECT txt, count(*)
FROM coll_eq_demo
GROUP BY txt;
```
回到源码：
```text
texteq()
text_cmp()
execGrouping.c
execExprInterp.c equality function call
```
重点不是结果具体有几组，因为 ICU rules 取决于 locale。
重点是 equality 不再等同于 byte equality。

### 实验五：collation version mismatch 的诊断脚本
执行：
```sql
SELECT collname, collprovider, collversion,
       pg_collation_actual_version(oid) AS actual
FROM pg_collation
WHERE collversion IS NOT NULL
ORDER BY collname;
```
如果发现 mismatch，继续：
```sql
SELECT pg_describe_object(refclassid, refobjid, refobjsubid) AS dependent
FROM pg_depend
WHERE classid = 'pg_collation'::regclass;
```
这只是练习依赖追踪。
生产环境处理需要谨慎计划 `ALTER COLLATION ... REFRESH VERSION` 和 `REINDEX`。

## 13. 讨论题
1. 为什么 `SortSupportData` 需要同时携带 `ssup_collation`、`ssup_reverse` 和 `ssup_nulls_first`，而不是只保存一个 comparator 指针？

2. 为什么 Merge Join 可以使用 sort support comparator，却不能使用 tuplesort 的 abbreviated key？

3. 如果一个扩展 btree opclass 没有实现 `BTSORTSUPPORT_PROC`，哪些功能仍然正确，哪些路径会变慢？

4. 为什么 nondeterministic collation 会影响 equality，而不仅仅影响 `ORDER BY`？

5. btree index 的物理顺序为什么会受 ICU/libc 版本变化影响？`collversion` mismatch 为什么不能自动修复旧 index？

6. `ApplySortComparator()` 在 NULL 处理上做了什么？为什么 comparator 本身不应该返回 NULL？

7. hash aggregate 为什么既需要 hash function，又需要 equality function？collation OID 在哪一步仍然重要？

8. 如果 EXPLAIN 显示 Sort 没有 spill，但查询仍很慢，你会用哪些入口判断是不是 collation comparator 成本？

## 14. 本节小结
本节主链路是：
```text
collation OID / opfamily / operator
  -> SortSupportData 或 FmgrInfo / ScanKey / hash table state
  -> Sort、Merge Join、btree、hash 各自的 hot path
  -> SQL ordering/equality 语义必须保持相容
```
核心状态是 `pg_collation`、`pg_locale_t`、`SortSupportData`、`ScanKey.sk_collation`、tuple hash table 的 `collations/eqfunctions/hashfunctions`。
这些状态多为 backend-local 或 executor-local，内存由 MemoryContext 管，外部 temp file 由 tuplesort/ResourceOwner 路径兜底。
正确性来自多层 contract：collation resolution、opclass support function、NULL ordering、deterministic collation tie-break、abbreviation full comparator、hash/equality 相容和 provider version check。
异常路径不是边角料：缺少 sort support 会 fallback 到 `BTORDER_PROC` shim；abbreviation 会按成本 abort；`strxfrm()` 不安全会禁用优化；nondeterministic collation 会关闭 equalimage 类优化；collation version mismatch 需要运维层 rebuild。
观测上，EXPLAIN 能看到 sort 节点、spill 和节点耗时，却看不到 comparator 分支。
`trace_sort`、perf、gdb 和 catalog 查询是必要补充。

可迁移规律是：
```text
内核 hot path 很少直接执行“语义本身”；
它先把语义编译成小状态和函数指针，
再让各模块在自己的资源边界内消费这些状态。
优化可以替换表示，不能改变 contract。
```
判断仍然依赖 workload、硬件、平台 provider、PostgreSQL build 选项和版本。
尤其是 libc/ICU 行为、`strxfrm()` 安全性、abbreviation 收益、排序是否 spill、hash bucket 内冲突，都不能只靠一个源码分支或一个 EXPLAIN 字段下结论。
