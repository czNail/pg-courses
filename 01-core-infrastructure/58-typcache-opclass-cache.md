# PostgreSQL typcache / operator cache / opclass cache
## 课程定位
前置知识：已经理解 `MemoryContext`、`ResourceOwner`、syscache/catcache、relcache 和 shared invalidation 的基本边界。
本节唯一主问题：
```text
typcache、operator cache 和 opclass/opfamily cache 如何把 catalog 元数据变成表达式、排序、索引和 planner 可以低成本使用的语义判断？
```
核心矛盾：表达式分析、排序、聚合、索引匹配和 join planning 需要快速回答“这个类型能否比较、能否相等、能否 hash、哪个 operator 和 support function 表示这件事”；但这些事实不是 `pg_type` 单行能给出的，它们分散在 `pg_operator`、`pg_opclass`、`pg_opfamily`、`pg_amop`、`pg_amproc`、`pg_range`、`pg_constraint` 和 relcache 中，并且会被 DDL 和 invalidation 改变。
一句话运行模型：
```text
lsyscache 用 syscache 把 operator/opclass/opfamily catalog 包成短查询；
lookup_type_cache(type, flags) 在 backend-local TypeCacheEntry 中懒加载 pg_type、默认 btree/hash opclass、opfamily operator、support proc、rowtype/range/domain 信息；
planner 和 executor 只消费已经压缩过的 OID、FmgrInfo 和布尔判断；
invalidation 主要清 flags 和派生指针，让下一次 lookup 重新从 catalog 推导。
```
学完后应能判断：
- 为什么 `ORDER BY`、`GROUP BY`、`DISTINCT` 和 `HashAggregate` 不是直接读 `pg_operator`。
- 为什么 `oprcanmerge` 是 planner hint，而 btree opfamily 才是 mergejoin 语义的实证来源。
- 为什么 `oprcanhash` 对普通 operator 是信号，但数组、record、range、multirange 还要 typcache 复核元素或字段。
- 为什么 typcache entry 长到 backend 退出，但其中的 operator、tupledesc、domain constraint 信息仍会失效。
- 为什么 `pg_opclass` invalidation 会粗粒度清空 typcache 的 operator flags，而不是精确删除某个 entry。
- 为什么 opclass/opfamily cache 不是一个独立全局 cache，而是 syscache helper、typcache 派生状态和 planner/executor 调用约定的组合。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e`。
## 1. 本节在总主线中的位置
第 56 节已经讲过 syscache/catcache 的 tuple lifetime。
第 57 节已经讲过 relcache 如何把多个 catalog row 拼成 `RelationData`。
第 59 节会继续讲 shared invalidation message 如何跨 backend 传播。
本节夹在它们中间。
它回答的是更高一层的问题：
```text
当 parser、planner、executor 问“这个类型可排序吗？”
或“这个 equality operator 能 hashjoin 吗？”
或“这个索引列的 opclass 属于哪个 family？”
PostgreSQL 怎样避免每次都重新扫描一串 catalog？
```
这不是单一 cache。
`typcache.c` 维护类型级派生事实。
`lsyscache.c` 维护大量轻量 catalog helper。
`syscache.c` 和 `catcache.c` 提供 tuple lookup 和 invalidation 基座。
`relcache.c` 提供 composite row type 的 tuple descriptor 和 relation invalidation。
`indexcmds.c`、parser、planner、executor 是主要消费者。
本节只追一条主线：
```text
一个类型 OID
  -> 找默认 btree/hash opclass
  -> 找 opfamily 中的 equality/order/hash support
  -> 缓存到 TypeCacheEntry
  -> parser/planner/executor 用它决定表达式、排序、索引和 plan shape
  -> catalog 变化后清派生状态并在下次 lookup 重算
```
这条线能解释四类现象。
第一，`GROUP BY some_type` 报“could not identify an equality operator”。
第二，`ORDER BY some_type` 报“could not identify an ordering operator”。
第三，一个 `=` operator 有 `oprcanmerge` hint，但 planner 仍可能找不到 mergejoin family。
第四，一个 composite 或 array 的顶层 operator 存在，但字段或元素类型不支持 hash，hash 路径仍不能使用。
注意本节不是 catalog 设计课。
我们不会展开 `CREATE OPERATOR CLASS` 的全部 DDL 语法。
也不会展开每个 index AM 的策略号体系。
本节只看那些会进入高频 runtime 判断的 catalog 事实。
## 2. 核心矛盾与一句话运行模型
表达式和 planner 希望看到的是简单事实：
```text
type T has equality operator E
type T has less-than operator L
type T has btree compare support function C
type T has standard hash support function H
operator O belongs to mergejoinable opfamily F
operator O has hash support functions HL/HR
index column uses opclass C and therefore opfamily F
```
catalog 里保存的不是这些成品事实。
真实来源分散成多层。
| 来源 | 提供什么 |
| --- | --- |
| `pg_type` | 类型的长度、byval、align、storage、typtype、typrelid、typelem、collation。 |
| `pg_operator` | operator 名字、左右输入类型、结果类型、底层函数、commutator、negator、merge/hash hint。 |
| `pg_opclass` | 一个 AM 下某类型的 opclass、是否 default、所属 opfamily、opcintype。 |
| `pg_opfamily` | 语义 family 和所属 access method。 |
| `pg_amop` | opfamily 中 operator 对应的策略号和 search/order purpose。 |
| `pg_amproc` | opfamily 中 support function 对应的 procnum。 |
| `pg_range` | range subtype、subtype opclass、canonical/subdiff function。 |
| relcache | composite type 对应 relation 的 `TupleDesc`。 |
如果每次 planner 判断都直接读这些 catalog：
- `GROUP BY`、`ORDER BY`、`DISTINCT`、`UNION` 分析会反复查同一类型。
- join planning 会在大量 join clause 上反复找 opfamily。
- hash aggregate、memoize、hash join 会反复找 hash support proc。
- array、record、range 的泛型 support function 会在每行执行中泄漏或重复初始化。
如果把结果永久缓存且不失效：
- `ALTER TYPE`、`ALTER TABLE`、`CREATE OPERATOR CLASS` 会让旧事实继续支配 planner。
- composite rowtype 改列后，`record_eq`、`record_cmp` 的字段能力判断会过期。
- domain constraint 修改后，domain input/check 路径会使用旧约束。
- default opclass 增删后，类型的 equality/hash/sort 能力可能改变。
PostgreSQL 的选择是分层。
| 层次 | 代码位置 | 运行语义 |
| --- | --- | --- |
| catalog tuple cache | `syscache.c` / `catcache.c` | 按 key 缓存单行或列表，负责 tuple lifetime 和 invalidation。 |
| operator helper | `lsyscache.c` | 用 syscache 查询 `pg_operator`、`pg_amop`、`pg_amproc`、`pg_opclass`、`pg_opfamily`，返回 OID/布尔值/list。 |
| type derived cache | `typcache.c` | 把类型级能力压缩到 `TypeCacheEntry`，懒加载并跨调用复用。 |
| consumer decisions | parser/planner/executor | 用 OID、`FmgrInfo`、opfamily list、布尔字段决定语义和 plan shape。 |
核心不变量：
```text
catcache 保证 catalog tuple copy 的指针安全；
typcache 保证类型派生事实能低成本复用；
opfamily 保证 operator/support function 属于同一语义集合；
invalidation 只传播“可能过期”，不传播重算后的事实。
```
因此不能把 `pg_operator` 的一个字段当成完整语义。
`oprcanmerge` 只是提示 planner 值得继续找 btree opfamily。
`oprcanhash` 也不是“所有调用场景都能 hash”。
`TypeCacheEntry.hash_proc` 为 valid 才表示 typcache 已经把 default hash opclass、equality operator 一致性和容器元素能力一起考虑过。
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/utils/typcache.h` | `TypeCacheEntry` 字段、`TYPECACHE_*` flags、rowtype/range/domain public API。 |
| 2 | `src/backend/utils/cache/typcache.c` | `lookup_type_cache()`、懒加载、opclass/opfamily 推导、composite/range/domain/enum/record 边界、invalidation callbacks。 |
| 3 | `src/backend/utils/cache/lsyscache.c` | operator cache、opfamily helper、opclass helper、planner 语义判断 helper。 |
| 4 | `src/include/utils/lsyscache.h` | `get_opfamily_member()`、`get_opfamily_proc()`、`op_mergejoinable()`、`op_hashjoinable()` 等调用边界。 |
| 5 | `src/backend/commands/indexcmds.c` | `GetDefaultOpClass()` 如何扫描 default opclass，并处理 exact/binary-compatible/preferred 选择。 |
| 6 | `src/include/catalog/pg_type.h` | `pg_type` fixed 字段和 `TYPEOID` syscache。 |
| 7 | `src/include/catalog/pg_operator.h` | `pg_operator` 字段、`OPEROID`/`OPERNAMENSP` syscache。 |
| 8 | `src/include/catalog/pg_opclass.h` | opclass 的 `opcmethod`、`opcfamily`、`opcintype`、`opcdefault`。 |
| 9 | `src/include/catalog/pg_opfamily.h` | opfamily 的 access method 归属。 |
| 10 | `src/include/catalog/pg_amop.h` | strategy number、operator、purpose、sortfamily 和 `AMOP*` syscache。 |
| 11 | `src/include/catalog/pg_amproc.h` | support proc number 和 `AMPROCNUM` syscache。 |
| 12 | `src/backend/parser/parse_oper.c` | `get_sort_group_operators()` 如何把类型能力暴露给 parser/analyzer。 |
| 13 | `src/backend/optimizer/plan/initsplan.c` | `check_mergejoinable()`、`check_hashjoinable()`、`check_memoizable()` 如何消费 operator/typcache 判断。 |
| 14 | `src/backend/executor/execGrouping.c` | executor 如何用 operator OID 找 equality function 和 hash support function。 |
| 15 | `src/backend/utils/cache/inval.c` | syscache/relcache invalidation 如何触发 typcache callback。 |
| 16 | `src/backend/access/transam/xact.c` | `AtEOXact_TypeCache()` 在 commit/abort/subxact cleanup 中的位置。 |
推荐阅读顺序：
```text
typcache.h 的 TypeCacheEntry
  -> typcache.c 的 lookup_type_cache()
  -> indexcmds.c 的 GetDefaultOpClass()
  -> lsyscache.c 的 get_opfamily_member()/get_opfamily_proc()
  -> parser/parse_oper.c 的 get_sort_group_operators()
  -> optimizer/plan/initsplan.c 的 op_mergejoinable()/op_hashjoinable() 消费点
  -> typcache.c 的 TypeCacheTypCallback/TypeCacheOpcCallback/TypeCacheRelCallback
```
这个顺序按 runtime 问题推进。
不要从 `lsyscache.c` 顶部线性读到尾。
它有很多 helper，但本节只关心三组：operator、opfamily/opclass、planner 兼容性判断。
## 4. 关键数据结构与状态
### 4.1 `TypeCacheEntry`
`TypeCacheEntry` 是 backend-local 的类型派生事实容器。
它存放在 `TypeCacheHash` 里。
key 是 `type_id`。
entry 一旦创建，通常活到 backend 退出。
源码文件头明确说：如果类型被 drop，entry 可能变成 wasted storage；这是有意取舍，因为很多长寿命对象可以安全缓存 `TypeCacheEntry *`。
关键字段可以分成几组。
| 字段组 | 字段 | 语义 |
| --- | --- | --- |
| pg_type copy | `typlen`、`typbyval`、`typalign`、`typstorage`、`typtype`、`typrelid`、`typelem`、`typcollation` | 直接来自 `pg_type` fixed 字段。 |
| default opclass | `btree_opf`、`btree_opintype`、`hash_opf`、`hash_opintype` | default btree/hash opclass 派生出的 opfamily 和 opcintype。 |
| operator | `eq_opr`、`lt_opr`、`gt_opr` | 类型级 equality/order operator。 |
| support proc | `cmp_proc`、`hash_proc`、`hash_extended_proc` | btree compare、standard hash、extended hash support function。 |
| fmgr cache | `eq_opr_finfo`、`cmp_proc_finfo`、`hash_proc_finfo`、`hash_extended_proc_finfo` | 为热路径预构造的函数调用信息。 |
| composite | `tupDesc`、`tupDesc_identifier` | composite row type 的 tuple descriptor 和变化标识。 |
| range | `rngelemtype`、`rng_opfamily`、`rng_collation`、`rng_cmp_proc_finfo`、`rng_canonical_finfo`、`rng_subdiff_finfo` | range subtype 和 range support 信息。 |
| multirange | `rngtype` | multirange 对应的 range type。 |
| domain | `domainBaseType`、`domainBaseTypmod`、`domainData` | domain base type 和约束 cache。 |
| private | `flags`、`enumData`、`nextDomain` | 已计算哪些字段、enum 比较 cache、domain list 链接。 |
`TypeCacheEntry` 的 raw field 不能单独解释。
例如：
```text
hash_proc != InvalidOid
```
表示 typcache 已经找到合适 hash support function。
但它背后还隐含：
- 已检查 default hash opclass。
- 如果 `eq_opr` 已确定，hash opclass 的 equality operator 要匹配它。
- 如果是 array/record/range/multirange，要确认元素或字段的 hash 能力。
- 对应 `TCFLAGS_CHECKED_HASH_PROC` 已置位。
再如：
```text
eq_opr == InvalidOid
```
可能表示类型真的没有 equality。
也可能表示相关 flags 被 invalidation 清掉，下一次带 `TYPECACHE_EQ_OPR` 的 lookup 才会重算。
所以语义必须是：
```text
field + flags + invalidation state + lookup flags
```
而不是裸字段。
### 4.2 `TYPECACHE_*` public flags
调用者不会说“请查 `pg_amop`”。
调用者说“我需要哪类能力”。
常见 flags 如下。
| flag | 调用者想要什么 |
| --- | --- |
| `TYPECACHE_EQ_OPR` | equality operator OID。 |
| `TYPECACHE_LT_OPR` / `TYPECACHE_GT_OPR` | less-than / greater-than operator OID。 |
| `TYPECACHE_CMP_PROC` | btree compare support function。 |
| `TYPECACHE_HASH_PROC` | standard hash support function。 |
| `TYPECACHE_HASH_EXTENDED_PROC` | extended hash support function。 |
| `TYPECACHE_*_FINFO` | 对应 function 的 `FmgrInfo`。 |
| `TYPECACHE_TUPDESC` | composite type tuple descriptor。 |
| `TYPECACHE_BTREE_OPFAMILY` / `TYPECACHE_HASH_OPFAMILY` | default btree/hash opfamily。 |
| `TYPECACHE_RANGE_INFO` | range subtype、opfamily、compare/canonical/subdiff function。 |
| `TYPECACHE_MULTIRANGE_INFO` | multirange 对应 range type。 |
| `TYPECACHE_DOMAIN_BASE_INFO` | domain base type 和 typmod。 |
| `TYPECACHE_DOMAIN_CONSTR_INFO` | domain constraints。 |
这个 API 设计让 hot path 可以只付需要的成本。
`lookup_type_cache(INT4OID, TYPECACHE_EQ_OPR)` 不会顺便加载 range info。
`lookup_type_cache(RECORDOID, TYPECACHE_HASH_PROC)` 也不会构造无关 domain constraints。
### 4.3 private flags
`typcache.c` 内部还有 `TCFLAGS_*`。
它们表达“已经检查过什么”和“容器元素是否具备能力”。
典型组合如下。
| private flag | 语义 |
| --- | --- |
| `TCFLAGS_HAVE_PG_TYPE_DATA` | `pg_type` fixed 字段当前已加载。 |
| `TCFLAGS_CHECKED_BTREE_OPCLASS` | 已查过 default btree opclass。 |
| `TCFLAGS_CHECKED_HASH_OPCLASS` | 已查过 default hash opclass。 |
| `TCFLAGS_CHECKED_EQ_OPR` | 已推导 equality operator。 |
| `TCFLAGS_CHECKED_CMP_PROC` | 已推导 btree compare support function。 |
| `TCFLAGS_CHECKED_HASH_PROC` | 已推导 standard hash support function。 |
| `TCFLAGS_CHECKED_ELEM_PROPERTIES` | 已检查 array/range/multirange 元素能力。 |
| `TCFLAGS_CHECKED_FIELD_PROPERTIES` | 已检查 composite 字段能力。 |
| `TCFLAGS_CHECKED_DOMAIN_CONSTRAINTS` | 已加载 domain constraints。 |
`TCFLAGS_OPERATOR_FLAGS` 是一个内部掩码。
它大体表示 equality/comparison/hashing 相关派生信息。
`TypeCacheOpcCallback()` 和 composite relcache invalidation 会用它粗粒度清理 operator 派生事实。
### 4.4 `TypeCacheHash` 与 `RelIdToTypeIdCacheHash`
`TypeCacheHash` 是 backend-local `HTAB`。
它在第一次 `lookup_type_cache()` 时创建。
它使用和 `TYPEOID` syscache 兼容的 hash function。
这不是为了共享 entry，而是为了让 syscache invalidation 的 `hashvalue` 可以更快定位相关 typcache entry。
`RelIdToTypeIdCacheHash` 也是 backend-local。
它把 relation OID 映射到 composite type OID。
原因是 relcache invalidation callback 只有 relid。
但 typcache 要失效的是 composite type entry。
在 `TypeCacheRelCallback()` 里不能依赖 syscache 去找 relid 对应 type，因为 callback 可能在事务外执行。
所以 typcache 维护自己的小映射。
这就是第 57 节 relcache 和本节 typcache 的连接点：
```text
ALTER TABLE 改 rowtype
  -> relcache inval by relid
  -> TypeCacheRelCallback(relid)
  -> RelIdToTypeIdCacheHash 找 composite type
  -> 清 tupDesc 和 operator flags
```
### 4.5 operator/opclass/opfamily catalog 结构
operator cache 这个词在本节不是指一张独立的 hash。
多数 operator helper 只是 `lsyscache.c` 里的 syscache wrapper。
本节只需要抓住四组字段。
`pg_operator` 给出 `oprleft`、`oprright`、`oprresult`、`oprcode`，以及 `oprcanmerge`、`oprcanhash` 这两个 planner hint。
`pg_opclass` 给出 `opcmethod`、`opcfamily`、`opcintype`、`opcdefault`。
`pg_amop` 把 `(opfamily, lefttype, righttype, strategy)` 映射到 operator。
`pg_amproc` 把 `(opfamily, lefttype, righttype, procnum)` 映射到 support function。
本节最常见的编号是 `BTEqualStrategyNumber`、`BTLessStrategyNumber`、`BTGreaterStrategyNumber`、`HTEqualStrategyNumber`、`BTORDER_PROC`、`HASHSTANDARD_PROC`、`HASHEXTENDED_PROC`。
这些编号不是 typcache 自己发明的语义。
它们属于 index access method 的 operator family 协议。
typcache 只是把 default opclass 的 family 用来回答类型级问题。
## 5. 主流程源码 walkthrough：从 `GROUP BY` 到 typcache
先看 parser/analyzer 的典型路径。
`GROUP BY`、`DISTINCT`、`ORDER BY`、set operation 都需要知道表达式类型的排序和分组 operator。
`parse_oper.c` 中的入口是：
```text
get_sort_group_operators(argtype, needLT, needEQ, needGT,
                         &ltOpr, &eqOpr, &gtOpr, &isHashable)
```
这一步没有直接查 `pg_operator`。
它构造 `TYPECACHE_*` flags：
```text
TYPECACHE_LT_OPR
TYPECACHE_EQ_OPR
TYPECACHE_GT_OPR
TYPECACHE_HASH_PROC
```
然后调用：
```text
lookup_type_cache(argtype, cache_flags)
```
### 5.1 第一次进入 `lookup_type_cache()`
第一次进入时，typcache 初始化两个 backend-local hash：
```text
TypeCacheHash
RelIdToTypeIdCacheHash
```
同时注册四类 callback：
```text
CacheRegisterRelcacheCallback(TypeCacheRelCallback)
CacheRegisterSyscacheCallback(TYPEOID, TypeCacheTypCallback)
CacheRegisterSyscacheCallback(CLAOID, TypeCacheOpcCallback)
CacheRegisterSyscacheCallback(CONSTROID, TypeCacheConstrCallback)
```
这里已经能看到 typcache 的依赖边界。
`TYPEOID` 变化影响 `pg_type` fixed 字段和 domain constraint 重新计算。
`CLAOID` 变化影响 default opclass 派生出的 equality/order/hash 信息。
relcache 变化影响 composite tupledesc 和 record/array 这类泛型 operator 的可用性判断。
`CONSTROID` 变化影响 domain constraints。
`lookup_type_cache()` 还确保 `CacheMemoryContext` 存在。
类型 cache entry、`FmgrInfo` 附属数据、record cache、enum cache 等长寿命对象都依赖这个 context。
### 5.2 查 `pg_type` 并创建 entry
如果 `TypeCacheHash` 没有这个 type OID：
```text
SearchSysCache1(TYPEOID, type_id)
```
它先确认 type 存在且不是 shell type，然后才 `HASH_ENTER` 创建 `TypeCacheEntry`。
entry 复制 `pg_type` 的固定字段，例如 `typlen`、`typbyval`、`typalign`、`typstorage`、`typtype`、`typrelid`、`typelem`、`typarray`、`typcollation`。
如果是 domain，还把 entry 串进 `firstDomainTypeEntry`。
这里会释放 syscache tuple；typcache 不保存 `HeapTuple`，只保存必要字段和派生事实。
这和第 56 节 catcache 的 lifetime 分工一致。
### 5.3 查 default btree opclass
如果调用者需要 equality、ordering、compare proc 或 btree opfamily，而 entry 尚未检查 default btree opclass，typcache 调用：
```text
GetDefaultOpClass(type_id, BTREE_AM_OID)
```
`GetDefaultOpClass()` 在 `indexcmds.c`，会先对 domain 取 base type，再扫描指定 AM 下 `opcdefault` 为真的 `pg_opclass`。
选择规则是 exact match 优先，其次 binary-compatible match；binary-compatible 多个时 preferred type 可以打破平局，多个 exact default 则报 catalog 不一致。
typcache 拿到 opclass 后，继续用 `lsyscache.c`：
```text
get_opclass_family(opclass)
get_opclass_input_type(opclass)
```
这两个函数都通过 `SearchSysCache1(CLAOID, opclass)` 读 `pg_opclass`。
得到的结果写入 `btree_opf` 和 `btree_opintype`。
然后清掉依赖 btree opclass 的已检查 flags。
如果之前类型只有 hash opclass，后来新增了 btree default opclass，equality operator 也要重新选择。
### 5.4 查 default hash opclass
如果需要 hash proc、extended hash proc、hash opfamily，或者 equality 没有 btree 来源，typcache 会检查 default hash opclass：
```text
GetDefaultOpClass(type_id, HASH_AM_OID)
```
成功后写入 `hash_opf` 和 `hash_opintype`。
这一步只清 hash proc 相关 flags。
如果 equality 已经从 btree opfamily 得到，hash opclass 不会覆盖它。
也就是说，type-level equality 优先从 default btree opclass 来；没有 btree equality 时才从 default hash opclass 补。
### 5.5 从 opfamily 找 equality operator
如果调用者请求 `TYPECACHE_EQ_OPR` 或 `TYPECACHE_EQ_OPR_FINFO`，typcache 用 opfamily 查 operator：
```text
get_opfamily_member(btree_opf, btree_opintype, btree_opintype,
                    BTEqualStrategyNumber)
```
如果 btree 没找到，再尝试 hash family：
```text
get_opfamily_member(hash_opf, hash_opintype, hash_opintype,
                    HTEqualStrategyNumber)
```
`get_opfamily_member()` 本身是短 wrapper：
```text
SearchSysCache4(AMOPSTRATEGY,
                opfamily, lefttype, righttype, strategy)
```
命中 `pg_amop` 后返回 `amopopr`。
未命中返回 `InvalidOid`。
这里 typcache 又加了一层容器类型检查：`array_eq` 要确认 element type 有 equality，`record_eq` 要确认所有非 dropped 字段有 equality。
否则 `eq_opr` 会回到 `InvalidOid`。
顶层 operator 存在，不代表所有具体 composite/array 类型都能安全分组或比较。
### 5.6 从 opfamily 找排序 operator 和 compare proc
`TYPECACHE_LT_OPR` 和 `TYPECACHE_GT_OPR` 走 btree opfamily。
它们分别使用：
```text
BTLessStrategyNumber
BTGreaterStrategyNumber
```
`TYPECACHE_CMP_PROC` 使用：
```text
get_opfamily_proc(btree_opf, btree_opintype, btree_opintype,
                  BTORDER_PROC)
```
`get_opfamily_proc()` 通过 `SearchSysCache4(AMPROCNUM, ...)` 查 `pg_amproc`。
对 array/record，typcache 同样会确认 element/field 支持 compare。
如果是 `array_cmp` 或 `record_cmp`，但内部类型不可比较，typcache 不会把它报告为可用。
### 5.7 从 hash opfamily 找 hash proc
`TYPECACHE_HASH_PROC` 使用：
```text
get_opfamily_proc(hash_opf, hash_opintype, hash_opintype,
                  HASHSTANDARD_PROC)
```
但这里有一个额外一致性检查：如果 `eq_opr` 已经确定，typcache 要确认它等于 hash opfamily 里的 equality operator，否则不报告 hash proc。
原因是 hash table 的 correctness 依赖 `equality says equal -> hash values must be compatible`。
如果 equality 来自一个 family，hash support 来自另一个不一致 family，就可能把相等值放进不同 bucket。
对 array/record/range/multirange，typcache 还会确认 element 或 subtype 支持 hash。
这就是 `TYPECACHE_HASH_PROC` 比单查 `pg_amproc` 更强的原因。
### 5.8 构造 `FmgrInfo`
如果调用者请求 `TYPECACHE_EQ_OPR_FINFO`、`TYPECACHE_CMP_PROC_FINFO`、`TYPECACHE_HASH_PROC_FINFO`，typcache 会构造 `FmgrInfo`。
equality operator 先通过：
```text
get_opcode(eq_opr)
```
找到 `pg_operator.oprcode`。
然后调用：
```text
fmgr_info_cxt(function_oid, &typentry->..._finfo, CacheMemoryContext)
```
源码注释强调一个工程细节：这些 `FmgrInfo` 实际在 hash table entry 中，但告诉 fmgr 附属数据属于 `CacheMemoryContext`。
typcache 只有在 operator/function OID 真正变化时才把 `fn_oid` 清成 `InvalidOid`，以减少长会话中重复初始化 support function 带来的 memory leakage。
### 5.9 返回给 parser
`get_sort_group_operators()` 从 `TypeCacheEntry` 读：
```text
lt_opr
eq_opr
gt_opr
hash_proc != InvalidOid
```
如果调用者声明必须有排序 operator，但没有，报：
```text
could not identify an ordering operator for type ...
```
如果必须有 equality，但没有，报：
```text
could not identify an equality operator for type ...
```
否则把 operator OID 和 hashable 布尔值写入 `SortGroupClause` 等上层结构。
到这里，parser 已经不需要知道 default opclass 怎么选、opfamily strategy number 是多少、`pg_amproc` 怎么查。
这就是 typcache 的抽象价值：把多 catalog 推理压缩成类型级稳定接口。
## 6. 主流程源码 walkthrough：operator cache 如何支撑 planner
`lsyscache.c` 的 operator helper 是另一条主线。
它不像 typcache 那样维护一个大 `TypeCacheEntry`。
它主要用 syscache 做短查询，把 catalog row 转成 planner 可消费的 OID/list/布尔值。
### 6.1 `get_opcode()`：executor 函数入口
executor 最终要调用函数，不调用 operator。
`execGrouping.c` 中：
```text
execTuplesMatchPrepare()
  -> get_opcode(eqOperators[i])
  -> ExecBuildGroupingEqual(...)
```
`get_opcode()` 做的事情很窄：
```text
SearchSysCache1(OPEROID, opno)
  -> pg_operator.oprcode
```
返回的是 `RegProcedure`。
如果没有 operator tuple，返回 `InvalidOid`。
这个 helper 不判断 operator 是否适合 hashjoin、mergejoin 或排序。
它只回答“operator 背后的函数是谁”。
### 6.2 `op_mergejoinable()`：hint 加 typcache 特例
planner 在 `initsplan.c` 的 `check_mergejoinable()` 中调用：
```text
op_mergejoinable(opno, exprType(leftarg))
```
对普通 operator，`op_mergejoinable()` 读取：
```text
pg_operator.oprcanmerge
```
但源码注释说得很清楚：这是 hint。
之后 planner 还会调用：
```text
get_mergejoin_opfamilies(opno)
```
如果找不到 btree equality opfamily，`RestrictInfo.mergeopfamilies` 仍然是 `NIL`。
也就是说：
```text
oprcanmerge = true
```
只表示值得继续找。
真正让 mergejoin clause 成立的是 `pg_amop` 中属于 ordering AM 的 equality strategy。
对 `array_eq` 和 `record_eq`，`op_mergejoinable()` 不只看 `pg_operator`。
它调用：
```text
lookup_type_cache(inputtype, TYPECACHE_CMP_PROC)
```
只有 `cmp_proc` 是 `F_BTARRAYCMP` 或 `F_BTRECORDCMP` 时，才返回 true。
这是因为 array/record 的可排序性依赖 element/field 类型。
### 6.3 `get_mergejoin_opfamilies()`：从 operator 找 family list
`get_mergejoin_opfamilies(opno)` 做 list lookup：
```text
SearchSysCacheList1(AMOPOPID, opno)
```
遍历 operator 所有 `pg_amop` 成员关系。
它只保留：
```text
get_opmethod_canorder(amopmethod) == true
IndexAmTranslateStrategy(amopstrategy, ...) == COMPARE_EQ
```
结果是 opfamily OID list。
planner 会把这个 list 放进 `RestrictInfo.mergeopfamilies`。
后续 pathkey、equivalence class、mergejoin path 构造都会用它判断语义兼容。
源码注释还保留一个现实妥协。
operator 可能属于多个 opfamily。
返回 list 的顺序在正常 syscache index scan 下通常按 OID 稳定。
如果关闭系统索引使用，顺序可能不稳定，但那已经不是性能优先场景。
### 6.4 `op_hashjoinable()`：hash hint 与容器复核
`check_hashjoinable()` 调用：
```text
op_hashjoinable(opno, exprType(leftarg))
```
普通 operator 路径读 `pg_operator.oprcanhash`。
对 `array_eq`、`record_eq`、`range_eq`、`multirange_eq`，它调用 typcache：
```text
lookup_type_cache(inputtype, TYPECACHE_HASH_PROC)
```
只有 `hash_proc` 对应 `F_HASH_ARRAY`、`F_HASH_RECORD`、`F_HASH_RANGE` 或 `F_HASH_MULTIRANGE` 时，才认为 hashjoinable。
这让 planner 避免生成运行时才发现元素类型不能 hash 的 plan。
### 6.5 `get_op_hash_functions()`：executor hash 函数
executor hash table 需要 hash support function。
`execTuplesHashPrepare()` 对每个 equality operator 调用：
```text
get_op_hash_functions(eq_opr, &left_hash_function, &right_hash_function)
```
这个 helper 遍历 `AMOPOPID` list。
它找 hash AM 中 strategy 为 `HTEqualStrategyNumber` 的成员。
然后通过 `get_opfamily_proc()` 找左右类型的 `HASHSTANDARD_PROC`。
对单类型 equality，左右 hash function 应相同。
对 cross-type equality，左右 hash function 可能不同。
`execTuplesHashPrepare()` 当前断言不支持 cross-type case：
```text
Assert(left_hash_function == right_hash_function)
```
这说明 planner/executor 接口也有边界。
不是 catalog 能表达 cross-type hash family，所有 executor consumer 就自动支持 cross-type hash。
### 6.6 `equality_ops_are_compatible()`
planner 有时要判断两个 equality operator 是否表达兼容 equality。
例如去重、subquery pullup、distinct 与 group by 的匹配会用到。
`equality_ops_are_compatible(opno1, opno2)` 的规则是：
```text
same operator -> true
否则遍历 opno1 的 pg_amop entries
如果 opno2 也在同一个 opfamily
且该 AM routine 声明 amconsistentequality
则 true
```
这把语义判断从“名字相同”提升到“同一 opfamily 承诺一致 equality”。
`comparison_ops_are_compatible()` 类似，但检查 `amconsistentordering`。
这就是 opfamily 的核心价值：
```text
它不是索引 DDL 的附属品，而是 planner 证明语义兼容的证据。
```
## 7. 主流程源码 walkthrough：索引和 opclass 如何接入
索引列不会只保存 operator。
`pg_index.indclass` 保存每个 key column 的 opclass OID。
`lsyscache.c` 的：
```text
get_index_column_opclass(index_oid, attno)
```
先读 `pg_index`。
如果是 key attribute，返回 `indclass[attno - 1]`。
非 key included column 没有 opclass，返回 `InvalidOid`。
建索引或解析 index inference 时，会走 opclass lookup。
`indexcmds.c` 中的 `GetDefaultOpClass(type_id, am_id)` 用于没有显式指定 opclass 的情况。
`parse_clause.c` 中 `ON CONFLICT` inference 如果用户写了 opclass，会调用：
```text
get_opclass_oid(BTREE_AM_OID, name, false)
```
planner 匹配 unique index 时，opclass/collation/expression 共同参与判断。
typcache 的 default opclass 则服务另一类问题：
```text
没有具体 index 时，类型默认应该如何比较、排序、hash？
```
这两个场景容易混淆。
| 场景 | 使用什么 |
| --- | --- |
| `ORDER BY x` | 类型 default btree opclass，经 typcache 找 `<`/`>`/compare。 |
| `GROUP BY x` | 类型 equality/hash 能力，经 typcache 找 `=` 和 hash proc。 |
| `CREATE INDEX ON t(x)` 未指定 opclass | `GetDefaultOpClass(attrType, accessMethodId)`。 |
| 已有 index scan | index relation 的 `indclass` 和 AM routine。 |
| `ON CONFLICT (x opclass)` | parser 解析显式 opclass，planner 匹配 index。 |
| mergejoin clause | operator 的 opfamily membership list。 |
| hashjoin clause | operator 的 hash family 和 hash support function。 |
同一个 catalog 事实会被不同模块以不同粒度消费。
typcache 不替代 index relcache。
index relcache 需要的是“这个具体 index column 用哪个 opclass”。
typcache 需要的是“这个类型默认能否提供某种语义”。
## 8. composite、array、range、multirange 与 domain 的特殊性
基本类型的 typcache 推导相对直接。
复杂类型的问题是：顶层泛型 operator 存在，但运行时能否成功取决于内部类型。
### 8.1 array
array equality、compare、hash 的 support function 是泛型函数。
`array_eq()`、`array_cmp()`、`hash_array()` 会在运行时处理元素。
所以 typcache 在看到 `ARRAY_EQ_OP`、`F_BTARRAYCMP`、`F_HASH_ARRAY` 等泛型入口时，会调用 `cache_array_element_properties()`。
它通过 `get_base_element_type()` 找 element type，再递归 `lookup_type_cache()` 确认元素的 equality、compare、hash 和 extended hash 能力。
如果元素不可 hash，array 的 hash proc 不会报告为 valid。
### 8.2 composite / record
composite type 的 tuple descriptor 来自 relcache。
`load_typcache_tupdesc()` 打开 `typentry->typrelid`：
```text
relation_open(typrelid, AccessShareLock)
RelationGetDescr(rel)
tdrefcount++
relation_close(rel, AccessShareLock)
```
这里 typcache 持有 tupledesc 引用，但不把它登记到 `CurrentResourceOwner`。
原因是这个引用要活过当前 query。
失效时 `InvalidateCompositeTypeCacheEntry()` 会手动递减 `tdrefcount`，为 0 时 `FreeTupleDesc()`。
字段能力检查在 `cache_record_field_properties()`。
它会 pin tupledesc，遍历非 dropped attribute，对每个 `atttypid` 调用 typcache。
只有所有非 dropped 字段都具备 equality/compare/hash，对应能力才置位。
对裸 `RECORDOID`，typcache 没有具体 tupledesc。
源码选择是假定 equality 和 comparison 可能工作，但不声称 hashing 一定工作；这是避免 planner 过早选择可能运行失败的 hash 路径。
### 8.3 range
range info 来自 `pg_range`。
`load_rangetype_info()` 读取 `rngsubtype`、`rngcollation`、`rngsubopc`、`rngcanonical`、`rngsubdiff`，然后通过 subtype opclass 找：
```text
get_opclass_family(rngsubopc)
get_opclass_input_type(rngsubopc)
get_opfamily_proc(opfamily, opcintype, opcintype, BTORDER_PROC)
```
如果没有 subtype compare support function，直接 ERROR：
```text
missing support function BTORDER_PROC(...) in opfamily ...
```
range 的 equality/order/hash 不只是顶层 `range_eq`。
它需要 subtype opclass 保证边界值的比较语义。
### 8.4 multirange
multirange 通过：
```text
get_multirange_range(multirange_type)
lookup_type_cache(range_type, TYPECACHE_RANGE_INFO)
```
接到 range type，hash 能力最终仍会追到 range subtype。
### 8.5 domain
domain 的 default opclass lookup 会看 base type。
`GetDefaultOpClass()` 一开始就调用 `getBaseType(type_id)`。
typcache 还维护 `domainBaseType`、`domainBaseTypmod`、`domainData`。
domain constraint cache 有自己的 refcount。
`DomainConstraintRef` 允许长寿命调用者引用 domain constraints。
`TypeCacheConstrCallback()` 在任何 `pg_constraint` syscache invalidation 时，遍历 domain entry 清掉 `TCFLAGS_CHECKED_DOMAIN_CONSTRAINTS`。
这个 callback 有意粗粒度。
源码注释承认它无法轻易判断这次 constraint 变化是不是 domain constraint。
表约束变化可能导致无用 flush，但比不缓存 domain constraints 更可接受。
## 9. 生命周期 / ownership / cleanup
### 9.1 谁创建
`TypeCacheHash` 在第一次 `lookup_type_cache()` 时创建。
`TypeCacheEntry` 在第一次查询某 type OID 时创建。
它们是 backend-local。
其他 backend 不能访问这个指针。
operator helper 返回的 OID/list 多数来自 syscache lookup。
它们没有独立长寿命 owner。
`get_opfamily_member()` 返回一个 `Oid`。
`get_mergejoin_opfamilies()` 返回 caller context 中的 list。
`get_opcode()` 返回一个 function OID。
### 9.2 谁持有
`TypeCacheEntry *` 可以被长寿命代码保存。
源码设计允许这样做，因为 entry 活到 backend 退出。
但保存 pointer 不代表保存的派生字段永远新鲜。
字段新鲜度由 flags 和 invalidation 控制。
`FmgrInfo` 存在 entry 中。
它的附属数据用 `CacheMemoryContext`。
composite `tupDesc` 是 refcounted tupledesc。
typcache 对它的引用不进 `ResourceOwner`。
由 typcache invalidation 手动释放。
`lookup_rowtype_tupdesc()` 返回给 caller 的 tupledesc 会调用 `PinTupleDesc()`。
这会进入 refcount/ResourceOwner 规则。
caller 必须 `ReleaseTupleDesc()`。
`lookup_rowtype_tupdesc_copy()` 则返回当前 context 中的 copy，不 refcount。
### 9.3 谁释放
普通 `TypeCacheEntry` 不逐个释放。
backend 退出时进程内存整体回收。
这和 catcache tuple 不同。
catcache entry 会被 invalidation 标 dead 并在 refcount 归零后物理删除。
typcache entry 更像“进程生命周期索引”。
失效一般做两件事：
```text
清 flags
释放或清空派生子对象
```
例如 composite relcache invalidation：
```text
InvalidateCompositeTypeCacheEntry()
  -> tdrefcount--
  -> maybe FreeTupleDesc()
  -> tupDesc = NULL
  -> tupDesc_identifier = 0
  -> flags &= ~TCFLAGS_OPERATOR_FLAGS
```
opclass invalidation：
```text
TypeCacheOpcCallback()
  -> scan TypeCacheHash
  -> flags &= ~TCFLAGS_OPERATOR_FLAGS
```
type invalidation：
```text
TypeCacheTypCallback()
  -> clear TCFLAGS_HAVE_PG_TYPE_DATA
  -> clear TCFLAGS_CHECKED_DOMAIN_CONSTRAINTS
```
domain constraint invalidation：
```text
TypeCacheConstrCallback()
  -> scan firstDomainTypeEntry
  -> clear TCFLAGS_CHECKED_DOMAIN_CONSTRAINTS
```
### 9.4 ERROR / abort 时谁兜底
`lookup_type_cache()` 维护 `in_progress_list`。
原因是构造 entry 期间可能 ERROR 或被 invalidation 打断。
有些 composite type entry 需要进入 `RelIdToTypeIdCacheHash`，这样未来 relcache invalidation 才能找到它。
如果构造中途异常，entry 可能已经在 `TypeCacheHash` 中，但 mapping 还没补全。
事务结束和子事务结束都会调用：
```text
AtEOXact_TypeCache()
AtEOSubXact_TypeCache()
```
它们执行：
```text
finalize_in_progress_typentries()
  -> 对 in_progress_list 中的 type
  -> hash_search(TypeCacheHash)
  -> insert_rel_type_cache_if_needed()
  -> in_progress_list_len = 0
```
`xact.c` 中 commit 顺序是：
```text
ResourceOwnerRelease(... BEFORE_LOCKS ...)
AtEOXact_RelationCache(true)
AtEOXact_TypeCache()
AtEOXact_Inval(true)
ResourceOwnerRelease(... LOCKS ...)
```
abort 路径也调用 `AtEOXact_TypeCache()`。
subtransaction commit/abort 调用 `AtEOSubXact_TypeCache()`。
这不是释放 typcache entry。
它是把可能半成品 entry 的辅助映射补齐，避免后续 invalidation 找不到 composite type。
## 10. invalidation 如何连接 typcache、operator cache 和 opclass cache
typcache 依赖 shared invalidation，但它不是 receiver 的第一站。
catalog update 先生成 syscache/relcache invalidation message。
receiver 执行 message 时，syscache 或 relcache callback 再调用 typcache callback。
### 10.1 `pg_type` invalidation
`TypeCacheTypCallback()` 注册在 `TYPEOID`。
它接收 `hashvalue`。
如果 `hashvalue == 0`，按约定表示 whole cache invalidation。
否则用：
```text
hash_seq_init_with_hash_value(TypeCacheHash, hashvalue)
```
只扫描同 hash bucket 的 entry。
它清：
```text
TCFLAGS_HAVE_PG_TYPE_DATA
TCFLAGS_CHECKED_DOMAIN_CONSTRAINTS
```
下一次 `lookup_type_cache()` 发现没有 `TCFLAGS_HAVE_PG_TYPE_DATA`，会重新 `SearchSysCache1(TYPEOID, type_id)`。
注意它不是删除 entry。
这让长寿命 `TypeCacheEntry *` 仍可安全解引用。
### 10.2 `pg_opclass` invalidation
`TypeCacheOpcCallback()` 注册在 `CLAOID`。
当任何 `pg_opclass` row invalidated 时，它扫描整个 `TypeCacheHash`。
它清：
```text
TCFLAGS_OPERATOR_FLAGS
```
源码注释解释了粗粒度策略。
理论上可以只清依赖特定 opclass 的 typcache entry。
但 `pg_opclass` 更新在生产中很少。
为少见 DDL 维护精确依赖图不划算。
更重要的是，typcache 不监听 `pg_amop` 或 `pg_amproc`。
源码给出的理由是：
```text
ALTER OPERATOR FAMILY ADD/DROP OPERATOR/FUNCTION
不允许增删 opclass 的 primary operators/functions；
它只能影响 cross-type family members；
typcache 缓存的是 default opclass 的 primary same-type members。
```
这是一条版本和语义边界。
如果你改的是 cross-type opfamily member，planner 的 cross-type helper 会通过 syscache list 重新看 catalog。
typcache 不会为此维护类型级同类型字段。
### 10.3 relcache invalidation
`TypeCacheRelCallback()` 注册在 relcache callback。
它处理 composite type。
如果 relid valid，它用 `RelIdToTypeIdCacheHash` 找 composite typid。
找到后清：
```text
tupDesc
tupDesc_identifier
TCFLAGS_OPERATOR_FLAGS
```
如果 relid 是 `InvalidOid`，按约定清所有 composite type。
它还会处理 domain over composite。
如果 domain entry 已知道 base type 是 composite，就清 operator flags。
这是因为 domain 的 record/field comparability 可能因底层 composite schema 改变。
### 10.4 `pg_constraint` invalidation
`TypeCacheConstrCallback()` 不按 hash 精确定位。
它遍历 `firstDomainTypeEntry`。
所有 domain entry 的 `TCFLAGS_CHECKED_DOMAIN_CONSTRAINTS` 都清掉。
这会让下次 domain constraint lookup 重扫 `pg_constraint`。
粗粒度是 deliberate trade-off。
domain 类型数量通常不会像临时表 composite type 那样膨胀。
### 10.5 operator helper 的 invalidation
`lsyscache.c` helper 依赖 syscache。
例如：
```text
get_opcode() -> OPEROID
get_opfamily_member() -> AMOPSTRATEGY
get_opfamily_proc() -> AMPROCNUM
get_opclass_family() -> CLAOID
```
这些 helper 本身不持有长期 tuple。
它们的“cache”来自 syscache/catcache。
因此 invalidation 主要由第 56 节讲过的 catcache 处理。
如果 planner 已经把结果 OID 保存在一个 plan 结构中，后续 plan cache invalidation 又是另一层。
plan cache 会通过 relcache/syscache callback 标记 cached plan 过期。
typcache 不负责回收已经生成的 plan。
## 11. 正确性机制层次
本节正确性不是一个锁或一个 cache 保证的。
| 层次 | 机制 | 保证什么 | 不保证什么 |
| --- | --- | --- | --- |
| catalog visibility | MVCC / command counter | lookup 读到当前命令边界应可见的 catalog row。 | 不保证本地派生字段自动更新。 |
| catcache | `SearchSysCache*()`、refcount、invalidation | catalog tuple copy 指针安全和 tuple 级失效。 | 不知道类型级 equality/hash 语义。 |
| typcache flags | `TYPECACHE_*` / `TCFLAGS_*` | 只在需要时重算派生字段。 | 不阻塞 DDL，也不保证 field 单独语义完整。 |
| opfamily | `pg_amop` / `pg_amproc` | operator 与 support function 属于同一 AM family。 | 不代表所有 executor consumer 支持所有 cross-type 组合。 |
| relcache | relation lock + tupledesc refcount | composite tupledesc 可安全使用并在 DDL 后失效。 | 不替代 typcache 的字段能力判断。 |
| invalidation | syscache/relcache callbacks | catalog 变化后清本地派生状态。 | 不发送新 operator、tupledesc 或 opfamily 内容。 |
| transaction cleanup | `AtEOXact_TypeCache()` | ERROR/abort 后补齐 in-progress composite mapping。 | 不释放所有 typcache entry。 |
表达式正确性尤其依赖 opfamily 一致性。
hash join 和 hash aggregate 要求：
```text
if eq(a, b) then hash(a) == hash(b)
```
typcache 的 `hash_proc` 查找必须和 `eq_opr` 对齐。
merge join 和 pathkey 要求：
```text
ordering operator、equality operator、compare support function 属于兼容 ordering semantics
```
planner 因此要看 opfamily list，而不是只看 operator 名称。
composite/array 正确性还依赖递归能力检查。
顶层 `record_eq` 存在，但某字段没有 equality，`GROUP BY composite_col` 不能被认为可行。
顶层 `hash_record` 存在，但某字段没有 hash，hash aggregate 不能安全使用。
## 12. 错误路径 / 异常路径 / fallback
### 12.1 类型不存在或 shell type
`lookup_type_cache()` 在创建 entry 前查 `TYPEOID`。
不存在时报用户可见 ERROR。
shell type 也报 ERROR。
这是为了防止用无效 type OID 创建长寿命 typcache entry。
一些路径如 `record_in()` 可能接收用户提供的 OID，所以错误必须是面向用户的，而不是 assert。
### 12.2 没有 equality 或 ordering operator
`get_sort_group_operators()` 根据 caller 的 `needLT/needEQ/needGT` 决定是否报错。
典型 SQL 现象是：
```text
could not identify an equality operator for type ...
could not identify an ordering operator for type ...
```
源码解释不是“找不到名为 = 或 < 的 operator”这么简单。
它是：
```text
default btree/hash opclass
  -> opfamily member
  -> container element/field ability
```
这条链上任意一步不成立，都会表现为类型没有所需能力。
### 12.3 default opclass 不唯一或不确定
`GetDefaultOpClass()` 处理 default opclass 时可能失败。
多个 exact default opclass 是 catalog inconsistent，报 duplicate object。
多个 binary-compatible default 且无法用 preferred type 唯一选择，则返回 `InvalidOid`。
调用者看到的是类型缺少默认比较/索引能力。
这类问题经常来自 extension 或手工 catalog/opclass 定义错误。
### 12.4 range subtype opclass 缺 support function
`load_rangetype_info()` 从 `pg_range.rngsubopc` 找 opfamily。
如果找不到 `BTORDER_PROC`，直接 ERROR。
range 类型要求 subtype 有排序语义。
这里不能退化成“不可排序但可存在”的半状态，因为 range bound 比较是基本语义。
### 12.5 stale composite tupledesc
`ALTER TABLE` 改 composite rowtype 后，relcache invalidation 到达。
如果 typcache 持有旧 tupledesc，会在 callback 中释放引用并置空。
下一次请求 `TYPECACHE_TUPDESC` 时重新打开 relation 获取 tupledesc。
如果某段代码已经 pin 了旧 tupledesc，它靠 refcount 保持内存安全。
但语义新鲜度要靠 relation lock、command boundary 和 invalidation。
### 12.6 lookup 中途 ERROR
`lookup_type_cache()` 可能在扫描 syscache、打开 relation、构造 tupledesc、加载 domain constraints 时 ERROR。
如果 ERROR 发生在 entry 已进入 `TypeCacheHash` 之后，`in_progress_list` 记录会在事务或子事务结束时由 `finalize_in_progress_typentries()` 收尾。
这个 fallback 很细，但它防止了后续 relcache invalidation 找不到 composite typcache entry 的风险。
### 12.7 enum cache miss retry
`compare_values_of_enum()` 有自己的异常路径。
如果 enum cache 中找不到传入 enum value，说明 enum 可能变化过。
它会重新 `load_enum_cache_data()` 再找一次。
如果仍找不到，才报 cache corruption 类错误。
这个例子说明 typcache 不只有 invalidation callback。
有些局部 cache 会在运行时 miss 时主动刷新。
## 13. 成本、资源与跨模块传播
### 13.1 CPU 成本
hot path 希望只是 `hash_search(TypeCacheHash)`，发现 flags 已满足，然后读 OID 或 `FmgrInfo`。
miss 或 invalidation 后的 slow path 会放大为 `SearchSysCache(TYPEOID)`、`GetDefaultOpClass()` 扫 `pg_opclass`、`SearchSysCache(CLAOID)`、`SearchSysCache(AMOPSTRATEGY)`、`SearchSysCache(AMPROCNUM)`，并可能递归 lookup element/field type。
对 composite type，字段数越多，`cache_record_field_properties()` 越贵。
它会按非 dropped 字段递归查 typcache。
对多列 `GROUP BY`、extended statistics、join clauses，类型能力检查会被重复触发，但 typcache 命中后成本会降到 backend-local hash。
### 13.2 memory 成本
`TypeCacheEntry` 活到 backend 退出。
大量临时 type、composite type、record typmod、domain 或 enum 使用可能让单 backend 的 `CacheMemoryContext` 增长。
这通常可接受，因为类型定义变动远少于数据行处理。
但长连接池、动态创建类型、反复创建临时表的 workload 会让 typcache/relcache/catcache memory 更明显。
`pg_backend_memory_contexts` 可以看到 `CacheMemoryContext` 的总量。
它看不到每个 `TypeCacheEntry` 的细粒度语义。
### 13.3 invalidation fan-out
`pg_opclass` invalidation 会让每个已有 `TypeCacheEntry` 清 operator flags。
这在 DDL 频繁修改 opclass 的 workload 下是 O(number of cached types)。
PostgreSQL 接受这个成本，是因为 opclass DDL 在生产热路径中很少。
相比维护精确依赖图，粗粒度 scan 更简单也更稳。
`pg_constraint` invalidation 遍历 domain list。
domain 数量通常较小。
composite relcache invalidation 通过 `RelIdToTypeIdCacheHash` 定位，避免扫描所有临时表自动产生的 rowtype。
### 13.4 planner 成本传播
typcache 输出会影响 planner 搜索空间。
只有类型 hashable，`HashAggregate`、`Hash Join`、`Memoize`、hash-based `DISTINCT` 才有机会进入候选。
只有类型 sortable，`Sort`、`GroupAggregate`、`Merge Join`、pathkey / `EquivalenceClass` 才有可行语义。
这不是简单性能开关。
它改变 plan shape 的可行集合。
某个类型缺 hash proc 时，planner 可能退到 sort-based plan。
某个 operator 不在 mergejoin opfamily 中时，mergejoin path 根本不会形成。
### 13.5 index AM 边界
opfamily 是 access method 协议的一部分。
btree、hash、GiST、GIN、SP-GiST、BRIN 的 strategy/proc 解释不同。
`get_opmethod_canorder()` 对内置 AM 做快速 hardcode。
非内置 AM 会调用 `GetIndexAmRoutineByAmId(amoid, false)`。
planner 判断 equality/order compatibility 时会看 AM routine 的能力字段，例如 `amconsistentequality`、`amconsistentordering`。
所以 extension index AM 不能只写 catalog row。
它还要在 AM routine 中表达 planner 可依赖的语义承诺。
### 13.6 跨模块连接
| 模块 | 连接点 |
| --- | --- |
| parser/analyzer | `get_sort_group_operators()` 决定 `SortGroupClause` 的 eq/sort operator。 |
| optimizer | `op_mergejoinable()`、`get_mergejoin_opfamilies()`、`op_hashjoinable()`、`equality_ops_are_compatible()`。 |
| executor | `get_opcode()`、`get_op_hash_functions()`、`FmgrInfo` 用于实际函数调用。 |
| relcache | composite tupledesc 来源和 relcache invalidation。 |
| catcache/syscache | 所有 catalog tuple lookup 和 helper 的基础。 |
| plancache | catalog invalidation 后，已生成 plan 需要通过 callback 标 stale。 |
| index AM | opclass/opfamily strategy/proc 语义由 AM 定义和解释。 |
本节没有 WAL 或后台进程直接推进状态。
这是 backend-local cache。
跨 backend 的推进来自 shared invalidation message，由普通 backend 在安全点接收和执行。
## 14. 观测与诊断入口
### 14.1 SQL 可见现象
最直接的观测是错误信息。
```sql
SELECT x FROM t GROUP BY x;
SELECT x FROM t ORDER BY x;
```
如果类型缺 equality 或 ordering，错误会在 parse/analyze 阶段出现。
源码入口是 `get_sort_group_operators()`。
这类错误不是 executor 运行到某行才发现，而是 parser/analyzer 无法从 typcache 得到必要 operator。
### 14.2 `EXPLAIN` 可见 plan shape
对同一 join clause：
```sql
EXPLAIN SELECT * FROM a JOIN b ON a.x = b.x;
```
如果 operator hashjoinable，可能看到 `Hash Join`。
如果 operator mergejoinable 且 pathkeys/sort 成本合适，可能看到 `Merge Join`。
如果二者都不成立，通常只能走 nested loop 或其他替代路径。
但 `EXPLAIN` 只能看到 planner 最终选择。
它看不到“为什么某个 path 没进入候选”；要解释原因，需要回到 `check_mergejoinable()`、`check_hashjoinable()`、`get_mergejoin_opfamilies()`、`op_hashjoinable()`。
### 14.3 catalog 查询
可以用 catalog 查询验证一条链。
示例方向：
```sql
SELECT opc.oid, opc.opcname, opc.opcfamily, opc.opcintype
FROM pg_opclass opc
JOIN pg_am am ON am.oid = opc.opcmethod
WHERE am.amname IN ('btree', 'hash')
  AND opc.opcdefault;
```
再查 `pg_amop`：
```sql
SELECT amopfamily, amoplefttype, amoprighttype, amopstrategy, amopopr
FROM pg_amop
WHERE amopfamily = $1;
```
再查 `pg_amproc`：
```sql
SELECT amprocfamily, amproclefttype, amprocrighttype, amprocnum, amproc
FROM pg_amproc
WHERE amprocfamily = $1;
```
这些查询能看到 catalog facts。
它们不能证明当前 backend typcache 是否已经失效或重算。
### 14.4 memory context
`pg_backend_memory_contexts` 可以观察 `CacheMemoryContext` 的总体增长。
你通常只能看到 `CacheMemoryContext` 增长。
看不到每个 type 的 `eq_opr`、`hash_proc`、`flags`。
要看具体 entry，需要 gdb 或临时 instrumentation。
### 14.5 gdb 断点
推荐断点：`lookup_type_cache`、`GetDefaultOpClass`、`get_opfamily_member`、`get_opfamily_proc`、`get_sort_group_operators`、`op_mergejoinable`、`op_hashjoinable`、`TypeCacheOpcCallback`、`TypeCacheRelCallback`。
一个有用的调试路径是：
```text
break get_sort_group_operators
run SELECT ... GROUP BY ...
print argtype
next 到 lookup_type_cache
finish
print *typentry
```
对 invalidation：
```text
break TypeCacheOpcCallback
break TypeCacheRelCallback
在另一个 session 执行相关 DDL
回到当前 backend 触发 AcceptInvalidationMessages()
```
注意 invalidation 接收通常发生在安全点。
不是 DDL commit 的瞬间，所有 backend 的 C 栈都立刻进入 callback。
### 14.6 日志和 perf
普通 server log 不会记录 typcache hit/miss。
`debug_print_plan` 可以看到 plan 变化，但不是 typcache 诊断。
CPU profiling 中，如果 workload 反复触发 catalog miss 或频繁 DDL invalidation，可能看到 `lookup_type_cache`、`SearchSysCache*`、`GetDefaultOpClass`、`get_opfamily_member`、`get_opfamily_proc`。
但多数 OLTP 查询中，typcache 成本会被 executor expression、buffer access、lock wait 或 planner 搜索成本淹没。
不要把一次火焰图上的 `lookup_type_cache` 栈直接解释成 opclass 问题。
先区分：
- 是否是首次 query warmup。
- 是否有动态创建类型或临时表。
- 是否有 DDL/invalidation 风暴。
- 是否是 composite/record 字段递归检查。
- 是否是 planner 搜索空间本身很大。
## 15. 课堂实验
### 实验 1：跟读 `GROUP BY` 的类型能力推导
目标：看到 SQL 错误或成功如何回到 typcache。
在本地 PostgreSQL debug build 上设置断点：
```text
break get_sort_group_operators
break lookup_type_cache
break GetDefaultOpClass
break get_opfamily_member
break get_opfamily_proc
```
执行：
```sql
SELECT typname FROM pg_type GROUP BY typname;
```
在 `get_sort_group_operators()` 记录 `argtype`，进入 `lookup_type_cache()` 观察 `TYPECACHE_EQ_OPR` 和 `TYPECACHE_HASH_PROC`，再到 `GetDefaultOpClass()` 确认 btree/hash AM 是否被查询。
要画出的状态：
```text
argtype
  -> btree_opf/hash_opf
  -> eq_opr
  -> hash_proc
  -> SortGroupClause eqop/hashable
```
### 实验 2：观察 array 或 composite 的递归能力检查
目标：理解顶层泛型 operator 不等于具体类型一定可 hash。
找一个元素类型不具备 hash 或排序能力的自定义类型，或者用 extension 类型做实验。
创建 array 或 composite 列，对列执行 `GROUP BY`、`ORDER BY` 或 hash aggregation 场景。
在 `cache_array_element_properties()` 或 `cache_record_field_properties()` 设置断点，观察每个 element/field 的 `lookup_type_cache()` flags。
如果不方便构造缺能力类型，可以先用已有可比较类型跑通链路，再在 gdb 中观察代码如何决定 `TCFLAGS_HAVE_FIELD_HASHING`。
关键不是得到某个特定错误，而是看到能力判断递归到内部类型。
### 实验 3：opclass invalidation 的粗粒度清理
目标：看到 `pg_opclass` 变化如何让 typcache 清 operator flags。
在 session A 中跑一次触发 typcache 的查询：
```sql
SELECT 1 GROUP BY 1;
```
在 session A 的 backend 上断点：
```text
break TypeCacheOpcCallback
break lookup_type_cache
```
在 session B 执行会修改 opclass 相关 catalog 的 DDL。
实际生产库不要随意改 operator class；实验库中可创建 extension opclass 或临时测试 opclass。
回到 session A，执行下一条查询触发 invalidation 接收；观察 `TypeCacheOpcCallback()` 扫描 `TypeCacheHash` 并清 `TCFLAGS_OPERATOR_FLAGS`，再观察下一次 `lookup_type_cache()` 重算 opclass/opfamily。
诊断结论：
```text
typcache entry 仍存在；
operator/hash/compare 派生事实被清；
下一次按 flags 懒加载。
```
## 16. 常见误区
误区一：把 typcache 当成 syscache 的别名。syscache 缓存 catalog tuple，typcache 缓存从 default opclass 和 opfamily strategy 推导出的类型级事实。
误区二：认为 operator cache 是全局共享 operator hash。`lsyscache.c` 的 operator helpers 多数只是 syscache wrapper，没有所有 backend 共享的 operator C 指针表。
误区三：看到 `oprcanmerge` 就认为 mergejoin 一定可用。`oprcanmerge` 是 hint，planner 还要 `get_mergejoin_opfamilies()` 找到 ordering opfamily。
误区四：看到 `oprcanhash` 就认为 hash join 一定可用。array、record、range、multirange 还要 typcache 复核内部类型，executor 还要能找到 hash support function。
误区五：认为 default opclass 只影响 `CREATE INDEX`。它也支撑类型级 equality、ordering、compare、hash 判断。
误区六：认为 typcache invalidation 会释放 entry。多数情况下它只清 flags 或释放子对象，entry 本身活到 backend 退出。
误区七：把 composite tupledesc refcount 当成语义新鲜度。refcount 只保证内存安全，`ALTER TABLE` 后的新鲜度依赖 relcache invalidation 和重新 lookup。
误区八：用 `pg_stat_*` 解释 typcache。`pg_stat_*` 基本看不到 typcache hit/miss，需要 catalog 查询、`EXPLAIN`、gdb、perf 或临时日志接回源码。
## 17. 讨论题
1. 为什么 `lookup_type_cache()` 要按 flags 懒加载，而不是第一次就填满所有字段？
2. 为什么 default btree opclass 的 equality 优先于 default hash opclass 的 equality？
3. 为什么 `hash_proc` 必须和 `eq_opr` 所属 hash opfamily 的 equality operator 一致？
4. 为什么 `TypeCacheOpcCallback()` 选择扫描全部 `TypeCacheHash`，而不是维护 opclass 到 type 的精确依赖表？
5. 为什么 typcache 不监听 `pg_amop` 和 `pg_amproc` 的所有变化？
6. 为什么 `record_eq` 存在不代表任意 composite type 都能 `GROUP BY`？
7. 为什么 relcache callback 传入 relid 时，typcache 不能临时用 syscache 去查 composite type OID？
8. `oprcanmerge` 和 `get_mergejoin_opfamilies()` 分别承担什么语义？
9. 哪些状态能用 SQL catalog 查询看到，哪些必须用 gdb 才能看到？
10. 如果一个长连接 backend 反复创建临时 composite type，typcache memory 会怎样增长？
## 18. 本节小结
本节主链路是：
```text
type OID / operator OID
  -> syscache/lsyscache 读取 catalog facts
  -> typcache 按 flags 懒加载类型级派生事实
  -> parser/planner/executor 消费 OID、FmgrInfo、opfamily list 和布尔判断
  -> syscache/relcache invalidation 清 flags 或子对象
  -> 下一次 lookup 重算
```
核心状态是 backend-local。
`TypeCacheEntry` 活到 backend 退出。
`TypeCacheEntry *` 可以长期保存。
但 `eq_opr`、`cmp_proc`、`hash_proc`、`tupDesc`、`domainData` 这些派生字段会被 invalidation 清理或重建。
ownership 要分层看：syscache tuple 由 catcache refcount 和 ResourceOwner 管，typcache entry 由 backend 生命周期管，composite tupledesc 有 refcount，domain constraints 有 `DomainConstraintRef`，`FmgrInfo` 附属数据进入 `CacheMemoryContext`。
正确性来自 opfamily 语义，而不是 operator 名字。
planner 要证明 equality/order/hash 兼容，必须通过 `pg_amop`、`pg_amproc` 和 AM routine 能力字段。
typcache 再把这些证明压缩成类型级字段。
异常路径的共同模式是：不删除长寿命 entry，只清派生状态，保留指针安全，并让下一次 lookup 在正确 catalog visibility 下重算。
可观测入口有限。
SQL 错误和 `EXPLAIN` 可以看到结果。
catalog 查询可以看到 opclass/opfamily facts。
`pg_backend_memory_contexts` 只能看 memory context 粒度。
具体 flags 和 callback 行为通常要靠 gdb、perf 或临时 instrumentation。
可迁移规律：
```text
高频 planner/executor 判断不要直接依赖分散 catalog row；
先把 catalog facts 压缩成本地派生状态；
用 invalidation 清“语义有效性”，用 refcount/MemoryContext 保“指针安全”；
用 opfamily 这类语义集合证明跨 operator/support function 的一致性。
```
这条规律会在 plan cache、partition cache、统计信息 cache 和 index AM 课程中反复出现。
