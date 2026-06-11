# PostgreSQL Executor hook 与 planner / utility hook 边界

## 课程定位

前置知识：熟悉
QueryDesc、PlannedStmt、EState、ExecutorStart
/ Run / Finish / End，以及扩展通过 hook
包裹核心入口的基本模式。

本节唯一主问题：

```text
什么问题应该放在 executor hook 观测，什么问题必须在 planner hook、ProcessUtility hook 或 event trigger 处理，如何避免在错误层次修补行为？
```

核心矛盾：扩展希望用一个统一入口观察或改写所有 SQL；但
PostgreSQL 把
parse、rewrite、plan、execute、utility
command
和对象访问拆成不同生命周期，每一层能看到的状态、能安全修改的对象和必须负责的
cleanup 都不同。

学完后应能判断：能判断一个扩展需求属于计划改写、执行观测、DDL
拦截、对象审计还是 EXPLAIN 输出增强，并能解释为什么
executor hook 不是万能入口。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前面几节已经把 auto_explain 和 executor
hook 的慢查询捕获路径放到运行时看清楚。

04 目录的总主线不是背执行节点名称，而是把
PlanState、tuple
流、执行期状态和观测指标放到同一条生命周期里。

```text
Plan / QueryDesc
  -> ExecutorStart 构造运行态
  -> ExecProcNode 推进 tuple 流
  -> instrumentation / pg_stat / wait event 记录现象
  -> EXPLAIN 把现象格式化
  -> 源码入口解释现象
```

本节只保留一个主问题。相邻机制会被引用，但只服务这条主线：先确认状态在哪一层产生，再确认它怎样被推进、观测和清理。

下一节会进入并行查询：当一个 PlanState tree 被
worker 复制执行时，观测指标如何跨进程回到 leader。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
把 SQL 处理看成分层状态机：post_parse_analyze_hook 看 Query，planner_hook 看 Query 到 PlannedStmt，Executor hook 看 QueryDesc 的运行期推进，ProcessUtility_hook 看 CMD_UTILITY，object_access_hook / event trigger 看对象级副作用。
```

这个模型要避免两个极端。一个极端是把源码读成 API
清单，知道每个函数名，却不知道哪一个状态在时间线上发生了变化。另一个极端是只看
EXPLAIN 或日志现象，把所有异常都归因到同一个模块。

读本节时要一直追问三个问题：第一，当前状态是
plan-time、executor
runtime、worker-local、DSM
还是格式化阶段状态；第二，它的 owner 和 cleanup
点是谁；第三，用户能通过哪个观测入口看到它的影响。

如果一个字段或函数不能帮助回答本节唯一主问题，就不要把它扩展成并列主题。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/executor/executor.h` | 声明 ExecutorStart_hook、ExecutorRun_hook、ExecutorFinish_hook、ExecutorEnd_hook 及标准入口。 |
| 2 | `src/backend/executor/execMain.c` | 在 ExecutorStart / Run / Finish / End 中调用 hook 或标准实现，展示运行期状态边界。 |
| 3 | `src/include/optimizer/planner.h` | 声明 planner_hook_type，说明 planner hook 接收 Query 并返回 PlannedStmt。 |
| 4 | `src/backend/optimizer/plan/planner.c` | planner() 在 hook 与 standard_planner() 之间分派，并上报 plan id。 |
| 5 | `src/include/tcop/utility.h` | 声明 ProcessUtility_hook_type 和 utility command 的调用协议。 |
| 6 | `src/backend/tcop/utility.c` | ProcessUtility() 调用 hook 或 standard_ProcessUtility()，处理 DDL / VACUUM / EXPLAIN 等命令。 |
| 7 | `src/include/parser/analyze.h` | 声明 post_parse_analyze_hook，覆盖 parse analysis 后、planning 前的 Query 观察点。 |
| 8 | `src/backend/parser/analyze.c` | parse_analyze 系列函数在生成 Query 后调用 post_parse_analyze_hook。 |
| 9 | `src/backend/commands/explain.c` | ExplainOneQuery_hook、ExplainOnePlan()、ExecutorStart / Run / Finish / End 的 EXPLAIN 特殊路径。 |
| 10 | `src/include/catalog/objectaccess.h` | object_access_hook 的对象级事件类型与宏边界。 |
| 11 | `src/backend/catalog/objectaccess.c` | 对象创建、删除、修改、namespace search、函数执行等访问事件的 hook 分派。 |

阅读顺序按 mental model
排列，不按文件名排序。先找入口，再找状态结构，再找
ownership 和 cleanup，最后才看观测输出。

本节源码核对基线：

```text
cd /home/highgo/postgres
git rev-parse --abbrev-ref HEAD
git rev-parse HEAD
```

期望输出分别是 `master` 和
`bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。

## 4. 关键数据结构与状态边界

| 状态 | 本节语义 |
| --- | --- |
| `Query` | parse analysis 后的语义树，适合检查 RTE、targetlist、CTE、权限相关形态，但还没有 path 和 PlanState。 |
| `PlannedStmt` | planner 输出的计划描述，适合看 join order、plan node、parallel flag、planId，不适合读取执行期 actual rows。 |
| `QueryDesc` | executor 与调用者之间的运行句柄，start 后持有 EState、PlanState、instrumentation 和 DestReceiver。 |
| `EState` | 一次 executor invocation 的 backend-local 根状态，包含 snapshot、range table、tuple table、instrument option。 |
| `ProcessUtilityContext` | utility command 的调用上下文，说明命令处于顶层、函数内、portal 内或 EXPLAIN 内。 |
| `ExplainState` | EXPLAIN 输出状态，能影响格式化和是否执行 ANALYZE，但不是执行器语义本身。 |
| `object_access_hook` | 对象访问事件的回调，不拥有 QueryDesc，也不能替代 DDL 命令层的权限和锁顺序。 |
| `previous hook 指针` | 扩展链式调用的顺序协议，保存和调用 previous hook 是生态兼容性边界。 |

这些状态不能孤立读取。raw field 不是语义；field
加 flag、lifecycle
state、owner、调用阶段和观测口径，才构成本节真正需要掌握的语义。

`Query` 的关键点是：parse analysis
后的语义树，适合检查
RTE、targetlist、CTE、权限相关形态，但还没有
path 和 PlanState。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`PlannedStmt` 的关键点是：planner
输出的计划描述，适合看 join order、plan
node、parallel
flag、planId，不适合读取执行期 actual
rows。 诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`QueryDesc` 的关键点是：executor
与调用者之间的运行句柄，start 后持有
EState、PlanState、instrumentation
和 DestReceiver。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`EState` 的关键点是：一次 executor
invocation 的 backend-local
根状态，包含 snapshot、range
table、tuple table、instrument
option。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`ProcessUtilityContext`
的关键点是：utility command
的调用上下文，说明命令处于顶层、函数内、portal 内或
EXPLAIN 内。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`ExplainState` 的关键点是：EXPLAIN
输出状态，能影响格式化和是否执行
ANALYZE，但不是执行器语义本身。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`object_access_hook`
的关键点是：对象访问事件的回调，不拥有
QueryDesc，也不能替代 DDL 命令层的权限和锁顺序。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`previous hook 指针`
的关键点是：扩展链式调用的顺序协议，保存和调用 previous
hook 是生态兼容性边界。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

## 5. 主流程源码 walkthrough

下面按时间顺序走一条主链路。每一步都回答“状态改变了什么”，而不是只记录函数名。

### 5.1. 步骤 1

普通 SELECT 进入 parser
后，parse_analyze 系列函数构造 Query，并在
`post_parse_analyze_hook`
存在时调用它。

这里可以审计 Query 形态或计算 query
id，但还不能看 plan cost、Plan node
或执行时间。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.2. 步骤 2

`planner()` 收到 Query、query
string、cursorOptions、boundParams
和 ExplainState，并在 `planner_hook`
存在时交给扩展。

这层能改写或包裹 planning，但扩展若要多次调用
standard_planner()，必须理解它会修改
Query 输入。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.3. 步骤 3

planner 返回 PlannedStmt
后，`pgstat_report_plan_id()` 记录
plan id。

这说明 plan id 的产生和执行时间无关，不能用
executor hook 事后改变计划身份。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.4. 步骤 4

正常可执行语句进入 `ExecutorStart()`，hook
或 `standard_ExecutorStart()` 创建
EState 和 PlanState tree。

这层适合打开
instrumentation、记录启动时状态、包裹执行生命周期。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.5. 步骤 5

`ExecutorRun()` 被 Portal 或
EXPLAIN 驱动，可被分批调用，方向和 count
来自调用者。

扩展在这里看到的是运行推进，不应该在这里改写 join
order 或 DDL 行为。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.6. 步骤 6

`ExecutorFinish()` drain
ModifyTable 副作用和 AFTER
trigger，并在 `ExecutorEnd()`
前单独存在。

慢查询观测若忽略 Finish，会把触发器和 queued
side effect 排除在执行时间之外。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.7. 步骤 7

`ExecutorEnd()` 清理
PlanState、tuple table、snapshot 和
executor context。

hook 必须调用 previous
或标准实现，否则状态会残留，下一层诊断会被污染。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.8. 步骤 8

CMD_UTILITY 走
`ProcessUtility()`，如
DDL、VACUUM、EXPLAIN、COPY、事务控制命令。

这些命令可能没有普通 executor 生命周期，放在
executor hook 中无法稳定看到。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.9. 步骤 9

对象级审计可落到 `object_access_hook` 或
event trigger。

这类入口看的是对象事件，不是某条 SQL 的 tuple
流，因此和 executor instrumentation
是互补关系。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.10. 步骤 10

`ExplainOneQuery_hook` 专门服务
EXPLAIN 计划输出增强。

它能改变解释过程，但不应该假装自己是普适执行观测入口。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

把这条链路合起来看，本节关注的不是某个函数是否复杂，而是这些函数共同维护了一个可解释的状态故事。

## 6. 关键状态随时间推进

同一状态在不同时间点有不同含义。下面把本节主状态压成一条时间线，方便读源码时定位断点。

| 阶段 | 阅读问题 |
| --- | --- |
| 准备阶段 | 输入对象已经存在，但本节的 runtime 状态可能还没有创建；此时错误通常由调用者或上层协议负责。 |
| 初始化阶段 | owner、memory context、DSM slot、instrumentation 或 hook wrapper 被建立；这一步之后 cleanup 责任开始变明确。 |
| 首次执行 | 延迟初始化、worker launch、函数指针首调用或状态机首个 phase 往往在这里发生。 |
| 稳定推进 | tuple、计数器、phase、reader 或 filter 计数随调用推进；诊断时最需要把单次调用和累计口径分开。 |
| 尾部收敛 | EOF、worker finish、trigger drain、queue detach、InstrEndLoop 或 ExplainFlushWorkersState 会把临时状态固化成可输出指标。 |
| 清理释放 | MemoryContext、ResourceOwner、DSM、DSA、reader、worker_instrument 等按 owner 释放；这之后裸指针不再有语义。 |

准备阶段 的判断标准是：输入对象已经存在，但本节的
runtime
状态可能还没有创建；此时错误通常由调用者或上层协议负责。
如果现场问题发生在这个阶段，应该优先回看本节第 5
节对应的源码入口，而不是跳到无关模块。

初始化阶段 的判断标准是：owner、memory
context、DSM slot、instrumentation
或 hook wrapper 被建立；这一步之后 cleanup
责任开始变明确。 如果现场问题发生在这个阶段，应该优先回看本节第
5 节对应的源码入口，而不是跳到无关模块。

首次执行 的判断标准是：延迟初始化、worker
launch、函数指针首调用或状态机首个 phase
往往在这里发生。 如果现场问题发生在这个阶段，应该优先回看本节第
5 节对应的源码入口，而不是跳到无关模块。

稳定推进
的判断标准是：tuple、计数器、phase、reader 或
filter
计数随调用推进；诊断时最需要把单次调用和累计口径分开。
如果现场问题发生在这个阶段，应该优先回看本节第 5
节对应的源码入口，而不是跳到无关模块。

尾部收敛 的判断标准是：EOF、worker
finish、trigger drain、queue
detach、InstrEndLoop 或
ExplainFlushWorkersState
会把临时状态固化成可输出指标。
如果现场问题发生在这个阶段，应该优先回看本节第 5
节对应的源码入口，而不是跳到无关模块。

清理释放
的判断标准是：MemoryContext、ResourceOwner、DSM、DSA、reader、worker_instrument
等按 owner 释放；这之后裸指针不再有语义。
如果现场问题发生在这个阶段，应该优先回看本节第 5
节对应的源码入口，而不是跳到无关模块。

## 7. 生命周期 / ownership / cleanup

生命周期问题要回答谁创建、谁持有、谁释放，以及 ERROR
或提前结束时谁兜底。

扩展的 `_PG_init()` 通常保存 previous
hook 指针，再把全局 hook 变量指向自己的
wrapper。

wrapper 在进入时决定是否采样、是否递归保护、是否切换
MemoryContext，并在合适位置调用 previous
hook 或 standard 实现。

executor hook 中的 QueryDesc
由调用者创建，扩展不能接管其生命周期，只能在允许的字段或私有结构中挂观测状态。

planner hook 中的 Query 可能被
standard_planner() 修改，想保留原始形态必须
copyObject()。

ProcessUtility hook 需要把
queryString、readOnlyTree、context、params、queryEnv、dest
和 qc 原样传递给下游。

ERROR 通过 longjmp 打断普通返回链，hook
中临时状态要么挂到合适 MemoryContext，要么用
PG_TRY / PG_FINALLY 恢复全局 guard。

这里最重要的边界是：MemoryContext 只能保证
backend-local 内存随 context 释放；DSM
/ DSA / tuple queue / worker /
hook 全局变量 / instrumentation
counter 还需要各自的 owner 协议。

因此不能写成“事务结束会释放”。本节所有对象都要说清楚是
query context、node private
context、ParallelContext、DSM、worker-local
state，还是扩展全局 hook 链。

## 8. 正确性机制层次

| 层次 | 机制与不变量 |
| --- | --- |
| 语义层次 | planner hook 处理计划选择；executor hook 处理运行观测；utility hook 处理 utility command；对象 hook 处理对象事件。 |
| 调用链兼容 | previous hook 必须被保存并调用，否则多个扩展加载顺序会互相覆盖。 |
| 事务与 snapshot | executor hook 进入时 snapshot 已按调用者协议激活，扩展不能绕过 `standard_ExecutorStart()` 自己拼装 EState。 |
| 递归保护 | hook 内执行 SQL 会重新进入 hook，必须区分外层语句和扩展内部语句。 |
| MemoryContext | 观测对象要挂到与 QueryDesc 或语句同寿命的 context，不能默认放入 TopMemoryContext。 |
| 错误路径 | hook wrapper 不能在 ERROR 后留下全局 flag、未恢复 previous 状态或半初始化 instrumentation。 |

语义层次 这一层保证的是：planner hook
处理计划选择；executor hook
处理运行观测；utility hook 处理 utility
command；对象 hook 处理对象事件。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

调用链兼容 这一层保证的是：previous hook
必须被保存并调用，否则多个扩展加载顺序会互相覆盖。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

事务与 snapshot 这一层保证的是：executor
hook 进入时 snapshot
已按调用者协议激活，扩展不能绕过
`standard_ExecutorStart()` 自己拼装
EState。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

递归保护 这一层保证的是：hook 内执行 SQL 会重新进入
hook，必须区分外层语句和扩展内部语句。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

MemoryContext 这一层保证的是：观测对象要挂到与
QueryDesc 或语句同寿命的 context，不能默认放入
TopMemoryContext。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

错误路径 这一层保证的是：hook wrapper 不能在
ERROR 后留下全局 flag、未恢复 previous
状态或半初始化 instrumentation。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

PostgreSQL
的执行器正确性通常来自多层机制叠加，而不是某一个字段或某一个锁。读源码时要把这些层次拆开。

## 9. 错误路径 / 异常路径 / fallback

| 场景 | 应如何解释 |
| --- | --- |
| 把 DDL 审计放进 executor hook | 很多 DDL 走 ProcessUtility，不一定构造普通 PlanState，结果会漏报或误判。 |
| 在 ExecutorRun 改写计划 | PlanState tree 已经构造，修改 planner-time 结构容易破坏 runtime invariant。 |
| 不调用 previous hook | auto_explain、pg_stat_statements 或其它扩展的观测链会被截断。 |
| hook 内执行 SQL 无递归 guard | 扩展自己的查询也被记录或再次改写，可能产生无限递归。 |
| 在 planner hook 持有 Query 裸指针到执行期 | Query 所在 context 和语义都不保证跨层有效。 |
| 把 EXPLAIN hook 当执行 hook | EXPLAIN ONLY 不执行计划，EXPLAIN ANALYZE 的执行路径也有特殊 cleanup 顺序。 |

场景“把 DDL 审计放进 executor
hook”的处理思路是：很多 DDL 走
ProcessUtility，不一定构造普通
PlanState，结果会漏报或误判。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“在 ExecutorRun
改写计划”的处理思路是：PlanState tree
已经构造，修改 planner-time 结构容易破坏
runtime invariant。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“不调用 previous
hook”的处理思路是：auto_explain、pg_stat_statements
或其它扩展的观测链会被截断。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“hook 内执行 SQL 无递归
guard”的处理思路是：扩展自己的查询也被记录或再次改写，可能产生无限递归。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“在 planner hook 持有 Query
裸指针到执行期”的处理思路是：Query 所在 context
和语义都不保证跨层有效。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“把 EXPLAIN hook 当执行
hook”的处理思路是：EXPLAIN ONLY
不执行计划，EXPLAIN ANALYZE 的执行路径也有特殊
cleanup 顺序。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

fallback 路径通常比正常路径更能暴露边界。比如
worker 启动失败、EXPLAIN
ONLY、不启动并行、nloops 为零、previous
hook 为空，都不是异常噪音，而是系统有意保留的边界。

## 10. 成本、资源与跨模块传播

每个 hook 都在高频路径上，哪怕只多一次
gettimeofday 或字符串拼接，也会放大到所有查询。

planner hook
的成本落在每次生成计划时，prepared statement
和 generic plan 会改变用户观察到的频率。

ExecutorRun hook 的成本可能按 cursor
fetch 次数重复出现，不能简单等同于一条 SQL 一次。

ProcessUtility hook 覆盖 DDL
和管理命令，日志量可能由迁移脚本和 autovacuum
类命令放大。

对象访问 hook 可能在 namespace
search、函数执行或 DDL 细节中多次触发，需要控制粒度。

跨层传递状态时，最好传递稳定 id
和短生命周期指针，不要复制整个 plan tree
作为常规日志。

成本传播要沿调用链解释：一个本地字段变化，可能在 EXPLAIN
输出中变成 rows/loops；一个 DSM
布局选择，可能变成 worker
明细是否可读；一个统计误差，可能变成 join order 和
I/O 放大。

不要只问“这个函数贵不贵”。更有用的问题是：这个成本随
tuple 数、worker 数、plan node
数、loops 数、输出格式、日志频率或统计目标如何扩张。

## 11. 观测与诊断入口

在 gdb 给
`ExecutorStart()`、`planner()`、`ProcessUtility()`
和 `ExplainOneQuery()` 分别下断点，观察同一
SQL 经过哪些层。

用 `EXPLAIN SELECT 1` 对比普通
`SELECT 1`，确认 EXPLAIN 先进入
utility，再可能进入 ExplainOneQuery /
ExplainOnePlan。

在扩展 hook 中记录
`queryDesc->operation`、`plannedstmt->commandType`
和 `estate->es_instrument`，不要只记录
SQL 字符串。

用 `CREATE TABLE AS SELECT ...`
观察 utility 层如何再次解释内部 query。

用 `auto_explain` 和一个自定义 executor
hook 同时加载，检查 previous hook
链是否全部执行。

用 `pg_stat_activity` 的 query id
/ plan id 辅助确认 planner 与
executor 边界。

诊断时建议保留三个层次的证据：SQL / EXPLAIN
看到的现象，gdb
或日志看到的运行态字段，以及源码中对应的状态推进点。只有三者能互相解释，结论才稳。

如果只有
EXPLAIN，没有源码入口，容易过度归因；如果只有源码，没有可复现现象，容易把正常实现细节误判成问题。

## 12. 课堂实验

### 12.1. 区分 SELECT 与 DDL 的入口

LOAD 一个最小 hook 扩展

对 `SELECT 1`、`CREATE TABLE t(a
int)`、`EXPLAIN SELECT 1`
分别打印进入层次

预期 SELECT 进入 planner 和
executor；DDL 进入
ProcessUtility；EXPLAIN 先是
utility 命令

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.2. 验证 ExecutorRun 可多次进入

DECLARE CURSOR FOR SELECT
generate_series(1,100)

FETCH 10 FROM cursor 多次执行

观察同一个 executor 生命周期中 Run 边界和
tuple count 的关系

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.3. 验证 previous hook 链

同时启用 auto_explain 和自定义 hook

在 wrapper 中先打印 previous 是否为空

确认调用 previous 后仍能看到 auto_explain
输出

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.4. 观察 EXPLAIN ONLY

执行 `EXPLAIN SELECT * FROM
pg_class`

断在 ExecutorStart，检查 eflags 是否带
EXPLAIN only 语义

确认没有普通 ExecutorRun 输出 tuple 给客户端

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.5. 递归 guard 练习

在 hook 内执行一条内部查询前设置 guard

内部查询返回后清理 guard

人为制造 ERROR，确认 PG_FINALLY 后 guard
被恢复

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

这些实验不要求一次证明所有结论。每次只验证一个状态边界，符合本目录“一个主问题一课”的写法。

## 13. 常见误区

把 EXPLAIN 的显示文本当作内部结构本身。

把一个计数器当成完整语义，忽略
loops、worker、phase 或 cleanup。

把 planner-time 的状态和
executor-time 的状态混用。

把 leader 本地状态和 worker 本地状态混用。

把 DSM 中的共享数据和 backend-local
指针混用。

只看正常路径，不读
ERROR、fallback、shutdown 或
EXPLAIN ONLY 路径。

这些误区的共同根源，是没有先问“这个状态属于哪个生命周期”。只要先定位生命周期，大多数源码路径都会自然收敛。

## 14. 讨论题

1. 如果一个扩展要拒绝某类 DDL，为什么
ProcessUtility hook 比
ExecutorStart hook 更合适？

2. 如果一个扩展要观察 actual rows 和
buffer usage，为什么 planner hook
看不到这些状态？

3. 为什么 hook wrapper 中调用 previous
hook 的位置会改变观测口径？

4. 为什么 EXPLAIN 相关 hook
适合输出增强，却不适合替代 runtime profiler？

5. 如何判断一个 hook 内分配的对象应该挂在
TopMemoryContext、query context
还是临时 context？

6. 当多个扩展同时修改 planner hook
时，怎样设计才能减少不可预期交互？

讨论时不要停在观点层面。每个回答都应能指回一个源码入口、一个状态字段和一个可复现的
runtime 现象。

## 15. 诊断案例：hook 放错层时如何回收问题

下面用现场诊断方式把本节模型压实。每个案例都从可见现象进入，再回到源码入口，最后给出归因边界。

### 15.1. DDL 没有被 executor hook 记录

现象是 SELECT 都能被扩展记录，CREATE
TABLE、ALTER TABLE 或 VACUUM
却缺失。源码路线不是继续追 ExecutorRun，而是从
ProcessUtility() 看 CMD_UTILITY
的分派。

断点放在
`src/backend/tcop/utility.c` 的
ProcessUtility()，再对比
`src/backend/executor/execMain.c`
的 ExecutorStart() 是否进入。

结论通常是需求属于 utility hook 或 event
trigger，而不是 executor
hook。修复方向是移动插桩层次，并保留 previous
ProcessUtility_hook。

### 15.2. EXPLAIN SELECT 被误认为普通 SELECT

现象是扩展在 EXPLAIN
场景中记录到不同调用顺序，甚至只看到 EXPLAIN
自身。源码路线应从 explain.c 的
ExplainOneQuery() 和
ExplainOnePlan() 开始。

断点观察 ExplainOnePlan() 中的
CreateQueryDesc、ExecutorStart、ExecutorRun、ExecutorFinish、ExplainPrintPlan
和 ExecutorEnd。

结论是 EXPLAIN 是 utility command
包住执行器路径。EXPLAIN ONLY 和 EXPLAIN
ANALYZE 的 executor 行为不同，hook
记录要标明 eflags 和 analyze 选项。

### 15.3. 多个扩展互相覆盖 hook

现象是加载新扩展后 auto_explain
或另一个审计扩展突然失效。源码入口本身没有复杂逻辑，关键在扩展是否保存并调用
previous hook。

检查 `_PG_init()` 是否把旧 hook
保存到静态变量，wrapper
是否在所有正常路径和异常保护后调用 previous 或
standard 实现。

结论是 hook 链是扩展生态协议。不能只把全局 hook
变量改成自己的函数，也不能在采样不命中时直接返回。

### 15.4. hook 内执行 SQL 导致递归

现象是日志重复、栈不断增长，或扩展记录了自己的内部
SQL。源码层面要看 executor hook
是否被重新进入。

在 wrapper 中打印递归 guard、当前 nesting
level 和 queryDesc 指针。人为制造
ERROR，确认 guard 在 PG_FINALLY
或等价路径中恢复。

结论是 hook 里做任何 SQL、SPI 或 catalog
查询都要设计递归边界。只靠布尔变量还不够，还要考虑 ERROR
longjmp。

### 15.5. planner hook 修改了执行期才知道的事实

现象是扩展想在 planner hook 中根据 actual
rows 或 buffer usage 改 plan。源码上
planner() 只持有 Query、参数和
ExplainState，没有执行后的
instrumentation。

对同一 SQL 分别断在 planner() 和
ExecutorEnd()，比较能看到的状态。planner
hook 能改 path/plan，不能读 actual
rows。

结论是计划选择和执行观测是不同层。想做反馈优化，需要持久化历史观测，再在下一次
planning 使用，而不是同一次执行中倒改。

### 15.6. 最小断点路线

如果课堂时间有限，只保留下面这条断点路线。它覆盖本节主问题中的状态创建、推进、观测和
cleanup 边界。

```text
ProcessUtility()
  -> standard_ProcessUtility() or previous hook
planner()
  -> planner_hook or standard_planner()
ExecutorStart()
  -> ExecutorStart_hook or standard_ExecutorStart()
ExecutorRun()
  -> ExecutorRun_hook or standard_ExecutorRun()
ExecutorFinish()
  -> ExecutorFinish_hook or standard_ExecutorFinish()
ExecutorEnd()
  -> ExecutorEnd_hook or standard_ExecutorEnd()
```

跑实验时不要一次打开所有断点。先确认第一处状态变化，再沿返回值或
owner
指针走到下一处。这样更容易判断偏差是来自调用层次、共享状态、worker
明细，还是 EXPLAIN 格式化。

## 16. 本节小结

Executor hook 是运行期观测入口，不是
parser、planner、DDL 和对象事件的统一替代品。

每层 hook
的输入对象不同：Query、PlannedStmt、QueryDesc、utility
statement 和 object event 不能混用。

previous hook
链是扩展生态的基本协议，破坏它会制造难诊断的兼容问题。

ERROR-safe cleanup、递归 guard 和
MemoryContext 归属比打印一条日志更重要。

真正可迁移的规律是：先判断状态属于哪一层生命周期，再决定在哪一层插桩。

本节最后沉淀的系统规律是：观测入口必须和状态生命周期对齐。入口选错，看到的不是不完整，就是语义已经过期。

下一节会进入并行查询：当一个 PlanState tree 被
worker 复制执行时，观测指标如何跨进程回到 leader。
