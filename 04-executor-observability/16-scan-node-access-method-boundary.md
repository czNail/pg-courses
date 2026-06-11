# PostgreSQL Scan 节点与 table AM / index AM 访问边界

## 课程定位

前置知识：熟悉 MVCC
snapshot、TupleTableSlot、ExecScan、SeqScan、IndexScan
和 table/index access method 的基本概念。

本节唯一主问题：

```text
扫描节点如何把 MVCC 可见性、table AM / index AM、qual 过滤和 slot 输出串起来，同时避免执行器直接理解 heap page 细节？
```

核心矛盾：执行器要统一驱动
SeqScan、IndexScan、BitmapHeapScan、ForeignScan 等节点；但
tuple 获取、可见性、页格式、索引回表和 recheck 属于访问方法或存储层。

学完后应能判断本节状态应该挂在哪个 owner 下、在哪个边界失效、哪个观测入口能看到它的影响。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

04 目录的主线是从 Plan 进入 PlanState，再进入 tuple
流、表达式状态、内存生命周期和可观测性。

本节只回答一个问题：扫描节点如何把 MVCC 可见性、table AM / index
AM、qual 过滤和 slot 输出串起来，同时避免执行器直接理解 heap page 细节？

不要把本节写成函数百科。所有源码入口都要服务同一条线：问题 -> 状态 -> 主流程 -> 边界
-> 异常 -> 诊断。

前面的课程提供 executor 调度框架；本节把某个具体 runtime
状态放到这个框架里，说明它如何随时间推进。

后续课程会继续复用同一读法，只是主对象从当前状态换成其它执行节点、观测指标或扩展 hook。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
scan PlanState 用 ExecScanExtended 统一 reset、qual 和 projection
accessMtd 返回下一个候选 slot
table AM / index AM 隐藏存储细节
executor 只消费 slot 和 recheck 语义
```

这里的 tension 可以压缩成：

```text
统一 executor 协议和低 hot path 成本
  vs
运行时状态、生命周期、异常 cleanup 和观测口径必须足够精确
```

PostgreSQL
的选择不是把所有细节推给一个万能对象，而是让少数状态承担清晰边界。读源码时要先找到这些边界。

如果一个字段、函数或指标不能解释本节唯一主问题，就不要把它展开成并列主题。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | src/include/executor/execScan.h | `ExecScanFetch()`、`ExecScanExtended()`。 |
| 2 | src/backend/executor/nodeSeqscan.c | `SeqNext()`、`ExecSeqScan()`、`ExecInitSeqScan()`。 |
| 3 | src/backend/executor/nodeIndexscan.c | `IndexNext()`、`ExecIndexScan()`、`IndexRecheck()`。 |
| 4 | src/include/access/tableam.h | `table_beginscan()`、`table_scan_getnextslot()`。 |
| 5 | src/backend/access/table/tableam.c | `table_beginscan_common()` 构造 TableScanDesc。 |
| 6 | src/backend/access/heap/heapam.c | `heap_getnextslot()` 填充 heap scan slot。 |
| 7 | src/backend/access/index/indexam.c | `index_beginscan()`、`index_getnext_slot()`。 |
| 8 | src/include/access/amapi.h | `IndexAmRoutine` 定义 index AM 回调边界。 |

阅读顺序按 mental model 展开：入口、状态结构、ownership、hot
path、reset/rescan、cleanup、观测。

如果本地其它版本函数拆分不同，优先寻找相同语义的入口，而不是按函数名硬匹配。

## 4. 关键数据结构与状态边界

| 状态 | 本节语义 |
| --- | --- |
| ScanState | 和 owner、生命周期、失效点一起解释，不能孤立读取。 |
| TableScanDesc | 和 owner、生命周期、失效点一起解释，不能孤立读取。 |
| IndexScanDesc | 和 owner、生命周期、失效点一起解释，不能孤立读取。 |
| Snapshot | 和 owner、生命周期、失效点一起解释，不能孤立读取。 |
| ss_ScanTupleSlot | 和 owner、生命周期、失效点一起解释，不能孤立读取。 |
| ps_ProjInfo | 和 owner、生命周期、失效点一起解释，不能孤立读取。 |

`ScanState`
的语义来自它被谁创建、谁更新、谁读取、何时失效。只看到字段值，还不能判断它是否仍然代表当前
tuple 或当前执行阶段。

诊断 `ScanState` 时要同时记录所在 memory context 或上层
owner。executor 中大量 bug 和误判都来自把裸指针当成长期语义。

`TableScanDesc`
的语义来自它被谁创建、谁更新、谁读取、何时失效。只看到字段值，还不能判断它是否仍然代表当前
tuple 或当前执行阶段。

诊断 `TableScanDesc` 时要同时记录所在 memory context 或上层
owner。executor 中大量 bug 和误判都来自把裸指针当成长期语义。

`IndexScanDesc`
的语义来自它被谁创建、谁更新、谁读取、何时失效。只看到字段值，还不能判断它是否仍然代表当前
tuple 或当前执行阶段。

诊断 `IndexScanDesc` 时要同时记录所在 memory context 或上层
owner。executor 中大量 bug 和误判都来自把裸指针当成长期语义。

`Snapshot`
的语义来自它被谁创建、谁更新、谁读取、何时失效。只看到字段值，还不能判断它是否仍然代表当前
tuple 或当前执行阶段。

诊断 `Snapshot` 时要同时记录所在 memory context 或上层
owner。executor 中大量 bug 和误判都来自把裸指针当成长期语义。

`ss_ScanTupleSlot`
的语义来自它被谁创建、谁更新、谁读取、何时失效。只看到字段值，还不能判断它是否仍然代表当前
tuple 或当前执行阶段。

诊断 `ss_ScanTupleSlot` 时要同时记录所在 memory context 或上层
owner。executor 中大量 bug 和误判都来自把裸指针当成长期语义。

`ps_ProjInfo`
的语义来自它被谁创建、谁更新、谁读取、何时失效。只看到字段值，还不能判断它是否仍然代表当前
tuple 或当前执行阶段。

诊断 `ps_ProjInfo` 时要同时记录所在 memory context 或上层
owner。executor 中大量 bug 和误判都来自把裸指针当成长期语义。

这些状态大多是 backend-local。并行执行中不能把普通指针直接交给 worker，必须通过
DSM、序列化或 worker 侧重建。

## 5. 主流程源码 walkthrough

```text
ExecInitSeqScan  -> 改变或消费本节主状态
ExecSeqScan  -> 改变或消费本节主状态
SeqNext  -> 改变或消费本节主状态
ExecScanExtended  -> 改变或消费本节主状态
ExecInitIndexScan  -> 改变或消费本节主状态
IndexNext  -> 改变或消费本节主状态
table_scan_getnextslot  -> 改变或消费本节主状态
index_getnext_slot  -> 改变或消费本节主状态
```

第 1 步看
`ExecInitSeqScan`：先确认函数入口前状态是否已经初始化，再确认函数内部是否绑定、清理、重置或转交状态。

离开 `ExecInitSeqScan` 前要问调用者能依赖什么：一个有效
slot、一个参数值、一个 context、一个 scan
descriptor、一个已格式化指标，还是一个需要 recheck 的候选结果。

如果 `ExecInitSeqScan` 是 hot
path，调试时用条件断点和计数，不要盲目逐步执行。executor 的调用栈很深，关键是状态边界。

第 2 步看
`ExecSeqScan`：先确认函数入口前状态是否已经初始化，再确认函数内部是否绑定、清理、重置或转交状态。

离开 `ExecSeqScan` 前要问调用者能依赖什么：一个有效 slot、一个参数值、一个
context、一个 scan descriptor、一个已格式化指标，还是一个需要 recheck
的候选结果。

如果 `ExecSeqScan` 是 hot
path，调试时用条件断点和计数，不要盲目逐步执行。executor 的调用栈很深，关键是状态边界。

第 3 步看
`SeqNext`：先确认函数入口前状态是否已经初始化，再确认函数内部是否绑定、清理、重置或转交状态。

离开 `SeqNext` 前要问调用者能依赖什么：一个有效 slot、一个参数值、一个
context、一个 scan descriptor、一个已格式化指标，还是一个需要 recheck
的候选结果。

如果 `SeqNext` 是 hot
path，调试时用条件断点和计数，不要盲目逐步执行。executor 的调用栈很深，关键是状态边界。

第 4 步看
`ExecScanExtended`：先确认函数入口前状态是否已经初始化，再确认函数内部是否绑定、清理、重置或转交状态。

离开 `ExecScanExtended` 前要问调用者能依赖什么：一个有效
slot、一个参数值、一个 context、一个 scan
descriptor、一个已格式化指标，还是一个需要 recheck 的候选结果。

如果 `ExecScanExtended` 是 hot
path，调试时用条件断点和计数，不要盲目逐步执行。executor 的调用栈很深，关键是状态边界。

第 5 步看
`ExecInitIndexScan`：先确认函数入口前状态是否已经初始化，再确认函数内部是否绑定、清理、重置或转交状态。

离开 `ExecInitIndexScan` 前要问调用者能依赖什么：一个有效
slot、一个参数值、一个 context、一个 scan
descriptor、一个已格式化指标，还是一个需要 recheck 的候选结果。

如果 `ExecInitIndexScan` 是 hot
path，调试时用条件断点和计数，不要盲目逐步执行。executor 的调用栈很深，关键是状态边界。

第 6 步看
`IndexNext`：先确认函数入口前状态是否已经初始化，再确认函数内部是否绑定、清理、重置或转交状态。

离开 `IndexNext` 前要问调用者能依赖什么：一个有效 slot、一个参数值、一个
context、一个 scan descriptor、一个已格式化指标，还是一个需要 recheck
的候选结果。

如果 `IndexNext` 是 hot
path，调试时用条件断点和计数，不要盲目逐步执行。executor 的调用栈很深，关键是状态边界。

第 7 步看
`table_scan_getnextslot`：先确认函数入口前状态是否已经初始化，再确认函数内部是否绑定、清理、重置或转交状态。

离开 `table_scan_getnextslot` 前要问调用者能依赖什么：一个有效
slot、一个参数值、一个 context、一个 scan
descriptor、一个已格式化指标，还是一个需要 recheck 的候选结果。

如果 `table_scan_getnextslot` 是 hot
path，调试时用条件断点和计数，不要盲目逐步执行。executor 的调用栈很深，关键是状态边界。

第 8 步看
`index_getnext_slot`：先确认函数入口前状态是否已经初始化，再确认函数内部是否绑定、清理、重置或转交状态。

离开 `index_getnext_slot` 前要问调用者能依赖什么：一个有效
slot、一个参数值、一个 context、一个 scan
descriptor、一个已格式化指标，还是一个需要 recheck 的候选结果。

如果 `index_getnext_slot` 是 hot
path，调试时用条件断点和计数，不要盲目逐步执行。executor 的调用栈很深，关键是状态边界。

主流程是本节所有其它章节的骨架。正确性、异常、成本和观测都要回扣这条链路。

## 6. 状态随时间推进的完整故事

初始化阶段：围绕本节主状态，确认 owner 是否已经建立，状态是否仍有效，是否需要
reset/rescan/materialize/recheck/cleanup。

初始化阶段 中最容易误读的是“指针还在”就等于“语义还在”。PostgreSQL executor
经常通过 reset、clear、callback 或 descriptor 变化表达失效。

第一次执行阶段：围绕本节主状态，确认 owner 是否已经建立，状态是否仍有效，是否需要
reset/rescan/materialize/recheck/cleanup。

第一次执行阶段 中最容易误读的是“指针还在”就等于“语义还在”。PostgreSQL
executor 经常通过 reset、clear、callback 或 descriptor
变化表达失效。

每 tuple 周期：围绕本节主状态，确认 owner 是否已经建立，状态是否仍有效，是否需要
reset/rescan/materialize/recheck/cleanup。

每 tuple 周期 中最容易误读的是“指针还在”就等于“语义还在”。PostgreSQL
executor 经常通过 reset、clear、callback 或 descriptor
变化表达失效。

参数变化或 rescan 阶段：围绕本节主状态，确认 owner
是否已经建立，状态是否仍有效，是否需要
reset/rescan/materialize/recheck/cleanup。

参数变化或 rescan 阶段 中最容易误读的是“指针还在”就等于“语义还在”。PostgreSQL
executor 经常通过 reset、clear、callback 或 descriptor
变化表达失效。

正常结束阶段：围绕本节主状态，确认 owner 是否已经建立，状态是否仍有效，是否需要
reset/rescan/materialize/recheck/cleanup。

正常结束阶段 中最容易误读的是“指针还在”就等于“语义还在”。PostgreSQL executor
经常通过 reset、clear、callback 或 descriptor 变化表达失效。

ERROR 或提前退出阶段：围绕本节主状态，确认 owner 是否已经建立，状态是否仍有效，是否需要
reset/rescan/materialize/recheck/cleanup。

ERROR 或提前退出阶段 中最容易误读的是“指针还在”就等于“语义还在”。PostgreSQL
executor 经常通过 reset、clear、callback 或 descriptor
变化表达失效。

诊断阶段：围绕本节主状态，确认 owner 是否已经建立，状态是否仍有效，是否需要
reset/rescan/materialize/recheck/cleanup。

诊断阶段 中最容易误读的是“指针还在”就等于“语义还在”。PostgreSQL executor
经常通过 reset、clear、callback 或 descriptor 变化表达失效。

完整故事的价值是可以从任何 runtime 现象切入，再判断它属于初始化错误、hot path
成本、rescan 失效还是 cleanup 遗漏。

## 7. 生命周期 / ownership / cleanup

谁创建这个状态，谁持有它，谁在正常路径释放它，谁在 ERROR 或 abort 后兜底。

把这句话落到本节，就是要能在源码里指出具体 owner 和 cleanup
函数，而不是只说“随查询结束释放”。

MemoryContext 只负责内存；slot、descriptor、scan
desc、tuplestore、callback 和 relation refcount
还有额外语义。

把这句话落到本节，就是要能在源码里指出具体 owner 和 cleanup
函数，而不是只说“随查询结束释放”。

per-query、per-tuple、节点私有 context 和 portal context
的边界必须分开记录。

把这句话落到本节，就是要能在源码里指出具体 owner 和 cleanup
函数，而不是只说“随查询结束释放”。

如果一个对象能跨 tuple 保存，就必须说明它为什么不会被下一次 reset 破坏。

把这句话落到本节，就是要能在源码里指出具体 owner 和 cleanup
函数，而不是只说“随查询结束释放”。

如果一个对象持有外部资源，删除 parent context 之前必须能找到显式 cleanup。

把这句话落到本节，就是要能在源码里指出具体 owner 和 cleanup
函数，而不是只说“随查询结束释放”。

正常路径可以依赖 MemoryContext 批量释放内存；但只要对象持有非内存资源，就必须在
context 删除前显式结束。

## 8. 正确性机制层次

raw field 不是语义；field 加 owner、flag、lifetime、lock 或
visibility context 才是语义。

这个正确性点不是抽象原则，而是保护本节主流程中某个状态不会被错误读取、错误保留或错误暴露。

hot path 上的 fast path 不能绕过
descriptor、parameter、reset、recheck 或 cleanup 边界。

这个正确性点不是抽象原则，而是保护本节主流程中某个状态不会被错误读取、错误保留或错误暴露。

rescan 后仍可复用的状态必须独立于旧参数，否则应该失效或重建。

这个正确性点不是抽象原则，而是保护本节主流程中某个状态不会被错误读取、错误保留或错误暴露。

virtual tuple、byref Datum 和 callback
资源都需要单独说明生命周期。

这个正确性点不是抽象原则，而是保护本节主流程中某个状态不会被错误读取、错误保留或错误暴露。

观测指标只能证明某个切面，不能代替源码中的状态变化。

这个正确性点不是抽象原则，而是保护本节主流程中某个状态不会被错误读取、错误保留或错误暴露。

如果新增代码改变了状态生命周期，就必须同时更新异常路径、rescan 行为和诊断口径。

## 9. 错误路径 / 异常路径 / fallback

常见路径通常保留低成本 fast path，但必须和通用路径返回同一语义。

fallback 的意义是在不改变语义边界的前提下降低常见路径成本，或者在异常时保护状态不泄漏。

异常路径先保证状态不泄漏、不被错误复用，再讨论性能。

fallback 的意义是在不改变语义边界的前提下降低常见路径成本，或者在异常时保护状态不泄漏。

ERROR longjmp 让普通 C 栈 cleanup 不可靠，因此状态必须挂在可统一清理的
owner 下。

fallback 的意义是在不改变语义边界的前提下降低常见路径成本，或者在异常时保护状态不泄漏。

当无法直接观测内部状态时，使用更小 SQL、条件断点或内存 context 快照构造间接证据。

fallback 的意义是在不改变语义边界的前提下降低常见路径成本，或者在异常时保护状态不泄漏。

版本函数拆分可能变化，但入口、状态、失效和 cleanup 的读法不应变化。

fallback 的意义是在不改变语义边界的前提下降低常见路径成本，或者在异常时保护状态不泄漏。

阅读异常路径时优先找
`elog(ERROR)`、`ereport(ERROR)`、`Assert`、`if (state
== NULL)`、`clear`、`end`、`rescan`。

## 10. 成本、资源与跨模块传播

成本可能随 rows、columns、loops、work_mem、descriptor
宽度、scan pages 或参数变化放大。

成本传播要回到本节主状态：它是在 per-tuple 中增长，还是在
per-query、节点私有对象、AM 层或 instrumentation 中增长。

CPU、内存、临时文件、buffer pin、relation refcount、JIT 和
instrumentation 可能同时参与传播。

成本传播要回到本节主状态：它是在 per-tuple 中增长，还是在
per-query、节点私有对象、AM 层或 instrumentation 中增长。

EXPLAIN 中的单个数字不能代表所有 backend、worker 或所有生命周期阶段。

成本传播要回到本节主状态：它是在 per-tuple 中增长，还是在
per-query、节点私有对象、AM 层或 instrumentation 中增长。

优化前先确认瓶颈属于 planner 选择、executor 状态、storage AM 还是
workload shape。

成本传播要回到本节主状态：它是在 per-tuple 中增长，还是在
per-query、节点私有对象、AM 层或 instrumentation 中增长。

扩展插桩也有成本，尤其是 hot path 上的计时、字符串构造和内存分配。

成本传播要回到本节主状态：它是在 per-tuple 中增长，还是在
per-query、节点私有对象、AM 层或 instrumentation 中增长。

定位成本时先用 EXPLAIN 或 view
确认可见现象，再用源码入口解释状态变化，最后用更小实验验证。

## 11. 观测与诊断入口

EXPLAIN 节点名映射 Exec 函数。

这个观测点只说明一个切面。要形成结论，必须把它和
owner、lifetime、reset/rescan 或 AM callback 对齐。

断点 SeqNext。

这个观测点只说明一个切面。要形成结论，必须把它和
owner、lifetime、reset/rescan 或 AM callback 对齐。

断点 IndexNext。

这个观测点只说明一个切面。要形成结论，必须把它和
owner、lifetime、reset/rescan 或 AM callback 对齐。

断点 heap_getnextslot。

这个观测点只说明一个切面。要形成结论，必须把它和
owner、lifetime、reset/rescan 或 AM callback 对齐。

建议每次诊断都写四列：可见现象、断点或视图、对应源码入口、解释现象所需状态。

## 12. 常见误区

误区：看到 `ScanState` 就以为已经理解本节。实际还要知道它的创建者、读取者、失效条件和
cleanup 边界。

误区：看到 `TableScanDesc`
就以为已经理解本节。实际还要知道它的创建者、读取者、失效条件和 cleanup 边界。

误区：看到 `IndexScanDesc`
就以为已经理解本节。实际还要知道它的创建者、读取者、失效条件和 cleanup 边界。

误区：看到 `Snapshot` 就以为已经理解本节。实际还要知道它的创建者、读取者、失效条件和
cleanup 边界。

误区：把 EXPLAIN 输出当成内部状态完整镜像。EXPLAIN 是观测结果，不会展示所有
slot、context、descriptor、callback 和 AM 私有状态。

误区：把所有内存增长都叫 leak。executor 中很多增长是正常
retention、work_mem 内算法状态或观测时机造成的。

误区：只看当前版本 helper 名字。课程结论依赖的是边界模型，而不是 helper
永远处在同一行。

## 13. 课堂实验

实验一：用 SQL 观察本节可见现象。

```text
CREATE TABLE t_scan AS SELECT g AS id, md5(g::text) AS v FROM generate_series(1,10000) g;
CREATE INDEX ON t_scan(id);
EXPLAIN (ANALYZE, VERBOSE, BUFFERS) SELECT * FROM t_scan WHERE id = 42;
```

实验二：用 gdb 或 lldb 给第 5
节主流程中的两个函数下断点，记录状态进入和离开函数时的变化。

```text
break ExecSeqScan
break ExecScanExtended
run
```

实验三：缩小数据规模，只保留能触发同一状态变化的最小 SQL。小实验更适合验证源码模型。

## 14. 源码阅读练习

下面练习按入口 -> 状态 -> hot path -> cleanup ->
观测组织。每一项都要写状态变化，不要只摘函数名。

练习 1：打开
`/home/highgo/postgres/src/include/executor/execScan.h`，找到本节主状态第一次被创建或绑定的位置。

继续在 `src/include/executor/execScan.h` 中找一个
cleanup、reset、recheck 或 instrumentation
边界，说明它如何保护本节唯一主问题。

练习 2：打开
`/home/highgo/postgres/src/backend/executor/nodeSeqscan.c`，找到本节主状态第一次被创建或绑定的位置。

继续在 `src/backend/executor/nodeSeqscan.c` 中找一个
cleanup、reset、recheck 或 instrumentation
边界，说明它如何保护本节唯一主问题。

练习 3：打开
`/home/highgo/postgres/src/backend/executor/nodeIndexscan.c`，找到本节主状态第一次被创建或绑定的位置。

继续在 `src/backend/executor/nodeIndexscan.c` 中找一个
cleanup、reset、recheck 或 instrumentation
边界，说明它如何保护本节唯一主问题。

练习 4：打开
`/home/highgo/postgres/src/include/access/tableam.h`，找到本节主状态第一次被创建或绑定的位置。

继续在 `src/include/access/tableam.h` 中找一个
cleanup、reset、recheck 或 instrumentation
边界，说明它如何保护本节唯一主问题。

练习 5：打开
`/home/highgo/postgres/src/backend/access/table/tableam.c`，找到本节主状态第一次被创建或绑定的位置。

继续在 `src/backend/access/table/tableam.c` 中找一个
cleanup、reset、recheck 或 instrumentation
边界，说明它如何保护本节唯一主问题。

练习 6：打开
`/home/highgo/postgres/src/backend/access/heap/heapam.c`，找到本节主状态第一次被创建或绑定的位置。

继续在 `src/backend/access/heap/heapam.c` 中找一个
cleanup、reset、recheck 或 instrumentation
边界，说明它如何保护本节唯一主问题。

练习 7：打开
`/home/highgo/postgres/src/backend/access/index/indexam.c`，找到本节主状态第一次被创建或绑定的位置。

继续在 `src/backend/access/index/indexam.c` 中找一个
cleanup、reset、recheck 或 instrumentation
边界，说明它如何保护本节唯一主问题。

练习 8：打开
`/home/highgo/postgres/src/include/access/amapi.h`，找到本节主状态第一次被创建或绑定的位置。

继续在 `src/include/access/amapi.h` 中找一个
cleanup、reset、recheck 或 instrumentation
边界，说明它如何保护本节唯一主问题。

### 14.1 从现象回到源码的排查剧本

压缩用户现象：围绕 `扫描节点如何把 MVCC 可见性、table AM / index
AM、qual 过滤和 slot 输出串起来，同时避免执行器直接理解 heap page
细节？`，只记录能解释本节主状态的事实。

如果记录内容不能说明状态在何处变化，它就是背景材料，不应成为本节结论。

选择断点入口：围绕 `扫描节点如何把 MVCC 可见性、table AM / index
AM、qual 过滤和 slot 输出串起来，同时避免执行器直接理解 heap page
细节？`，只记录能解释本节主状态的事实。

如果记录内容不能说明状态在何处变化，它就是背景材料，不应成为本节结论。

记录 owner：围绕 `扫描节点如何把 MVCC 可见性、table AM / index
AM、qual 过滤和 slot 输出串起来，同时避免执行器直接理解 heap page
细节？`，只记录能解释本节主状态的事实。

如果记录内容不能说明状态在何处变化，它就是背景材料，不应成为本节结论。

记录失效边界：围绕 `扫描节点如何把 MVCC 可见性、table AM / index
AM、qual 过滤和 slot 输出串起来，同时避免执行器直接理解 heap page
细节？`，只记录能解释本节主状态的事实。

如果记录内容不能说明状态在何处变化，它就是背景材料，不应成为本节结论。

记录返回语义：围绕 `扫描节点如何把 MVCC 可见性、table AM / index
AM、qual 过滤和 slot 输出串起来，同时避免执行器直接理解 heap page
细节？`，只记录能解释本节主状态的事实。

如果记录内容不能说明状态在何处变化，它就是背景材料，不应成为本节结论。

回到 EXPLAIN 或视图验证：围绕 `扫描节点如何把 MVCC 可见性、table AM /
index AM、qual 过滤和 slot 输出串起来，同时避免执行器直接理解 heap page
细节？`，只记录能解释本节主状态的事实。

如果记录内容不能说明状态在何处变化，它就是背景材料，不应成为本节结论。

### 14.2 最小源码复盘清单

复盘项 `可见现象`：把它落到本节时，必须能指出真实源码入口和状态边界。

如果 `可见现象` 不能解释第 5 节主流程中的某一步，就说明还没有找到本节核心状态。

复盘项 `入口函数`：把它落到本节时，必须能指出真实源码入口和状态边界。

如果 `入口函数` 不能解释第 5 节主流程中的某一步，就说明还没有找到本节核心状态。

复盘项 `owner`：把它落到本节时，必须能指出真实源码入口和状态边界。

如果 `owner` 不能解释第 5 节主流程中的某一步，就说明还没有找到本节核心状态。

复盘项 `写入者`：把它落到本节时，必须能指出真实源码入口和状态边界。

如果 `写入者` 不能解释第 5 节主流程中的某一步，就说明还没有找到本节核心状态。

复盘项 `读取者`：把它落到本节时，必须能指出真实源码入口和状态边界。

如果 `读取者` 不能解释第 5 节主流程中的某一步，就说明还没有找到本节核心状态。

复盘项 `失效点`：把它落到本节时，必须能指出真实源码入口和状态边界。

如果 `失效点` 不能解释第 5 节主流程中的某一步，就说明还没有找到本节核心状态。

复盘项 `cleanup`：把它落到本节时，必须能指出真实源码入口和状态边界。

如果 `cleanup` 不能解释第 5 节主流程中的某一步，就说明还没有找到本节核心状态。

复盘项 `观测入口`：把它落到本节时，必须能指出真实源码入口和状态边界。

如果 `观测入口` 不能解释第 5 节主流程中的某一步，就说明还没有找到本节核心状态。

复盘项 `反例`：把它落到本节时，必须能指出真实源码入口和状态边界。

如果 `反例` 不能解释第 5 节主流程中的某一步，就说明还没有找到本节核心状态。

复盘项 `成本传播`：把它落到本节时，必须能指出真实源码入口和状态边界。

如果 `成本传播` 不能解释第 5 节主流程中的某一步，就说明还没有找到本节核心状态。

### 14.3 最小改动审查点

审查 `新增字段` 时，先确认它是否改变 owner、生命周期、失效点或用户可见语义。

如果只改变内部组织，也要说明不会破坏 ERROR cleanup、parallel
worker、instrumentation 和已有 fast path。

审查 `新增 fast path` 时，先确认它是否改变
owner、生命周期、失效点或用户可见语义。

如果只改变内部组织，也要说明不会破坏 ERROR cleanup、parallel
worker、instrumentation 和已有 fast path。

审查 `新增内存分配` 时，先确认它是否改变 owner、生命周期、失效点或用户可见语义。

如果只改变内部组织，也要说明不会破坏 ERROR cleanup、parallel
worker、instrumentation 和已有 fast path。

审查 `新增观测指标` 时，先确认它是否改变 owner、生命周期、失效点或用户可见语义。

如果只改变内部组织，也要说明不会破坏 ERROR cleanup、parallel
worker、instrumentation 和已有 fast path。

审查 `新增 rescan 行为` 时，先确认它是否改变
owner、生命周期、失效点或用户可见语义。

如果只改变内部组织，也要说明不会破坏 ERROR cleanup、parallel
worker、instrumentation 和已有 fast path。

审查 `新增测试` 时，先确认它是否改变 owner、生命周期、失效点或用户可见语义。

如果只改变内部组织，也要说明不会破坏 ERROR cleanup、parallel
worker、instrumentation 和已有 fast path。

### 14.4 版本与口径提醒

本课基于指定 master commit 讲当前实现。

其它版本函数拆分可能不同，但边界模型应保持。

实验时一次只改变一个 GUC 或 SQL shape。

性能归因要区分 planner、executor、storage AM 和 workload。

correctness 结论必须能复现或用断点验证。

### 14.5 源码复盘场景

复盘目标：把 `扫描访问方法边界` 放回真实执行现场。主角是
ScanState、TableScanDesc、IndexScanDesc、slot，主线是
executor 协议到 table/index AM。

下面每个场景都只验证一个状态边界。不要让一个实验同时承担正确性、性能、观测和版本差异四类问题。

场景 `空结果路径`：确认没有 tuple 返回时，状态是否仍被正确 clear，调用者拿到的是空
slot、NULL 指针还是已经结束的 scan descriptor。

套回本节时，先写下
ScanState、TableScanDesc、IndexScanDesc、slot
中哪个对象被创建、读取、失效或释放，再解释它如何影响 executor 协议到
table/index AM。

如果这个场景无法连接到第 5 节的主流程，就说明它不是本节主问题的一部分，应放到后续课程或旁注中。

场景 `宽行路径`：确认 descriptor、slot deform、projection 或
memory counter 的成本是否随列数和 varlena 值放大。

套回本节时，先写下
ScanState、TableScanDesc、IndexScanDesc、slot
中哪个对象被创建、读取、失效或释放，再解释它如何影响 executor 协议到
table/index AM。

如果这个场景无法连接到第 5 节的主流程，就说明它不是本节主问题的一部分，应放到后续课程或旁注中。

场景 `重复执行路径`：确认同一 QueryDesc 或 PlanState
被多次拉取时，per-query 状态没有被提前删除。

套回本节时，先写下
ScanState、TableScanDesc、IndexScanDesc、slot
中哪个对象被创建、读取、失效或释放，再解释它如何影响 executor 协议到
table/index AM。

如果这个场景无法连接到第 5 节的主流程，就说明它不是本节主问题的一部分，应放到后续课程或旁注中。

场景 `rescan 路径`：确认参数变化、方向变化或 mark/restore
后，旧状态是复用、rewind、reset 还是重建。

套回本节时，先写下
ScanState、TableScanDesc、IndexScanDesc、slot
中哪个对象被创建、读取、失效或释放，再解释它如何影响 executor 协议到
table/index AM。

如果这个场景无法连接到第 5 节的主流程，就说明它不是本节主问题的一部分，应放到后续课程或旁注中。

场景 `ERROR 路径`：确认 longjmp 后内存由上层 context
兜底，非内存资源则有明确 cleanup 或 callback。

套回本节时，先写下
ScanState、TableScanDesc、IndexScanDesc、slot
中哪个对象被创建、读取、失效或释放，再解释它如何影响 executor 协议到
table/index AM。

如果这个场景无法连接到第 5 节的主流程，就说明它不是本节主问题的一部分，应放到后续课程或旁注中。

场景 `并行路径`：确认 backend-local 指针没有直接跨 worker
使用，需要序列化、DSM 或 worker 侧重建。

套回本节时，先写下
ScanState、TableScanDesc、IndexScanDesc、slot
中哪个对象被创建、读取、失效或释放，再解释它如何影响 executor 协议到
table/index AM。

如果这个场景无法连接到第 5 节的主流程，就说明它不是本节主问题的一部分，应放到后续课程或旁注中。

场景 `观测路径`：确认 EXPLAIN、view、日志或 profile
看到的是哪个生命周期阶段，而不是把快照当成全过程。

套回本节时，先写下
ScanState、TableScanDesc、IndexScanDesc、slot
中哪个对象被创建、读取、失效或释放，再解释它如何影响 executor 协议到
table/index AM。

如果这个场景无法连接到第 5 节的主流程，就说明它不是本节主问题的一部分，应放到后续课程或旁注中。

场景 `扩展 hook 路径`：确认扩展插桩没有改变
CurrentMemoryContext、没有漏调用 previous hook，也没有在 hot
path 做重操作。

套回本节时，先写下
ScanState、TableScanDesc、IndexScanDesc、slot
中哪个对象被创建、读取、失效或释放，再解释它如何影响 executor 协议到
table/index AM。

如果这个场景无法连接到第 5 节的主流程，就说明它不是本节主问题的一部分，应放到后续课程或旁注中。

场景 `资源释放路径`：确认 slot、descriptor、scan
desc、tuplestore、tuplesort、relation refcount 或
callback 都在正确顺序释放。

套回本节时，先写下
ScanState、TableScanDesc、IndexScanDesc、slot
中哪个对象被创建、读取、失效或释放，再解释它如何影响 executor 协议到
table/index AM。

如果这个场景无法连接到第 5 节的主流程，就说明它不是本节主问题的一部分，应放到后续课程或旁注中。

场景 `fast path
路径`：确认优化路径和通用路径返回相同语义，只是减少不必要的表达式、投影或资源操作。

套回本节时，先写下
ScanState、TableScanDesc、IndexScanDesc、slot
中哪个对象被创建、读取、失效或释放，再解释它如何影响 executor 协议到
table/index AM。

如果这个场景无法连接到第 5 节的主流程，就说明它不是本节主问题的一部分，应放到后续课程或旁注中。

场景 `诊断误判路径`：确认慢或大不是 planner 选择、workload shape、GUC
或存储访问造成，再归因到本节状态。

套回本节时，先写下
ScanState、TableScanDesc、IndexScanDesc、slot
中哪个对象被创建、读取、失效或释放，再解释它如何影响 executor 协议到
table/index AM。

如果这个场景无法连接到第 5 节的主流程，就说明它不是本节主问题的一部分，应放到后续课程或旁注中。

场景 `版本差异路径`：确认函数名变动时仍能找到相同的 owner、hot path、失效点和
cleanup 边界。

套回本节时，先写下
ScanState、TableScanDesc、IndexScanDesc、slot
中哪个对象被创建、读取、失效或释放，再解释它如何影响 executor 协议到
table/index AM。

如果这个场景无法连接到第 5 节的主流程，就说明它不是本节主问题的一部分，应放到后续课程或旁注中。

### 14.6 工程改动审查点

新增字段必须说明初始化值、正常更新点、rescan 行为、ERROR 后释放方式和可观测方式。

在 `扫描访问方法边界` 中，这条审查点必须能落到一个真实函数或结构体字段；否则它只是泛泛建议。

新增 fast path 必须证明不会跳过
descriptor、slot、parameter、visibility、reset 或
cleanup 边界。

在 `扫描访问方法边界` 中，这条审查点必须能落到一个真实函数或结构体字段；否则它只是泛泛建议。

新增内存分配必须说明 context 归属；如果使用
TopMemoryContext，必须有强理由。

在 `扫描访问方法边界` 中，这条审查点必须能落到一个真实函数或结构体字段；否则它只是泛泛建议。

新增指标必须说明采样时机，避免把 per-tuple、per-query
和节点私有状态混成一个数字。

在 `扫描访问方法边界` 中，这条审查点必须能落到一个真实函数或结构体字段；否则它只是泛泛建议。

新增测试应覆盖正常路径、空结果、rescan、ERROR 或提前结束中的至少一个边界场景。

在 `扫描访问方法边界` 中，这条审查点必须能落到一个真实函数或结构体字段；否则它只是泛泛建议。

代码 review 时要追问改动改变的是用户可见语义，还是 executor 内部状态组织。

在 `扫描访问方法边界` 中，这条审查点必须能落到一个真实函数或结构体字段；否则它只是泛泛建议。

性能 review 时要追问成本随 rows、loops、columns、work_mem
或参数变化如何放大。

在 `扫描访问方法边界` 中，这条审查点必须能落到一个真实函数或结构体字段；否则它只是泛泛建议。

cleanup review 时要追问 MemoryContext 删除前是否已经释放非内存资源。

在 `扫描访问方法边界` 中，这条审查点必须能落到一个真实函数或结构体字段；否则它只是泛泛建议。

并行 review 时要追问 worker 是否能重建同等语义，而不是复用 leader 指针。

在 `扫描访问方法边界` 中，这条审查点必须能落到一个真实函数或结构体字段；否则它只是泛泛建议。

观测 review 时要追问输出数字是否能回到一个源码 counters 或
instrumentation 字段。

在 `扫描访问方法边界` 中，这条审查点必须能落到一个真实函数或结构体字段；否则它只是泛泛建议。

做完这两组复盘后，最终答案应压缩成三句话：现象是什么，哪个状态解释它，哪个源码边界保证或破坏这个状态。

## 15. 讨论题

如果把本节主状态放错 owner，会出现什么现象？

回答时必须回到源码中的具体状态和生命周期，不要只给概念性判断。

哪个观测入口最容易误导？为什么？

回答时必须回到源码中的具体状态和生命周期，不要只给概念性判断。

如果要给本机制加 instrumentation，插桩应放在哪里？

回答时必须回到源码中的具体状态和生命周期，不要只给概念性判断。

哪类 workload 会放大本节成本？应先改 SQL、GUC、schema 还是源码？

回答时必须回到源码中的具体状态和生命周期，不要只给概念性判断。

## 16. 本节小结

本节不是 API 清单，而是一条围绕唯一主问题展开的 runtime 状态链。

可迁移规律是：运行时对象的语义来自字段、owner、生命周期、失效条件和观测边界的组合。

诊断 executor 问题时，先固定源码基线，再建立状态模型，最后用实验把模型压回运行时事实。

下一节或后续课程会继续沿用这套方法，只是把主对象替换成新的执行节点、观测指标或扩展插桩边界。
