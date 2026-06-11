# PostgreSQL 从 Buffers / I/O timing 定位存储访问路径

## 课程定位

前置知识：已经能从 EXPLAIN 找到执行节点入口，也知道 pg_stat 和 EXPLAIN ANALYZE 的观测边界。

本节唯一主问题：

```text
如何沿着 scan node、buffer access、read timing 和 heap / index fetch 计数判断慢 SQL 是缓存未命中、随机 I/O、回表过多还是可见性检查成本？
```

核心矛盾：用户看到的是 Buffers 和 I/O Timings 这种压缩后的输出，但真正成本分散在执行节点、buffer manager、storage manager、read stream 和 pg_stat I/O 计数之间。

学完后应能把一行 `Buffers: shared hit=... read=...` 映射回具体访问路径，而不是把所有 read 都解释成同一种磁盘慢。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 47、48 节原计划从 EXPLAIN 节点名和 actual/estimate 偏差进入源码。
本节接着处理慢 SQL 排查中最常见的第二条线索：Buffers 和 I/O timing。
Buffers 不直接告诉你“哪个函数慢”。
它告诉你一次执行在 shared、local、temp 三类缓冲上发生了多少命中、读取、脏页和写出。
I/O timing 进一步告诉你读写等待时间是否进入本次执行剖面。
本节唯一主问题是把这些观测值沿源码写入点还原成访问路径。
下一节会把同样的方法用在 wait event 上。
04 目录到这里的诊断链路可以压缩成：

```text
EXPLAIN node
  -> actual / estimate gap
  -> Buffers / I/O timing
  -> wait event
  -> profiler stack
  -> extension profiler closed loop
```

本节只处理自己这一环，不把相邻主题扩成第二个主问题。

阅读时要保持一条线：先从用户可见现象进入，再回到状态写入点，最后再回到可验证实验。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
执行节点通过 table AM / index AM 访问 tuple；buffer manager 在命中、读入、写出和临时文件路径上累加 `pgBufferUsage`；Instrumentation 在节点开始和停止时做差；EXPLAIN 再把差值格式化成 Buffers 与 I/O Timings。
```

这里的 tension 是：诊断希望把成本归因到某个 plan node，但 buffer 和 I/O 计数本质上来自更底层的全局计数差值。

因此本节只问一个问题：这些差值能可靠说明哪一段访问路径，不能说明什么。

这个模型的诊断含义是：

- 不要先背函数名，先判断状态属于哪个生命周期边界。

- 不要把用户可见指标当成源码真相，指标只是某个状态边界的投影。

- 不要把一次采样、一次累计值和一次执行剖面混在一起解释。

- 每个观测值都要回答：谁写入，何时可见，谁清理，异常时是否仍可靠。

本节后面的源码 walkthrough 都围绕这一个模型展开。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/executor/instrument.h` | `BufferUsage`、`Instrumentation`、`INSTRUMENT_BUFFERS`、`INSTRUMENT_IO` 的状态边界。 |
| 2 | `src/backend/executor/instrument.c` | `pgBufferUsage`、`InstrStartNode()`、`InstrStopNode()`、`BufferUsageAccumDiff()` 如何把全局计数转成节点差值。 |
| 3 | `src/backend/commands/explain.c` | `show_buffer_usage()` 如何输出 Buffers 和 I/O Timings。 |
| 4 | `src/backend/storage/buffer/bufmgr.c` | `ReadBufferExtended()`、`StartReadBuffers()`、`WaitReadBuffers()`、buffer hit/read/write 计数来源。 |
| 5 | `src/backend/storage/buffer/localbuf.c` | local buffer 和 temp/local I/O 计数路径。 |
| 6 | `src/backend/storage/file/buffile.c` | Sort、Hash spill 等临时文件读写如何进入 temp block 计数。 |
| 7 | `src/backend/storage/aio/read_stream.c` | 顺序扫描和预读路径中 read stream 如何推进 Start/Wait。 |
| 8 | `src/backend/utils/activity/pgstat_io.c` | `pgstat_prepare_io_time()`、`pgstat_count_io_op_time()` 如何把 I/O timing 写入统计。 |
| 9 | `src/backend/executor/nodeSeqscan.c` | SeqScan 如何通过 table scan 路径触发底层读。 |
| 10 | `src/backend/executor/nodeIndexscan.c` | IndexScan 如何把索引访问、回表和节点 instrumentation 连起来。 |
| 11 | `src/backend/access/heap/heapam_handler.c` | heap table AM scan 如何通过 `ReadBufferExtended()` 访问 heap page。 |

阅读顺序按 mental model 排列，不按文件名排序。

建议先从入口和状态结构读起，再追 ownership、cleanup、异常路径和观测输出。

本节源码核对基线：

```text
cd /home/highgo/postgres
git rev-parse --abbrev-ref HEAD
git rev-parse HEAD
```

期望分支为 `master`，提交为本课开头列出的完整提交号。

## 4. 关键数据结构与状态

### 4.1. `pgBufferUsage`

backend-local 的累计计数器，执行期间不断增加，不能重置为单次执行值。

它的持有者是：当前 backend 的执行路径和存储路径。

它的读取者是：Instrumentation、EXPLAIN、pg_stat_statements 等差值计算者。

生命周期边界是：backend 生命周期内持续累加，某段观测通过起止快照做差。

诊断价值是：判断 Buffers 输出是本节点差值、查询差值还是扩展自己截取的差值。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何沿着 scan node、buffer access、read timing 和 heap / index fetch 计数判断慢 SQL 是缓存未命中、随机 I/O、回表过多还是可见性检查成本？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.2. `BufferUsage.shared_blks_hit/read`

shared buffer 命中和需要读入的块数。

它的持有者是：buffer manager。

它的读取者是：EXPLAIN、pg_stat_statements、手工 instrumentation。

生命周期边界是：在访问 shared buffer 时累加，不表示单个 relation 的完整事实。

诊断价值是：区分缓存命中多、真实读多、还是 loops 放大造成读数看起来大。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何沿着 scan node、buffer access、read timing 和 heap / index fetch 计数判断慢 SQL 是缓存未命中、随机 I/O、回表过多还是可见性检查成本？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.3. `BufferUsage.temp_blks_read/written`

临时文件路径上的块计数。

它的持有者是：BufFile、tuplesort、hash spill 等节点。

它的读取者是：EXPLAIN 和累计统计。

生命周期边界是：通常随 Sort、Hash、Material、Agg spill 出现。

诊断价值是：判断慢点是否已经从 buffer cache 转向 work_mem / temp file 边界。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何沿着 scan node、buffer access、read timing 和 heap / index fetch 计数判断慢 SQL 是缓存未命中、随机 I/O、回表过多还是可见性检查成本？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.4. `shared_blk_read_time`

开启 `track_io_timing` 后累加 shared read 的时间。

它的持有者是：底层读路径和 pgstat I/O 计时函数。

它的读取者是：EXPLAIN I/O Timings 和统计视图。

生命周期边界是：只有开启计时时才有意义，且计时本身有成本。

诊断价值是：判断 shared read 是否真的花时间，还是 read 计数来自较快缓存层。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何沿着 scan node、buffer access、read timing 和 heap / index fetch 计数判断慢 SQL 是缓存未命中、随机 I/O、回表过多还是可见性检查成本？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.5. `NodeInstrumentation.bufusage_start`

节点开始执行时的 buffer usage 快照。

它的持有者是：Instrumentation wrapper。

它的读取者是：`InstrStopNode()`。

生命周期边界是：每次节点 start/stop 周期内有效。

诊断价值是：判断节点级 Buffers 为什么是差值而不是节点私有计数器。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何沿着 scan node、buffer access、read timing 和 heap / index fetch 计数判断慢 SQL 是缓存未命中、随机 I/O、回表过多还是可见性检查成本？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.6. `ReadBuffersOperation`

批量读 buffer 时在 Start/Wait 之间保存读请求状态。

它的持有者是：buffer manager 和 read stream。

它的读取者是：顺序扫描、预读路径和等待路径。

生命周期边界是：一次读批次内有效。

诊断价值是：解释为什么 I/O 可以先发起、后等待，等待时间不一定完全贴在读请求发起点。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何沿着 scan node、buffer access、read timing 和 heap / index fetch 计数判断慢 SQL 是缓存未命中、随机 I/O、回表过多还是可见性检查成本？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.7. `TableScanInstrumentation / IndexScanInstrumentation`

扫描节点额外的节点级统计，例如 read stream I/O 形态或 index search。

它的持有者是：具体 scan node。

它的读取者是：EXPLAIN 节点输出。

生命周期边界是：PlanState 生命周期内保存，parallel worker 需要汇总。

诊断价值是：把 generic Buffers 和具体 scan 行为联系起来。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：如何沿着 scan node、buffer access、read timing 和 heap / index fetch 计数判断慢 SQL 是缓存未命中、随机 I/O、回表过多还是可见性检查成本？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

本节最重要的不变量是：

```text
raw field 不是语义；
field + writer + reader + lifecycle + cleanup 才是诊断语义。
```

## 5. 主流程源码 walkthrough

下面按一次慢查询诊断或 profiler 采样的时间线走主流程。

```text
ExecutorStart() 设置 instrumentation options
ExecInitNode() 构造 PlanState tree
ExecProcNodeFirst() 安装 instrumentation wrapper
ExecProcNodeInstr() 调用 InstrStartNode()
节点通过 table AM / index AM 访问数据
ReadBufferExtended() / StartReadBuffers() 推进 buffer 访问
InstrStopNode() 做 BufferUsageAccumDiff()
ExplainNode() / show_buffer_usage() 输出
```

### 5.1. ExecutorStart() 设置 instrumentation options

EXPLAIN ANALYZE 或 auto_explain 打开 buffers/io 选项时，`QueryDesc` 和 `EState` 会携带 instrumentation 需求。

这一段改变的状态边界是：是否分配节点级 `NodeInstrumentation`。

回到诊断时要验证：本次执行是否真的启用了 `INSTRUMENT_BUFFERS` 或 `INSTRUMENT_IO`。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.2. ExecInitNode() 构造 PlanState tree

每个节点在初始化时得到自己的运行时状态，扫描节点同时准备 table/index AM 所需状态。

这一段改变的状态边界是：Plan 到 PlanState 的状态化边界。

回到诊断时要验证：慢点是否归属于 scan node、join node、Sort/Hash 这种 blocking node。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.3. ExecProcNodeFirst() 安装 instrumentation wrapper

第一次执行节点时，如果存在 instrumentation，`ExecProcNode` 会转向 `ExecProcNodeInstr()`。

这一段改变的状态边界是：节点调用边界。

回到诊断时要验证：Buffers 是在节点 start/stop 之间做差，不是每个节点各自维护一套 buffer manager。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.4. ExecProcNodeInstr() 调用 InstrStartNode()

节点开始前保存 `pgBufferUsage` 和计时起点。

这一段改变的状态边界是：全局累计计数到节点局部差值的起点。

回到诊断时要验证：若节点没有 instrumentation，则不能期待节点级 Buffers。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.5. 节点通过 table AM / index AM 访问数据

SeqScan、IndexScan、BitmapHeapScan 等节点进入 heap/index/table AM，最终触发 buffer manager 读页。

这一段改变的状态边界是：执行器与存储层边界。

回到诊断时要验证：read/hit 的来源在 buffer manager，不在 EXPLAIN 输出函数。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.6. ReadBufferExtended() / StartReadBuffers() 推进 buffer 访问

命中直接累加 hit；未命中进入读入、等待或预读路径，并按 shared/local/temp 分类更新计数。

这一段改变的状态边界是：buffer cache 与 storage manager 边界。

回到诊断时要验证：read 增加不等价于慢，必须结合 timing、wait event 和访问模式。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.7. InstrStopNode() 做 BufferUsageAccumDiff()

节点返回 tuple 或结束本次调用时，把当前 `pgBufferUsage` 减去 start 快照。

这一段改变的状态边界是：节点级观测值形成点。

回到诊断时要验证：loops 多的节点会把多轮差值累积，不能只看单轮。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.8. ExplainNode() / show_buffer_usage() 输出

EXPLAIN 遍历 PlanState，把节点 instrumentation 中的 buffer usage 格式化成文本或结构化字段。

这一段改变的状态边界是：源码状态到用户可见输出的边界。

回到诊断时要验证：输出格式不是事实来源，事实来源是 instrumentation 差值。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

## 6. 生命周期 / ownership / cleanup

### 6.1. 查询 instrumentation 创建

创建或进入者：ExecutorStart / standard_ExecutorStart。

正常清理者：ExecutorEnd 和 executor memory context reset。

异常路径依赖：ExecutorEnd hook 或 ERROR cleanup 必须不留下跨查询状态。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.2. 节点 buffer usage 快照

创建或进入者：InstrStartNode。

正常清理者：InstrStopNode。

异常路径依赖：如果节点 ERROR，中途快照不会自然变成可信输出。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.3. shared/local/temp 计数累计

创建或进入者：buffer manager、localbuf、buffile。

正常清理者：backend 退出时随进程消失。

异常路径依赖：它们是累计值，单次执行必须通过差值解释。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.4. I/O timing

创建或进入者：开启 `track_io_timing` 后的底层读写路径。

正常清理者：跟随 BufferUsage 差值或 pg_stat flush。

异常路径依赖：关闭计时时不能把 0 解释成没有 I/O 等待。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.5. parallel worker buffer usage

创建或进入者：worker 本地累计后通过共享 instrumentation 汇总。

正常清理者：ExecParallelFinish / instrumentation merge。

异常路径依赖：leader 与 worker 的归因需要看 per-worker 输出。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.6. EXPLAIN 输出对象

创建或进入者：ExplainState 和 per-node traversal。

正常清理者：命令结束后释放。

异常路径依赖：格式化失败不应改变执行结果语义。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

## 7. 正确性机制层次

本节的正确性不是一个机制单独保证的。

下面这些层次组合起来，才让观测结果既能靠近现场，又不破坏执行语义。

| 层次 | 机制 | 本节不变量 | 常见误读 |
| --- | --- | --- | --- |
| 差值边界 | `InstrStartNode()` / `InstrStopNode()` | 节点 Buffers 是 start/stop 之间的累计差。 | 每个节点自己拥有独立 buffer 计数器。 |
| 分类边界 | shared/local/temp fields | 计数按 buffer 类型和临时文件类型分类。 | shared read 一定来自某张表的 heap 主 fork。 |
| 计时边界 | `track_io_timing` 和 `pgstat_prepare_io_time()` | 只有开启后才累加 I/O 时间。 | 没有 timing 就没有 I/O。 |
| 访问方法边界 | table AM / index AM / buffer manager | 执行器不直接理解 heap page 读写细节。 | 看到 Seq Scan 就能直接定位到 md.c 某一行。 |
| 并行边界 | worker instrumentation merge | worker 计数需要回收到 leader 输出。 | leader 的 Buffers 就是所有 worker 细节。 |
| 输出边界 | `show_buffer_usage()` | 输出只展示已有计数，不重新测量。 | 改输出格式能改变统计语义。 |

读源码时可以按这个顺序检查：

```text
先找写入点
  -> 再找读取点
  -> 再找清理点
  -> 再看 ERROR / abort / backend exit 是否能兜底
  -> 最后把用户可见输出映射回这一条状态链
```

1. `差值边界` 这一层主要依赖 `InstrStartNode()` / `InstrStopNode()`。

它保证的是：节点 Buffers 是 start/stop 之间的累计差。

不要把它误读成：每个节点自己拥有独立 buffer 计数器。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

2. `分类边界` 这一层主要依赖 shared/local/temp fields。

它保证的是：计数按 buffer 类型和临时文件类型分类。

不要把它误读成：shared read 一定来自某张表的 heap 主 fork。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

3. `计时边界` 这一层主要依赖 `track_io_timing` 和 `pgstat_prepare_io_time()`。

它保证的是：只有开启后才累加 I/O 时间。

不要把它误读成：没有 timing 就没有 I/O。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

4. `访问方法边界` 这一层主要依赖 table AM / index AM / buffer manager。

它保证的是：执行器不直接理解 heap page 读写细节。

不要把它误读成：看到 Seq Scan 就能直接定位到 md.c 某一行。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

5. `并行边界` 这一层主要依赖 worker instrumentation merge。

它保证的是：worker 计数需要回收到 leader 输出。

不要把它误读成：leader 的 Buffers 就是所有 worker 细节。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

6. `输出边界` 这一层主要依赖 `show_buffer_usage()`。

它保证的是：输出只展示已有计数，不重新测量。

不要把它误读成：改输出格式能改变统计语义。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

## 8. 错误路径 / 异常路径 / fallback

### 8.1. read 高但 timing 低

可见现象：Buffers read 很多，I/O Timings 却很小。

源码上应先回到：`bufmgr.c` 和 `pgstat_io.c`。

正确处理方式是：检查 OS cache、预读、timing 是否开启，以及 read 是否来自较快路径。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.2. hit 高但仍然慢

可见现象：shared hit 很高，SQL 仍然 CPU 慢。

源码上应先回到：`execExprInterp.c`、`heaptuple.c`、scan node。

正确处理方式是：转向 CPU profile、tuple deform、qual/filter、visibility 检查，而不是继续追磁盘。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.3. temp blocks 出现

可见现象：Sort/Hash/Agg 节点显示 temp read/write。

源码上应先回到：`buffile.c`、`tuplesort.c`、`nodeHash.c`。

正确处理方式是：把问题边界切到 work_mem、spill 批次和 blocking node。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.4. Index Scan read 高

可见现象：索引节点或上层回表节点 read 多。

源码上应先回到：`nodeIndexscan.c`、`heapam_handler.c`。

正确处理方式是：区分索引页访问、heap fetch、可见性检查和随机回表。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.5. 并行 Buffers 难解释

可见现象：总 Buffers 不高但某些 worker 慢。

源码上应先回到：`execParallel.c`、scan worker instrumentation。

正确处理方式是：看 per-worker 输出和倾斜，不只看 leader 汇总。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

## 9. 成本、资源与跨模块传播

成本分析要围绕同一个主问题。

不要把所有可能成本都列一遍。

### 9.1. buffer miss

主要扩张因子：访问块数和缓存冷热。

放大方式：shared read 上升，可能伴随 IO timing 或 wait event。

控制办法：用预热、索引选择、访问顺序和 relation size 验证。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.2. random heap fetch

主要扩张因子：Index Scan 回表次数。

放大方式：read/hit 分散到大量 heap page。

控制办法：比较 Index Only Scan、covering index、聚簇度和过滤率。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.3. visibility cost

主要扩张因子：tuple 数和版本链长度。

放大方式：Buffers 不一定高，但 CPU profile 可能落在 visibility / deform。

控制办法：结合 rows removed、heap tuple visibility 函数和 VACUUM 状态。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.4. spill

主要扩张因子：work_mem、batch 数、输入行数。

放大方式：temp blocks 和 temp timing 上升。

控制办法：看 Sort Method、Hash Batches、Disk Usage。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.5. instrumentation overhead

主要扩张因子：节点数量和 loops。

放大方式：开启 buffers/io/timing 后每个节点 start/stop 都有额外差值和计时。

控制办法：必要时关闭 timing 或换 pg_stat_statements。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.6. parallel merge

主要扩张因子：worker 数和节点树大小。

放大方式：leader 汇总 worker BufferUsage 与 per-worker 输出。

控制办法：定位倾斜时优先看 worker 细节。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

## 10. 观测与诊断入口

观测入口要分清当前值、累计值、单次执行剖面和外部采样。

同一个慢 SQL 结论最好至少由两类信号互相校验。

### 10.1. EXPLAIN (ANALYZE, BUFFERS, I/O)

能看到：单次执行的节点级 buffer 和 I/O timing。

看不到：不能证明长期平均，也不能直接给出 OS cache 命中。

源码入口：`explain.c`、`instrument.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.2. pg_stat_statements

能看到：按 query id 聚合的 block I/O 和时间。

看不到：不能还原某一次执行的具体 plan node。

源码入口：`contrib/pg_stat_statements/pg_stat_statements.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.3. pg_stat_io

能看到：backend type、context、object、op 维度的累计 I/O。

看不到：不是某个 plan node 的完整切片。

源码入口：`pgstat_io.c`、`pgstatfuncs.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.4. pg_stat_activity wait_event

能看到：当前是否卡在 DataFileRead 等等待点。

看不到：不能表示历史累计耗时。

源码入口：`wait_event.c`、`waiteventset.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.5. perf / eBPF

能看到：CPU 栈和内核 I/O 栈。

看不到：不能自动映射到 SQL 语义。

源码入口：外部 profiler + executor symbols。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.6. gdb 断点

能看到：读写计数更新点和调用栈。

看不到：线上使用风险高。

源码入口：`ReadBufferExtended()`、`InstrStopNode()`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

## 11. 常见误区

下面这些误区都来自把观测值当成源码事实。

误区一：看到一个指标就直接给结论。

正确做法是先找到写入点、读取点和清理点。

误区二：把单次执行剖面当成长期平均。

正确做法是把 EXPLAIN、pg_stat、wait event 和 profiler 的时间窗口写清楚。

误区三：把父节点时间当成父节点独占 CPU。

正确做法是区分 inclusive、exclusive、loops 和子节点成本。

误区四：把没有观测值解释成没有问题。

正确做法是检查是否关闭开关、未命中采样、被阈值过滤、被 ring 覆盖或权限隐藏。

误区五：把当前源码实现说成跨版本契约。

正确做法是写明本课基于 `/home/highgo/postgres` 的 `master` 分支和 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。

## 12. 课堂实验

### 12.1. 冷缓存与热缓存对比

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
EXPLAIN (ANALYZE, BUFFERS, I/O) SELECT count(*) FROM big_table;
重复执行同一条 SQL。
对比 shared read、shared hit 和 I/O timing。
```

预期现象：第一次更可能出现 read，第二次更可能转为 hit；是否明显取决于 OS cache 和 shared_buffers。

回到源码时检查：`ReadBufferExtended()` 命中和未命中路径。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.2. Index Scan 回表定位

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
CREATE INDEX ON t(k);
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM t WHERE k BETWEEN 1 AND 10000;
再改成只读索引覆盖列。
```

预期现象：回表多时 heap 访问会放大 Buffers；覆盖列可能转向 Index Only Scan。

回到源码时检查：`nodeIndexscan.c` 与 heap fetch 路径。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.3. Sort spill 观察

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
SET work_mem = '1MB';
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM t ORDER BY wide_col;
提高 work_mem 后重跑。
```

预期现象：低 work_mem 可能出现 temp blocks 和 Disk Usage。

回到源码时检查：`tuplesort.c`、`buffile.c`、`show_buffer_usage()`。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.4. 关闭 timing 的差异

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
SET track_io_timing = off;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM t;
SET track_io_timing = on;
EXPLAIN (ANALYZE, BUFFERS, I/O) SELECT * FROM t;
```

预期现象：Buffers 仍可见，但 I/O Timings 是否出现取决于计时开关。

回到源码时检查：`pgstat_prepare_io_time()` 和 `show_buffer_usage()`。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

## 13. 讨论题

1. 为什么 `shared_blks_read` 不能直接等价为物理磁盘读？

2. 一个 Index Scan 慢，但 Buffers hit 很高，下一步应查什么？

3. 为什么节点级 Buffers 是差值而不是存储层直接按节点写入？

4. 并行查询中只看总 Buffers 会遗漏什么？

5. 开启 I/O timing 的收益和成本边界在哪里？

## 14. 本节小结

本节只沉淀一个模型：

```text
Buffers 是 buffer manager 累计计数在 instrumentation 边界上的差值。
I/O Timings 只有在计时开启时才表示读写时间。
shared/local/temp 分类先说明访问对象类型，再结合节点判断访问路径。
read 多不必然慢，hit 多也不必然快。
慢 SQL 诊断要把 EXPLAIN 输出、wait event 和 CPU profile 合起来看。
```

下一节会把同样的“观测值回源码”方法用于 wait event。
