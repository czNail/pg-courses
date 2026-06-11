# PostgreSQL 从 wait event 定位并发瓶颈源码

## 课程定位

前置知识：已经理解 Buffers / I/O timing 的来源，也知道 wait_event_info 是当前 backend 状态而不是历史累计。

本节唯一主问题：

```text
当慢 SQL 卡在 lock、buffer pin、LWLock、IPC 或 client wait 时，如何从 wait event 名称反查源码等待点和相关共享状态？
```

核心矛盾：wait event 名称很像答案，但它只是等待点写入的四字节状态；真正的阻塞原因可能在锁表、buffer pin、latch、condition variable、客户端或并行 worker 通信里。

学完后应能从 pg_stat_activity 中的 wait_event_type/name 反查到设置点、等待对象和 cleanup 边界。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节从 Buffers 和 I/O timing 回到存储访问路径。
本节处理另一类慢 SQL 现场：SQL 没有明显消耗 CPU，也没有持续增长 rows，却在某个等待点停住。
wait event 的优势是当前性。
它的弱点也是当前性：采样时看到什么，只说明采样瞬间 backend 报告了哪个等待点。
本节唯一主问题是从这个等待名反查源码，而不是重新讲锁管理器或 buffer manager 的完整设计。
下一节会进入 CPU profile，把没有等待事件的热点映射回执行器函数。

```text
pg_stat_activity.wait_event
  -> wait_event_type / wait_event
  -> pgstat_get_wait_event() decode
  -> wait path setter
  -> shared state or latch object
  -> owning subsystem
```

本节只处理自己这一环，不把相邻主题扩成第二个主问题。

阅读时要保持一条线：先从用户可见现象进入，再回到状态写入点，最后再回到可验证实验。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
等待路径在进入阻塞前调用 `pgstat_report_wait_start()` 或通过 WaitEventSet/LWLock/ConditionVariable 间接设置 `wait_event_info`；阻塞结束后清零；SQL 层读取 PGPROC 中的当前值并解码。
```

这里的 tension 是：为了低成本，wait event 只记录压缩后的等待类别和 id；为了诊断，又必须能从这个 id 回到具体源码等待点。

本节只问一个问题：看到一个 wait event 后，怎样沿写入点找到真正的共享状态和持有者。

这个模型的诊断含义是：

- 不要先背函数名，先判断状态属于哪个生命周期边界。

- 不要把用户可见指标当成源码真相，指标只是某个状态边界的投影。

- 不要把一次采样、一次累计值和一次执行剖面混在一起解释。

- 每个观测值都要回答：谁写入，何时可见，谁清理，异常时是否仍可靠。

本节后面的源码 walkthrough 都围绕这一个模型展开。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/utils/wait_event.h` | `pgstat_report_wait_start()`、`pgstat_report_wait_end()` 的内联写入边界。 |
| 2 | `src/backend/utils/activity/wait_event.c` | `pgstat_get_wait_event_type()`、`pgstat_get_wait_event()` 和自定义 wait event 名称解析。 |
| 3 | `src/include/storage/proc.h` | `PGPROC.wait_event_info`、buffer pin wait 相关字段。 |
| 4 | `src/backend/utils/adt/pgstatfuncs.c` | pg_stat_activity 相关函数如何读取 backend status。 |
| 5 | `src/backend/storage/lmgr/lwlock.c` | LWLock 等待如何用 tranche 设置 wait event。 |
| 6 | `src/backend/storage/lmgr/proc.c` | `ProcSleep()`、`ProcWaitForSignal()` 等 lock/signal 等待路径。 |
| 7 | `src/backend/storage/lmgr/condition_variable.c` | `ConditionVariableSleep()` 的 wait event 参数和清理。 |
| 8 | `src/backend/storage/ipc/waiteventset.c` | `WaitEventSetWait()` 统一等待入口。 |
| 9 | `src/backend/storage/ipc/latch.c` | `WaitLatch()` / `WaitLatchOrSocket()` 到 WaitEventSet 的封装。 |
| 10 | `src/backend/libpq/be-secure.c` | ClientRead / ClientWrite 等客户端等待点。 |
| 11 | `src/backend/libpq/pqmq.c` | 并行查询 shm_mq 等 IPC 等待路径。 |

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

### 4.1. `PGPROC.wait_event_info`

shared memory 中当前等待信息，pg_stat_activity 读取它。

它的持有者是：当前 backend 的等待路径。

它的读取者是：其他 backend 通过 backend status / pg_stat_activity 读取。

生命周期边界是：backend 注册 PGPROC 后有效，等待结束应清零。

诊断价值是：判断采样瞬间 backend 等的是哪类对象。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：当慢 SQL 卡在 lock、buffer pin、LWLock、IPC 或 client wait 时，如何从 wait event 名称反查源码等待点和相关共享状态？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.2. `my_wait_event_info`

当前进程写 wait event 的目标指针。

它的持有者是：backend 初始化和 wait_event 模块。

它的读取者是：所有等待上报函数。

生命周期边界是：早期指向本地变量，MyProc 就绪后指向 PGPROC 字段。

诊断价值是：解释为什么早期路径也能安全调用 wait start/end。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：当慢 SQL 卡在 lock、buffer pin、LWLock、IPC 或 client wait 时，如何从 wait event 名称反查源码等待点和相关共享状态？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.3. `wait event class`

高位类别，区分 LWLock、Lock、Buffer、Client、IPC、IO 等。

它的持有者是：wait event 常量定义和写入点。

它的读取者是：解码函数和 SQL 层。

生命周期边界是：随等待写入和清理。

诊断价值是：先把诊断空间缩到某个子系统。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：当慢 SQL 卡在 lock、buffer pin、LWLock、IPC 或 client wait 时，如何从 wait event 名称反查源码等待点和相关共享状态？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.4. `LWLock tranche`

LWLock wait event 中的具体 tranche id。

它的持有者是：LWLock 初始化和等待路径。

它的读取者是：`GetLWLockIdentifier()`。

生命周期边界是：锁存在期间稳定。

诊断价值是：区分 BufferMappingLock、ProcArrayLock、WALWriteLock 等共享结构。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：当慢 SQL 卡在 lock、buffer pin、LWLock、IPC 或 client wait 时，如何从 wait event 名称反查源码等待点和相关共享状态？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.5. `LOCKTAG`

heavyweight lock 等待的逻辑对象标签。

它的持有者是：lock manager。

它的读取者是：死锁检测、pg_locks、wait event 解码。

生命周期边界是：锁请求生命周期内有效。

诊断价值是：把 Lock wait 从名字推进到 relation、transactionid、tuple 等对象。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：当慢 SQL 卡在 lock、buffer pin、LWLock、IPC 或 client wait 时，如何从 wait event 名称反查源码等待点和相关共享状态？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.6. `ConditionVariable`

条件变量等待对象，调用方传入 wait_event_info。

它的持有者是：具体子系统的 shared state。

它的读取者是：等待 backend 和唤醒者。

生命周期边界是：条件满足或进程中断时退出等待。

诊断价值是：判断等待名背后的条件是谁推进。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：当慢 SQL 卡在 lock、buffer pin、LWLock、IPC 或 client wait 时，如何从 wait event 名称反查源码等待点和相关共享状态？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.7. `Latch / WaitEventSet`

统一等待文件描述符、latch、socket、timeout 的对象。

它的持有者是：调用方和 latch 模块。

它的读取者是：事件循环和 wait_event 输出。

生命周期边界是：一次等待调用或长期 wait set 生命周期。

诊断价值是：定位 Client、IPC、后台进程主循环等待。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：当慢 SQL 卡在 lock、buffer pin、LWLock、IPC 或 client wait 时，如何从 wait event 名称反查源码等待点和相关共享状态？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

本节最重要的不变量是：

```text
raw field 不是语义；
field + writer + reader + lifecycle + cleanup 才是诊断语义。
```

## 5. 主流程源码 walkthrough

下面按一次慢查询诊断或 profiler 采样的时间线走主流程。

```text
采样 pg_stat_activity
pg_stat_activity 读取 backend status
pgstat_get_wait_event_type() 解 class
pgstat_get_wait_event() 解具体名
回到写入点
找到等待对象
检查清理路径
合并其它信号
```

### 5.1. 采样 pg_stat_activity

诊断从 `wait_event_type` 和 `wait_event` 开始，但只把它当作当前值。

这一段改变的状态边界是：SQL 可见状态边界。

回到诊断时要验证：是否反复采样都落在同一等待点。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.2. pg_stat_activity 读取 backend status

`pgstatfuncs.c` 读取 backend status entry，间接拿到 PGPROC 中的 wait_event_info。

这一段改变的状态边界是：共享状态读取边界。

回到诊断时要验证：采样不是阻塞链证明。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.3. pgstat_get_wait_event_type() 解 class

高位 class 被转成 LWLock、Lock、Buffer、Client、IPC、IO 等类型。

这一段改变的状态边界是：分类边界。

回到诊断时要验证：先按 class 找子系统，不要直接搜索所有源码。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.4. pgstat_get_wait_event() 解具体名

不同 class 走不同解码函数；LWLock 会看 tranche，Lock 会看 lock tag type。

这一段改变的状态边界是：名称解析边界。

回到诊断时要验证：名称是否是内置、extension 或 injection point。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.5. 回到写入点

在源码中搜索 `WAIT_EVENT_...` 或 tranche 名，定位调用 `pgstat_report_wait_start()`、`WaitEventSetWait()`、`LWLockAcquire()` 的路径。

这一段改变的状态边界是：源码等待点。

回到诊断时要验证：等待点是否包住真正阻塞调用。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.6. 找到等待对象

对 Lock 看 LOCKTAG 和 pg_locks；对 LWLock 看 tranche；对 BufferPin 看 buffer pin 持有者；对 Client 看 socket 方向。

这一段改变的状态边界是：共享对象边界。

回到诊断时要验证：是否能找到持有者或推进者。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.7. 检查清理路径

等待结束应调用 `pgstat_report_wait_end()` 或由 wrapper 清理。

这一段改变的状态边界是：cleanup 边界。

回到诊断时要验证：是否可能 stale wait event。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.8. 合并其它信号

结合 pg_locks、EXPLAIN、Buffers、perf 和日志确认等待是否解释总耗时。

这一段改变的状态边界是：诊断闭环。

回到诊断时要验证：是否只是一次短暂采样误导。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

## 6. 生命周期 / ownership / cleanup

### 6.1. wait storage 初始化

创建或进入者：backend startup 调用 `pgstat_set_wait_event_storage()`。

正常清理者：shutdown 前 `pgstat_reset_wait_event_storage()`。

异常路径依赖：MyProc 未就绪前使用本地 storage。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.2. 进入等待

创建或进入者：等待路径写入 wait_event_info。

正常清理者：等待结束清零。

异常路径依赖：ERROR 或中断路径要依赖 wrapper 或 PG_FINALLY 风格清理。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.3. LWLock 等待

创建或进入者：LWLockAcquire 内部排队并报告 wait start。

正常清理者：获取锁或退出等待后 wait end。

异常路径依赖：如果等待名存在但锁已释放，说明采样已过期。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.4. heavyweight lock 等待

创建或进入者：ProcSleep 管理等待队列和 deadlock 检查。

正常清理者：授予锁、中断或 ERROR 后退出。

异常路径依赖：pg_locks 才能补充锁对象和阻塞者。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.5. ConditionVariable 等待

创建或进入者：调用方循环检查条件并 sleep。

正常清理者：条件满足后 cancel sleep。

异常路径依赖：必须重新检查条件，不能把唤醒等价为条件成立。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.6. Client / IPC 等待

创建或进入者：WaitEventSet/Latch/socket 路径。

正常清理者：socket 可读写、latch set、timeout 或 postmaster death。

异常路径依赖：等待原因可能在外部客户端或并行 worker。

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
| 当前状态 | `PGPROC.wait_event_info` | 只表达采样瞬间正在等什么。 | 历史耗时累计。 |
| 写入清理 | `pgstat_report_wait_start/end` | 等待点必须成对或由 wrapper 保证清理。 | 写入一次后自动过期。 |
| 类别解码 | `WAIT_EVENT_CLASS_MASK` | 先定位子系统类别。 | 类别就是根因。 |
| 锁对象 | pg_locks / LOCKTAG | heavyweight lock 能关联逻辑对象和阻塞者。 | wait event 名称包含完整对象。 |
| LWLock tranche | tranche id | LWLock 能关联共享结构类型。 | 能直接告诉哪个 backend 持有。 |
| 中断语义 | CHECK_FOR_INTERRUPTS / latch | 等待路径必须响应取消和 postmaster death。 | 等待不可中断才能正确。 |

读源码时可以按这个顺序检查：

```text
先找写入点
  -> 再找读取点
  -> 再找清理点
  -> 再看 ERROR / abort / backend exit 是否能兜底
  -> 最后把用户可见输出映射回这一条状态链
```

1. `当前状态` 这一层主要依赖 `PGPROC.wait_event_info`。

它保证的是：只表达采样瞬间正在等什么。

不要把它误读成：历史耗时累计。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

2. `写入清理` 这一层主要依赖 `pgstat_report_wait_start/end`。

它保证的是：等待点必须成对或由 wrapper 保证清理。

不要把它误读成：写入一次后自动过期。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

3. `类别解码` 这一层主要依赖 `WAIT_EVENT_CLASS_MASK`。

它保证的是：先定位子系统类别。

不要把它误读成：类别就是根因。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

4. `锁对象` 这一层主要依赖 pg_locks / LOCKTAG。

它保证的是：heavyweight lock 能关联逻辑对象和阻塞者。

不要把它误读成：wait event 名称包含完整对象。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

5. `LWLock tranche` 这一层主要依赖 tranche id。

它保证的是：LWLock 能关联共享结构类型。

不要把它误读成：能直接告诉哪个 backend 持有。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

6. `中断语义` 这一层主要依赖 CHECK_FOR_INTERRUPTS / latch。

它保证的是：等待路径必须响应取消和 postmaster death。

不要把它误读成：等待不可中断才能正确。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

## 8. 错误路径 / 异常路径 / fallback

### 8.1. stale wait event

可见现象：backend 看似一直等待，但业务已推进。

源码上应先回到：`pgstat_report_wait_end()` 调用路径。

正确处理方式是：检查是否有异常路径漏清理，或采样来自已变化状态。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.2. Lock wait 无法解释对象

可见现象：只看到 wait_event=transactionid 或 relation。

源码上应先回到：`proc.c`、`lock.c`、`pg_locks`。

正确处理方式是：用 pg_locks 查锁对象和 blocking pid。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.3. LWLock wait 频繁但短

可见现象：采样偶尔看到 BufferMapping 或 WALWrite。

源码上应先回到：`lwlock.c`。

正确处理方式是：通过高频采样或 perf 判断是否真是瓶颈。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.4. ClientRead 慢 SQL

可见现象：backend 等客户端读请求或发送结果。

源码上应先回到：`be-secure.c`、`pqcomm.c`。

正确处理方式是：把问题边界转向客户端、网络和 DestReceiver。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.5. IPC 等待

可见现象：Gather / parallel worker 相关 wait event。

源码上应先回到：`pqmq.c`、`execParallel.c`。

正确处理方式是：判断是 worker 慢、leader 等 tuple queue，还是消息队列背压。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

## 9. 成本、资源与跨模块传播

成本分析要围绕同一个主问题。

不要把所有可能成本都列一遍。

### 9.1. 采样误差

主要扩张因子：采样频率和等待持续时间。

放大方式：短等待容易被漏掉，长等待容易被过度解释。

控制办法：重复采样并结合累计指标。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.2. 锁队列

主要扩张因子：等待者数量和锁模式冲突。

放大方式：ProcSleep、deadlock 检测、唤醒成本增加。

控制办法：用 pg_locks 和事务边界定位持有者。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.3. LWLock 热点

主要扩张因子：backend 数和共享结构竞争。

放大方式：短临界区被高并发放大。

控制办法：结合 tranche 名和 perf 栈。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.4. BufferPin

主要扩张因子：长扫描、cursor、replication conflict。

放大方式：阻塞清理或回收，不一定是锁冲突。

控制办法：查持 pin 路径和长事务。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.5. Client wait

主要扩张因子：结果集大小、网络、客户端消费速度。

放大方式：执行器可能已被输出端背压。

控制办法：看 DestReceiver、socket wait 和客户端。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.6. IPC wait

主要扩张因子：worker 数、tuple queue、leader participation。

放大方式：并行计划等待被归因到消息队列。

控制办法：看 per-worker EXPLAIN 和 leader 行为。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

## 10. 观测与诊断入口

观测入口要分清当前值、累计值、单次执行剖面和外部采样。

同一个慢 SQL 结论最好至少由两类信号互相校验。

### 10.1. pg_stat_activity

能看到：当前 wait_event_type/name。

看不到：不能给出等待持续历史。

源码入口：`pgstatfuncs.c`、`wait_event.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.2. pg_locks

能看到：heavyweight lock 对象和阻塞关系。

看不到：不覆盖 LWLock 和普通 latch。

源码入口：`lockfuncs.c`、`lock.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.3. pg_blocking_pids()

能看到：阻塞当前 lock request 的 backend。

看不到：不解释 LWLock、Client、IPC。

源码入口：`lockfuncs.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.4. wait event source search

能看到：等待常量的写入点。

看不到：不能替代理解等待对象。

源码入口：`rg WAIT_EVENT_...`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.5. perf / eBPF offcpu

能看到：阻塞栈和调度等待。

看不到：需要符号和采样解释。

源码入口：外部 profiler + backend symbols。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.6. 日志与 deadlock report

能看到：长等待或死锁上下文。

看不到：不是所有等待都会记录。

源码入口：`proc.c` deadlock 检查路径。

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

### 12.1. 重现 heavyweight lock wait

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
Session A: BEGIN; UPDATE t SET v = v + 1 WHERE id = 1;
Session B: UPDATE t SET v = v + 1 WHERE id = 1;
Session C: SELECT wait_event_type, wait_event FROM pg_stat_activity WHERE pid = <B>;
```

预期现象：Session B 可能显示 Lock 类等待，并可用 pg_locks 找到对象。

回到源码时检查：`ProcSleep()`、`lock.c` 和 pg_locks 输出。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.2. 观察 ClientRead / ClientWrite

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
运行返回大量结果的查询。
让客户端慢速读取或暂停。
采样 pg_stat_activity。
```

预期现象：backend 可能进入 ClientWrite 或相关 client wait。

回到源码时检查：`be-secure.c`、`pqcomm.c`。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.3. 搜索等待点

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
cd /home/highgo/postgres
rg "WAIT_EVENT_DATA_FILE_READ|WAIT_EVENT_CLIENT_READ|WAIT_EVENT_MESSAGE_QUEUE" src
```

预期现象：从名称反查到具体源码调用点。

回到源码时检查：`wait_event.c` 解码和对应调用者。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.4. 对比等待和 CPU

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
采样 pg_stat_activity wait_event。
同时对 backend pid 跑 perf top 或 perf record。
比较是否 wait_event 为空但 CPU 栈很热。
```

预期现象：wait event 为空不表示没有慢点，可能是 CPU-bound。

回到源码时检查：下一节的 executor hotspot 映射。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

## 13. 讨论题

1. 为什么 wait event 只能作为当前状态而不能直接累计总等待时间？

2. 看到 LWLock wait 时，为什么还需要 tranche 名和调用栈？

3. 为什么 ClientWrite 可能让一个 SQL 看起来慢但不是执行器算得慢？

4. ConditionVariableSleep 为什么必须循环检查条件？

5. 如何判断一次 wait event 采样是否足以作为根因证据？

## 14. 本节小结

本节只沉淀一个模型：

```text
wait event 是当前 backend 写入 PGPROC 的四字节状态。
诊断顺序是 class -> name -> 写入点 -> 等待对象 -> 持有者或推进者。
Lock、LWLock、Buffer、Client、IPC 的源码入口不同，不能混用解释。
wait start/end 的清理边界决定 stale wait event 风险。
等待诊断必须和 pg_locks、Buffers、EXPLAIN、profile 合并。
```

下一节进入没有明显等待时的 CPU profiler 栈映射。
