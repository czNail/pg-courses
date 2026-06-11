# PostgreSQL pg_stat 架构与 Backend 本地计数刷新

## 课程定位

前置知识：已经理解单次 EXPLAIN 如何在 executor 生命周期内收集节点级事实。

本节唯一主问题：

```text
执行器路径上的 tuple、block、function、transaction 等统计为什么先进入 backend-local pending 状态，再按事务边界或 flush 时机进入共享统计体系？
```

核心矛盾：统计系统要低成本记录高频事件，又要让 SQL 视图能跨 backend 查询累计结果；如果每次 tuple 或 block 事件都直接加共享计数，锁竞争和 cache line 抖动会吞掉执行器热路径。

学完后应能判断：能解释 pgstat_count_* 宏、pgStatPending、pgstat_report_stat、AtEOXact_PgStat 和共享统计 entry 之间的生命周期。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前几节讨论 EXPLAIN。

EXPLAIN 适合单次执行剖析。

pg_stat 体系适合跨语句、跨 backend 的累计观察。

本节是 pg_stat 的入口课。

重点不在所有视图字段。

重点在本地计数如何安全、低成本地刷新。

本节刻意只处理一个问题：观测状态如何在执行器或统计体系中找到稳定边界。

凡是不能帮助解释这个边界的函数和字段，都只在后续课程或源码练习中出现。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
执行路径把高频统计先累加到 backend-local pending entry 或 fixed pending counters，事务结束时处理 transactional stats，非事务内的 pgstat_report_stat 再按时间、force 和 nowait 策略把 pending 状态刷新到共享统计 entry。
```

这句话里有三层含义。

第一层是入口边界：用户看到的 SQL 选项、统计视图或扩展 hook，必须先被转换成执行期状态。

第二层是状态边界：执行路径不能在每个热函数里直接做复杂诊断，只能在少数明确对象上累计事实。

第三层是解释边界：输出或视图只是读取这些事实，不能把估算、累计、平均、延迟刷新混成一个语义。

本节后面的所有源码 walkthrough 都围绕这三层展开。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/utils/activity/pgstat.c` | pgstat_report_stat、pgstat_prep_pending_entry、pgstat_flush_pending_entries 管理 pending entry 和刷新节奏。 |
| 2 | `src/include/utils/pgstat_internal.h` | PgStat_EntryRef、pgStatPending、pgstat_report_fixed、kind callbacks 等内部状态。 |
| 3 | `src/include/pgstat.h` | pgstat_count_* 宏和公开统计接口。 |
| 4 | `src/backend/utils/activity/pgstat_xact.c` | AtEOXact_PgStat、AtEOSubXact_PgStat、AtPrepare_PgStat 处理事务统计边界。 |
| 5 | `src/backend/utils/activity/pgstat_relation.c` | relation pending stats、transactional tuple counters 和 flush callback。 |
| 6 | `src/backend/utils/activity/pgstat_database.c` | database stats、connection time、temp file、deadlock 等计数。 |
| 7 | `src/backend/utils/activity/pgstat_io.c` | I/O fixed stats 和 pgstat_report_fixed 触发。 |
| 8 | `src/backend/utils/activity/pgstat_backend.c` | backend-local I/O/WAL 统计刷新。 |
| 9 | `src/backend/utils/adt/pgstatfuncs.c` | SQL-callable pg_stat 函数从共享统计快照读取数据。 |

阅读时不要按文件名背 API。

先找入口状态，再找运行期持有者，然后看 cleanup、fallback 和输出路径。

如果一个函数只是在表格中出现，而没有参与本节主问题，就不要把它扩展成新的主线。

## 4. 关键数据结构与状态边界

本节需要关注的不是单个字段名字，而是字段组合在生命周期中的语义。

### 4.1 `pgStatPending`

backend-local dlist，保存需要刷新的 variable-numbered stats entry。

### 4.2 `pgStatPendingContext`

pending entry 的 MemoryContext，避免每个事件直接访问共享内存。

### 4.3 `PgStat_EntryRef.pending`

某个统计对象在当前 backend 的待刷新增量。

### 4.4 `pgstat_report_fixed`

fixed stats 有待刷新时设置的 backend-local 标志。

### 4.5 `PgStat_SubXactStatus`

事务和子事务内 relation stats 的栈状态。

### 4.6 `PgStat_TableStatus.counts`

relation 的本地累计计数，等待 flush 进入共享 entry。

### 4.7 `PgStatShared_*`

共享内存中的统计 entry，SQL 视图最终读取这里的快照。

贯穿本节的不变量是：观测状态不应该破坏被观测对象的执行语义。

## 5. 主流程源码 walkthrough

主流程按时间推进，而不是按文件归类。

```text
执行器或存储路径调用 pgstat_count_*
  ->
pgstat_prep_pending_entry()
  ->
事务内 DML 计数
  ->
AtEOXact_PgStat()
  ->
pgstat_report_stat()
  ->
pgstat_flush_pending_entries()
  ->
pgstat_report_fixed 分支
  ->
pgstatfuncs.c 读取
```

### 5.1 `执行器或存储路径调用 pgstat_count_*`

时间线推进到 `执行器或存储路径调用 pgstat_count_*` 时，关键变化是：例如 heap scan、heap insert、index scan、buffer read/hit。

高频事件先进入本 backend 的本地结构。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.2 `pgstat_prep_pending_entry()`

时间线推进到 `pgstat_prep_pending_entry()` 时，关键变化是：必要时创建 PgStat Pending context 和 pending entry。

统计对象按 kind、database、object id 定位。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.3 `事务内 DML 计数`

时间线推进到 `事务内 DML 计数` 时，关键变化是：pgstat_count_heap_insert/update/delete 先写 relation transaction stack。

提交和回滚对 live/dead/change 语义不同。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.4 `AtEOXact_PgStat()`

时间线推进到 `AtEOXact_PgStat()` 时，关键变化是：事务结束时合并或撤销 transactional stats，并清理 stats snapshot。

这一步仍是 backend-local 语义整理。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.5 `pgstat_report_stat()`

时间线推进到 `pgstat_report_stat()` 时，关键变化是：在非事务内按 force、PGSTAT_MIN_INTERVAL、PGSTAT_MAX_INTERVAL 决定是否刷新。

刷新不是每条 tuple 立即发生。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.6 `pgstat_flush_pending_entries()`

时间线推进到 `pgstat_flush_pending_entries()` 时，关键变化是：遍历 pgStatPending，调用 kind 的 flush_pending_cb。

每类统计自己知道如何把本地增量合入共享 entry。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.7 `pgstat_report_fixed 分支`

时间线推进到 `pgstat_report_fixed 分支` 时，关键变化是：固定数量统计调用 flush_static_cb。

I/O、WAL、lock 等 fixed stats 不一定走 pending entry list。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

### 5.8 `pgstatfuncs.c 读取`

时间线推进到 `pgstatfuncs.c 读取` 时，关键变化是：SQL 函数获取统计快照，再由系统视图展示。

用户看到的是已刷新的共享统计，不是 backend 内每个瞬间。

读源码时先确认这个函数是否真的改变状态。

如果它只是把状态传给下一层，就继续向下追。

如果它把状态从局部对象复制到共享对象，就标记这是 ownership 或可见性边界。

如果它只负责输出，不能反向假设它就是统计来源。

主流程结束后，用户看到的是输出或视图；源码中真正重要的是状态已经在哪个边界完成折叠。

## 6. 状态随时间推进的完整故事

把本节主问题压成一条对象生命周期，可以避免在函数清单里迷路。

阶段 1：创建或启用

观察 `pgStatPending` 在 `执行器或存储路径调用 pgstat_count_*` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 2：挂载或关联

观察 `pgStatPendingContext` 在 `pgstat_prep_pending_entry()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 3：热路径累计

观察 `PgStat_EntryRef.pending` 在 `事务内 DML 计数` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 4：周期折叠

观察 `pgstat_report_fixed` 在 `AtEOXact_PgStat()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 5：输出或刷新

观察 `PgStat_SubXactStatus` 在 `pgstat_report_stat()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

阶段 6：清理或失效

观察 `PgStat_TableStatus.counts` 在 `pgstat_flush_pending_entries()` 前后的变化。

这个阶段不要急着解释用户输出。

先确认状态是否已经产生，是否仍是本地状态，是否已经被折叠成共享或输出状态。

只有完成这个判断，后面的诊断结论才不会跨层。

完整故事的终点不是“函数返回”，而是用户可以稳定看到或推断这个状态。

## 7. 生命周期 / ownership / cleanup

1. pending entry 在当前 backend 内创建，成功 flush 后删除。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

2. transactional relation stats 分配在 TopTransactionContext，事务结束自动释放。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

3. 共享统计 entry 生命周期由对象创建、删除、reset 和 pgstat kind 管理。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

4. pgstat snapshot 会缓存读取结果，AtEOXact_PgStat 会清理当前 backend 的 stats snapshot。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

5. backend 退出时会强制 pgstat_report_stat(true)，尽量把本地统计推入共享体系。

这里要同时回答谁创建、谁持有、谁在正常路径读取、谁在异常路径兜底。

一个常见错误是把“能读到”误认为“拥有”。

执行器、统计系统和扩展 hook 中，这两者经常分离。

## 8. 正确性机制层次

正确性不是由单个计数器保证的。

层次 1：高频计数不直接锁共享 entry，是 pg_stat 可扩展性的核心。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 2：事务统计必须区分 attempted actions、commit 后 live/dead delta 和 abort 后死 tuple 语义。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 3：flush 只能在不处于事务时进行，避免把事务 stop time 和可见性边界混在一起。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 4：nowait flush 失败时 pending 会保留，并返回建议 idle timeout。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

层次 5：SQL 视图看到的统计可能滞后，不代表执行路径没有计数。

把这个层次和本节主流程对齐，才能知道它保证什么、不能保证什么。

本节涉及的是观测正确性，不是事务隔离本身。

观测正确性的最低要求是：不要因为收集指标而改变执行语义。

## 9. 错误路径 / 异常路径 / fallback

异常路径 1：force=true 可以绕过最小间隔，shutdown、测试和长时间 pending 会使用强制刷新。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 2：nowait=true 时拿不到统计 entry 锁会 partial flush，避免诊断系统阻塞执行路径。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 3：track_counts 关闭时 relation 统计不会启用。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 4：对象 drop/create 的统计要通过 transactional drop 逻辑保证提交/回滚语义。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径 5：parallel worker 的统计可能需要特殊汇总，不能简单视作普通 backend top-level 语句。

遇到这个路径时，先判断状态是缺失、延迟、被聚合，还是被有意隐藏。

异常路径的课程价值在于保护诊断结论。

看不到某个指标，不等于底层事件没有发生。

## 10. 成本、资源与跨模块传播

成本点 1：本地 pending 累加避免了每个 tuple 对共享锁的竞争。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 2：flush 把成本批量化，但引入 pg_stat 视图延迟。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 3：pending entry 的 hash/ref 管理有内存成本，但比共享内存高频原子更新更可控。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 4：PGSTAT_MIN_INTERVAL 限制过于频繁刷新，保护共享统计锁。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本点 5：PGSTAT_MAX_INTERVAL 防止 pending 长时间滞留，保护可观测性。

它对应的跨模块传播要从执行器边界继续追到存储、WAL、JIT、MemoryContext、统计共享状态或扩展 hook。

成本分析必须问两个问题。

第一，事件发生在热路径还是收尾路径。

第二，成本是每 tuple、每 node、每 query、每 flush，还是每输出字段支付。

## 11. 观测与诊断入口

入口 1：对表执行 DML 后，在同一事务内和提交后查询 pg_stat_all_tables，观察统计刷新差异。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 2：调用 pg_stat_force_next_flush 或等待刷新间隔，可以验证视图滞后。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 3：使用 pg_stat_reset 后观察 shared stats reset scope。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 4：在 gdb 中断 pgstat_prep_pending_entry，观察 pending entry 创建。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

入口 5：在 gdb 中断 pgstat_report_stat，观察 force、nowait、pending_since 和 last_flush。

观察结果出来后，要回到第 3 节的源码入口确认它的来源。

诊断闭环的顺序应当是：看到现象，定位状态，回到源码，再修正解释。

## 12. 课堂实验

实验 1：创建表后执行 INSERT/UPDATE/DELETE，提交前后观察 n_tup_ins、n_tup_upd、n_tup_del。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 2：在事务中 ROLLBACK，比较 inserted tuples 和 dead/live delta 的解释。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 3：执行大量 SELECT 后观察 seq_scan、seq_tup_read 刷新是否有延迟。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

实验 4：修改 track_counts 后重启或重新连接，验证 relation stats 是否启用。

实验前先写下你预期哪个状态会变化。

实验后再用源码解释为什么变化发生在这个边界，而不是相邻模块。

课堂实验不追求压测结果。

它只验证一个 runtime 现象能否回到源码状态。

## 13. 常见误区

误区 1：不要把 pg_stat 视图当成单条 SQL 的实时 trace。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 2：不要以为每个 pgstat_count_* 都直接更新共享内存。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 3：不要把统计滞后误判成执行器没有扫描或没有写入。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

误区 4：不要把 committed tuple 变化和 attempted DML 计数混为一谈。

修正这个误区的方法是重新确认粒度：节点、查询、对象、backend、本地 pending、共享 entry 或输出格式。

大多数误读都不是因为字段名难，而是因为把两个生命周期不同的事实放在一起比较。

## 14. 源码复盘清单

复盘 1：从 `执行器或存储路径调用 pgstat_count_*` 回到 `src/backend/utils/activity/pgstat.c`。

先确认 `pgStatPending` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：例如 heap scan、heap insert、index scan、buffer read/hit。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：本地 pending 累加避免了每个 tuple 对共享锁的竞争。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 2：从 `pgstat_prep_pending_entry()` 回到 `src/include/utils/pgstat_internal.h`。

先确认 `pgStatPendingContext` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：必要时创建 PgStat Pending context 和 pending entry。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：flush 把成本批量化，但引入 pg_stat 视图延迟。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 3：从 `事务内 DML 计数` 回到 `src/include/pgstat.h`。

先确认 `PgStat_EntryRef.pending` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：pgstat_count_heap_insert/update/delete 先写 relation transaction stack。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：pending entry 的 hash/ref 管理有内存成本，但比共享内存高频原子更新更可控。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 4：从 `AtEOXact_PgStat()` 回到 `src/backend/utils/activity/pgstat_xact.c`。

先确认 `pgstat_report_fixed` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：事务结束时合并或撤销 transactional stats，并清理 stats snapshot。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：PGSTAT_MIN_INTERVAL 限制过于频繁刷新，保护共享统计锁。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 5：从 `pgstat_report_stat()` 回到 `src/backend/utils/activity/pgstat_relation.c`。

先确认 `PgStat_SubXactStatus` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在非事务内按 force、PGSTAT_MIN_INTERVAL、PGSTAT_MAX_INTERVAL 决定是否刷新。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：PGSTAT_MAX_INTERVAL 防止 pending 长时间滞留，保护可观测性。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 6：从 `pgstat_flush_pending_entries()` 回到 `src/backend/utils/activity/pgstat_database.c`。

先确认 `PgStat_TableStatus.counts` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：遍历 pgStatPending，调用 kind 的 flush_pending_cb。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：本地 pending 累加避免了每个 tuple 对共享锁的竞争。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 7：从 `pgstat_report_fixed 分支` 回到 `src/backend/utils/activity/pgstat_io.c`。

先确认 `PgStatShared_*` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：固定数量统计调用 flush_static_cb。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：flush 把成本批量化，但引入 pg_stat 视图延迟。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 8：从 `pgstatfuncs.c 读取` 回到 `src/backend/utils/activity/pgstat_backend.c`。

先确认 `pgStatPending` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：SQL 函数获取统计快照，再由系统视图展示。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：pending entry 的 hash/ref 管理有内存成本，但比共享内存高频原子更新更可控。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 9：从 `执行器或存储路径调用 pgstat_count_*` 回到 `src/backend/utils/adt/pgstatfuncs.c`。

先确认 `pgStatPendingContext` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：例如 heap scan、heap insert、index scan、buffer read/hit。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：PGSTAT_MIN_INTERVAL 限制过于频繁刷新，保护共享统计锁。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 10：从 `pgstat_prep_pending_entry()` 回到 `src/backend/utils/activity/pgstat.c`。

先确认 `PgStat_EntryRef.pending` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：必要时创建 PgStat Pending context 和 pending entry。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：PGSTAT_MAX_INTERVAL 防止 pending 长时间滞留，保护可观测性。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 11：从 `事务内 DML 计数` 回到 `src/include/utils/pgstat_internal.h`。

先确认 `pgstat_report_fixed` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：pgstat_count_heap_insert/update/delete 先写 relation transaction stack。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：本地 pending 累加避免了每个 tuple 对共享锁的竞争。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 12：从 `AtEOXact_PgStat()` 回到 `src/include/pgstat.h`。

先确认 `PgStat_SubXactStatus` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：事务结束时合并或撤销 transactional stats，并清理 stats snapshot。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：flush 把成本批量化，但引入 pg_stat 视图延迟。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 13：从 `pgstat_report_stat()` 回到 `src/backend/utils/activity/pgstat_xact.c`。

先确认 `PgStat_TableStatus.counts` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在非事务内按 force、PGSTAT_MIN_INTERVAL、PGSTAT_MAX_INTERVAL 决定是否刷新。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：pending entry 的 hash/ref 管理有内存成本，但比共享内存高频原子更新更可控。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 14：从 `pgstat_flush_pending_entries()` 回到 `src/backend/utils/activity/pgstat_relation.c`。

先确认 `PgStatShared_*` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：遍历 pgStatPending，调用 kind 的 flush_pending_cb。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：PGSTAT_MIN_INTERVAL 限制过于频繁刷新，保护共享统计锁。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 15：从 `pgstat_report_fixed 分支` 回到 `src/backend/utils/activity/pgstat_database.c`。

先确认 `pgStatPending` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：固定数量统计调用 flush_static_cb。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：PGSTAT_MAX_INTERVAL 防止 pending 长时间滞留，保护可观测性。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 16：从 `pgstatfuncs.c 读取` 回到 `src/backend/utils/activity/pgstat_io.c`。

先确认 `pgStatPendingContext` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：SQL 函数获取统计快照，再由系统视图展示。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：本地 pending 累加避免了每个 tuple 对共享锁的竞争。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 17：从 `执行器或存储路径调用 pgstat_count_*` 回到 `src/backend/utils/activity/pgstat_backend.c`。

先确认 `PgStat_EntryRef.pending` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：例如 heap scan、heap insert、index scan、buffer read/hit。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：flush 把成本批量化，但引入 pg_stat 视图延迟。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 18：从 `pgstat_prep_pending_entry()` 回到 `src/backend/utils/adt/pgstatfuncs.c`。

先确认 `pgstat_report_fixed` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：必要时创建 PgStat Pending context 和 pending entry。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：pending entry 的 hash/ref 管理有内存成本，但比共享内存高频原子更新更可控。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 19：从 `事务内 DML 计数` 回到 `src/backend/utils/activity/pgstat.c`。

先确认 `PgStat_SubXactStatus` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：pgstat_count_heap_insert/update/delete 先写 relation transaction stack。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：PGSTAT_MIN_INTERVAL 限制过于频繁刷新，保护共享统计锁。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 20：从 `AtEOXact_PgStat()` 回到 `src/include/utils/pgstat_internal.h`。

先确认 `PgStat_TableStatus.counts` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：事务结束时合并或撤销 transactional stats，并清理 stats snapshot。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：PGSTAT_MAX_INTERVAL 防止 pending 长时间滞留，保护可观测性。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 21：从 `pgstat_report_stat()` 回到 `src/include/pgstat.h`。

先确认 `PgStatShared_*` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在非事务内按 force、PGSTAT_MIN_INTERVAL、PGSTAT_MAX_INTERVAL 决定是否刷新。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：本地 pending 累加避免了每个 tuple 对共享锁的竞争。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 22：从 `pgstat_flush_pending_entries()` 回到 `src/backend/utils/activity/pgstat_xact.c`。

先确认 `pgStatPending` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：遍历 pgStatPending，调用 kind 的 flush_pending_cb。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：flush 把成本批量化，但引入 pg_stat 视图延迟。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 23：从 `pgstat_report_fixed 分支` 回到 `src/backend/utils/activity/pgstat_relation.c`。

先确认 `pgStatPendingContext` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：固定数量统计调用 flush_static_cb。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：pending entry 的 hash/ref 管理有内存成本，但比共享内存高频原子更新更可控。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 24：从 `pgstatfuncs.c 读取` 回到 `src/backend/utils/activity/pgstat_database.c`。

先确认 `PgStat_EntryRef.pending` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：SQL 函数获取统计快照，再由系统视图展示。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：PGSTAT_MIN_INTERVAL 限制过于频繁刷新，保护共享统计锁。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 25：从 `执行器或存储路径调用 pgstat_count_*` 回到 `src/backend/utils/activity/pgstat_io.c`。

先确认 `pgstat_report_fixed` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：例如 heap scan、heap insert、index scan、buffer read/hit。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：PGSTAT_MAX_INTERVAL 防止 pending 长时间滞留，保护可观测性。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 26：从 `pgstat_prep_pending_entry()` 回到 `src/backend/utils/activity/pgstat_backend.c`。

先确认 `PgStat_SubXactStatus` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：必要时创建 PgStat Pending context 和 pending entry。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：本地 pending 累加避免了每个 tuple 对共享锁的竞争。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 27：从 `事务内 DML 计数` 回到 `src/backend/utils/adt/pgstatfuncs.c`。

先确认 `PgStat_TableStatus.counts` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：pgstat_count_heap_insert/update/delete 先写 relation transaction stack。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：flush 把成本批量化，但引入 pg_stat 视图延迟。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 28：从 `AtEOXact_PgStat()` 回到 `src/backend/utils/activity/pgstat.c`。

先确认 `PgStatShared_*` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：事务结束时合并或撤销 transactional stats，并清理 stats snapshot。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：pending entry 的 hash/ref 管理有内存成本，但比共享内存高频原子更新更可控。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 29：从 `pgstat_report_stat()` 回到 `src/include/utils/pgstat_internal.h`。

先确认 `pgStatPending` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：在非事务内按 force、PGSTAT_MIN_INTERVAL、PGSTAT_MAX_INTERVAL 决定是否刷新。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：PGSTAT_MIN_INTERVAL 限制过于频繁刷新，保护共享统计锁。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

复盘 30：从 `pgstat_flush_pending_entries()` 回到 `src/include/pgstat.h`。

先确认 `pgStatPendingContext` 在这一段源码中是被创建、被读取、被累计，还是仅被传递。

再解释用户能看到的现象是否真的来自：遍历 pgStatPending，调用 kind 的 flush_pending_cb。

如果解释需要跨模块，优先沿着第 3 节的阅读顺序继续追，而不是跳到无关 API。

成本判断也要具体到这一段：PGSTAT_MAX_INTERVAL 防止 pending 长时间滞留，保护可观测性。

最后问一句：这个结论是单次执行事实、累计事实，还是输出格式造成的呈现方式。

## 15. 讨论题

1. 为什么 pg_stat 选择本地 pending 加批量 flush，而不是共享原子计数？

回答时必须引用至少一个第 3 节源码入口。

2. 统计视图滞后对性能诊断有哪些好处和坏处？

回答时必须引用至少一个第 3 节源码入口。

3. relation stats 为什么需要事务栈，而 buffer hit/read 这类计数不需要同样语义？

回答时必须引用至少一个第 3 节源码入口。

4. 如果每秒刷新一次仍嫌慢，应该改系统还是换诊断工具？

回答时必须引用至少一个第 3 节源码入口。

讨论题的目标不是扩展范围，而是检查本节唯一主问题是否已经能独立解释。

## 16. 本节小结

pg_stat 的核心不是视图字段，而是本地高频计数和共享低频刷新的分层。

这种分层用短期不实时换取低热路径成本和跨 backend 可查询性。

理解 pending、事务边界和 flush 后，才能正确解读 pg_stat 的延迟与累计语义。

可以带走的可迁移规律是：

```text
先找到状态归属，再解释输出字段；先确认生命周期，再讨论成本和正确性。
```

下一节继续沿同一条主线，把这个边界推进到相邻观测机制。
