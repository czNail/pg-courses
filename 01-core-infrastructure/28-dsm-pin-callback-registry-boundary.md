# PostgreSQL DSM pin、detach callback 与长期共享状态边界

## 课程定位

前置知识：已经理解 DSM control segment、`dsm_create()` / `dsm_attach()` / `dsm_detach()` 的 refcount 生命周期，知道 `ResourceOwner` 会在 ERROR / abort 时释放 DSM mapping。

本节唯一主问题：

```text
为什么 DSM 要区分 mapping pin 和 segment pin，dsm_pin_mapping()、
dsm_pin_segment()、on_dsm_detach() 以及 postmaster cleanup 如何避免
短事务资源释放、后台进程退出和长期共享对象互相踩边界？
```

核心矛盾：有些 DSM 只服务一次并行操作，应该随事务或执行上下文快速释放；另一些 DSM 承载 per-session 状态、named DSM、DSA、dshash 或 registry，需要活过当前事务甚至活到 postmaster shutdown。PostgreSQL 必须同时避免两类错误：

```text
短期资源被 pin 过久:
  内存、OS object、临时文件和共享状态泄漏。

长期共享状态没有 pin 够:
  创建者事务结束或 backend 退出后，其它 backend 手里的 handle 指向已销毁 segment。
```

学完后应能判断：

```text
什么时候只需要 dsm_pin_mapping()；
什么时候还必须 dsm_pin_segment()；
为什么 on_dsm_detach() 在 unmap 前执行；
为什么 detach callback 不能随意 ERROR；
为什么 named DSM/DSA 要 pin segment，也要 pin mapping；
为什么 DSA 和 SharedFileSet 把自己的 cleanup 挂到 containing DSM detach 上。
```

本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `2c4bd2bf5700`。

## 1. 本节在总主线中的位置

上一节讲的是普通 DSM 生命周期：

```text
dsm_create():
  创建 segment 和 backend-local descriptor，默认挂 CurrentResourceOwner。

dsm_attach():
  通过 handle 找到 active control slot，增加 refcount。

dsm_detach():
  执行 callback，unmap 当前 backend，降低 refcount，最后引用负责 destroy。
```

这套默认语义适合“一次操作用完就释放”的 DSM。例如 parallel query 的 DSM 通常随 `ParallelContext` 结束而释放。

本节讨论默认语义不够用的场景：

```text
当前事务结束了，但本 backend 还要继续使用这块映射；
当前 backend 退出了，但 segment 还要让未来 backend attach；
DSM 内部放了 DSA、shm_mq、SharedFileSet 等上层对象，它们必须在 DSM unmap 前清理。
```

所以 DSM 增加了三类边界工具：

```text
dsm_pin_mapping():
  保护当前 backend 的 mapping 生命周期。

dsm_pin_segment():
  保护全局 segment 生命周期。

on_dsm_detach():
  保护“依赖 DSM 内容的上层对象”在 unmap 前完成 cleanup。
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
mapping pin 把当前 backend 的 dsm_segment 从 ResourceOwner 中摘掉，使映射活到显式 detach 或 backend exit；
segment pin 在 control segment 中增加一个全局引用，使 segment 即使没有普通 backend mapping 也不会销毁；
detach callback 挂在 backend-local dsm_segment 上，在 unmap 和 refcount-- 之前执行，让 DSM 内的上层对象有最后一次安全访问共享内存的机会。
```

本节 tension 是：

```text
ResourceOwner 默认应该积极回收短生命周期 DSM mapping
  vs
有些共享状态必须跨事务、跨 backend、跨后续 attach 长期存在
```

PostgreSQL 没有用一个 “keep forever” 开关混掉所有语义，而是拆成两个独立问题：

```text
当前进程是否还保留映射？
  -> dsm_pin_mapping()

全局 segment 是否还保留，即使无人映射？
  -> dsm_pin_segment()
```

这两个 pin 可以组合，但不能互相替代。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/storage/dsm.h` | pin、unpin、mapping 查找和 detach callback 的公开接口。 |
| 2 | `src/backend/storage/ipc/dsm.c` | `dsm_pin_mapping()`、`dsm_pin_segment()`、`dsm_unpin_segment()`、`on_dsm_detach()`、`reset_on_dsm_detach()` 的核心实现。 |
| 3 | `src/backend/storage/ipc/dsm_impl.c` | Windows 下 segment pin 需要把 handle duplicate 到 postmaster；其它实现通常无需额外动作。 |
| 4 | `src/backend/access/common/session.c` | per-session DSM：mapping pin 和 DSA mapping pin 如何让 session 状态活过当前事务。 |
| 5 | `src/backend/storage/ipc/dsm_registry.c` | named DSM / DSA / dshash：为什么必须同时 pin segment 和 pin mapping。 |
| 6 | `src/backend/utils/mmgr/dsa.c` | DSA 如何 pin backing DSM segment，并用 detach callback 释放 area refcount、unpin segment。 |
| 7 | `src/backend/storage/ipc/shm_mq.c` | shm_mq 如何用 `on_dsm_detach()` 防止 DSM 先消失导致对端永久等待。 |
| 8 | `src/backend/storage/file/sharedfileset.c` | SharedFileSet 如何把临时文件生命周期绑到最后一个 DSM detach。 |
| 9 | `src/backend/storage/ipc/ipc.c` | `on_exit_reset()` 为什么要调用 `reset_on_dsm_detach()`，避免 fork 后继承错误 cleanup。 |

推荐阅读顺序：

```text
先读 dsm.c 的 pin / callback API
  -> 再读 dsm_registry.c 的 named DSM 主流程
  -> 再读 dsa.c 的 segment pin 和 on_dsm_detach 组合
  -> 最后看 shm_mq / SharedFileSet 作为 callback 案例
```

不要把 `dsm_pin_mapping()` 和 `dsm_pin_segment()` 当成同一个资源的两种写法。它们保护的是不同生命周期边界。

## 4. 关键数据结构与状态

### 4.1 `dsm_segment.resowner`：mapping owner

`dsm_segment` 是 backend-local descriptor。它的 `resowner` 字段只说明当前 backend 的 mapping 是否由 `ResourceOwner` 管：

```text
resowner != NULL:
  mapping 属于某个 ResourceOwner；
  owner release 时 ResOwnerReleaseDSM() 调 dsm_detach()。

resowner == NULL:
  mapping 不再属于 ResourceOwner；
  需要显式 dsm_detach()，或等 dsm_backend_shutdown() 在 backend exit 时清理。
```

`dsm_pin_mapping()` 只做一件事：

```text
如果 seg->resowner != NULL:
  ResourceOwnerForgetDSM(seg->resowner, seg)
  seg->resowner = NULL
```

它不会改变 control segment 中的 `refcnt`，也不会保证 segment 在所有 backend 都 detach 后继续存在。

### 4.2 `dsm_control_item.pinned` 与 pin 引用

`dsm_pin_segment()` 修改的是 shared control state：

```text
control item:
  pinned = true
  refcnt++
  impl_private_pm_handle = implementation-specific pin handle
```

它的语义是：

```text
即使当前没有普通 backend mapping，这个 segment 也保留到 postmaster shutdown，
或者直到某个代码显式 dsm_unpin_segment(handle)。
```

因此如果创建时 `refcnt = 2`，随后 `dsm_pin_segment()`，状态变成：

```text
refcnt = 3
pinned = true

含义：
  1 个普通 mapping
  1 个 segment pin
  1 个 moribund sentinel 基准
```

如果创建者随后 `dsm_detach()`：

```text
refcnt: 3 -> 2
pinned: true
segment 仍 active，可被未来 backend dsm_attach(handle)
```

如果 `dsm_unpin_segment(handle)` 且没有普通 mapping：

```text
refcnt: 2 -> 1
pinned: false
触发 destroy
destroy 成功后 refcnt = 0
```

### 4.3 `on_detach` callback 链

`on_dsm_detach()` 把 callback 存到 backend-local `dsm_segment.on_detach` 链表。它不进入 shared control segment，所以每个 attach 的 backend 都需要自己注册需要的 callback。

`dsm_detach()` 的顺序是：

```text
HOLD_INTERRUPTS()
while on_detach not empty:
  pop callback
  pfree callback node
  function(seg, arg)
RESUME_INTERRUPTS()

unmap segment
refcnt--
free dsm_segment descriptor
```

两个细节很重要：

```text
callback 在 unmap 前执行:
  因为 callback 通常需要读取 DSM 内的共享结构。

callback 调用前先从链表 pop:
  防止 callback 抛错后再次进入 dsm_detach() 形成无限递归。
```

这就是为什么 callback 适合做“最后一次访问 DSM 内容的 cleanup”，而不是普通业务逻辑。

## 5. 主流程源码 walkthrough

### 5.1 mapping pin：per-session DSM 活过当前事务

`src/backend/access/common/session.c` 是最直接的 mapping pin 案例。

`GetSessionDsmHandle()` 创建 per-session DSM：

```text
GetSessionDsmHandle()
  -> 如果 CurrentSession->segment 已存在，返回 handle
  -> 切到 TopMemoryContext
  -> dsm_create(size, DSM_CREATE_NULL_IF_MAXSEGMENTS)
  -> shm_toc_create()
  -> dsa_create_in_place(..., seg)
  -> SharedRecordTypmodRegistryInit(..., seg, dsa)
  -> dsm_pin_mapping(seg)
  -> dsa_pin_mapping(dsa)
  -> CurrentSession->segment = seg
  -> CurrentSession->area = dsa
```

这里为什么只做 `dsm_pin_mapping(seg)`，没有做 `dsm_pin_segment(seg)`？

```text
per-session DSM 的生命周期属于创建它的 backend session。
只要当前 backend 保持 mapping，这个普通 mapping 自身就让 refcnt > 1。
它不需要在所有 backend detach 后继续存在。
```

如果不 pin mapping，会发生什么？

```text
dsm_create() 默认把 mapping 挂到 CurrentResourceOwner；
当前事务或短生命周期 owner 结束时 ResourceOwnerReleaseDSM() 调 dsm_detach()；
CurrentSession->segment 将指向已经释放的 backend-local descriptor；
后续 session 状态访问会踩悬空状态。
```

源码注释也强调：只有初始化成功到足够安全后才 pin。初始化中途失败时，已经注册的 detach callback 会随 ResourceOwner cleanup 执行，避免留下半初始化的 `CurrentSession` 状态。

### 5.2 segment pin + mapping pin：named DSM 活过当前 backend

`GetNamedDSMSegment()` 是 segment pin 和 mapping pin 同时出现的典型路径：

```text
GetNamedDSMSegment(name, size, init_callback, found, arg)
  -> init_dsm_registry()
  -> dshash_find_or_insert(registry, name, found)
  -> 如果 registry entry 没有 handle:
       seg = dsm_create(size, 0)
       init_callback(dsm_segment_address(seg), arg)
       dsm_pin_segment(seg)
       dsm_pin_mapping(seg)
       state->handle = dsm_segment_handle(seg)
     否则:
       seg = dsm_find_mapping(state->handle)
       如果当前 backend 尚未映射:
          seg = dsm_attach(state->handle)
          dsm_pin_mapping(seg)
  -> 返回 dsm_segment_address(seg)
```

这里两个 pin 都需要：

```text
dsm_pin_segment(seg):
  创建者 backend 退出或 detach 后，named segment 仍能被未来 backend attach。

dsm_pin_mapping(seg):
  当前 backend 可以在后续调用中直接复用 mapping；
  不会被当前 ResourceOwner 在事务结束时释放。
```

如果只做 `dsm_pin_mapping()`：

```text
创建者 backend 一直活着时没事；
一旦创建者退出，最后一个普通 mapping detach 后 segment 被 destroy；
registry 里还保存着 handle，但未来 backend attach 会失败。
```

如果只做 `dsm_pin_segment()`：

```text
segment 全局还在；
但当前 backend 的 mapping 会在 ResourceOwner release 时 dsm_detach()；
调用方手里的返回地址可能失效，后续访问变成悬空本地指针。
```

这正是本节的核心边界：segment 存活和 mapping 存活是两件事。

### 5.3 DSA：segment pin 让共享 heap 控制自己的 backing segments

`dsa_create_ext()` 创建 DSA area：

```text
dsa_create_ext()
  -> segment = dsm_create(init_segment_size, 0)
  -> dsm_pin_segment(segment)
  -> area = create_internal(..., dsm_segment_handle(segment), segment, ...)
  -> on_dsm_detach(segment, dsa_on_dsm_detach_release_in_place, address)
```

源码注释直接说明原因：

```text
All segments backing this area are pinned, so that DSA can explicitly
control their lifetime.
```

如果不 pin backing segment，会出现这种问题：

```text
某个新 DSM segment 刚加入 DSA area；
当前只有创建者 backend 映射了它；
创建者 backend 事务结束或 ERROR detach；
DSM refcount 归零并 destroy；
但 DSA control state 里仍然认为这个 segment 是 area 的一部分。
```

DSA 的释放不是靠每个 DSM segment 自己的最后 mapping 自动决定，而是靠 DSA area 的 refcount 和 `dsa_release_in_place()`：

```text
dsa_release_in_place(place)
  -> control->refcnt--
  -> 如果 DSA area refcnt 变成 0:
       遍历 control->segment_handles[]
       dsm_unpin_segment(handle)
```

当 DSA 决定某个 segment 整体空了时，也会主动释放：

```text
destroy_superblock()
  -> segment 全空且不是 control segment
  -> dsm_unpin_segment(dsm_segment_handle(segment))
  -> dsm_detach(segment)
  -> segment_handles[index] = DSM_HANDLE_INVALID
  -> freed_segment_counter++
```

这里体现了 segment pin 的系统意义：

```text
DSM 默认根据 mapping refcount 自动销毁；
DSA 需要更高层的 heap policy 决定何时释放 backing segment；
所以 DSA 必须先 pin segment，把释放权拿回自己手里。
```

### 5.4 detach callback：DSM 内对象的最后清理窗口

`shm_mq_attach()` 展示了 callback 的最小用途：

```text
shm_mq_attach(mq, seg, handle)
  -> 初始化 backend-local shm_mq_handle
  -> 如果 seg != NULL:
       on_dsm_detach(seg, shm_mq_detach_callback, mq)
```

`shm_mq_detach_callback()` 做的是：

```text
shm_mq_detach_internal(mq)
  -> mq_detached = true
  -> SetLatch(counterparty)
```

为什么这个 callback 要挂到 DSM detach？

```text
如果当前 backend 因 ERROR 或 ResourceOwner release 直接 detach DSM，
但没有通知 shm_mq 对端，
对端可能继续等待这个队列被填充或 drain。
```

callback 在 DSM unmap 前执行，因此还能访问 `mq` 这个 DSM 内对象，并设置 detach 状态、唤醒对端。

`SharedFileSet` 是另一个例子：

```text
SharedFileSetInit(fileset, seg)
  -> on_dsm_detach(seg, SharedFileSetOnDetach, fileset)

SharedFileSetOnDetach()
  -> fileset->refcnt--
  -> 如果最后一个 detach，删除临时文件目录
```

这个 callback 不是为了释放 DSM 内存，而是把“共享临时文件目录”的生命周期绑到“最后一个使用该 DSM 的 backend detach”上。

## 6. 生命周期 / ownership / cleanup

### 谁创建

| 对象 | 创建者 | 典型文件 |
| --- | --- | --- |
| session DSM | 需要 session shared state 的 backend | `session.c` |
| named DSM | 第一个调用 `GetNamedDSMSegment()` 的 backend | `dsm_registry.c` |
| DSA backing DSM | DSA area 创建者或扩容者 | `dsa.c` |
| shm_mq queue | 放在已有 DSM 中，由调用者创建 | `shm_mq.c` / `parallel.c` |
| SharedFileSet | 放在已有 DSM 中，由上层执行节点创建 | `sharedfileset.c` |

### 谁持有

要区分三层 ownership：

| 层次 | 状态 | 保护什么 |
| --- | --- | --- |
| mapping ownership | `dsm_segment.resowner` | 当前 backend 的 mapping 是否随 ResourceOwner 释放。 |
| segment ownership | `dsm_control_item.refcnt` / `pinned` | 全局 DSM segment 是否继续存在。 |
| 上层对象 ownership | DSA refcount、SharedFileSet refcount、shm_mq sender/receiver | DSM 内对象的业务生命周期。 |

### 谁释放

普通短生命周期 DSM：

```text
ResourceOwner release 或显式 dsm_detach()
  -> mapping detach
  -> 最后 mapping destroy segment
```

mapping-pinned DSM：

```text
dsm_pin_mapping()
  -> 不随 ResourceOwner release
  -> 显式 dsm_detach() 或 dsm_backend_shutdown() 时清理
```

segment-pinned DSM：

```text
dsm_pin_segment()
  -> 没有普通 mapping 也不 destroy
  -> dsm_unpin_segment(handle) 或 postmaster shutdown cleanup 时释放
```

callback 关联对象：

```text
dsm_detach()
  -> callback 先运行
  -> 上层对象释放自己的 refcount、通知对端、删除临时文件等
  -> DSM 再 unmap / refcnt--
```

### ERROR / abort 时谁兜底

如果 mapping 没 pin：

```text
ResourceOwnerRelease()
  -> ResOwnerReleaseDSM()
  -> dsm_detach()
  -> detach callback 运行
```

如果 mapping 已 pin：

```text
ResourceOwner 不再知道这个 mapping；
ERROR 不会释放它；
必须靠显式 dsm_detach() 或 backend exit 的 dsm_backend_shutdown()。
```

这不是漏洞，而是 API 合约：调用 `dsm_pin_mapping()` 的代码承诺这块 mapping 的生命周期不再是当前 ResourceOwner。

## 7. 正确性机制层次

| 机制 | 保证 | 不保证 |
| --- | --- | --- |
| `dsm_pin_mapping()` | 当前 backend 的 mapping 不被当前 ResourceOwner 自动 detach。 | segment 在所有 backend detach 后仍存在。 |
| `dsm_pin_segment()` | control segment 多一个 pin 引用；无普通 mapping 时 segment 仍存在。 | 当前 backend 的 mapped address 仍有效。 |
| `dsm_unpin_segment()` | 释放 segment pin；如果这是最后引用，触发 destroy。 | 自动清理所有 backend-local 指针。 |
| `on_dsm_detach()` | unmap 前执行上层 cleanup。 | callback 一定成功；callback 后对象仍可继续使用。 |
| `cancel_on_dsm_detach()` | 显式清理上层对象后取消自动 callback。 | 取消 DSM refcount 递减。 |
| `reset_on_dsm_detach()` | fork 后清掉继承来的 detach callback 和 control_slot，避免子进程执行 postmaster/父进程 cleanup。 | 普通 backend 业务层面的资源释放。 |
| postmaster cleanup | shutdown / crash 后尽量销毁遗留 external DSM。 | 正常运行期的业务失效协议。 |

本节不涉及 WAL 或 MVCC。pin 只表达共享内存资源存活，不表达 tuple 可见性、事务提交，也不保证 DSM 内对象没有语义过期。

## 8. 错误路径 / 异常路径 / fallback

### 8.1 初始化中途失败：先别急着 pin

`GetSessionDsmHandle()` 有一个重要顺序：

```text
dsm_create()
-> 创建 shm_toc
-> 创建 in-place DSA
-> 初始化 typmod registry
-> 最后 dsm_pin_mapping(seg)
```

源码注释说明：如果没有走到最后，已经安装的 cleanup callback 会在 DSM 被 ResourceOwner detach 时运行，避免留下破损的 `CurrentSession`。

规律：

```text
只有对象达到“后续可以安全长期使用”的状态后，才 pin mapping。
```

### 8.2 named DSM 初始化失败

`GetNamedDSMSegment()` 在 holding registry entry lock 时执行：

```text
seg = dsm_create(size, 0)
init_callback(...)
dsm_pin_segment(seg)
dsm_pin_mapping(seg)
state->handle = dsm_segment_handle(seg)
```

如果 `init_callback` ERROR，`state->handle` 还没有发布，segment 仍由 ResourceOwner 管理，ERROR cleanup 会 detach/destroy。这样其它 backend 不会在 registry 中看到一个未初始化完成的 handle。

### 8.3 `dsm_pin_segment()` 重复调用

`dsm_pin_segment()` 检查：

```text
if (dsm_control->item[seg->control_slot].pinned)
  elog(ERROR, "cannot pin a segment that is already pinned");
```

segment pin 不是引用计数式的多 owner pin。它是一个单 bit 加一个额外 refcount。调用者需要清楚谁负责 `dsm_unpin_segment()`。

### 8.4 `dsm_unpin_segment()` 不要求当前 backend attach

`dsm_unpin_segment()` 参数是 `dsm_handle`，不是 `dsm_segment *`。源码注释说明这是为了支持“一个 segment 的 handle 存在另一个 segment 内”这类场景。

这对 DSA 很重要：释放 DSA area 时，它遍历 `segment_handles[]`，即使当前 backend 不一定映射所有 backing segments，也可以逐个 unpin。

### 8.5 detach callback 抛错

`dsm_detach()` 在调用 callback 前先 pop callback node，并用 `HOLD_INTERRUPTS()` 包住 callback 链。这样做是为了降低 cleanup 被中断或递归的风险。

但 callback 仍然不应该依赖会失败的高层逻辑。SharedFileSet 的注释写得很直白：它在 ERROR cleanup paths 中运行，不能因为删除文件失败再抛 ERROR。

### 8.6 fork 后继承 callback 的问题

`on_exit_reset()` 会调用 `reset_on_dsm_detach()`：

```text
before_shmem_exit_index = 0
on_shmem_exit_index = 0
on_proc_exit_index = 0
reset_on_dsm_detach()
```

`reset_on_dsm_detach()` 会清空每个 inherited DSM descriptor 的 callback，并把 `control_slot = INVALID_CONTROL_SLOT`，避免 fork 出来的子进程误以为自己应该执行父进程或 postmaster 的 DSM cleanup。

这属于“继承了地址空间，不等于继承 ownership”的典型边界。

## 9. 成本、资源与跨模块传播

### pin 的成本

`dsm_pin_mapping()` 是 backend-local 操作：

```text
ResourceOwnerForgetDSM()
seg->resowner = NULL
```

它不需要拿 `DynamicSharedMemoryControlLock`。

`dsm_pin_segment()` / `dsm_unpin_segment()` 要修改 shared control item，需要持有 `DynamicSharedMemoryControlLock`。它们不应出现在高频数据路径里。

### 长期 pin 的资源代价

segment pin 会让 DSM segment 即使无人映射也继续占用资源：

```text
external DSM:
  OS shared memory object / mmap file / Windows mapping handle。

main region DSM:
  min_dynamic_shared_memory 预留区域里的页。

control segment:
  slot 长期保持 refcnt > 1，不会变成可复用的 0。
```

因此 named DSM / DSA / dshash 适合少量长期共享基础设施，不适合为每个用户对象无限创建。

### callback 的传播边界

`on_dsm_detach()` 让 DSM 成为上层资源生命周期的汇合点：

```text
shm_mq:
  DSM detach -> 标记队列 detached -> 唤醒对端。

DSA:
  DSM detach -> DSA area refcnt-- -> 最后一个 release 时 unpin backing segments。

SharedFileSet:
  DSM detach -> fileset refcnt-- -> 最后一个 detach 删除临时文件。

record typmod registry:
  session DSM detach -> 清理共享 typmod registry 状态。
```

这很好用，也很危险：callback 太多会让 `dsm_detach()` 成为跨模块 cleanup 枢纽。课程阅读时要始终追问：

```text
callback 是否只做必须在 unmap 前完成的事？
callback 是否可以在 ERROR cleanup 中安全运行？
callback 是否有对应的 cancel_on_dsm_detach()？
```

## 10. 观测与诊断入口

### SQL

DSM pin 本身没有专门的系统视图。可以间接观察：

```sql
SELECT * FROM pg_get_dsm_registry_allocations();
```

这个函数遍历 DSM registry 中的 named DSM / DSA / dshash entry，可用于确认 registry 对象是否已经创建、name/type/handle 是否符合预期。

`min_dynamic_shared_memory` 预留区域可通过 `pg_shmem_allocations` 看到名字类似 `Preallocated DSM` 的 main shmem allocation，但看不到每个 pseudo DSM slot 的 pin 状态。

### 日志

postmaster shutdown 或 crash cleanup 时可能看到：

```text
cleaning up orphaned dynamic shared memory with ID ...
cleaning up dynamic shared memory control segment with ID ...
```

这说明 cleanup 正在按 control segment 记录清理仍存在的 DSM。它不能告诉你是哪个业务模块忘了 unpin，只能说明有 segment 活到了 cleanup 阶段。

### gdb

推荐断点：

```gdb
break dsm_pin_mapping
break dsm_pin_segment
break dsm_unpin_segment
break on_dsm_detach
break dsm_detach
break GetNamedDSMSegment
break dsa_release_in_place
```

观察 mapping pin：

```gdb
p seg->resowner
next
p seg->resowner
```

观察 segment pin：

```gdb
p dsm_control->item[seg->control_slot].refcnt
p dsm_control->item[seg->control_slot].pinned
next
p dsm_control->item[seg->control_slot].refcnt
p dsm_control->item[seg->control_slot].pinned
```

观察 detach callback：

```gdb
break dsm_detach
break shm_mq_detach_callback
break SharedFileSetOnDetach
break dsa_on_dsm_detach_release_in_place
```

关键是确认 callback 发生在 `mapped_address` 清空之前。

## 11. 常见误区

### 误区 1：`dsm_pin_mapping()` 会让 segment 永远存在

不会。它只让当前 backend 的 mapping 不被 ResourceOwner 自动 detach。当前 backend 退出时，`dsm_backend_shutdown()` 仍会 detach。

### 误区 2：`dsm_pin_segment()` 会让当前地址一直可用

不会。它只保护全局 segment。当前 backend 的 `mapped_address` 是否还有效，取决于 mapping 是否仍 attach。

### 误区 3：named DSM 只需要把 handle 记到 registry

不够。handle 只是发现入口。如果 segment 没有 pin，创建者 detach 后 handle 可能指向已销毁对象。

### 误区 4：detach callback 是析构函数，可以做任意 cleanup

不对。它运行在 DSM detach 路径，常常处于 ERROR cleanup。它应该短小、可重复推理、尽量不失败，并且只做必须在 DSM unmap 前完成的事。

### 误区 5：segment pin 是普通引用计数，可以多次 pin 多次 unpin

当前 DSM segment pin 是 `pinned` bit 加一个额外 refcount。重复 pin 会 ERROR。需要多 owner 语义时，上层模块要自己维护 refcount，并在 0 时调用一次 `dsm_unpin_segment()`。

### 误区 6：fork 继承了 DSM mapping，就继承了 cleanup 责任

不是。`on_exit_reset()` 会清掉 inherited exit callbacks 和 DSM detach callbacks，并让 inherited descriptor 不再降 control refcount。ownership 要由新进程的启动协议重新建立。

## 12. 课堂实验

### 实验 1：观察 per-session DSM 的 mapping pin

目标：看到 `GetSessionDsmHandle()` 如何在初始化完成后 pin mapping。

触发路径可以来自 parallel query，因为 parallel context 会调用 `GetSessionDsmHandle()`：

```sql
SET max_parallel_workers_per_gather = 2;
SET min_parallel_table_scan_size = 0;
SET parallel_setup_cost = 0;
SET parallel_tuple_cost = 0;

CREATE TABLE dsm_pin_demo AS
SELECT g AS id, md5(g::text) AS payload
FROM generate_series(1, 1000000) g;
ANALYZE dsm_pin_demo;

EXPLAIN (ANALYZE, VERBOSE)
SELECT count(*) FROM dsm_pin_demo;
```

gdb：

```gdb
break GetSessionDsmHandle
break dsm_pin_mapping
```

观察：

```gdb
p seg->resowner
next
p seg->resowner
```

问题：为什么这里没有 `dsm_pin_segment()`？

### 实验 2：跟踪 named DSM 创建

目标：理解 named DSM 为什么需要 segment pin + mapping pin。

如果环境中有使用 `GetNamedDSMSegment()` 的扩展或测试模块，可以在：

```gdb
break GetNamedDSMSegment
break dsm_pin_segment
break dsm_pin_mapping
```

观察创建分支：

```text
dsm_create()
-> init_callback()
-> dsm_pin_segment()
-> dsm_pin_mapping()
-> state->handle = ...
```

讨论：为什么发布 `state->handle` 必须在初始化和 pin 之后？

### 实验 3：观察 shm_mq 的 detach callback

目标：确认 DSM detach 会先通知 shm_mq 对端。

用 parallel query 或 `test_shm_mq` 模块触发 `shm_mq_attach()`，设置断点：

```gdb
break on_dsm_detach
break dsm_detach
break shm_mq_detach_callback
```

检查顺序：

```text
dsm_detach()
  -> shm_mq_detach_callback()
  -> dsm_impl_op(DSM_OP_DETACH)
  -> refcnt--
```

问题：如果 callback 在 unmap 之后执行，会发生什么？

## 13. 讨论题

1. 为什么 `dsm_pin_mapping()` 不增加 control segment 中的 `refcnt`？
2. 为什么 `dsm_pin_segment()` 不把当前 `dsm_segment.resowner` 置空？
3. named DSM 创建时，为什么顺序是 `init_callback` -> `dsm_pin_segment` -> `dsm_pin_mapping` -> 发布 handle？
4. DSA 为什么不能只依赖每个 backend 的普通 DSM mapping refcount 来管理 backing segments？
5. `on_dsm_detach()` callback 应该遵守哪些限制，才能安全运行在 ERROR cleanup 路径？
6. `reset_on_dsm_detach()` 为什么要把 `control_slot` 置为 `INVALID_CONTROL_SLOT`，而不只是清空 callback？

## 14. 本节小结

本节的核心结论是：

```text
DSM 生命周期有三层，不是一把 refcount 解决所有问题。
```

三层分别是：

```text
当前 backend mapping:
  dsm_segment.resowner / dsm_pin_mapping()

全局 segment 存活:
  dsm_control_item.refcnt / pinned / dsm_pin_segment()

DSM 内上层对象:
  on_dsm_detach() callback 保护 unmap 前 cleanup
```

最可迁移的系统规律：

```text
当一个资源同时有“本地映射状态”和“全局对象状态”时，
pin 必须按层次拆开；
当上层对象依赖底层内存仍可访问时，
cleanup 必须挂在 unmap 前的最后安全窗口。
```

下一节进入 `shm_toc`：拿到一块长期或短期 DSM 后，为什么 PostgreSQL 不能在里面保存普通指针，而要用 magic、key 和相对 offset 建立对象发现协议。
