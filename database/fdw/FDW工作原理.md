# FDW工作原理

本文介绍 PostgreSQL 外部数据包装器（FDW）的构成、回调接口与生命周期，以 file_fdw 为例，面向未接触过 FDW 的 C/C++ 数据库开发者。

## 一、FDW的构成

摘取自[PostgreSQL 16官方文档 Chapter 59. Writing a Foreign Data Wrapper](https://www.postgresql.org/docs/16/fdwhandler.html)（与最新PostgreSQL 18的文档对比，实际内容基本没有差异）：

- > "The FDW author needs to implement a handler function, and optionally a validator function."
- > "Most of the effort in writing an FDW is in implementing these callback functions."
- > "The scan-related functions are required, the rest are optional."

可以得出，FDW的主要组成部分如下：

1. **Handler Function（必须）**
   - **作用**：返回FdwRoutine结构体
   - **入口**：`file_fdw_handler()` 函数
   - **说明**：必须实现，返回包含回调函数指针的FdwRoutine结构体

2. **Validator Function（可选）**
   - **作用**：验证CREATE/ALTER命令中的选项
   - **入口**：`file_fdw_validator()` 函数
   - **说明**：可选实现，用于验证FDW、SERVER、USER MAPPING、FOREIGN TABLE的选项

3. **FdwRoutine中的回调函数（核心内容）**
   - **作用**：处理查询规划和执行
   - **入口**：通过handler返回的FdwRoutine结构体
   - **说明**：必须实现scan-related函数，其他回调函数可选

接下来以项目内置的 file_fdw 为例。

## 二、Handler函数和Validator函数

### 2.1 FdwRoutine结构体

- **定义位置**：`src/include/foreign/fdwapi.h`
- **作用**：一个结构体，包含各种函数指针，用于定义FDW的回调函数接口

### 2.2 Handler函数

- **函数名**：`file_fdw_handler()`
- **官方文档**："The handler function must be registered as taking no arguments and returning type fdw_handler."
- **作用**：返回FdwRoutine结构体，包含所有回调函数的指针
- **源码位置**：`contrib/file_fdw/file_fdw.c`
- **说明**：每个FDW必须实现自己的handler函数，该函数返回包含回调函数指针的FdwRoutine结构体。PostgreSQL通过`GetFdwRoutine()`等函数调用handler获取FdwRoutine结构体（源码位置：`src/backend/foreign/foreign.c:321`）。

### 2.3 Validator函数

- **函数名**：`file_fdw_validator()`
- **官方文档**："The validator function is responsible for validating options given in CREATE and ALTER commands for its foreign data wrapper, as well as foreign servers, user mappings, and foreign tables using the wrapper."
- **作用**：验证CREATE/ALTER命令中的选项（FDW、SERVER、USER MAPPING、FOREIGN TABLE及其列属性）
- **源码位置**：`contrib/file_fdw/file_fdw.c:198`
- **说明**：每个FDW可以选择实现自己的validator函数来验证选项。验证函数负责验证 CREATE 和 ALTER 命令中给定的选项，以及使用包装器的外部服务器、用户映射和外部表。如果未提供validator函数，选项在创建或修改时不会被检查。

## 三、FdwRoutine中的回调函数

### 3.1 Scan-related Functions

根据[PostgreSQL 16官方文档Section 59.4](https://www.postgresql.org/docs/16/fdw-planning.html)，scan-related函数的主要参数说明如下：

#### **主要参数**：

- **`PlannerInfo *root`**：规划器的全局查询信息
  - "The information in `root` and `baserel` can be used to reduce the amount of information that has to be fetched from the foreign table (and therefore reduce the cost)."
  - 说明：包含整个查询的规划信息，可用于优化需要从外部表获取的数据量

- **`RelOptInfo *baserel`**：规划器关于此表的信息
  - "`baserel->baserestrictinfo` is particularly interesting, as it contains restriction quals (`WHERE` clauses) that should be used to filter the rows to be fetched."
  - "`baserel->reltarget->exprs` can be used to determine which columns need to be fetched."
  - "`baserel->fdw_private` is a `void` pointer that is available for FDW planning functions to store information relevant to the particular foreign table."
  - 说明：包含表的关系信息，其中`baserestrictinfo`包含限制条件（WHERE子句），`reltarget`可用于确定需要获取哪些列，`fdw_private`可用于在规划函数之间传递信息

- **`Oid foreigntableid`**：外部表的`pg_class` OID
  - "`foreigntableid` is the `pg_class` OID of the foreign table. (`foreigntableid` could be obtained from the planner data structures, but it's passed explicitly to save effort.)"
  - 说明：外部表的系统目录OID，显式传递以避免从规划器数据结构中查找

#### **GetForeignRelSize**

在查询规划开始时被调用，获取外部表的大小估算。

**官方文档**：
- > "Obtain relation size estimates for a foreign table. This is called at the beginning of planning for a query that scans a foreign table."
- > "This function should update baserel->rows to be the expected number of rows returned by the table scan, after accounting for the filtering done by the restriction quals."

**函数签名**：
```c
void GetForeignRelSize(PlannerInfo *root,
                       RelOptInfo *baserel,
                       Oid foreigntableid);
```

**作用**：
- 更新`baserel->rows`：估算扫描后返回的行数（考虑过滤条件）
- 可选更新`baserel->width`和`baserel->tuples`

**源码位置**：`contrib/file_fdw/file_fdw.c:514`

#### **GetForeignPaths**

在查询规划期间被调用，为外部表扫描创建可能的访问路径。

**官方文档**：
- > "Create possible access paths for a scan on a foreign table. This is called during query planning."
- > "This function must generate at least one access path (`ForeignPath` node) for a scan on the foreign table and must call `add_path` to add each such path to `baserel->pathlist`."
- > "Each access path must contain cost estimates, and can contain any FDW-private information that is needed to identify the specific scan method intended."

**函数签名**：
```c
void GetForeignPaths(PlannerInfo *root,
                     RelOptInfo *baserel,
                     Oid foreigntableid);
```

**作用**：
- 必须生成至少一个`ForeignPath`节点并调用`add_path`添加到`baserel->pathlist`
- 可以生成多个访问路径（例如，包含有效`pathkeys`表示预排序结果的路径）
- 每个访问路径必须包含成本估算，可以包含FDW私有信息

**源码位置**：`contrib/file_fdw/file_fdw.c:545`

#### **GetForeignPlan**

在查询规划结束时被调用，从选中的外部访问路径创建`ForeignScan`计划节点。

**官方文档**：
- > "Create a `ForeignScan` plan node from the selected foreign access path. This is called at the end of query planning."
- > "The parameters are as for `GetForeignRelSize`, plus the selected `ForeignPath` (previously produced by `GetForeignPaths`, `GetForeignJoinPaths`, or `GetForeignUpperPaths`), the target list to be emitted by the plan node, the restriction clauses to be enforced by the plan node, and the outer subplan of the `ForeignScan`, which is used for rechecks performed by `RecheckForeignScan`."
- > "This function must create and return a `ForeignScan` plan node; it's recommended to use make_foreignscan to build the `ForeignScan` node."
- > "In `GetForeignPlan`, generally the passed-in target list can be copied into the plan node as-is. The passed `scan_clauses` list contains the same clauses as `baserel->baserestrictinfo`, but may be re-ordered for better execution efficiency."

**函数签名**：
```c
ForeignScan *GetForeignPlan(PlannerInfo *root,
                            RelOptInfo *baserel,
                            Oid foreigntableid,
                            ForeignPath *best_path,
                            List *tlist,
                            List *scan_clauses,
                            Plan *outer_plan);
```

**参数说明**：
- `best_path`：选中的外部访问路径 
- `tlist`：计划节点要输出的目标列表
- `scan_clauses`：计划节点要执行的限制条件
- `outer_plan`：`ForeignScan`的外层子计划 

**作用**：
- 必须创建并返回`ForeignScan`计划节点，推荐使用`make_foreignscan`构建

**源码位置**：`contrib/file_fdw/file_fdw.c:598`

#### **BeginForeignScan**

在执行器启动时被调用，执行外部扫描开始前所需的初始化。

**官方文档**：
- > "Begin executing a foreign scan. This is called during executor startup."
- > "It should perform any initialization needed before the scan can start, but not start executing the actual scan (that should be done upon the first call to `IterateForeignScan`)."
- > "eflags contains flag bits describing the executor's operating mode for this plan node."

**函数签名**：
```c
void BeginForeignScan(ForeignScanState *node,
                      int eflags);
```

**参数说明**：
- `node`：`ForeignScanState`节点（已创建，但`fdw_state`字段仍为NULL）
- `eflags`：描述执行器操作模式的标志位

**返回值**：无（void）

**作用**：
- 执行扫描开始前所需的初始化（但不开始实际扫描）

**源码位置**：`contrib/file_fdw/file_fdw.c:665`

#### **IterateForeignScan**

在查询执行期间被循环调用，从外部数据源获取一行数据。

**官方文档**：
- > "Fetch one row from the foreign source, returning it in a tuple table slot (the node's `ScanTupleSlot` should be used for this purpose). Return NULL if no more rows are available."

**函数签名**：
```c
TupleTableSlot *IterateForeignScan(ForeignScanState *node);
```

**参数说明**：
- `node`：`ForeignScanState`节点

**返回值**：`TupleTableSlot`指针（包含一行数据），如果没有更多行则返回NULL

**作用**：
- 返回的元组必须匹配`fdw_scan_tlist`目标列表（如果提供），否则必须匹配被扫描的外部表的行类型
- 在短生命周期的内存上下文中被调用（每次调用之间会重置）
- 如果需要长生命周期的存储，应在`BeginForeignScan`中创建内存上下文

**源码位置**：`contrib/file_fdw/file_fdw.c:719`

#### **ReScanForeignScan**

在需要重新扫描时被调用（如嵌套循环），从开始处重新启动扫描。

**官方文档**：
- > "Restart the scan from the beginning. Note that any parameters the scan depends on may have changed value, so the new scan does not necessarily return exactly the same rows."

**函数签名**：
```c
void ReScanForeignScan(ForeignScanState *node);
```

**作用**：
- 扫描依赖的任何参数可能已更改值，因此新扫描不一定返回完全相同的行

**源码位置**：`contrib/file_fdw/file_fdw.c`

#### **EndForeignScan**

在扫描结束时被调用，结束扫描并释放资源。

**官方文档**：
- > "End the scan and release resources. It is normally not important to release palloc'd memory, but for example open files and connections to remote servers should be cleaned up."

**函数签名**：
```c
void EndForeignScan(ForeignScanState *node);
```
**作用**：
- 通常不需要释放palloc分配的内存
- 但应清理打开的文件和到远程服务器的连接等资源

**源码位置**：`contrib/file_fdw/file_fdw.c`

### 3.2 其他回调函数（可选）

根据[PostgreSQL 16官方文档Section 59.2](https://www.postgresql.org/docs/16/fdw-callbacks.html)和`FdwRoutine`结构体定义（`src/include/foreign/fdwapi.h`），除了scan-related函数外，FDW还可以实现以下可选回调函数：

- > "Remaining functions are optional. Set the pointer to NULL for any that are not provided."

**主要可选功能分类**：

1. **远程JOIN和上层关系处理**（[59.2.2](https://www.postgresql.org/docs/16/fdw-callbacks.html#FDW-CALLBACKS-JOIN-SCAN), [59.2.3](https://www.postgresql.org/docs/16/fdw-callbacks.html#FDW-CALLBACKS-UPPER-PATHS)）
   - `GetForeignJoinPaths`：支持在远程服务器执行JOIN操作
   - `GetForeignUpperPaths`：支持远程执行聚合、窗口函数等上层关系处理

2. **外部表更新操作**（[59.2.4](https://www.postgresql.org/docs/16/fdw-callbacks.html#FDW-CALLBACKS-UPDATE)）
   - `PlanForeignModify`, `BeginForeignModify`, `ExecForeignInsert`, `ExecForeignUpdate`, `ExecForeignDelete`, `EndForeignModify`等：支持对外部表的INSERT、UPDATE、DELETE操作

3. **EXPLAIN支持**（[59.2.7](https://www.postgresql.org/docs/16/fdw-callbacks.html#FDW-CALLBACKS-EXPLAIN)）
   - `ExplainForeignScan`, `ExplainForeignModify`, `ExplainDirectModify`：为EXPLAIN命令提供额外的输出信息

4. **ANALYZE支持**（[59.2.8](https://www.postgresql.org/docs/16/fdw-callbacks.html#FDW-CALLBACKS-ANALYZE)）
   - `AnalyzeForeignTable`：支持对外部表执行ANALYZE收集统计信息

5. **其他功能**
   - **TRUNCATE**（[59.2.5](https://www.postgresql.org/docs/16/fdw-callbacks.html#FDW-CALLBACKS-TRUNCATE)）：`ExecForeignTruncate`
   - **行锁定**（[59.2.6](https://www.postgresql.org/docs/16/fdw-callbacks.html#FDW-CALLBACKS-ROW-LOCKING)）：`GetForeignRowMarkType`, `RefetchForeignRow`, `RecheckForeignScan`
   - **IMPORT FOREIGN SCHEMA**（[59.2.9](https://www.postgresql.org/docs/16/fdw-callbacks.html#FDW-CALLBACKS-IMPORT-SCHEMA)）：`ImportForeignSchema`
   - **并行执行**（[59.2.10](https://www.postgresql.org/docs/16/fdw-callbacks.html#FDW-CALLBACKS-PARALLEL)）：`IsForeignScanParallelSafe`, `EstimateDSMForeignScan`等
   - **异步执行**（[59.2.11](https://www.postgresql.org/docs/16/fdw-callbacks.html#FDW-CALLBACKS-ASYNC)）：`IsForeignPathAsyncCapable`, `ForeignAsyncRequest`等
   - **路径重参数化**（[59.2.12](https://www.postgresql.org/docs/16/fdw-callbacks.html#FDW-CALLBACKS-REPARAMETERIZATION)）：`ReparameterizeForeignPathByChild`

---

## 四、FDW生命周期

### 4.0 file_fdw使用流程概览与统一说明

**统一说明**（适用于4.1-4.4所有阶段）：
- **handler函数调用时机**：handler函数只在查询时通过`GetFdwRoutine()`调用，CREATE阶段只是记录其OID到系统表，不会调用handler函数
- **validator函数调用时机**：如果CREATE语句包含OPTIONS，且FDW提供了validator函数，validator会在`transformGenericOptions()`函数内部被调用（`foreigncmds.c:191`）来验证选项有效性
- **依赖关系存储**：所有依赖关系都记录在`pg_depend`系统表中，用于防止误删除和级联删除

以`file_fdw`为例，使用FDW的典型流程如下：

```sql
-- 步骤1：创建扩展（注册FDW）
CREATE EXTENSION file_fdw;

-- 步骤2：创建外部服务器（配置数据源连接信息）
-- 注意：服务器名称（如file_server）是用户自定义的，可以有多个服务器
CREATE SERVER file_server FOREIGN DATA WRAPPER file_fdw;
-- 注意：对于file_fdw，CREATE SERVER的OPTIONS没有实际意义（因为访问本地文件不需要连接配置）
-- 真正的配置都在CREATE FOREIGN TABLE的OPTIONS中（如filename、format等）

-- 对于mysql_fdw、oracle_fdw等需要连接远程服务器的FDW，CREATE SERVER的OPTIONS才有实际意义：
-- CREATE SERVER mysql_server1 FOREIGN DATA WRAPPER mysql_fdw OPTIONS (host '192.168.1.100', port '3306');
-- CREATE SERVER mysql_server2 FOREIGN DATA WRAPPER mysql_fdw OPTIONS (host '192.168.1.101', port '3306');
-- 注意：CREATE SERVER时不会建立连接，连接是在实际查询时建立的

-- 步骤3：创建用户映射（可选，file_fdw通常不需要，但其他FDW如mysql_fdw需要）
-- CREATE USER MAPPING FOR CURRENT_USER SERVER file_server OPTIONS (...);

-- 步骤4：创建外部表（定义表结构）
-- 注意：创建外部表时需要指定使用哪个SERVER
CREATE FOREIGN TABLE foreign_file_table (
    col1 INTEGER,
    col2 TEXT
) SERVER file_server
OPTIONS (filename '/path/to/file.csv', format 'csv');

-- 步骤5：查询外部表（使用FDW）
SELECT * FROM foreign_file_table WHERE col1 > 100;
```

**说明**：
- **服务器名称**：`file_server`是用户自定义的名称，用于标识这个外部服务器配置
- **file_fdw的SERVER**：对于`file_fdw`，CREATE SERVER主要是框架要求，其OPTIONS没有实际意义（因为访问本地文件不需要连接配置）。真正的配置都在CREATE FOREIGN TABLE的OPTIONS中（如filename、format等）
- **mysql_fdw/oracle_fdw的SERVER**：对于需要连接远程服务器的FDW，CREATE SERVER的OPTIONS才有实际意义（如host、port等），用于配置如何连接到远程数据源。注意：CREATE SERVER时不会建立连接，连接是在实际查询时建立的
- **多个服务器**：可以创建多个SERVER，每个有不同的名称和配置（如连接不同的MySQL服务器）
- **SERVER与FDW的关系**：一个FDW（如`mysql_fdw`）可以创建多个SERVER，每个SERVER可以连接不同的远程数据源

接下来将逐步阐述每个阶段发生了什么。

### 4.1 CREATE EXTENSION阶段（注册阶段）

根据[官方文档](https://www.postgresql.org/docs/16/sql-createextension.html)：
- > "Loading an extension essentially amounts to running the extension's script file. The script will typically create new SQL objects such as functions, data types, operators and index support methods."

对于`file_fdw`的安装，我们查看`contrib/file_fdw/file_fdw--1.0.sql`：

```sql
-- 1. 创建handler函数（注册到pg_proc系统表）
CREATE FUNCTION file_fdw_handler()
RETURNS fdw_handler
AS 'MODULE_PATHNAME'
LANGUAGE C STRICT;

-- 2. 创建validator函数（注册到pg_proc系统表）
CREATE FUNCTION file_fdw_validator(text[], oid)
RETURNS void
AS 'MODULE_PATHNAME'
LANGUAGE C STRICT;

-- 3. 创建FDW对象（注册到pg_foreign_data_wrapper系统表）
CREATE FOREIGN DATA WRAPPER file_fdw
  HANDLER file_fdw_handler
  VALIDATOR file_fdw_validator;
```

**说明**：
- `CREATE FUNCTION`语句会将函数注册到`pg_proc`系统表中，这里不详细介绍
- 我们重点关注`CREATE FOREIGN DATA WRAPPER`语句的执行过程

当执行`CREATE FOREIGN DATA WRAPPER`命令时，PostgreSQL内核会调用`CreateForeignDataWrapper()`函数（源码位置：`src/backend/commands/foreigncmds.c:557`）。

**函数核心作用**：将FDW对象注册到`pg_foreign_data_wrapper`系统表，建立FDW与handler/validator函数的关联关系。

**函数执行流程**：

#### 1. 权限检查

```c
// foreigncmds.c:576-582
/* Must be superuser */
if (!superuser())
    ereport(ERROR,
            (errcode(ERRCODE_INSUFFICIENT_PRIVILEGE),
             errmsg("permission denied to create foreign-data wrapper \"%s\"",
                    stmt->fdwname),
             errhint("Must be system admin to create a foreign-data wrapper.")));
```

**作用**：只有超级用户才能创建FDW。

#### 2. 确定所有者

```c
// foreigncmds.c:584-585
/* For now the owner cannot be specified on create. Use effective user ID. */
ownerId = GetUserId();
```

**作用**：使用当前有效用户ID作为FDW的所有者。

#### 3. 检查重复名称

```c
// foreigncmds.c:587-594
/*
 * Check that there is no other foreign-data wrapper by this name.
 */
if (GetForeignDataWrapperByName(stmt->fdwname, true) != NULL)
    ereport(ERROR,
            (errcode(ERRCODE_DUPLICATE_OBJECT),
             errmsg("foreign-data wrapper \"%s\" already exists",
                    stmt->fdwname)));
```

**作用**：确保FDW名称唯一，如果已存在同名FDW则抛出错误。

#### 4. 插入系统表

```c
// foreigncmds.c:596-631
/*
 * Insert tuple into pg_foreign_data_wrapper.
 */
memset(values, 0, sizeof(values));
memset(nulls, false, sizeof(nulls));

// 生成OID并设置基本字段
fdwId = GetNewOidWithIndex(rel, ForeignDataWrapperOidIndexId,
                           Anum_pg_foreign_data_wrapper_oid);
values[Anum_pg_foreign_data_wrapper_oid - 1] = ObjectIdGetDatum(fdwId);
values[Anum_pg_foreign_data_wrapper_fdwname - 1] =
    DirectFunctionCall1(namein, CStringGetDatum(stmt->fdwname));
values[Anum_pg_foreign_data_wrapper_fdwowner - 1] = ObjectIdGetDatum(ownerId);

// 查找handler和validator函数
/* Lookup handler and validator functions, if given */
parse_func_options(pstate, stmt->func_options,
                   &handler_given, &fdwhandler,
                   &validator_given, &fdwvalidator);

values[Anum_pg_foreign_data_wrapper_fdwhandler - 1] = ObjectIdGetDatum(fdwhandler);
values[Anum_pg_foreign_data_wrapper_fdwvalidator - 1] = ObjectIdGetDatum(fdwvalidator);

// ACL字段初始化为NULL
nulls[Anum_pg_foreign_data_wrapper_fdwacl - 1] = true;

// 处理OPTIONS（可能调用validator）
fdwoptions = transformGenericOptions(ForeignDataWrapperRelationId,
                                     PointerGetDatum(NULL),
                                     stmt->options,
                                     fdwvalidator);

if (PointerIsValid(DatumGetPointer(fdwoptions)))
    values[Anum_pg_foreign_data_wrapper_fdwoptions - 1] = fdwoptions;
else
    nulls[Anum_pg_foreign_data_wrapper_fdwoptions - 1] = true;

// 构建并插入元组
tuple = heap_form_tuple(rel->rd_att, values, nulls);
CatalogTupleInsert(rel, tuple);
heap_freetuple(tuple);
```

**作用**：
- 生成FDW的OID
- 设置基本字段（oid、fdwname、fdwowner）
- 查找并设置handler和validator函数的OID
- 处理OPTIONS（如果提供了validator函数，会在`transformGenericOptions()`内部调用validator验证选项）
- 插入到`pg_foreign_data_wrapper`系统表

#### 5. 记录依赖关系

```c
// foreigncmds.c:635-659
/* record dependencies */
myself.classId = ForeignDataWrapperRelationId;
myself.objectId = fdwId;
myself.objectSubId = 0;

// 记录对handler函数的依赖
if (OidIsValid(fdwhandler))
{
    referenced.classId = ProcedureRelationId;
    referenced.objectId = fdwhandler;
    referenced.objectSubId = 0;
    recordDependencyOn(&myself, &referenced, DEPENDENCY_NORMAL);
}

// 记录对validator函数的依赖
if (OidIsValid(fdwvalidator))
{
    referenced.classId = ProcedureRelationId;
    referenced.objectId = fdwvalidator;
    referenced.objectSubId = 0;
    recordDependencyOn(&myself, &referenced, DEPENDENCY_NORMAL);
}

// 记录对所有者的依赖
recordDependencyOnOwner(ForeignDataWrapperRelationId, fdwId, ownerId);

// 记录对扩展的依赖（如果是在扩展中创建的）
/* dependency on extension */
recordDependencyOnCurrentExtension(&myself, false);
```

**作用**：在`pg_depend`系统表中记录FDW对handler函数、validator函数、所有者和扩展的依赖关系，防止误删除。

**系统表变化**：

| 系统表 | 操作 | 说明 |
|--------|------|------|
| `pg_proc` | 已存在 | handler和validator函数在`CREATE FUNCTION`时已注册 |
| `pg_foreign_data_wrapper` | INSERT | 插入1条FDW记录，包含fdwname、fdwhandler、fdwvalidator、fdwoptions等 |
| `pg_depend` | INSERT | 插入依赖关系记录（FDW对handler/validator函数、所有者、扩展的依赖） |

**CREATE EXTENSION阶段总结**：
1. 注册handler和validator函数到`pg_proc`系统表
2. 注册FDW对象到`pg_foreign_data_wrapper`系统表，记录handler/validator函数的OID
3. 处理并验证OPTIONS（如果提供了validator函数）
4. 建立依赖关系，确保函数不会被误删除
5. 为后续的CREATE SERVER、CREATE FOREIGN TABLE等操作做好准备

### 4.2 CREATE SERVER阶段（配置阶段）

当执行`CREATE SERVER file_server FOREIGN DATA WRAPPER file_fdw;`命令时，PostgreSQL内核会调用`CreateForeignServer()`函数（源码位置：`src/backend/commands/foreigncmds.c:837`）。

**函数核心作用**：创建外部服务器对象，将SERVER注册到`pg_foreign_server`系统表，建立SERVER与FDW的关联关系。

**函数执行流程**：

#### 1. 确定所有者（For now the owner cannot be specified on create）

```c
// foreigncmds.c:854-855
/* For now the owner cannot be specified on create. Use effective user ID. */
ownerId = GetUserId();
```

**作用**：使用当前有效用户ID作为SERVER的所有者。

#### 2. 检查重复名称

```c
// foreigncmds.c:857-886
/*
 * Check that there is no other foreign server by this name.  If there is
 * one, do nothing if IF NOT EXISTS was specified.
 */
srvId = get_foreign_server_oid(stmt->servername, true);
if (OidIsValid(srvId))
{
    if (stmt->if_not_exists)
    {
        // 如果是在扩展脚本中，检查现有对象是否属于扩展
        ObjectAddressSet(myself, ForeignServerRelationId, srvId);
        checkMembershipInCurrentExtension(&myself);
        // 跳过创建，返回NOTICE
        ereport(NOTICE, ...);
        return InvalidObjectAddress;
    }
    else
        ereport(ERROR, ...);  // 抛出错误
}
```

**作用**：确保SERVER名称唯一，支持`IF NOT EXISTS`选项。

#### 3. 查找FDW并检查权限

```c
// foreigncmds.c:888-896
/*
 * Check that the FDW exists and that we have USAGE on it. Also get the
 * actual FDW for option validation etc.
 */
fdw = GetForeignDataWrapperByName(stmt->fdwname, false);

aclresult = object_aclcheck(ForeignDataWrapperRelationId, fdw->fdwid, ownerId, ACL_USAGE);
if (aclresult != ACLCHECK_OK)
    aclcheck_error(aclresult, OBJECT_FDW, fdw->fdwname);
```

**作用**：查找FDW对象，检查用户是否有`USAGE`权限（比创建FDW的要求低，创建FDW需要超级用户权限）。

#### 4. 插入系统表

```c
// foreigncmds.c:898-944
/*
 * Insert tuple into pg_foreign_server.
 */
memset(values, 0, sizeof(values));
memset(nulls, false, sizeof(nulls));

// 生成OID并设置基本字段
srvId = GetNewOidWithIndex(rel, ForeignServerOidIndexId,
                           Anum_pg_foreign_server_oid);
values[Anum_pg_foreign_server_oid - 1] = ObjectIdGetDatum(srvId);
values[Anum_pg_foreign_server_srvname - 1] =
    DirectFunctionCall1(namein, CStringGetDatum(stmt->servername));
values[Anum_pg_foreign_server_srvowner - 1] = ObjectIdGetDatum(ownerId);
values[Anum_pg_foreign_server_srvfdw - 1] = ObjectIdGetDatum(fdw->fdwid);  // 关键：关联FDW

// 设置服务器类型和版本（可选）
/* Add server type if supplied */
if (stmt->servertype)
    values[Anum_pg_foreign_server_srvtype - 1] =
        CStringGetTextDatum(stmt->servertype);
else
    nulls[Anum_pg_foreign_server_srvtype - 1] = true;

/* Add server version if supplied */
if (stmt->version)
    values[Anum_pg_foreign_server_srvversion - 1] =
        CStringGetTextDatum(stmt->version);
else
    nulls[Anum_pg_foreign_server_srvversion - 1] = true;

// ACL字段初始化为NULL
/* Start with a blank acl */
nulls[Anum_pg_foreign_server_srvacl - 1] = true;

// 处理OPTIONS（可能调用validator）
/* Add server options */
srvoptions = transformGenericOptions(ForeignServerRelationId,
                                     PointerGetDatum(NULL),
                                     stmt->options,
                                     fdw->fdwvalidator);

if (PointerIsValid(DatumGetPointer(srvoptions)))
    values[Anum_pg_foreign_server_srvoptions - 1] = srvoptions;
else
    nulls[Anum_pg_foreign_server_srvoptions - 1] = true;

// 构建并插入元组
tuple = heap_form_tuple(rel->rd_att, values, nulls);
CatalogTupleInsert(rel, tuple);
heap_freetuple(tuple);
```

**作用**：
- 生成SERVER的OID
- 设置基本字段（oid、srvname、srvowner、srvfdw）
- 设置服务器类型和版本（可选）
- 处理OPTIONS（如果提供了validator函数，会在`transformGenericOptions()`内部调用validator验证选项）
- 插入到`pg_foreign_server`系统表

#### 5. 记录依赖关系

```c
// foreigncmds.c:946-959
/* record dependencies */
myself.classId = ForeignServerRelationId;
myself.objectId = srvId;
myself.objectSubId = 0;

// 记录对FDW的依赖
referenced.classId = ForeignDataWrapperRelationId;
referenced.objectId = fdw->fdwid;
referenced.objectSubId = 0;
recordDependencyOn(&myself, &referenced, DEPENDENCY_NORMAL);

// 记录对所有者的依赖
recordDependencyOnOwner(ForeignServerRelationId, srvId, ownerId);

// 记录对扩展的依赖（如果是在扩展中创建的）
/* dependency on extension */
recordDependencyOnCurrentExtension(&myself, false);
```

**作用**：在`pg_depend`系统表中记录SERVER对FDW、所有者和扩展的依赖关系，防止误删除。

**系统表变化**：

| 系统表 | 操作 | 说明 |
|--------|------|------|
| `pg_foreign_server` | INSERT | 插入1条SERVER记录，包含srvname、srvfdw（关联FDW）、srvtype、srvversion、srvoptions等 |
| `pg_depend` | INSERT | 插入依赖关系记录（SERVER对FDW、所有者、扩展的依赖） |

**file_fdw vs mysql_fdw的区别**：
- **file_fdw**：CREATE SERVER主要是框架要求，其OPTIONS对`file_fdw`没有实际意义（因为访问本地文件不需要连接配置）
- **mysql_fdw/oracle_fdw等**：CREATE SERVER的OPTIONS用于配置如何连接到远程数据源（如host、port等）。注意：CREATE SERVER时不会建立连接，连接是在实际查询时建立的

**CREATE SERVER阶段总结**：
1. 验证FDW存在且用户有权限使用
2. 注册SERVER对象到`pg_foreign_server`系统表，建立与FDW的关联关系
3. 处理并验证OPTIONS（如果提供了validator函数）
4. 建立依赖关系，确保FDW不会被误删除
5. 为后续的CREATE FOREIGN TABLE操作做好准备

### 4.3 CREATE USER MAPPING阶段（用户映射阶段）

当执行`CREATE USER MAPPING FOR user_name SERVER server_name OPTIONS (...);`命令时，PostgreSQL内核会调用`CreateUserMapping()`函数（源码位置：`src/backend/commands/foreigncmds.c:1099`）。

**函数核心作用**：创建用户映射对象，将USER MAPPING注册到`pg_user_mapping`系统表，建立用户与SERVER的关联关系，用于提供远程数据源的认证信息。

**说明**：
- `file_fdw`通常不需要用户映射（因为访问本地文件不需要认证）
- 但其他FDW（如`mysql_fdw`、`postgres_fdw`）需要用户映射来提供远程数据源的认证信息（如username、password等）

**函数执行流程**：

#### 1. 确定用户OID

```c
// foreigncmds.c:1117-1120
if (role->roletype == ROLESPEC_PUBLIC)
    useId = ACL_ID_PUBLIC;  // PUBLIC映射，适用于所有用户
else
    useId = get_rolespec_oid(stmt->user, false);  // 特定用户映射
```

**作用**：支持PUBLIC和特定用户两种类型的映射。

#### 2. 查找SERVER

```c
// foreigncmds.c:1122-1123
/* Check that the server exists. */
srv = GetForeignServerByName(stmt->servername, false);
```

**作用**：确保SERVER存在。

#### 3. 权限检查

```c
// foreigncmds.c:1125
user_mapping_ddl_aclcheck(useId, srv->serverid, stmt->servername);
```

**作用**：检查用户是否有权限创建该用户映射。
- **SERVER所有者**：可以创建该SERVER上的任何用户映射
- **普通用户**：只能创建自己的用户映射，且需要有SERVER的`USAGE`权限

#### 4. 检查重复映射

```c
// foreigncmds.c:1127-1157
/*
 * Check that the user mapping is unique within server.
 */
umId = GetSysCacheOid2(USERMAPPINGUSERSERVER, Anum_pg_user_mapping_oid,
                       ObjectIdGetDatum(useId),
                       ObjectIdGetDatum(srv->serverid));

if (OidIsValid(umId))
{
    if (stmt->if_not_exists)
    {
        // 跳过创建，返回NOTICE
        ereport(NOTICE, ...);
        return InvalidObjectAddress;
    }
    else
        ereport(ERROR, ...);  // 抛出错误
}
```

**作用**：确保每个SERVER上每个用户的映射唯一，支持`IF NOT EXISTS`选项。

#### 5. 插入系统表

```c
// foreigncmds.c:1159-1188
/*
 * Insert tuple into pg_user_mapping.
 */
memset(values, 0, sizeof(values));
memset(nulls, false, sizeof(nulls));

// 生成OID并设置基本字段
umId = GetNewOidWithIndex(rel, UserMappingOidIndexId,
                          Anum_pg_user_mapping_oid);
values[Anum_pg_user_mapping_oid - 1] = ObjectIdGetDatum(umId);
values[Anum_pg_user_mapping_umuser - 1] = ObjectIdGetDatum(useId);
values[Anum_pg_user_mapping_umserver - 1] = ObjectIdGetDatum(srv->serverid);  // 关键：关联SERVER

// 处理OPTIONS（可能调用validator）
/* Add user options */
useoptions = transformGenericOptions(UserMappingRelationId,
                                     PointerGetDatum(NULL),
                                     stmt->options,
                                     fdw->fdwvalidator);

if (PointerIsValid(DatumGetPointer(useoptions)))
    values[Anum_pg_user_mapping_umoptions - 1] = useoptions;
else
    nulls[Anum_pg_user_mapping_umoptions - 1] = true;

// 构建并插入元组
tuple = heap_form_tuple(rel->rd_att, values, nulls);
CatalogTupleInsert(rel, tuple);
heap_freetuple(tuple);
```

**作用**：
- 生成USER MAPPING的OID
- 设置基本字段（oid、umuser、umserver）
- 处理OPTIONS（如果提供了validator函数，会在`transformGenericOptions()`内部调用validator验证选项）
- 插入到`pg_user_mapping`系统表

#### 6. 记录依赖关系

```c
// foreigncmds.c:1190-1204
/* Add dependency on the server */
myself.classId = UserMappingRelationId;
myself.objectId = umId;
myself.objectSubId = 0;

referenced.classId = ForeignServerRelationId;
referenced.objectId = srv->serverid;
referenced.objectSubId = 0;
recordDependencyOn(&myself, &referenced, DEPENDENCY_NORMAL);

// 如果useId有效（不是PUBLIC），记录对用户的依赖
if (OidIsValid(useId))
{
    /* Record the mapped user dependency */
    recordDependencyOnOwner(UserMappingRelationId, umId, useId);
}
```

**作用**：在`pg_depend`系统表中记录USER MAPPING对SERVER和用户的依赖关系，防止误删除。

**注意**：USER MAPPING不记录对扩展的依赖（见代码注释：`foreigncmds.c:1206-1210`），因为角色不是扩展成员。

**系统表变化**：

| 系统表 | 操作 | 说明 |
|--------|------|------|
| `pg_user_mapping` | INSERT | 插入1条USER MAPPING记录，包含umuser（用户OID，PUBLIC时为InvalidOid）、umserver（关联SERVER）、umoptions等 |
| `pg_depend` | INSERT | 插入依赖关系记录（USER MAPPING对SERVER的依赖，以及对用户的依赖（如果不是PUBLIC）） |

**file_fdw vs mysql_fdw的区别**：
- **file_fdw**：通常不需要用户映射（因为访问本地文件不需要认证）
- **mysql_fdw/postgres_fdw等**：需要用户映射来提供远程数据源的认证信息（如username、password等）。注意：CREATE USER MAPPING时不会建立连接，连接是在实际查询时建立的

**CREATE USER MAPPING阶段总结**：
1. 验证SERVER存在且用户有权限创建映射
2. 注册USER MAPPING对象到`pg_user_mapping`系统表，建立用户与SERVER的关联关系
3. 处理并验证OPTIONS（如果提供了validator函数）
4. 建立依赖关系，确保SERVER不会被误删除
5. 为后续的查询操作提供认证信息

### 4.4 CREATE FOREIGN TABLE阶段（定义外部表阶段）

当执行`CREATE FOREIGN TABLE foreign_file_table (...) SERVER file_server OPTIONS (...);`命令时，PostgreSQL内核的处理方式与`CREATE TABLE`基本相同，只是多了一个额外的步骤。

**重要说明**：CREATE FOREIGN TABLE与CREATE TABLE共享相同的处理逻辑。在PostgreSQL中，外部表首先是一个"表"（需要在`pg_class`系统表中），然后才是"外部表"（在`pg_foreign_table`系统表中）。因此，CREATE FOREIGN TABLE的实际创建步骤与CREATE TABLE完全一样，只是在最后多了一个步骤来创建`pg_foreign_table`条目。

**调用链**（`src/backend/tcop/utility.c:1144-1223`）：
```c
case T_CreateStmt:
case T_CreateForeignTableStmt:  // CREATE FOREIGN TABLE和CREATE TABLE共享处理逻辑
{
    // ...
    else if (IsA(stmt, CreateForeignTableStmt))
    {
        // CREATE FOREIGN TABLE的处理：创建pg_class条目，然后创建pg_foreign_table条目
        address = DefineRelation(&cstmt->base, RELKIND_FOREIGN_TABLE, ...);
        CreateForeignTable(cstmt, address.objectId);  // 多出的这一步
    }
}
```

**函数核心作用**：`CreateForeignTable()`函数（源码位置：`src/backend/commands/foreigncmds.c:1403`）在`DefineRelation()`之后被调用，负责创建`pg_foreign_table`系统表条目，建立外部表与SERVER的关联关系。

**函数执行流程**：

#### 1. 命令计数器递增

```c
// foreigncmds.c:1418-1422
/*
 * Advance command counter to ensure the pg_attribute tuple is visible;
 * the tuple might be updated to add constraints in previous step.
 */
CommandCounterIncrement();
```

**作用**：确保`DefineRelation()`创建的对象在当前命令中可见，特别是`pg_attribute`系统表的更新。

#### 2. 确定所有者

```c
// foreigncmds.c:1426-1429
/*
 * For now the owner cannot be specified on create. Use effective user ID.
 */
ownerId = GetUserId();
```

**作用**：使用当前有效用户ID作为外部表的所有者（与`pg_class`中的所有者相同）。

#### 3. 查找SERVER并检查权限

```c
// foreigncmds.c:1431-1440
/*
 * Check that the foreign server exists and that we have USAGE on it. Also
 * get the actual FDW for option validation etc.
 */
server = GetForeignServerByName(stmt->servername, false);
aclresult = object_aclcheck(ForeignServerRelationId, server->serverid, ownerId, ACL_USAGE);
if (aclresult != ACLCHECK_OK)
    aclcheck_error(aclresult, OBJECT_FOREIGN_SERVER, server->servername);

fdw = GetForeignDataWrapper(server->fdwid);
```

**作用**：
- 查找SERVER对象，确保SERVER存在
- 检查用户是否有`USAGE`权限使用该SERVER
- 获取SERVER关联的FDW对象，用于后续的validator函数调用

#### 4. 插入系统表

```c
// foreigncmds.c:1442-1465
/*
 * Insert tuple into pg_foreign_table.
 */
memset(values, 0, sizeof(values));
memset(nulls, false, sizeof(nulls));

// 设置基本字段
values[Anum_pg_foreign_table_ftrelid - 1] = ObjectIdGetDatum(relid);  // 关联pg_class
values[Anum_pg_foreign_table_ftserver - 1] = ObjectIdGetDatum(server->serverid);  // 关联SERVER
// relid参数：由DefineRelation()返回，是pg_class系统表中外部表记录的OID

// 处理OPTIONS（可能调用validator）
/* Add table generic options */
ftoptions = transformGenericOptions(ForeignTableRelationId,
                                    PointerGetDatum(NULL),
                                    stmt->options,
                                    fdw->fdwvalidator);

if (PointerIsValid(DatumGetPointer(ftoptions)))
    values[Anum_pg_foreign_table_ftoptions - 1] = ftoptions;
else
    nulls[Anum_pg_foreign_table_ftoptions - 1] = true;

// 构建并插入元组
tuple = heap_form_tuple(ftrel->rd_att, values, nulls);
CatalogTupleInsert(ftrel, tuple);
heap_freetuple(tuple);
```

**作用**：
- 设置基本字段（ftrelid、ftserver）
- 处理OPTIONS（如果提供了validator函数，会在`transformGenericOptions()`内部调用validator验证选项）
- 插入到`pg_foreign_table`系统表

#### 5. 记录依赖关系

```c
// foreigncmds.c:1467-1475
/* Add pg_class dependency on the server */
myself.classId = RelationRelationId;  // pg_class系统表
myself.objectId = relid;              // 外部表在pg_class中的OID
myself.objectSubId = 0;

referenced.classId = ForeignServerRelationId;
referenced.objectId = server->serverid;
referenced.objectSubId = 0;
recordDependencyOn(&myself, &referenced, DEPENDENCY_NORMAL);
```

**作用**：在`pg_depend`系统表中记录外部表（`pg_class`中的记录）对SERVER的依赖关系，防止误删除。

**注意**：这里记录的是`pg_class`中的外部表记录对SERVER的依赖，而不是`pg_foreign_table`对SERVER的依赖。这是因为外部表的主要对象是`pg_class`中的记录，`pg_foreign_table`只是辅助信息。

**系统表变化**：

| 系统表 | 操作 | 说明 |
|--------|------|------|
| `pg_class` | INSERT | 插入1条外部表记录（由`DefineRelation()`创建，`relkind='f'`） |
| `pg_attribute` | INSERT | 插入列定义记录（由`DefineRelation()`创建） |
| `pg_foreign_table` | INSERT | 插入1条外部表记录，包含ftrelid（关联pg_class）、ftserver（关联SERVER）、ftoptions等 |
| `pg_depend` | INSERT | 插入依赖关系记录（外部表（pg_class中的记录）对SERVER的依赖） |

**关联关系**：
- `pg_foreign_table.ftrelid` → `pg_class.oid`：关联外部表与`pg_class`
- `pg_foreign_table.ftserver` → `pg_foreign_server.oid`：关联外部表与SERVER
- 查询时通过这个关联链查找FDW的handler函数

**file_fdw vs mysql_fdw的区别**：
- **file_fdw**：CREATE FOREIGN TABLE的OPTIONS用于指定文件路径、格式等（如`filename='/path/to/file.csv'`, `format='csv'`）
- **mysql_fdw/postgres_fdw等**：CREATE FOREIGN TABLE的OPTIONS用于指定表级选项（如远程表名、列映射等）

**CREATE FOREIGN TABLE阶段总结**：
1. 先通过`DefineRelation()`创建`pg_class`和`pg_attribute`系统表条目（`relkind='f'`）
2. 验证SERVER存在且用户有权限使用
3. 注册外部表对象到`pg_foreign_table`系统表，建立与SERVER的关联关系
4. 处理并验证OPTIONS（如果提供了validator函数）
5. 建立依赖关系，确保SERVER不会被误删除
6. 为后续的查询操作做好准备

### 4.5 GDB调试验证：CREATE阶段只调用validator函数

通过GDB调试可以验证：在CREATE阶段（CREATE EXTENSION、CREATE SERVER、CREATE FOREIGN TABLE），**只有validator函数会被调用**，handler函数和其他FDW回调函数不会被调用。

#### 断点设置

使用GDB脚本设置8个断点，覆盖file_fdw的所有主要函数：

```gdb
# 断点列表
0. file_fdw_validator      (CREATE阶段)
1. file_fdw_handler        (规划阶段)
2. fileGetForeignRelSize   (规划阶段)
3. fileGetForeignPaths     (规划阶段)
4. fileGetForeignPlan      (规划阶段)
5. fileBeginForeignScan    (执行阶段)
6. fileIterateForeignScan  (执行阶段)
7. fileEndForeignScan      (执行阶段)
```

**GDB断点信息**：
```
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x000078faa7d786a0 in file_fdw_validator at file_fdw.c:199
2       breakpoint     keep y   0x000078faa7d785b4 in file_fdw_handler at file_fdw.c:175
3       breakpoint     keep y   0x000078faa7d7919f in fileGetForeignRelSize at file_fdw.c:525
4       breakpoint     keep y   0x000078faa7d79207 in fileGetForeignPaths at file_fdw.c:548
5       breakpoint     keep y   0x000078faa7d7932b in fileGetForeignPlan at file_fdw.c:606
6       breakpoint     keep y   0x000078faa7d7949e in fileBeginForeignScan at file_fdw.c:666
7       breakpoint     keep y   0x000078faa7d795b9 in fileIterateForeignScan at file_fdw.c:720
8       breakpoint     keep y   0x000078faa7d7978d in fileEndForeignScan at file_fdw.c:794
```

#### 实际测试：CREATE阶段

**psql中执行的SQL命令**：

```sql
unvdb=# CREATE EXTENSION IF NOT EXISTS file_fdw;
CREATE EXTENSION

unvdb=# CREATE SERVER IF NOT EXISTS file_server FOREIGN DATA WRAPPER file_fdw;
CREATE SERVER

unvdb=# CREATE FOREIGN TABLE IF NOT EXISTS test_foreign_table (
unvdb(#     col1 INTEGER,
unvdb(#     col2 TEXT,
unvdb(#     col3 INTEGER
unvdb(# ) SERVER file_server
unvdb-# OPTIONS (
unvdb(#     filename '/home/unvdb/work/test_foreign_table.csv',
unvdb(#     format 'csv',
unvdb(#     header 'true'
unvdb(# );
CREATE FOREIGN TABLE
```

**GDB调试输出**：

**第1次触发validator（CREATE EXTENSION）**：
```
Breakpoint 1, file_fdw_validator (fcinfo=0x7ffc54c120b0) at file_fdw.c:199
199     {

========================================
=== [CREATE阶段] file_fdw_validator 被调用 ===
========================================
调用栈（前10帧）：
#0  file_fdw_validator (fcinfo=0x7ffc54c120b0) at file_fdw.c:199
#1  0x0000000000bdf227 in FunctionCall2Coll (flinfo=0x7ffc54c12130, collation=0, arg1=101402808, arg2=2328) at fmgr.c:1132
#2  0x0000000000be0059 in OidFunctionCall2Coll (functionId=16390, collation=0, arg1=101402808, arg2=2328) at fmgr.c:1398
#3  0x00000000006bc45b in transformGenericOptions (catalogId=2328, oldOptions=0, options=0x0, fdwvalidator=16390) at foreigncmds.c:191
#4  0x00000000006bd2ff in CreateForeignDataWrapper (pstate=0x60b4418, stmt=0x60aabc0) at foreigncmds.c:619
#5  0x0000000000a054a9 in ProcessUtilitySlow (...) at utility.c:1591
#6  0x00000000006b5d98 in execute_sql_string (...) at extension.c:818
#7  0x00000000006b6a1e in execute_extension_script (...) at extension.c:1117
...
```

**第2次触发validator（CREATE SERVER）**：
```
Breakpoint 1, file_fdw_validator (fcinfo=0x7ffc54c12a80) at file_fdw.c:199
199     {

========================================
=== [CREATE阶段] file_fdw_validator 被调用 ===
========================================
调用栈（前10帧）：
#0  file_fdw_validator (fcinfo=0x7ffc54c12a80) at file_fdw.c:199
#1  0x0000000000bdf227 in FunctionCall2Coll (...) at fmgr.c:1132
#2  0x0000000000be0059 in OidFunctionCall2Coll (...) at fmgr.c:1398
#3  0x00000000006bc45b in transformGenericOptions (catalogId=1417, oldOptions=0, options=0x0, fdwvalidator=16390) at foreigncmds.c:191
#4  0x00000000006bdc9e in CreateForeignServer (stmt=0x5fd8b40) at foreigncmds.c:930
#5  0x0000000000a0550b in ProcessUtilitySlow (...) at utility.c:1599
...
```

**第3次触发validator（CREATE FOREIGN TABLE）**：
```
Breakpoint 1, file_fdw_validator (fcinfo=0x7ffc54c12ad0) at file_fdw.c:199
199     {

========================================
=== [CREATE阶段] file_fdw_validator 被调用 ===
========================================
调用栈（前10帧）：
#0  file_fdw_validator (fcinfo=0x7ffc54c12ad0) at file_fdw.c:199
#1  0x0000000000bdf227 in FunctionCall2Coll (...) at fmgr.c:1132
#2  0x0000000000be0059 in OidFunctionCall2Coll (...) at fmgr.c:1398
#3  0x00000000006bc45b in transformGenericOptions (catalogId=3118, oldOptions=0, options=0x5fd91e0, fdwvalidator=16390) at foreigncmds.c:191
#4  0x00000000006bee5e in CreateForeignTable (stmt=0x5fd9370, relid=16393) at foreigncmds.c:1451
#5  0x0000000000a046dc in ProcessUtilitySlow (...) at utility.c:1218
...
```

**断点命中统计**：

执行完CREATE命令后，检查断点命中次数：

```gdb
(gdb) info breakpoints 1
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x000078faa7d786a0 in file_fdw_validator at file_fdw.c:199
        breakpoint already hit 3 times
```

**验证结果**：

1. ✅ **validator函数被调用了3次**：
   - 第1次：`CREATE EXTENSION` → 执行`file_fdw--1.0.sql` → `CREATE FOREIGN DATA WRAPPER file_fdw` → 调用validator
   - 第2次：`CREATE SERVER` → 调用validator（即使没有OPTIONS也会调用，传入空数组）
   - 第3次：`CREATE FOREIGN TABLE` → 调用validator（验证OPTIONS参数）

2. ✅ **handler函数和其他FDW函数未被调用**：
   - 断点2-8（handler、GetForeignRelSize、GetForeignPaths等）均未触发
   - 说明CREATE阶段只涉及元数据操作，不涉及查询规划或执行

3. ✅ **validator调用位置**：
   - 所有validator调用都通过`transformGenericOptions()`函数
   - 调用路径：`CreateForeignDataWrapper/CreateForeignServer/CreateForeignTable` → `transformGenericOptions` → `OidFunctionCall2Coll` → `file_fdw_validator`

#### 重要发现

**即使CREATE命令没有提供OPTIONS，validator函数也会被调用**：

根据源码`foreigncmds.c:181-191`：

```c
if (OidIsValid(fdwvalidator))
{
    Datum		valarg = result;
    
    /*
     * Pass a null options list as an empty array, so that validators
     * don't have to be declared non-strict to handle the case.
     */
    if (DatumGetPointer(valarg) == NULL)
        valarg = PointerGetDatum(construct_empty_array(TEXTOID));
    OidFunctionCall2(fdwvalidator, valarg, ObjectIdGetDatum(catalogId));
}
```

即使OPTIONS为空（NULL），PostgreSQL也会构造一个空数组传给validator，这样validator不需要声明为non-strict也能处理空选项的情况。

#### CREATE阶段总结

- **只调用validator函数**：用于验证OPTIONS参数（即使为空也会调用）
- **不调用handler函数**：handler函数只在查询规划阶段被调用
- **不调用FDW回调函数**：GetForeignRelSize、GetForeignPaths等只在查询规划/执行阶段被调用
- **目的**：CREATE阶段只是注册元数据，不涉及实际的数据访问

### 4.6 SELECT查询阶段（使用阶段）

通过GDB调试可以验证：在执行`EXPLAIN SELECT`或`SELECT`查询时，会依次调用handler函数和FDW的规划/执行回调函数。

**重要说明**：与CREATE阶段不同，SELECT查询阶段没有"同名函数"的概念。我们需要从**查询执行流程**的角度来理解：PostgreSQL的查询执行分为两个阶段——**规划阶段（Planner）**和**执行阶段（Executor）**。FDW的函数是在这两个阶段中被调用的。

#### 4.6.1 EXPLAIN查询测试

**测试SQL命令**：

```sql
unvdb=# EXPLAIN SELECT * FROM test_foreign_table WHERE col1 > 2;
                              QUERY PLAN                               
-----------------------------------------------------------------------
 Foreign Scan on test_foreign_table  (cost=0.00..1.10 rows=1 width=40)
   Filter: (col1 > 2)
   Foreign File: /home/unvdb/work/test_foreign_table.csv
   Foreign File Size: 80 b
(4 rows)
```

**GDB调试输出**：

**断点2：file_fdw_handler（规划阶段开始时调用）**
```
Breakpoint 2, file_fdw_handler (fcinfo=0x7ffc54c12640) at file_fdw.c:175
175             FdwRoutine *fdwroutine = makeNode(FdwRoutine);

========================================
=== [规划阶段] file_fdw_handler 被调用 ===
========================================
调用栈（前10帧）：
#0  file_fdw_handler (fcinfo=0x7ffc54c12640) at file_fdw.c:175
#1  0x0000000000bdf013 in FunctionCall0Coll (flinfo=0x7ffc54c12690, collation=0) at fmgr.c:1090
#2  0x0000000000bdff99 in OidFunctionCall0Coll (functionId=16389, collation=0) at fmgr.c:1378
#3  0x00000000007c5b49 in GetFdwRoutine (fdwhandler=16389) at foreign.c:326
#4  0x00000000007c5df4 in GetFdwRoutineByServerId (serverid=16392) at foreign.c:397
#5  0x00000000007c5e18 in GetFdwRoutineByRelId (relid=16393) at foreign.c:414
#6  0x00000000007c5e49 in GetFdwRoutineForRelation (relation=0x78faa7dd4038, makecopy=true) at foreign.c:437
#7  0x00000000008fab28 in get_relation_info (root=0x60d9798, relationObjectId=16393, inhparent=false, rel=0x60d9438) at plancat.c:504
#8  0x0000000000901ecb in build_simple_rel (root=0x60d9798, relid=1, parent=0x0) at relnode.c:338
#9  0x00000000008b56f5 in add_base_rels_to_query (root=0x60d9798, jtnode=0x5fd9dd8) at initsplan.c:166
```

**断点3：fileGetForeignRelSize（规划阶段：估算表大小）**
```
Breakpoint 3, fileGetForeignRelSize (root=0x60d9798, baserel=0x60d9438, foreigntableid=16393) at file_fdw.c:525
525             fdw_private = (FileFdwPlanState *) palloc(sizeof(FileFdwPlanState));

========================================
=== [规划阶段] fileGetForeignRelSize 被调用 ===
========================================
调用栈（前10帧）：
#0  fileGetForeignRelSize (root=0x60d9798, baserel=0x60d9438, foreigntableid=16393) at file_fdw.c:525
#1  0x000000000087bb72 in set_foreign_size (root=0x60d9798, rel=0x60d9438, rte=0x5fd9270) at allpaths.c:911
#2  0x000000000087b1fb in set_rel_size (root=0x60d9798, rel=0x60d9438, rti=1, rte=0x5fd9270) at allpaths.c:395
#3  0x000000000087b07f in set_base_rel_sizes (root=0x60d9798) at allpaths.c:325
#4  0x000000000087ad7b in make_one_rel (root=0x60d9798, joinlist=0x60da6b8) at allpaths.c:186
#5  0x00000000008bb8d6 in query_planner (root=0x60d9798, qp_callback=0x8c1f89 <standard_qp_callback>, qp_extra=0x7ffc54c12b50) at planmain.c:278
#6  0x00000000008be29a in grouping_planner (root=0x60d9798, tuple_fraction=0) at planner.c:1495
#7  0x00000000008bd978 in subquery_planner (glob=0x60d8e18, parse=0x5fd9160, parent_root=0x0, hasRecursion=false, tuple_fraction=0) at planner.c:1064
#8  0x00000000008bbf74 in standard_planner (parse=0x5fd9160, query_string=0x5fd80d8 "EXPLAIN SELECT * FROM test_foreign_table WHERE col1 > 2;", cursorOptions=2048, boundParams=0x0) at planner.c:413
#9  0x00000000008bbcaf in planner (parse=0x5fd9160, query_string=0x5fd80d8 "EXPLAIN SELECT * FROM test_foreign_table WHERE col1 > 2;", cursorOptions=2048, boundParams=0x0) at planner.c:281
```

**断点4：fileGetForeignPaths（规划阶段：生成访问路径）**
```
Breakpoint 4, fileGetForeignPaths (root=0x60d9798, baserel=0x60d9438, foreigntableid=16393) at file_fdw.c:548
548     {

========================================
=== [规划阶段] fileGetForeignPaths 被调用 ===
========================================
调用栈（前10帧）：
#0  fileGetForeignPaths (root=0x60d9798, baserel=0x60d9438, foreigntableid=16393) at file_fdw.c:548
#1  0x000000000087bc0d in set_foreign_pathlist (root=0x60d9798, rel=0x60d9438, rte=0x5fd9270) at allpaths.c:932
#2  0x000000000087b45a in set_rel_pathlist (root=0x60d9798, rel=0x60d9438, rti=1, rte=0x5fd9270) at allpaths.c:492
#3  0x000000000087b125 in set_base_rel_pathlists (root=0x60d9798) at allpaths.c:354
#4  0x000000000087ae75 in make_one_rel (root=0x60d9798, joinlist=0x60da6b8) at allpaths.c:224
#5  0x00000000008bb8d6 in query_planner (root=0x60d9798, qp_callback=0x8c1f89 <standard_qp_callback>, qp_extra=0x7ffc54c12b50) at planmain.c:278
#6  0x00000000008be29a in grouping_planner (root=0x60d9798, tuple_fraction=0) at planner.c:1495
#7  0x00000000008bd978 in subquery_planner (glob=0x60d8e18, parse=0x5fd9160, parent_root=0x0, hasRecursion=false, tuple_fraction=0) at planner.c:1064
#8  0x00000000008bbf74 in standard_planner (parse=0x5fd9160, query_string=0x5fd80d8 "EXPLAIN SELECT * FROM test_foreign_table WHERE col1 > 2;", cursorOptions=2048, boundParams=0x0) at planner.c:413
#9  0x00000000008bbcaf in planner (parse=0x5fd9160, query_string=0x5fd80d8 "EXPLAIN SELECT * FROM test_foreign_table WHERE col1 > 2;", cursorOptions=2048, boundParams=0x0) at planner.c:281
```

**断点5：fileGetForeignPlan（规划阶段：创建查询计划节点）**
```
Breakpoint 5, fileGetForeignPlan (root=0x60d9798, baserel=0x60d9438, foreigntableid=16393, best_path=0x6114250, tlist=0x61146c0, scan_clauses=0x61136d8, outer_plan=0x0) at file_fdw.c:606
606             Index           scan_relid = baserel->relid;

========================================
=== [规划阶段] fileGetForeignPlan 被调用 ===
========================================
调用栈（前10帧）：
#0  fileGetForeignPlan (root=0x60d9798, baserel=0x60d9438, foreigntableid=16393, best_path=0x6114250, tlist=0x61146c0, scan_clauses=0x61136d8, outer_plan=0x0) at file_fdw.c:606
#1  0x00000000008af342 in create_foreignscan_plan (root=0x60d9798, best_path=0x6114250, tlist=0x61146c0, scan_clauses=0x61136d8) at createplan.c:4138
#2  0x00000000008a8a51 in create_scan_plan (root=0x60d9798, best_path=0x6114250, flags=1) at createplan.c:766
#3  0x00000000008a82a7 in create_plan_recurse (root=0x60d9798, best_path=0x6114250, flags=1) at createplan.c:411
#4  0x00000000008a81b9 in create_plan (root=0x60d9798, best_path=0x6114250) at createplan.c:347
#5  0x00000000008bbfd6 in standard_planner (parse=0x5fd9160, query_string=0x5fd80d8 "EXPLAIN SELECT * FROM test_foreign_table WHERE col1 > 2;", cursorOptions=2048, boundParams=0x0) at planner.c:420
#6  0x00000000008bbcaf in planner (parse=0x5fd9160, query_string=0x5fd80d8 "EXPLAIN SELECT * FROM test_foreign_table WHERE col1 > 2;", cursorOptions=2048, boundParams=0x0) at planner.c:281
#7  0x00000000009fa4a7 in pg_plan_query (querytree=0x5fd9160, query_string=0x5fd80d8 "EXPLAIN SELECT * FROM test_foreign_table WHERE col1 > 2;", cursorOptions=2048, boundParams=0x0) at postgres.c:904
#8  0x00000000006a92d9 in ExplainOneQuery (query=0x5fd9160, cursorOptions=2048, into=0x0, es=0x60d8b78, queryString=0x5fd80d8 "EXPLAIN SELECT * FROM test_foreign_table WHERE col1 > 2;", params=0x0, queryEnv=0x0) at explain.c:406
#9  0x00000000006a8df6 in ExplainQuery (pstate=0x6119d70, stmt=0x5fd8fa0, params=0x0, dest=0x6119ce0) at explain.c:290
```

**断点6：fileBeginForeignScan（执行阶段：初始化扫描）**
```
Breakpoint 6, fileBeginForeignScan (node=0x611b9d0, eflags=33) at file_fdw.c:666
666     {

========================================
=== [执行阶段] fileBeginForeignScan 被调用 ===
========================================
调用栈（前10帧）：
#0  fileBeginForeignScan (node=0x611b9d0, eflags=33) at file_fdw.c:666
#1  0x00000000007871db in ExecInitForeignScan (node=0x60da198, estate=0x611b788, eflags=33) at nodeForeignscan.c:286
#2  0x000000000076904b in ExecInitNode (node=0x60da198, estate=0x611b788, eflags=33) at execProcnode.c:285
#3  0x000000000075e407 in InitPlan (queryDesc=0x6114980, eflags=33) at execMain.c:968
#4  0x000000000075d2e1 in standard_ExecutorStart (queryDesc=0x6114980, eflags=33) at execMain.c:266
#5  0x000000000075d056 in ExecutorStart (queryDesc=0x6114980, eflags=1) at execMain.c:145
#6  0x00000000006a9854 in ExplainOnePlan (plannedstmt=0x6113a48, into=0x0, es=0x60d8b78, queryString=0x5fd80d8 "EXPLAIN SELECT * FROM test_foreign_table WHERE col1 > 2;", params=0x0, queryEnv=0x0, planduration=0x7ffc54c12e60, bufusage=0x0) at explain.c:590
#7  0x00000000006a9396 in ExplainOneQuery (query=0x5fd9160, cursorOptions=2048, into=0x0, es=0x60d8b78, queryString=0x5fd80d8 "EXPLAIN SELECT * FROM test_foreign_table WHERE col1 > 2;", params=0x0, queryEnv=0x0) at explain.c:419
#8  0x00000000006a8df6 in ExplainQuery (pstate=0x6119d70, stmt=0x5fd8fa0, params=0x0, dest=0x6119ce0) at explain.c:290
#9  0x0000000000a03ca5 in standard_ProcessUtility (pstmt=0x5fd9050, queryString=0x5fd80d8 "EXPLAIN SELECT * FROM test_foreign_table WHERE col1 > 2;", readOnlyTree=false, context=PROCESS_UTILITY_TOPLEVEL, params=0x0, queryEnv=0x0, dest=0x6119ce0, qc=0x7ffc54c13250) at utility.c:870
```

**断点8：fileEndForeignScan（执行阶段：结束扫描）**
```
Breakpoint 8, fileEndForeignScan (node=0x611b9d0) at file_fdw.c:794
794             FileFdwExecutionState *festate = (FileFdwExecutionState *) node->fdw_state;

========================================
=== [执行阶段] fileEndForeignScan 被调用 ===
========================================
调用栈（前10帧）：
#0  fileEndForeignScan (node=0x611b9d0) at file_fdw.c:794
#1  0x0000000000787256 in ExecEndForeignScan (node=0x611b9d0) at nodeForeignscan.c:310
#2  0x00000000007697f2 in ExecEndNode (node=0x611b9d0) at execProcnode.c:687
#3  0x000000000075f761 in ExecEndPlan (planstate=0x611b9d0, estate=0x611b788) at execMain.c:1509
#4  0x000000000075d810 in standard_ExecutorEnd (queryDesc=0x6114980) at execMain.c:503
#5  0x000000000075d75a in ExecutorEnd (queryDesc=0x6114980) at execMain.c:474
#6  0x00000000006a9a02 in ExplainOnePlan (plannedstmt=0x6113a48, into=0x0, es=0x60d8b78, queryString=0x5fd80d8 "EXPLAIN SELECT * FROM test_foreign_table WHERE col1 > 2;", params=0x0, queryEnv=0x0, planduration=0x7ffc54c12e60, bufusage=0x0) at explain.c:652
#7  0x00000000006a9396 in ExplainOneQuery (query=0x5fd9160, cursorOptions=2048, into=0x0, es=0x60d8b78, queryString=0x5fd80d8 "EXPLAIN SELECT * FROM test_foreign_table WHERE col1 > 2;", params=0x0, queryEnv=0x0) at explain.c:419
#8  0x00000000006a8df6 in ExplainQuery (pstate=0x6119d70, stmt=0x5fd8fa0, params=0x0, dest=0x6119ce0) at explain.c:290
#9  0x0000000000a03ca5 in standard_ProcessUtility (pstmt=0x5fd9050, queryString=0x5fd80d8 "EXPLAIN SELECT * FROM test_foreign_table WHERE col1 > 2;", params=0x0, queryEnv=0x0, dest=0x6119ce0, qc=0x7ffc54c13250) at utility.c:870
```

**断点命中统计**：

执行完`EXPLAIN SELECT`查询后，检查断点命中次数：

```gdb
(gdb) info breakpoints
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x000078faa7d786a0 in file_fdw_validator at file_fdw.c:199
        breakpoint already hit 3 times
2       breakpoint     keep y   0x000078faa7d785b4 in file_fdw_handler at file_fdw.c:175
        breakpoint already hit 1 time
3       breakpoint     keep y   0x000078faa7d7919f in fileGetForeignRelSize at file_fdw.c:525
        breakpoint already hit 1 time
4       breakpoint     keep y   0x000078faa7d79207 in fileGetForeignPaths at file_fdw.c:548
        breakpoint already hit 1 time
5       breakpoint     keep y   0x000078faa7d7932b in fileGetForeignPlan at file_fdw.c:606
        breakpoint already hit 1 time
6       breakpoint     keep y   0x000078faa7d7949e in fileBeginForeignScan at file_fdw.c:666
        breakpoint already hit 1 time
7       breakpoint     keep y   0x000078faa7d795b9 in fileIterateForeignScan at file_fdw.c:720
        (未命中)
8       breakpoint     keep y   0x000078faa7d7978d in fileEndForeignScan at file_fdw.c:794
        breakpoint already hit 1 time
```

**验证结果**：

1. ✅ **规划阶段的4个函数都被调用**：
   - `file_fdw_handler`：1次（获取FdwRoutine结构体）
   - `fileGetForeignRelSize`：1次（估算表大小）
   - `fileGetForeignPaths`：1次（生成访问路径）
   - `fileGetForeignPlan`：1次（创建查询计划节点）

2. ✅ **执行阶段的初始化和结束函数被调用**：
   - `fileBeginForeignScan`：1次（初始化扫描）
   - `fileEndForeignScan`：1次（结束扫描）

3. ⚠️ **执行阶段的迭代函数未被调用**：
   - `fileIterateForeignScan`：0次（未命中）

**为什么`fileIterateForeignScan`没有被调用？**

`EXPLAIN`命令的作用是显示查询计划，而不是实际执行查询。虽然`EXPLAIN`会执行规划阶段和部分执行阶段（用于获取统计信息，如文件大小），但**不会实际迭代数据**。因此`fileIterateForeignScan`不会被调用。

要看到`fileIterateForeignScan`被调用，需要执行实际的`SELECT`查询（不带`EXPLAIN`），此时会循环调用`fileIterateForeignScan`来获取每一行数据。

**EXPLAIN vs SELECT的区别**：

| 命令 | 规划阶段 | 执行阶段初始化 | 执行阶段迭代 | 返回数据 |
|------|---------|--------------|------------|---------|
| `EXPLAIN SELECT ...` | ✅ 执行 | ✅ 执行（获取统计信息） | ❌ 不执行 | ❌ 只显示计划 |
| `SELECT ...` | ✅ 执行 | ✅ 执行 | ✅ 执行（循环获取数据） | ✅ 返回实际数据 |

#### 4.6.2 SELECT查询测试

执行实际的`SELECT`查询时，会看到`fileIterateForeignScan`被循环调用，每次返回一行数据。与`EXPLAIN`查询的区别在于，`SELECT`会实际迭代数据，因此`fileIterateForeignScan`会被多次调用。

**测试SQL命令**：

```sql
unvdb=# SELECT * FROM test_foreign_table WHERE col1 > 2;
 col1 |  col2  | col3 
------+--------+------
    3 | value3 |  300
    4 | value4 |  400
    5 | value5 |  500
(3 rows)
```

**GDB调试输出**：

**规划阶段**（与EXPLAIN相同，handler函数可能已在之前调用过，这里从GetForeignRelSize开始）：

**断点3：fileGetForeignRelSize（规划阶段：估算表大小）**
```
Breakpoint 3, fileGetForeignRelSize (root=0x6113698, baserel=0x5fd9d20, foreigntableid=16393) at file_fdw.c:525
525             fdw_private = (FileFdwPlanState *) palloc(sizeof(FileFdwPlanState));

========================================
=== [规划阶段] fileGetForeignRelSize 被调用 ===
========================================
调用栈（前10帧）：
#0  fileGetForeignRelSize (root=0x6113698, baserel=0x5fd9d20, foreigntableid=16393) at file_fdw.c:525
#1  0x000000000087bb72 in set_foreign_size (root=0x6113698, rel=0x5fd9d20, rte=0x5fd9240) at allpaths.c:911
#2  0x000000000087b1fb in set_rel_size (root=0x6113698, rel=0x5fd9d20, rti=1, rte=0x5fd9240) at allpaths.c:395
#3  0x000000000087b07f in set_base_rel_sizes (root=0x6113698) at allpaths.c:325
#4  0x000000000087ad7b in make_one_rel (root=0x6113698, joinlist=0x61145e8) at allpaths.c:186
#5  0x00000000008bb8d6 in query_planner (root=0x6113698, qp_callback=0x8c1f89 <standard_qp_callback>, qp_extra=0x7ffc54c130d0) at planmain.c:278
#6  0x00000000008be29a in grouping_planner (root=0x6113698, tuple_fraction=0) at planner.c:1495
#7  0x00000000008bd978 in subquery_planner (glob=0x5fd9020, parse=0x5fd9130, parent_root=0x0, hasRecursion=false, tuple_fraction=0) at planner.c:1064
#8  0x00000000008bbf74 in standard_planner (parse=0x5fd9130, query_string=0x5fd80d8 "SELECT * FROM test_foreign_table WHERE col1 > 2;", cursorOptions=2048, boundParams=0x0) at planner.c:413
#9  0x00000000008bbcaf in planner (parse=0x5fd9130, query_string=0x5fd80d8 "SELECT * FROM test_foreign_table WHERE col1 > 2;", cursorOptions=2048, boundParams=0x0) at planner.c:281
```

**断点4：fileGetForeignPaths（规划阶段：生成访问路径）**
```
Breakpoint 4, fileGetForeignPaths (root=0x6113698, baserel=0x5fd9d20, foreigntableid=16393) at file_fdw.c:548
548     {

========================================
=== [规划阶段] fileGetForeignPaths 被调用 ===
========================================
调用栈（前10帧）：
#0  fileGetForeignPaths (root=0x6113698, baserel=0x5fd9d20, foreigntableid=16393) at file_fdw.c:548
#1  0x000000000087bc0d in set_foreign_pathlist (root=0x6113698, rel=0x5fd9d20, rte=0x5fd9240) at allpaths.c:932
#2  0x000000000087b45a in set_rel_pathlist (root=0x6113698, rel=0x5fd9d20, rti=1, rte=0x5fd9240) at allpaths.c:492
#3  0x000000000087b125 in set_base_rel_pathlists (root=0x6113698) at allpaths.c:354
#4  0x000000000087ae75 in make_one_rel (root=0x6113698, joinlist=0x61145e8) at allpaths.c:224
#5  0x00000000008bb8d6 in query_planner (root=0x6113698, qp_callback=0x8c1f89 <standard_qp_callback>, qp_extra=0x7ffc54c130d0) at planmain.c:278
#6  0x00000000008be29a in grouping_planner (root=0x6113698, tuple_fraction=0) at planner.c:1495
#7  0x00000000008bd978 in subquery_planner (glob=0x5fd9020, parse=0x5fd9130, parent_root=0x0, hasRecursion=false, tuple_fraction=0) at planner.c:1064
#8  0x00000000008bbf74 in standard_planner (parse=0x5fd9130, query_string=0x5fd80d8 "SELECT * FROM test_foreign_table WHERE col1 > 2;", cursorOptions=2048, boundParams=0x0) at planner.c:413
#9  0x00000000008bbcaf in planner (parse=0x5fd9130, query_string=0x5fd80d8 "SELECT * FROM test_foreign_table WHERE col1 > 2;", cursorOptions=2048, boundParams=0x0) at planner.c:281
```

**断点5：fileGetForeignPlan（规划阶段：创建查询计划节点）**
```
Breakpoint 5, fileGetForeignPlan (root=0x6113698, baserel=0x5fd9d20, foreigntableid=16393, best_path=0x6114f90, tlist=0x6115610, scan_clauses=0x61147e8, outer_plan=0x0) at file_fdw.c:606
606             Index           scan_relid = baserel->relid;

========================================
=== [规划阶段] fileGetForeignPlan 被调用 ===
========================================
调用栈（前10帧）：
#0  fileGetForeignPlan (root=0x6113698, baserel=0x5fd9d20, foreigntableid=16393, best_path=0x6114f90, tlist=0x6115610, scan_clauses=0x61147e8, outer_plan=0x0) at file_fdw.c:606
#1  0x00000000008af342 in create_foreignscan_plan (root=0x6113698, best_path=0x6114f90, tlist=0x6115610, scan_clauses=0x61147e8) at createplan.c:4138
#2  0x00000000008a8a51 in create_scan_plan (root=0x6113698, best_path=0x6114f90, flags=1) at createplan.c:766
#3  0x00000000008a82a7 in create_plan_recurse (root=0x6113698, best_path=0x6114f90, flags=1) at createplan.c:411
#4  0x00000000008a81b9 in create_plan (root=0x6113698, best_path=0x6114f90) at createplan.c:347
#5  0x00000000008bbfd6 in standard_planner (parse=0x5fd9130, query_string=0x5fd80d8 "SELECT * FROM test_foreign_table WHERE col1 > 2;", cursorOptions=2048, boundParams=0x0) at planner.c:420
#6  0x00000000008bbcaf in planner (parse=0x5fd9130, query_string=0x5fd80d8 "SELECT * FROM test_foreign_table WHERE col1 > 2;", cursorOptions=2048, boundParams=0x0) at planner.c:281
#7  0x00000000009fa4a7 in pg_plan_query (querytree=0x5fd9130, query_string=0x5fd80d8 "SELECT * FROM test_foreign_table WHERE col1 > 2;", cursorOptions=2048, boundParams=0x0) at postgres.c:904
#8  0x00000000009fa5f3 in pg_plan_queries (querytrees=0x5fd9cd0, query_string=0x5fd80d8 "SELECT * FROM test_foreign_table WHERE col1 > 2;", cursorOptions=2048, boundParams=0x0) at postgres.c:996
#9  0x00000000009fa9e4 in exec_simple_query (query_string=0x5fd80d8 "SELECT * FROM test_foreign_table WHERE col1 > 2;") at postgres.c:1193
```

**执行阶段**（完整的执行流程）：

**断点6：fileBeginForeignScan（执行阶段：初始化扫描）**
```
Breakpoint 6, fileBeginForeignScan (node=0x60ccf20, eflags=32) at file_fdw.c:666
666     {

========================================
=== [执行阶段] fileBeginForeignScan 被调用 ===
========================================
调用栈（前10帧）：
#0  fileBeginForeignScan (node=0x60ccf20, eflags=32) at file_fdw.c:666
#1  0x00000000007871db in ExecInitForeignScan (node=0x60cecf8, estate=0x60cccd8, eflags=32) at nodeForeignscan.c:286
#2  0x000000000076904b in ExecInitNode (node=0x60cecf8, estate=0x60cccd8, eflags=32) at execProcnode.c:285
#3  0x000000000075e407 in InitPlan (queryDesc=0x6119bb8, eflags=32) at execMain.c:968
#4  0x000000000075d2e1 in standard_ExecutorStart (queryDesc=0x6119bb8, eflags=32) at execMain.c:266
#5  0x000000000075d056 in ExecutorStart (queryDesc=0x6119bb8, eflags=0) at execMain.c:145
#6  0x0000000000a00eb3 in PortalStart (portal=0x6052e78, params=0x0, eflags=0, snapshot=0x0) at pquery.c:517
#7  0x00000000009faa77 in exec_simple_query (query_string=0x5fd80d8 "SELECT * FROM test_foreign_table WHERE col1 > 2;") at postgres.c:1235
#8  0x00000000009ff4f1 in PostgresMain (dbname=0x6010dd8 "unvdb", username=0x5fd2828 "unvdb") at postgres.c:4637
#9  0x000000000092d564 in BackendRun (port=0x60089b0) at postmaster.c:4587
```

**断点7：fileIterateForeignScan（执行阶段：循环获取数据，被调用多次）**

第1次调用（返回第1行数据：col1=3）：
```
Breakpoint 7, fileIterateForeignScan (node=0x60ccf20) at file_fdw.c:720
720     {

========================================
=== [执行阶段] fileIterateForeignScan 被调用 ===
========================================
调用栈（前5帧）：
#0  fileIterateForeignScan (node=0x60ccf20) at file_fdw.c:720
#1  0x0000000000786cfb in ForeignNext (node=0x60ccf20) at nodeForeignscan.c:62
#2  0x000000000076d6be in ExecScanFetch (node=0x60ccf20, accessMtd=0x786c53 <ForeignNext>, recheckMtd=0x786d4e <ForeignRecheck>) at execScan.c:132
#3  0x000000000076d75f in ExecScan (node=0x60ccf20, accessMtd=0x786c53 <ForeignNext>, recheckMtd=0x786d4e <ForeignRecheck>) at execScan.c:198
#4  0x0000000000786e5f in ExecForeignScan (pstate=0x60ccf20) at nodeForeignscan.c:132
```

第2次调用（返回第2行数据：col1=4）：
```
Breakpoint 7, fileIterateForeignScan (node=0x60ccf20) at file_fdw.c:720
720     {
...
（调用栈相同）
```

第3次调用（返回第3行数据：col1=5）：
```
Breakpoint 7, fileIterateForeignScan (node=0x60ccf20) at file_fdw.c:720
720     {
...
（调用栈相同）
```

第4次调用（返回NULL，表示没有更多数据）：
```
Breakpoint 7, fileIterateForeignScan (node=0x60ccf20) at file_fdw.c:720
720     {
...
（调用栈相同，但返回NULL）
```

**断点8：fileEndForeignScan（执行阶段：结束扫描）**
```
Breakpoint 8, fileEndForeignScan (node=0x60ccf20) at file_fdw.c:794
794             FileFdwExecutionState *festate = (FileFdwExecutionState *) node->fdw_state;

========================================
=== [执行阶段] fileEndForeignScan 被调用 ===
========================================
调用栈（前10帧）：
#0  fileEndForeignScan (node=0x60ccf20) at file_fdw.c:794
#1  0x0000000000787256 in ExecEndForeignScan (node=0x60ccf20) at nodeForeignscan.c:310
#2  0x00000000007697f2 in ExecEndNode (node=0x60ccf20) at execProcnode.c:687
#3  0x000000000075f761 in ExecEndPlan (planstate=0x60ccf20, estate=0x60cccd8) at execMain.c:1509
#4  0x000000000075d810 in standard_ExecutorEnd (queryDesc=0x6119bb8) at execMain.c:503
#5  0x000000000075d75a in ExecutorEnd (queryDesc=0x6119bb8) at execMain.c:474
#6  0x00000000006d8a5d in PortalCleanup (portal=0x6052e78) at portalcmds.c:299
#7  0x0000000000c155a7 in PortalDrop (portal=0x6052e78, isTopCommit=false) at portalmem.c:503
#8  0x00000000009fab95 in exec_simple_query (query_string=0x5fd80d8 "SELECT * FROM test_foreign_table WHERE col1 > 2;") at postgres.c:1284
#9  0x00000000009ff4f1 in PostgresMain (dbname=0x6010dd8 "unvdb", username=0x5fd2828 "unvdb") at postgres.c:4637
```

**验证结果**：

1. ✅ **规划阶段的函数都被调用**（与EXPLAIN相同）：
   - `fileGetForeignRelSize`：1次
   - `fileGetForeignPaths`：1次
   - `fileGetForeignPlan`：1次

2. ✅ **执行阶段的函数都被调用**：
   - `fileBeginForeignScan`：1次（初始化扫描，打开文件）
   - `fileIterateForeignScan`：**4次**（前3次返回数据行，第4次返回NULL表示结束）
   - `fileEndForeignScan`：1次（结束扫描，关闭文件）

3. ✅ **`fileIterateForeignScan`的调用次数**：
   - 查询返回3行数据（col1=3, 4, 5）
   - `fileIterateForeignScan`被调用了4次：
     - 第1-3次：每次返回一行数据
     - 第4次：返回NULL，表示没有更多数据

**SELECT查询的调用顺序总结**：

1. **规划阶段**（3-4个函数，按顺序调用）：
   - `file_fdw_handler` → 获取FdwRoutine结构体（可能已在之前调用）
   - `fileGetForeignRelSize` → 估算表大小和行数
   - `fileGetForeignPaths` → 生成访问路径
   - `fileGetForeignPlan` → 创建查询计划节点

2. **执行阶段**（3个函数，完整执行）：
   - `fileBeginForeignScan` → 初始化扫描（打开文件）
   - `fileIterateForeignScan` → **循环调用**（每次返回一行数据，直到返回NULL）
   - `fileEndForeignScan` → 结束扫描（关闭文件）

**断点命中统计（累计）**：

执行完`EXPLAIN SELECT`和`SELECT`查询后，检查所有断点的累计命中次数：

```gdb
(gdb) info breakpoints
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x000078faa7d786a0 in file_fdw_validator at file_fdw.c:199
        breakpoint already hit 3 times        # CREATE阶段：3次
2       breakpoint     keep y   0x000078faa7d785b4 in file_fdw_handler at file_fdw.c:175
        breakpoint already hit 1 time         # EXPLAIN查询：1次
3       breakpoint     keep y   0x000078faa7d7919f in fileGetForeignRelSize at file_fdw.c:525
        breakpoint already hit 2 times         # EXPLAIN 1次 + SELECT 1次
4       breakpoint     keep y   0x000078faa7d79207 in fileGetForeignPaths at file_fdw.c:548
        breakpoint already hit 2 times         # EXPLAIN 1次 + SELECT 1次
5       breakpoint     keep y   0x000078faa7d7932b in fileGetForeignPlan at file_fdw.c:606
        breakpoint already hit 2 times         # EXPLAIN 1次 + SELECT 1次
6       breakpoint     keep y   0x000078faa7d7949e in fileBeginForeignScan at file_fdw.c:666
        breakpoint already hit 2 times         # EXPLAIN 1次 + SELECT 1次
7       breakpoint     keep y   0x000078faa7d795b9 in fileIterateForeignScan at file_fdw.c:720
        breakpoint already hit 6 times         # SELECT查询：4次（实际返回3行数据+1次NULL）
8       breakpoint     keep y   0x000078faa7d7978d in fileEndForeignScan at file_fdw.c:794
        breakpoint already hit 2 times         # EXPLAIN 1次 + SELECT 1次
```

**断点命中次数分析**：

| 断点 | 函数名 | CREATE阶段 | EXPLAIN查询 | SELECT查询 | 总计 |
|------|--------|-----------|------------|-----------|------|
| 1 | file_fdw_validator | 3次 | 0次 | 0次 | 3次 |
| 2 | file_fdw_handler | 0次 | 1次 | 0次** | 1次 |
| 3 | fileGetForeignRelSize | 0次 | 1次 | 1次 | 2次 |
| 4 | fileGetForeignPaths | 0次 | 1次 | 1次 | 2次 |
| 5 | fileGetForeignPlan | 0次 | 1次 | 1次 | 2次 |
| 6 | fileBeginForeignScan | 0次 | 1次 | 1次 | 2次 |
| 7 | fileIterateForeignScan | 0次 | 0次 | 4次 | 4次 |
| 8 | fileEndForeignScan | 0次 | 1次 | 1次 | 2次 |

**注：`file_fdw_handler`在SELECT查询时不会再次调用，因为FdwRoutine结构体已被缓存。

**原因分析**：

根据源码`src/backend/foreign/foreign.c:428-459`，`GetFdwRoutineForRelation`函数实现了缓存机制：

```c
FdwRoutine *
GetFdwRoutineForRelation(Relation relation, bool makecopy)
{
    FdwRoutine *fdwroutine;
    FdwRoutine *cfdwroutine;

    if (relation->rd_fdwroutine == NULL)
    {
        /* Get the info by consulting the catalogs and the FDW code */
        fdwroutine = GetFdwRoutineByRelId(RelationGetRelid(relation));
        // ↑ 这里会调用handler函数

        /* Save the data for later reuse in CacheMemoryContext */
        cfdwroutine = (FdwRoutine *) MemoryContextAlloc(CacheMemoryContext,
                                                        sizeof(FdwRoutine));
        memcpy(cfdwroutine, fdwroutine, sizeof(FdwRoutine));
        relation->rd_fdwroutine = cfdwroutine;  // ← 缓存到Relation结构体中

        return fdwroutine;
    }

    /* We have valid cached data --- does the caller want a copy? */
    if (makecopy)
    {
        fdwroutine = (FdwRoutine *) palloc(sizeof(FdwRoutine));
        memcpy(fdwroutine, relation->rd_fdwroutine, sizeof(FdwRoutine));
        return fdwroutine;
    }

    /* Only a short-lived reference is needed, so just hand back cached copy */
    return relation->rd_fdwroutine;  // ← 直接返回缓存，不调用handler
}
```

**执行流程**：

1. **EXPLAIN查询时**：
   - 第一次调用`GetFdwRoutineForRelation`
   - `relation->rd_fdwroutine == NULL`，需要调用handler
   - 调用链：`GetFdwRoutineForRelation` → `GetFdwRoutineByRelId` → `GetFdwRoutine` → `OidFunctionCall0(fdwhandler)` → `file_fdw_handler`
   - 将返回的FdwRoutine缓存到`relation->rd_fdwroutine`

2. **SELECT查询时**：
   - 再次调用`GetFdwRoutineForRelation`
   - `relation->rd_fdwroutine != NULL`，已有缓存
   - 直接返回缓存的`rd_fdwroutine`，**不再调用handler函数**

**总结**：

- handler函数只在**第一次**需要FdwRoutine时被调用
- 后续查询会重用缓存的FdwRoutine结构体，提高性能
- 这是PostgreSQL的优化机制，避免重复调用handler函数

**断点命中统计（累计）**：

执行完CREATE阶段、`EXPLAIN SELECT`和`SELECT`查询后，检查所有断点的累计命中次数：

```gdb
(gdb) info breakpoints
Num     Type           Disp Enb Address            What
1       breakpoint     keep y   0x000078faa7d786a0 in file_fdw_validator at file_fdw.c:199
        breakpoint already hit 3 times        # CREATE阶段：3次
2       breakpoint     keep y   0x000078faa7d785b4 in file_fdw_handler at file_fdw.c:175
        breakpoint already hit 1 time         # EXPLAIN查询：1次
3       breakpoint     keep y   0x000078faa7d7919f in fileGetForeignRelSize at file_fdw.c:525
        breakpoint already hit 2 times         # EXPLAIN 1次 + SELECT 1次
4       breakpoint     keep y   0x000078faa7d79207 in fileGetForeignPaths at file_fdw.c:548
        breakpoint already hit 2 times         # EXPLAIN 1次 + SELECT 1次
5       breakpoint     keep y   0x000078faa7d7932b in fileGetForeignPlan at file_fdw.c:606
        breakpoint already hit 2 times         # EXPLAIN 1次 + SELECT 1次
6       breakpoint     keep y   0x000078faa7d7949e in fileBeginForeignScan at file_fdw.c:666
        breakpoint already hit 2 times         # EXPLAIN 1次 + SELECT 1次
7       breakpoint     keep y   0x000078faa7d795b9 in fileIterateForeignScan at file_fdw.c:720
        breakpoint already hit 6 times         # SELECT查询：4次（实际返回3行数据+1次NULL）
8       breakpoint     keep y   0x000078faa7d7978d in fileEndForeignScan at file_fdw.c:794
        breakpoint already hit 2 times         # EXPLAIN 1次 + SELECT 1次
```

**断点命中次数分析表**：

| 断点 | 函数名 | CREATE阶段 | EXPLAIN查询 | SELECT查询 | 总计 |
|------|--------|-----------|------------|-----------|------|
| 1 | file_fdw_validator | 3次 | 0次 | 0次 | 3次 |
| 2 | file_fdw_handler | 0次 | 1次 | 0次** | 1次 |
| 3 | fileGetForeignRelSize | 0次 | 1次 | 1次 | 2次 |
| 4 | fileGetForeignPaths | 0次 | 1次 | 1次 | 2次 |
| 5 | fileGetForeignPlan | 0次 | 1次 | 1次 | 2次 |
| 6 | fileBeginForeignScan | 0次 | 1次 | 1次 | 2次 |
| 7 | fileIterateForeignScan | 0次 | 0次 | 4次 | 4次 |
| 8 | fileEndForeignScan | 0次 | 1次 | 1次 | 2次 |

**注：`file_fdw_handler`在SELECT查询时不会再次调用，因为FdwRoutine结构体已被缓存。详细原因分析见下方"handler函数缓存机制"部分。

**待补充的测试**：
- `SELECT col1, col2 FROM test_foreign_table WHERE col1 BETWEEN 2 AND 4 ORDER BY col1;`
- `EXPLAIN (VERBOSE, BUFFERS) SELECT * FROM test_foreign_table WHERE col1 > 2;`

#### handler函数缓存机制

**为什么SELECT查询时handler函数不会被再次调用？**

根据源码`src/backend/foreign/foreign.c:428-459`，`GetFdwRoutineForRelation`函数实现了缓存机制：

```c
FdwRoutine *
GetFdwRoutineForRelation(Relation relation, bool makecopy)
{
    FdwRoutine *fdwroutine;
    FdwRoutine *cfdwroutine;

    if (relation->rd_fdwroutine == NULL)
    {
        /* Get the info by consulting the catalogs and the FDW code */
        fdwroutine = GetFdwRoutineByRelId(RelationGetRelid(relation));
        // ↑ 这里会调用handler函数

        /* Save the data for later reuse in CacheMemoryContext */
        cfdwroutine = (FdwRoutine *) MemoryContextAlloc(CacheMemoryContext,
                                                        sizeof(FdwRoutine));
        memcpy(cfdwroutine, fdwroutine, sizeof(FdwRoutine));
        relation->rd_fdwroutine = cfdwroutine;  // ← 缓存到Relation结构体中

        return fdwroutine;
    }

    /* We have valid cached data --- does the caller want a copy? */
    if (makecopy)
    {
        fdwroutine = (FdwRoutine *) palloc(sizeof(FdwRoutine));
        memcpy(fdwroutine, relation->rd_fdwroutine, sizeof(FdwRoutine));
        return fdwroutine;
    }

    /* Only a short-lived reference is needed, so just hand back cached copy */
    return relation->rd_fdwroutine;  // ← 直接返回缓存，不调用handler
}
```

**执行流程**：

1. **EXPLAIN查询时**：
   - 第一次调用`GetFdwRoutineForRelation`
   - `relation->rd_fdwroutine == NULL`，需要调用handler
   - 调用链：`GetFdwRoutineForRelation` → `GetFdwRoutineByRelId` → `GetFdwRoutine` → `OidFunctionCall0(fdwhandler)` → `file_fdw_handler`
   - 将返回的FdwRoutine缓存到`relation->rd_fdwroutine`

2. **SELECT查询时**：
   - 再次调用`GetFdwRoutineForRelation`
   - `relation->rd_fdwroutine != NULL`，已有缓存
   - 直接返回缓存的`rd_fdwroutine`，**不再调用handler函数**

**总结**：

- handler函数只在**第一次**需要FdwRoutine时被调用
- 后续查询会重用缓存的FdwRoutine结构体，提高性能
- 这是PostgreSQL的优化机制，避免重复调用handler函数

#### SELECT查询阶段总结

**EXPLAIN查询的调用顺序**：

1. **规划阶段**（4个函数，按顺序调用）：
   - `file_fdw_handler` → 获取FdwRoutine结构体
   - `fileGetForeignRelSize` → 估算表大小和行数
   - `fileGetForeignPaths` → 生成访问路径
   - `fileGetForeignPlan` → 创建查询计划节点

2. **执行阶段**（2个函数，EXPLAIN不迭代数据）：
   - `fileBeginForeignScan` → 初始化扫描（打开文件，获取统计信息）
   - `fileEndForeignScan` → 结束扫描（关闭文件）
   - `fileIterateForeignScan` → **未被调用**（EXPLAIN不实际获取数据）

**SELECT查询的调用顺序**：

1. **规划阶段**（3-4个函数，按顺序调用）：
   - `file_fdw_handler` → 获取FdwRoutine结构体（可能已在之前调用，被缓存）
   - `fileGetForeignRelSize` → 估算表大小和行数
   - `fileGetForeignPaths` → 生成访问路径
   - `fileGetForeignPlan` → 创建查询计划节点

2. **执行阶段**（3个函数，完整执行）：
   - `fileBeginForeignScan` → 初始化扫描（打开文件）
   - `fileIterateForeignScan` → **循环调用**（每次返回一行数据，直到返回NULL）
   - `fileEndForeignScan` → 结束扫描（关闭文件）

**关键区别总结**：

| 阶段 | 调用方式 | 调用的FDW函数 | 说明 |
|------|---------|--------------|------|
| CREATE阶段 | 通过"同名函数"直接调用 | validator（3次） | 只验证OPTIONS，不涉及数据访问 |
| EXPLAIN查询 | 通过查询执行流程调用 | handler + 规划函数 + 部分执行函数 | 不实际迭代数据，只获取统计信息 |
| SELECT查询 | 通过查询执行流程调用 | handler + 规划函数 + 完整执行函数 | 完整执行，循环获取所有数据 |

---

## 五、FDW工作原理总结

本文以 **file_fdw** 为例，用最简单的 SELECT（如 `SELECT * FROM foreign_table WHERE col1 > 2`）描述 FDW 的基本运行链。CREATE 阶段只做元数据与选项校验，不调用 handler 与 scan-related；查询阶段经 `GetFdwRoutineForRelation()`（`src/backend/foreign/foreign.c`）调用 handler，再按规划/执行链调用 scan-related 回调。validator 仅在 CREATE 时通过 `transformGenericOptions()` 调用（`foreigncmds.c:191`），SELECT/EXPLAIN 等查询阶段不调用。

### 5.1 基本运行链（file_fdw + 简单 SELECT）

| 阶段 / 操作 | 调用的 FDW 函数 | 说明 |
|-------------|-----------------|------|
| CREATE … 带 OPTIONS 时 | **validator**（每类 CREATE 带 OPTIONS 时各 1 次） | 入口：`foreigncmds.c` 中 `transformGenericOptions()` |
| EXPLAIN 查询外部表 | **handler**（首次） + **GetForeignRelSize** + **GetForeignPaths** + **GetForeignPlan** + **BeginForeignScan** + **EndForeignScan** | 不调用 validator、不调用 `IterateForeignScan` |
| SELECT 查询外部表（简单扫描） | **handler**（首次，之后用 `relation->rd_fdwroutine` 缓存） + 同上规划三函数 + **BeginForeignScan** + **IterateForeignScan**（多次） + **EndForeignScan** | 完整扫描，直到 `IterateForeignScan` 返回 NULL |

### 5.2 CREATE 阶段与系统表

| 阶段 | FDW 侧被调用 | 主要系统表 |
|------|--------------|------------|
| CREATE EXTENSION | 无（仅写入 handler/validator 的 OID） | `pg_proc`、`pg_foreign_data_wrapper`、`pg_extension`、`pg_depend` |
| CREATE SERVER | validator（若有 OPTIONS） | `pg_foreign_server`、`pg_depend` |
| CREATE USER MAPPING | validator（若有 OPTIONS） | `pg_user_mapping`、`pg_depend` |
| CREATE FOREIGN TABLE | validator（若有 OPTIONS） | `pg_class`、`pg_foreign_table`、`pg_attribute`、`pg_depend` 等 |

### 5.3 FDW 与普通 Extension 的区别

| 维度 | 普通 Extension | FDW |
|------|----------------|-----|
| 注册 | `CREATE EXTENSION` | `CREATE EXTENSION` + **CREATE FOREIGN DATA WRAPPER** |
| 与规划/执行器 | 无标准接口 | 通过 FdwRoutine 的 **scan-related 回调** 接入 Planner 与 Executor |
| 额外系统表 | 无强制要求 | **pg_foreign_data_wrapper**、pg_foreign_server、pg_foreign_table 等 |

相同点：均通过 CREATE EXTENSION 安装，函数进 `pg_proc`，扩展记录在 `pg_extension`。

---

## 六、参考资源

### PostgreSQL官方文档
- [Chapter 59. Writing a Foreign Data Wrapper](https://www.postgresql.org/docs/16/fdwhandler.html)

### UDB-TX源码文件（信息来源）

**系统表定义**：
- `src/include/catalog/pg_proc.h`：pg_proc系统表定义
- `src/include/catalog/pg_foreign_data_wrapper.h`：pg_foreign_data_wrapper系统表定义
- `src/include/catalog/pg_extension.h`：pg_extension系统表定义

**函数实现**：
- `src/backend/catalog/pg_proc.c:582`：`ProcedureCreate()` - 插入到pg_proc
- `src/backend/commands/extension.c:1915`：`InsertExtensionTuple()` - 插入到pg_extension
- `src/backend/commands/foreigncmds.c:631`：`CreateForeignDataWrapper()` - 插入到pg_foreign_data_wrapper
- `src/backend/commands/foreigncmds.c:191`：`transformGenericOptions()` - 调用validator
- `src/backend/foreign/foreign.c:326`：`GetFdwRoutine()` - 调用handler

**FDW实现示例**：
- `contrib/file_fdw/file_fdw.c`：file_fdw实现
- `src/include/foreign/fdwapi.h`：FdwRoutine结构体定义

---

