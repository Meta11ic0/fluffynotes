# FDW 规划期回调走读（以 postgres_fdw 为例）

> **范围**：只讲 **规划期**（`Query` → `PlannedStmt`）。执行期见 [§9](#9-执行期回调速查)。  
> **前置**：Analyzer/Rewriter 已产出带 **`rtable` + `jointree`** 的 `Query`（主线见 `sql_pipeline_faq.md` §2–§4、§5）。  
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

**阅读路线**：§1 → §2（示例 SQL + §2.1～2.4 主线递进 + §2.5 FDW 插入点 + §2.6～2.10 源码）→ §5 外表两条 SQL → §6 查表。

---

## 1. 规划器入口：带着什么进 `planner()`

TCOP 在分析、重写之后调用 `pg_plan_query` → `planner()`（`postgres.c:899` → `planner.c:333`）。此时交给规划器的 **不是 SQL 字符串**，而是 **`Query` 树**（`rtable`、`jointree`、聚合/排序/limit 等字段；`commandType` 区分 SELECT/DML/UTILITY）。

规划器 **不再** 解析表名绑定 catalog——那是 Analyzer 的职责。规划器在 `Query` 上 **改树（预处理）**，在 **`query_planner()`** 阶段才为参与扫描的来源建立 **`RelOptInfo` 与 `pathlist`**，最后在 **`create_plan()`** 产出 **`Plan`**。§2 用与 `sql_pipeline_faq.md` 相同的一条 GROUP BY 示例 SQL，按 **Query → 预处理 → Path → Plan** 递进说明；外表与 FDW 回调见 §2.5 及 §2.6～2.10。

---

## 2. `planner()` 主线拆解 —— 一条 SQL 逐步演变

下文与 `sql_pipeline_faq.md` 使用同一条 GROUP BY 示例 SQL。外表与 FDW 回调的插入点见 §2.5，源码走读见 §2.6～2.10。

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

**叙述顺序**：§2.1～2.4 沿 `planner()` 调用链展开——(1) Rewriter 之后的 `Query`；(2) `subquery_planner` 等对 `Query` 的预处理；(3) `query_planner()` 为 `jointree` 中的来源建立 `RelOptInfo` 并生成 `Path`；(4) `create_plan()` 将选中的 `Path` 转为 `Plan`。各结构在首次出现的阶段再说明，文首不设 `Query` 与 `RelOptInfo` 的对照表。`Query` 在 Analyzer 中的构造见 `sql_pipeline_faq.md` §3、§5.6。

---

### 2.1 进 `planner()` 时的 `Query`

Rewriter 对本例通常不改树。进入 `planner()` 时，规划器面对的是 **语义化的 `Query` 节点**，核心字段如下。

**`rtable`（range table，不是 SQL 里的「一张表」）**  
是一个列表，`List`，元素为 **`RangeTblEntry`（RTE）**：本次查询登记的所有范围来源（基表、视图展开、CTE、子查询、JOIN 结果伪表等），带 `relid`、别名等。字段名 **`rtable`** 表示 range table 列表；没有名为 `RTABLE` 的独立节点类型。

**`jointree`**  
`FromExpr` / `JoinExpr` / `RangeTblRef` 组成的树，描述 **哪些 RTE 参与扫描、如何连接、ON/WHERE 谓词挂在哪**。`jointree` 通过 **`RangeTblRef(n)`** 引用 `rtable` 第 `n` 项（从 1 起计，与 FAQ 一致）。

本例在进 planner 时的形态（与 `sql_pipeline_faq.md` §3、§5.6 一致）：

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

`standard_planner()`（`planner.c:351`）调用 **`subquery_planner()`**（`planner.c:775`）。入口处创建 **`PlannerInfo *root`**（`planner.c:790`）：**当前这一层 `Query` 的规划上下文**，`root->parse` 指向待规划的 `Query`，并预留 `simple_rel_array`、`upper_rels` 等。

在约 **866～1368 行**，`subquery_planner` **只改写 `Query`**（`rtable`、`jointree`、表达式），**不**调用 `build_simple_rel`，因而此阶段一般还没有基表上的 **`pathlist`**。对本例：

- **`preprocess_relation_rtes`**（866）：按 RTE 读取 `users` / `orders` 的 catalog 与统计线索。
- **`pull_up_sublinks`**（880）、**`pull_up_subqueries`**（895）：无 SubLink / FROM 子查询时为本例 no-op；有子查询或 `EXISTS` 时会改 `jointree` 或 `rtable` 结构。
- **`preprocess_expression`、外连接化简等**（1022+、1358）：常量折叠、可能的外连接削弱（本例 LEFT + HAVING 时常仍为 LEFT）。

预处理结束后仍是 **`Query` 树**；**`RelOptInfo` 在随后 `grouping_planner` → `query_planner` 中才出现**（§2.3）。

---

### 2.3 从 `Query` 到 `Path`：`grouping_planner` 与 `query_planner`

`subquery_planner` 在 **1373 行** 调用 **`grouping_planner()`**（`planner.c:1775`），开始 **代价驱动的 Path 搜索**。此时工作对象从「改 `Query`」转为「在 **`RelOptInfo` 上挂候选 `Path`**」。

**`RelOptInfo` 与 `pathlist`**  
每个 **参与本次扫描或连接的逻辑关系**（基表、连接结果、上层 GROUP/SORT 等）对应一个 `RelOptInfo`；其 **`pathlist`** 存放多种实现该关系的候选 Path（顺序扫描、索引扫描、`ForeignPath`、Hash Join 等）及代价。

**为何遍历 `jointree` 而非整个 `rtable`**  
`rtable` 是 Analyzer 产出的 **完整登记表**；`jointree` 标明 **本次 FROM 子句实际要扫描/连接的范围**。`rtable` 中可有 **不被 `jointree` 引用** 的项（例如纯 `INSERT INTO ft VALUES (...)` 的目标表、视图展开遗留 RTE），不会为之 `build_simple_rel`。本例 `jointree` 引用 **rtable[1] users、rtable[2] orders**；**rtable[3] JOIN 伪表** 在连接阶段再生成 join rel，不先作为基表 `build_simple_rel`。

**`query_planner()`**（`planmain.c:54`）负责 **FROM/WHERE** 段（`planner.c:1988-1995` 注释）：

1. **`add_base_rels_to_query`**（`initsplan.c:163-186`）递归 **`jointree`**，对每个 `RangeTblRef(n)` 调用 **`build_simple_rel(root, n)`**（`relnode.c:212`），得到空的 `RelOptInfo`。
2. **`get_relation_info`**（`plancat.c:121`）：基表统计、索引；若 `RELKIND_FOREIGN_TABLE` 则设置 **`rel->fdwroutine`**（尚未 `GetForeignRelSize`）。
3. **`deconstruct_jointree`** + **`make_one_rel`**（`allpaths.c:297`）：拆分 restrict、枚举 JOIN 顺序、生成基表与 join Path。

`query_planner` 返回的 **`current_rel`** 表示 join + WHERE 之后的逻辑结果（本例：LEFT JOIN 且 `u.active` 已作为 restrict）。随后在同一 `grouping_planner` 内自下而上叠上层 Path：**`create_grouping_paths`**（GROUP/HAVING）→ **`create_ordered_paths`**（ORDER）→ **`create_limit_path`**（LIMIT，挂到 `UPPERREL_FINAL`）。本例无窗口，**`create_window_paths`** 跳过。

`grouping_planner` 返回后，`subquery_planner` 内 **`set_cheapest`**（1395）、**`return root`**（1397），`PlannerInfo` 的 `upper_rels[FINAL]` 上已有最便宜的上层 Path。

---

### 2.4 到 `Plan`：`standard_planner` 收尾

`subquery_planner` 返回后，`standard_planner` 在 **542～544 行** 从 `UPPERREL_FINAL` 取 **`get_cheapest_fractional_path`**，再 **`create_plan(root, best_path)`** 将 Path 树转为执行器使用的 **`Plan` 树**。

**本例调用骨架**（行号锚点：`planner.c` / `planmain.c` / `relnode.c`）：

```text
standard_planner @351
  subquery_planner @775
    PlannerInfo @790
    [866～1368]  预处理 Query（本例：mostly qual 化简）
    grouping_planner @1373
      preprocess_targetlist @1908     （纯 SELECT，无 AddForeignUpdateTargets）
      query_planner → RelOptInfo/Path  （users ⨝ orders）
      create_grouping_paths → ordered → limit
    set_cheapest; return @1395～1397
  get_cheapest_fractional_path @542
  create_plan @544
```

**本例可能的 Plan 形态**（本地表）：`Limit` → `Sort` → `HashAggregate`（GROUP + HAVING）→ `Hash Left Join`（或 Nested Loop）→ `Seq Scan users` + `Seq Scan orders`。若 `orders` 为外表，扫描侧常为 **`ForeignScan`**（§2.10）。

---

### 2.5 FDW 在本例中的插入点

仅当某 RTE 为 **`RELKIND_FOREIGN_TABLE`** 且其 **`RangeTblRef` 出现在 `jointree`** 时，规划期才会走 FDW 回调；纯本地两表时 FDW 函数表项不被调用。

| 阶段 | 本例（若 `orders` 为外表） |
|------|---------------------------|
| §2.2 预处理 | 通常 no-op |
| §2.3 `make_one_rel` | `GetForeignRelSize` → `GetForeignPaths`（`allpaths.c:986`、`1007`） |
| §2.3 上层路径 | 可能 `GetForeignUpperPaths`（GROUP/ORDER 下推） |
| §2.4 `create_plan` | `GetForeignPlan` → `ForeignScan`（`createplan.c:4004`） |

下文 §2.6～2.10 按函数列出 **源码位置与 postgres_fdw 钩子**；§5 用两条更短的 SQL 对照回调顺序。

---

### 2.6 `standard_planner`：先规划 Path，再生成 Plan

`planner()` 进入 `standard_planner()`（`planner.c:351`）后，对顶层语句依次做两件事：

```c
	root = subquery_planner(glob, parse, NULL, NULL, NULL, false,
							tuple_fraction, NULL);    /* planner.c:537 */

	final_rel = fetch_upper_rel(root, UPPERREL_FINAL, NULL);
	best_path = get_cheapest_fractional_path(final_rel, tuple_fraction);
	top_plan = create_plan(root, best_path);            /* planner.c:544 */
```

§2.7～§2.9 发生在 **`subquery_planner` 返回之前**；§2.10 是 **`create_plan`**。§2.1～2.4 已给出主线概览。

---

### 2.7 `subquery_planner` 前半：预处理 `Query`

`subquery_planner()` 收到 Rewriter 之后的 **`Query`**（§2 示例：两表 LEFT JOIN + GROUP/HAVING/ORDER/LIMIT）。开头创建 **`PlannerInfo`**（`planner.c:790`），随后到约 **1368 行** 为止，工作对象是 **`Query` 树**（`rtable`、`jointree`、表达式），**不**调用 `build_simple_rel`，因而一般还没有基表上的 **`pathlist`**。

| 步骤 | 行号 | 效果 |
|------|------|------|
| `preprocess_relation_rtes` | 866 | 按 RTE 读 catalog |
| `pull_up_sublinks` | 880 | `EXISTS` / `ANY` 改写 `jointree`（**本例跳过**） |
| `pull_up_subqueries` | 895 | FROM 子查询上拉（**本例 no-op**） |

```c
	if (parse->hasSubLinks)
		pull_up_sublinks(root);
	...
	pull_up_subqueries(root);
```

规划前合并子查询：改的是当前 `Query`，不是执行完子查询再往 `rtable` 里加项。本例 **无** SubLink / FROM 子查询，**pull_up_*** 为 no-op，但 **866** 的 `preprocess_relation_rtes` 仍会处理 `users`、`orders` 两个 RTE。

---

### 2.8 `grouping_planner`：在预处理后的 `Query` 上造 Path

`subquery_planner` 在 **1373 行** 调用 `grouping_planner`（定义在 `planner.c:1775`）。调用返回后，同一函数内 **`set_cheapest`**（1395）、**`return root`**（1397），才把 `PlannerInfo` 交回 `standard_planner`。

```c
	grouping_planner(root, tuple_fraction, setops);     /* planner.c:1373 */
```

`grouping_planner` 内部按 SQL 语义自下而上叠 Path：**scan/join → GROUP/窗口 → ORDER → LIMIT**，DML 再挂 **`ModifyTablePath`**。本例：`users`⨝`orders` → `create_grouping_paths`（`count` + HAVING）→ `create_ordered_paths` → `create_limit_path`（与 §2.3 一致）。

#### 2.8.1 目标列与 DML：`preprocess_targetlist`

`grouping_planner` 里较早的一步（`planner.c:1908`）：

```c
		preprocess_targetlist(root);
```

在 **`planner.c:1908` 看不到 `AddForeignUpdateTargets` 字样**；UPDATE/DELETE/MERGE 且目标是 **外表** 时，调用链是：

```text
preprocess_targetlist()           preptlist.c:66
  add_row_identity_columns()      preptlist.c:127
    AddForeignUpdateTargets()     appendinfo.c:994
      postgresAddForeignUpdateTargets   postgres_fdw.c:1964
```

**`preptlist.c` 何时进 `add_row_identity_columns`**（`preptlist.c:121-128`）：

```c
	if ((command_type == CMD_UPDATE || command_type == CMD_DELETE ||
		 command_type == CMD_MERGE) &&
		!target_rte->inh)
	{
		root->processed_tlist = tlist;
		add_row_identity_columns(root, result_relation,
								 target_rte, target_relation);
```

**`appendinfo.c` 何时调 FDW**（`appendinfo.c:985-996`）：

```c
	else if (relkind == RELKIND_FOREIGN_TABLE)
	{
		fdwroutine = GetFdwRoutineForRelation(target_relation, false);
		if (fdwroutine->AddForeignUpdateTargets != NULL)
			fdwroutine->AddForeignUpdateTargets(root, rtindex,
												target_rte, target_relation);
```

对 **UPDATE/DELETE/MERGE 的目标外表**，在 `processed_tlist` 中追加 **junk 列**（如远程 `ctid`、整行 `Var`），供 `ModifyTable` 定位待修改行。

纯 **`SELECT`** 走 `preprocess_targetlist`，通常不进 `add_row_identity_columns`，因而不会调 `AddForeignUpdateTargets`。

#### 2.8.2 scan/join：`query_planner()`

注释写明这一段对应 **FROM/WHERE**（`planner.c:1988-1995`）：

```c
		current_rel = query_planner(root, standard_qp_callback, &qp_extra);
```

返回值 `current_rel` 表示 **join + where 之后** 的逻辑结果（本例：`u.active` 与 JOIN ON 已作为 restrict），上面挂多种候选 **Path**。  
**`RelOptInfo`、`build_simple_rel`、外表 `GetForeignRelSize` / `GetForeignPaths`** 都在 `query_planner` 内部完成，见 §2.9；只为 **`jointree` 里的 `RangeTblRef(1)`、`(2)`** 建基表 rel，不为未引用的 rtable 项建 rel。

#### 2.8.3 GROUP / ORDER / LIMIT / DML

| 顺序 | 函数 | SQL | FDW（若适用） |
|------|------|-----|----------------|
| 1 | `create_grouping_paths` | GROUP / HAVING | `GetForeignUpperPaths` |
| 2 | `create_window_paths` | 窗口 | 同上 |
| 3 | `create_ordered_paths` | ORDER BY | 同上 |
| 4 | `create_limit_path` | LIMIT | — |
| 5 | `create_modifytable_path` | DML | `planner.c:2452` |

---

### 2.9 `query_planner` 内部：从 `jointree` 到外表 Path

**入口**：`planmain.c:54` 的 `query_planner()`。

#### （1）`jointree` 与 `rtable` 的分工

主线说明见 **§2.3**。此处补典型边界：**纯 `INSERT INTO ft VALUES (...)`** 的目标表可在 `rtable` 中但不在 `jointree` 当扫描源 → 不 `build_simple_rel` → 通常不调 `GetForeignRelSize`；视图展开等遗留 RTE 同理。本例只为 `jointree` 中的 `RangeTblRef(1)`、`(2)` 建基表 rel。

**源码**：`planmain.c:168-173` + `add_base_rels_to_query`（`initsplan.c:163-186`）。

#### （2）`add_base_rels_to_query` → `build_simple_rel`

**做什么**：

1. **递归遍历 `jointree`**（`FromExpr` / `JoinExpr` / `RangeTblRef`）。
2. 每遇到一个 **`RangeTblRef(n)`**，调用 **`build_simple_rel(root, n)`**（`relnode.c:212`）。
3. `build_simple_rel` 为 **`rtable` 第 n 项** 创建一个空的 **`RelOptInfo`**（`pathlist`  initially 空、`fdwroutine` 先 NULL）。

将 `rtable` 第 `n` 项对应的逻辑关系实例化为可供路径生成的 **`RelOptInfo`**。

#### （3）`get_relation_info`（基表 / 外表）

对 **`RTE_RELATION`**（含普通表、**外表**），`build_simple_rel` 里会调 **`get_relation_info`**（`relnode.c:365` → `plancat.c:121` 起）：

- 读 **统计信息、索引列表**；
- 若 **`RELKIND_FOREIGN_TABLE`** → **`rel->fdwroutine = GetFdwRoutineForRelation(...)`**（`plancat.c:532-546`）。

此时 **仍未** 调 `GetForeignRelSize`；只是挂好 FDW 函数表。

#### （4）`deconstruct_jointree` → `make_one_rel`

- 把 **ON / WHERE** 拆到各 rel / join 的 restrict 列表；
- **`make_one_rel`**：基表估大小、生成 Path、**枚举 JOIN 顺序**；
- 外表在 **`set_rel_size` / `set_rel_pathlist`** 里走 **`set_foreign_*`** → **`GetForeignRelSize` / `GetForeignPaths`**（`allpaths.c:986`、`1007`）。若 **`orders` 为外表**，在本步对 `orders` 的 `RelOptInfo` 估行数、生成 `ForeignPath`；本地表仍走 `set_baserel_size` / 索引 Path。

**返回给 `grouping_planner`**：`current_rel`（join rel，本例 LEFT JOIN 后的中间结果）。

---

### 2.10 `create_plan`：Path → Plan（含 FDW）

`subquery_planner` 返回后，`standard_planner` 选出 **`best_path`**（`planner.c:542`），**`create_plan`**（`544`）：

- 扫描路径 → **`GetForeignPlan`**（`createplan.c:4004`）→ 计划节点 **`ForeignScan`**（本例 SELECT 若外表参与扫描则出现在 join 子树一侧）
- DML → **`PlanForeignModify`** / **`PlanDirectModify`**（`createplan.c:7204-7214`）；本例 **CMD_SELECT** 不走

**本例 Plan 草图**（本地表、一种可能）：`Limit` → `Sort` → `HashAggregate`（GROUP + HAVING）→ `Hash Left Join`（或 Nested Loop）→ `Seq Scan users` + `Seq Scan orders`。外表时 `orders` 一侧常为 **`ForeignScan`**。

---

## 3. 何时因语句类型分叉

**源码**：`src/include/nodes/nodes.h:272-281`（`CmdType`）

| 类型 | `commandType` | 规划链 |
|------|---------------|--------|
| 查询 | `CMD_SELECT` | §2 全程；无 `ModifyTable` |
| DML | `CMD_INSERT` / `UPDATE` / `DELETE` / `MERGE` | §2.8.1 + §2.8.2 `GetForeign*` + §2.8.3 `ModifyTable` |
| DDL / 工具 | **`CMD_UTILITY`** | `utilityStmt` 另路，不经 `query_planner` 的 FDW 扫描 |

`CREATE FOREIGN TABLE`、`IMPORT FOREIGN SCHEMA` 走 **工具回调**（`postgres_fdw.c:816`），不在上表路径里。

---

## 4. 何时因「外表」分叉

### 4.1 没有 `RTE_FOREIGN`

外表在 RTE 层是 **`RTE_RELATION`** + **`rte->relkind == RELKIND_FOREIGN_TABLE`**（`parsenodes.h:1120` 起 `RTEKind` 无 FOREIGN 项）。

### 4.2 分叉点（接 §2.9）

| 步骤 | 文件 | 外表分支 |
|------|------|----------|
| `get_relation_info` | `plancat.c:532-546` | 取 `fdwroutine` |
| `set_rel_size` | `allpaths.c:435-439` | `set_foreign_size` → `GetForeignRelSize` |
| `set_rel_pathlist` | `allpaths.c:532-536` | `set_foreign_pathlist` → `GetForeignPaths` |
| `create_plan` | `createplan.c:4004` | `GetForeignPlan` → `ForeignScan` |

**源码**（`GetForeignRelSize` 调用）：

```c
	rel->fdwroutine->GetForeignRelSize(root, rel, rte->relid);   /* allpaths.c:986 */
```

---

## 5. SQL 走通：回调顺序

外表 `ft`，已 `CREATE EXTENSION postgres_fdw`。

### 5.1 `SELECT * FROM ft WHERE id = 1;`

| 阶段（§2） | FDW 回调 / 要点 |
|------------|-----------------|
| §2.7 预处理 | 通常 no-op |
| §2.9 `make_one_rel` | `GetForeignRelSize` → `GetForeignPaths` |
| §2.8.3 上层 | 可能 `GetForeignUpperPaths` |
| §2.10 `create_plan` | `GetForeignPlan` → `ForeignScan` |

### 5.2 `UPDATE ft SET col = 1 WHERE id = 1;`

| 阶段 | FDW 回调 / 要点 |
|------|-----------------|
| §2.8.1 | `postgresAddForeignUpdateTargets` |
| §2.8.2 / §2.9 | `GetForeignRelSize` / `GetForeignPaths` |
| §2.8.3 | `create_modifytable_path` |
| §2.10 | `postgresPlanForeignModify`（或 `PlanDirectModify`） |

**INSERT `VALUES`**：目标 `ft` 常 **不在 jointree** → 通常 **无** `GetForeignRelSize`；仍有 `PlanForeignModify`（`createplan.c:7113-7115` 注释）。

---

## 6. postgres_fdw 注册回调一览

`postgres_fdw_handler()`（`postgres_fdw.c:770-830`）。

### 规划期

| 字段 | 实现 | 内核调用（与 §2 对应） |
|------|------|------------------------|
| `GetForeignRelSize` | `postgres_fdw.c:841` | §2.9 `allpaths.c:986` |
| `GetForeignPaths` | `postgres_fdw.c:1232` | §2.9 `allpaths.c:1007` |
| `GetForeignPlan` | `postgres_fdw.c:1446` | §2.10 `createplan.c:4004` |
| `GetForeignJoinPaths` | `postgres_fdw.c:7212` | §7 |
| `GetForeignUpperPaths` | `postgres_fdw.c:7597` | §2.8.3 |
| `AddForeignUpdateTargets` | `postgres_fdw.c:1964` | §2.8.1 |
| `PlanForeignModify` | `postgres_fdw.c:1992` | §2.10 |
| `PlanDirectModify` | `postgres_fdw.c:2664` | §2.10 |

### 执行期（列名）

`BeginForeignScan`, `IterateForeignScan`, `EndForeignScan`, `BeginForeignModify`, `ExecForeignInsert`, `ExecForeignUpdate`, `ExecForeignDelete`, `EndForeignModify`, … 见 [§9](#9-执行期回调速查)。

---

## 7. Join / Upper 下推

- **`GetForeignJoinPaths`**：`joinpath.c:362-364`；同 server 两外表 JOIN。
- **`GetForeignUpperPaths`**：`planner.c:2495-2503` 等；聚合/排序下推远程。

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
| SQL 主线五站 | `sql_pipeline_faq.md` §1.1、§5（与本文 §2 同一条示例 SQL） |
| `planner` 入口 | `planner.c:333`、`planmain.c:54` |
| `RelOptInfo` | `optimizer/util/relnode.c:212` |
| FDW API | `src/include/foreign/fdwapi.h:208` |
| postgres_fdw | `contrib/postgres_fdw/postgres_fdw.c:770-830` |

---

*行号对应当前 workspace；合并上游后可能偏移。*
