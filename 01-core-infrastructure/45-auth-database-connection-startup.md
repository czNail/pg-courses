# PostgreSQL auth database connection startup：认证、database attach 与 startup GUC
## 课程定位
前置知识：已经理解 postmaster fork backend、`PGPROC` / `ProcArray`、MemoryContext / ResourceOwner，以及 FE/BE 协议里 startup packet 的基本形态。
本节唯一主问题：
```text
认证、database attach、role/session 初始化和 GUC startup packet
如何形成一个普通 backend 的 backend-local 初始状态？
```
核心矛盾：
```text
客户端在 startup packet 里一次性给出 user、database、options 和若干 GUC
  vs
backend 不能信任这些字符串，也不能在权限、database 和 cleanup 边界未建立前直接应用它们
```
本节要建立的判断框架：
- startup packet 只表达客户端意图，不等于认证完成。
- HBA / auth 只决定认证方法并验证身份，不等于 database 已 attach。
- role/session 初始化把认证后的 user name 变成 PostgreSQL role OID 和 effective user。
- database attach 把 catalog identity、database lock、`MyDatabaseId`、`MyProc->databaseId` 和物理路径连接起来。
- startup packet GUC 必须等 role 权限和 database 默认状态明确后才进入 GUC 系统。
- 连接启动失败要按阶段定位，不能统一叫“认证失败”。
学完后应能独立判断：
- 一个连接失败发生在 packet parse、HBA match、认证方法、role lookup、database attach、GUC 设置还是 login trigger。
- 哪些状态只在 `Port` 里，哪些状态进入 backend-local globals，哪些状态发布到 shared memory。
- 哪些状态能从日志、`pg_stat_activity`、`pg_hba_file_rules`、协议消息或 gdb 看到。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e`。
## 1. 本节在总主线中的位置
上一节待补主题是 backend fork/exec bootstrap。
本节从普通 backend 子进程已经被 postmaster 创建、并拿到 client socket 开始。
此时还没有：
- 读完 startup packet。
- 知道 HBA match 到哪一行。
- 验证客户端认证身份。
- 查出 PostgreSQL role OID。
- attach 到目标 database。
- 设置 `MyDatabaseId` 和 `MyProc->databaseId`。
- 应用 startup packet 里的 GUC。
- 向客户端发送完整的 startup response。
本节主线：
```text
BackendMain()
  -> BackendInitialize()
     -> pq_init()
     -> ProcessSSLStartup()
     -> ProcessStartupPacket()
  -> InitProcess()
  -> PostgresMain(dbname, username)
     -> BaseInit()
     -> InitPostgres()
        -> InitProcessPhase2()
        -> StartTransactionCommand()
        -> PerformAuthentication()
        -> InitializeSessionUserId()
        -> database lookup / lock / recheck / attach
        -> CheckMyDatabase()
        -> process_startup_options()
        -> process_settings()
        -> InitializeSearchPath()
        -> InitializeClientEncoding()
        -> InitializeSession()
        -> CommitTransactionCommand()
     -> BeginReportingGUCOptions()
     -> BackendKeyData
     -> EventTriggerOnLogin()
     -> ReadyForQuery in main loop
```
下一节待补主题会继续讲连接完成后异步 signal、config reload、query cancel 和 session 退出。
本节只覆盖普通客户端 backend。
physical walsender、background worker、standalone backend、parallel worker 只在边界处说明。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
BackendInitialize() 在尚未依赖 shared memory cleanup 的阶段读取 startup packet，
把 user、database、options 和普通 GUC name/value 暂存在 Port；
PostgresMain() 建立 backend runtime 后进入 InitPostgres()；
InitPostgres() 在初始化事务中完成 HBA/auth、role lookup、database lookup/lock/recheck、
MyDatabaseId/MyProc->databaseId 发布、database 默认 GUC 和 startup GUC 应用；
PostgresMain() 最后报告 GUC、发送 cancel key，并进入可接受查询的主循环。
```
这条链路里有四个不同身份：
- startup packet `user`：客户端声明的字符串。
- authentication identity：认证方法确认的外部身份，存入 `MyClientConnectionInfo.authn_id`。
- authenticated role：`InitializeSessionUserId()` 查出的 PostgreSQL role OID，存入 `AuthenticatedUserId`。
- effective user：权限检查使用的 `CurrentUserId`，初始等于 session user，后续可被 `SET ROLE` 或 `SECURITY DEFINER` 改变。
也有三个 database 阶段：
- startup packet database：客户端给出的 database name，空时默认等于 user。
- catalog database：`pg_database` 中查到并加锁 recheck 的 OID 和属性。
- attached database：`MyDatabaseId` 已设置，`MyProc->databaseId` 已发布，database path、encoding、locale 和 ACL 已初始化。
startup packet GUC 的关键边界：
- parse 时只进入 `Port.cmdline_options` 或 `Port.guc_options`。
- 认证后才知道 `am_superuser`。
- database check 后才有 server/client encoding 默认值。
- `process_startup_options()` 才真正调用 `SetConfigOption(..., PGC_S_CLIENT)`。
所以连接启动不是一个单一“认证函数”。
它是把不可信网络输入逐步收敛成可清理、可观测、可被其他 backend 正确理解的 runtime state。
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/tcop/backend_startup.c` | `BackendMain()`、`BackendInitialize()`、`ProcessStartupPacket()`。 |
| 2 | `src/include/libpq/libpq-be.h` | `Port`、`ClientConnectionInfo`、startup packet 暂存状态。 |
| 3 | `src/backend/libpq/pqcomm.c` | `pq_init()`、socket buffer、`socket_close()`、低层 FE/BE I/O。 |
| 4 | `src/backend/libpq/hba.c` | `load_hba()`、`check_hba()`、`hba_getauthmethod()`。 |
| 5 | `src/backend/libpq/auth.c` | `ClientAuthentication()`、认证方法分发、`set_authn_id()`、`auth_failed()`。 |
| 6 | `src/backend/tcop/postgres.c` | `PostgresMain()`、信号初始化、`InitPostgres()` 调用、startup response。 |
| 7 | `src/backend/utils/init/postinit.c` | `InitPostgres()`、`PerformAuthentication()`、database attach、startup GUC。 |
| 8 | `src/backend/utils/init/miscinit.c` | user id globals、`InitializeSessionUserId()`、`InitializeSystemUser()`。 |
| 9 | `src/backend/utils/misc/guc.c` | `SetConfigOption()`、GUC source/context 权限、`BeginReportingGUCOptions()`。 |
| 10 | `src/backend/tcop/dest.c` | `ReadyForQuery()` 的客户端完成边界。 |
推荐从入口按时间读：
```text
BackendMain()
  -> BackendInitialize()
  -> ProcessStartupPacket()
  -> PostgresMain()
  -> InitPostgres()
  -> PerformAuthentication()
  -> ClientAuthentication()
  -> InitializeSessionUserId()
  -> database attach block
  -> CheckMyDatabase()
  -> process_startup_options()
  -> BeginReportingGUCOptions()
```
不要先横向读所有认证方法。
本节不是认证算法课，而是连接启动状态机课。
## 4. 关键数据结构与状态
### 4.1 `Port`
`Port` 是当前 backend 的连接对象。
它是 backend-local 状态，通过 `MyProcPort` 访问。
普通 backend 中 `Port` 和其指向的 startup 字符串在 `TopMemoryContext`。
关键字段：
- `sock`：客户端 socket。
- `proto`：FE/BE 协议版本。
- `laddr` / `raddr`：本地和远端地址，供 HBA、日志和 socket 选项使用。
- `remote_host` / `remote_port`：日志与状态显示用文本。
- `remote_hostname` / `remote_hostname_resolv`：HBA hostname 规则相关的 DNS 状态。
- `database_name`：startup packet 解析出的 database。
- `user_name`：startup packet 解析出的 user。
- `cmdline_options`：startup packet 的 `options` 字段。
- `guc_options`：普通 startup name/value，按 name/value 交替保存在 List 中。
- `application_name`：连接授权日志使用的副本。
- `hba`：runtime HBA match 后选中的 `HbaLine`。
`Port` 不是 shared memory。
其它 backend 不能直接看到它。
它保存的是连接启动输入和认证中间状态，不是完整 session truth。
### 4.2 `ClientConnectionInfo`
`ClientConnectionInfo` 保存认证阶段产生的身份：
- `authn_id`：认证方法确认的外部身份字符串。
- `auth_method`：产生该身份的 HBA authentication method。
它的语义不是 PostgreSQL role。
例如：
- password / MD5 / SCRAM 通常把数据库用户名作为认证身份。
- peer / ident / cert 可能产生 OS user、ident user 或证书字段。
- trust 可能不设置 `authn_id`。
`InitializeSystemUser()` 只在 `authn_id` 非 NULL 时执行。
它把 `SYSTEM_USER` 的状态构造成：
```text
auth_method:authn_id
```
这和 `SESSION_USER`、`CURRENT_USER` 是不同边界。
### 4.3 HBA runtime 状态
HBA 文件由 `hba.c` 解析成 `HbaLine` list。
普通 fork 模式下，backend 继承 postmaster 已解析的 HBA 数据。
`ClientAuthentication()` 中调用：
```text
hba_getauthmethod(port)
  -> check_hba(port)
```
runtime match 维度包括：
- local / host / hostssl / hostnossl / hostgss / hostnogss。
- SSL state 和 GSS encryption state。
- client IP、CIDR、hostname、samehost、samenet。
- database name 和 replication 特殊值。
- user name、role OID、role membership。
如果没有匹配行，`check_hba()` 会创建临时 `HbaLine`：
```text
auth_method = uaImplicitReject
```
显式 `reject` 和 implicit reject 的错误消息不同。
这对定位 HBA 问题很重要。
### 4.4 user id globals
`miscinit.c` 中的核心 user 状态：
- `AuthenticatedUserId`：连接启动确定，正常不变。
- `SessionUserId`：初始等于 authenticated user，可被 `SET SESSION AUTHORIZATION` 改变。
- `OuterUserId`：outer level 当前角色，可被 `SET ROLE` 改变。
- `CurrentUserId`：当前 effective user，普通权限检查使用它。
- `SystemUser`：认证方法和认证身份组合出的 SQL `SYSTEM_USER`。
这些字段不能单独解释语义。
例如 `CurrentUserId` 的含义还依赖：
- `SetRoleIsActive`
- `SecurityRestrictionContext`
- 是否在 SECURITY DEFINER 或本地 user id change 中
启动时 `InitializeSessionUserId()` 通过 GUC 设置 `session_authorization`。
GUC assign hook 再设置 `SessionUserId`、`OuterUserId`、`CurrentUserId` 和内部 `is_superuser` GUC。
所以 role/session 初始化和 GUC 系统在启动阶段已经耦合。
### 4.5 database attach 状态
database attach 的核心状态：
- `MyDatabaseId`：backend-local 当前 database OID。
- `MyDatabaseTableSpace`：database 默认 tablespace。
- `MyDatabaseHasLoginEventTriggers`：是否有 login event trigger。
- `MyProc->databaseId`：发布到 shared memory 的 database membership。
- database path：后续 storage path 解析所需的本地状态。
- database encoding / locale：影响 `server_encoding`、`client_encoding`、LC_CTYPE 和 collation。
关键不变量：
```text
只有在 pg_database row 被 lock 和 recheck 后，才能设置 MyDatabaseId。
只有 MyDatabaseId 可靠后，才能发布 MyProc->databaseId。
```
这服务于 concurrent `DROP DATABASE` 和 `CountOtherDBBackends()` 的正确性。
### 4.6 startup GUC 状态
startup packet 中的 GUC 有两种入口：
```text
options = "-c name=value"
普通 name/value = search_path, application_name, statement_timeout, ...
```
前者进入：
```text
Port.cmdline_options
```
后者进入：
```text
Port.guc_options
```
真正应用在：
```text
process_startup_options()
  -> process_postgres_switches()
  -> SetConfigOption(name, value, gucctx, PGC_S_CLIENT)
```
其中：
```text
gucctx = am_superuser ? PGC_SU_BACKEND : PGC_BACKEND
```
所以 startup GUC 是“连接开始收到、认证和 database 初始化后应用”的状态。
它不是 packet parse 的副作用。
## 5. 主流程源码 walkthrough
### 5.1 `BackendMain()`：普通 backend 入口
`backend_startup.c` 的 `BackendMain()` 顺序很短：
```text
BackendInitialize(MyClientSocket, canAcceptConnections)
InitProcess()
MemoryContextSwitchTo(TopMemoryContext)
PostgresMain(MyProcPort->database_name, MyProcPort->user_name)
```
这个顺序定义了连接启动的第一条 cleanup 边界。
`BackendInitialize()` 不依赖 shared memory。
`InitProcess()` 之后 backend 才进入需要 shared-state cleanup 的世界。
`PostgresMain()` 不再解析 packet，只接收 `Port` 中已经保存的 database/user 字符串。
### 5.2 `BackendInitialize()`：建立 `Port` 并读 startup packet
`BackendInitialize()` 先做：
```text
ReserveExternalFD()
ClientAuthInProgress = true
MyProcPort = pq_init(client_sock)
whereToSendOutput = DestRemote
```
`pq_init()`：
- 分配 `Port`。
- 复制 socket 地址。
- 设置 TCP_NODELAY / SO_KEEPALIVE。
- 分配 libpq send buffer。
- 注册 `on_proc_exit(socket_close, 0)`。
- 将 socket 设成 nonblocking。
- 创建 frontend/backend wait set。
然后 `BackendInitialize()` 设置 startup packet 阶段专用 timeout：
```text
RegisterTimeout(STARTUP_PACKET_TIMEOUT, StartupPacketTimeoutHandler)
enable_timeout_after(STARTUP_PACKET_TIMEOUT, AuthenticationTimeout * 1000)
```
这一段刻意发生在 shared memory 前。
如果客户端迟迟不发 packet，backend 可以快速退出，不需要清理 ProcArray、lock、transaction 或 database state。
源码还会检查：
```text
check_on_shmem_exit_lists_are_empty()
```
这保护“startup packet 阶段没有 shared memory cleanup 责任”的不变量。
### 5.3 SSL / GSS negotiation 和特殊 packet
`ProcessStartupPacket()` 可能先读到特殊 request code：
- `CANCEL_REQUEST_CODE`
- `NEGOTIATE_SSL_CODE`
- `NEGOTIATE_GSS_CODE`
SSL / GSS negotiation 完成后会释放当前 buffer 并 `goto retry`，等待真正的 startup packet。
所以 SSLRequest 不是普通 startup packet。
CancelRequest 也不是认证失败。
它处理完后不会进入 `PostgresMain()`。
握手成功后还会检查：
```text
pq_buffer_remaining_data() > 0
```
如果发现握手前已有未加密数据，报 protocol violation。
这是防止 startup encryption negotiation 被混入明文数据的边界。
### 5.4 `ProcessStartupPacket()`：解析客户端意图
协议 3 startup packet 是 name/value 序列。
`ProcessStartupPacket()` 将字段分流：
- `database` -> `port->database_name`
- `user` -> `port->user_name`
- `options` -> `port->cmdline_options`
- `replication` -> `am_walsender` / `am_db_walsender`
- `_pq.*` -> protocol-level option negotiation
- 其它字段 -> `port->guc_options`
必须有 user。
database 为空时默认等于 user。
database 和 user 会截断到 `NAMEDATALEN - 1`，使后续 catalog lookup 遵守 PostgreSQL name 语义。
普通 physical walsender 会把 `port->database_name[0]` 设为空。
logical replication 使用 `replication=database`，仍需要 attach 到具体 database。
### 5.5 连接接收状态检查
startup packet 成功后，`BackendInitialize()` 根据 postmaster 传来的 `CAC_state` 决定是否拒绝：
- `CAC_STARTUP`
- `CAC_NOTHOTSTANDBY`
- `CAC_SHUTDOWN`
- `CAC_RECOVERY`
- `CAC_TOOMANY`
这些拒绝发生在认证前。
原因是系统状态已经不允许连接时，不应继续消耗认证交换成本。
这也是 `pg_isready` 类工具能在不完成认证的情况下得到服务器状态的基础。
如果 `ProcessStartupPacket()` 返回非 OK，`BackendInitialize()` 会 `proc_exit(0)`。
此时 backend 还没有进入 `InitProcess()`。
### 5.6 `PostgresMain()`：普通 backend runtime
`PostgresMain()` 先设置普通 backend 信号处理：
- `SIGHUP` -> config reload。
- `SIGINT` -> query cancel。
- `SIGTERM` -> die。
- `SIGQUIT` -> under postmaster 时 quickdie。
- `SIGUSR1` -> procsignal handler。
- `SIGPIPE` -> ignore。
然后：
```text
BaseInit()
unblock signals
generate MyCancelKey
InitPostgres(dbname, InvalidOid, username, InvalidOid, flags, NULL)
```
cancel key 在 `InitPostgres()` 前生成。
它会在 `InitPostgres()` 的 `ProcSignalInit()` 中发布到共享状态。
客户端要到 `BackendKeyData` 消息时才看到它。
### 5.7 `InitPostgres()`：先进入可管理的 backend 状态
`InitPostgres()` 一开始调用：
```text
InitProcessPhase2()
```
源码注释说：
```text
Once I have done this, I am visible to other backends!
```
随后初始化：
- `pgstat_beinit()` / `pgstat_bestart_initial()`
- `SharedInvalBackendInit(false)`
- `ProcSignalInit(MyCancelKey, MyCancelKeyLength)`
- timeout handlers
- `RelationCacheInitialize()`
- `InitCatalogCache()`
- `InitPlanCache()`
- `EnablePortalManager()`
- `RelationCacheInitializePhase2()`
- `before_shmem_exit(ShutdownPostgres, 0)`
认证前需要这些基础设施，是因为认证和 role/database lookup 可能要访问 catalog、syscache、invalidation、timeout、pgstat 和 procsignal。
例如 password/SCRAM 需要查 role password。
database attach 需要查 `pg_database`。
### 5.8 初始化事务
普通 backend 接着：
```text
SetCurrentStatementStartTimestamp()
StartTransactionCommand()
XactIsoLevel = XACT_READ_COMMITTED
```
这是一笔 startup initialization transaction。
它不是用户显式事务。
后续认证 catalog access、role lookup、database lookup、permission check、`pg_db_role_setting` 都在这笔事务中进行。
如果这之后失败，就需要 transaction abort 和 resource cleanup。
所以 `ShutdownPostgres()` 必须在第一笔事务前注册。
### 5.9 `PerformAuthentication()`：HBA/auth 阶段
普通 multiuser 路径：
```text
PerformAuthentication(MyProcPort)
InitializeSessionUserId(username, useroid, false)
if (MyClientConnectionInfo.authn_id)
  InitializeSystemUser(...)
am_superuser = superuser()
```
`PerformAuthentication()` 做：
- 设置 `ClientAuthInProgress = true`。
- `EXEC_BACKEND` 下可能重新 `load_hba()` / `load_ident()`。
- 记录 authentication start timestamp。
- 用 `STATEMENT_TIMEOUT` infrastructure 设置 `authentication_timeout`。
- `set_ps_display("authentication")`。
- 调用 `ClientAuthentication(port)`。
- 关闭 timeout。
- 记录 auth end timestamp。
- 按配置记录 authorization log。
- `set_ps_display("startup")`。
- `ClientAuthInProgress = false`。
同一个 `STATEMENT_TIMEOUT` handler 在认证阶段会发送 `SIGTERM`，普通 query 阶段才是 `SIGINT` query cancel。
这是生命周期阶段改变 timeout 语义的典型例子。
### 5.10 `ClientAuthentication()`：HBA match 和认证方法
`ClientAuthentication()` 先调用：
```text
hba_getauthmethod(port)
```
`check_hba()` 顺序扫描 parsed HBA lines。
它会先用：
```text
get_role_oid(port->user_name, true)
```
注意 `missing_ok = true`。
HBA 阶段不会因为 role 不存在立即报错。
这样可避免过早泄漏 role 存在性，也让 password/SCRAM 能按认证路径处理。
认证方法分发包括：
- `uaReject` / `uaImplicitReject`
- `uaGSS` / `uaSSPI`
- `uaPeer` / `uaIdent`
- `uaMD5` / `uaSCRAM` / `uaPassword`
- `uaPAM` / `uaBSD` / `uaLDAP`
- `uaCert`
- `uaTrust`
- `uaOAuth`
成功后：
```text
ClientAuthentication_hook(port, status)
sendAuthRequest(port, AUTH_REQ_OK, NULL, 0)
```
`AUTH_REQ_OK` 不必立即 flush。
它可以和后续 startup response 一起发给客户端。
失败后：
```text
auth_failed(port, elevel, status, logdetail)
```
多数失败是 `FATAL`，当前 backend 退出。
### 5.11 `set_authn_id()`：认证身份
认证方法成功后应调用：
```text
set_authn_id(port, id)
```
它把身份复制到 `TopMemoryContext`：
```text
MyClientConnectionInfo.authn_id = MemoryContextStrdup(TopMemoryContext, id)
MyClientConnectionInfo.auth_method = port->hba->auth_method
```
它禁止重复设置。
如果同一连接设置两次 `authn_id`，会 `FATAL`。
但 `trust` 路径可能不调用 `set_authn_id()`。
因此 `SYSTEM_USER` 可能为 NULL。
这不是 role/session 初始化失败，而是认证方法没有可记录的外部身份。
### 5.12 `InitializeSessionUserId()`：role/session 初始化
认证成功后，普通路径调用：
```text
InitializeSessionUserId(username, InvalidOid, false)
```
它先：
```text
AcceptInvalidationMessages()
```
然后通过 syscache 查 `AUTHNAME`。
查不到：
```text
FATAL: role "..." does not exist
```
查到后：
```text
SetAuthenticatedUserId(roleid)
SetConfigOption("session_authorization", rname, PGC_BACKEND, PGC_S_OVERRIDE)
```
`session_authorization` 的 assign hook 会设置 session/outer/current user。
随后检查：
- `rolcanlogin`
- `rolconnlimit`
这说明“密码正确但 role 不能登录”不是认证算法失败。
它是 role/session 初始化失败。
### 5.13 reserved slots 和 replication privilege
role 初始化后，`InitPostgres()` 才能判断：
```text
am_superuser = superuser()
```
然后检查：
- superuser reserved slots。
- `pg_use_reserved_connections` 权限。
- walsender 需要 `REPLICATION` role 属性。
这些检查必须在 role 已确定后做。
physical walsender 不 attach database，会在 replication 权限检查后走旁路：
```text
process_startup_options()
InitializeClientEncoding()
pgstat_bestart_final()
CommitTransactionCommand()
EmitConnectionWarnings()
return
```
普通 SQL backend 继续 database attach。
### 5.14 database lookup、lock、recheck
普通 backend 先用 name 查 `pg_database`：
```text
tuple = GetDatabaseTuple(in_dbname)
dboid = dbform->oid
```
查不到：
```text
FATAL: database "..." does not exist
```
然后拿 database object lock：
```text
LockSharedObject(DatabaseRelationId, dboid, 0, RowExclusiveLock)
```
拿锁后再查：
```text
tuple = GetDatabaseTupleByOid(dboid)
```
recheck：
- tuple 仍存在。
- name 没被 rename。
- database 不处于 invalid 状态。
这个顺序解决与 concurrent `DROP DATABASE` 的边界：
```text
在发布 MyProc->databaseId 之前，backend 持有 database lock。
发布之后，DROP DATABASE 可以通过 ProcArray/PGPROC 看到它。
```
### 5.15 attach：`MyDatabaseId` 与 `MyProc->databaseId`
recheck 成功后：
```text
MyDatabaseId = dboid
MyProc->databaseId = MyDatabaseId
```
`MyDatabaseId` 是 backend-local identity。
`MyProc->databaseId` 是 shared memory 中给其他 backend 看的 membership。
随后调用：
```text
InvalidateCatalogSnapshot()
```
原因是 role/database lookup 期间可能建立过 catalog snapshot。
在设置 `MyDatabaseId` 之前，backend 对 database-local invalidation 的处理边界还不完整。
旧 snapshot 不能跨过 database attach 边界继续使用。
### 5.16 database path、relcache phase 3 和 ACL
attach 后：
```text
fullpath = GetDatabasePath(MyDatabaseId, MyDatabaseTableSpace)
access(fullpath, F_OK)
ValidatePgVersion(fullpath)
SetDatabasePath(fullpath)
RelationCacheInitializePhase3()
initialize_acl()
```
这说明 database attach 不只是 OID。
它还要确认物理目录、PG_VERSION、database-local relcache 和 ACL 框架都可用。
如果 `base/<dboid>` 或 tablespace path 缺失，错误发生在 storage attach 阶段，不是 HBA。
### 5.17 `CheckMyDatabase()`
`CheckMyDatabase(dbname, am_superuser, override)` 做：
- re-fetch `pg_database` row。
- 检查 database name 没变。
- 检查 `datallowconn`。
- 检查 `CONNECT` privilege。
- 检查 database connection limit。
- 设置 database encoding。
- 设置 `server_encoding` internal GUC。
- 设置 `client_encoding` dynamic default。
- 设置 LC_CTYPE。
- 初始化 database collation。
- 检查 collation version 并可能产生 WARNING。
`CONNECT` privilege 检查发生在 database attach 后。
database connection limit 是近似检查，源码承认并发启动下不追求完美公平。
collation version mismatch 是 WARNING。
LC_CTYPE 不兼容是 FATAL。
### 5.18 startup packet GUC 应用
现在 `InitPostgres()` 才调用：
```text
process_startup_options(MyProcPort, am_superuser)
```
它先处理 `Port.cmdline_options`：
```text
pg_split_opts()
process_postgres_switches(ac, av, gucctx, NULL)
```
再处理 `Port.guc_options`：
```text
SetConfigOption(name, value, gucctx, PGC_S_CLIENT)
```
`gucctx` 根据 superuser 状态选择：
```text
PGC_SU_BACKEND or PGC_BACKEND
```
`guc.c` 里的 `set_config_with_handle()` 根据目标 GUC 的 context 判断能否设置。
例如：
- `PGC_POSTMASTER` 参数不能在连接启动时改变。
- `PGC_SU_BACKEND` / `PGC_SUSET` 需要 superuser 或 parameter ACL。
- `PGC_BACKEND` 参数只能在 connection start 设置。
- `PGC_USERSET` 普通用户可设置。
这解释了为什么 startup packet GUC 不能在 `ProcessStartupPacket()` 中直接应用。
当时还没有 role、superuser、parameter ACL 和 database default 边界。
### 5.19 database/role stored settings 和 session 收束
startup packet options 后：
```text
process_settings(MyDatabaseId, GetSessionUserId())
```
它读取 `pg_db_role_setting`，按更具体到更一般的顺序尝试：
- database + role
- role
- database
- global
随后：
```text
PostAuthDelay
InitializeSearchPath()
InitializeClientEncoding()
InitializeSession()
process_session_preload_libraries()
pgstat_bestart_final()
CommitTransactionCommand()
EmitConnectionWarnings()
```
这些步骤必须在 GUC 后。
例如 search path、client encoding 和 session preload libraries 都依赖最终 GUC 状态。
初始化事务提交后，backend 的初始 session state 才稳定。
### 5.20 startup response 与主循环边界
`InitPostgres()` 返回后，`PostgresMain()`：
```text
MemoryContextDelete(PostmasterContext)
SetProcessingMode(NormalProcessing)
BeginReportingGUCOptions()
on_proc_exit(log_disconnections, 0)
pgstat_report_connect(MyDatabaseId)
send BackendKeyData
create MessageContext
create RowDescriptionContext
EventTriggerOnLogin()
enter main loop
```
`BeginReportingGUCOptions()` 把带 `GUC_REPORT` 的变量发送为 `ParameterStatus`。
`BackendKeyData` 发送：
- `MyProcPid`
- `MyCancelKey`
`ReadyForQuery()` 在主循环准备接受命令时发送。
客户端收到 `AuthenticationOk` 只表示认证成功。
客户端收到 `ReadyForQuery` 才表示 startup cycle 完整结束。
## 6. 生命周期 / ownership / cleanup
### 6.1 `Port`、socket 和通信资源
创建者：
```text
pq_init()
```
持有者：
```text
当前 backend，通过 MyProcPort 访问。
```
内存生命周期：
```text
TopMemoryContext，随 backend 进程退出回收。
```
退出 cleanup：
```text
on_proc_exit(socket_close, 0)
```
`socket_close()` 清理 GSS context / credential，调用 `secure_close(MyProcPort)`，并把 `sock` 置为 `PGINVALID_SOCKET`。
它不试图释放所有 `Port` 内存。
进程退出会回收进程本地内存。
### 6.2 startup packet 阶段
`BackendInitialize()` 期间还没有 `InitProcess()`。
所以 timeout、bad packet、cancel request 失败路径主要关闭连接并退出 backend。
这时不应存在 shmem exit cleanup。
`check_on_shmem_exit_lists_are_empty()` 就是在保护这个假设。
一旦进入 `InitProcess()` 和 `InitPostgres()`，backend 已经有 shared memory 可见状态，失败路径必须走更完整的 cleanup。
### 6.3 HBA 和 `PostmasterContext`
普通 fork 模式下，HBA/ident 解析结果在 postmaster 中建立，backend 继承。
`BackendMain()` 暂时不能删除 `PostmasterContext`，因为 `InitPostgres()` 的认证还要使用 HBA data。
`PostgresMain()` 在 `InitPostgres()` 完成后删除：
```text
MemoryContextDelete(PostmasterContext)
```
所以 `Port.hba` 不是长期 session contract。
长期需要的认证身份必须复制到 `MyClientConnectionInfo`。
### 6.4 初始化事务与资源释放
`InitPostgres()` 中 catalog lookup、database lock、syscache ref、snapshot 都在 startup transaction 中。
如果中途失败：
- `FATAL` 退出当前 backend。
- transaction abort 释放锁、snapshot、buffer pin、cache ref。
- `before_shmem_exit(ShutdownPostgres, 0)` 保证退出前 abort 任何活动事务。
普通 startup 失败不会回到同一个 backend 的查询循环。
`PostgresMain()` 的 outer error recovery 主要保护已经进入命令循环后的 query ERROR。
### 6.5 database lock 与 `MyProc->databaseId`
database object lock 只持有到 startup transaction 结束。
它保护 attach 过程中的 `pg_database` row。
长期表示“我正在使用这个 database”的状态是：
```text
MyProc->databaseId
```
这是一种接力：
```text
startup transaction lock protects attach
PGPROC databaseId protects long-lived membership
```
### 6.6 GUC 生命周期
startup packet GUC 经过三层：
- packet buffer：`ProcessStartupPacket()` 临时 buffer。
- `Port` copy：`cmdline_options` / `guc_options`。
- GUC state：`SetConfigOption()` 后的 per-backend GUC 变量、source 和 reset 值。
连接成功后，诊断最终状态应看 GUC 系统：
```sql
SHOW name;
SELECT current_setting('name');
```
不要把 `Port.guc_options` 当作最终配置 truth。
## 7. 正确性机制层次
### 7.1 protocol framing
startup packet 先检查长度、协议版本、terminator 和 name/value 布局。
不合法 packet 不能进入 HBA、auth 或 GUC。
SSL/GSS negotiation 后禁止缓冲区里残留握手前明文数据。
这是协议层正确性和安全边界。
### 7.2 HBA first match
HBA 是顺序 first-match。
诊断时必须问：
```text
第一条匹配当前连接上下文的规则是什么？
```
不能只问：
```text
文件里有没有一条允许规则？
```
当前连接上下文包括 SSL/GSS、client address、database、user、replication 和 DNS 状态。
### 7.3 authentication vs authorization
认证回答：
```text
客户端是否证明了某个身份？
```
role/session 初始化回答：
```text
该 PostgreSQL role 是否存在、可登录、未超过 role 连接限制？
```
database attach 回答：
```text
目标 database 是否存在、未被 drop、可连接、物理目录有效，并且当前 user 有 CONNECT 权限？
```
这些都可能导致连接失败，但源码阶段不同。
### 7.4 lock + publication
database attach 的核心 ordering：
```text
lookup pg_database
lock database object
recheck pg_database
set MyDatabaseId
set MyProc->databaseId
commit startup transaction
```
这个顺序让 concurrent `DROP DATABASE` 要么被 database lock 阻挡，要么能在 ProcArray/PGPROC 中看到已连接 backend。
### 7.5 invalidation boundary
设置 `MyDatabaseId` 后要 `InvalidateCatalogSnapshot()`。
原因是 attach 前的 catalog snapshot 不能跨过 database-local invalidation 边界。
这体现了：
```text
snapshot validity = data contents + invalidation membership + lifecycle point
```
### 7.6 GUC source/context
startup GUC 的 source 是：
```text
PGC_S_CLIENT
```
context 是：
```text
PGC_BACKEND or PGC_SU_BACKEND
```
GUC 系统结合目标参数 context、当前 role、parameter ACL、source 和 action 决定能否设置。
权限判断不是 packet parser 的职责。
## 8. 错误路径 / 异常路径 / fallback
### 8.1 packet parse 失败
一个字节都没读到时，通常直接 fail，避免 health check 或端口探测造成日志噪声。
部分 length word 或不完整 packet 会记录 protocol violation。
长度过小、超过 `MAX_STARTUP_PACKET_LENGTH`、terminator 不在最后，也都会拒绝。
这些错误发生在 HBA 前。
### 8.2 SSL / GSS negotiation 异常
SSLRequest / GSSENCRequest 后如果发现未加密数据已经在 buffer 中，报 FATAL。
这是连接加密协商边界，不是密码认证失败。
### 8.3 no HBA entry 与 explicit reject
没有匹配 HBA 行：
```text
uaImplicitReject
```
错误会说：
```text
no pg_hba.conf entry ...
```
匹配到 reject 行：
```text
uaReject
```
错误会说：
```text
pg_hba.conf rejects connection ...
```
两者都在认证方法真正执行前失败。
### 8.4 password/SASL EOF 或失败
客户端被 challenge 后直接断开时，password 路径可能返回 `STATUS_EOF`。
`auth_failed()` 对 EOF 直接 `proc_exit(0)`，避免日志噪声。
密码错误、SCRAM 失败、外部认证系统拒绝，通常走 `FATAL`。
`logdetail` 只写 server log，不把敏感细节发给客户端。
### 8.5 role/session 失败
认证成功后仍可能失败：
- role 不存在。
- role 没有 `LOGIN`。
- role connection limit 超过。
- reserved connection slots 不允许普通用户占用。
- walsender role 没有 `REPLICATION`。
这些不是 HBA 失败。
它们发生在 `InitializeSessionUserId()` 及其后续权限检查。
### 8.6 database attach 失败
database attach 可能失败于：
- database name 不存在。
- lookup 后被 concurrent drop / rename。
- database invalid。
- database physical directory 缺失。
- `PG_VERSION` 无效。
- `datallowconn = false`。
- 缺少 `CONNECT` privilege。
- database connection limit 超过。
- LC_CTYPE 与 OS 不兼容。
这些错误都发生在认证成功之后。
### 8.7 startup GUC 失败
startup packet 可以请求 GUC。
但可能失败于：
- postmaster-only 参数不能连接时设置。
- backend-only 参数过了 connection start 就不能设置。
- SUSET / SU_BACKEND 参数缺少 superuser 或 parameter ACL。
- check hook 拒绝参数值。
典型错误：
```text
parameter "..." cannot be changed without restarting the server
permission denied to set parameter "..."
```
### 8.8 `EXEC_BACKEND` fallback
在 `EXEC_BACKEND` 构建下，子进程不能完全继承 postmaster 内存。
`PerformAuthentication()` 会按需创建 `PostmasterContext` 并重新：
```text
load_hba()
load_ident()
```
这会增加连接启动成本。
它是构建方式相关的 fallback，不是普通 fork 模式的 hot path。
## 9. 成本、资源与跨模块传播
每个普通连接启动至少消耗：
- process / socket 初始化。
- startup packet parse。
- HBA 顺序扫描。
- auth exchange 和可能的网络 round trip。
- `InitProcess` / `InitProcessPhase2`。
- pgstat、ProcSignal、sinval、relcache、catcache 初始化。
- startup transaction。
- role 和 database syscache lookup。
- database object lock。
- GUC processing。
- ParameterStatus / BackendKeyData / ReadyForQuery protocol output。
HBA 成本随这些因素扩大：
- HBA 行数。
- hostname rule 和 DNS。
- SSL/GSS 分支。
- role membership 展开。
auth 成本随认证方法变化：
- SCRAM/MD5/password hash 和 round trip。
- TLS client certificate。
- GSSAPI。
- PAM/LDAP/OAuth 外部服务。
database attach 成本依赖：
- catalog cache coldness。
- database path / tablespace filesystem latency。
- relcache phase 3。
- locale / collation check。
GUC 成本通常较小，但随 startup GUC 数量、check hook、assign hook 和 `pg_db_role_setting` 增长。
连接启动会传播到相邻模块：
| 模块 | 传播状态 |
| --- | --- |
| ProcArray / PGPROC | backend membership、`databaseId`、cancel key。 |
| pgstat | backend status、connect reporting。 |
| lock manager | startup database object lock。 |
| relcache / syscache | role/database/catalog lookup。 |
| GUC | startup packet、database/role settings、reported GUC。 |
| libpq protocol | auth request、ParameterStatus、BackendKeyData、ReadyForQuery。 |
短连接风暴通常不是单点问题。
它把 process creation、auth、catalog/cache、GUC 和 protocol 固定成本全部放大。
## 10. 观测与诊断入口
### 10.1 server log
连接日志可区分阶段：
```text
connection received: host=... port=...
connection authenticated: identity="..." method=... (file:line)
connection authenticated: user="..." method=... (file:line)
connection authorized: user=... database=... application_name=...
```
只有 `received` 没有 `authenticated`，优先查 packet、SSL/GSS、HBA、auth timeout。
已经 `authenticated` 但没有进入可用 session，优先查 role、database attach、startup GUC、login trigger。
### 10.2 `pg_hba_file_rules`
`pg_hba_file_rules` 能看：
- HBA parse result。
- rule order。
- line number。
- auth method。
- parse error。
它不能直接证明 runtime match。
runtime match 还依赖 client address、SSL/GSS、database、user、replication、DNS 和 role membership。
### 10.3 `pg_stat_activity`
可观察字段包括：
- `pid`
- `datid` / `datname`
- `usename`
- `application_name`
- `client_addr` / `client_port`
- `backend_start`
- `state`
- `wait_event_type` / `wait_event`
- `backend_type`
`InitPostgres()` 早期调用 `pgstat_bestart_initial()`，是为了认证或锁等待卡住时也有一定可见性。
但 `pg_stat_activity` 看不到：
- `authn_id`
- HBA line number
- startup packet 原始 GUC list
- database lock recheck 中间状态
这些要靠日志、gdb 或断点。
### 10.4 wait event
认证或 startup packet 阶段等待客户端 I/O 时，可能看到：
- `ClientRead`
- `ClientWrite`
wait event 只能说明在等 FE/BE I/O。
它不能区分正在等 startup packet、password packet、SASL continuation 还是普通 query。
需要结合日志、process title、gdb backtrace 和时间点。
### 10.5 protocol message
客户端协议中常见 startup 顺序：
```text
AuthenticationRequest*
AuthenticationOk
ParameterStatus*
BackendKeyData
ReadyForQuery
```
`AuthenticationOk` 不等于连接完成。
它只说明 auth 阶段完成。
`ReadyForQuery` 才表示 backend 已完成 startup cycle 并准备接受命令。
### 10.6 SQL 观测
连接成功后可检查：
```sql
SELECT current_user, session_user, system_user;
SELECT current_database();
SHOW application_name;
SHOW client_encoding;
SHOW server_encoding;
SHOW search_path;
SELECT current_setting('statement_timeout');
```
解释边界：
- `current_user` 是 effective user。
- `session_user` 是 session authorization。
- `system_user` 来自 `auth_method:authn_id`，可能为 NULL。
- `SHOW` 看到的是 GUC 系统最终状态，不是 startup packet 原始值。
### 10.7 gdb 断点
建议断点：
```text
BackendInitialize
ProcessStartupPacket
PerformAuthentication
ClientAuthentication
hba_getauthmethod
set_authn_id
InitializeSessionUserId
CheckMyDatabase
process_startup_options
SetConfigOption
BeginReportingGUCOptions
ReadyForQuery
```
建议观察：
```text
MyProcPort->user_name
MyProcPort->database_name
MyProcPort->cmdline_options
MyProcPort->guc_options
MyProcPort->hba->auth_method
MyClientConnectionInfo.authn_id
MyDatabaseId
MyProc->databaseId
```
`AuthenticatedUserId`、`SessionUserId` 等在 `miscinit.c` 中是 static，gdb 需要在对应 compilation unit 中查看。
## 11. 常见误区
1. 把 startup packet user 当成已认证用户。它只是客户端声明。
2. 把 `authn_id` 当成 role OID。它是认证身份，不是 PostgreSQL role。
3. 把 `SYSTEM_USER`、`SESSION_USER`、`CURRENT_USER` 混成一个状态。它们来自不同层。
4. 认为 `pg_hba_file_rules` 能证明 runtime match。它只能证明 HBA 文件解析结果。
5. 认为 database name 一解析就 attach 成功。真正 attach 要经过 catalog lookup、lock、recheck、path 和 permission。
6. 认为 startup packet GUC 在认证前生效。它在 `process_startup_options()` 中才应用。
7. 认为 `AuthenticationOk` 等于连接完成。`ReadyForQuery` 才是可发查询边界。
8. 认为 database lock 持有整个 session。长期 membership 是 `MyProc->databaseId`。
9. 把所有连接失败都叫认证失败。role、database、GUC 和 login trigger 都可能在认证后失败。
10. 把 startup ERROR cleanup 当作普通 query ERROR cleanup。startup 多数失败是当前 backend 退出。
## 12. 课堂实验
### 实验 1：startup GUC 何时生效
执行：
```bash
PGOPTIONS='-c statement_timeout=3s -c application_name=startup_lab' \
psql -X -d postgres \
  -c "select current_database(), current_user, session_user, system_user" \
  -c "show application_name" \
  -c "show statement_timeout"
```
观察：
- `application_name` 和 `statement_timeout` 已进入最终 GUC 状态。
- `current_user` / `session_user` 已来自 role/session 初始化。
- `system_user` 是否为 NULL 取决于认证方法是否设置 `authn_id`。
源码对应：
```text
ProcessStartupPacket()
  -> Port.cmdline_options
InitPostgres()
  -> process_startup_options()
    -> SetConfigOption()
PostgresMain()
  -> BeginReportingGUCOptions()
```
再测试一个不允许 startup 设置的参数：
```bash
PGOPTIONS='-c shared_buffers=1GB' psql -X -d postgres -c 'select 1'
```
预期失败点在 GUC startup 阶段，不在 HBA。
### 实验 2：HBA 解析结果与 runtime match
先看解析结果：
```sql
SELECT rule_number, line_number, type, database, user_name, address, auth_method, error
FROM pg_hba_file_rules
ORDER BY rule_number;
```
再用不同连接条件测试：
```bash
psql "host=127.0.0.1 dbname=postgres user=$USER sslmode=disable" -c 'select 1'
psql "host=localhost dbname=postgres user=$USER sslmode=prefer" -c 'select 1'
psql "dbname=postgres user=$USER" -c 'select 1'
```
对照 server log 中的：
```text
connection authenticated: ... method=... (file:line)
```
解释：
- `pg_hba_file_rules` 是文件视图。
- `(file:line)` 是 runtime match 的结果。
- socket 类型、SSL/GSS、DNS 和 address 都会改变 match。
### 实验 3：gdb 跟踪状态推进
测试环境可设置：
```conf
pre_auth_delay = 10
log_connections = all
```
断点：
```gdb
b ProcessStartupPacket
b ClientAuthentication
b set_authn_id
b InitializeSessionUserId
b CheckMyDatabase
b process_startup_options
b BeginReportingGUCOptions
```
观察：
```gdb
p MyProcPort->user_name
p MyProcPort->database_name
p MyProcPort->guc_options
p MyProcPort->hba->auth_method
p MyClientConnectionInfo.authn_id
p MyDatabaseId
p MyProc->databaseId
```
预期：
- packet parse 后 `Port` 有 user/database/GUC，但 `MyDatabaseId` 无效。
- auth 后可能有 `authn_id`。
- role 初始化后 user id globals 有效。
- database attach 后 `MyDatabaseId == MyProc->databaseId`。
- startup options 后 GUC 最终值可被 `SHOW` 看到。
### 实验 4：构造 database permission 失败
测试库中：
```sql
CREATE DATABASE startup_attach_lab;
REVOKE CONNECT ON DATABASE startup_attach_lab FROM PUBLIC;
```
用普通用户连接：
```bash
psql -d startup_attach_lab -c 'select 1'
```
预期：
```text
如果 HBA 和认证成功，失败发生在 CheckMyDatabase() 的 CONNECT privilege 检查。
```
恢复：
```sql
GRANT CONNECT ON DATABASE startup_attach_lab TO PUBLIC;
DROP DATABASE startup_attach_lab;
```
## 13. 讨论题
1. 为什么 `BackendInitialize()` 要在 `InitProcess()` 前读取 startup packet？
2. startup packet 已经有 `user`，为什么还要查 `pg_authid`？
3. `authn_id`、`AuthenticatedUserId`、`SessionUserId`、`CurrentUserId` 分别回答什么问题？
4. 为什么 HBA 阶段对不存在的 role 使用 `missing_ok = true`？
5. database attach 为什么要 lookup、lock、recheck 后才设置 `MyDatabaseId`？
6. `MyDatabaseId` 和 `MyProc->databaseId` 的边界有什么不同？
7. startup packet GUC 为什么必须等认证和 database 初始化后才应用？
8. 客户端收到 `AuthenticationOk` 后仍可能在哪些阶段失败？
9. `pg_hba_file_rules` 能诊断什么，不能诊断什么？
10. 短连接风暴下，连接 startup 成本可能分布在哪些源码阶段？
## 14. 本节小结
本节核心链路：
```text
BackendInitialize()
  -> ProcessStartupPacket()
PostgresMain()
  -> InitPostgres()
    -> PerformAuthentication()
    -> InitializeSessionUserId()
    -> database lookup / lock / recheck / attach
    -> CheckMyDatabase()
    -> process_startup_options()
    -> process_settings()
    -> session initialization
  -> BeginReportingGUCOptions()
  -> BackendKeyData
  -> ReadyForQuery
```
核心状态：
- `Port` 保存连接和 startup packet 暂存状态。
- `MyClientConnectionInfo` 保存认证身份。
- user id globals 保存 PostgreSQL role/session/effective user。
- `MyDatabaseId` 是 backend-local database identity。
- `MyProc->databaseId` 是 shared memory 中的 database membership。
- GUC 系统保存 startup packet、database/role setting 和 default 合成后的最终配置。
cleanup 和生命周期：
- startup packet 阶段尽量不触碰 shared memory，失败可简单退出。
- 进入 `InitPostgres()` 后，backend 已有 ProcArray、pgstat、sinval、procsignal 和初始化事务状态。
- startup 事务失败依赖 transaction abort 和 `before_shmem_exit` cleanup。
- database lock 保护 attach，长期 membership 由 `MyProc->databaseId` 表示。
- socket、SSL、GSS 由 `on_proc_exit(socket_close)` 收尾。
错误定位按阶段：
- packet / protocol：incomplete packet、invalid length、SSL/GSS negotiation 异常。
- HBA：implicit reject 或 explicit reject。
- auth：password/SCRAM/GSS/PAM/LDAP/OAuth 失败。
- role：role missing、cannot login、role connlimit、reserved slots。
- database：missing、dropped、invalid、no CONNECT、path missing、locale failure。
- GUC：permission denied 或 parameter context 不允许。
观测边界：
- server log 能区分 received、authenticated、authorized。
- `pg_hba_file_rules` 只能看解析结果。
- `pg_stat_activity` 能看部分 backend 状态，但看不到 HBA line 和 startup GUC 原始 list。
- wait event 只能提示 I/O 等待，不构成完整因果。
- `AuthenticationOk` 不是连接完成，`ReadyForQuery` 才是客户端可发查询的边界。
可迁移规律：
```text
连接启动不是把客户端输入复制成 backend state；
它是把不可信网络意图，按认证、授权、catalog identity、shared-state publication、
GUC 权限和协议报告的顺序，逐步收敛成可清理、可观测的 runtime state。
```
