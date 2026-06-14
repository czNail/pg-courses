# PostgreSQL RLS / security barrier rewrite 与 qual 下推边界

## 课程定位

前置知识：已经理解 `Query`、`RangeTblEntry`、rewrite rule、view expansion、`RestrictInfo` 和 base relation path 生成的大致流程；最好已经读过 05 目录中 planner 入口、子查询 pullup、predicate 下推和 `RestrictInfo` 课程。

本节唯一主问题：

```text
RLS、security barrier view 和 leakproof function 如何约束 qual 下推和 planner 搜索空间？
```

本节核心矛盾：优化器越早执行用户 predicate，越容易减少行数、打开 index path、缩小 join 和 subquery 搜索空间；但 RLS policy 与 security barrier view 要先隐藏不该被低权限表达式观察到的行和值。如果普通 predicate 可以随意越过安全 qual，错误信息、函数副作用、短路顺序和 operator 行为都可能成为数据泄漏通道。

学完后应能独立判断：一个 qual 没有被下推，是因为 cost 不划算、SQL 语义不允许、outer join / LATERAL 限制，还是因为 security barrier 与 leakproof 约束；也能解释 `hasRowSecurity` 为什么会进入 plan cache invalidation，而不是只影响当前 rewrite 输出。

本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。本 checkout 中没有 `src/backend/optimizer/plan/prepsecurity.c`；security qualifier 的 planner 侧处理分散在 `initsplan.c`、`restrictinfo.c`、`allpaths.c`、`equivclass.c`、`clauses.c` 和 path 生成代码中。

## 1. 本节在总主线中的位置

05 目录前面的课程已经讲过 planner 的基本阶段：rewrite 后的 `Query` 进入 `planner()`，经过 `subquery_planner()`、expression preprocessing、jointree deconstruction、base path 和 join path 搜索，最后被压成 `Plan`。

本节插在 rewrite 与 planner 搜索之间，专门处理一个容易被误判的边界：

```text
parse/analyze
  -> rewrite 展开 view 和 RLS policy
  -> RTE.securityQuals 记录安全 qual
  -> planner 把 qual 包装成 RestrictInfo
  -> security_level / leakproof 约束移动和执行顺序
  -> path 搜索只能在合法搜索空间内比较 cost
```

这条链路说明 PostgreSQL 没有把 RLS 当成 executor 末端的“过滤器插件”。它更早进入 query tree，成为 planner 必须尊重的语义边界。

如果把本节和普通 predicate pushdown 混在一起，很容易得出错误诊断：认为“没有下推就是 cost 不好”或“建索引应该就能推”。实际情况是，安全边界会先缩小合法搜索空间，cost model 只能在剩余空间里选择。

本节只处理 RLS / security barrier / leakproof 对 qual 下推和搜索空间的限制。RLS DDL、权限模型、executor WCO 错误文案和完整 policy 管理是相邻主题，只在服务主问题时出现。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
rewrite 阶段把 view security barrier 和 RLS policy 变成 RTE 上的 securityQuals / WithCheckOptions；planner 再用 RestrictInfo.security_level 标记不同信任层级，只有 leakproof qual 才能越过更低 security_level 的安全 qual 提前执行或下推。
```

这个模型里有三个关键点。

第一，RLS policy 的 `USING` 不是普通 WHERE 文本。它在 rewrite 阶段通过 `get_row_security_policies()` 变成 `RangeTblEntry.securityQuals`，并且在必要时把 `Query.hasRowSecurity` 置位。

第二，security barrier view 也不是普通 view。`ApplyRetrieveRule()` 把 view RTE 改成 subquery RTE，同时设置 `rte->security_barrier`。后续 `set_subquery_pathlist()` 会把这个标记转成“不要把 leaky qual 推进 subquery”的限制。

第三，leakproof 不是性能 hint，而是安全承诺。`pg_proc.proleakproof` 告诉 planner：这个函数或操作符不会通过错误信息、副作用或可观察行为泄漏参数值。只有有这个承诺，planner 才能把高 `security_level` 的 qual 提前到更低层级执行。

把三者放在一起看，主矛盾就是：

```text
早过滤带来的搜索空间和执行性能收益
  vs
安全 qual 必须先遮蔽敏感行和值的执行顺序约束
```

PostgreSQL 的选择不是完全禁止优化，而是把 qual 分层：默认保守，leakproof 打开受控提前执行。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/rewrite/rowsecurity.h` | `RowSecurityPolicy`、`RowSecurityDesc`、RLS hook 和 `get_row_security_policies()` 声明。 |
| 2 | `src/include/catalog/pg_policy.h` | `pg_policy` catalog 中 `polcmd`、`polpermissive`、`polroles`、`polqual`、`polwithcheck` 的持久化边界。 |
| 3 | `src/backend/rewrite/rowsecurity.c` | 按命令和 role 选择 policy，组合 permissive/restrictive quals，生成 `securityQuals` 和 WCO。 |
| 4 | `src/backend/rewrite/rewriteHandler.c` | `fireRIRrules()` 展开 view / RLS，`ApplyRetrieveRule()` 设置 `security_barrier`，并把 RLS qual prepend 到 RTE。 |
| 5 | `src/include/nodes/parsenodes.h` | `RangeTblEntry.securityQuals`、`security_barrier`、`Query.hasRowSecurity` 的结构边界。 |
| 6 | `src/include/nodes/pathnodes.h` | `PlannerInfo.qual_security_level`、`RelOptInfo.baserestrict_min_security`、`RestrictInfo.security_level/leakproof`。 |
| 7 | `src/backend/optimizer/plan/initsplan.c` | jointree deconstruction 时把 `securityQuals` 与普通 qual 分层包装成 `RestrictInfo`。 |
| 8 | `src/backend/optimizer/util/restrictinfo.c` | `make_restrictinfo()` 记录 `security_level`，`restriction_is_securely_promotable()` 判断是否能提前用于 index/TID 等路径。 |
| 9 | `src/backend/optimizer/path/allpaths.c` | `set_subquery_pathlist()` 与 `qual_is_pushdown_safe()` 处理 security barrier view 的 subquery pushdown 限制。 |
| 10 | `src/backend/optimizer/util/clauses.c` | `contain_leaked_vars()` 判断表达式是否会把 Var 传给非 leakproof 函数。 |
| 11 | `src/include/catalog/pg_proc.h` | `proleakproof` catalog 字段，决定函数是否能作为 leakproof 参与 planner 判断。 |
| 12 | `src/backend/optimizer/path/equivclass.c` | EC 派生等值条件时如何处理 `security_level` 和 leakproof operator。 |

推荐阅读顺序不是从 planner 开始，而是先读 rewrite：安全边界的来源在 rewrite，planner 只消费已经进入 `Query` / `RTE` 的状态。

一个实用跟读顺序是：

```text
pg_policy / relcache policy
  -> get_row_security_policies()
  -> fireRIRrules()
  -> RTE.securityQuals / Query.hasRowSecurity
  -> initsplan.c 包装 RestrictInfo
  -> allpaths.c / restrictinfo.c 决定是否下推或提前执行
```

这样读可以避免把 `security_level` 误认为 parser 直接给出的属性。它是 rewrite 结果进入 planner 后形成的 planner-local 判断材料。

## 4. 关键数据结构与状态边界

### 4.1. `pg_policy` 与 relcache 中的 policy

`pg_policy` 是持久化 catalog。课程只需要关注这些字段组合：

| 字段 | 语义 |
| --- | --- |
| `polrelid` | policy 归属的 table。 |
| `polcmd` | 适用命令，可能是 SELECT/INSERT/UPDATE/DELETE 或 `*`。 |
| `polpermissive` | permissive policy 用 OR 合并；restrictive policy 用 AND 叠加。 |
| `polroles` | policy 适用 role，零表示 PUBLIC。 |
| `polqual` | `USING` 表达式，控制已有行可见性。 |
| `polwithcheck` | `WITH CHECK` 表达式，控制新行或更新后行是否允许。 |

源码运行时不会每次规划都把 catalog tuple 当作最终状态。relcache 会构造 `RowSecurityDesc`，其中 `RowSecurityPolicy` 保存 policy name、命令、roles、permissive 标记、`qual`、`with_check_qual` 和 `hassublinks`。

这里的长期 owner 是 catalog 和 relcache；单次 rewrite 得到的是 copy 后的表达式树。不要把某次 `securityQuals` 指针理解成 policy 的长期身份。

### 4.2. `RangeTblEntry.securityQuals`

`parsenodes.h` 对 `securityQuals` 的注释是本节的结构锚点：parser 输出时总是 NIL，rewrite 会添加用于 security barrier view 或 RLS 的布尔表达式列表，并要求按 listed order 在 relation 返回行之前测试。

关键语义：

```text
RTE.securityQuals 不是普通 baserestrictinfo；
它表达“这些 qual 来源更可信，必须先于低可信 qual 生效”。
```

RLS 与 security barrier view 都可以向同一字段贡献 qual。后来的 planner 会把每一层安全 qual 转成不同或相同的 `security_level`，再决定普通用户 qual 是否能移动。

`securityQuals` 是 query tree 的一部分，跟随 `Query` 进入 planner。它不是 executor 插件，也不是 scan node 临时状态。

### 4.3. `RangeTblEntry.security_barrier`

当 relation RTE 被 view 的 ON SELECT rule 展开成 subquery RTE 时，`ApplyRetrieveRule()` 设置：

```text
rte->subquery = rule_action
rte->security_barrier = RelationIsSecurityView(relation)
```

这个布尔值的含义很窄：这个 subquery 来自 `security_barrier` view，外层 restriction qual 不能随意推入 view 内部，除非 qual 不会泄漏 subquery 输出值。

它不是“这个 view 更慢”的标签，也不是强制物化。它只改变 subquery qual pushdown 的合法性判断。

### 4.4. `Query.hasRowSecurity`

`get_row_security_policies()` 即使在 `RLS_NONE_ENV` 时也会把 `hasRowSecurity` 返回为 true。原因是当前环境可能因为 `row_security` GUC 或 current role 改变而让同一 query 需要重新规划。

因此 `Query.hasRowSecurity` 的语义是：

```text
这个 query 的正确计划依赖 RLS 环境，plan cache 不能只按 SQL 文本复用。
```

它不是“当前 query 一定追加了 securityQuals”。在 bypass 场景下，当前不追加 qual，但仍然需要把环境依赖记录下来。

### 4.5. `RestrictInfo.security_level` 与 `leakproof`

planner 进入 qual 分发后，会把 expression 包装成 `RestrictInfo`。安全相关字段是：

| 字段 | 语义 |
| --- | --- |
| `security_level` | 数值越高，来源越不可信；低 level qual 必须先执行。 |
| `leakproof` | 当前 clause 是否不含会泄漏 Var 的函数或操作。 |
| `clause_relids` | clause 实际引用的 relids。 |
| `required_relids` | clause 最早可以合法执行的 relids。 |
| `outer_relids` / `incompatible_relids` | outer join 与 clone clause 相关的禁止移动边界。 |

`security_level` 只和 `leakproof` 一起才构成安全移动语义。单独看到 `security_level > 0` 不能断言不能下推；如果 `leakproof = true`，它仍可能被提前执行。

`make_restrictinfo()` 里有一个性能优化：只有 `security_level > 0` 时才调用 `contain_leaked_vars()` 判断 leakproof；level 0 的 clause 直接把 `leakproof` 置为 false，实际语义更接近“不需要知道”。

### 4.6. `RelOptInfo.baserestrict_min_security`

base rel 上可能同时挂着安全 qual 和普通 qual。`RelOptInfo.baserestrict_min_security` 记录该 relation 的 `baserestrictinfo` 中最小安全层级。

`restriction_is_securely_promotable()` 的核心规则是：

```text
若 rinfo.security_level <= rel.baserestrict_min_security，则可提前；
否则只有 rinfo.leakproof 为 true 才可提前。
```

这个函数会被 index path、TID path 等需要把 qual 提前用作访问条件的路径调用。换句话说，安全边界不只影响 executor qual 顺序，还会影响是否能生成某些更早过滤的 path。

### 4.7. `pg_proc.proleakproof`

`pg_proc.h` 中 `proleakproof` 是函数级 catalog 属性。普通用户不能随便声明 leakproof；`functioncmds.c` 会限制只有 superuser 能定义或修改 leakproof 函数。

原因非常直接：leakproof 函数可能在安全 qual 之前看到还未过滤的敏感值。如果这个承诺是假的，planner 的合法移动就会变成数据泄漏。

操作符是否 leakproof 通常通过其底层函数判断，例如 `get_func_leakproof(get_opcode(opno))`。因此诊断 operator predicate 时不能只看 operator 名称，还要看它绑定的 function。

## 5. 从 SQL 现象进入源码

本节最容易观察的现象不是“RLS 是否过滤了行”，而是同一个外层 predicate 是否能穿透安全边界。

典型 SQL 形态：

```sql
CREATE TABLE accounts(id int, tenant_id int, secret text);
ALTER TABLE accounts ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_policy ON accounts
  USING (tenant_id = current_setting('app.tenant')::int);

CREATE VIEW safe_accounts WITH (security_barrier) AS
  SELECT id, secret FROM accounts WHERE tenant_id = 1;

EXPLAIN SELECT * FROM safe_accounts WHERE secret = 'x';
```

你要观察的不是结果行，而是 plan 中 qual 的位置：

```text
外层 Filter / Subquery Scan Filter
  vs
被推入 view 内部的 scan Filter 或 Index Cond
```

如果 predicate 调用了可能泄漏输入值的函数，例如会根据 `secret` 抛出不同错误的函数，security barrier view 不允许它先于 view 内部 qual 执行。

如果 predicate 使用的是 leakproof equality operator，planner 有更多空间把它下推或用于 index path。

实际计划仍然受统计、索引和子查询形态影响。安全规则只决定“允许不允许”，不保证“允许后一定选择”。

可靠诊断流程：

```text
1. 先确认 view 是否 security_barrier，table 是否启用 RLS。
2. 用 EXPLAIN 看 predicate 最终在哪个 plan 节点执行。
3. 查询 pg_proc / pg_operator 确认函数或 operator 是否 leakproof。
4. 在 rewriteHandler.c 看 RTE.securityQuals 与 security_barrier 是否已写入。
5. 在 allpaths.c / restrictinfo.c 看 qual 是否被判定为 unsafe 或不可提前。
```

`EXPLAIN` 只能展示最终执行位置，不会告诉你哪个限制阻止了下推。源码断点才是判断 security barrier、leakproof、subquery safety 或 outer join 限制的可靠入口。

## 6. 主流程源码 walkthrough

### 6.1. view 展开与 `security_barrier`

主链路从 `rewriteQuery()` 内部调用 `fireRIRrules()` 开始。对于有 ON SELECT rule 的 view，`fireRIRrules()` 会找到 rule lock，然后调用 `ApplyRetrieveRule()`。

`ApplyRetrieveRule()` 把原来指向 view relation 的 RTE 改成 `RTE_SUBQUERY`：

```text
relation RTE for view
  -> rule_action Query
  -> rte->subquery = rule_action
  -> rte->security_barrier = RelationIsSecurityView(relation)
```

这里保留了 view relation 的 `relid`、`relkind`、`rellockmode` 和 `perminfoindex`，因为执行前仍要锁 view 并检查 view 权限。也就是说，view 展开不是简单把 relation RTE 删除，而是把权限和安全边界保留下来。

如果 view 是 security barrier view，后续 planner 看见的是一个 subquery RTE，且 `security_barrier = true`。这会影响 `set_subquery_pathlist()` 中是否允许外层 qual 推入 subquery。

这个阶段还没有 cost 比较。它只是在 rewrite 结果中记录边界。

### 6.2. RLS policy 注入

`fireRIRrules()` 在处理完 view、CTE 和 SubLink 后，最后遍历 query rtable，对普通 relation RTE 调用 `get_row_security_policies()`。

调用条件很窄：只有 `RTE_RELATION` 且 `relkind` 是普通表或分区表才进入。view 已经变成 subquery RTE，外部表、函数、values、CTE 都不是这里的对象。

`get_row_security_policies()` 先用 `check_enable_rls()` 判断当前 relation 在当前 user / checkAsUser / GUC 环境下是否需要 RLS：

```text
RLS_NONE      -> 不追加 qual，也不标记 hasRowSecurity
RLS_NONE_ENV  -> 不追加 qual，但标记 hasRowSecurity
RLS_ENABLED   -> 选择 policy，追加 securityQuals / WCO，标记 hasRowSecurity
```

`RLS_NONE_ENV` 是 plan cache 边界的核心。如果当前 role 是 table owner 或有 bypass 权限，当前可能不需要 qual；但换 role 或改 `row_security` 后，同一个 prepared statement 不能继续使用旧计划。

### 6.3. policy 选择和组合

policy 按 command 和 role 选择。目标 relation 的命令来自 query command；非目标 relation 在 UPDATE ... FROM 等场景中按 SELECT policy 过滤。

`get_policies_for_relation()` 做两件事：

```text
catalog / relcache policies
  -> 按 polcmd 和 role 过滤
  -> permissive list 与 restrictive list 分开
  -> restrictive policies 按 name 排序
  -> extension hook policies 追加进对应 list
```

permissive policy 与 restrictive policy 的合并语义不同：

```text
permissive USING:   q1 OR q2 OR q3
restrictive USING:  r1 AND r2 AND r3
```

源码实现上，`add_security_quals()` 会先追加每个 restrictive qual，再把所有 permissive qual 合并成一个 OR 表达式追加。如果没有任何 permissive qual，会追加一个常量 false，形成 default deny。

这解释了一个常见现象：启用了 RLS 但没有 policy 时，不是“没有过滤条件”，而是“所有行都不可见”。

### 6.4. `WITH CHECK OPTION` 与 `securityQuals` 的分流

`USING` 控制已有行可见性，适合放入 `securityQuals`。`WITH CHECK` 控制新行或更新后行是否允许，进入 `Query.withCheckOptions`。

INSERT / UPDATE 路径会调用 `add_with_check_options()`。如果 policy 没有显式 `WITH CHECK`，默认使用 `USING` 表达式。INSERT ... ON CONFLICT 和 MERGE 还有额外 WCO，因为同一条语句可能在不同阶段执行 SELECT/UPDATE/INSERT 语义。

本节主线关注 qual 下推，所以 WCO 只需要记住一个边界：

```text
securityQuals 影响 planner qual 分发和下推；
WithCheckOptions 主要由 executor 在修改行时检查，不等同于 scan predicate。
```

不要把 INSERT 被 RLS 拒绝的错误路径，误诊成 SELECT scan qual 没有下推。

### 6.5. RLS qual 的 SubLink 递归

如果 policy qual 或 WCO 中包含 SubLink，`get_row_security_policies()` 会返回 `hasSubLinks = true`。`fireRIRrules()` 随后会对这些新加入的表达式执行 SubLink rewrite，并做额外锁获取。

这一步很重要，因为 RLS policy 不是 parser 原始 query 的一部分。它是 rewrite 中后加入的表达式；如果里面包含子查询，必须补上 RIR rule 处理和 lock 处理，否则安全 qual 内部引用的 relation 可能缺少 rewrite 或锁。

这也是为什么 `fireRIRrules()` 把 RLS 应用放在最后：避免 query walker 在普通递归中重复进入刚追加的 security quals。

### 6.6. planner 接收 `securityQuals`

planner 进入 `subquery_planner()` 后会 deconstruct jointree。这个阶段把普通 WHERE / JOIN qual、RTE 上的 `securityQuals`、outer join scope 信息一起转成 `RestrictInfo`。

安全 qual 的核心变化是产生 `security_level`：

```text
security barrier quals: 较低 security_level
用户 query text quals: 较高 security_level
```

具体数值是 planner-local 细节；长期不变量是 lower level 必须先于 higher level，除非 higher level qual 是 leakproof。

这一步开始，RLS / security barrier 不再只是 RTE list 上的一串表达式，而是进入了 path 搜索会消费的 `RestrictInfo` 列表。

### 6.7. `make_restrictinfo()` 的 leakproof 判断

`make_restrictinfo()` 接收 `security_level` 参数并写入 `RestrictInfo.security_level`。

当 `security_level > 0` 时，它调用 `contain_leaked_vars()` 判断 clause 是否 leakproof。这个判断不是“函数名是否叫 leakproof”，而是沿表达式树检查是否有 Var 被传给可能泄漏的函数、操作符、subscripting 或相关节点。

伪流程：

```text
if security_level > 0:
    leakproof = !contain_leaked_vars(clause)
else:
    leakproof = false  # 当前不需要知道
```

因此调试时看到 level 0 qual 的 `leakproof = false` 不代表它会泄漏。它只代表 planner 没必要为最低安全层级支付检查成本。

### 6.8. base path 生成时的提前执行限制

当 index path、TID path 或其他 scan path 想把某个 qual 用作访问条件时，不只是检查 operator 是否匹配索引。它还要检查这个 qual 是否可以在 relation 的其他 restriction 之前执行。

`restriction_is_securely_promotable()` 给出核心规则：

```text
rinfo.security_level <= rel.baserestrict_min_security
  or rinfo.leakproof
```

如果普通用户 qual 的 security level 高于 RLS/security barrier qual，且它不是 leakproof，就不能被提前用作 index qual。这会直接缩小搜索空间：某些理论上可用的 index path 根本不会生成，或者只能作为普通 filter 留在更高层。

这不是 cost model “低估索引收益”。这是 planner 在 cost 之前做出的合法性裁剪。

### 6.9. security barrier view 的 subquery pushdown

`allpaths.c` 的 `set_subquery_pathlist()` 处理 subquery RTE。对于来自 security barrier view 的 subquery，代码设置：

```text
safetyInfo.unsafeLeaky = rte->security_barrier
```

随后每个外层 restriction qual 都要经过 `qual_is_pushdown_safe()`。如果 `unsafeLeaky` 为 true，且 `contain_leaked_vars(qual)` 为 true，该 qual 不能推入 subquery。

结果是：

```text
leaky 外层 qual
  -> 留在 SubqueryScan 的 qpqual / upperrestrictlist
leakproof 外层 qual
  -> 可能被 subquery_push_qual() 放入 subquery WHERE/HAVING
```

“可能”两个字很重要。security barrier 只是其中一个限制。subquery 自身如果有 DISTINCT、window function、set operation、unsafe output column、volatile 限制，也可能阻止下推。

### 6.10. EquivalenceClass 派生条件的安全边界

等值条件进入 EC 后，planner 可能派生新的 join qual 或 restriction qual。例如 `a.x = b.x` 和 `b.x = 1` 可能导出 `a.x = 1`。

security barrier 下不能让高 security level 的 leaky 条件借 EC 生成低层可执行的新条件。`equivclass.c` 因此会记录 EC source 的最小和最大 security level，并在需要跨安全层级派生时要求 operator leakproof。

这部分不是本节主流程，但它解释了为什么安全边界会影响 join 搜索空间：不只是原始 qual 移动受限，planner 自动派生的等价条件也要受限。

### 6.11. executor 看到什么

到 `createplan.c` 和 executor 阶段，很多安全判断已经体现在 plan 结构里：某些 qual 留在 `Filter`，某些进入 `Index Cond`，某些停在 `Subquery Scan` 外层。

executor 不重新证明“这个 qual 是否可以越过 RLS”。它按 plan node 的 qual 顺序和节点边界执行。真正的安全搜索空间裁剪已经在 rewrite/planner 完成。

这也是诊断时要回到源码的原因：EXPLAIN 展示的是裁剪后的结果，不展示被拒绝的候选 path。

### 6.12. `securityQuals` 的列表顺序就是安全优先级

`rewriteHandler.c` 有一个很容易漏掉的细节：RLS 新生成的 `securityQuals` 不是 append 到末尾，而是 prepend 到 RTE 现有 `securityQuals` 前面。

源码注释给出的理由是：RLS condition 应该先于已有的 security-barrier view qual 应用。

因此同一个 RTE 上的安全层级大致是：

```text
RLS quals added last by rewrite
  -> placed at the front of rte->securityQuals
  -> planner assigns lower security_level
existing barrier quals from view expansion
  -> later list elements
  -> planner assigns higher barrier levels
regular user quals
  -> root->qual_security_level
  -> one above all barrier levels
```

这里的“lower level”不是权限更低，而是执行顺序更早。`security_level = 0` 的 qual 必须在 `security_level = 1` 的 leaky qual 之前执行。

这解释了为什么 `list_concat(securityQuals, rte->securityQuals)` 是正确性逻辑，不是普通 list 操作风格。

如果你在 gdb 中只看 `rte->securityQuals` 是否非空，而不看元素顺序，会漏掉 RLS 与 security barrier view 叠加时的优先级。

### 6.13. `PlannerInfo.qual_security_level` 给普通 qual 设上界

`planner.c` 在扫描 rtable 时计算：

```text
root->qual_security_level = max(list_length(rte->securityQuals))
```

它表示当前 query level 需要多少层 security barrier qual。

之后普通 query qual 在 `initsplan.c` 中分发时使用 `root->qual_security_level`。也就是说，普通 WHERE / JOIN qual 的 security level 总是高于所有 security barrier qual。

没有 RLS 或 security barrier view 时，这个值是 0，所有 qual 都在 level 0。planner 因此不需要额外检查 leakproof。

有安全 qual 时，level 编号形成一个局部排序：

```text
barrier sublist 0 -> security_level 0
barrier sublist 1 -> security_level 1
...
regular query qual -> security_level root->qual_security_level
```

这个设计把“来源可信度”压缩成一个整数比较，让后续 path 生成、EC 派生和 qual 排序能共用同一个判断。

### 6.14. `process_security_barrier_quals()` 为什么把 Var-free qual 固定在 rel 层

`process_security_barrier_quals()` 遍历 `rte->securityQuals`。每个元素已经在 expression preprocessing 中变成 implicit-AND sublist。

它把同一 sublist 内的 clause 分配相同 `security_level`，不同 sublist 逐级递增。

源码里有一个刻意的“cheat”：调用 `distribute_quals_to_rels()` 时把 `ojscope` 传成 `jtitem->qualscope`，而不是更自然的 NULL。

这主要影响 Var-free qual。

普通 Var-free qual 可能被推到 query 顶部成为 gating qual。但 security barrier qual 即使不引用 Var，也应该归属于当前 relation 的安全边界，而不是被提升到整棵树顶端。

换句话说：

```text
security barrier qual 的执行位置不能只由它引用哪些 Var 决定；
它还携带“保护这个 RTE 返回行”的边界语义。
```

这也是本节反复强调 raw expression 不是语义的原因。

### 6.15. `order_qual_clauses()` 处理同一 plan node 内的执行顺序

安全边界不只影响 qual 能不能下推，也影响同一 plan node 内多个 qual 的执行顺序。

优化器 README 对规则的总结是：

```text
lower security_level first;
higher security_level later;
higher level qual can move earlier only if leakproof.
```

最终 plan qual list 的排序由 `createplan.c` 中的 `order_qual_clauses()` 集中处理。

这个函数通常按 security level 再按 cost 排序，但对便宜的 leakproof qual 有例外：便宜且 leakproof 的 qual 可以像 level 0 一样提前，避免为了安全层级做明显低效的求值顺序。

这里还有一个成本边界：PostgreSQL 不会因为一个 qual 被标成 leakproof，就无条件把非常昂贵的表达式提前执行。README 中的规则是，只有成本低于大约 `10 * cpu_operator_cost` 的 leakproof qual 才按 level 0 排序。

因此安全排序不是“完全忽略成本”，而是：

```text
leaky qual 必须遵守 security_level；
便宜 leakproof qual 可以提前；
昂贵 leakproof qual 仍要避免制造无谓 CPU 成本。
```

诊断 CPU 回归时要同时看 `security_level`、`leakproof` 和 qual cost，而不是只看最终 `Filter` 文本顺序。

### 6.16. EC 规则防止“借等值推导泄漏”

EquivalenceClass 是安全边界最容易被绕开的地方。

考虑：

```text
barrier qual: t.x = t.y
query qual:   t.x = constant
query qual:   leaky_function(t.z)
```

如果 EC 直接用高层 query qual 推导出低层可执行的新 qual，就可能让 leaky function 在 barrier qual 之前观察到不该观察的数据。

因此 EC 处理有三条约束：

1. 作为 EC source 的 qual 必须是 leakproof，或者 `security_level = 0`。
2. EC-derived qual 使用 source 中的最小 `security_level`，不是最大值。
3. 如果 source 中存在高于 0 的 security level，派生 qual 选用的 equality operator 必须 leakproof；否则把 EC 标成 broken，回退到原始 source clauses。

这三条规则保留了常见 btree equality operator 的优化空间，同时防止不可信 query qual 通过 EC 生成一个更早执行的 leaky derived qual。

这也是为什么本节的主问题说“约束 planner 搜索空间”，而不只是“约束 WHERE 下推”。EC 是否能生成派生条件，会直接改变 join path、index path 和 rows 估算的候选集合。

### 6.17. `contain_leaked_vars()` 的真实判定范围

`contain_leaked_vars()` 的目标不是判断表达式是否含函数，而是判断“是否有 Var 被传给可能泄漏的节点”。

典型会检查的节点包括：

```text
FuncExpr
OpExpr
DistinctExpr
NullIfExpr
ScalarArrayOpExpr
CoerceViaIO
ArrayCoerceExpr
SubscriptingRef
RowCompareExpr
MinMaxExpr
```

普通 function / operator 节点会通过 `check_functions_in_node()` 查询 `get_func_leakproof()`。如果节点中存在非 leakproof 函数，并且节点下方包含 Var，就认为可能泄漏。

`RowCompareExpr` 会逐对检查比较 operator；`MinMaxExpr` 会查类型缓存中的 btree comparison function；`SubscriptingRef` 会使用 subscripting support routine 提供的 `fetch_leakproof` / `store_leakproof`。

因此下面两类表达式诊断不同：

```text
leaky_func(constant)
  -> 不含 Var，通常不构成“泄漏关系值”

leaky_func(secret_col)
  -> Var 进入 leaky function，不能越过 barrier
```

这能解释一些看似奇怪的计划：同一个函数包常量时可以被移动，包 relation column 时不能移动。

### 6.18. index path 和 TID path 的安全检查是候选生成前置条件

`indxpath.c` 中匹配 index column 前会先调用 `restriction_is_securely_promotable()`。

`tidpath.c` 在判断 TID qual 时也做同样检查。

这说明安全边界发生在 path candidate 形成之前：

```text
qual 是否匹配 index operator class
  之前先问
qual 是否允许早于同 rel 的低 security_level qual 执行
```

如果答案是否定的，候选 index qual 不会继续参与后续 path 构造。

所以 `EXPLAIN` 里没有 `Index Cond` 时，至少有三类可能：

```text
表达式不匹配索引
匹配但 cost 竞争失败
匹配但安全层级不允许提前执行
```

只有第三类需要看 `security_level/leakproof`；前两类分别回到 operator class、collation、expression index、partial index predicate 和 cost 诊断。

## 7. 生命周期 / ownership / cleanup

### 7.1. policy 的长期生命周期

`CREATE POLICY`、`ALTER POLICY`、`DROP POLICY` 修改 `pg_policy`。table 的 `relrowsecurity` 与 `relforcerowsecurity` 在 `pg_class` 中。relcache 会缓存 relation 的 row security 描述。

长期对象 owner 是 catalog；backend 通过 syscache/relcache 读取。DDL 变更通过 catalog invalidation / relcache invalidation 让其他 backend 的缓存语义过期。

### 7.2. rewrite 阶段对象生命周期

`get_row_security_policies()` 会 copy policy expression，把 Var varno 从 policy 中的 relation-local 编号改成当前 query 的 rt_index。生成的 `securityQuals` 和 WCO 都归属当前 query tree。

这些表达式是 backend-local memory，不跨 backend，不写回 catalog。ERROR 时由当前 query / transaction memory context 清理，不需要单独释放每个 qual。

### 7.3. planner 阶段对象生命周期

`RestrictInfo`、`RelOptInfo`、`pushdown_safety_info` 都是 planner-local 状态。它们在一次 `planner()` 调用内创建和消费。

`RestrictInfo.security_level` 不进入 catalog。它可能影响最终 `Plan` 的 qual 位置，但 `RestrictInfo` 指针本身不会成为 executor 的长期接口。

### 7.4. plan cache 与环境依赖

`Query.hasRowSecurity` 会让 plan cache 认识到计划依赖 role / RLS GUC 环境。这里的 invalidation 不只是 catalog DDL 触发，也包括“当前环境让 RLS 是否启用发生变化”。

因此 prepared statement 中的 RLS 行为不能只看第一次 PREPARE 时有没有 security qual。正确问题是：这个 cached plan 是否记录了 RLS 环境依赖，role/GUC 改变时是否需要 replanning。

字段级链路在 `plancache.h` 和 `plancache.c` 中：

```text
CachedPlanSource.dependsOnRLS
CachedPlanSource.rewriteRoleId
CachedPlanSource.rewriteRowSecurity
CachedPlan.dependsOnRole
```

`CompleteCachedPlan()` / revalidation 路径会通过 `extract_query_dependencies()` 保存 `dependsOnRLS`，同时记录 rewrite 时的 `GetUserId()` 和全局 `row_security` GUC 值。

`RevalidateCachedQuery()` 中，如果 cached query 当前有效但满足下面条件，就会把 query tree 标成无效并重新 parse/rewrite：

```text
dependsOnRLS
  and
(rewriteRoleId != GetUserId()
 or rewriteRowSecurity != row_security)
```

这就是 `RLS_NONE_ENV` 必须设置 `hasRowSecurity` 的原因：当前 rewrite 可能没有追加任何 policy qual，但 plan source 仍需要知道“我的正确性依赖 role/GUC 环境”。

还有一个性能边界：`CachedPlanAllowsSimpleValidityCheck()` 会拒绝 `dependsOnRLS` 的 plan 走简单有效性快路径。因为简单快路径只看 plan 指针和 search path 等廉价条件，不足以覆盖 RLS 环境变化。

所以在连接池里诊断 prepared statement 时，要把这三个状态分开：

```text
SQL text 是否一样
rewrite 发生时的 role / row_security 是否一样
generic plan 是否因为 dependsOnRLS 被迫走更保守的 revalidation
```

### 7.5. ERROR / abort cleanup

RLS rewrite 过程如果遇到错误，例如 policy subquery rewrite 失败、递归 rule 检测失败、权限或锁获取失败，会按普通 query rewrite ERROR 路径退出。

没有 shared memory 状态需要手工回滚。catalog 变更由事务语义处理；当前 query tree、policy copies、planner-local lists 都随 memory context 被释放。

## 8. 正确性机制层次

本节正确性不是由一个机制保证，而是多层边界叠加。

| 层次 | 机制 | 保证什么 | 不保证什么 |
| --- | --- | --- | --- |
| catalog | `pg_policy`、`pg_class.relrowsecurity`、`pg_proc.proleakproof` | 持久化 policy、RLS 开关、函数 leakproof 承诺 | 当前 query 已经应用这些状态 |
| relcache/syscache | policy 与 function metadata 缓存 | 低成本读取 catalog 语义 | 缓存永远不过期 |
| rewrite | `RTE.securityQuals`、WCO、`hasRowSecurity` | 把安全约束变成 query tree 状态 | 选择最优 plan |
| planner | `RestrictInfo.security_level/leakproof` | 限制 qual 移动、下推、EC 派生和 path 生成 | 执行时重新验证 policy 来源 |
| executor | plan qual、WCO | 按计划过滤行并检查写入行 | 重新搜索更优安全执行顺序 |
| invalidation | relcache/syscache/plancache | catalog、role、GUC 变化后不复用过期语义 | 阻塞并发修改 |

这里没有 MVCC 可见性替代 RLS。MVCC 决定 tuple version 是否对 snapshot 可见；RLS 决定 snapshot 可见的 tuple 是否对当前 role / policy 可见。

这里也没有 heavyweight lock 替代 security barrier。锁保护 DDL/DML 并发语义，不保护表达式执行顺序中的信息泄漏。

leakproof 的正确性依赖函数作者承诺。PostgreSQL 通过 superuser 限制降低风险，但无法从 C 函数体自动证明它绝不泄漏。

## 9. 错误路径 / 异常路径 / fallback

### 9.1. 没有 permissive policy：default deny

启用 RLS 后，如果没有匹配的 permissive policy，`add_security_quals()` 会追加常量 false。

runtime 现象是 SELECT 看不到任何行，UPDATE/DELETE 找不到可操作行。源码原因不是 planner 估算错，而是 rewrite 已经把可见性压成 false qual。

诊断时要检查：policy 是否匹配当前 role 和 command；是否只有 restrictive policy；是否以为 `USING` 会自动应用到所有命令。

### 9.2. `RLS_NONE_ENV`：当前 bypass 但仍需 replanning

table owner、bypassrls 角色或特定 `row_security` 环境可能让当前查询不追加 RLS qual。源码仍可能把 `hasRowSecurity` 置 true，因为换 role 或 GUC 后结果会变。

这类问题在 prepared statement、connection pool 和 `SET ROLE` 场景中最容易被误诊。看到当前计划没有 RLS qual，不代表 plan cache 可以无条件复用。

### 9.3. policy qual 含 SubLink

RLS policy 可以包含 subquery。rewrite 会在追加这些 qual 后额外对 SubLink 执行 RIR rules 和锁获取。

异常风险包括 rule recursion、权限检查、锁获取和子查询自身 RLS。失败时走普通 ERROR 路径，当前 query 构造中止。

### 9.4. security barrier view 遇到 leaky predicate

外层 predicate 如果把 view 输出列传给非 leakproof 函数，`qual_is_pushdown_safe()` 会拒绝推入 subquery。

fallback 是把 predicate 留在上层 SubqueryScan 或更高节点执行。结果可能是更多行先从 view 内部流出到上层，再过滤；但安全顺序保持正确。

### 9.5. leakproof 标记错误

如果 superuser 错把会泄漏输入值的函数标成 leakproof，planner 可能把它提前到安全 qual 之前执行。PostgreSQL 的 planner 逻辑没有办法在运行时发现这个承诺是假的。

因此 leakproof 是扩展和内核函数声明中的高风险属性。它不是“让索引更容易用”的普通优化开关。

### 9.6. EC 派生失败或受限

安全层级较高的 leaky equality 不能随意参与派生低层执行的 EC 条件。fallback 是少生成或延迟使用派生 qual。

runtime 现象可能是 join order 或 index path 少了某些看似自然的等值条件。诊断时要看 EC source 的 security level 和 operator leakproof，而不是只看 SQL 中有没有等号。

## 10. 成本、资源与跨模块传播

安全边界影响性能的方式是“先缩小合法搜索空间，再让 cost model 在剩余空间里比较”。常见扩张变量如下。

| 变量 | 传播路径 |
| --- | --- |
| policy 数量 | rewrite 要 copy、改写、合并更多 qual；planner 要包装更多 `RestrictInfo`。 |
| policy 表达式复杂度 | `contain_leaked_vars()`、选择率估算、predicate 分类和 executor qual cost 增加。 |
| policy 中 SubLink | 触发额外 rewrite、锁、权限和子查询规划；也可能引入 RLS 递归依赖。 |
| security barrier view 层数 | 外层 qual 每过一层 subquery 都要重新判断 pushdown safety。 |
| 非 leakproof 用户 predicate | 更少 index/subquery pushdown，更多 rows 到上层再过滤。 |
| 分区数量 | 每个 child relation 的 RLS、restriction 和 path 生成都会放大局部判断成本。 |
| role / GUC 切换频率 | plan cache 更容易因 RLS 环境依赖而 replanning。 |

跨模块传播要具体看边界。

rewrite 和 relcache 的边界：`pg_policy` 不是每次从 catalog 线性扫描到 executor，而是通过 relcache policy 描述进入 rewrite。DDL invalidation 负责让缓存过期。

rewrite 和 planner 的边界：`RTE.securityQuals` 是 query tree 状态；planner 不重新查询 policy catalog 来决定安全 qual 内容。

planner 和 path 生成的边界：`security_level` 限制的是 path candidate 是否存在，以及 qual 能不能成为 index qual / pushed-down qual，不只是最终 `Filter` 文本排序。

planner 和 stats 的边界：即使 leaky qual 不能提前执行，planner 仍可能用它估算结果行数；但是否能用 extended stats 还要看表达式 leakproof 和用户对统计对象相关列的权限。

plan cache 和会话状态的边界：RLS 依赖 role 与 `row_security` GUC。`SET ROLE`、权限变化或 GUC 变化可能让 cached plan 失效或需要重建。

## 11. 观测与诊断入口

### 11.1. SQL / catalog 入口

确认 table 和 view 状态：

```sql
SELECT relname, relrowsecurity, relforcerowsecurity
FROM pg_class
WHERE relname IN ('accounts', 'safe_accounts');

SELECT polname, polcmd, polpermissive, polroles, polqual, polwithcheck
FROM pg_policy
WHERE polrelid = 'accounts'::regclass;

SELECT reloptions
FROM pg_class
WHERE oid = 'safe_accounts'::regclass;
```

确认函数 leakproof：

```sql
SELECT p.oid::regprocedure, p.proleakproof, p.provolatile, p.procost
FROM pg_proc p
WHERE p.proname = 'your_function_name';
```

如果 predicate 是 operator，继续查 operator 底层函数：

```sql
SELECT o.oid::regoperator, o.oprcode::regprocedure, p.proleakproof
FROM pg_operator o
JOIN pg_proc p ON p.oid = o.oprcode
WHERE o.oprname = '=';
```

### 11.2. EXPLAIN 入口

重点看 qual 的位置：

```text
Index Cond       -> qual 被用作访问条件
Filter           -> scan 后普通过滤
Subquery Scan Filter -> 没有推入 security barrier view 内部
Join Filter      -> 留在 join 层
One-Time Filter  -> pseudoconstant gating
```

同一个 SQL 可以通过开关和改写定位原因：

```text
去掉 security_barrier view option
  -> 如果 qual 下推，说明 barrier 是关键限制
替换成 leakproof operator
  -> 如果 qual 下推，说明 leaky 函数是关键限制
关闭某类 join method
  -> 如果 qual 位置不变，说明不是 join method cost 问题
SET ROLE / SET row_security
  -> 如果计划变化，说明 RLS 环境依赖生效
```

### 11.3. gdb / 源码断点

建议断点：

```text
rewriteHandler.c:fireRIRrules
rewriteHandler.c:ApplyRetrieveRule
rowsecurity.c:get_row_security_policies
rowsecurity.c:add_security_quals
optimizer/plan/initsplan.c:distribute_qual_to_rels
optimizer/util/restrictinfo.c:make_restrictinfo
optimizer/util/restrictinfo.c:restriction_is_securely_promotable
optimizer/path/allpaths.c:qual_is_pushdown_safe
optimizer/util/clauses.c:contain_leaked_vars
```

断点观察对象：

```text
rte->securityQuals
rte->security_barrier
parse->hasRowSecurity
rinfo->security_level
rinfo->leakproof
rel->baserestrict_min_security
safetyInfo.unsafeLeaky
```

### 11.4. 哪些状态看不到

`EXPLAIN` 看不到被拒绝的 path candidate，也看不到 `security_level` 数值。你只能从 qual 位置推断一部分结果。

`pg_stat_*` 基本看不到 RLS policy 组合和 leakproof 判断。它只能告诉你 query 变慢、执行次数、IO 或 CPU 变化，不能证明原因。

`pg_policy` 能看到 catalog 表达式，但看不到 rewrite 后按当前 role、command、checkAsUser、view expansion 和 SubLink 递归处理后的最终 `securityQuals`。

`proleakproof` 能看到函数承诺，但看不到这个承诺是否真实可靠。

## 12. 常见误区

误区一：把 RLS 当成 executor 末端过滤器。

RLS 的 `USING` qual 在 rewrite 阶段进入 `RTE.securityQuals`，planner 会据此限制 qual 移动和 path 生成。它不是 executor 最后才追加的黑盒 filter。

误区二：认为 security barrier view 一定物化。

security barrier 限制的是 leaky qual pushdown 和执行顺序，不等价于强制 materialize。planner 仍可能在合法范围内优化 subquery。

误区三：看到没用索引就调 cost 参数。

如果 predicate 的 `security_level` 高于安全 qual 且不是 leakproof，它可能根本不能提前作为 index qual。调 `random_page_cost` 不会让非法 path 变合法。

误区四：把 leakproof 当成 immutable。

`IMMUTABLE` 说明结果是否只依赖输入；`LEAKPROOF` 说明不会通过错误或副作用泄漏输入值。两者解决的问题不同。

误区五：认为 table owner 当前 bypass RLS 就不需要 plan invalidation。

`RLS_NONE_ENV` 明确表示当前环境 bypass，但环境改变后可能需要 RLS。`hasRowSecurity` 记录的是依赖，不只是当前是否追加 qual。

误区六：只看 `pg_policy` 判断最终执行顺序。

policy 还要经过 role、command、permissive/restrictive 合并、SubLink rewrite、`security_level` 包装、leakproof 判断和 path 合法性检查。catalog 只是起点。

## 13. 课堂实验

### 实验 1：security barrier view 阻止 leaky qual 下推

目标：观察同一个外层 predicate 在普通 view 与 security barrier view 下的位置差异。

步骤：

```sql
CREATE TABLE sb_t(id int primary key, tenant_id int, secret text);
INSERT INTO sb_t SELECT g, g % 2, md5(g::text) FROM generate_series(1,1000) g;
CREATE INDEX ON sb_t(secret);

CREATE VIEW v_plain AS SELECT * FROM sb_t WHERE tenant_id = 1;
CREATE VIEW v_barrier WITH (security_barrier) AS SELECT * FROM sb_t WHERE tenant_id = 1;

EXPLAIN (VERBOSE, COSTS OFF)
SELECT * FROM v_plain WHERE secret = md5('10');

EXPLAIN (VERBOSE, COSTS OFF)
SELECT * FROM v_barrier WHERE secret = md5('10');
```

继续替换 predicate 为自定义非 leakproof 函数，或查询 equality operator 的 `proleakproof`，观察 `Filter` / `Index Cond` / `Subquery Scan` 位置变化。

源码回看：在 `set_subquery_pathlist()` 观察 `safetyInfo.unsafeLeaky`，在 `qual_is_pushdown_safe()` 观察是否因 `contain_leaked_vars()` 返回 unsafe。

### 实验 2：RLS default deny 与 `hasRowSecurity`

目标：区分“没有 policy”与“没有 RLS”。

步骤：

```sql
CREATE TABLE rls_t(id int, owner_name text);
INSERT INTO rls_t VALUES (1, current_user), (2, 'other');
ALTER TABLE rls_t ENABLE ROW LEVEL SECURITY;

EXPLAIN (VERBOSE, COSTS OFF) SELECT * FROM rls_t;
SELECT * FROM rls_t;

CREATE POLICY own_rows ON rls_t
  USING (owner_name = current_user);

EXPLAIN (VERBOSE, COSTS OFF) SELECT * FROM rls_t;
SELECT * FROM rls_t;
```

源码回看：在 `add_security_quals()` 看没有 permissive qual 时追加 `false`；在有 policy 后看 permissive qual 如何被 OR 合并。

### 实验 3：断点观察 `security_level` 与 index path

目标：确认“不能作为 index qual”可能是安全边界，不是 cost。

步骤：

```text
1. 给 RLS 表增加可索引列和外层 predicate。
2. 在 make_restrictinfo() 打断点，记录 rinfo->security_level 和 rinfo->leakproof。
3. 在 restriction_is_securely_promotable() 打断点，记录 rel->baserestrict_min_security。
4. 对比 leakproof operator 与非 leakproof function 包装后的 predicate。
```

观察结论：如果 `restriction_is_securely_promotable()` 返回 false，对应 qual 不能提前用于某些 path。此时 EXPLAIN 中缺少 index cond 是合法性裁剪的结果。

## 14. 讨论题

1. 为什么 RLS policy 要在 rewrite 阶段变成 `securityQuals`，而不是 executor 扫描 tuple 后再调用 policy hook？

2. `RLS_NONE_ENV` 为什么必须让 query 标记 `hasRowSecurity`？如果 prepared statement 在 `SET ROLE` 后复用旧计划，会错在哪里？

3. permissive policy 用 OR 合并、restrictive policy 用 AND 叠加，这对 default deny 语义有什么影响？

4. security barrier view 为什么只阻止 leaky qual 下推，而不是禁止所有 qual 下推？这个设计保留了哪些优化空间？

5. `provolatile = immutable` 与 `proleakproof = true` 分别承诺什么？为什么一个 immutable 函数仍可能不是 leakproof？

6. 如果 `EXPLAIN` 显示 predicate 留在 `Subquery Scan Filter`，你如何区分原因是 security barrier、window/DISTINCT 限制、volatile 限制还是 cost 选择？

7. EC 派生 join qual 为什么也要考虑 `security_level`？如果只限制原始 qual 下推，会留下什么泄漏通道？

8. 扩展通过 RLS hook 注入 policy 时，为什么 restrictive hook policy 仍要排序？这对错误可复现性和诊断有什么帮助？

## 15. 本节小结

本节的核心链路是：catalog 和 relcache 保存 RLS policy；rewrite 把 view 与 RLS 安全边界写入 `RTE.securityQuals`、`rte->security_barrier`、WCO 和 `Query.hasRowSecurity`；planner 再把这些表达式包装成带 `security_level` 与 `leakproof` 的 `RestrictInfo`；path 生成只能在安全合法的搜索空间内考虑下推、index qual、subquery pushdown 和 EC 派生。

核心状态边界是：`pg_policy` 和 `pg_proc.proleakproof` 是 catalog 承诺；`RTE.securityQuals` 是 rewrite 后 query tree 状态；`RestrictInfo.security_level/leakproof` 是 planner-local 状态；最终 plan 只体现裁剪结果，不保存完整的被拒绝候选空间。

ownership 和 cleanup 上，policy catalog 由事务和 invalidation 管理；rewrite copy 出来的 qual 和 planner-local 对象由当前 query memory context 管理；ERROR 不需要手工释放每个表达式。长期正确性依赖 relcache/syscache/plancache invalidation，尤其是 role 和 `row_security` GUC 变化。

错误路径上，没有 permissive policy 会 default deny；`RLS_NONE_ENV` 当前不加 qual 但仍记录环境依赖；policy SubLink 需要额外 rewrite 和锁；leaky qual 在 security barrier 下只能留在上层；错误标记 leakproof 会破坏安全承诺。

可观测性上，`EXPLAIN` 能看到 qual 最终位置，却看不到 `security_level` 和被拒绝 path；catalog 能看到 policy 和函数属性，却看不到当前 rewrite 后的最终 qual；真正定位需要把 SQL 现象、catalog 查询和源码断点连起来。

可迁移规律：优化器的搜索空间不是从 SQL 文本自然展开的全集，而是先被语义和安全边界裁剪后的合法集合。任何“为什么没有下推 / 为什么没用索引”的诊断，都必须先问候选 path 是否被允许存在，再讨论它的 cost 是否更低。
