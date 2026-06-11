# PostgreSQL Cost Model Units 与 Planner 参数

## 课程定位

前置知识：已经理解 rows 如何从 selectivity 和 join cardinality 传播到 `Path.rows`，也知道 planner 比较的是候选 path 的 `startup_cost` 与 `total_cost`。

本节唯一主问题：

```text
PostgreSQL 为什么把 seq_page_cost、random_page_cost、cpu_tuple_cost、cpu_operator_cost、parallel_setup_cost 等定义为相对单位，而不是预测真实毫秒？
```

核心矛盾：planner 必须在执行前比较不同 path；但真实耗时取决于硬件、缓存、并发、数据布局、OS read-ahead、表空间、work_mem、CPU 表达式代价和运行时抖动。PostgreSQL 因此选择可调的相对成本单位，让不同 path 能在同一尺度上比较，而不是承诺 cost 等于 wall-clock time。

学完后应能判断：一次计划选择变化，是来自 rows 估错、page/CPU cost 参数不匹配、tablespace override、parallel cost、startup/total cost 权衡，还是具体 path cost 函数的模型边界。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节停在：

```text
RelOptInfo.rows
  -> Path.rows
  -> Plan.plan_rows
```

rows 只是数量。

planner 还要回答：

```text
用 seq scan 读这些行贵不贵？
用 index scan 随机访问贵不贵？
先排序再 merge join 值不值？
hash join build 是否可能 spill？
并行启动和 tuple 传输是否能被节省的 CPU 抵消？
```

这些问题进入成本模型。

本节只讲成本单位和参数。

它不深入每一种 scan path 的完整公式。

也不展开 join algorithm 的所有成本分支。

它建立一条主线：

```text
cost 参数
  + rows / pages / width / qual cost
  + path-specific formula
  -> startup_cost / total_cost
  -> add_path() 比较
```

下一组课程会把这条主线应用到具体 scan、sort、join、parallel path。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
costsize.c 定义全局成本参数和默认值；
planner 在 cost_*() 中读取当前会话 GUC、tablespace page cost、rows、pages、qual cost、parallel 参数；
把它们合成 Path.startup_cost / Path.total_cost；
add_path() 用这些相对值保留或丢弃候选 path；
EXPLAIN 显示这些成本，但它们不是毫秒。
```

这个模型有四个边界。

第一，cost 是相对单位。

默认情况下：

```text
seq_page_cost = 1.0
random_page_cost = 4.0
cpu_tuple_cost = 0.01
cpu_index_tuple_cost = 0.005
cpu_operator_cost = 0.0025
parallel_tuple_cost = 0.1
parallel_setup_cost = 1000.0
```

这些默认值定义在 `src/include/optimizer/cost.h`。

第二，cost 参数不是统计信息。

统计回答：

```text
会有多少行？
值分布是什么？
选择率是多少？
```

cost 参数回答：

```text
读取一个顺序页相对于 CPU tuple 处理贵多少？
随机页相对于顺序页贵多少？
并行启动和 tuple 传输在当前环境下是否昂贵？
```

第三，cost 不是执行时间。

它只需要在候选 path 之间保持比较意义。

第四，cost 传播离不开 rows。

如果 rows 错了，调成本参数通常只是在错误输入上重新加权。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/path/costsize.c` | 成本模型总入口；顶部注释解释成本单位、`startup_cost`、`total_cost`、partial fetch 插值；定义全局成本变量。 |
| 2 | `src/include/optimizer/cost.h` | 默认成本常量、enable flag、`cost_*()` 原型。 |
| 3 | `src/include/optimizer/optimizer.h` | 广泛使用的成本参数 extern 声明。 |
| 4 | `src/backend/utils/cache/spccache.c` | `get_tablespace_page_costs()` 如何读取 tablespace override 或回退全局参数。 |
| 5 | `src/backend/optimizer/path/costsize.c` | `cost_seqscan()`、`cost_index()`、`cost_sort()`、`cost_gather()`、join cost 函数如何消费参数。 |
| 6 | `src/backend/optimizer/util/pathnode.c` | `create_*_path()` 如何调用 cost 函数并保存结果。 |
| 7 | `src/backend/optimizer/util/pathnode.c` | `add_path()` 如何比较候选 path。 |
| 8 | `src/backend/commands/explain.c` | EXPLAIN 如何展示 cost 与 rows。 |
| 9 | `src/backend/utils/misc/postgresql.conf.sample` | 默认参数的配置展示，如 `seq_page_cost`、`random_page_cost`、`effective_cache_size`。 |

推荐阅读顺序：

```text
先读 costsize.c 顶部注释
  -> 看 cost.h 默认值
  -> 看 cost_seqscan() 的最小完整例子
  -> 看 get_tablespace_page_costs()
  -> 看 cost_gather() 的 parallel 参数
  -> 最后用 EXPLAIN 验证 cost 变化
```

不要先读 `cost_index()` 的全部公式。

它很长，包含 correlation、cache、visibility、loop_count 等细节。

本节先建立单位模型。

## 4. 成本单位不是毫秒

`costsize.c` 顶部注释说得很直接：

```text
Path costs are measured in arbitrary units established by basic parameters.
```

这个 arbitrary 不是随便。

它表示：

```text
planner 只要求不同工作类型能放在同一个相对尺度上比较；
不要求 total_cost 等于真实毫秒。
```

默认基准是：

```text
一次顺序 page fetch 成本为 1.0。
```

其他参数相对它定义。

例如：

```text
random_page_cost = 4.0
```

表示随机页读取默认被认为约等于四个顺序页成本。

如果数据库完全在 RAM 中，`costsize.c` 注释也说明：

```text
把 seq_page_cost 和 random_page_cost 设得更接近是合理的。
```

这不是让 cost 变成毫秒。

而是让 planner 的相对偏好更贴近当前硬件和缓存环境。

## 5. `startup_cost` 与 `total_cost`

每个 path 主要有两类成本：

```text
startup_cost:
  取出第一行之前已经花掉的成本。

total_cost:
  取出所有行的总成本。
```

`costsize.c` 顶部还给出 partial fetch 插值：

```text
actual_cost = startup_cost
  + (total_cost - startup_cost) * tuples_to_fetch / path->rows
```

这条公式解释了很多计划选择。

如果有 `LIMIT 1`：

```text
低 startup_cost 的 path 可能胜过低 total_cost 的 path。
```

如果要取完整结果：

```text
total_cost 更重要。
```

例如：

```text
Index Scan:
  startup 可能低；
  对大量随机页访问 total 可能高。

Sort:
  startup 往往高；
  一旦排序完成，输出很顺。
```

所以诊断时不能只看 total cost。

要结合：

```text
tuple_fraction
LIMIT / EXISTS / cursor
startup_cost 与 total_cost 差距
```

## 6. 全局参数与默认值

`src/backend/optimizer/path/costsize.c` 中定义了全局变量：

```text
seq_page_cost
random_page_cost
cpu_tuple_cost
cpu_index_tuple_cost
cpu_operator_cost
parallel_tuple_cost
parallel_setup_cost
recursive_worktable_factor
effective_cache_size
```

默认值来自 `src/include/optimizer/cost.h`。

`src/include/optimizer/optimizer.h` 里也有 extern 声明，因为很多 optimizer 模块需要读取它们。

这些值在实际运行中来自 GUC。

例如可以：

```sql
SHOW seq_page_cost;
SHOW random_page_cost;
SHOW cpu_tuple_cost;
SHOW cpu_operator_cost;
SHOW effective_cache_size;
```

也可以会话级调整：

```sql
SET random_page_cost = 1.1;
SET effective_cache_size = '16GB';
```

注意：

```text
这些设置改变 planner 的成本比较；
不会改变 executor 的真实执行算法实现。
```

如果调低 `random_page_cost` 后 planner 更愿意使用 index scan，原因是模型认为随机 I/O 不再那么贵。

不是 index scan 本身变快了。

## 7. Tablespace override

page cost 有一个重要局部覆盖。

`src/backend/utils/cache/spccache.c` 中的 `get_tablespace_page_costs()` 会读取表空间选项：

```text
random_page_cost
seq_page_cost
```

如果表空间没有设置，就回退全局参数。

源码语义：

```text
if tablespace opts missing or value < 0:
  use global random_page_cost / seq_page_cost
else:
  use tablespace override
```

`cost_seqscan()` 会调用：

```text
get_tablespace_page_costs(baserel->reltablespace, NULL, &spc_seq_page_cost)
```

`cost_index()` 会同时读取：

```text
spc_random_page_cost
spc_seq_page_cost
```

这解释了一个诊断现象：

```text
同样的表结构和统计信息，
放在不同 tablespace 上可能得到不同计划。
```

尤其在冷热盘、SSD/HDD、远端存储混用时，tablespace override 比全局参数更精确。

但它只覆盖 page cost。

CPU 参数仍是全局会话模型。

## 8. `cost_seqscan()` 的最小模型

`cost_seqscan()` 是理解成本单位最好的入口。

它先设置 rows：

```text
path->rows = param_info ? param_info->ppi_rows : baserel->rows
```

然后计算磁盘成本：

```text
disk_run_cost = spc_seq_page_cost * baserel->pages
```

再计算 CPU 成本：

```text
get_restriction_qual_cost()
cpu_per_tuple = cpu_tuple_cost + qpqual_cost.per_tuple
cpu_run_cost = cpu_per_tuple * baserel->tuples
```

最后 targetlist 成本按输出行数计：

```text
cpu_run_cost += path->pathtarget->cost.per_tuple * path->rows
```

这里有一个重要区别：

```text
扫描成本:
  取决于 baserel->pages 和 baserel->tuples。

输出投影成本:
  取决于 path->rows。
```

这就是为什么 rows 估错不一定等比例改变 seq scan 总成本。

如果表很大，顺序读 page 成本可能仍占主要部分。

如果 qual 或 projection 很贵，CPU 参数和 rows 会更敏感。

## 9. `cost_index()` 与 `effective_cache_size`

`cost_index()` 比 `cost_seqscan()` 复杂。

本节只抓两个核心点。

第一，index path 需要估算 fetch 多少 heap page。

这会受到：

```text
index selectivity
correlation
loop_count
random_page_cost
seq_page_cost
effective_cache_size
```

影响。

第二，`effective_cache_size` 不是内存分配。

`costsize.c` 注释说它是 PostgreSQL buffer 加 OS cache 的粗略 page 数估计。

它不会给 executor 更多缓存。

它只让 planner 估算：

```text
随机 page fetch 中，有多少可能已经在缓存里。
```

如果 `effective_cache_size` 设得过小，planner 可能过度悲观看待 index scan。

如果设得过大，planner 可能过度乐观看待随机访问。

诊断 index/seq scan 选择时，通常先看：

```text
rows 是否估准
random_page_cost 是否贴近介质
effective_cache_size 是否贴近实际缓存环境
correlation 是否影响 heap page fetch 估算
```

不要一上来只调 `enable_seqscan`。

## 10. CPU cost 与 qual cost

CPU 参数主要包括：

```text
cpu_tuple_cost
cpu_index_tuple_cost
cpu_operator_cost
```

`cost_qual_eval()` 会估算表达式求值成本。

函数、operator、targetlist 都可能贡献：

```text
startup
per_tuple
```

`cpu_operator_cost` 也会在很多地方作为基本单位：

```text
operator comparison
hash function
sort comparison
merge join comparison
function support cost
```

一个常见误区是：

```text
CPU cost 只影响 CPU-bound 查询。
```

实际上它也会影响 join 选择。

例如 hash join build 里：

```text
hash function cost
inner tuple insert cost
outer tuple probe cost
```

都会与 CPU 参数相关。

如果存储很快、数据常驻内存，CPU cost 的相对权重会更明显。

这时只调 page cost 可能不足以解释计划变化。

## 11. Parallel cost

并行路径有两个核心参数：

```text
parallel_setup_cost:
  启动并行执行的固定成本。

parallel_tuple_cost:
  worker 向 leader 传递 tuple 的成本。
```

`cost_gather()` 中：

```text
startup_cost += parallel_setup_cost
run_cost += parallel_tuple_cost * path->path.rows
```

`cost_gather_merge()` 还会加 merge heap 的比较成本，并对 tuple transfer 加一点额外系数。

这解释了：

```text
小查询通常不值得并行；
大查询是否并行取决于节省的 scan/join/aggregate 工作是否超过 setup 与传输成本。
```

如果把 parallel cost 设得太低，planner 会更愿意并行。

但真实系统中并行可能带来：

```text
worker 启动延迟
共享内存和队列开销
leader merge/projection 压力
CPU contention
I/O contention
```

cost 参数只是近似表达这些成本。

不是执行期保证。

## 12. enable flag 与 disabled nodes

很多人调计划时会使用：

```sql
SET enable_hashjoin = off;
SET enable_nestloop = off;
SET enable_seqscan = off;
```

当前 `costsize.c` 顶部注释说明：

```text
每个 path 会记录 disabled_nodes 数量；
包含较少 disabled nodes 的 path 应被认为更便宜。
```

历史上曾经用很大的常量增加 startup cost。

现在很多 enable flag 通过 disabled node count 表达。

这有助于避免一个禁用设置扭曲整棵计划树的 cost 数值。

但源码里仍然存在 `disable_cost` 这样的常量，用于少数特殊场景。

例如 hash join 中，如果内侧 MCV bucket 可能超过 hash memory，源码会用 `disable_cost` 强烈劝退该 hash path。

诊断时要区分：

```text
用户 enable_* GUC 影响 path 可选性偏好；
cost 参数影响相对单位；
特殊 disable_cost 影响某些病态 path 的惩罚。
```

## 13. 生命周期 / ownership / cleanup

成本参数本身是 backend-local 会话状态。

planner 在一次 planning 中读取当前值。

### 谁创建

全局变量在 `costsize.c` 中定义。

默认常量在 `cost.h` 中。

GUC 系统在会话中设置当前值。

tablespace page cost 来自 tablespace reloptions，并通过 spccache 读取。

### 谁持有

当前 backend 持有 GUC 值。

`Path` 持有计算后的：

```text
rows
startup_cost
total_cost
disabled_nodes
```

`Plan` 持有最终复制后的成本字段。

### 谁释放

Path 和 planner 搜索结构随 planner context 释放。

最终 Plan 树进入 executor。

GUC 值不随单次 planner 结束释放。

它属于会话或事务级设置。

`SET LOCAL` 会在事务边界恢复。

### ERROR 路径

如果 planning 中报错，planner context 清理临时对象。

成本参数不会因此被重置，除非它们本来是事务级 `SET LOCAL` 并随事务 abort 恢复。

## 14. 正确性与 fallback

cost model 不保证最佳计划。

它保证：

```text
合法 path 可以被比较；
估算缺失时仍能继续；
成本值不会轻易破坏 path search；
执行结果正确性不依赖 cost。
```

### fallback 1：缺省成本参数

如果用户不调，默认值提供一个通用假设。

它适合教学和一般环境。

不保证适合所有硬件。

### fallback 2：tablespace 未设置

`get_tablespace_page_costs()` 会回退全局 page cost。

这让 tablespace override 是局部优化，而不是必须配置项。

### fallback 3：rows 先 clamp

成本公式通常消费已经 clamp 的 rows。

这避免 NaN 或 0 rows 污染 cost。

### fallback 4：operator/function cost 粗略

很多表达式成本用 `cpu_operator_cost` 或 `procost * cpu_operator_cost` 近似。

如果函数真实成本差异极大，可以通过函数 cost 设置改善。

但 planner 仍然无法知道运行时 cache、branch、JIT、数据相关行为。

## 15. 观测与诊断入口

### 先看 EXPLAIN cost，不把它当毫秒

```sql
EXPLAIN
SELECT ...
```

输出：

```text
cost=startup..total rows=... width=...
```

诊断时先问：

```text
startup 与 total 谁影响当前查询？
rows 是否已经偏？
width 是否明显不合理？
哪个 path 被放弃可能与 page/CPU/parallel cost 有关？
```

### 再看实际时间

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...
```

实际时间用于校验模型。

但不要用：

```text
total_cost / actual_time
```

机械换算。

不同节点、缓存状态、并发环境下比例会变。

### 查看当前参数

```sql
SHOW seq_page_cost;
SHOW random_page_cost;
SHOW cpu_tuple_cost;
SHOW cpu_index_tuple_cost;
SHOW cpu_operator_cost;
SHOW parallel_setup_cost;
SHOW parallel_tuple_cost;
SHOW effective_cache_size;
```

### 查看 tablespace override

```sql
SELECT spcname, spcoptions
FROM pg_tablespace;
```

如果表在特殊 tablespace：

```sql
SELECT relname, reltablespace::regclass
FROM pg_class
WHERE relname = 'your_table';
```

### 回到源码入口

```text
相对单位说明:
  costsize.c 顶部注释

默认值:
  cost.h

全局变量:
  costsize.c
  optimizer.h

tablespace override:
  spccache.c: get_tablespace_page_costs()

seq scan:
  costsize.c: cost_seqscan()

index scan:
  costsize.c: cost_index()

parallel:
  costsize.c: cost_gather()
  costsize.c: cost_gather_merge()

sort:
  costsize.c: cost_sort()
```

## 16. 课堂实验

### 实验一：random_page_cost 改变 index 偏好

准备数据：

```sql
DROP TABLE IF EXISTS s24_t;

CREATE TABLE s24_t AS
SELECT g AS id,
       md5(g::text) AS payload
FROM generate_series(1, 200000) AS g;

CREATE INDEX ON s24_t(id);
ANALYZE s24_t;
```

默认参数下：

```sql
EXPLAIN
SELECT *
FROM s24_t
WHERE id BETWEEN 1000 AND 120000;
```

调低随机页成本：

```sql
SET random_page_cost = 1.1;

EXPLAIN
SELECT *
FROM s24_t
WHERE id BETWEEN 1000 AND 120000;
```

讨论：

```text
计划是否变化？
如果变化，是 rows 变了，还是 page cost 权重变了？
```

### 实验二：startup 与 LIMIT

```sql
RESET random_page_cost;

EXPLAIN
SELECT *
FROM s24_t
ORDER BY id
LIMIT 1;
```

再比较：

```sql
EXPLAIN
SELECT *
FROM s24_t
ORDER BY payload
LIMIT 1;
```

讨论：

```text
startup_cost 为什么重要？
LIMIT 下 total_cost 是否仍然是唯一判断？
```

### 实验三：parallel setup cost

```sql
SET max_parallel_workers_per_gather = 2;
SET min_parallel_table_scan_size = 0;
SET parallel_setup_cost = 1000;
SET parallel_tuple_cost = 0.1;

EXPLAIN
SELECT count(*)
FROM s24_t;
```

然后：

```sql
SET parallel_setup_cost = 0;
SET parallel_tuple_cost = 0;

EXPLAIN
SELECT count(*)
FROM s24_t;
```

讨论：

```text
并行计划是否更容易出现？
cost_gather() 中哪两个参数被改变？
```

### 实验四：CPU cost 对表达式过滤的影响

```sql
RESET parallel_setup_cost;
RESET parallel_tuple_cost;

EXPLAIN
SELECT *
FROM s24_t
WHERE md5(payload) LIKE 'a%';
```

调高：

```sql
SET cpu_operator_cost = 0.05;

EXPLAIN
SELECT *
FROM s24_t
WHERE md5(payload) LIKE 'a%';
```

讨论：

```text
计划变化来自 CPU expression cost，还是 rows/selectivity？
```

## 17. 源码练习

练习一：

阅读 `costsize.c` 顶部注释。

回答：

```text
为什么 cost 是 arbitrary units？
为什么 seq_page_cost 与 random_page_cost 可以按硬件调整？
为什么 effective_cache_size 不是 shared_buffers？
```

练习二：

阅读 `cost.h` 默认值。

写出：

```text
每个默认成本参数的数值；
它们相对哪个基准解释；
哪些参数与并行有关。
```

练习三：

阅读 `cost_seqscan()`。

回答：

```text
disk_run_cost 怎么算？
cpu_per_tuple 怎么算？
为什么 tlist cost 用 path->rows？
```

练习四：

阅读 `get_tablespace_page_costs()`。

回答：

```text
tablespace 没有设置时如何 fallback？
哪些成本参数支持 tablespace override？
```

练习五：

阅读 `cost_gather()`。

回答：

```text
parallel_setup_cost 加到 startup 还是 run？
parallel_tuple_cost 乘的是哪一层 rows？
```

## 18. 常见误区

误区一：

```text
EXPLAIN cost 可以直接换算成毫秒。
```

不能。

它是相对单位。

可以比较同一 planning 环境下的候选 path。

不能当跨环境绝对时间。

误区二：

```text
调 random_page_cost 能修复 rows 偏差。
```

不能。

rows 来自统计和 selectivity。

random_page_cost 只改变随机页读取相对权重。

误区三：

```text
effective_cache_size 会分配缓存。
```

不会。

它只是 planner 的缓存规模假设。

误区四：

```text
enable_seqscan=off 可以证明 index 一定更好。
```

它只能强迫 planner 尽量避开 seq scan，用于实验。

不应该当长期调优手段。

误区五：

```text
parallel cost 越低越好。
```

过低会让小查询也倾向并行。

真实系统可能因此增加启动、同步和竞争成本。

## 19. 讨论题

问题一：

```text
为什么 PostgreSQL 选择相对成本单位，而不是维护一个毫秒预测模型？
```

回答应覆盖：

```text
缓存状态
硬件差异
并发环境
执行前不可知信息
path 比较只需要相对顺序
```

问题二：

```text
如果一台机器全量数据常驻内存，random_page_cost 应该如何思考？
```

不要只给数值。

要说明：

```text
随机 I/O 与顺序 I/O 差距缩小；
但 CPU、cache miss、tuple visibility、index traversal 仍然存在。
```

问题三：

```text
为什么 LIMIT 查询里低 total_cost 不一定胜出？
```

必须引用：

```text
startup_cost
total_cost
costsize.c 顶部的 partial fetch 插值公式
```

问题四：

```text
一个计划实际运行慢，但 estimated rows 很准。下一步应该查什么？
```

可能方向：

```text
page cost 是否贴合存储
effective_cache_size 是否合理
CPU expression cost 是否低估
parallel setup/tuple cost 是否低估
work_mem spill 是否被正确估计
```

## 20. 本节小结

本节的核心模型是：

```text
cost model 把 rows、pages、width、qual cost 和可调相对单位合成 startup_cost / total_cost；
这些值用于 path 比较，不是 wall-clock time 预测。
```

源码主链路：

```text
cost.h 默认值
  -> costsize.c 全局参数
  -> cost_*() 函数读取 rows/pages/qual cost
  -> Path.startup_cost / Path.total_cost
  -> add_path() 比较
  -> Plan 成本字段
  -> EXPLAIN 展示
```

诊断主链路：

```text
先确认 rows 是否准；
再确认 page/CPU/parallel 参数是否符合环境；
再看 startup/total cost 对查询目标的影响；
最后回到具体 cost_*() 函数解释计划选择。
```

本节最重要的边界：

```text
统计估算决定“多少”；
成本参数决定“多贵”；
path cost 函数决定“这种做法如何把多少转成多贵”。
```

下一节开始进入具体 scan path：

```text
同样的 rows 和 cost 参数，在 seq scan、index scan、bitmap scan 中会被不同公式解释，因此计划选择会表现出明显的路径特征。
```
