# PostgreSQL 表、索引、函数统计与执行节点对应关系

## 课程定位

前置知识：已经理解核心 pg_stat flush 机制和 pg_stat_statements 的 queryid 聚合。

本节唯一主问题：

```text
pg_stat_all_tables、pg_stat_all_indexes、pg_stat_user_functions 等视图里的计数如何对应执行节点、访问方法和函数调用路径？
```

核心矛盾：用户看到的是按对象累计的统计视图，但执行时事件发生在 heap AM、index AM、buffer manager、expression interpreter、SRF、trigger 和 DML 节点之间；如果不区分对象维度和执行节点维度，很容易把累计计数误读成某条 SQL 的节点事实。

学完后应能判断：能把表 scan/DML、索引 scan/fetch、函数 call/time 的计数分别追到 pgstat_relation.c、pgstat_function.c 和访问方法调用点。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 30 节解释了 pg_stat pending 和 flush。

第 31 节解释了 queryid 维度聚合。

本节换一个维度：对象维度。

表、索引、函数统计不是 EXPLAIN 节点。

它们是执行路径在对象边界上留下的累计痕迹。

下一节会继续处理统计刷新延迟和 reset scope 的诊断误区。

本节刻意只处理一个问题：观测状态如何在执行器或统计体系中找到稳定边界。

凡是不能帮助解释这个边界的函数和字段，都只在后续课程或源码练习中出现。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
relation 打开时初始化 pgstat_enabled，真正产生 scan/DML/buffer 事件时通过 pgstat_count_* 把计数挂到 relation pending entry；index AM 在扫描入口和 tuple 返回处更新 index stats；函数执行器在调用前后用 pgstat_init_function_usage / pgstat_end_function_usage 记录 total/self time，最终 SQL 视图从共享统计 entry 读取累计值。
```

这句话里有三层含义。

第一层是入口边界：用户看到的 SQL 选项、统计视图或扩展 hook，必须先被转换成执行期状态。

第二层是状态边界：执行路径不能在每个热函数里直接做复杂诊断，只能在少数明确对象上累计事实。

第三层是解释边界：输出或视图只是读取这些事实，不能把估算、累计、平均、延迟刷新混成一个语义。

本节后面的所有源码 walkthrough 都围绕这三层展开。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/pgstat.h` | pgstat_count_heap_scan、pgstat_count_heap_getnext、pgstat_count_index_scan、pgstat_count_index_tuples、pgstat_count_heap_insert/update/delete 等宏和接口。 |
| 2 | `src/backend/utils/activity/pgstat_relation.c` | pgstat_init_relation、pgstat_assoc_relation、relation DML counters、AtEOXact_PgStat_Relations 和 fetch 函数。 |
| 3 | `src/backend/access/heap/heapam.c` | heap scan、insert、update、delete 路径调用表统计接口。 |
| 4 | `src/backend/access/heap/heapam_handler.c` | table AM fetch / getnext 对统计的补充调用。 |
| 5 | `src/backend/access/index/indexam.c` | index_getnext_tid、index_fetch_heap、index_getbitmap 等路径更新 index tuples/fetch。 |
| 6 | `src/backend/access/nbtree/nbtsearch.c` | btree index scan 入口更新 index scan 计数。 |
| 7 | `src/backend/storage/buffer/bufmgr.c` | buffer read/hit 计数可归因到 relation。 |
| 8 | `src/backend/utils/activity/pgstat_function.c` | pgstat_init_function_usage、pgstat_end_function_usage、function flush 和 fetch。 |
| 9 | `src/backend/executor/execExprInterp.c` | 表达式执行中函数调用统计入口。 |
| 10 | `src/backend/executor/execSRF.c` | set-returning function 调用统计入口。 |
| 11 | `src/backend/utils/fmgr/fmgr.c` | fmgr 层函数调用和 track_functions 的关系。 |
| 12 | `src/backend/utils/adt/pgstatfuncs.c` | pg_stat_* SQL 函数读取 relation、index、function 共享统计。 |

阅读时不要按文件名背 API。

先找入口状态，再找运行期持有者，然后看 cleanup、fallback 和输出路径。

如果一个函数只是在表格中出现，而没有参与本节主问题，就不要把它扩展成新的主线。

## 4. 关键数据结构与状态边界

本节需要关注的不是单个字段名字，而是字段组合在生命周期中的语义。

### 4.1 `Relation.pgstat_enabled`

relcache 中表示该 relation 是否需要统计。

### 4.2 `Relation.pgstat_info`

relcache 和 PgStat_TableStatus pending entry 的互相链接。

### 4.3 `PgStat_TableStatus.counts`

表和索引对象的本地累计计数。

### 4.4 `PgStat_TableXactStatus`

表 DML 事务级计数，提交/回滚时合并语义不同。

### 4.5 `PgStat_FunctionCallUsage`

一次函数调用前后保存的计时状态。

### 4.6 `total_func_time`

backend 内函数调用总时间，用于区分 self time 与 nested function time。

### 4.7 `PgStat_StatTabEntry / PgStat_StatFuncEntry`

SQL 视图读取的共享统计结构。

贯穿本节的不变量是：观测状态不应该破坏被观测对象的执行语义。

## 5. 主流程源码 walkthrough

主流程按时间推进，而不是按文件归类。

```text
relation open
  ->
第一次产生统计事件
  ->
heap scan
  ->
index scan
  ->
heap fetch from index
  ->
DML
  ->
buffer hit/read
  ->
function call start
  ->
function call end
  ->
SQL view fetch
```

### 5.1 `relation open`

时间线推进到 `relation open` 时，关键变化是：pgstat_init_relation 判断 relkind 和 track_counts，设置 pgstat_enabled。

不是每次打开都立刻创建 shared stats 引用。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.2 `第一次产生统计事件`

时间线推进到 `第一次产生统计事件` 时，关键变化是：pgstat_should_count_relation 触发 pgstat_assoc_relation。

pending entry 延迟创建，避免只打开不使用的 relcache 成本。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.3 `heap scan`

时间线推进到 `heap scan` 时，关键变化是：heapam.c 在 scan begin 和 getnext 路径调用 pgstat_count_heap_scan/getnext。

pg_stat_all_tables 的 seq_scan、seq_tup_read 来自这里。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.4 `index scan`

时间线推进到 `index scan` 时，关键变化是：index AM 或具体 AM 调用 pgstat_count_index_scan 和 pgstat_count_index_tuples。

pg_stat_all_indexes 的 idx_scan、idx_tup_read 不是 executor 节点直接写的。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.5 `heap fetch from index`

时间线推进到 `heap fetch from index` 时，关键变化是：index_fetch_heap 或 table AM fetch 调用 pgstat_count_heap_fetch。

idx_tup_fetch 与回表可见性和 index-only scan 行为有关。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.6 `DML`

时间线推进到 `DML` 时，关键变化是：heap_insert/update/delete 调用 pgstat_count_heap_insert/update/delete。

这些先进入事务统计栈，再由 AtEOXact_PgStat_Relations 合并。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.7 `buffer hit/read`

时间线推进到 `buffer hit/read` 时，关键变化是：bufmgr 在 relation buffer 访问时调用 pgstat_count_buffer_hit/read。

表视图的 block 计数来自 buffer manager，不来自 Scan 节点本身。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.8 `function call start`

时间线推进到 `function call start` 时，关键变化是：executor 或 fmgr 调用 pgstat_init_function_usage。

track_functions 决定是否启用。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.9 `function call end`

时间线推进到 `function call end` 时，关键变化是：pgstat_end_function_usage 计算 total/self time 和 calls。

递归和嵌套函数需要 total_func_time 校正。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.10 `SQL view fetch`

时间线推进到 `SQL view fetch` 时，关键变化是：pgstatfuncs.c 从共享统计快照读取 entry。

用户看到的是对象维度累计。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

主流程结束后，用户看到的是输出或视图；源码中真正重要的是状态已经在哪个边界完成折叠。

## 6. 状态随时间推进的完整故事

把本节主问题压成一条对象生命周期，可以避免在函数清单里迷路。

阶段 1：创建或启用

观察 `Relation.pgstat_enabled` 在 `relation open` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 2：挂载或关联

观察 `Relation.pgstat_info` 在 `第一次产生统计事件` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 3：热路径累计

观察 `PgStat_TableStatus.counts` 在 `heap scan` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 4：周期折叠

观察 `PgStat_TableXactStatus` 在 `index scan` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 5：输出或刷新

观察 `PgStat_FunctionCallUsage` 在 `heap fetch from index` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 6：清理或失效

观察 `total_func_time` 在 `DML` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

完整故事的终点不是“函数返回”，而是用户可以稳定看到或推断这个状态。

## 7. 生命周期 / ownership / cleanup

1. relcache entry 生命周期内持有 pgstat_info 链接，unlink 时打断互相引用。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

2. relation pending stats 在 backend-local pending context 中累积，flush 后进入共享 entry。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

3. DML 事务状态在 TopTransactionContext 下，事务结束释放。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

4. function pending stats 按 function OID 创建 entry，flush 后更新共享 function stats。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

5. 对象 drop/create 会通过 transactional stats drop 逻辑维护共享 entry 生命周期。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

一个常见错误是把“能读到”误认为“拥有”。

执行器、统计系统和扩展 hook 中，这两者经常分离。

## 8. 正确性机制层次

正确性不是由单个计数器保证的。

层次 1：表统计和索引统计按对象累计，不按计划节点累计。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 2：seq_scan 来自 heap scan 入口，不代表某一次 EXPLAIN 中 Seq Scan 节点的 loops。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 3：idx_tup_read 表示索引返回的 index tuple 数，idx_tup_fetch 涉及回表取 heap tuple。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 4：n_tup_ins/upd/del 的事务语义不同于 live/dead tuple delta。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 5：function total_time 包含函数调用总耗时，self_time 会扣除嵌套函数已计入的时间。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

本节涉及的是观测正确性，不是事务隔离本身。

观测正确性的最低要求是：不要因为收集指标而改变执行语义。

## 9. 错误路径 / 异常路径 / fallback

异常路径 1：track_counts 关闭时 relation stats 不会启用。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 2：partitioned table 没有普通 storage，但仍可能参与统计边界。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 3：index-only scan 可能让 idx_tup_fetch 与 idx_tup_read 的关系不同于普通 index scan。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 4：set-returning function value-per-call 模式可能多次 init/end，但 finalize 控制用户意义上的 call。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 5：被并发 drop 的函数在 track_functions 打开时需要 AcceptInvalidationMessages 和 syscache 检查。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径的课程价值在于保护诊断结论。

看不到某个指标，不等于底层事件没有发生。

## 10. 成本、资源与跨模块传播

成本点 1：relation stats 延迟关联避免只打开 relation 就分配 pending entry。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 2：scan/getnext 计数在访问方法热路径上，因此宏必须非常轻。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 3：DML 事务计数增加了 TopTransactionContext 中的 per-table 栈状态。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 4：function timing 需要读时钟，track_functions=all 会增加函数调用密集 workload 成本。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 5：SQL 视图聚合时读取快照，成本和统计对象数量相关。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本分析必须问两个问题。

第一，事件发生在热路径还是收尾路径。

第二，成本是每 tuple、每 node、每 query、每 flush，还是每输出字段支付。

## 11. 观测与诊断入口

入口 1：执行 Seq Scan 后观察 pg_stat_all_tables.seq_scan 和 seq_tup_read。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 2：执行 Index Scan 后比较 pg_stat_all_indexes.idx_scan、idx_tup_read、idx_tup_fetch。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 3：执行 INSERT/UPDATE/DELETE 后提交，观察 n_tup_ins、n_tup_upd、n_tup_del。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 4：打开 track_functions=pl 或 all，调用函数后观察 pg_stat_user_functions。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 5：对 index-only scan 查询比较 Heap Fetches 与 idx_tup_fetch，理解指标边界。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

诊断闭环的顺序应当是：看到现象，定位状态，回到源码，再修正解释。

## 12. 课堂实验

实验 1：创建表和索引，运行 WHERE id 点查，查看表和索引统计变化。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 2：运行 SELECT count(*) 强制 Seq Scan，观察 seq_tup_read。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 3：运行 UPDATE 后 ROLLBACK，再观察表统计，区分 attempted action 和 live/dead delta。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 4：创建 PL/pgSQL 函数，设置 track_functions=pl，调用多次后查看 calls、total_time、self_time。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

课堂实验不追求压测结果。

它只验证一个 runtime 现象能否回到源码状态。

## 13. 常见误区

误区 1：不要把 pg_stat_all_tables 的 seq_scan 当成 EXPLAIN 的 Seq Scan 节点次数。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 2：不要把 idx_tup_read 和实际返回给客户端的 rows 混为一谈。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 3：不要把 function self_time 当成 SQL 语句总耗时。它只在函数统计边界内成立。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 4：不要忽略统计刷新延迟，刚执行完不一定立刻在视图中看到。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

大多数误读都不是因为字段名难，而是因为把两个生命周期不同的事实放在一起比较。

## 14. 源码复盘清单

复盘 1：从 `relation open` 回到 `src/include/pgstat.h`。

先确认 `Relation.pgstat_enabled` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：pgstat_init_relation 判断 relkind 和 track_counts，设置 pgstat_enabled。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：relation stats 延迟关联避免只打开 relation 就分配 pending entry。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 2：从 `第一次产生统计事件` 回到 `src/backend/utils/activity/pgstat_relation.c`。

先确认 `Relation.pgstat_info` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：pgstat_should_count_relation 触发 pgstat_assoc_relation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：scan/getnext 计数在访问方法热路径上，因此宏必须非常轻。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 3：从 `heap scan` 回到 `src/backend/access/heap/heapam.c`。

先确认 `PgStat_TableStatus.counts` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：heapam.c 在 scan begin 和 getnext 路径调用 pgstat_count_heap_scan/getnext。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：DML 事务计数增加了 TopTransactionContext 中的 per-table 栈状态。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 4：从 `index scan` 回到 `src/backend/access/heap/heapam_handler.c`。

先确认 `PgStat_TableXactStatus` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：index AM 或具体 AM 调用 pgstat_count_index_scan 和 pgstat_count_index_tuples。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：function timing 需要读时钟，track_functions=all 会增加函数调用密集 workload 成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 5：从 `heap fetch from index` 回到 `src/backend/access/index/indexam.c`。

先确认 `PgStat_FunctionCallUsage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：index_fetch_heap 或 table AM fetch 调用 pgstat_count_heap_fetch。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：SQL 视图聚合时读取快照，成本和统计对象数量相关。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 6：从 `DML` 回到 `src/backend/access/nbtree/nbtsearch.c`。

先确认 `total_func_time` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：heap_insert/update/delete 调用 pgstat_count_heap_insert/update/delete。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：relation stats 延迟关联避免只打开 relation 就分配 pending entry。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 7：从 `buffer hit/read` 回到 `src/backend/storage/buffer/bufmgr.c`。

先确认 `PgStat_StatTabEntry / PgStat_StatFuncEntry` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：bufmgr 在 relation buffer 访问时调用 pgstat_count_buffer_hit/read。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：scan/getnext 计数在访问方法热路径上，因此宏必须非常轻。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 8：从 `function call start` 回到 `src/backend/utils/activity/pgstat_function.c`。

先确认 `Relation.pgstat_enabled` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：executor 或 fmgr 调用 pgstat_init_function_usage。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：DML 事务计数增加了 TopTransactionContext 中的 per-table 栈状态。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 9：从 `function call end` 回到 `src/backend/executor/execExprInterp.c`。

先确认 `Relation.pgstat_info` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：pgstat_end_function_usage 计算 total/self time 和 calls。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：function timing 需要读时钟，track_functions=all 会增加函数调用密集 workload 成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 10：从 `SQL view fetch` 回到 `src/backend/executor/execSRF.c`。

先确认 `PgStat_TableStatus.counts` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：pgstatfuncs.c 从共享统计快照读取 entry。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：SQL 视图聚合时读取快照，成本和统计对象数量相关。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 11：从 `relation open` 回到 `src/backend/utils/fmgr/fmgr.c`。

先确认 `PgStat_TableXactStatus` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：pgstat_init_relation 判断 relkind 和 track_counts，设置 pgstat_enabled。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：relation stats 延迟关联避免只打开 relation 就分配 pending entry。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 12：从 `第一次产生统计事件` 回到 `src/backend/utils/adt/pgstatfuncs.c`。

先确认 `PgStat_FunctionCallUsage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：pgstat_should_count_relation 触发 pgstat_assoc_relation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：scan/getnext 计数在访问方法热路径上，因此宏必须非常轻。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 13：从 `heap scan` 回到 `src/include/pgstat.h`。

先确认 `total_func_time` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：heapam.c 在 scan begin 和 getnext 路径调用 pgstat_count_heap_scan/getnext。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：DML 事务计数增加了 TopTransactionContext 中的 per-table 栈状态。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 14：从 `index scan` 回到 `src/backend/utils/activity/pgstat_relation.c`。

先确认 `PgStat_StatTabEntry / PgStat_StatFuncEntry` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：index AM 或具体 AM 调用 pgstat_count_index_scan 和 pgstat_count_index_tuples。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：function timing 需要读时钟，track_functions=all 会增加函数调用密集 workload 成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 15：从 `heap fetch from index` 回到 `src/backend/access/heap/heapam.c`。

先确认 `Relation.pgstat_enabled` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：index_fetch_heap 或 table AM fetch 调用 pgstat_count_heap_fetch。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：SQL 视图聚合时读取快照，成本和统计对象数量相关。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 16：从 `DML` 回到 `src/backend/access/heap/heapam_handler.c`。

先确认 `Relation.pgstat_info` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：heap_insert/update/delete 调用 pgstat_count_heap_insert/update/delete。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：relation stats 延迟关联避免只打开 relation 就分配 pending entry。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 17：从 `buffer hit/read` 回到 `src/backend/access/index/indexam.c`。

先确认 `PgStat_TableStatus.counts` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：bufmgr 在 relation buffer 访问时调用 pgstat_count_buffer_hit/read。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：scan/getnext 计数在访问方法热路径上，因此宏必须非常轻。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 18：从 `function call start` 回到 `src/backend/access/nbtree/nbtsearch.c`。

先确认 `PgStat_TableXactStatus` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：executor 或 fmgr 调用 pgstat_init_function_usage。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：DML 事务计数增加了 TopTransactionContext 中的 per-table 栈状态。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 19：从 `function call end` 回到 `src/backend/storage/buffer/bufmgr.c`。

先确认 `PgStat_FunctionCallUsage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：pgstat_end_function_usage 计算 total/self time 和 calls。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：function timing 需要读时钟，track_functions=all 会增加函数调用密集 workload 成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 20：从 `SQL view fetch` 回到 `src/backend/utils/activity/pgstat_function.c`。

先确认 `total_func_time` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：pgstatfuncs.c 从共享统计快照读取 entry。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：SQL 视图聚合时读取快照，成本和统计对象数量相关。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 21：从 `relation open` 回到 `src/backend/executor/execExprInterp.c`。

先确认 `PgStat_StatTabEntry / PgStat_StatFuncEntry` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：pgstat_init_relation 判断 relkind 和 track_counts，设置 pgstat_enabled。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：relation stats 延迟关联避免只打开 relation 就分配 pending entry。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 22：从 `第一次产生统计事件` 回到 `src/backend/executor/execSRF.c`。

先确认 `Relation.pgstat_enabled` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：pgstat_should_count_relation 触发 pgstat_assoc_relation。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：scan/getnext 计数在访问方法热路径上，因此宏必须非常轻。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 23：从 `heap scan` 回到 `src/backend/utils/fmgr/fmgr.c`。

先确认 `Relation.pgstat_info` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：heapam.c 在 scan begin 和 getnext 路径调用 pgstat_count_heap_scan/getnext。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：DML 事务计数增加了 TopTransactionContext 中的 per-table 栈状态。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 24：从 `index scan` 回到 `src/backend/utils/adt/pgstatfuncs.c`。

先确认 `PgStat_TableStatus.counts` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：index AM 或具体 AM 调用 pgstat_count_index_scan 和 pgstat_count_index_tuples。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：function timing 需要读时钟，track_functions=all 会增加函数调用密集 workload 成本。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 25：从 `heap fetch from index` 回到 `src/include/pgstat.h`。

先确认 `PgStat_TableXactStatus` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：index_fetch_heap 或 table AM fetch 调用 pgstat_count_heap_fetch。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：SQL 视图聚合时读取快照，成本和统计对象数量相关。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 26：从 `DML` 回到 `src/backend/utils/activity/pgstat_relation.c`。

先确认 `PgStat_FunctionCallUsage` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：heap_insert/update/delete 调用 pgstat_count_heap_insert/update/delete。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：relation stats 延迟关联避免只打开 relation 就分配 pending entry。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 27：从 `buffer hit/read` 回到 `src/backend/access/heap/heapam.c`。

先确认 `total_func_time` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：bufmgr 在 relation buffer 访问时调用 pgstat_count_buffer_hit/read。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：scan/getnext 计数在访问方法热路径上，因此宏必须非常轻。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

## 15. 讨论题

1. 为什么对象维度统计应该在 access method 和 fmgr 层计数，而不是在 ExplainNode 中计数？

回答时必须引用至少一个第 3 节源码入口。

2. idx_tup_read 与 idx_tup_fetch 的差异能帮助定位哪些问题？

回答时必须引用至少一个第 3 节源码入口。

3. track_functions 默认关闭或较保守的原因是什么？

回答时必须引用至少一个第 3 节源码入口。

4. 对象统计、queryid 统计和节点级 EXPLAIN 三者怎样组合诊断慢 SQL？

回答时必须引用至少一个第 3 节源码入口。

讨论题的目标不是扩展范围，而是检查本节唯一主问题是否已经能独立解释。

## 16. 本节小结

表、索引、函数统计是对象维度的累计事实。

它们来自访问方法、buffer manager 和函数调用边界，不是 PlanState 节点字段。

把对象维度与节点维度分清，是 pg_stat 与 EXPLAIN 联合诊断的基本前提。

可以带走的可迁移规律是：

```text
先找到状态归属，再解释输出字段；先确认生命周期，再讨论成本和正确性。
```

下一节继续沿同一条主线，把这个边界推进到相邻观测机制。
