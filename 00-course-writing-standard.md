# PostgreSQL 内核课程编写标准

用途：后续新增、补写、重写 PostgreSQL 内核课程时，都按本文档做质量标准。

目标读者：有 PostgreSQL 使用经验，准备进入源码级内核研发、问题定位和性能诊断的工程师。

核心原则：

> 课程不是源码百科，也不是 API 清单。课程要帮助内核研发人员建立稳定 mental model，能沿真实源码主链路定位问题，并理解模块边界、状态变化、生命周期、错误路径和观测手段。

一节课最终只沉淀三类长期有用的内容：

| 类型 | 作用 |
| --- | --- |
| model | 稳定的系统抽象、边界和不变量 |
| runtime | 一条真实生命周期或调用主链路 |
| case | 一个可复现、可观测、能回到源码解释的问题 |

---

## 1. 合格课程的核心要求

每节课至少回答 6 个问题：

```text
这个机制解决什么内核问题？
核心状态放在哪里，谁能访问？
正常路径中状态如何变化？
谁创建、谁持有、谁释放，ERROR/abort 时怎么办？
它和哪些相邻模块形成正确性边界？
线上或实验环境中如何观察、验证、诊断？
```

如果一节课缺少以下任意三项，通常还没有达到内核研发培训标准：

- 具体问题，而不是泛泛介绍模块。
- 当前源码中真实存在的核心文件和入口函数。
- 至少一条能跟到源码的主流程 walkthrough。
- 关键数据结构的语义解释和状态边界。
- 生命周期 / ownership / cleanup。
- 至少一种异常路径或 fallback。
- 跨模块连接。
- 可执行的实验或源码练习。
- 讨论题和本节小结。

---

## 2. 推荐章节结构

新课默认使用下面结构。可以按主题微调，但不要把“主链路、生命周期、异常路径、观测入口”省掉。

```text
# PostgreSQL <主题>

## 课程定位
## 1. 本节在总主线中的位置
## 2. 核心文件分工
## 3. 一句话运行模型
## 4. 关键数据结构与状态
## 5. 主流程源码 walkthrough
## 6. 生命周期 / ownership / cleanup
## 7. 正确性边界：并发、锁、WAL、可见性或缓存
## 8. 错误路径 / 异常路径 / fallback
## 9. 与其他模块的连接
## 10. 观测与诊断入口
## 11. 常见误区
## 12. 课堂实验
## 13. 讨论题
## 14. 本节小结
```

有些主题不涉及并发、WAL 或观测指标，可以说明“不涉及的原因”。不要为了模板硬凑内容。

---

## 3. 课程定位

开头必须短而明确：

```text
前置知识是什么？
本节解决哪个具体问题？
学完后能独立判断什么边界？
```

推荐写法：

````markdown
## 课程定位

前面课程解释了 ...
本节继续回答：

```text
<一个具体问题>
```

学完后，应该能回答：

- ...
- ...

一句话模型：

> ...
````

好问题：

```text
已经不可见的旧 tuple version，什么时候可以真正移除？
```

弱问题：

```text
学习 VACUUM 相关源码。
```

---

## 4. 源码文件与阅读顺序

每节课必须列核心源码文件。表里写职责，不写“实现相关函数”这种空话。

````markdown
## 核心文件分工

| 文件 | 作用 |
| --- | --- |
| `src/backend/.../xxx.c` | ... |
| `src/include/.../xxx.h` | ... |

建议阅读顺序：

```text
xxx.h: 核心结构和状态位
  -> xxx.c:init/create path
  -> yyy.c:main runtime path
  -> zzz.c:cleanup/error/invalidation path
```
````

要求：

- 文件和函数必须尽量对应当前本地 PostgreSQL 源码真实存在的位置。
- 关键 `.h` 要列，因为内核研发经常先读状态结构和宏。
- 阅读顺序按 mental model 展开，不按文件名排序。
- 如果源码版本差异明显，要说明“本课基于哪个版本或分支”。

---

## 5. 数据结构与状态边界

不要整段复制结构体源码。只讲影响理解和诊断的字段组合。

```markdown
| 字段 / 状态 | 含义 | 边界 |
| --- | --- | --- |
| `xxx` | ... | ... |
```

必须优先讲清楚：

- 这个状态是 backend-local、static shared memory、DSM，还是文件系统状态。
- 是否有 owner、refcount、pin、generation、epoch、LSN、XID、SubXID。
- 是否允许其他 backend 直接访问。
- 指针能否跨进程使用，是否必须用 handle、offset 或 shared memory address。
- 字段何时初始化、何时更新、何时失效。
- 单个字段是否不能单独解释语义。

内核课要强调：

```text
raw field 不是语义；
field + flag + lifecycle state + lock/visibility context 才是语义。
```

---

## 6. 主流程 walkthrough

每节课至少要有一条真实主流程。

```text
入口函数()
  -> 中间函数A()
     -> 改变什么状态
  -> 中间函数B()
     -> 触发什么锁 / WAL / stats / wait / cleanup
  -> 返回给调用者什么语义
```

要求：

- 调用链要能在源码中跟到。
- 不只列函数名，要解释每一步改变了什么状态。
- 标出正确性边界：持锁区、critical section、WAL-before-data、snapshot horizon、invalidation、resource cleanup 等。
- 如果有主入口和旁路入口，要说明差异。

---

## 7. 生命周期与 ownership

每节课必须回答：

```text
谁创建？
谁持有？
谁释放？
ERROR / abort 时谁兜底？
长期对象如何失效？
```

推荐表格：

```markdown
| 对象 | 创建者 | owner | 释放 / 失效 |
| --- | --- | --- | --- |
| ... | ... | ... | ... |
```

必须区分这些机制：

- MemoryContext 管内存生命周期。
- ResourceOwner 管外部资源、pin、refcount 和 cleanup。
- refcount / pin 管“正在使用”。
- invalidation 管“语义过期”。
- WAL / redo 管 crash recovery。
- lock / latch / condition variable 管并发等待。

不要只写“事务结束时释放”。要尽量写到具体路径，例如：

```text
TopTransactionContext reset
ResourceOwner release
AtEOXact_* hook
shared invalidation
smgr close
```

---

## 8. 错误路径与异常路径

内核研发课程必须讲非 happy path。按主题选择 1 到 3 个最关键的异常路径即可。

常见异常路径：

- `ereport(ERROR)` 后的 cleanup。
- transaction abort / subtransaction abort。
- OOM / critical section。
- lock wait / deadlock / timeout。
- cache invalidation / invalidation queue overflow。
- WAL flush failure / crash recovery / redo。
- replication disconnect / timeout / WAL removed。
- snapshot overflow / subxid overflow。
- hash / sort spill。
- GEQO fallback。
- checkpoint / fsync request failure。

推荐格式：

```text
正常路径
  -> 某一步失败或状态过期
  -> 状态如何标记
  -> cleanup / retry / fallback 如何发生
  -> 调用者和观测入口看到什么
```

只有 happy path 的课程，对内核研发人员帮助有限。

---

## 9. 跨模块连接

每节课至少连接 2 到 4 个相邻模块，说明边界，而不是泛泛说“相关”。

常见连接：

- MemoryContext vs ResourceOwner。
- PGPROC / ProcArray vs Snapshot / VACUUM horizon。
- Buffer Manager vs WAL-before-data。
- WAL writer / checkpointer vs foreground latency。
- catcache / relcache vs Planner / Executor / logical decoding。
- Planner estimated rows vs Executor actual rows。
- Replication slot vs WAL retention / xmin retention。
- wait event vs 源码里的 latch / LWLock / IO。

推荐格式：

```markdown
| 本节机制 | 连接模块 | 为什么重要 |
| --- | --- | --- |
| ... | ... | ... |
```

---

## 10. 观测与诊断

只要主题能被观测，就要给入口，并说明指标粒度。

常见入口：

- `EXPLAIN (ANALYZE, BUFFERS, WAL)`
- `pg_stat_activity`
- `pg_stat_io`
- `pg_stat_wal`
- `pg_stat_replication`
- `pg_replication_slots`
- `pg_locks`
- `pg_stat_database`
- wait event
- server log
- `MemoryContextStats()`
- GUC / debug logging / injection point

推荐格式：

```markdown
| 现象 | 先看 | 再回到源码 |
| --- | --- | --- |
| ... | ... | ... |
```

必须说明粒度：

- 单 query。
- backend 当前状态。
- database / instance 累计统计。
- shared memory 当前状态。
- WAL / replication LSN 进度。

不要把统计指标解释成完整因果。课程要说明“能看到什么”和“看不到什么”。

---

## 11. 常见误区

误区不要求固定数量，优先选择 3 到 6 个真实研发风险。

好误区：

- 把 `xmax` 有值理解成 tuple 已删除。
- 把 WalUsage 理解成 WAL fsync 时间。
- 把 MemoryContext 当 ResourceOwner。
- 把 invalidation 当 lock。
- 把 estimated cost 当毫秒。
- 把 wait event 当完整性能归因。

推荐表格：

```markdown
| 误区 | 正确理解 |
| --- | --- |
| ... | ... |
```

---

## 12. 课堂实验

每节课给 1 到 3 个实验，实验要服务源码理解，不要变成普通 DBA 操作题。

优先选择：

1. 源码跟读实验：给入口函数、断点位置、需要画出的状态变化。
2. SQL 现象实验：给 SQL、系统视图或 EXPLAIN，再回到源码解释。
3. 边界实验：模拟 timeout、spill、slot retention、cache invalidation、long transaction。
4. 修改源码实验：加日志、计数器或 assert，观察行为，不要求改产品代码。

推荐格式：

````markdown
### 实验：<标题>

操作：

```sql
...
```

观察：

- ...

回到源码：

- `file.c:function()`
````

---

## 13. 讨论题

每节课给 4 到 8 个讨论题。题目要检查边界感，不考函数参数背诵。

好问题：

```text
为什么 invalidation 到达时 refcount > 0 的 catcache entry 不能立即 free？
```

弱问题：

```text
SearchCatCacheInternal 的参数有哪些？
```

讨论题建议覆盖：

- 为什么需要这个机制。
- 它和相邻模块的边界。
- 一个字段为什么不能直接解释为语义。
- ERROR / abort 时会发生什么。
- 哪个指标能看到，哪个看不到。
- 状态滞后、估算错误、消息丢失时 fallback 是什么。

---

## 14. README 与课程顺序

每个目录的 `README.md` 保持简短，面向选课和导航，不写成长课程。

README 只需要包含：

- 本目录约几节课。
- 本组课程解决什么核心问题。
- 每节课一句话学什么。

建议长度：30 到 120 行。不要在 README 里复制每节课的完整内容。

---

## 15. 写作前检查

开始写课前先做：

```text
1. 打开本地 PostgreSQL 源码，确认文件真实存在。
2. 用 rg 找主入口函数。
3. 找关键结构体定义。
4. 找 cleanup / error / invalidation / wait / stats 路径。
5. 找和其他模块的连接点。
6. 决定一条主 walkthrough。
7. 决定 1-3 个课堂实验。
```

不要只凭记忆写源码课。

---

## 16. 写作后检查

每次新增或大改课程后检查：

```bash
rg -n '^```' <file>
git diff --check -- <file>
```

人工 review：

```text
[ ] 课程定位清楚，核心问题具体
[ ] 核心文件、入口函数、阅读顺序可信
[ ] 至少一条真实源码主链路
[ ] 关键结构讲语义和边界，而不是字段清单
[ ] 生命周期和 ownership 讲清楚
[ ] 至少覆盖一种 ERROR / abort / overflow / fallback
[ ] 和至少两个相邻模块建立连接
[ ] 有观测或诊断入口，并说明指标粒度
[ ] 有常见误区
[ ] 有课堂实验
[ ] 有讨论题
[ ] 本节小结能沉淀 model / runtime / case
[ ] Markdown 围栏和 diff check 通过
```

---

## 17. 风格标准

- 使用中文。
- 面向内核研发人员，不写成入门科普。
- 少用抽象大词，多讲状态、边界、生命周期和不变量。
- 函数、字段、文件名使用反引号。
- 代码块使用 fenced code block。
- 不粘贴大段源码，只摘最小必要结构或用伪调用链。
- 文件名使用两位编号加主题：`NN-topic-name.md`。
- 标题格式统一：`# PostgreSQL <主题>`。
- 默认使用 ASCII 标点和代码符号；中文正文可使用中文标点。

---

## 18. 课程结尾模板

结尾不需要长，但必须把本节压缩成可复用的 mental model。

````markdown
## 本节小结

本节核心链路：

```text
入口
  -> 核心状态变化
  -> cleanup / invalidation / WAL / stats
```

核心边界：

```text
模块 A
  -> 管 ...

模块 B
  -> 管 ...
```

继续追问：

```text
对象在哪里？
谁拥有？
谁释放？
什么时候失效？
错误路径如何收尾？
哪些指标能观测？
```
````

这个模板不是强制文案，但结尾必须让学员带走能迁移到下一节课的判断框架。
