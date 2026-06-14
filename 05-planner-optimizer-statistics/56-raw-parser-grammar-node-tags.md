# PostgreSQL Raw Parser、Grammar 与 Node Tags
## 课程定位
前置知识：已经知道 PostgreSQL 一条 SQL 会经过 parse、analyze、rewrite、planner 和 executor，也知道 05 目录前面的课程从 `Query` 进入 planner 开始讲起。
本节唯一主问题：
```text
raw parser 如何从 SQL 文本生成 raw parse tree，gram.y、NodeTag 和 location 如何服务后续错误定位？
```
核心矛盾：raw parser 必须在不知道 catalog、权限、类型推断和 GUC 执行结果的前提下，把自由文本压缩成足够结构化的 Node tree；但这棵树又必须保留足够的源码位置和节点身份，让后续 parse analysis、utility command、错误报告和诊断能把问题重新指回用户输入。
学完后应能判断：一个错误位置来自 scanner、grammar action、parse analysis 还是后续模块；一个 raw parse node 的 `NodeTag` 只是运行期类型身份，不是 SQL 语义；一个 `location` 是 byte offset，不是字符列号，也不是生命周期独立的 source span。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e`。
## 1. 本节在总主线中的位置
05 目录前 55 节从 planner 入口、统计、选择率、Path 搜索、createplan 和诊断流程讲起。
那条主线的隐含前提是：planner 已经拿到经过 parse analysis 和 rewrite 的 `Query`。
本节向前补一段边界：
```text
SQL text
  -> scanner
  -> bison grammar
  -> raw parse tree: List<RawStmt>
  -> parse analysis
  -> Query
  -> rewrite
  -> planner
```
这里不讲 parse analysis 如何查 catalog、解析类型、展开 `*`、检查权限。
本节只讲 raw parser 阶段：
```text
characters -> tokens -> grammar reductions -> raw Node tree
```
这个阶段的输出是“语法树”，不是“语义树”。
例如 `SELECT a FROM t` 在 raw parse tree 中可以有 `ColumnRef`、`RangeVar`、`ResTarget`。
但 `a` 是哪一列，`t` 在哪个 schema，`a + 1` 应该调用哪个 operator，这些都不在本节阶段解决。
这条边界对优化器很重要。
如果 raw parser 过早访问 catalog，多语句字符串会出现时间错位。
`gram.y` 文件头的注释明确提醒：不要在 grammar action 中访问数据库，也不要依赖可变状态。
典型反例是：
```sql
SET search_path = other_schema;
SELECT * FROM t;
```
整个字符串在执行第一条 `SET` 前就会被 raw parser 解析。
如果 grammar action 依赖当前 `search_path`，第二条语句看到的就是错误时间点的状态。
所以本节的主线是：
```text
尽早结构化
  vs
推迟语义解释
```
`NodeTag` 和 `location` 正是在这条边界上工作。
`NodeTag` 让一棵 backend-local 的 C struct tree 能被通用 walker、copy、out/read 和调试工具识别。
`location` 让后续阶段即使已经离开 grammar action，也能把错误落回原 SQL 文本。
但二者都不是完整语义。
它们只是把 raw parse tree 接给后续阶段时必须携带的最小可追踪元数据。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
raw_parser() 为一段 SQL 文本创建 scanner，把 token 流交给 gram.y 生成的 bison parser；grammar action 用 makeNode() 构造带 NodeTag 的 raw nodes，并把 token byte offset 写入 location；顶层 stmtmulti 把每条非空语句包装成 RawStmt，最后返回 List<RawStmt> 给 parse analysis 或调用者。
```
这句话里有四个边界。
第一，scanner 只负责词法切分、关键字识别、字面量扫描和 token 起始位置。
第二，`parser.c` 中的 `base_yylex()` 是 scanner 和 grammar 之间的过滤层。
第三，`gram.y` 的 action 创建 raw parse nodes，但不做需要 catalog 的语义解释。
第四，`RawStmt` 包装 statement 级别的位置和长度，内部节点保存更细的 token 起点。
核心 tension 可以写成：
```text
SQL 文本允许空白、注释、多语句、复杂关键字和局部错误
  vs
后续模块需要一棵 typed、可遍历、可复制、可报错定位的 raw tree
```
PostgreSQL 的选择不是在 scanner 中生成完整 AST。
也不是在 parse analysis 阶段重新解析文本。
它把 token 到 tree 的结构化工作集中在 bison grammar。
同时用 `NodeTag` 保证节点运行期类型，用 `ParseLoc` 保存位置。
这样后续模块可以只消费结构，而不必理解 flex/bison 的内部状态。
这也带来代价。
grammar action 很多，raw node 类型很多，`location` 粒度并不总是完整 span。
新增语法时，维护者不仅要让语法通过，还要选择正确节点类型、正确字段、正确位置和正确错误位置。
如果只改语法规则，不维护 `NodeTag` 生成支持和 location 传播，错误会从“能执行”退化为“难诊断”。
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/parser/parser.h` | `RawParseMode` 与 `raw_parser()` 对外边界。 |
| 2 | `src/backend/parser/parser.c` | `raw_parser()` 主入口、scanner 初始化、`base_yylex()` token 过滤。 |
| 3 | `src/backend/parser/scan.l` | flex scanner、token 起始位置、`scanner_errposition()`、scanner cleanup。 |
| 4 | `src/backend/parser/gram.y` | bison grammar、`parse_toplevel`、`stmtmulti`、grammar action 和 `makeRawStmt()`。 |
| 5 | `src/include/nodes/nodes.h` | `NodeTag`、`Node`、`makeNode()`、`ParseLoc` 的基础约定。 |
| 6 | `src/include/nodes/parsenodes.h` | `RawStmt` 和大量 raw parse node 的字段语义。 |
| 7 | `src/backend/nodes/makefuncs.c` | 常用 node 构造 helper，避免 grammar action 手写重复字段。 |
| 8 | `src/backend/nodes/copyfuncs.c` | 节点复制规则，说明 Node tree 如何在阶段间被复制。 |
| 9 | `src/backend/nodes/outfuncs.c` | `nodeToString()` 调试输出以及 location 是否输出的开关。 |
| 10 | `src/backend/nodes/readfuncs.c` | `nodeRead()` 对 location 的默认丢弃和恢复开关。 |
| 11 | `src/tools/pgindent/typedefs.list` | `NodeTag`、`RawStmt` 等 typedef 对格式化和源码风格的影响。 |
阅读顺序不要从 `gram.y` 第一行一路读到最后。
更好的顺序是先抓主入口：
```text
raw_parser()
  -> scanner_init()
  -> parser_init()
  -> base_yyparse()
  -> scanner_finish()
```
然后再看输出节点：
```text
parse_toplevel
  -> stmtmulti
  -> makeRawStmt()
  -> RawStmt.stmt / stmt_location / stmt_len
```
最后看错误位置：
```text
scan.l: SET_YYLLOC()
  -> gram.y: @n / YYLLOC_DEFAULT
  -> node.location
  -> parser_errposition()
  -> scanner_errposition()
  -> errposition()
```
本节基于当前本地 master 源码。
稳定语义是 raw parser 不访问 catalog、输出 raw tree、location 是原文本偏移。
当前实现细节是 flex/bison、`base_yylex()` lookahead filter、以及节点支持代码由 `gen_node_support.pl` 生成或辅助维护。
如果未来文件移动，应优先保持前一段语义边界，再更新具体路径。
## 4. 关键数据结构与状态
### SQL text
`raw_parser(const char *str, RawParseMode mode)` 的输入是一个 C string。
scanner 会复制一份 scan buffer，并在末尾加 flex 需要的两个结束字符。
这意味着 token 的位置以 scanner 的 `scanbuf` 为参照。
语义上它仍然对应原始 SQL 文本的 byte offset。
不要把 raw parser 的 `str` 理解成一条 SQL。
默认模式下它可以是 semicolon 分隔的多语句字符串。
空语句会被丢弃。
最后一条语句的 `stmt_len` 可以是 0，表示直到字符串末尾。
### `RawParseMode`
`RawParseMode` 决定顶层 grammar 入口接受什么形态。
默认值 `RAW_PARSE_DEFAULT` 解析多条 SQL command，返回 `List<RawStmt>`。
PL/pgSQL 或类型名解析可以注入一个 synthetic lookahead token。
这不是另开一套 parser。
`raw_parser()` 通过 `yyextra.lookahead_token` 把 `MODE_TYPE_NAME`、`MODE_PLPGSQL_EXPR`、`MODE_PLPGSQL_ASSIGNn` 交给同一个 bison parser。
这样 grammar 可以在 `parse_toplevel` 中选择不同入口：
```text
stmtmulti
MODE_TYPE_NAME Typename
MODE_PLPGSQL_EXPR PLpgSQL_Expr
MODE_PLPGSQL_ASSIGNn PLAssignStmt
```
这个设计保留一套 token 和 grammar 基础设施。
代价是 `parser.c` 和 `gram.y` 必须共同维护 mode token 数组和对应 grammar 分支。
### scanner token state
`scan.l` 负责把 byte stream 切成 token。
关键状态包括：
| 状态 | 语义 |
| --- | --- |
| `scanbuf` | scanner 持有的输入文本副本。 |
| `scanbuflen` | 原文本长度，不含两个 flex 结束字符。 |
| `yylloc` | 当前 token 起始 byte offset。 |
| literal buffer | 字符串、quoted identifier 等扫描过程中的临时缓冲。 |
| keyword tables | `ScanKeywords` 与 `ScanKeywordTokens`。 |
`scan.l` 中的 `SET_YYLLOC()` 本质是：
```c
*(yylloc) = yytext - yyextra->scanbuf;
```
所以 raw parser 的 location 单位是 byte offset。
用户最终看到的 cursor position 需要转换成字符位置。
这个转换发生在 `scanner_errposition()`：
```text
byte offset -> pg_mbstrlen_with_len(scanbuf, location) + 1 -> errposition()
```
这也是为什么 multibyte encoding 下不能把 `location + 1` 当成用户列号。
### `base_yylex()` filter
`base_yylex()` 位于 `parser.c`，在 flex scanner 和 bison parser 之间。
它解决两个问题。
第一个问题是 SQL grammar 有些地方需要多 token lookahead。
PostgreSQL 希望 bison grammar 保持 LALR(1)，所以在 filter 层把若干组合 token 压成单 token。
例如 `NOT`、`NULLS`、`WITH`、`FORMAT` 等 token 会触发 lookahead 逻辑。
第二个问题是 Unicode identifier 和 string 的归一化。
`UIDENT`、`USCONST` 相关序列可以在 filter 层转换成普通 `IDENT`、`SCONST`。
这样主 grammar 不需要为所有 Unicode 变体复制 production。
这个 filter 是性能和复杂度折中。
如果直接在 `scan.l` 中识别多词 token，就要处理两个词之间的注释和空白，还可能重新引入 scanner backtrack。
如果完全交给 grammar，就会扩大 grammar 分支。
### `NodeTag` 与 `Node`
`nodes.h` 中约定每个 node 的第一个字段是 `NodeTag type`。
基础结构是：
```c
typedef struct Node
{
    NodeTag type;
} Node;
```
`makeNode(_type_)` 会调用 `newNode(sizeof(_type_), T_##_type_)`。
`newNode()` 使用 `palloc0()` 分配并写入 tag。
这个约定让 C 语言里的不同 struct 可以被当成 `Node *` 遍历、检查和输出。
但它不是面向对象继承。
它是“首字段布局约定 + enum tag + cast 宏”的组合。
因此 `NodeTag` 回答的是：
```text
这块内存按哪个 node struct 解释？
```
它不回答：
```text
这个 ColumnRef 绑定到哪个 catalog attribute？
这个 A_Const 应该被推断成 int4 还是 numeric？
这个 SelectStmt 是否有权限访问目标 relation？
```
这些是 parse analysis 或更后阶段的问题。
### `RawStmt`
`parsenodes.h` 中的 `RawStmt` 是 statement wrapper。
核心字段是：
```c
typedef struct RawStmt
{
    NodeTag  type;
    Node    *stmt;
    ParseLoc stmt_location;
    ParseLoc stmt_len;
} RawStmt;
```
`stmt` 指向真正的 raw statement node，例如 `SelectStmt`、`InsertStmt`、`CreateStmt`。
`stmt_location` 是该语句在输入字符串中的起始位置。
`stmt_len` 是 byte length。
`stmt_len = 0` 表示到字符串末尾。
这个 wrapper 的作用不是给 SQL 增加语义。
它是多语句字符串的切片信息。
后续日志、query jumble、错误报告和 utility 处理经常需要知道“当前 statement 在原始 query string 的哪一段”。
### `ParseLoc`
`ParseLoc` 只是 typedef 到 `int`。
但这个 typedef 对 node support generator 有意义。
`nodes.h` 注释说明 `-1` 表示 unknown。
它的语义需要结合三个上下文：
```text
字段名是否是 location
该 node 是否来自原始用户 token
当前调用者是否仍持有原始 query string
```
单独看一个 `location = 5` 没有意义。
只有知道它对应哪个 SQL text，才能变成用户可读位置。
### raw parse nodes
raw parse tree 中常见节点包括：
| Node | 阶段语义 |
| --- | --- |
| `SelectStmt` | 未分析的 SELECT 结构，包含 target、from、where、group、sort、limit 等 raw 字段。 |
| `ResTarget` | target list 或 DML target 的原始目标项。 |
| `ColumnRef` | 未解析的名字链，如 `a`、`t.a`、`schema.t.a`。 |
| `A_Const` | 未完成类型推断的字面量。 |
| `A_Expr` | 未解析 operator 的表达式。 |
| `RangeVar` | 未解析 schema search path 的 relation-like 名字。 |
| `TypeName` | 未解析或部分解析的类型名语法。 |
这些节点的共同特征是“不依赖 catalog 当前状态”。
`RangeVar` 里可以保存 schema/name/relpersistence/location。
但它不保存 relation OID。
`ColumnRef` 保存字段名 list。
但它不保存 varno/varattno。
`A_Const` 保存 literal 形态。
但它不承担完整类型选择。
这就是 raw tree 和 analyzed `Query` 的边界。
### node support files
`copyfuncs.c`、`outfuncs.c`、`readfuncs.c` 是理解 Node tree 生命周期的辅助入口。
`copyObject()` 需要知道每个 node 字段如何复制。
`nodeToString()` 需要知道每个 node 如何调试输出。
`nodeRead()` 需要从 textual node representation 恢复节点。
`readfuncs.c` 顶部注释提醒：location 字段写出主要用于调试；读取时默认通常把 location 设为 `-1`，因为新的调用者未必拥有同一段 query text。
这说明 location 的 ownership 不在 node 本身。
node 只保存 offset。
原始文本和调用上下文才决定这个 offset 是否仍可解释。
## 5. 主流程源码 walkthrough
本节主流程从 `raw_parser()` 进入。
入口位于 `src/backend/parser/parser.c`。
伪调用链如下：
```text
raw_parser(str, mode)
  -> scanner_init(str, &yyextra.core_yy_extra, &ScanKeywords, ScanKeywordTokens)
  -> initialize yyextra lookahead according to RawParseMode
  -> parser_init(&yyextra)
  -> base_yyparse(yyscanner)
     -> base_yylex()
        -> core_yylex()
     -> grammar reductions in gram.y
     -> pg_yyget_extra(yyscanner)->parsetree = ...
  -> scanner_finish(yyscanner)
  -> return yyextra.parsetree
```
### 第一步：创建 scanner
`scanner_init()` 复制输入字符串，建立 flex scan buffer。
scan buffer 末尾追加两个 `YY_END_OF_BUFFER_CHAR`。
它还初始化 literal buffer、keyword tables、encoding 相关状态和若干 scanner options。
状态变化是：
```text
caller-owned const char *str
  -> scanner-owned scanbuf
  -> token location references scanbuf byte offsets
```
这里没有创建 raw parse tree。
这里只是让 scanner 可以稳定地扫描一段文本。
### 第二步：处理 parse mode
默认模式下，`yyextra.have_lookahead = false`。
非默认模式下，`raw_parser()` 从 `mode_token[]` 取一个 synthetic token。
例如 `RAW_PARSE_TYPE_NAME` 对应 `MODE_TYPE_NAME`。
然后设置：
```text
have_lookahead = true
lookahead_token = MODE_...
lookahead_yylloc = 0
```
这意味着 bison parser 看到的第一个 token 不是用户文本中的 token，而是 mode marker。
grammar 的 `parse_toplevel` 因此可以分发到不同分支。
这个设计避免为 PL/pgSQL expression 或 type name 单独维护一套 parser。
### 第三步：初始化 bison parser
`parser_init(&yyextra)` 初始化 bison parser 所需的额外状态。
`yyextra` 同时连接 scanner extra state 和 grammar 输出 `parsetree`。
这一层的关键是：bison parser 不直接返回一棵树。
它通过 `pg_yyget_extra(yyscanner)->parsetree` 写结果。
顶层 grammar action 决定这个结果是什么 list。
### 第四步：扫描 token
bison 调用 `base_yylex()` 要 token。
`base_yylex()` 先检查是否已有 lookahead token。
没有则调用 `core_yylex()`。
`core_yylex()` 来自 `scan.l`。
它识别关键字、identifier、operator、numeric literal、quoted string、dollar-quoted string、comment、parameter marker 等。
每个 token 的起始 byte offset 会写到 `llocp`。
注意，注释和空白通常不会作为 grammar token 返回。
但它们仍然影响后续 token 的 offset。
所以错误 cursor 能跳过注释和空白指向真正 token。
### 第五步：lookahead filter
`base_yylex()` 对少数 token 做 lookahead。
它可能读取下一个 token，把组合情况替换为单个 token，也可能把下一个 token 缓存在 `yyextra->lookahead_*`。
状态变化是：
```text
current token consumed
optional next token scanned and cached
possibly return combined token to bison
```
这就是为什么 `base_yylex()` 既要保存 token 值，也要保存 location 和 scan buffer 中暂时被改写的字符。
错误定位也必须考虑这层 filter。
如果某个 Unicode escape 在 filter 中处理失败，错误位置仍要通过 scanner callback 指回原 SQL。
### 第六步：grammar 归约
`gram.y` 的顶层 production 是 `parse_toplevel`。
默认路径是：
```text
parse_toplevel
  -> stmtmulti
```
`stmtmulti` 做两件事。
第一，把非空 top-level statement 包装成 `RawStmt`。
第二，遇到 semicolon 时更新前一条 statement 的 `stmt_len`。
核心逻辑可以压缩成：
```text
stmtmulti ';' toplevel_stmt
  -> updateRawStmtEnd(previous, semicolon_location)
  -> append makeRawStmt(next, next_location)
```
如果输入是：
```sql
SELECT 1; SELECT 2
```
第一条 `RawStmt` 的 `stmt_len` 会在遇到分号时确定。
第二条 `RawStmt` 的 `stmt_len` 可能保持 0，表示到字符串末尾。
如果输入有连续分号：
```sql
SELECT 1;; SELECT 2;
```
空 statement 会被丢弃。
这避免给空语句硬造一个不可靠 location。
### 第七步：grammar action 构造节点
每个具体 production 的 action 负责创建对应 raw node。
典型形式是：
```c
ResTarget *n = makeNode(ResTarget);
n->name = ...;
n->val = ...;
n->location = @1;
```
实际源码字段因节点而异；例如 `SelectStmt` 的 statement 级位置不在自身字段中，而在外层 `RawStmt.stmt_location`。
更常见的是 helper：
```text
makeIntConst()
makeStringConst()
makeTypeCast()
makeColumnRef()
makeAConst()
makeAndExpr()
```
这些 helper 集中处理常见字段和 location。
例如 `makeRawStmt()` 会设置：
```text
stmt
stmt_location
stmt_len = 0
```
`makeColumnRef()` 会把字段选择和下标 indirection 整理成 `ColumnRef` 或 `A_Indirection`。
`makeNullAConst()`、`makeIntConst()`、`makeFloatConst()` 等会创建 `A_Const` 并写入 location。
这里仍然没有 catalog lookup。
字段名仍是字符串。
操作符仍是名字。
类型名仍是 raw type syntax。
### 第八步：location 默认规则
`gram.y` 覆盖了 bison 的默认 location 规则。
PostgreSQL 只跟踪非终结符的起始位置。
规则可以概括为：
```text
Current = first RHS location that is not -1
empty production -> -1
```
所以 `@1`、`@2`、`@$` 都是 byte offset。
当 leading RHS 是可空 production 时，默认 location 会跳过 `-1` 找第一个真实 token。
这个选择对错误定位很重要。
它避免空 production 把整个节点的位置污染成“未知”。
但它也意味着大多数 node 只有起始位置，没有结束位置。
如果需要 span，需要额外字段保存。
`RawStmt.stmt_len` 是 statement-level span。
某些 list-like JSON 或 array 结构也可能有 start/end 字段。
不能假设所有 node 都有完整 source range。
### 第九步：保存结果并 cleanup
`base_yyparse()` 成功后，顶层 action 已经把结果写到 `yyextra.parsetree`。
`raw_parser()` 调用 `scanner_finish(yyscanner)`。
`scanner_finish()` 不销毁所有 flex 控制结构。
源码注释说明，小块控制存储留给 parsing context 回收更便宜。
它只在 scan buffer 或 literal buffer 大到超过阈值时显式 `pfree()`。
之后 `raw_parser()` 返回 `yyextra.parsetree`。
如果 `base_yyparse()` 返回错误，`raw_parser()` 返回 `NIL`。
但普通语法错误通常通过 `ereport(ERROR)` longjmp，不会以普通返回值继续执行。
这个返回分支主要是 bison error code 层面的兜底。
### 第十步：交给 parse analysis
调用者拿到 `List<RawStmt>` 后，后续阶段逐条处理。
parse analysis 会把 raw tree 变成 `Query` 或 utility `Query`。
对 optimizable statement，转换很复杂。
对 utility statement，raw statement 通常进入 `Query.utilityStmt`，大量工作推迟到执行 utility command 时。
这解释了为什么 `RawStmt` 既是 parser 输出，又不是 planner 输入。
planner 通常不直接消费 `RawStmt`。
planner 消费的是 analyze/rewrite 后的 `Query`。
## 6. 生命周期 / ownership / cleanup
### 谁创建
raw parse tree 由当前 backend 在当前 memory context 中创建。
通常它发生在处理客户端 query message 或内部调用 raw parser 的路径中。
节点由 grammar action、helper 函数和 `makeNode()` 创建。
`makeNode()` 使用 `palloc0()`。
所以字段默认是 0 或 NULL。
这对 grammar action 很重要。
很多 bool、list、location 字段如果不显式设置，就会保留默认值。
但 location 的 unknown 约定是 `-1`，不是 0。
所以需要 unknown location 的字段必须显式写 `-1`，不能依赖 `palloc0()`。
### 谁持有
raw parse tree 是 backend-local memory。
它不在 shared memory 中。
其他 backend 不能访问这棵树。
指针不能跨进程保存。
调用者持有返回的 list 指针，并在同一 backend 的后续 parse/analyze/rewrite/planner 路径中消费。
对简单查询协议的一段多语句字符串，`List<RawStmt>` 的每个元素对应一个可执行 statement。
对 prepared statement 或 extended protocol，raw parsing 和 parse analysis 的调用位置不同，但 raw tree 仍然是 backend-local。
### 谁释放
raw parse tree 的释放通常依赖 MemoryContext 生命周期。
不是每个 node 手动 `pfree()`。
如果解析或分析过程中 ERROR，PostgreSQL 的错误恢复会回滚到合适的 error context，并 reset 短生命周期 memory context。
这就是为什么 `gram.y` 让 bison 使用 `palloc/pfree`。
源码中定义：
```text
YYMALLOC = palloc
YYFREE = pfree
```
这样 bison 栈或临时分配不会绕过 PostgreSQL 的 memory context 体系。
### scanner cleanup
scanner 自己有 scan buffer 和 literal buffer。
`scanner_finish()` 会释放“大到值得释放”的 buffer。
小控制结构可以留给 memory context 回收。
这里体现的是成本取舍：
```text
显式释放所有小块
  vs
依赖 context reset 批量回收，减少 hot path 操作
```
raw parsing 是每条 SQL 都会走的热路径。
小块内存的逐一释放不一定划算。
### location ownership
`location` 不是对 SQL 文本的 owning reference。
它只是 offset。
要解释它，调用者必须同时持有同一段 query string。
因此，当 node tree 被写出再读回时，`readfuncs.c` 默认会把 location 字段读成 `-1`。
除非显式要求 restore location fields。
这是正确的。
如果一个节点来自缓存、序列化或测试 fixture，旧 offset 指向的新文本可能完全不同。
强行保留 location 会制造误导性错误 cursor。
### NodeTag lifecycle
`NodeTag` 在 node 分配时写入。
后续 walker、copy、equal、out/read、debugger 都依赖这个 tag。
它不是可变状态。
一般不应该在节点创建后随意 `NodeSetTag()`。
少数基础设施可能需要特殊处理，但普通 grammar action 应该使用 `makeNode(Type)`。
如果新增节点类型，需要让 `NodeTag`、node support generator、copy/equal/out/read/query jumble 等支持保持一致。
否则问题可能不是编译失败，而是某条调试输出、copy 或 equal 路径在运行期失败。
### ERROR / abort 时谁兜底
raw parser 阶段的错误大多通过 `ereport(ERROR)` 跳出。
MemoryContext 负责回收已经创建的 node 和 scanner buffer。
error context callback 负责补充 cursor position。
没有 ResourceOwner 参与 raw parse tree cleanup。
因为这里没有 buffer pin、lock、smgr reference、catcache reference 或文件句柄这类外部资源。
如果某个 grammar action 引入外部资源，那通常就是设计错误。
raw parser 应保持纯 backend-local memory work。
## 7. 正确性机制层次
raw parser 的正确性不是由一个机制保证的。
它由几层边界叠加。
| 机制 | 主要保证 | 不要误解为 |
| --- | --- | --- |
| scanner rules | token 切分、字面量合法性、token 起始位置 | SQL 语义正确 |
| `base_yylex()` | 多词 token 和 Unicode token 的单 token 化 | 完整语法分析 |
| bison grammar | token 序列符合 SQL grammar，生成 raw tree | 名字解析、类型解析、权限检查 |
| `NodeTag` | C struct 运行期类型身份 | 节点语义已完成 |
| `location` | 能回到原文本某个 byte offset | 完整 source span 或字符列号 |
| MemoryContext | ERROR 后批量回收 parser 内存 | 外部资源 cleanup |
### 语法正确性
语法正确性来自 grammar。
例如 `SELECT FROM` 会在 grammar 层失败，因为 token 序列无法归约成合法 statement。
此时错误位置通常指向第一个让 bison 无法继续的 token。
这不是 PostgreSQL “知道你缺少 target list”。
这是 parser 在某个状态下没有合法 shift/reduce 动作。
错误消息可以更友好，但底层机制仍是 grammar 状态机。
### 语义延迟
`SELECT not_exist FROM t` 可以通过 raw parser。
它会形成 `ColumnRef`。
错误要到 parse analysis 查 range table 和 column namespace 时才能报出。
这类错误仍可能使用 raw node 上的 `location`。
所以 raw parser 不负责发现该错误，却负责给后续阶段留下足够定位信息。
这是本节最重要的不变量：
```text
raw parser 不做语义判断，但要为未来语义错误保留位置证据。
```
### NodeTag 正确性
通用 node 处理依赖 `NodeTag`。
例如 `IsA(node, SelectStmt)`、`castNode(SelectStmt, node)`、`copyObject()`、`nodeToString()` 都以 tag 为入口。
如果一个节点内存被错误 tag 标记，后续会按错误 struct layout 解释字段。
这类 bug 可能表现为 assert、crash、调试输出错误或 planner/analyzer 读到荒谬字段。
所以 grammar action 中 `makeNode()` 不是语法糖。
它是 node tree runtime type safety 的入口。
### location 正确性
location 的正确性有两个层次。
第一层是 scanner 把 token 起点记录成 byte offset。
第二层是 grammar action 把合适 token 的 offset 赋给合适 node 或错误报告。
例如一个 `TypeCast` 的 location 应该指向 cast syntax 的相关位置，还是 inner expression 的起点？
这取决于后续错误希望指向什么。
新增语法时不能机械使用 `@1`。
应问：
```text
后续如果这个节点报错，用户最应该看到哪个 token？
```
如果是整个 clause 的错误，可能使用 clause keyword。
如果是某个 option 的错误，应该保存 option token 的 location。
`gram.y` 中大量 `parser_errposition(@n)` 就是在 grammar 层选择错误归属。
### 不涉及的正确性机制
本节不涉及 MVCC visibility。
不涉及 heavyweight lock。
不涉及 LWLock。
不涉及 WAL。
不涉及 shared invalidation。
原因很简单：raw parse tree 是当前 backend 的短生命周期内存结构。
它不访问 shared catalog state。
它不修改持久数据。
它不需要跨 backend 协调。
如果你发现 raw parser 路径需要锁或 WAL，通常说明读错了边界，已经进入 parse analysis、DDL execution 或 catalog 修改阶段。
## 8. 错误路径 / 异常路径 / fallback
### 词法错误
词法错误来自 `scan.l`。
典型例子：
```sql
SELECT 'abc;
```
scanner 会在 EOF 状态发现 unterminated quoted string。
错误通过 `scanner_yyerror()` 报出。
cursor position 使用当前 `yylloc`。
如果错误发生在字符串内部处理函数中，scanner 可能安装 error context callback。
`setup_scanner_errposition_callback()` 把位置和 scanner 放入 `error_context_stack`。
被调用函数即使不知道 scanner，也能在 ERROR 时补充 cursor。
取消时调用 `cancel_scanner_errposition_callback()`。
这是 raw parser 错误定位的一个重要 fallback。
### 语法错误
语法错误来自 bison parser。
`gram.y` 中 `base_yyerror()` 调用 `parser_yyerror(msg)`。
`parser_yyerror` 最终映射到 `scanner_yyerror(msg, yyscanner)`。
所以 grammar 语法错误仍使用 scanner 的 `yylloc` 和 scan buffer。
典型现象：
```sql
SELECT FROM t;
```
错误 cursor 通常指向 `FROM`。
含义是：parser 在需要 expression 或 target 项的位置遇到了 `FROM`。
它不是 semantic error。
### grammar action 主动报错
有些语法组合形式上能归约，但 PostgreSQL 在 action 中拒绝。
例如某些重复选项、不允许的 clause 组合、非法窗口定义等。
这类路径通常使用：
```text
ereport(ERROR,
        (errcode(ERRCODE_SYNTAX_ERROR),
         errmsg(...),
         parser_errposition(@n)));
```
此时 `@n` 是 action 作者选择的错误 token。
这和 bison 自动 syntax error 不同。
自动错误指向无法继续的位置。
主动错误应指向违反规则的具体 clause 或 option。
### Unicode escape 错误
Unicode identifier/string 的处理部分在 `parser.c` filter 中完成。
当 `UESCAPE` 不是简单字符串、escape 字符非法、surrogate pair 非法时，错误要指回 Unicode escape 的源位置。
源码使用 scanner errposition callback 包住可能报错的转换逻辑。
这说明错误定位不只发生在 `scan.l` 或 `gram.y`。
`parser.c` 作为中间 filter 也会产生 raw parser 阶段错误。
### 多语句长度更新异常
`stmtmulti` 中 `updateRawStmtEnd()` 有一个防重复更新保护：
```text
if stmt_len > 0, return
```
这是为了处理类似：
```sql
select foo ;; select bar
```
同一个前置 statement 可能在连续 semicolon 场景下多次成为“上一条 statement”。
长度一旦确定，不应被后续空语句再次覆盖。
这个细节影响日志和 query text 切片。
如果没有这个保护，连续分号可能把第一条语句的长度扩大到错误位置。
### OOM 和 parser 内存异常
raw parser 分配失败会走 PostgreSQL ERROR 路径。
因为 bison 使用 `palloc`，MemoryContext 能统一清理已分配对象。
调用者看到的是 ERROR，而不是部分 raw tree。
这点对扩展或内部调用者很重要。
不要假设 `raw_parser()` 在失败时总会返回 `NIL`。
大多数错误不会以普通返回值返回。
### `location = -1` fallback
当位置未知时，PostgreSQL 使用 `-1`。
`scanner_errposition(location, yyscanner)` 看到负数会 no-op。
后续错误仍会报出，但没有 cursor。
这比报一个错误 cursor 更安全。
错误的位置如果不可信，就不应该伪造。
新增节点时，如果没有可靠 token，可以使用 `-1`。
但如果后续错误需要定位，就应该重新考虑 grammar action 是否遗漏了 location。
## 9. 成本、资源与跨模块传播
raw parsing 是每条 SQL 的前置成本。
它通常不是 OLTP 慢查询的主瓶颈。
但在短查询、高 QPS、复杂 SQL、批量 prepared statement、ORM 生成超长 SQL 的场景中，它会变成可见 CPU 成本。
### CPU 成本
成本随输入文本长度近似线性增长。
scanner 必须读取每个字符。
多字节编码、Unicode escape、长字符串 literal、长 identifier 会增加处理成本。
grammar 成本随 token 数和 grammar 状态转移增长。
一般不是指数型搜索。
真正指数或组合爆炸的部分在 planner join search，不在 raw parser。
### allocation 成本
每个 raw node 都是 `palloc0()`。
复杂 query 会创建大量小对象和 list cells。
MemoryContext 批量释放降低 cleanup 成本。
但创建期间仍有 allocation 和 cache locality 成本。
这就是为什么 helper、list 构造和 parser action 的简洁性对 hot path 有意义。
### location 成本
记录 location 本身只是写 int。
真正有成本的是错误发生时从 byte offset 转成字符位置。
`scanner_errposition()` 调用 `pg_mbstrlen_with_len(scanbuf, location)`。
这需要从文本起点扫描到错误 byte offset。
因为错误路径不是 hot path，这个设计合理。
正常解析不为每个 token 计算字符列号。
### `base_yylex()` filter 成本
filter 增加了一层函数调用和少量 lookahead。
但它避免 scanner backtrack 和 grammar 复杂化。
源码注释明确说：直接在 `scan.l` 中识别多词 token 会受注释影响，可能带来更高性能成本。
因此这里的取舍是：
```text
少量 token filter 开销
  vs
scanner backtrack / grammar 分支膨胀
```
### 跨模块传播：parse analysis
parse analysis 消费 raw tree。
它把 `ColumnRef` 变成 `Var`、把 `A_Const` 变成 `Const`、把 `A_Expr` 解析成具体 operator 或 function。
这个阶段大量错误仍使用 raw node 的 location。
例如不存在列、参数类型不一致、函数不存在，都可能指回 raw syntax 中的 token。
所以 raw parser 的 location 质量直接影响 semantic error 质量。
### 跨模块传播：utility commands
utility statement 通常不会完全转成 optimizable `Query`。
raw node 会进入 `Query.utilityStmt`。
执行 utility command 时再处理更多细节。
这意味着某些 raw node 生命周期比普通 SELECT 的 raw node 更直接影响执行阶段。
但它仍然是 backend-local tree，不是 shared state。
### 跨模块传播：query jumble 和日志
`RawStmt.stmt_location` 与 `stmt_len` 可帮助从多语句 query string 中取出当前 statement。
`queryjumble` 还会跟踪常量位置，用于 query normalization。
但 `RawStmt` 自身标注了 `no_query_jumble`。
这符合语义：statement wrapper 是文本切片容器，不是 query 语义的一部分。
### 跨模块传播：debug support
`nodeToString()`、`debug_print_parse`、gdb 调试、回归测试 fixture 都可能观察 raw 或 analyzed node tree。
`outfuncs.c` 默认不一定输出真实 location。
源码有 `write_location_fields` 开关。
这样可以避免调试输出或测试结果被不稳定位置噪声污染。
需要位置时再显式打开。
### 不涉及后台进程
本节没有后台进程参与状态推进。
postmaster、checkpointer、walwriter、autovacuum、startup process 都不参与 raw parse tree 的创建和维护。
如果一个现象需要后台进程解释，它通常已经不在 raw parser 范围内。
## 10. 观测与诊断入口
raw parser 的可观测性比 planner 低。
没有 `pg_stat_raw_parser`。
没有 wait event。
没有 WAL 或 buffer 指标。
但仍有几类入口可以把现象拉回源码。
### 入口 1：错误 cursor
最直接现象是错误消息中的 `LINE` 和 caret。
例如：
```sql
SELECT FROM t;
```
观测重点不是错误文案本身。
而是 caret 指向哪个 token。
如果 caret 指向第一个无法继续解析的 token，多半是 bison syntax error。
如果 caret 指向某个 option 或 clause，多半是 grammar action 主动 `parser_errposition(@n)`。
如果是 semantic error，例如列不存在，则错误来自 parse analysis，但位置来自 raw node。
### 入口 2：`debug_print_parse`
可以在测试环境启用：
```sql
SET client_min_messages = log;
SET debug_print_parse = on;
SELECT a + 1 FROM t WHERE b = 10;
```
这个输出通常是分析后的 parse tree，不是纯 raw tree。
它仍能帮助你看到 `location` 是否传播到后续 node。
但不要把它当成 raw parser 的完整 dump。
纯 raw tree 更适合通过断点或临时日志观察。
### 入口 3：gdb 断点
源码跟读建议断点：
```text
raw_parser
base_yylex
scanner_errposition
base_yyerror
makeRawStmt
transformTopLevelStmt
```
在 `raw_parser()` 观察 `str` 和 `mode`。
在 `base_yylex()` 观察 token、`llocp` 和 lookahead。
在 `makeRawStmt()` 观察 statement node type、`stmt_location`。
在 `scanner_errposition()` 观察 byte offset 如何变成用户位置。
### 入口 4：临时 `nodeToString()`
开发环境可以在 parser 调用后临时打印 raw tree。
注意两点。
第一，不能把这种日志提交到产品代码。
第二，默认 location 输出可能被屏蔽。
需要看 location 时，要确认调用路径是否打开 write location fields。
`outfuncs.c` 的开关就是为了控制这一点。
### 入口 5：源码搜索
常用命令：
```bash
cd /home/nail/postgres
rg -n "raw_parser|base_yylex|scanner_errposition|makeRawStmt" src/backend/parser src/include/parser
rg -n "typedef struct RawStmt|typedef enum NodeTag|ParseLoc" src/include/nodes
rg -n "parser_errposition\\(|YYLLOC_DEFAULT|stmtmulti" src/backend/parser/gram.y
```
这些命令可以把入口、位置机制和 grammar wrapper 连接起来。
### 能看到什么
你能直接看到：
| 入口 | 能看到 |
| --- | --- |
| SQL 错误消息 | 用户字符位置和错误 token 附近文本。 |
| gdb | byte offset、token、node tag、raw node 字段。 |
| debug 日志 | 后续阶段的 parse tree 或 selected dump。 |
| 源码搜索 | 哪些 grammar action 显式选择 error position。 |
### 看不到什么
你不能直接从 SQL 层看到：
| 不可见状态 | 原因 |
| --- | --- |
| 每个 token 的 raw byte offset | scanner 内部状态。 |
| bison state stack | parser 内部生成代码状态。 |
| 所有 raw nodes | 默认没有 SQL 可见 dump。 |
| lookahead cache | `parser.c` 局部状态。 |
| MemoryContext 中 parser allocation 明细 | 需要内存调试或断点。 |
所以 raw parser 诊断常常需要源码断点。
仅靠 `pg_stat_*` 无法定位这一层问题。
## 11. 常见误区
### 误区 1：raw parser 会解析表名和列名
不会。
raw parser 只形成 `RangeVar`、`ColumnRef` 等 raw nodes。
schema search path、relation OID、column attno 都是后续 parse analysis 的工作。
如果错误是“relation does not exist”或“column does not exist”，它不是 raw grammar 拒绝了 SQL。
它是后续语义阶段使用 raw node 的 location 报错。
### 误区 2：`NodeTag` 等于 SQL 节点语义
`NodeTag` 只说明 C struct 类型。
`T_ColumnRef` 不说明这个名字绑定到哪一列。
`T_A_Const` 不说明最终类型。
`T_SelectStmt` 不说明 query 是否合法访问 catalog。
真实语义来自字段组合、生命周期阶段和消费者。
### 误区 3：`location` 是字符列号
不是。
raw location 是 byte offset。
用户看到的列号在错误报告时通过 encoding-aware 函数转换。
对 multibyte 字符串，`location + 1` 可能不是 caret 列号。
### 误区 4：每个 node 都有完整 start/end span
多数 node 只有起始 location。
`RawStmt` 有 statement length。
部分特殊节点有 start/end 字段。
但不能假设任意 node 都能映射到完整 SQL 子串。
### 误区 5：`stmt_len = 0` 表示空语句
不是。
在 `RawStmt` 中，`stmt_len = 0` 表示 statement 到字符串末尾。
空语句在 `stmtmulti` 中被丢弃，不会形成 `RawStmt`。
### 误区 6：语法错误和语义错误都来自 `gram.y`
语法错误可能来自 scanner、parser filter 或 grammar。
语义错误通常来自 parse analysis 或 utility execution。
同样的 caret 不代表同样的模块。
诊断时先问：
```text
错误发生时，是否已经有 raw tree？
是否已经进入 transform 阶段？
错误位置来自 token yylloc 还是 node.location？
```
### 误区 7：可以在 grammar action 中查 catalog
不应该。
多语句字符串会在执行前整体 raw parse。
catalog 访问和 GUC 依赖必须推迟到 parse analysis 或 execution 的正确时间点。
这是 `gram.y` 文件头反复强调的边界。
## 12. 课堂实验
### 实验 1：观察多语句 `RawStmt` 切片
目标：验证 `stmtmulti` 如何创建 `RawStmt` 并更新 `stmt_len`。
步骤：
```bash
cd /home/nail/postgres
gdb --args ./tmp_install/usr/local/pgsql/bin/postgres -D /path/to/data
```
在 backend 中打断点：
```text
break raw_parser
break makeRawStmt
break updateRawStmtEnd
```
执行：
```sql
SELECT 1;; SELECT 2;
```
观察：
```text
第一条 SELECT 创建 RawStmt
第一个分号更新第一条 stmt_len
第二个分号不创建空 RawStmt
第二条 SELECT 创建新的 RawStmt
末尾分号更新第二条 stmt_len
```
源码回扣：
```text
gram.y: stmtmulti
gram.y: makeRawStmt()
gram.y: updateRawStmtEnd()
```
结论：`RawStmt` 是 statement slice，不是每个分号一个节点。
### 实验 2：区分语法错误和语义错误的位置来源
目标：比较 bison syntax error 与 parse analysis error。
执行：
```sql
SELECT FROM t;
SELECT not_exist FROM t;
```
第一条通常在 raw parser 阶段失败。
第二条如果 `t` 存在但列不存在，会进入 parse analysis 后失败。
断点建议：
```text
break scanner_yyerror
break base_yyerror
break scanner_errposition
break transformColumnRef
```
观察：
```text
语法错误使用当前 yylloc
语义错误使用 ColumnRef.location
两者最终都可能调用 scanner_errposition 或 parser_errposition 类机制
```
结论：错误 cursor 相同形式，不代表错误来自同一阶段。
### 实验 3：验证 multibyte location 是 byte offset
目标：确认 raw location 与用户列号的差异。
执行含多字节字符的 SQL，例如：
```sql
SELECT '中文' + FROM t;
```
在 `scanner_errposition()` 断点观察：
```text
location 是 byte offset
pg_mbstrlen_with_len(scanbuf, location) + 1 才是 errposition
```
结论：不要在调试中用 `location + 1` 推断用户看到的 caret 列。
### 实验 4：新增一个临时 grammar action 日志
目标：理解 location 选择如何影响后续诊断。
只在本地实验分支临时修改，不提交：
```text
在 makeRawStmt() 或某个 expression helper 中 elog(DEBUG1, ...)
打印 nodeTag(stmt)、stmt_location、stmt_len 或 expression location。
```
执行一组 SQL：
```sql
SELECT a + 1 FROM t WHERE b = 10;
SELECT (a + 1)::int FROM t;
```
观察不同节点 location 指向哪个 token。
结论：location 是 grammar action 的设计选择，不是自动完整 source map。
## 13. 讨论题
1. 为什么 `gram.y` 不应该访问 catalog？请用多语句字符串中的 `SET` 或 DDL 例子解释时间错位。
2. `RawStmt.stmt_len = 0` 为什么表示“到字符串末尾”，而不是“长度为 0”？这对日志和 query jumble 有什么影响？
3. 如果新增一个 raw parse node，只加 struct 不更新 node support，会在哪些路径上出问题？
4. 为什么 `location` 使用 byte offset，而不是 scanner 阶段直接计算字符列号？
5. 一个 `ColumnRef.location` 能不能证明列已经绑定成功？为什么？
6. 语法错误、grammar action 主动错误、parse analysis 错误在错误位置来源上有什么不同？
7. 为什么 `readfuncs.c` 默认读入 location 时会把它变成 `-1`？保留旧 location 有什么风险？
8. `base_yylex()` filter 为什么比在 scanner 中识别所有多词 token 更适合 PostgreSQL 当前 grammar？
## 14. 本节小结
raw parser 的主链路是：
```text
raw_parser()
  -> scanner_init()
  -> base_yyparse()
  -> base_yylex()/core_yylex()
  -> gram.y actions
  -> RawStmt list
  -> scanner_finish()
```
核心状态是三类。
第一类是 scanner 状态：`scanbuf`、token、`yylloc`、literal buffer。
第二类是 grammar 输出：`RawStmt`、`SelectStmt`、`ColumnRef`、`A_Const` 等 raw nodes。
第三类是跨阶段元数据：`NodeTag` 和 `ParseLoc location`。
ownership 很清楚。
raw parse tree 是 backend-local memory。
它由当前解析路径创建，由调用者在同一 backend 内消费，由 MemoryContext 在正常结束或 ERROR 后回收。
它不在 shared memory 中，不需要 lock、WAL、invalidation 或后台进程推进。
正确性边界也很清楚。
raw parser 保证 token 序列能归约成 raw syntax tree。
它不保证名字存在、类型可解析、权限满足或执行计划合理。
但它必须给后续阶段留下足够好的 `NodeTag` 和 `location`，否则后续阶段即使能发现错误，也难以把错误指回用户 SQL。
异常路径主要有四类。
scanner 报词法错误。
bison 报 syntax error。
grammar action 主动用 `parser_errposition(@n)` 报更精确的语法限制。
后续 parse analysis 用 raw node location 报语义错误。
诊断时不能只看 caret，要判断当前错误发生在哪个阶段。
成本上，raw parser 主要消耗 CPU、tokenization、grammar reduction 和大量小对象 allocation。
它随 SQL 文本长度、token 数和 raw node 数增长。
它通常不是复杂查询的最大成本，但在短查询高频场景中会变得可见。
本节可迁移的系统规律是：
```text
早期阶段可以推迟语义判断，但不能推迟可追踪性。
```
PostgreSQL raw parser 不查 catalog、不做类型解析、不拿锁。
它只负责把文本压缩成结构，并把运行期类型和错误定位证据带到后续阶段。
这个设计让 parse analysis、rewriter、planner 能在各自正确的时间点做语义工作，同时还能把错误回指到原始 SQL。
下一步如果继续向后追，应进入 parse analysis：`ColumnRef` 如何变成 `Var`，`A_Const` 如何变成 `Const`，以及 raw `SelectStmt` 如何变成 planner 能消费的 `Query`。
