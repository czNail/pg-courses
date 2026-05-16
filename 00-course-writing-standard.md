# PostgreSQL 内核课程生成标准

用途：作为 AI 新增、补写、重写 PostgreSQL 内核课程时的生成约束。

生成目标：面向有 PostgreSQL 使用经验，准备进入源码级内核研发、问题定位和性能诊断的工程师。

核心原则：

> 课程不是源码百科，也不是 API 清单。课程首先围绕一个核心内核矛盾展开，形成可线性阅读的推理链：问题 -> 状态 -> 主流程 -> 边界 -> 异常 -> 诊断。模板只是辅助，不是目标。

课程组可以覆盖多个讨论主题，但每节课必须只有一个唯一主问题，并围绕一个“不可同时完全满足”的系统 tension 展开。如果出现第二个同等重要的问题，就拆成下一节课。课程不是覆盖率竞赛；如果一个状态、字段、函数不能帮助解释本节核心矛盾，就不要展开。

不要把课程写成 systems research discussion。trade-off、演化和社区哲学只能服务一个可复现、可验证的 runtime 现象；否则删掉。

也不要把课程写成 runtime case 堆积。每节课最后必须把具体 runtime 现象压缩回一个可迁移的系统规律，形成 `runtime -> reusable abstraction` 的闭环。

允许课程从 bug、profiling、regression、commit、mailing list discussion 或异常 runtime 行为直接展开；不强制所有课程都先从抽象模型开始。

一节课最终只沉淀三类长期有用的内容：

| 类型 | 作用 |
| --- | --- |
| model | 稳定的系统抽象、边界和不变量 |
| runtime | 一条真实生命周期或调用主链路 |
| case | 一个可复现、可观测、能回到源码解释的问题 |

好的课程要讲清楚 PostgreSQL 为什么形成今天的样子：很多设计是历史连续性、correctness、性能、扩展生态和兼容性的共同结果。

---

## 1. 合格课程的核心要求

每节课至少回答这些问题：

```text
这个机制解决什么内核问题？
本节唯一主问题是什么？
为什么需要这个 abstraction，而不是更简单的实现？
课程是否沿一条线展开，而不是在清单项之间跳转？
是否存在一个状态随时间推进的完整故事？
本节可亲手复现并验证的 runtime 现象是什么？
核心状态放在哪里，谁能访问？
正常路径中状态如何变化？
谁创建、谁持有、谁释放，ERROR/abort 时怎么办？
它依赖哪些正确性机制：visibility、lock、pin、refcount、WAL、ordering？
它的 hot path 成本如何随 backend 数、relation 数、tuple 数、WAL 量等因素扩张？
它和哪些相邻模块形成边界或资源传播？
如果涉及 shared state，哪些后台进程参与推进这个状态？
哪些状态能观测、只能推断、完全不可见？
是否能在 SQL、日志、pg_stat、perf、gdb 或断点中看到一个关键状态变化？
哪些复杂性是 PostgreSQL 社区愿意接受的，哪些通常不值得？
本节最后能沉淀出哪个可迁移的系统规律？
哪些结论是 workload-dependent、hardware-dependent、version-dependent 或只能近似推断？
哪些问题来自 kernel implementation，哪些来自 workload、schema、SQL pattern 或 operational configuration？
```

课程至少应包含：

- 具体问题，而不是泛泛介绍模块。
- 一个唯一主问题；其它内容只能服务这个问题。
- 当前源码中真实存在的核心文件和入口函数。
- 至少一条能跟到源码的主流程 walkthrough。
- 至少一个状态随时间推进的完整故事。
- 至少一个能“看到 -> 解释 -> 回到源码”的 runtime 现象。
- 章节顺序能自然推进，后一节回答前一节留下的问题。
- 关键数据结构的语义解释和状态边界。
- 生命周期 / ownership / cleanup。
- 至少一种异常路径或 fallback。
- 跨模块连接，以及主题相关的成本、资源或正确性机制层次。
- 可执行的实验或源码练习。
- 本节小结。

---

## 2. 推荐章节结构

新课默认使用下面结构。可以按主题微调，但不要把“核心矛盾、主链路、生命周期、异常路径、观测入口”省掉。

章节结构必须服务线性阅读：先建立问题和状态，再进入主流程；正确性、异常路径、成本和观测都要回扣同一条主流程。不要把各节写成互不相干的资料卡片。

章节顺序可以按课程材料调整。如果最好的入口是一个 bug、性能 regression、profiling 火焰图、错误 patch 或 mailing list 争论，可以先从现场现象讲起，再回到状态、边界和源码。

```text
# PostgreSQL <主题>

## 课程定位
## 1. 本节在总主线中的位置
## 2. 核心矛盾与一句话运行模型
## 3. 核心文件分工与阅读顺序
## 4. 关键数据结构与状态
## 5. 主流程源码 walkthrough
## 6. 生命周期 / ownership / cleanup
## 7. 正确性机制层次
## 8. 错误路径 / 异常路径 / fallback
## 9. 成本、资源与跨模块传播
## 10. 观测与诊断入口
## 11. 常见误区
## 12. 课堂实验
## 13. 讨论题
## 14. 本节小结
```

轻量 runtime note、profiling case study、bug investigation、single-path walkthrough 可以压缩结构；小而尖的材料不必扩成完整大课，但必须保留：

```text
唯一主问题
  -> 可复现 runtime 现象
  -> 关键源码入口
  -> 状态变化 / 边界
  -> 可迁移小结
```

有些主题不涉及并发、WAL 或观测指标，可以说明“不涉及的原因”。不要为了模板硬凑内容。

写作时每节开头可以用一句话承接上一节，结尾用一句话引出下一节。避免频繁前后跳转、提前解释尚未定义的状态，或在后文才补上前文理解所需的关键概念。

---

## 3. 课程定位

开头必须短而明确：

```text
前置知识是什么？
本节唯一主问题是什么？
本节围绕哪个核心内核矛盾？
学完后能独立判断什么边界？
```

课程问题要具体，例如：

```text
已经不可见的旧 tuple version，什么时候可以真正移除？
为什么 WAL 必须先于数据页落盘？
为什么 buffer pin 和 buffer lock 必须分离？
为什么 snapshot 获取会成为 ProcArray 的扩展性瓶颈？
```

好的问题应带有 tension，例如 reclaim vs visibility、latency vs crash safety、global visibility vs scalability、generic abstraction vs CPU efficiency。

如果一个主题下有多个同等重要的问题，按问题拆成多节课。不要在一节课里同时讲多个主矛盾。

不要写成：

```text
学习 VACUUM 相关源码。
```

---

## 4. 源码文件与阅读顺序

每节课必须列核心源码文件。表里写职责，不写“实现相关函数”这种空话。

要求：

- 文件和函数必须尽量对应当前本地 PostgreSQL 源码真实存在的位置。
- 关键 `.h` 要列，因为内核研发经常先读状态结构和宏。
- 阅读顺序按 mental model 展开，不按文件名排序。
- 如果源码版本差异明显，要说明“本课基于哪个版本或分支”。
- 区分稳定语义和当前版本实现路径；不要把当前调用链误写成系统本质。
- 保留真实代码路径的 awkwardness：历史痕迹、重复逻辑、callback 链、double indirection、retry loop、状态耦合和 cleanup 顺序。不要把源码重写成理想化 architecture。
- 源码阅读优先级：

```text
入口
  -> 状态结构
  -> ownership / cleanup
  -> wait / retry / fallback
  -> invalidation / stats / WAL
```

不要从函数顶部线性读到尾、背 API、孤立理解单文件。优先找 state transition、ownership、cleanup、invalidation、retry / fallback。

---

## 5. 数据结构与状态边界

不要整段复制结构体源码。只讲影响理解和诊断的字段组合。

必须优先讲清楚：

- 这个状态是 backend-local、static shared memory、DSM，还是文件系统状态。
- owner、refcount、pin、generation、epoch、LSN、XID、SubXID 等字段的组合语义。
- 是否允许其他 backend 直接访问；指针能否跨进程使用。
- 字段何时初始化、更新、失效，单个字段是否不能单独解释语义。

内核课要强调：

```text
raw field 不是语义；
field + flag + lifecycle state + lock/visibility context 才是语义。
```

---

## 6. 主流程 walkthrough

每节课至少要有一条真实主流程，并把“时间”作为主轴。优先讲一个对象或状态从创建、被引用、状态变化、并发观察、失效到 cleanup / reclaim 的过程。

课程要在可观察行为和内部状态之间往返：SQL / runtime 现象 -> internal state transition -> 源码边界 -> 再回到可验证现象。避免停留在过高层概念，或陷入过低层字段细节。

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

必须区分这些机制：

- MemoryContext 管内存生命周期。
- ResourceOwner 管外部资源、pin、refcount 和 cleanup。
- refcount / pin 管“正在使用”。
- invalidation 管“语义过期”。
- WAL / redo 管 crash recovery。
- lock / latch / condition variable 管并发等待。

不要只写“事务结束时释放”。要尽量写到具体路径，例如 `TopTransactionContext reset`、`ResourceOwner release`、`AtEOXact_*`、shared invalidation。

---

## 8. 正确性机制层次

PostgreSQL 的正确性通常不是一个机制单独保证的。课程要说明本节依赖哪些机制、各自保证什么、不能保证什么。

建议按主题选择说明：

| 机制 | 主要保证 | 不要误解为 |
| --- | --- | --- |
| MVCC visibility / snapshot | 读到哪个版本 | 并发互斥 |
| heavyweight lock | 逻辑对象级排他或兼容性 | 内存安全 |
| LWLock / spinlock / atomic | shared memory 并发访问 | 事务语义 |
| pin / refcount | 对象正在被使用 | 对象语义仍然有效 |
| invalidation | 缓存语义过期通知 | 阻塞并发修改 |
| WAL / redo | crash safety 和恢复顺序 | 前台延迟一定低 |

如果涉及 memory ordering、critical section、WAL-before-data、ProcArray synchronization，要明确它们所在的边界。

---

## 9. 错误路径与异常路径

内核研发课程必须讲非 happy path。按主题选择 1 到 3 个最关键的异常路径，例如 ERROR cleanup、abort、OOM、lock wait、cache invalidation、WAL failure、replication timeout、snapshot overflow、spill、fallback、fsync failure。

不要把 fallback 当附录。要解释系统在压力、退化、overflow、信息不完整时，如何继续维持 correctness。

```text
正常路径
  -> 某一步失败或状态过期
  -> 状态如何标记
  -> cleanup / retry / fallback 如何发生
  -> 调用者和观测入口看到什么
```

只有 happy path 的课程，对内核研发人员帮助有限。

---

## 10. 成本、资源与跨模块传播

课程要说明这个机制为什么可能变慢，以及资源压力会如何扩散。不要把 correctness 讲完就结束。

性能成本模型按主题选择 2 到 4 项：

- CPU：tuple deforming、visibility check、expression evaluation、hash lookup。
- cache / branch：结构体布局、indirection、hash miss。
- contention：LWLock、WALInsertLock、ProcArray、lock table。
- slow path：spill、fallback、cache miss、redo、wait、retry。
- amplification：行数、分区数、subxid 数、连接数、relation 数、WAL 量。

讨论成本时要有数量级意识：说明它如何随 backend 数、relation 数、tuple 数、partition 数、WAL 量、subxid 数、cache miss 或 contention 扩张。

如果当前瓶颈被优化，要说明新的瓶颈通常会迁移到哪里，避免把性能问题解释成单点问题。

资源模型优先讲压力来源和传播路径：

| 资源 | 常见传播 |
| --- | --- |
| shared memory / local memory | memory accounting、fragmentation、OOM、context reset |
| WAL bandwidth / flush | foreground latency、replication lag、checkpoint pressure |
| IO queue / temp file | sort / hash spill、vacuum、checkpoint burst |
| xid horizon / snapshot | vacuum lag、bloat、clog retention |
| lock / ProcArray | wait、snapshot scalability、backend fan-out |

跨模块连接要说明边界，而不是泛泛说“相关”。每节课至少连接 2 到 4 个相邻模块。

涉及 shared state 时，必须说明哪些后台进程参与状态推进，例如 bgwriter、checkpointer、walwriter、autovacuum、startup、logical launcher、archiver。

---

## 11. 观测与诊断

只要主题能被观测，就要给入口，并说明指标粒度。

每节课必须锚定一个具体 runtime truth，而不只是给一组实验步骤。例如 snapshot scalability degradation、pin wait、spill、WAL flush stall、relcache invalidation、vacuum delay、ProcArray scan amplification。课程要让这个现象完成：

```text
看到现象
  -> 用状态和边界解释
  -> 回到源码验证
```

常见入口包括 `EXPLAIN (ANALYZE, BUFFERS, WAL)`、`pg_stat_activity`、`pg_stat_io`、`pg_stat_wal`、`pg_locks`、wait event、server log、`MemoryContextStats()`、debug logging / injection point。

必须说明粒度：

- 单 query。
- backend 当前状态。
- database / instance 累计统计。
- shared memory 当前状态。
- WAL / replication LSN 进度。

不要把统计指标解释成完整因果。课程要说明“能看到什么”和“看不到什么”。

高级诊断必须区分三类状态：能直接观测、只能从现象推断、几乎不可见。不要让读者误以为 `pg_stat_*` 就等于 runtime reality。

如果问题主要是 CPU / runtime overhead，要说明哪些现象必须依赖 `perf`、flamegraph、profiling 或断点实验，而不是只靠 `pg_stat_*`。

真实系统分析经常是不完整信息下的近似推理。对 workload-dependent、hardware-dependent、timing-sensitive、版本相关或存在争议的解释，要明确标注推断边界，不要为了“完整解释”而伪确定化。

不要把所有现象都解释成源码问题。课程应区分 kernel implementation、workload、schema shape、SQL pattern、xid age、checkpoint / autovacuum 配置、replication topology 等因素分别贡献了什么。

---

## 12. 常见误区

误区不要求固定数量，优先选择 3 到 6 个真实研发风险，例如：

- 把 `xmax` 有值理解成 tuple 已删除。
- 把 MemoryContext 当 ResourceOwner。
- 把 invalidation 当 lock。
- 把 wait event 当完整性能归因。

---

## 13. 课堂实验

每节课给 1 到 3 个实验，实验要服务源码理解，不要变成普通 DBA 操作题。

优先选择：

1. 源码跟读实验：给入口函数、断点位置、需要画出的状态变化。
2. SQL 现象实验：给 SQL、系统视图或 EXPLAIN，再回到源码解释。
3. 边界实验：模拟 timeout、spill、slot retention、cache invalidation、long transaction。
4. 修改源码实验：加日志、计数器或 assert，观察行为，不要求改产品代码。

---

## 14. 讨论题

每节课给 4 到 8 个讨论题。题目要检查边界感，不考函数参数背诵。

讨论题优先覆盖：

- 为什么需要这个机制。
- 它和相邻模块的边界。
- 一个字段为什么不能直接解释为语义。
- ERROR / abort 时会发生什么。
- 哪个指标能看到，哪个看不到。
- 状态滞后、估算错误、消息丢失时 fallback 是什么。

---

## 15. 可选深化段落

不是每节课都需要这些段落；但如果主题天然相关，就应该显式写出来。

| 段落 | 何时写 | 必须回答 |
| --- | --- | --- |
| 机制演化背景 | 设计明显有历史包袱或 trade-off | 最早解决什么问题，今天复杂性来自 correctness、性能、兼容性还是历史连续性 |
| 替代方案未采用原因 | 存在明显更现代或更简单的方案 | PostgreSQL 为什么没有采用：compatibility、correctness、extension ABI、operational stability 还是 migration cost |
| 社区工程取舍 | 本节涉及 patch design 或复杂性引入 | 哪些复杂性值得接受，哪些通常会因 contention、locality、maintainability 或 correctness cost 被拒绝 |
| 版本演化注意 | PG14 到当前版本行为、入口或资料明显变化 | 哪些 patch 改变 runtime 行为，哪些旧资料不宜直接引用，本课基于哪个版本 |
| extension / hook 边界 | 涉及 planner hook、executor hook、access method、logical decoding、background worker 等 | 哪些 API 稳定，哪些内部结构不能当 public contract |
| 社区案例 | 有真实 bug、commit 或 mailing list discussion 能解释设计选择 | 当时的 trade-off，最终方案的代价 |
| 未解决问题 / ongoing discussion | 当前实现仍有明显限制或社区争议 | 主要限制、扩展性问题、正在尝试改变它的 patch 或方向 |

---

## 16. README 与课程顺序

每个目录的 `README.md` 保持简短，面向选课和导航，不写成长课程。

一个目录可以规划多个讨论主题，但应按“一个主问题一节课”拆分。README 负责呈现这些问题之间的顺序关系，而不是把多个主问题塞进同一节。

README 只需要包含：

- 本目录约几节课。
- 本组课程解决什么核心问题。
- 每节课一句话学什么。

建议长度：30 到 120 行。不要在 README 里复制每节课的完整内容。

---

## 17. AI 生成流程

生成或重写课程时按顺序执行：

```text
1. 打开本地 PostgreSQL 源码，确认文件真实存在。
2. 用 rg 找主入口函数。
3. 确认本节唯一主问题；如果问题过多，拆课。
4. 找关键结构体定义。
5. 找 cleanup / error / invalidation / wait / stats 路径。
6. 找 hot path、slow path 和主要资源压力。
7. 找成本随规模扩张的变量。
8. 如果涉及 shared state，找参与状态推进的后台进程。
9. 找和其他模块的连接点。
10. 决定一个可复现、可验证的 runtime 现象。
11. 决定一条主 walkthrough。
12. 决定本节要压缩出的可迁移系统规律。
13. 标注 workload / hardware / timing / version 相关的推断边界。
14. 判断是否写成完整课程，还是轻量 runtime note / case study。
15. 决定 1-3 个课堂实验。
```

不要只凭记忆生成源码课。

---

## 18. AI 自动校验

生成后必须自动执行：

```bash
rg -n '^```' <file>
git diff --check -- <file>
```

内容质量按第 1 节核心要求和第 2 节章节结构自检，不再保留单独勾选清单。

---

## 19. 风格标准

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

## 20. 课程结尾模板

结尾不需要长，但必须把本节压缩成可复用的 mental model。

小结至少沉淀：

- 本节核心链路。
- 核心状态和边界。
- ownership / cleanup / invalidation。
- 错误路径如何收尾。
- 哪些指标能观测，哪些不能。
- 从本节 runtime 现象抽象出的可迁移系统规律。
- 哪些判断仍然依赖 workload、硬件、版本或只能近似推断。

不要重复正文；让学员带走能迁移到下一节课的判断框架。
