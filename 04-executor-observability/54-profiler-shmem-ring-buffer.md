# PostgreSQL Profiler 共享内存、环形缓冲与采样输出

## 课程定位

前置知识：已经有 query hook 和节点级计时模型，知道事件可能来自多个 backend。

本节唯一主问题：

```text
profiler 何时需要共享内存 ring buffer，如何控制事件大小、丢弃策略、锁竞争和 backend 退出后的数据可读性？
```

核心矛盾：profiler 事件需要跨 backend 被 SQL 读取，但共享内存是固定大小、需要启动期申请、并且任何写入锁竞争都可能反过来拖慢 executor。

学完后应能设计一个 bounded、可丢弃、lock-light 的事件缓冲，而不是把 profiler 结果随意写进无限增长结构。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节定义了节点级事件的语义。
如果事件只写日志，诊断很难交互式查询。
如果事件只放 backend-local memory，backend 退出后就丢失。
因此扩展常会引入共享内存 ring buffer。
本节唯一主问题是 ring buffer 的共享状态和丢弃策略。
下一节会给 ring buffer 前面加 GUC、采样率和线上保护。

```text
shared_preload_libraries
  -> _PG_init
  -> RequestAddinShmemSpace
  -> RequestNamedLWLockTranche
  -> shmem_startup_hook or callbacks
  -> ShmemInitStruct
  -> backend writes bounded events
  -> SRF reads copy
```

本节只处理自己这一环，不把相邻主题扩成第二个主问题。

阅读时要保持一条线：先从用户可见现象进入，再回到状态写入点，最后再回到可验证实验。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
profiler 在启动期申请固定大小 shared memory 和 named LWLock；初始化时用 `ShmemInitStruct()` 取得 header 与 event array；backend 命中采样后只写固定大小事件，ring 满时按策略覆盖或丢弃；读取端在短锁内复制快照，再在本地格式化输出。
```

这里的 tension 是：诊断希望保留尽可能多细节，执行路径却要求每次写入成本有上界。

本节只问一个问题：如何让共享事件缓冲有固定资源边界和明确丢弃语义。

这个模型的诊断含义是：

- 不要先背函数名，先判断状态属于哪个生命周期边界。

- 不要把用户可见指标当成源码真相，指标只是某个状态边界的投影。

- 不要把一次采样、一次累计值和一次执行剖面混在一起解释。

- 每个观测值都要回答：谁写入，何时可见，谁清理，异常时是否仍可靠。

本节后面的源码 walkthrough 都围绕这一个模型展开。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/storage/ipc/shmem.c` | `RequestAddinShmemSpace()`、`ShmemInitStruct()` 和 shmem index 边界。 |
| 2 | `src/backend/storage/ipc/ipci.c` | shared memory request/init 总流程。 |
| 3 | `src/include/storage/lwlock.h` | `LWLockAcquire()`、`LWLockRelease()` 和 named tranche 使用接口。 |
| 4 | `src/backend/storage/lmgr/lwlock.c` | LWLock 等待和 tranche 名称解析。 |
| 5 | `contrib/pg_stat_statements/pg_stat_statements.c` | 真实扩展示例：shared memory、LWLock、hash table、SRF 读取。 |
| 6 | `src/backend/utils/activity/pgstat_shmem.c` | 共享统计状态和锁保护模式。 |
| 7 | `src/include/storage/shmem.h` | 扩展 shared memory 请求 API 声明。 |
| 8 | `src/backend/utils/misc/guc.c` | 影响 ring 大小的 GUC 何时必须在启动前确定。 |
| 9 | `src/backend/utils/fmgr/funcapi.c` | SRF 输出前复制和本地内存上下文。 |

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

### 4.1. `profiler_shmem_header`

共享区头部，保存 magic、version、capacity、write_pos、dropped。

它的持有者是：postmaster startup / shmem init。

它的读取者是：backend writer、SRF reader。

生命周期边界是：postmaster 生命周期内固定。

诊断价值是：判断 ring 是否属于当前扩展版本和大小。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：profiler 何时需要共享内存 ring buffer，如何控制事件大小、丢弃策略、锁竞争和 backend 退出后的数据可读性？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.2. `event slots`

固定大小事件数组。

它的持有者是：shared memory 初始化。

它的读取者是：backend 写入、SQL 读取。

生命周期边界是：ring capacity 固定，槽位被覆盖或复用。

诊断价值是：保证事件大小有上界。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：profiler 何时需要共享内存 ring buffer，如何控制事件大小、丢弃策略、锁竞争和 backend 退出后的数据可读性？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.3. `write_pos`

下一个写入位置或全局序号。

它的持有者是：backend writer。

它的读取者是：reader 计算窗口。

生命周期边界是：持续单调推进或按 capacity 取模。

诊断价值是：区分覆盖和事件顺序。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：profiler 何时需要共享内存 ring buffer，如何控制事件大小、丢弃策略、锁竞争和 backend 退出后的数据可读性？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.4. `dropped_count`

因 ring 满、采样、锁竞争或事件过大丢弃的数量。

它的持有者是：writer。

它的读取者是：SRF / view。

生命周期边界是：扩展生命周期内累计。

诊断价值是：避免把缺失事件误读成没有慢点。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：profiler 何时需要共享内存 ring buffer，如何控制事件大小、丢弃策略、锁竞争和 backend 退出后的数据可读性？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.5. `profiler_lwlock`

保护 ring header 和 slot 写入的 named LWLock。

它的持有者是：startup request。

它的读取者是：writer/reader。

生命周期边界是：postmaster 生命周期内有效。

诊断价值是：控制共享状态并发访问。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：profiler 何时需要共享内存 ring buffer，如何控制事件大小、丢弃策略、锁竞争和 backend 退出后的数据可读性？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.6. `event payload`

query id、plan node id、pid、timestamp、duration、flags 等值字段。

它的持有者是：profiler hook。

它的读取者是：ring reader。

生命周期边界是：写入后只保存值，不保存 backend 指针。

诊断价值是：跨进程可读的最小事件。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：profiler 何时需要共享内存 ring buffer，如何控制事件大小、丢弃策略、锁竞争和 backend 退出后的数据可读性？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

### 4.7. `reader snapshot`

SRF 在本地 memory context 中复制的一批事件。

它的持有者是：SQL 读取函数。

它的读取者是：tuple forming loop。

生命周期边界是：一次函数调用内有效。

诊断价值是：缩短持锁时间，避免读者阻塞写者。

读这个状态时不要只看字段名。

先问它在 backend-local、shared memory、DSM、文件系统还是调用栈里。

再问它的写入者是否和读取者在同一条时间线上。

最后问它失败时会留下错误值、缺失值，还是只能让上层退化成推断。

在本节唯一主问题中，它只服务于：profiler 何时需要共享内存 ring buffer，如何控制事件大小、丢弃策略、锁竞争和 backend 退出后的数据可读性？

如果这个状态不能解释这一点，就不应该把它扩成另一条主线。

本节最重要的不变量是：

```text
raw field 不是语义；
field + writer + reader + lifecycle + cleanup 才是诊断语义。
```

## 5. 主流程源码 walkthrough

下面按一次慢查询诊断或 profiler 采样的时间线走主流程。

```text
_PG_init() 读取启动期 GUC
RequestAddinShmemSpace()
RequestNamedLWLockTranche()
ShmemInitStruct()
backend 写事件
ring 满时执行策略
SRF 读取复制快照
backend exit 后读取
```

### 5.1. _PG_init() 读取启动期 GUC

ring size、shmem enable、LWLock tranche 数量必须在 shared memory sizing 前确定。

这一段改变的状态边界是：启动期配置边界。

回到诊断时要验证：是否要求 shared_preload_libraries。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.2. RequestAddinShmemSpace()

扩展声明固定 shared memory 大小。

这一段改变的状态边界是：main shmem sizing 边界。

回到诊断时要验证：size 是否由上限 GUC 控制并检查溢出。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.3. RequestNamedLWLockTranche()

声明 profiler 需要的 named LWLock。

这一段改变的状态边界是：锁资源 sizing 边界。

回到诊断时要验证：是否给 wait event 一个可读 tranche 名。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.4. ShmemInitStruct()

postmaster 或第一个 attach 者初始化 header 和 event slots。

这一段改变的状态边界是：共享状态创建/attach 边界。

回到诊断时要验证：found=true/false 两条路径是否一致。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.5. backend 写事件

采样命中后构造固定 payload，短锁内写入 slot，推进 write_pos。

这一段改变的状态边界是：hot path 写入边界。

回到诊断时要验证：是否可能因锁竞争选择丢弃。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.6. ring 满时执行策略

覆盖旧事件、丢弃新事件或按 backend 分区丢弃。

这一段改变的状态边界是：资源上界边界。

回到诊断时要验证：dropped_count 是否可见。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.7. SRF 读取复制快照

读者短锁内复制 header 和 slots 到本地，再释放锁格式化 tuple。

这一段改变的状态边界是：读写隔离边界。

回到诊断时要验证：是否避免长时间持锁输出。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

### 5.8. backend exit 后读取

因为事件是值拷贝，backend 退出后仍可读；但 PID 只作为历史标识。

这一段改变的状态边界是：跨 backend 生命周期边界。

回到诊断时要验证：是否误用已失效指针或 memory context。

注意这里讲的是状态推进，不是函数名背诵。

同一个函数名在不同调用者下可能代表不同成本来源。

只有把调用点、状态变化和可见输出放在一起，才能判断它是不是当前问题的根因。

## 6. 生命周期 / ownership / cleanup

### 6.1. 共享内存请求

创建或进入者：_PG_init / shmem request hook。

正常清理者：postmaster 退出释放 main shmem。

异常路径依赖：未 preload 时不能期望 shmem 存在。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.2. header 初始化

创建或进入者：ShmemInitStruct found=false 分支。

正常清理者：集群重启后重建。

异常路径依赖：版本不匹配应拒绝读取或重置。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.3. event 写入

创建或进入者：executor hook / node wrapper。

正常清理者：slot 被覆盖或 ring reset。

异常路径依赖：ERROR 路径可丢弃当前事件。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.4. 锁持有

创建或进入者：writer/reader 短临界区。

正常清理者：LWLockRelease。

异常路径依赖：不能在持锁时做内存分配、输出格式化或调用可能 ERROR 的复杂逻辑。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.5. reader snapshot

创建或进入者：SRF first call。

正常清理者：SRF done 后释放 multi_call_memory_ctx。

异常路径依赖：读取时 backend 退出不影响已复制值。

这里要区分三件事。

MemoryContext 管内存批量释放。

ResourceOwner 管 pin、refcount、文件句柄、锁等外部资源。

全局或共享状态必须有独立的可见性、锁或代际边界。

把这些混成一句“事务结束释放”，会丢掉定位问题所需的时间顺序。

### 6.6. reset 操作

创建或进入者：管理员函数或 GUC reload 不应改变 capacity。

正常清理者：清零 head/tail/dropped。

异常路径依赖：reset 需要排它锁和权限检查。

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
| 启动期大小 | RequestAddinShmemSpace | main shmem 大小必须启动前确定。 | 运行时可以随便扩 ring。 |
| 锁保护 | named LWLock | header 和 slot 更新需要并发保护。 | atomic 一个字段就保护整个事件。 |
| 固定 payload | 值拷贝 | 共享区不能保存 backend-local 指针。 | sourceText 指针可以跨 backend 保存。 |
| 丢弃语义 | dropped_count / overwrite flag | 缺失事件必须可解释。 | 没有事件等于没有慢 SQL。 |
| 读者隔离 | copy then format | SRF 不应长时间持锁。 | 读取时直接在共享区 form tuple。 |
| 版本边界 | magic / version | 扩展升级要识别布局。 | 结构体布局永远兼容。 |

读源码时可以按这个顺序检查：

```text
先找写入点
  -> 再找读取点
  -> 再找清理点
  -> 再看 ERROR / abort / backend exit 是否能兜底
  -> 最后把用户可见输出映射回这一条状态链
```

1. `启动期大小` 这一层主要依赖 RequestAddinShmemSpace。

它保证的是：main shmem 大小必须启动前确定。

不要把它误读成：运行时可以随便扩 ring。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

2. `锁保护` 这一层主要依赖 named LWLock。

它保证的是：header 和 slot 更新需要并发保护。

不要把它误读成：atomic 一个字段就保护整个事件。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

3. `固定 payload` 这一层主要依赖 值拷贝。

它保证的是：共享区不能保存 backend-local 指针。

不要把它误读成：sourceText 指针可以跨 backend 保存。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

4. `丢弃语义` 这一层主要依赖 dropped_count / overwrite flag。

它保证的是：缺失事件必须可解释。

不要把它误读成：没有事件等于没有慢 SQL。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

5. `读者隔离` 这一层主要依赖 copy then format。

它保证的是：SRF 不应长时间持锁。

不要把它误读成：读取时直接在共享区 form tuple。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

6. `版本边界` 这一层主要依赖 magic / version。

它保证的是：扩展升级要识别布局。

不要把它误读成：结构体布局永远兼容。

如果一次诊断只能看到这一层的投影，就必须保留不确定性。

## 8. 错误路径 / 异常路径 / fallback

### 8.1. 未 shared_preload

可见现象：函数报扩展未加载或 shmem 未初始化。

源码上应先回到：`ShmemInitStruct()`、扩展 `_PG_init()`。

正确处理方式是：要求 shared_preload 或退化为 backend-local。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.2. ring 过小

可见现象：dropped_count 快速上升。

源码上应先回到：ring header。

正确处理方式是：调大启动期 GUC 或提高采样阈值。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.3. 锁竞争

可见现象：profiler LWLock wait 出现。

源码上应先回到：`LWLockAcquire()`。

正确处理方式是：缩短临界区、分区 ring、try-lock 丢弃。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.4. 事件过大

可见现象：source text 或 plan string 无法放入 payload。

源码上应先回到：event payload 设计。

正确处理方式是：只存 queryId、plan_node_id、fixed fields，大文本另行处理。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

### 8.5. reader 阻塞 writer

可见现象：查询 view 本身影响业务 SQL。

源码上应先回到：SRF read path。

正确处理方式是：复制快照后释放锁，再 form tuple。

这类路径通常不是“继续收集更多指标”就能解决。

要先判断观测值是在失败前、失败中还是失败后写入。

如果写入点本身在异常路径里，就要检查 cleanup 是否仍然执行。

## 9. 成本、资源与跨模块传播

成本分析要围绕同一个主问题。

不要把所有可能成本都列一遍。

### 9.1. 共享锁

主要扩张因子：写入事件频率和 backend 数。

放大方式：所有 writer 争用同一 LWLock。

控制办法：partitioned ring 或 try-lock drop。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.2. cache line

主要扩张因子：write_pos 和 dropped_count 热更新。

放大方式：多 backend false sharing。

控制办法：按分区 header 或 padding。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.3. 事件大小

主要扩张因子：payload 字段数和字符串长度。

放大方式：ring cache footprint 增大。

控制办法：只保存定长值。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.4. 读取成本

主要扩张因子：capacity 和 view 查询频率。

放大方式：复制和 form tuple 成本。

控制办法：限制返回窗口和 top N。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.5. 启动内存

主要扩张因子：ring capacity GUC。

放大方式：main shared memory 固定占用。

控制办法：保守默认和最大值。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

### 9.6. 丢弃策略

主要扩张因子：峰值事件速率。

放大方式：覆盖会丢历史，丢弃会丢新热点。

控制办法：把策略和计数暴露给用户。

诊断时不要只看绝对值。

要看它是否随 rows、loops、backend 数、buffer miss、计划节点深度或采样率线性放大。

如果它只在某个节点或某类等待里集中出现，说明问题边界比 SQL 文本更窄。

## 10. 观测与诊断入口

观测入口要分清当前值、累计值、单次执行剖面和外部采样。

同一个慢 SQL 结论最好至少由两类信号互相校验。

### 10.1. pg_get_shmem_allocations

能看到：扩展申请的 shared memory 名称和大小。

看不到：ring 内事件内容。

源码入口：`shmem.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.2. pg_stat_activity wait_event

能看到：profiler LWLock 是否造成等待。

看不到：事件丢弃原因。

源码入口：`lwlock.c`、`wait_event.c`。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.3. profiler info view

能看到：capacity、write_pos、dropped_count、reset_time。

看不到：单个事件源码栈。

源码入口：扩展 SRF。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.4. profiler events view

能看到：固定事件 payload。

看不到：未采样查询。

源码入口：扩展 SRF。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.5. gdb shared header

能看到：header 字段实时值。

看不到：业务语义。

源码入口：shmem 地址和结构体。

使用时先问它是当前状态、累计状态，还是本次执行状态。

再问它是否会因为采样、刷新延迟、并行 worker 汇总或输出格式化而偏移。

最后把结论回扣到本节主流程中的某个写入点。

### 10.6. pg_stat_statements 源码

能看到：成熟 shared hash + SRF 模式。

看不到：ring buffer 具体策略。

源码入口：`pg_stat_statements.c`。

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

### 12.1. 确认 shmem API

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
cd /home/highgo/postgres
rg "RequestAddinShmemSpace|ShmemInitStruct|RequestNamedLWLockTranche" src contrib/pg_stat_statements
```

预期现象：能看到扩展和核心共享内存的标准入口。

回到源码时检查：`shmem.c`、`pg_stat_statements.c`。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.2. ring 满策略演练

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
设置很小 capacity。
运行大量短查询。
查询 profiler info view 的 dropped_count。
```

预期现象：事件缺失应由 dropped_count 解释，而不是静默消失。

回到源码时检查：ring header 更新逻辑。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.3. 读写锁竞争观察

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
高并发运行 SQL。
频繁查询 profiler events view。
采样 pg_stat_activity wait_event。
```

预期现象：若 profiler LWLock 成为热点，应降低读取频率或分区。

回到源码时检查：`LWLockAcquire()` wait event。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

### 12.4. backend exit 后读取

实验目标：把可见现象压回源码状态边界。

建议准备：

```text
运行查询后断开 session。
在另一个 session 查询 profiler view。
```

预期现象：固定值事件仍可读，backend-local 指针相关字段不应存在。

回到源码时检查：event payload 设计。

实验完成后不要只记录数值。

还要记录计划节点、GUC、并发状态、缓存冷热和是否开启 timing。

这些条件决定同一个观测值能否迁移到另一个现场。

## 13. 讨论题

1. 为什么 profiler ring buffer 必须有固定上界？

2. 覆盖旧事件和丢弃新事件分别适合什么现场？

3. 为什么共享区不能保存 `sourceText` 指针？

4. 读取 view 时为什么要先复制再格式化？

5. 什么时候 shared memory 不值得引入，日志或 backend-local 就够了？

## 14. 本节小结

本节只沉淀一个模型：

```text
共享 ring buffer 解决跨 backend 可读性，但引入固定资源和锁竞争。
shared memory 大小必须启动期申请。
事件 payload 应定长、值拷贝、可丢弃。
ring 满和锁竞争都要有显式丢弃计数。
读者短锁复制快照，不能持锁输出。
```

下一节给 profiler 加上 GUC、采样率和线上保护。
