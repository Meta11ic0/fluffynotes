# FDW 规划期回调走读（以 postgres_fdw 为例）

> **源码基线**：PostgreSQL **19beta1**（核对日期 2026-07-17）。文中行号以该版本源码为准；合并上游后可能偏移。  
> **范围**：主体为 **规划期**（`Query` → `PlannedStmt`）；[§9](#9-执行期回调速查) 为执行边界速查。  
> **前置**：Analyzer/Rewriter 已产出带 **`rtable` + `jointree`** 的 `Query`。  
> **代码块**：上方 **源码**：`path:行号` 便于跳转；片段为节选。

---

## 目录

1. [规划器入口：带着什么进 `planner()`](#1-规划器入口带着什么进-planner)
2. [**`planner()` 主线拆解** — 一条 SQL 逐步演变（推荐先读）](#2-planner-主线拆解--一条-sql-逐步演变)
3. [何时因语句类型分叉](#3-何时因语句类型分叉)
4. [何时因「外表」分叉](#4-何时因外表分叉)
5. [SQL 走通：回调顺序](#5-sql-走通回调顺序)
6. [postgres_fdw 注册回调一览](#6-postgres_fdw-注册回调一览)
7. [Join / Upper 下推](#7-join--upper-下推)
8. [未实现的规划期回调（NULL）](#8-未实现的规划期回调null)
9. [执行期回调速查](#9-执行期回调速查)
10. [参考索引](#10-参考索引)

**阅读建议**：§1 → §2（示例 SQL + §2.1～2.5 主线递进）→ §5 外表两条 SQL → §6 查表。

---

## 1. 规划器入口：带着什么进 `planner()`

经过分析和重写后，TCOP 调用 `pg_plan_query` → `planner()`（`postgres.c:899` → `planner.c:333`）。此时交给规划器的是 **`Query` 树**（`rtable`、`jointree`、聚合/排序/limit 等字段；`commandType` 区分 SELECT/DML/UTILITY）。

规划器 **不再** 解析表名绑定 catalog。那是 Analyzer 的职责。规划器在 `Query` 上 **改树（预处理）**，在 **`query_planner()`** 阶段才为参与扫描的来源建立 **`RelOptInfo` 与 `pathlist`**，最后在 **`create_plan()`** 产出 **`Plan`**。§2 用下面这条 GROUP BY 示例 SQL，按 **Query → 预处理 → Path → Plan** 递进说明；外表与 FDW 回调见 §2.5。

---

## 2. `planner()` 主线拆解，一条 SQL 逐步演变

下文使用这条 GROUP BY 示例 SQL。外表与 FDW 回调的插入点见 §2.5。

```sql
SELECT u.id, u.name, count(o.id) AS paid_orders
FROM users u
LEFT JOIN orders o
  ON o.user_id = u.id AND o.status = 'paid'
WHERE u.active = true
GROUP BY u.id, u.name
HAVING count(o.id) > 0
ORDER BY paid_orders DESC
LIMIT 10;
```

§2.1～2.4 沿 `planner()` 调用链展开，依次覆盖 (1) Rewriter 之后的 `Query`；(2) `subquery_planner` 等对 `Query` 的预处理；(3) `query_planner()` 为 `jointree` 中的来源建立 `RelOptInfo` 并生成 `Path`；(4) `create_plan()` 将选中的 `Path` 转为 `Plan`。各结构在首次出现的阶段再展开说明，文首不设 `Query` 与 `RelOptInfo` 的对照表。`Query` 由 Analyzer 绑定 catalog 后产出，本文从进 `planner()` 时的形态开始。

---

### 2.1 进 `planner()` 时的 `Query`

Rewriter 对本例通常不改树。进入 `planner()` 时，规划器面对的是 **语义化的 `Query` 节点**，核心字段如下。

**`rtable`（range table，不是 SQL 里的"一张表"）**  
是一个列表，`List`，元素为 **`RangeTblEntry`（RTE）**：本次查询登记的所有范围来源（基表、视图展开、CTE、子查询、JOIN 结果伪表等），带 `relid`、别名等。字段名 **`rtable`** 表示 range table 列表；没有名为 `RTABLE` 的独立节点类型。

**`jointree`**  
`FromExpr` / `JoinExpr` / `RangeTblRef` 组成的树，描述 **哪些 RTE 参与扫描、如何连接、ON/WHERE 谓词挂在哪**。`jointree` 通过 **`RangeTblRef(n)`** 引用 `rtable` 第 `n` 项（从 1 起计）。

本例在进 planner 时的形态：

```text
Query
  commandType = CMD_SELECT
  rtable[1]  RTE_RELATION   alias u   → users（relid 来自 catalog）
  rtable[2]  RTE_RELATION   alias o   → orders
  rtable[3]  RTE_JOIN       LEFT      → 连接结果伪表
  jointree: FromExpr
    fromlist → JoinExpr(LEFT, rtable[1] ref, rtable[2] ref)
      quals:  o.user_id = u.id AND o.status = 'paid'    ← ON
    quals:    u.active = true                           ← WHERE（无 Query->where 字段）
  targetList:  u.id, u.name, Aggref(count(o.id)) AS paid_orders
  groupClause: u.id, u.name
  havingQual:  count(o.id) > 0
  sortClause:  paid_orders DESC
  limitCount:  10
  hasAggs = true
```

**谓词与子句在 `Query` 中的位置**（`deconstruct_jointree` / 上层路径生成会按此拆分）：

- `ON … AND o.status = 'paid'` → `JoinExpr->quals`
- `WHERE u.active = true` → 顶层 `FromExpr->quals`
- `GROUP BY` / `HAVING` → `groupClause` / `havingQual`（不在 `jointree`）
- `ORDER BY` / `LIMIT` → `sortClause` / `limitCount`

此阶段 **仅有 `Query` 树**，尚无 `RelOptInfo` 或 `pathlist`；后者在 §2.3 进入 `query_planner()` 后才为参与扫描的来源建立。

---

### 2.2 规划预处理：`subquery_planner` 与 `PlannerInfo`

`standard_planner()`（`planner.c:351`）调用 **`subquery_planner()`**（`planner.c:775`）。入口处创建 **`PlannerInfo *root`**（`planner.c:790`），代表**当前这一层 `Query` 的规划上下文**。`root->parse` 指向待规划的 `Query`，并预留 `simple_rel_array`、`upper_rels` 等。

在约 **866～1368 行**，`subquery_planner` **只改写 `Query`**（`rtable`、`jointree`、表达式），**不**调用 `build_simple_rel`，因而此阶段一般还没有基表上的 **`pathlist`**。对本例：

- **`preprocess_relation_rtes`**（866）：按 RTE 读取 `users` / `orders` 的 catalog 与统计线索。
- **`pull_up_sublinks`**（880）、**`pull_up_subqueries`**（895）：无 SubLink / FROM 子查询时为本例跳过；有子查询或 `EXISTS` 时会改 `jointree` 或 `rtable`。规划前合并子查询改的是当前 `Query`，不会先执行子查询再往 `rtable` 追加项。

```c
	if (parse->hasSubLinks)
		pull_up_sublinks(root);
	...
	pull_up_subqueries(root);
```

- **`preprocess_expression`、外连接化简等**（1022+、1358）：常量折叠、可能的外连接削弱（本例 LEFT + HAVING 时常仍为 LEFT）。

预处理结束后仍是 **`Query` 树**；**`RelOptInfo` 在随后 `grouping_planner` → `query_planner` 中才出现**（§2.3）。

---

### 2.3 从 `Query` 到 `Path`：`grouping_planner` 与 `query_planner`

`subquery_planner` 在 **1373 行** 调用 **`grouping_planner()`**（`planner.c:1775`），开始 **代价驱动的 Path 搜索**。此时工作重心从"改 `Query`"转到"在 **`RelOptInfo` 上挂候选 `Path`**"。

**`RelOptInfo` 与 `pathlist`**  
每个 **参与本次扫描或连接的逻辑关系**（基表、连接结果、上层 GROUP/SORT 等）对应一个 `RelOptInfo`；其 **`pathlist`** 存放多种实现该关系的候选 Path（顺序扫描、索引扫描、`ForeignPath`、Hash Join 等）及代价。

**为何遍历 `jointree` 而非整个 `rtable`**  
`rtable` 是 Analyzer 产出的 **完整登记表**；`jointree` 标明 **本次 FROM 子句实际要扫描/连接的范围**。`rtable` 中可有 **不被 `jointree` 引用** 的项（例如纯 `INSERT INTO ft VALUES (...)` 的目标表、视图展开遗留 RTE），不会为之 `build_simple_rel`。本例 `jointree` 引用 **rtable[1] users、rtable[2] orders**；**rtable[3] JOIN 伪表** 在连接阶段再生成 join rel，不先作为基表 `build_simple_rel`。

**`query_planner()`**（`planmain.c:54`）负责 **FROM/WHERE** 段（`planner.c:1988-1995` 注释）：

1. **`add_base_rels_to_query`**（`initsplan.c:178`）递归 **`jointree`**，对每个 `RangeTblRef(n)` 调用 **`build_simple_rel(root, n)`**（`relnode.c:212`），得到空的 `RelOptInfo`。
2. 对 **`RTE_RELATION`**（含外表），`build_simple_rel` 内调 **`get_relation_info`**（`relnode.c:365` → `plancat.c:121`）：读统计与索引；若 **`RELKIND_FOREIGN_TABLE`** 则设置 **`rel->fdwroutine = GetFdwRoutineForRelation(...)`**（`plancat.c:532-546`），此时尚未调 `GetForeignRelSize`。
3. **`deconstruct_jointree`** + **`make_one_rel`**（`allpaths.c:179`）：拆分 restrict、枚举 JOIN 顺序、生成基表与 join Path。外表在 **`set_rel_size` / `set_rel_pathlist`** 里走 **`set_foreign_*`** → **`GetForeignRelSize` / `GetForeignPaths`**（`allpaths.c:986`、`1007`）。若 **`orders` 为外表**，在本步对其 `RelOptInfo` 估行数并生成 `ForeignPath`。

**典型边界**：纯 **`INSERT INTO ft VALUES (...)`** 的目标表可在 `rtable` 中但不在 `jointree` 当扫描源 → 不 `build_simple_rel` → 通常不调 `GetForeignRelSize`（`planmain.c:91-94`、`107-159`）。

`query_planner` 返回的 **`current_rel`** 表示 join + WHERE 之后的逻辑结果。随后在同一 `grouping_planner` 内自下而上一层层叠加上层 Path：

| 顺序 | 函数 | SQL | FDW（若适用） |
|------|------|-----|----------------|
| 1 | `create_grouping_paths` | GROUP / HAVING | `GetForeignUpperPaths` |
| 2 | `create_window_paths` | 窗口 | 同上 |
| 3 | `create_ordered_paths` | ORDER BY | 同上 |
| 4 | `create_limit_path` | LIMIT | — |
| 5 | `create_modifytable_path` | DML | `planner.c:2452` |

本例：`users`⨝`orders` → `create_grouping_paths`（`count` + HAVING）→ `create_ordered_paths` → `create_limit_path`（与上文一致）。无窗口，**`create_window_paths`** 跳过。

`grouping_planner` 返回后，`subquery_planner` 内 **`set_cheapest`**（1395）、**`return root`**（1397），`PlannerInfo` 的 `upper_rels[FINAL]` 上已有最便宜的上层 Path。

---

### 2.4 到 `Plan`：`standard_planner` 收尾

`subquery_planner` 返回后，`standard_planner` 依次：

```c
	root = subquery_planner(glob, parse, NULL, NULL, NULL, false,
							tuple_fraction, NULL);    /* planner.c:537 */

	final_rel = fetch_upper_rel(root, UPPERREL_FINAL, NULL);
	best_path = get_cheapest_fractional_path(final_rel, tuple_fraction);
	top_plan = create_plan(root, best_path);            /* planner.c:544 */
```

**`create_plan`** 将选中的 Path 转为 **`Plan`**：扫描路径 → **`GetForeignPlan`**（`createplan.c:4004`）→ **`ForeignScan`**；DML → **`PlanForeignModify`** / **`PlanDirectModify`**（`createplan.c:7204-7214`）。本例 **CMD_SELECT** 不走 Modify 分支。

**本例调用骨架**（行号锚点：`planner.c` / `planmain.c` / `relnode.c`）：

```text
standard_planner @351
  subquery_planner @775
    PlannerInfo @790
    [866～1368]  预处理 Query（本例主要为 qual 化简）
    grouping_planner @1373
      preprocess_targetlist @1908     （纯 SELECT，无 AddForeignUpdateTargets）
      query_planner → RelOptInfo/Path  （users ⨝ orders）
      create_grouping_paths → ordered → limit
    set_cheapest; return @1395～1397
  get_cheapest_fractional_path @542
  create_plan @544
```

**本例可能的 Plan 形态**（本地表）：`Limit` → `Sort` → `HashAggregate`（GROUP + HAVING）→ `Hash Left Join`（或 Nested Loop）→ `Seq Scan users` + `Seq Scan orders`。若 `orders` 为外表，扫描侧常为 **`ForeignScan`**。

---

### 2.5 FDW 在本例中的插入点

仅当某 RTE 为 **`RELKIND_FOREIGN_TABLE`** 且其 **`RangeTblRef` 出现在 `jointree`** 时，规划期才会走 FDW 回调；纯本地两表时 FDW 函数表项不被调用。

| 阶段 | 本例（若 `orders` 为外表） |
|------|---------------------------|
| §2.2 预处理 | 通常跳过 |
| §2.3 `make_one_rel` | `GetForeignRelSize` → `GetForeignPaths`（`allpaths.c:986`、`1007`） |
| §2.3 上层路径 | 可能 `GetForeignUpperPaths`（GROUP/ORDER 下推） |
| §2.4 `create_plan` | `GetForeignPlan` → `ForeignScan`（`createplan.c:4004`） |

§5 用两条更短的 SQL 对照回调顺序。

#### 2.5.1 UPDATE/DELETE/MERGE：目标列与 junk 列

`grouping_planner` 里 **`preprocess_targetlist`**（`planner.c:1908`）对 **UPDATE/DELETE/MERGE** 且目标为外表时，经：

```text
preprocess_targetlist()           preptlist.c:66
  add_row_identity_columns()      preptlist.c:127
    AddForeignUpdateTargets()     appendinfo.c:994
      postgresAddForeignUpdateTargets   postgres_fdw.c:1964
```

在 `processed_tlist` 中追加 **junk 列**（远程 `ctid`、整行 `Var` 等），供 `ModifyTable` 定位待修改行。**纯 SELECT** 与 **INSERT** 通常不进 `add_row_identity_columns`（`preptlist.c:108-128`）。

---

## 3. 何时因语句类型分叉

**源码**：`src/include/nodes/nodes.h:270` 起（`CmdType`）

| 类型 | `commandType` | 规划链 |
|------|---------------|--------|
| 查询 | `CMD_SELECT` | §2 全程；无 `ModifyTable` |
| DML | `CMD_INSERT` / `UPDATE` / `DELETE` / `MERGE` | §2.5.1 + §2.3 `GetForeign*` + `create_modifytable_path` |
| DDL / 工具 | **`CMD_UTILITY`** | `utilityStmt` 另路，不经 `query_planner` 的 FDW 扫描 |

`CREATE FOREIGN TABLE`、`IMPORT FOREIGN SCHEMA` 走 **工具回调**（`postgres_fdw.c:816`），不在上表路径里。

---

## 4. 何时因"外表"分叉

### 4.1 没有 `RTE_FOREIGN`

外表在 RTE 层是 **`RTE_RELATION`** + **`rte->relkind == RELKIND_FOREIGN_TABLE`**（`parsenodes.h:1122` 起 `RTEKind` 无 FOREIGN 项）。

### 4.2 分叉点（接 §2.3）

| 步骤 | 文件 | 外表分支 |
|------|------|----------|
| `get_relation_info` | `plancat.c:532-546` | 取 `fdwroutine` |
| `set_rel_size` | `allpaths.c:407`（外表分支约 435–439） | `set_foreign_size` → `GetForeignRelSize` |
| `set_rel_pathlist` | `allpaths.c:516`（外表分支约 532–537） | `set_foreign_pathlist` → `GetForeignPaths` |
| `create_plan` | `createplan.c:4004` | `GetForeignPlan` → `ForeignScan` |

**源码**（`GetForeignRelSize` 调用）：

```c
	rel->fdwroutine->GetForeignRelSize(root, rel, rte->relid);   /* allpaths.c:986 */
```

---

## 5. SQL 走通，回调顺序

外表 `ft`，已 `CREATE EXTENSION postgres_fdw`。

### 5.1 `SELECT * FROM ft WHERE id = 1;`

| 阶段（§2） | FDW 回调 / 要点 |
|------------|-----------------|
| §2.2 预处理 | 通常跳过 |
| §2.3 `make_one_rel` | `GetForeignRelSize` → `GetForeignPaths` |
| §2.3 上层 | 可能 `GetForeignUpperPaths` |
| §2.4 `create_plan` | `GetForeignPlan` → `ForeignScan` |

### 5.2 `UPDATE ft SET col = 1 WHERE id = 1;`

| 阶段 | FDW 回调 / 要点 |
|------|-----------------|
| §2.5.1 | `postgresAddForeignUpdateTargets` |
| §2.3 | `GetForeignRelSize` / `GetForeignPaths` |
| §2.3 | `create_modifytable_path` |
| §2.4 | `postgresPlanForeignModify`（或 `PlanDirectModify`） |

**INSERT `VALUES`**：目标 `ft` 常 **不在 jointree** 作扫描源 → 通常 **无** `GetForeignRelSize`；仍有 `PlanForeignModify`（`createplan.c` 约 7110 行注释：INSERT 目标表不是 baserel）。

---

## 6. postgres_fdw 注册回调一览

`postgres_fdw_handler()`（`postgres_fdw.c:770-830`）。

### 规划期

| 字段 | 实现 | 内核调用（与 §2 对应） |
|------|------|------------------------|
| `GetForeignRelSize` | `postgres_fdw.c:841` | §2.3 `allpaths.c:986` |
| `GetForeignPaths` | `postgres_fdw.c:1232` | §2.3 `allpaths.c:1007` |
| `GetForeignPlan` | `postgres_fdw.c:1446` | §2.4 `createplan.c:4004` |
| `GetForeignJoinPaths` | `postgres_fdw.c:7212` | §7 |
| `GetForeignUpperPaths` | `postgres_fdw.c:7597` | §2.3 |
| `AddForeignUpdateTargets` | `postgres_fdw.c:1964` | §2.5.1 |
| `PlanForeignModify` | `postgres_fdw.c:1992` | §2.4 |
| `PlanDirectModify` | `postgres_fdw.c:2664` | §2.4 |

### 执行期（列名）

`BeginForeignScan`, `IterateForeignScan`, `EndForeignScan`, `BeginForeignModify`, `ExecForeignInsert`, `ExecForeignUpdate`, `ExecForeignDelete`, `EndForeignModify`, … 见 [§9](#9-执行期回调速查)。

---

## 7. Join / Upper 下推

- **`GetForeignJoinPaths`**：`joinpath.c:362-364`；同 server 两外表 JOIN。
- **`GetForeignUpperPaths`**：`planner.c:2500` 等（`UPPERREL_GROUP_AGG` / `ORDERED` / `FINAL` 等多处）；聚合/排序下推远程。

postgres_fdw **均已实现**（注册 `postgres_fdw.c:819`、`822`）。

---

## 8. 未实现的规划期回调（NULL）

| 回调 | 影响 |
|------|------|
| `GetForeignRowMarkType` | `FOR UPDATE` 用默认 `ROW_MARK_COPY` |
| `IsForeignScanParallelSafe` | 视为不可并行 |
| `ReparameterizeForeignPathByChild` | 无 FDW 自定义重参数化 |

---

## 9. 执行期回调速查

| 回调 | 时机 |
|------|------|
| `BeginForeignScan` / `IterateForeignScan` / `EndForeignScan` | `ForeignScan` 执行 |
| `BeginForeignModify` / `ExecForeign*` / `EndForeignModify` | `ModifyTable` 修改外表 |
| `ExplainForeignScan` 等 | `EXPLAIN` |

---

## 10. 参考索引

| 主题 | 文件 |
|------|------|
| `planner` 入口 | `planner.c:333`、`planmain.c:54` |
| `RelOptInfo` | `optimizer/util/relnode.c:212` |
| FDW API | `src/include/foreign/fdwapi.h:208` |
| postgres_fdw | `contrib/postgres_fdw/postgres_fdw.c:770-830` |

---

*行号见文首源码基线；合并上游后可能偏移。*
