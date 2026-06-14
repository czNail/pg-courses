# PostgreSQL vacuum cost delay / throttling

## 课程定位
前置知识：已经理解 lazy VACUUM 的 heap / index 分阶段清理、freeze / anti-wraparound、visibility map、autovacuum launcher / worker 调度，以及 buffer manager 的 shared buffer 访问模型。
本节唯一主问题：
```text
vacuum cost accounting、delay 和 worker 平滑策略如何在维护吞吐与前台查询之间折中？
```
核心矛盾：
```text
VACUUM 必须持续读取、修改和回收大量页面，
否则 dead tuple、XID age、MultiXact age 和统计信息会变成全局风险；
但 VACUUM 又和前台查询共享 buffer pool、I/O 队列、WAL 带宽和 CPU，
如果不节流，后台维护会把前台延迟推高。
```
学完后应能独立判断：
- `vacuum_cost_page_hit`、`vacuum_cost_page_miss`、`vacuum_cost_page_dirty` 分别在哪些 buffer 路径累加。
- `vacuum_cost_limit` 和 `vacuum_cost_delay` 如何从 GUC 变成一次 VACUUM 的有效运行参数。
- `vacuum_delay_point()` 为什么是“安全点睡眠”，不是 I/O 调度器。
- autovacuum worker 为什么要动态分摊 cost limit。
- parallel vacuum 为什么需要共享 cost balance，而不是每个 worker 各睡各的。
- failsafe / anti-wraparound 为什么会关闭 cost delay。
- 哪些现象能从 wait event、progress view、verbose log 直接看到，哪些只能推断。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e`。

## 1. 本节在总主线中的位置
第 28 节解释 lazy VACUUM 为什么要分成 heap scan、index vacuum、heap cleanup 和 final cleanup。
第 29 节解释 freeze 与 anti-wraparound 的语义边界。
第 39 节解释 autovacuum launcher / worker 如何选择 database 和 relation。
本节接在这些内容之后，关注另一个问题：
```text
当一个 VACUUM 已经被选中并开始执行，
它如何限制自己对 shared buffers、I/O 和前台查询的扰动？
```
这不是“VACUUM 是否正确回收”的问题。
正确性由 MVCC horizon、heap / index 协议、freeze、WAL、locks 和 page pins 保证。
cost delay 只回答运行期资源节奏：
```text
做了多少近似 I/O 工作？
是否到了需要让出一小段时间的点？
多个 worker 是否应该共享同一个吞吐预算？
遇到 wraparound 风险时是否还允许慢慢做？
```
本节的 runtime 现象锚点：
```text
把 autovacuum_vacuum_cost_delay 设为非零，
制造一张需要大量 VACUUM 的表，
观察 pg_stat_activity.wait_event = VacuumDelay、
pg_stat_progress_vacuum.delay_time、
VACUUM VERBOSE / autovacuum log 中的 delay time，
再回到 vacuum_delay_point() 和 bufmgr.c 的 cost 累加点。
```
这个现象证明：cost delay 不是事后日志，也不是 planner 估算。
它是 VACUUM 执行过程中由 buffer 访问真实累加、在主循环安全点触发的 backend-local 节流状态。

## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
buffer manager 在 VACUUM 访问页面时把 hit / miss / dirty 折算成 cost credits；
VACUUM 主循环周期性调用 vacuum_delay_point()；
当有效 cost balance 超过有效 cost limit 时，
backend 按 vacuum_cost_delay * balance / limit 睡眠，并把 balance 清零；
autovacuum worker 会按当前参与平衡的 worker 数动态缩小每个 worker 的 limit；
parallel vacuum worker 则通过 shared atomic balance 让做更多 I/O 的 worker 多睡。
```
这里有三个层次的折中。
第一层是本地节流：
```text
一次 VACUUM backend 做得越多，
它越频繁进入 VacuumDelay wait event。
```
第二层是 autovacuum worker 平滑：
```text
同一实例内同时运行多个 autovacuum worker 时，
如果它们都使用全局默认 cost limit，
总 I/O 压力会随 worker 数线性放大；
所以 PostgreSQL 把一个有效 limit 分摊给参与平衡的 worker。
```
第三层是 parallel vacuum 平滑：
```text
一张表的 parallel vacuum 里，
leader 和 parallel workers 共享 cost balance；
哪个进程实际做了更多 I/O，哪个进程更可能睡眠。
```
这三个层次都不是精确的 I/O token bucket。
它们不直接控制 kernel block layer。
它们不区分不同磁盘、不同 tablespace、不同 cgroup。
它们也不知道前台 SQL 的真实 SLA。
PostgreSQL 接受这种近似，是因为代价很低：
- 不需要在 shared buffer hot path 引入复杂调度器。
- 不需要让前台查询和 VACUUM 竞争同一个全局 semaphore。
- 不需要在每次 page access 时进入 heavyweight 协调。
- 可以通过 GUC 和 reloptions 在线调整 autovacuum 的节奏。
因此本节的核心 mental model 是：
```text
cost delay 是 VACUUM 自愿让出的时间片，
不是资源隔离边界；
它保护前台查询的方式是降低后台维护的平均消耗速率，
而不是保证任意单次前台查询不会被 I/O、WAL 或 buffer churn 影响。
```

## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/commands/vacuum.c` | `vacuum()` 初始化 cost 状态，`vacuum_delay_point()` 触发睡眠，`compute_parallel_delay()` 平滑 parallel vacuum。 |
| 2 | `src/include/commands/vacuum.h` | `vacuum_cost_delay`、`vacuum_cost_limit`、`VacuumFailsafeActive`、parallel cost 变量声明。 |
| 3 | `src/include/miscadmin.h` | `VacuumCostPageHit`、`VacuumCostPageMiss`、`VacuumCostPageDirty`、`VacuumCostLimit`、`VacuumCostDelay`、`VacuumCostBalance`、`VacuumCostActive`。 |
| 4 | `src/backend/utils/init/globals.c` | 普通 VACUUM cost 变量默认值：hit=1、miss=2、dirty=20、limit=200、delay=0。 |
| 5 | `src/backend/storage/buffer/bufmgr.c` | buffer hit、read miss、dirty page 在真实访问路径中累加 VACUUM cost。 |
| 6 | `src/backend/access/heap/vacuumlazy.c` | lazy VACUUM 主循环调用 `vacuum_delay_point(false)`；failsafe 触发后关闭 cost delay。 |
| 7 | `src/backend/commands/analyze.c` | ANALYZE 也复用 `vacuum_delay_point(true)`，progress 计入 ANALYZE delay。 |
| 8 | `src/backend/postmaster/autovacuum.c` | `VacuumUpdateCosts()`、`AutoVacuumUpdateCostLimit()`、`autovac_recalculate_workers_for_balance()`、reloptions 与 worker balance。 |
| 9 | `src/backend/commands/vacuumparallel.c` | parallel vacuum 把 cost 参数传播给 workers，并汇总 worker delay。 |
| 10 | `src/backend/utils/misc/guc_parameters.dat` | GUC 上下文、默认值、范围与 SIGHUP / USERSET 边界。 |
| 11 | `src/backend/catalog/system_views.sql` | `pg_stat_progress_vacuum.delay_time` 和 `pg_stat_progress_analyze.delay_time` 的视图来源。 |
| 12 | `src/backend/utils/activity/wait_event_names.txt` | `VACUUM_DELAY` wait event 的描述。 |
推荐阅读顺序：
```text
GUC defaults
  -> globals and effective per-backend variables
  -> bufmgr cost accounting
  -> vacuum_delay_point()
  -> autovacuum worker cost limit rebalance
  -> parallel vacuum shared balance
  -> failsafe disabling delay
  -> progress / wait event observation
```
不要从 autovacuum launcher 的 database schedule 开始。
第 39 节已经讲过“谁被调度”。
本节要跟的主线是“已经在运行的 VACUUM 如何自我节流”。

## 4. 关键数据结构与状态
### 4.1 全局 GUC 值与有效运行值
PostgreSQL 同时保留两类 cost 变量。
第一类是 GUC 绑定的全局默认值。
它们声明在 `miscadmin.h`，定义在 `globals.c`，由 GUC 系统更新。
| 变量 | 默认值 | 语义 |
| --- | --- | --- |
| `VacuumCostPageHit` | `1` | VACUUM 访问已在 buffer cache 中的页面时记多少 cost。 |
| `VacuumCostPageMiss` | `2` | VACUUM 发起读入不在 cache 中的页面时记多少 cost。 |
| `VacuumCostPageDirty` | `20` | VACUUM 第一次把页面标脏时记多少 cost。 |
| `VacuumCostLimit` | `200` | 普通 cost balance 达到多少 credits 后考虑睡眠。 |
| `VacuumCostDelay` | `0` | 普通手工 VACUUM 默认不睡眠。 |
| `VacuumCostBalance` | `0` | 当前 backend 从上次睡眠后累积的 cost。 |
| `VacuumCostActive` | `false` | 当前 backend 是否启用 cost delay。 |
第二类是一次 VACUUM 的有效值。
它们定义在 `vacuum.c`：
| 变量 | 语义 |
| --- | --- |
| `vacuum_cost_delay` | 当前 backend 本次 VACUUM / ANALYZE 使用的 delay。 |
| `vacuum_cost_limit` | 当前 backend 本次 VACUUM / ANALYZE 使用的 limit。 |
| `VacuumCostBalanceLocal` | parallel vacuum 中当前 worker 本地累计的 cost。 |
| `VacuumSharedCostBalance` | parallel vacuum leader / worker 共享的 atomic cost balance。 |
| `VacuumActiveNWorkers` | parallel vacuum 当前活跃参与者数量。 |
| `parallel_vacuum_worker_delay_ns` | parallel worker 暂存的 delay time，上报给 leader。 |
为什么要分两套？
因为不同执行来源的有效参数不同。
手工 `VACUUM` 默认使用普通 GUC：
```text
vacuum_cost_delay = VacuumCostDelay
vacuum_cost_limit = VacuumCostLimit
```
autovacuum worker 先看 relation storage parameters，再看 autovacuum GUC，最后才继承普通 GUC。
因此你不能只看 `SHOW vacuum_cost_delay` 就断定 autovacuum 是否会睡。
### 4.2 普通 VACUUM 与 autovacuum 的默认差异
当前源码默认值是：
| GUC | 默认值 | 上下文 | 说明 |
| --- | --- | --- | --- |
| `vacuum_cost_delay` | `0` | `PGC_USERSET` | 手工 VACUUM 默认不节流。 |
| `vacuum_cost_limit` | `200` | `PGC_USERSET` | 普通 cost limit。 |
| `vacuum_cost_page_hit` | `1` | `PGC_USERSET` | cache hit 的 credits。 |
| `vacuum_cost_page_miss` | `2` | `PGC_USERSET` | read miss 的 credits。 |
| `vacuum_cost_page_dirty` | `20` | `PGC_USERSET` | dirty page 的 credits。 |
| `autovacuum_vacuum_cost_delay` | `2ms` | `PGC_SIGHUP` | autovacuum 默认会节流。 |
| `autovacuum_vacuum_cost_limit` | `-1` | `PGC_SIGHUP` | `-1` 表示使用 `vacuum_cost_limit`。 |
| `track_cost_delay_timing` | `off` | `PGC_SUSET` | 是否累计 delay time 到 progress / verbose。 |
这个默认组合表达了社区取舍：
```text
手工 VACUUM 通常由管理员显式发起，默认追求尽快完成；
autovacuum 是后台长期维护，默认应该让出一部分资源。
```
但这只是默认。
生产系统里，手工维护窗口可能也会设置非零 `vacuum_cost_delay`。
高写入系统里，autovacuum 也可能需要提高 cost limit 或降低 delay，避免长期追不上 dead tuple 产生速度。
### 4.3 `WorkerInfoData.wi_dobalance`
autovacuum worker 的共享状态在 `autovacuum.c` 的 `WorkerInfoData` 中。
本节关心其中一个字段：
```text
wi_dobalance: Whether this worker should be included in balance calculations
```
源码用 `pg_atomic_flag` 保存这个布尔语义。
这里有一个容易读反的点。
`pg_atomic_unlocked_test_flag()` 的名字来自 TAS lock 语义。
它返回 true 表示 flag 当前是 0。
在 autovacuum balance 语义中：
```text
tab->at_dobalance = true
  -> pg_atomic_test_set_flag(&wi_dobalance)
  -> flag 变成 1
  -> pg_atomic_unlocked_test_flag(&wi_dobalance) 返回 false
  -> worker 被纳入 balance 计数
```
所以阅读 `AutoVacuumUpdateCostLimit()` 时要反着理解这个 helper 名字：
```text
unlocked_test_flag(...) == true
  -> 当前 worker 不参与 cost limit balance
unlocked_test_flag(...) == false
  -> 当前 worker 参与 cost limit balance
```
为什么某些 worker 不参与？
如果 relation storage parameter 明确设置了 `autovacuum_vacuum_cost_limit` 或 `autovacuum_vacuum_cost_delay`，源码把 `at_dobalance` 设为 false。
这个 relation 使用自己的节流参数，不再分享实例级 autovacuum cost budget。
### 4.4 `VacuumFailsafeActive`
`VacuumFailsafeActive` 是一次 VACUUM 的全局标志。
注释明确说：
```text
If failsafe mode has been engaged,
we will not re-enable cost-based delay for the table
until after vacuuming has completed.
```
这个变量由 table AM 侧设置。
在 heap lazy VACUUM 中，`lazy_check_wraparound_failsafe()` 触发后会：
- 设置 `VacuumFailsafeActive = true`。
- 放弃 `BufferAccessStrategy`，允许使用全部 shared buffers。
- 禁用进一步 index vacuuming。
- 禁用 index cleanup。
- 禁用 heap truncation。
- 更新 progress mode 为 `failsafe`。
- 关闭 `VacuumCostActive`。
- 清零 `VacuumCostBalance`。
这是本节最重要的异常路径。
当 XID / MultiXact wraparound 风险足够高时，系统不再把“温和维护”放在第一位。
它优先让 VACUUM 快速推进 freeze 边界。

## 5. 主流程源码 walkthrough
本节主流程从一次普通 VACUUM 开始。
### 5.1 初始化 cost 状态
`vacuum()` 进入执行前会重置运行态。
关键路径：
```text
ExecVacuum()
  -> vacuum()
     -> VacuumUpdateCosts()
     -> VacuumCostBalance = 0
     -> VacuumCostBalanceLocal = 0
     -> VacuumSharedCostBalance = NULL
     -> VacuumActiveNWorkers = NULL
     -> foreach relation
        -> vacuum_rel()
           -> heap_vacuum_rel()
```
`PG_FINALLY` 中会清理：
```text
in_vacuum = false
VacuumCostActive = false
VacuumFailsafeActive = false
VacuumCostBalance = 0
```
这说明 cost balance 是一次执行内的 backend-local 状态。
它不是 durable stats。
它也不是 autovacuum launcher 用来调度下一张表的长期积分。
### 5.2 buffer manager 累加 cost
VACUUM cost accounting 不在 `vacuumlazy.c` 里手写“我读了一页”。
它挂在 buffer manager 的真实访问路径上。
buffer hit 路径：
```c
if (VacuumCostActive)
    VacuumCostBalance += VacuumCostPageHit;
```
读 miss 路径：
```c
if (VacuumCostActive)
    VacuumCostBalance += VacuumCostPageMiss * io_buffers_len;
```
脏页路径：
```c
if (VacuumCostActive)
    VacuumCostBalance += VacuumCostPageDirty;
```
读 miss 的注释很关键：
```text
Track vacuum cost when issuing IO, not after waiting for it.
```
原因是如果等 I/O 完成后才记 cost，VACUUM 可能在短时间内发出大量 I/O，然后才发现自己应该睡。
当前实现选择在发起 I/O 时记账。
这是一个节流点前移的设计。
### 5.3 lazy VACUUM 主循环调用 delay point
heap lazy scan 的主循环中，每处理页面附近会调用：
```c
vacuum_delay_point(false);
```
`false` 表示这次是 VACUUM，不是 ANALYZE。
ANALYZE 采样路径也会调用：
```c
vacuum_delay_point(true);
```
因此 `vacuum_delay_point()` 是维护类操作的共用节流入口。
但本节重点仍是 VACUUM。
它的基本结构是：
```text
vacuum_delay_point(is_analyze)
  -> CHECK_FOR_INTERRUPTS()
  -> parallel worker refresh shared delay params
  -> if !VacuumCostActive and !ConfigReloadPending return
  -> autovacuum worker process SIGHUP if needed
  -> if !VacuumCostActive return
  -> ordinary VACUUM: msec = delay * balance / limit
  -> parallel VACUUM: msec = compute_parallel_delay()
  -> cap msec to delay * 4
  -> report WAIT_EVENT_VACUUM_DELAY
  -> pg_usleep(msec * 1000)
  -> optionally report progress delay_time
  -> check postmaster death
  -> VacuumCostBalance = 0
  -> AutoVacuumUpdateCostLimit()
  -> CHECK_FOR_INTERRUPTS()
```
这里有几个边界要记住。
第一，函数总是先检查 interrupts。
cost delay 不能让取消请求、shutdown 请求或错误处理长时间失效。
第二，sleep 只在 `VacuumCostActive` 为 true 时发生。
普通手工 VACUUM 默认 delay 为 0，所以不会进入节流。
第三，sleep 时间不是固定的 `vacuum_cost_delay`。
普通路径公式是：
```text
msec = vacuum_cost_delay * VacuumCostBalance / vacuum_cost_limit
```
如果一次循环积累了 2 倍 limit，理论上睡 2 倍 delay。
源码再把单次睡眠上限压到 `vacuum_cost_delay * 4`。
第四，睡眠后清零 `VacuumCostBalance`。
这表示 balance 是“自上次让出时间片后”的局部债务。
第五，autovacuum worker 在睡眠后会调用 `AutoVacuumUpdateCostLimit()`。
这是 worker 平滑策略能够动态响应 worker 数变化的关键。
### 5.4 autovacuum worker 进入一张表
第 39 节已经讲过 worker 如何被 launcher 启动。
本节从 worker 已经拿到候选 relation 开始。
`do_autovacuum()` 在处理每张表前会：
```text
table_recheck_autovac()
  -> 得到 relation reloptions 和是否仍需 vacuum/analyze
  -> 保存 at_storage_param_vac_cost_delay
  -> 保存 at_storage_param_vac_cost_limit
  -> 设置 at_dobalance
  -> 更新 MyWorkerInfo->wi_dobalance
  -> autovac_recalculate_workers_for_balance()
  -> VacuumUpdateCosts()
  -> autovacuum_do_vac_analyze()
```
为什么 `VacuumUpdateCosts()` 要等到这里？
因为 cost 参数可能来自 relation storage parameters。
在真正 recheck 某张表之前，worker 只知道自己属于哪个 database，不知道当前 relation 是否覆盖了 autovacuum cost 配置。
### 5.5 `VacuumUpdateCosts()` 的选择顺序
autovacuum worker 有自己的有效参数选择顺序。
delay：
```text
if relation storage parameter vacuum_cost_delay >= 0:
    vacuum_cost_delay = relation value
else if autovacuum_vacuum_cost_delay >= 0:
    vacuum_cost_delay = autovacuum value
else:
    vacuum_cost_delay = VacuumCostDelay
```
limit：
```text
if relation storage parameter vacuum_cost_limit > 0:
    vacuum_cost_limit = relation value
else:
    if autovacuum_vacuum_cost_limit > 0:
        vacuum_cost_limit = autovacuum value
    else:
        vacuum_cost_limit = VacuumCostLimit
    if this worker participates in balance:
        vacuum_cost_limit = max(vacuum_cost_limit / nworkers_for_balance, 1)
```
最后，`VacuumUpdateCosts()` 根据有效 delay 设置 `VacuumCostActive`。
```text
if VacuumFailsafeActive:
    assert !VacuumCostActive
else if vacuum_cost_delay > 0:
    VacuumCostActive = true
else:
    VacuumCostActive = false
    VacuumCostBalance = 0
```
这说明 `VacuumCostActive` 不是单纯由 GUC 原始值决定。
它是当前 relation、当前 worker、failsafe 状态和 GUC reload 共同决定的运行态。
### 5.6 autovacuum worker cost limit 分摊
核心函数是：
```text
AutoVacuumUpdateCostLimit()
```
它读 `AutoVacuumShmem->av_nworkersForBalance`。
这个值由：
```text
autovac_recalculate_workers_for_balance()
```
重新计算。
计算时会遍历 `av_runningWorkers`，只统计：
- `worker->wi_proc != NULL`。
- worker 的 `wi_dobalance` flag 表示参与平衡。
如果有 4 个 worker 参与平衡，默认 autovacuum 继承的 limit 是 200。
那么每个 worker 的有效 limit 大约是：
```text
max(200 / 4, 1) = 50
```
这样单个 worker 更快达到睡眠点。
整体上，多个 worker 的总吞吐预算接近一个 autovacuum cost budget，而不是 4 倍。
这就是 worker 平滑策略的核心。
它不是把 worker 串行化。
它允许并发维护多张表。
但通过降低每个 worker 的 effective limit，把总体后台冲击压平。
### 5.7 parallel vacuum 的共享 cost balance
parallel vacuum 是另一种并发。
它不是多个 autovacuum worker 分别 vacuum 多张表。
它是一张表的 index cleanup 等阶段让 parallel worker 参与。
如果每个 parallel worker 都独立使用完整 `vacuum_cost_limit`，单张表的一次 parallel vacuum 会把节流预算放大。
所以 `vacuum_delay_point()` 在发现 `VacuumSharedCostBalance != NULL` 时，调用：
```text
compute_parallel_delay()
```
核心逻辑：
```text
shared_balance = atomic_add_fetch(shared, VacuumCostBalance)
VacuumCostBalanceLocal += VacuumCostBalance
if shared_balance >= vacuum_cost_limit
   and VacuumCostBalanceLocal > 0.5 * (vacuum_cost_limit / nworkers):
    msec = vacuum_cost_delay * VacuumCostBalanceLocal / vacuum_cost_limit
    atomic_sub_fetch(shared, VacuumCostBalanceLocal)
    VacuumCostBalanceLocal = 0
VacuumCostBalance = 0
```
这个算法表达了两个原则。
第一，所有参与者共享总 cost balance。
第二，只有本地实际做了足够多 I/O 的 worker 才睡。
如果某个 worker 做得少，它不应该因为别的 worker 的 I/O 被无谓惩罚。
如果某个 worker 做得多，它会把本地 debt 转成 sleep。
这就是 parallel vacuum 内部的平滑。

## 6. 生命周期 / ownership / cleanup
### 6.1 谁创建 cost 状态
GUC 默认值由 postmaster / backend 初始化时建立。
每个 backend 都有自己的全局变量副本。
`VacuumCostBalance`、`VacuumCostActive`、`vacuum_cost_delay`、`vacuum_cost_limit` 是进程内状态。
它们不是 shared memory。
autovacuum worker 的 `WorkerInfoData` 是 shared memory 中的 worker slot。
`av_nworkersForBalance` 是 autovacuum shared state。
parallel vacuum 的 `VacuumSharedCostBalance` 和 `VacuumActiveNWorkers` 来自 parallel vacuum 的共享状态。
它们只在 parallel vacuum 生命周期内有效。
### 6.2 谁持有 cost 状态
普通手工 VACUUM：
```text
当前 backend 持有所有 cost state。
```
autovacuum worker：
```text
当前 worker 持有 backend-local cost balance；
AutoVacuumShmem 持有 worker 列表和参与 balance 的 worker 数；
WorkerInfoData 记录当前 worker 是否参与 balance。
```
parallel vacuum：
```text
leader 和 workers 各自持有本地 cost balance；
共享 atomic balance 描述这一组参与者的总 debt；
active worker count 描述当前分摊阈值。
```
### 6.3 谁释放或重置
普通 VACUUM 的清理在 `vacuum()` 的 `PG_FINALLY` 中完成。
即使中途 ERROR，也会关闭 `VacuumCostActive` 并清零 balance。
autovacuum worker 退出或释放 worker slot 时，会把自己的 slot 放回 free list，并设置 rebalance signal。
这会促使剩余 worker 重新计算参与分摊的人数。
每张表开始前，worker 会重新保存 relation storage parameters。
每张表结束后，下一张表会再次经过 recheck 和 `VacuumUpdateCosts()`。
failsafe 状态在 relation 间会被重置。
源码在处理 relation 循环后明确保证：
```text
VacuumFailsafeActive = false
```
### 6.4 ERROR / abort 时谁兜底
普通 VACUUM：
- `PG_FINALLY` 清掉 cost 状态。
- 事务 abort 路径释放锁、buffer pins、resource owner 资源。
- cost delay 自身不持有外部资源。
autovacuum worker：
- 每张表的执行包在 `PG_TRY / PG_CATCH` 中。
- ERROR 后 `AbortOutOfAnyTransaction()`。
- `PortalContext` reset 清理 per-table allocations。
- worker 继续处理 schedule 中下一张表。
- 当前 table claim 会在错误清理路径中释放或在 worker 退出时被清掉。
parallel vacuum：
- shared cost state 属于 parallel vacuum state。
- leader 和 worker 退出时 parallel context 负责 worker cleanup。
- worker delay time 会周期性或退出前汇总到 progress。

## 7. 正确性机制层次
cost delay 本身不保证 MVCC 正确性。
它只调节执行速度。
因此本节要把“正确性边界”和“节流边界”分开。
| 层次 | 机制 | 保证 | 不要误解为 |
| --- | --- | --- | --- |
| MVCC horizon | snapshot xmin、OldestXmin、cutoffs | dead tuple 何时可回收 | 限制 VACUUM I/O 速率 |
| heap / index 协议 | `LP_DEAD`、index bulk delete、heap cleanup | 不留下危险 TID | 决定什么时候睡眠 |
| buffer pin / lock | page 正在被安全访问 | 内存安全与页面修改互斥 | 前台查询一定低延迟 |
| WAL / LSN | crash safety | 恢复顺序正确 | 后台写入不会抢 WAL 带宽 |
| cost accounting | hit / miss / dirty credits | 近似衡量 VACUUM buffer 工作量 | 精确 I/O 字节计量 |
| cost delay | `pg_usleep()` | 自愿让出执行时间 | OS 级 I/O 隔离 |
| autovacuum balance | `av_nworkersForBalance` | 多 worker 分摊 limit | 完整全局公平调度 |
| failsafe | `VacuumFailsafeActive` | wraparound 风险下加速必要工作 | 继续温和节流 |
`vacuum_delay_point()` 必须由调用者放在安全点。
这就是为什么源码注释说它应该在 VACUUM processing 的 major loop 中调用，通常每页一次。
它不是可以随意插入到任意 critical section 的函数。
如果在持有关键锁、pin 或 critical section 时睡眠，会放大并发等待。
当前 lazy VACUUM 的调用点是在页面处理主循环附近，目的是在进入下一轮工作前给系统让出时间。
这个边界比“睡眠函数在哪里”更重要。

## 8. 错误路径 / 异常路径 / fallback
### 8.1 配置 reload
autovacuum worker 支持运行中 reload cost 参数。
`vacuum_delay_point()` 检查：
```text
ConfigReloadPending && AmAutoVacuumWorkerProcess()
```
如果为真，它会：
- 清掉 `ConfigReloadPending`。
- 调用 `ProcessConfigFile(PGC_SIGHUP)`。
- 调用 `VacuumUpdateCosts()`。
- 对 parallel vacuum 传播 shared delay params。
这意味着 SIGHUP 后，正在 vacuum 一张表的 autovacuum worker 也可能改变节流节奏。
但 relation storage parameter 仍然保存在当前 worker 的全局变量中。
源码这样做是为了避免 reload 把当前表自己的 reloptions 覆盖掉。
### 8.2 delay 被关闭
如果 reload 后有效 delay 变成 0，`VacuumUpdateCosts()` 会：
```text
VacuumCostActive = false
VacuumCostBalance = 0
```
随后 `vacuum_delay_point()` 直接返回。
因此 delay 的启停不是只能在 relation 开始时发生。
autovacuum 的 SIGHUP 路径能影响正在运行的维护。
### 8.3 postmaster 死亡
`vacuum_delay_point()` 用 `pg_usleep()` 睡眠。
它没有用普通 `WaitLatch()` 模式。
源码注释说原因是这里需要微秒级睡眠时长。
为了避免 postmaster 已死时长时间无感，sleep 后会检查：
```text
IsUnderPostmaster && !PostmasterIsAlive()
```
如果 postmaster 不在，backend 直接 `exit(1)`。
这是一个进程生命周期兜底，不是 VACUUM 语义兜底。
### 8.4 failsafe 触发
heap lazy VACUUM 每扫描约 4GB 页面会检查 failsafe。
当 `vacuum_xid_failsafe_check()` 返回 true：
```text
VacuumFailsafeActive = true
do_index_vacuuming = false
do_index_cleanup = false
do_rel_truncate = false
VacuumCostActive = false
VacuumCostBalance = 0
```
progress mode 会变成 `failsafe`。
日志会出现 bypassing nonessential maintenance 的 WARNING。
这条路径的语义是：
```text
当 wraparound 风险压过普通维护成本控制时，
系统宁愿牺牲索引清理完整性、heap truncate 和温和节流，
也要尽快推进 freeze。
```
注意，failsafe 不是让 VACUUM 变得“不正确”。
它绕过的是 nonessential maintenance。
必要的 freeze / heap scan 仍要继续。
### 8.5 relation storage parameter 关闭 balance
如果表级 reloptions 显式配置了 cost delay 或 cost limit，`table_recheck_autovac()` 会让 `at_dobalance = false`。
这个 worker 不再参与实例级 autovacuum cost limit 分摊。
这样做避免了两个控制面互相覆盖：
```text
表级配置已经声明这张表要用自己的维护节奏；
全局 worker balance 不再把它强行压回平均值。
```
副作用是：大量表都配置自己的 cost limit 时，全局平滑能力会下降。
这是 DBA 操作层面的风险，不是源码 bug。
### 8.6 table 被删除或已被处理
autovacuum worker 在执行前会 recheck。
如果 relation 被删除，或统计信息已经不再需要 vacuum，worker 会跳过。
这不会产生 cost delay 的异常。
但它解释了为什么 cost 参数不能只在 candidate collection 时确定。
真正进入当前表前，worker 必须重新确认表仍然存在、仍然需要维护、reloptions 仍然有效。

## 9. 成本、资源与跨模块传播
### 9.1 cost credits 的近似性
默认 credits 是：
```text
page hit   = 1
page miss  = 2
page dirty = 20
limit      = 200
delay      = 0 for manual VACUUM
delay      = 2ms for autovacuum
```
这不是物理成本。
一个 page miss 只比 hit 多 1 credit，并不意味着真实磁盘读只比 cache hit 贵 2 倍。
dirty page 是 20 credits，也不是说它一定造成 20 倍延迟。
这些数值是节流启发式。
它把 VACUUM 对 shared buffers 和 I/O 的影响压缩成一个低成本计数器。
### 9.2 成本随规模如何扩张
VACUUM cost 主要随这些变量增长：
| 变量 | 扩张方式 |
| --- | --- |
| heap pages | lazy scan 每页进入 buffer 访问路径并调用 delay point。 |
| dead tuple 密度 | 更可能 dirtify page、触发 index cleanup、增加 WAL 和 CPU。 |
| index 数量 | index vacuum / cleanup 放大 I/O 和 CPU，但 page cost 仍主要来自 buffer 访问。 |
| visibility map 命中 | all-visible / all-frozen page 可被跳过，减少 buffer 访问和 delay point 触发。 |
| shared buffer 命中率 | hit 多则 credits 增长慢，miss / dirty 多则更快睡眠。 |
| autovacuum worker 数 | 如果参与 balance，每个 worker limit 变小；如果不参与，总体冲击可能放大。 |
| parallel vacuum worker 数 | shared balance 让做更多 I/O 的 worker 多睡，避免 per-worker limit 线性放大。 |
| checkpoint / bgwriter 状态 | dirty page 最终写回可能传播到 checkpoint I/O 压力。 |
| WAL 量 | VACUUM page changes 产生 WAL，cost delay 不直接按 WAL bytes 记账。 |
### 9.3 对前台查询的影响路径
前台查询受到的影响不是“VACUUM 在睡就安全，不睡就危险”这么简单。
常见传播路径：
```text
VACUUM heap/index scan
  -> shared buffer reads and evictions
  -> foreground cache hit rate changes
```
```text
VACUUM page cleanup / freeze
  -> dirty buffers
  -> checkpoint / bgwriter / backend write pressure
```
```text
VACUUM index cleanup
  -> index pages read and dirtied
  -> foreground index scan locality changes
```
```text
VACUUM WAL records
  -> WAL insert / flush / archive / replication pressure
```
```text
VACUUM relation locks
  -> DDL or conflicting maintenance waits
```
cost delay 只直接调节第一类和部分第二类路径的平均速率。
它不会直接限制 WAL bandwidth。
它不会阻止 checkpoint 在后面集中写脏页。
它不会让锁冲突消失。
它也不会解决 long transaction 导致的 cleanup horizon 停滞。
### 9.4 buffer access strategy 的边界
VACUUM 通常会使用 `BAS_VACUUM` buffer access strategy。
autovacuum worker 为所有表创建一个 strategy：
```text
GetAccessStrategyWithSize(BAS_VACUUM, VacuumBufferUsageLimit)
```
这和 cost delay 是两个不同层次。
buffer access strategy 控制 VACUUM 使用 shared buffers 的 ring 行为，降低对整个 buffer pool 的污染。
cost delay 控制 VACUUM 做一段工作后是否睡眠。
它们共同保护前台 workload，但边界不同：
```text
buffer strategy:
  控制 cache footprint
cost delay:
  控制平均执行速率
```
failsafe 触发后，heap lazy VACUUM 会把 `vacrel->bstrategy = NULL`。
这意味着它不再限制自己只用 VACUUM ring。
原因仍然是 wraparound 风险优先。
### 9.5 autovacuum worker balance 的成本模型
假设：
```text
autovacuum_vacuum_cost_limit = -1
vacuum_cost_limit = 200
autovacuum_vacuum_cost_delay = 2ms
3 个 autovacuum worker 都参与 balance
```
每个 worker 的有效 limit 是：
```text
max(200 / 3, 1) = 66
```
如果某个 worker 快速 dirty 页面，20 credits 一次，几次页面修改后就会触发 sleep。
如果一个 worker 的表级 reloptions 设置了自己的 cost limit，它不参与 balance。
剩余参与者的 `av_nworkersForBalance` 会重新计算。
这就是“平滑”的来源：
- worker 加入时，参与者数量增加，每个 worker limit 下降。
- worker 退出时，参与者数量减少，剩余 worker limit 上升。
- relation 级配置可以退出共享平滑。
- rebalancing 在表开始、worker 退出、sleep 后等点被重新检查。
### 9.6 parallel vacuum 与 autovacuum balance 的区别
不要把两者混成一个机制。
autovacuum balance：
```text
多个 autovacuum workers
多个 relation
AutoVacuumShmem->av_nworkersForBalance
每个 worker 有自己的 VacuumCostBalance
通过缩小 effective limit 平滑实例级后台维护压力
```
parallel vacuum balance：
```text
一张 relation 的一次 VACUUM
leader + parallel workers
VacuumSharedCostBalance
VacuumCostBalanceLocal
根据谁实际做了更多 I/O 决定谁睡
```
一个 autovacuum worker 也可能启动 parallel vacuum。
这时两层平滑会叠加：
- autovacuum 层先决定这个 worker 的 effective limit。
- parallel vacuum 层在这次 relation 内共享这个 limit。

## 10. 观测与诊断入口
### 10.1 能直接看到的状态
当前会话 GUC：
```sql
SHOW vacuum_cost_delay;
SHOW vacuum_cost_limit;
SHOW vacuum_cost_page_hit;
SHOW vacuum_cost_page_miss;
SHOW vacuum_cost_page_dirty;
SHOW autovacuum_vacuum_cost_delay;
SHOW autovacuum_vacuum_cost_limit;
SHOW track_cost_delay_timing;
```
运行中 wait event：
```sql
SELECT pid, backend_type, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event = 'VacuumDelay';
```
VACUUM progress：
```sql
SELECT pid, relid::regclass, phase, mode, started_by,
       heap_blks_scanned, heap_blks_vacuumed,
       index_vacuum_count, delay_time
FROM pg_stat_progress_vacuum;
```
ANALYZE progress：
```sql
SELECT pid, relid::regclass, phase, started_by, delay_time
FROM pg_stat_progress_analyze;
```
注意：`delay_time` 只有在 `track_cost_delay_timing=on` 时才会累加有意义的值。
视图里它来自 progress 参数中的纳秒累计，再除以 `1000000` 显示为毫秒。
### 10.2 日志入口
手工 `VACUUM (VERBOSE)` 或 `log_autovacuum_min_duration` 日志中，在 `track_cost_delay_timing=on` 时会看到：
```text
delay time: ... ms
```
autovacuum worker 在 DEBUG2 下还可能输出：
```text
Autovacuum VacuumUpdateCosts(db=..., rel=..., dobalance=..., cost_limit=..., cost_delay=..., active=..., failsafe=...)
```
这条 DEBUG 日志很适合源码实验。
它能验证：
- 当前 relation 是否参与 balance。
- 当前 effective cost_limit 是多少。
- 当前 effective cost_delay 是多少。
- failsafe 是否已经关闭 cost delay。
### 10.3 能推断但不能直接看到的状态
`VacuumCostBalance` 不能直接从 SQL 读取。
`VacuumCostActive` 也不能直接从 SQL 读取。
`AutoVacuumShmem->av_nworkersForBalance` 没有普通 SQL 视图。
`VacuumSharedCostBalance` 也没有 SQL 视图。
你只能通过这些现象推断：
- wait event 是否频繁出现 `VacuumDelay`。
- progress `delay_time` 是否增长。
- DEBUG2 `VacuumUpdateCosts` 中的 effective limit。
- `pg_stat_activity` 中 autovacuum worker 数量。
- `pg_stat_progress_vacuum.started_by` 是否为 `autovacuum`。
- `pg_stat_progress_vacuum.mode` 是否变成 `failsafe`。
- `VACUUM VERBOSE` 里的 delay time 和 buffer / WAL 统计。
### 10.4 不能从 cost delay 直接归因的现象
如果前台查询变慢，看到 autovacuum 不一定说明 cost delay 配置错误。
还需要区分：
- 前台查询是否在等 relation lock。
- shared buffers 是否被 VACUUM scan 改变命中率。
- checkpoint 是否正在集中写脏页。
- WAL flush 是否成为瓶颈。
- replication slot 是否阻止 cleanup。
- long transaction 是否导致 VACUUM 扫了很多却回收不了。
- autovacuum 是否处于 failsafe。
- 表级 reloptions 是否让 worker 退出 balance。
- 是否有手工 VACUUM 默认不节流。
cost delay 是诊断路径的一项，不是完整根因。
### 10.5 gdb / perf 入口
源码跟读时可以断在：
```text
vacuum_delay_point
VacuumUpdateCosts
AutoVacuumUpdateCostLimit
autovac_recalculate_workers_for_balance
compute_parallel_delay
lazy_check_wraparound_failsafe
TrackBufferHit
WaitReadBuffers
MarkBufferDirty
```
建议观察：
- `VacuumCostActive`
- `VacuumCostBalance`
- `vacuum_cost_delay`
- `vacuum_cost_limit`
- `MyWorkerInfo->wi_dobalance`
- `AutoVacuumShmem->av_nworkersForBalance`
- `VacuumSharedCostBalance`
- `VacuumCostBalanceLocal`
- `VacuumFailsafeActive`
perf / flamegraph 适合回答另一个问题：
```text
VACUUM 的 CPU 时间花在哪里？
```
它不适合直接观察 cost balance。
wait event 和 progress 更适合观察 delay 是否真的发生。

## 11. 常见误区
### 误区一：`vacuum_cost_delay=0` 表示 autovacuum 不节流
不一定。
autovacuum 默认使用 `autovacuum_vacuum_cost_delay=2ms`。
只有当 autovacuum GUC 为 `-1` 时，才回退到 `vacuum_cost_delay`。
手工 VACUUM 默认不节流，autovacuum 默认节流。
这两个默认行为不同。
### 误区二：`vacuum_cost_limit` 是每个 autovacuum worker 的固定 limit
不一定。
如果 worker 参与 balance，有效 limit 会除以 `av_nworkersForBalance`。
如果 relation 设置了自己的 cost reloptions，它可能退出 balance。
因此同一时刻不同 autovacuum worker 的 effective limit 可能不同。
### 误区三：cost credits 等于真实 I/O bytes
不是。
credits 是近似工作量。
它不等于读写字节数。
它不区分不同 storage device。
它也不直接反映 WAL bytes。
### 误区四：看到 `VacuumDelay` 就说明 VACUUM 是性能问题根因
不一定。
`VacuumDelay` 只说明 backend 正在按 cost delay 睡眠。
这反而可能说明节流正在生效。
前台慢可能来自锁、checkpoint、WAL、replication、long transaction、CPU、I/O 饱和或 schema 形态。
### 误区五：把 failsafe 当成普通加速选项
failsafe 是 anti-wraparound 兜底。
它会绕过 nonessential maintenance 并关闭 cost delay。
如果频繁进入 failsafe，问题通常不是“failsafe 太激进”，而是 autovacuum 长期追不上 XID / MultiXact 消耗。
需要检查 freeze age、worker 数、maintenance memory、long transactions、replication slots 和表级负载。
### 误区六：认为 cost delay 能保护 buffer pool
cost delay 只降低平均速率。
buffer pool footprint 主要由 buffer access strategy 和实际访问模式影响。
`vacuum_buffer_usage_limit` / `BAS_VACUUM` 与 cost delay 是互补机制。
### 误区七：认为手工 `VACUUM` 和 autovacuum 的参数完全一致
手工 VACUUM 使用普通 GUC。
autovacuum 还会看 autovacuum GUC、relation storage parameters、worker balance 和 SIGHUP reload。
同一个表被手工 VACUUM 与被 autovacuum 维护，节奏可能完全不同。

## 12. 课堂实验
### 实验 1：观察普通 cost delay
准备数据：
```sql
DROP TABLE IF EXISTS vacuum_delay_demo;
CREATE TABLE vacuum_delay_demo(id bigint, payload text);
INSERT INTO vacuum_delay_demo
SELECT g, repeat('x', 200)
FROM generate_series(1, 1000000) AS g;
DELETE FROM vacuum_delay_demo WHERE id % 2 = 0;
```
会话 A：
```sql
SET vacuum_cost_delay = '5ms';
SET vacuum_cost_limit = 50;
SET track_cost_delay_timing = on;
VACUUM (VERBOSE) vacuum_delay_demo;
```
会话 B：
```sql
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event = 'VacuumDelay';
```
另一个观察：
```sql
SELECT pid, relid::regclass, phase, delay_time
FROM pg_stat_progress_vacuum;
```
预期现象：
- `VacuumDelay` 间歇出现。
- `delay_time` 增长。
- `VACUUM VERBOSE` 输出中出现 delay time。
回到源码解释：
```text
bufmgr.c 累加 cost
  -> vacuumlazy.c 每页附近调用 vacuum_delay_point(false)
  -> vacuum.c 计算 msec
  -> WAIT_EVENT_VACUUM_DELAY
```
### 实验 2：比较手工 VACUUM 与 autovacuum 默认差异
查看默认：
```sql
SHOW vacuum_cost_delay;
SHOW autovacuum_vacuum_cost_delay;
SHOW vacuum_cost_limit;
SHOW autovacuum_vacuum_cost_limit;
```
预期：
```text
vacuum_cost_delay = 0
autovacuum_vacuum_cost_delay = 2ms
autovacuum_vacuum_cost_limit = -1
```
解释：
```text
手工 VACUUM 默认不睡。
autovacuum 默认睡。
autovacuum limit 默认继承普通 vacuum_cost_limit，
但 worker balance 可能进一步缩小 effective limit。
```
### 实验 3：观察 autovacuum worker balance
配置测试环境：
```sql
ALTER SYSTEM SET autovacuum_vacuum_cost_delay = '10ms';
ALTER SYSTEM SET autovacuum_vacuum_cost_limit = 300;
ALTER SYSTEM SET autovacuum_max_workers = 3;
ALTER SYSTEM SET log_min_messages = debug2;
SELECT pg_reload_conf();
```
制造多个表的 dead tuples，让多个 autovacuum worker 同时运行。
观察日志中的 DEBUG2：
```text
Autovacuum VacuumUpdateCosts(... dobalance=yes, cost_limit=..., cost_delay=...)
```
预期：
- 如果 3 个 worker 都参与 balance，effective `cost_limit` 接近 `300 / 3`。
- 如果 worker 数变化，后续 sleep / table boundary 后 limit 会改变。
源码断点：
```text
autovac_recalculate_workers_for_balance
AutoVacuumUpdateCostLimit
VacuumUpdateCosts
```
### 实验 4：验证表级 reloptions 退出 balance
为一张表设置 reloptions：
```sql
ALTER TABLE vacuum_delay_demo
SET (
  autovacuum_vacuum_cost_delay = 1,
  autovacuum_vacuum_cost_limit = 1000
);
```
再触发 autovacuum。
观察 DEBUG2：
```text
dobalance=no
cost_limit=1000
cost_delay=1
```
解释：
```text
table_recheck_autovac()
  -> at_storage_param_vac_cost_delay / limit
  -> at_dobalance = false
  -> worker 不参与 av_nworkersForBalance 分摊
```
### 实验 5：观察 failsafe 关闭 cost delay
这个实验不建议在生产库做。
在测试实例中构造接近 wraparound 的场景成本较高。
更适合作为源码实验：
```text
断点 lazy_check_wraparound_failsafe
临时调低 vacuum_failsafe_age
运行需要 aggressive vacuum 的表
观察 VacuumFailsafeActive、VacuumCostActive、progress mode
```
预期源码状态：
```text
VacuumFailsafeActive = true
VacuumCostActive = false
VacuumCostBalance = 0
pg_stat_progress_vacuum.mode = failsafe
```
讨论：
```text
为什么这个时候不能继续为了前台查询而慢慢睡？
```
### 实验 6：parallel vacuum delay
准备多索引大表：
```sql
DROP TABLE IF EXISTS parallel_vacuum_delay_demo;
CREATE TABLE parallel_vacuum_delay_demo(id bigint, k1 int, k2 int, payload text);
CREATE INDEX ON parallel_vacuum_delay_demo(k1);
CREATE INDEX ON parallel_vacuum_delay_demo(k2);
CREATE INDEX ON parallel_vacuum_delay_demo(payload text_pattern_ops);
INSERT INTO parallel_vacuum_delay_demo
SELECT g, g % 1000, g % 100, repeat('x', 100)
FROM generate_series(1, 1000000) AS g;
DELETE FROM parallel_vacuum_delay_demo WHERE id % 3 = 0;
```
运行：
```sql
SET max_parallel_maintenance_workers = 2;
SET vacuum_cost_delay = '5ms';
SET vacuum_cost_limit = 50;
SET track_cost_delay_timing = on;
VACUUM (PARALLEL 2, VERBOSE) parallel_vacuum_delay_demo;
```
源码观察：
```text
VacuumSharedCostBalance != NULL
compute_parallel_delay()
VacuumCostBalanceLocal
VacuumActiveNWorkers
```
预期：
- leader / worker 共享 cost debt。
- 做更多 I/O 的参与者更可能睡。
- progress delay time 需要 parallel worker 周期性上报。

## 13. 讨论题
1. 为什么 PostgreSQL 不在每次 buffer access 后立即睡眠，而是在 `vacuum_delay_point()` 的 major loop 安全点睡眠？
2. `vacuum_cost_page_dirty` 默认远高于 `vacuum_cost_page_hit`，它试图近似哪类资源压力？
3. 如果 `autovacuum_max_workers` 从 3 提高到 10，但所有 worker 都参与 balance，实例级维护吞吐一定提高 3 倍以上吗？为什么？
4. 表级设置 `autovacuum_vacuum_cost_limit` 为什么会让该 worker 退出全局 balance？这样做有什么风险？
5. `pg_stat_progress_vacuum.delay_time` 为 0，是否说明 VACUUM 没有节流？还需要检查哪个 GUC？
6. failsafe 触发后关闭 cost delay，这会如何影响前台查询？为什么 PostgreSQL 仍然这样做？
7. parallel vacuum 为什么不能让每个 worker 独立使用完整 `vacuum_cost_limit`？
8. cost delay 不能直接限制 WAL bandwidth。遇到 VACUUM 导致 WAL / replication 压力时，还应该看哪些指标？

## 14. 本节小结
本节主链路是：
```text
GUC / reloptions 决定有效 cost 参数
  -> buffer manager 在 hit / miss / dirty 路径累加 credits
  -> lazy VACUUM / ANALYZE 在主循环调用 vacuum_delay_point()
  -> 普通路径按 balance / limit 计算 sleep
  -> autovacuum worker 按参与者数量分摊 effective limit
  -> parallel vacuum 用 shared atomic balance 平滑一张表内的多个 worker
  -> failsafe 触发时关闭 cost delay 并绕过 nonessential maintenance
```
核心状态边界：
- `VacuumCostBalance` 是 backend-local 的短期 debt。
- `vacuum_cost_delay` / `vacuum_cost_limit` 是当前执行的有效参数。
- `VacuumCostActive` 表示当前 backend 是否启用 cost delay。
- `WorkerInfoData.wi_dobalance` 表示 autovacuum worker 是否参与实例级 balance。
- `AutoVacuumShmem->av_nworkersForBalance` 是当前参与分摊的 worker 数。
- `VacuumSharedCostBalance` 是 parallel vacuum 内部的共享 debt。
- `VacuumFailsafeActive` 会压过普通节流。
ownership / cleanup：
- 普通 cost state 由当前 backend 持有，`vacuum()` 的 `PG_FINALLY` 清理。
- autovacuum worker slot 由 `AutoVacuumShmem` 持有，worker 退出后归还并触发 rebalance。
- relation storage parameters 只影响当前 worker 当前表的有效 cost 参数。
- parallel vacuum shared cost state 随 parallel vacuum 生命周期结束。
- ERROR 不需要专门释放 cost delay 资源，但必须关闭 active 状态、清零 balance，并让 ResourceOwner / transaction cleanup 处理真实资源。
正确性边界：
- cost delay 不决定 tuple 是否可回收。
- cost delay 不保证 index / heap 一致性。
- cost delay 不替代 WAL、locks、pins、snapshot horizon 或 freeze。
- 它只降低 VACUUM 对共享资源的平均消耗速率。
异常路径：
- SIGHUP 可以让 autovacuum worker 在运行中更新 cost 参数。
- relation reloptions 可以退出全局 balance。
- postmaster death 在 sleep 后检查。
- failsafe 会关闭 delay，优先避免 wraparound。
能直接观测：
- `pg_stat_activity.wait_event = 'VacuumDelay'`。
- `pg_stat_progress_vacuum.delay_time`。
- `pg_stat_progress_vacuum.mode = 'failsafe'`。
- `VACUUM VERBOSE` 或 autovacuum log 中的 `delay time`。
- DEBUG2 `VacuumUpdateCosts` 中的 effective cost 参数。
不能直接观测：
- `VacuumCostBalance` 的瞬时值。
- `VacuumCostActive` 的瞬时值。
- `av_nworkersForBalance` 的 SQL 视图。
- parallel vacuum 的 shared cost balance。
可迁移规律：
```text
当后台维护必须持续推进，却又与前台业务共享关键资源时，
系统常用低成本、近似、可在线调整的自愿节流机制保护平均延迟；
但一旦进入 correctness emergency，
系统会撤销温和节流，把资源优先级让给必须完成的安全边界推进。
```
这些判断依赖 workload、硬件、GUC、表级 reloptions、worker 并发、parallel vacuum、checkpoint 状态、WAL 压力和版本实现。
不要把 cost delay 当成完整资源隔离。
也不要把 autovacuum 的温和默认值当成永远正确的生产配置。
本节真正要带走的是边界感：
```text
cost accounting 是近似记账；
delay point 是安全点睡眠；
worker balance 是实例级后台压力平滑；
parallel shared balance 是单表内工作量平滑；
failsafe 是正确性风险压过性能温和性的开关。
```
