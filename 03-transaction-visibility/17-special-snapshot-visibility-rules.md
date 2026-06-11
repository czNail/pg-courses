# PostgreSQL SnapshotAny / Dirty / Toast 等特殊可见性规则

## 课程定位

前置知识：已经理解普通 `SNAPSHOT_MVCC` 的 tuple 可见性判断，也理解当前事务自可见性为什么需要 command id 和特殊 routine。

本节唯一主问题：

```text
为什么系统维护、索引检查、toast 访问、锁冲突探测和 vacuum 判断不能全部使用普通 MVCC SELECT 的 visibility routine？
```

本节核心矛盾：

```text
普通 SELECT 需要一个稳定的用户可见世界；
但内核维护路径经常需要看见“用户不该看见但系统必须处理”的 tuple，例如未提交版本、toast chunk、所有物理 tuple、可能还不能 vacuum 的版本或逻辑解码历史版本。
```

学完本节后，你应该能独立判断：

- 为什么 `SnapshotData` 里有 `snapshot_type`，而不是所有路径都调用同一个 MVCC 判断。
- 为什么 `SnapshotAny` 可以看所有 tuple，但不能随便用于用户查询。
- 为什么 `SnapshotDirty` 会把冲突事务 XID 写回 snapshot 结构。
- 为什么 `SnapshotToast` 只服务 TOAST row 访问。
- 为什么 `SnapshotNonVacuumable` 回答的是 cleanup 问题，不是普通 SELECT 可见性问题。
- 为什么 `HeapTupleSatisfiesVisibility()` 是分发层，不是所有规则的统一语义。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

本节不展开 logical decoding 的 historic snapshot 全链路，也不展开 vacuum/pruning 的完整页级流程。

这里先讲特殊 visibility routine 为什么存在。

## 1. 本节在总主线中的位置

前几节一直围绕普通 MVCC SELECT。

但 PostgreSQL 内核不是只做普通 SELECT。

它还要：

```text
检查唯一索引冲突。
读取 TOAST chunk。
维护系统 catalog。
做 VACUUM 和 pruning。
追踪并发 UPDATE 的冲突事务。
逻辑解码历史 catalog。
检查某个物理 TID 是否存在。
```

这些路径的目标不同。

如果全部使用普通 `SNAPSHOT_MVCC`，会出现两类问题。

第一，系统看不到自己必须处理的物理状态。

例如索引检查可能需要知道一个 conflicting tuple 是否由未提交事务插入。

第二，系统得到的答案过于用户视角。

例如 VACUUM 需要判断 tuple 是否可能被任何事务需要，而不是当前 snapshot 是否能看见。

因此 PostgreSQL 把“可见性”拆成多个 routine。

它们都读 heap tuple header。

但它们回答的问题不同。

本节主线是：

```text
调用者先明确自己要问的问题
  -> 选择 SnapshotType
  -> HeapTupleSatisfiesVisibility() 分发到对应 routine
  -> routine 返回 bool 或通过 snapshot 输出额外冲突信息
  -> 调用者按维护语义继续处理
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
PostgreSQL 用 SnapshotType 把“用户可见性”“自可见性”“物理可见性”“dirty 冲突探测”“toast 特例”“历史 MVCC”“可 vacuum 性”分成不同问题；HeapTupleSatisfiesVisibility() 只负责按类型分发，真正语义由各 routine 服务自己的调用场景。
```

这里的 tension 是：

```text
统一接口能减少调用者复杂度；
但不同内核路径需要的可见性答案并不是同一个布尔问题。
```

普通 MVCC 问：

```text
这个 tuple 是否属于我的 snapshot？
```

Dirty snapshot 问：

```text
这个 tuple 是否存在，即使相关事务还没提交？
如果它受未完成事务影响，冲突 XID 是谁？
```

Any snapshot 问：

```text
这个物理 tuple 是否存在于 page 上？
```

Toast snapshot 问：

```text
这个 TOAST row 是否按 toast 访问规则有效？
```

NonVacuumable snapshot 问：

```text
这个 tuple 是否可能仍被某个事务需要？
```

这些问题不是同一个 abstraction。

所以不能只靠一个 `HeapTupleSatisfiesMVCC()`。

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/utils/snapshot.h` | 先读 `SnapshotType` 枚举和每种 snapshot 的语义注释。 |
| 2 | `src/backend/access/heap/heapam_visibility.c` | 主读 `HeapTupleSatisfiesVisibility()` 分发，以及 `Self`、`Any`、`Toast`、`Dirty`、`NonVacuumable` routine。 |
| 3 | `src/include/access/heapam.h` | 对照 visibility routine 的公开声明。 |
| 4 | `src/backend/access/heap/heapam.c` | 看 `heap_fetch()`、`heap_lock_tuple()`、冲突处理路径如何使用特殊 snapshot。 |
| 5 | `src/backend/access/heap/heapam_handler.c` | 看 table AM 回调中 `SnapshotDirty`、`HeapTupleSatisfiesVisibility()` 的调用点。 |
| 6 | `src/backend/access/heap/vacuumlazy.c` / `src/backend/access/heap/pruneheap.c` | 对照 vacuum/pruning 为什么需要 non-vacuumable 或 vacuum horizon 判断。 |
| 7 | `src/backend/access/heap/README.tuplock` | 理解 tuple lock 与 dirty visibility 的关系。 |

推荐阅读顺序：

```text
先读 snapshot.h 的 SnapshotType 注释
  -> 再读 HeapTupleSatisfiesVisibility() switch
  -> 每次只追一个调用场景
  -> 最后回到调用者，看它为什么不能用普通 MVCC
```

## 4. 一个 runtime 现象先定锚

先看一个锁冲突现象。

Session A：

```sql
DROP TABLE IF EXISTS dirty_demo;
CREATE TABLE dirty_demo(id int primary key, payload text);
INSERT INTO dirty_demo VALUES (1, 'a');

BEGIN;
UPDATE dirty_demo SET payload = 'a1' WHERE id = 1;
-- 不提交。
```

Session B：

```sql
UPDATE dirty_demo SET payload = 'b1' WHERE id = 1;
```

Session B 会等待或报冲突，取决于锁等待策略。

这里系统必须知道：

```text
目标 tuple 被另一个未完成事务影响。
冲突事务是谁。
如果等待后对方提交或回滚，应该如何重试。
```

普通 MVCC SELECT 只会回答：

```text
这个 tuple 对我的 snapshot 是否可见？
```

这不够。

锁冲突探测需要 dirty 视角。

类似地，VACUUM 也不能只问当前 snapshot。

TOAST 读取也不能把 TOAST chunk 当普通用户表行随意解释。

这就是特殊 snapshot 存在的 runtime 入口。

## 5. `SnapshotType` 是语义选择

`snapshot.h` 中定义多种类型：

```text
SNAPSHOT_MVCC
SNAPSHOT_SELF
SNAPSHOT_ANY
SNAPSHOT_TOAST
SNAPSHOT_DIRTY
SNAPSHOT_HISTORIC_MVCC
SNAPSHOT_NON_VACUUMABLE
```

这个枚举不是标签装饰。

它决定 `HeapTupleSatisfiesVisibility()` 调哪个 routine。

普通 MVCC：

```text
用户 SELECT 的稳定读视角。
```

Self：

```text
看当前事务当前命令效果。
```

Any：

```text
任何物理 tuple 都可见。
```

Toast：

```text
TOAST row 特殊规则。
```

Dirty：

```text
包含未完成事务效果，并把冲突 XID 通过 snapshot 输出。
```

Historic MVCC：

```text
逻辑解码等历史上下文。
```

NonVacuumable：

```text
判断 tuple 是否可能还不能被 vacuum。
```

同一个 `SnapshotData` 结构承载这些类型。

但字段含义并不总是一样。

例如 `SNAPSHOT_DIRTY` 会把 `snapshot->xmin` / `snapshot->xmax` 当作输出参数使用。

所以不要把 `SnapshotData` 的字段脱离 `snapshot_type` 单独解释。

## 6. `HeapTupleSatisfiesVisibility()` 分发层

`HeapTupleSatisfiesVisibility()` 的结构很直接。

它按 `snapshot->snapshot_type` switch：

```text
SNAPSHOT_MVCC:
  HeapTupleSatisfiesMVCC()

SNAPSHOT_SELF:
  HeapTupleSatisfiesSelf()

SNAPSHOT_ANY:
  HeapTupleSatisfiesAny()

SNAPSHOT_TOAST:
  HeapTupleSatisfiesToast()

SNAPSHOT_DIRTY:
  HeapTupleSatisfiesDirty()

SNAPSHOT_HISTORIC_MVCC:
  HeapTupleSatisfiesHistoricMVCC()

SNAPSHOT_NON_VACUUMABLE:
  HeapTupleSatisfiesNonVacuumable()
```

这层的价值是统一调用接口。

但它不统一语义。

调用者必须知道自己传入的 snapshot type 意味着什么。

如果把 `SnapshotAny` 传到用户 SELECT，会绕过 MVCC。

如果把普通 MVCC 用到冲突探测，就拿不到正在影响 tuple 的事务 ID。

如果把 Dirty 用在普通查询，就可能暴露未提交数据。

所以 snapshot type 是 correctness 边界。

不是性能选项。

## 7. `SnapshotAny`: 物理 tuple 视角

`HeapTupleSatisfiesAny()` 基本返回 true。

它表达的是：

```text
只要调用者已经拿到一个正常 heap tuple，就把它当成可见。
```

这适合需要直接检查物理状态的内部路径。

它不适合用户查询。

因为它会忽略：

```text
xmin 是否回滚。
xmax 是否提交。
snapshot running set。
current command boundary。
```

典型使用场景是系统内部已经通过其它方式保证安全，只需要按 TID 抓物理 tuple。

这里的关键不是：

```text
SnapshotAny 更强。
```

而是：

```text
SnapshotAny 问的是另一个问题。
```

它放弃用户可见性，换取物理检查能力。

## 8. `SnapshotDirty`: 冲突探测视角

`SNAPSHOT_DIRTY` 的语义更特殊。

它把未提交事务的效果也纳入考虑。

并且把影响 tuple 的 in-progress XID 写回 snapshot。

`snapshot.h` 注释说明：

```text
snapshot->xmin 可输出插入 tuple 的未完成事务。
snapshot->xmax 可输出删除或更新 tuple 的未完成事务。
snapshot->speculativeToken 可输出 speculative insertion token。
```

这让调用者能做下一步：

```text
等待冲突事务。
检查 speculative insertion。
重试可见性或锁定。
返回并发冲突。
```

普通 MVCC 返回 bool。

Dirty snapshot 返回 bool 加冲突信息。

这就是为什么它不能由普通 MVCC 替代。

唯一索引检查、tuple lock、UPDATE 冲突处理都会需要类似能力。

它们不是要向用户显示未提交数据。

它们是要知道：

```text
哪个未提交事务正在决定这个 tuple 的命运。
```

## 9. `SnapshotToast`: TOAST row 视角

TOAST 表存放大字段的外部 chunks。

读取主表 tuple 后，系统可能需要按 toast pointer 读取对应 chunks。

TOAST 可见性不是普通用户查询的完整 MVCC 问题。

它服务一个更窄的目标：

```text
给已经可见的主 tuple 取回它引用的大字段数据。
```

`HeapTupleSatisfiesToast()` 因此有特殊规则。

它不会展开成普通 SELECT 的所有语义。

如果 TOAST chunk 缺失或状态异常，通常说明底层数据一致性问题。

这类路径强调的是：

```text
主 tuple 可见后，toast chunk 应该按引用关系可取。
```

而不是：

```text
toast 表作为普通用户表参与 snapshot 查询。
```

## 10. `SnapshotNonVacuumable`: cleanup 视角

`SNAPSHOT_NON_VACUUMABLE` 回答的是 cleanup 问题。

它不是普通 SELECT。

它关心：

```text
这个 tuple 是否可能仍被某个事务看到，因而不能 vacuum。
```

`snapshot->vistest` 指向 `GlobalVisState`。

这个状态来自 relation-aware horizon。

返回 false 表示 tuple 肯定不再需要。

返回 true 表示它可能还不能被 cleanup。

这和 `HeapTupleSatisfiesMVCC()` 的返回方向完全不同。

普通 MVCC：

```text
true = 当前 snapshot 能看到。
```

NonVacuumable：

```text
true = 仍可能需要保留，不能随便 vacuum。
```

如果混淆这两个问题，会把不可见 tuple 提前删掉。

这会破坏旧 snapshot 或复制/解码路径。

## 11. Historic MVCC 的边界

`SNAPSHOT_HISTORIC_MVCC` 服务历史视角。

典型上下文是 logical decoding 需要按历史 catalog 内容解释 WAL。

它不是当前普通查询的 snapshot。

它对 `xip` / `subxip` 的含义可能不同。

例如 historic snapshot 中，某些数组可能表示 committed transactions。

这说明 `SnapshotData` 字段不能脱离 type 解读。

历史 MVCC 的完整链路涉及 reorder buffer、logical decoding 和 catalog timetravel。

本节不展开。

这里只记住：

```text
同一个 tuple header 可以被不同 snapshot type 按不同历史上下文解释。
```

## 12. 主流程源码 walkthrough

### 12.1 普通 SELECT

```text
heap scan
  -> HeapTupleSatisfiesVisibility(tuple, snapshot, buffer)
  -> snapshot_type = SNAPSHOT_MVCC
  -> HeapTupleSatisfiesMVCC()
  -> 返回当前 snapshot 是否可见
```

这是前两节的主线。

### 12.2 锁冲突或唯一性检查

```text
调用者构造 SnapshotDirty
  -> 读取目标 tuple
  -> HeapTupleSatisfiesDirty()
  -> 返回 tuple 是否按 dirty 规则存在
  -> snapshot->xmin/xmax/speculativeToken 输出冲突来源
  -> 调用者 wait / retry / report conflict
```

这里的关键是输出冲突 XID。

普通 MVCC 给不出。

### 12.3 物理 TID fetch

```text
heap_fetch(relation, SnapshotAny, tid, ...)
  -> 找到物理 tuple
  -> 按 Any 规则可见
```

调用者必须已经知道为什么可以绕过普通可见性。

### 12.4 TOAST 访问

```text
主 tuple 可见
  -> 需要外部 toast data
  -> 按 toast pointer 访问 toast relation
  -> 使用 toast visibility routine
```

它服务数据还原。

不是用户直接扫描 toast relation 的普通语义。

### 12.5 vacuum/pruning 判断

```text
得到 GlobalVisState 或 OldestXmin
  -> 判断 tuple 是否仍可能被任何观察者需要
  -> 决定能否 prune / vacuum
```

这个问题的答案不能从当前 query snapshot 推出。

## 13. 生命周期 / ownership / cleanup

特殊 snapshot 的生命周期差异很大。

普通 MVCC snapshot 通常要 active 或 registered。

Dirty snapshot 常作为栈上或局部结构使用。

它的部分字段是输出参数。

SnapshotAny / Toast / Self 往往不依赖完整 `xip` arrays。

NonVacuumable 依赖 `GlobalVisState`。

因此调用者必须保证：

```text
snapshot type 与 routine 匹配。
snapshot 中被该 routine 使用的字段已经初始化。
buffer 在调用期间有效并加锁。
返回值按该 snapshot type 的语义解释。
```

cleanup 责任也不在 `HeapTupleSatisfiesVisibility()`。

它只回答该 snapshot type 的问题。

等待事务、重试、清理 tuple、释放 buffer，都由调用者负责。

## 14. 正确性机制层次

第一层是问题定义。

调用者先决定自己要问用户可见性、物理存在性、冲突来源还是 cleanup 安全性。

第二层是 snapshot type。

它把问题编码进 `SnapshotData`。

第三层是 routine。

不同 routine 消费同一个 tuple header 的不同语义组合。

第四层是调用者后续动作。

Dirty snapshot 可能等待。

Vacuum snapshot 可能 prune。

Toast snapshot 可能读取 chunk。

Any snapshot 可能只做物理校验。

正确性来自：

```text
不要把一个问题的可见性答案用于另一个问题。
```

## 15. 错误路径 / 异常路径 / fallback

### 15.1 Dirty snapshot 输出 in-progress XID

如果 tuple 受未完成事务影响，Dirty routine 可能把 XID 写到 snapshot。

调用者要在释放 buffer 前复制必要状态。

然后等待事务或 speculative insertion。

等待后必须重试。

因为 tuple header 可能已经变化。

### 15.2 SnapshotAny 误用

如果用户查询误用 SnapshotAny，会读到回滚 tuple、未提交 tuple 或已删除 tuple。

这不是 fallback。

这是 correctness bug。

所以 SnapshotAny 只能在调用者明确需要物理视角时使用。

### 15.3 Toast chunk 缺失

TOAST 访问通常建立在主 tuple 已可见的基础上。

如果 toast chunk 不存在或不可用，往往说明数据损坏或不一致。

不能简单 fallback 到普通 MVCC。

### 15.4 NonVacuumable 近似

Global visibility 判断可能保守。

保守结果会推迟 cleanup。

它不能激进到提前删除仍可能需要的 tuple。

这和第 12 节的 cleanup horizon 主线一致。

## 16. 成本、资源与跨模块传播

特殊 routine 的成本因场景不同。

SnapshotAny 很便宜。

但它把 correctness 责任推给调用者。

Dirty snapshot 可能引出等待、重试和 conflict handling。

NonVacuumable 可能依赖 GlobalVisState。

Historic MVCC 依赖逻辑解码上下文。

Toast 路径受额外 relation fetch 和 chunk 读取成本影响。

跨模块传播包括：

```text
heapam_visibility.c:
  visibility routine 分发和核心判断。

snapshot.h:
  定义每种 snapshot 语义。

heapam.c:
  用特殊 snapshot 做 fetch、lock、update conflict 处理。

heapam_handler.c:
  table AM 回调向 executor 和索引路径暴露结果。

vacuumlazy.c / pruneheap.c:
  使用 cleanup-oriented visibility。

toast access:
  使用 toast-specific rule 还原大字段。
```

## 17. 观测与诊断入口

源码断点：

```text
break HeapTupleSatisfiesVisibility
break HeapTupleSatisfiesAny
break HeapTupleSatisfiesToast
break HeapTupleSatisfiesDirty
break HeapTupleSatisfiesSelf
break HeapTupleSatisfiesNonVacuumable
```

观察变量：

```text
snapshot->snapshot_type
snapshot->xmin
snapshot->xmax
snapshot->speculativeToken
snapshot->vistest
tuple->t_infomask
```

SQL 侧可以制造冲突：

```sql
BEGIN;
UPDATE dirty_demo SET payload = 'hold' WHERE id = 1;
```

另一个 session 执行：

```sql
UPDATE dirty_demo SET payload = 'wait' WHERE id = 1;
```

断点里观察等待前后 snapshot 输出字段。

诊断原则：

```text
先确认调用者问的是什么问题；
再确认 snapshot type 是否匹配；
最后才读具体 infomask 分支。
```

## 18. 常见误区

误区一：

```text
所有 tuple 可见性都应该等价于 MVCC SELECT。
```

不对。

内核维护路径有不同问题。

误区二：

```text
SnapshotAny 是更快的 MVCC。
```

不对。

它绕过用户可见性。

误区三：

```text
SnapshotDirty 表示可以向用户返回脏读。
```

不对。

它主要服务冲突探测和内部等待。

误区四：

```text
SnapshotData 字段在所有 snapshot type 下含义相同。
```

不对。

Dirty 和 Historic 等类型会重载部分字段含义。

误区五：

```text
NonVacuumable 的 true 就等于当前 snapshot 可见。
```

不对。

它回答的是能否被 cleanup。

## 19. 课堂实验

### 实验一：锁冲突触发 dirty 视角

Session A：

```sql
DROP TABLE IF EXISTS dirty_demo;
CREATE TABLE dirty_demo(id int primary key, payload text);
INSERT INTO dirty_demo VALUES (1, 'a');

BEGIN;
UPDATE dirty_demo SET payload = 'a1' WHERE id = 1;
```

Session B：

```sql
UPDATE dirty_demo SET payload = 'b1' WHERE id = 1;
```

源码断点：

```text
break HeapTupleSatisfiesDirty
break heap_lock_tuple
```

观察目标：

```text
系统需要知道哪个事务影响了 tuple。
普通 MVCC bool 不够。
```

### 实验二：观察 snapshot type 分发

断点：

```text
break HeapTupleSatisfiesVisibility
commands
  p snapshot->snapshot_type
  continue
end
```

执行不同 SQL：

```sql
SELECT * FROM dirty_demo;
SELECT * FROM dirty_demo WHERE id = 1 FOR UPDATE;
VACUUM dirty_demo;
```

观察不同调用路径。

### 实验三：源码阅读练习

阅读：

```text
src/include/utils/snapshot.h
src/backend/access/heap/heapam_visibility.c
```

把每个 `SnapshotType` 写成一句话问题。

不要写成函数清单。

例如：

```text
SNAPSHOT_DIRTY:
  哪个未完成事务正在影响这个 tuple？
```

## 20. 讨论题

1. 为什么特殊 snapshot 要编码成 `snapshot_type`，而不是每个 table AM 提供一组 callback？

2. SnapshotDirty 为什么需要把 XID 写回 snapshot？

3. SnapshotAny 的正确调用前提是什么？

4. Toast 可见性为什么不应当直接套普通用户 SELECT 语义？

5. NonVacuumable 和 MVCC visible 的返回值为什么不能混用？

6. 如果所有内部路径都只用 `HeapTupleSatisfiesMVCC()`，会丢失哪些信息？

## 21. 本节小结

本节的核心模型是：

```text
visibility routine 的名字相似，但回答的问题不同。
```

普通 MVCC 回答当前 snapshot 是否可见。

Self 回答当前事务包含当前命令时是否可见。

Any 回答物理 tuple 是否可见。

Dirty 回答未完成事务是否影响 tuple，并输出冲突信息。

Toast 回答 toast chunk 是否按内部规则可取。

NonVacuumable 回答 tuple 是否仍可能需要保留。

Historic MVCC 回答历史解码上下文中的可见性。

下一节会回到 UPDATE 主线：

```text
UPDATE 为什么生成新版本，t_ctid 如何把旧版本、当前版本和并发冲突连接起来。
```
