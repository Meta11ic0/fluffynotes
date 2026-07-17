# 最小 FDW 代码骨架与回调说明

> 定位：对照规划/执行链路的 **最小代码骨架**与 **7 个扫描回调**说明，不是「从零搭到可安装扩展」教程。**不含** `Makefile` / `.control` / 编译安装步骤；SQL 注册见 §4.2。骨架里的 `create_foreignscan_path` 按 **PostgreSQL 18 / current** 签名书写（含 `disabled_nodes`、`fdw_restrictinfo`）；PG 16/17 参数列表不同，见 §5.2。

以 `test_fdw` 为例，目标是看清 handler、validator 与扫描回调的触发顺序。

---

## 1. 先明确最小目标

一个最小 FDW（只支持最基础 `SELECT` 扫描）需要三部分：

1. `handler`（必须）：返回 `FdwRoutine *`
2. `validator`（建议）：校验并打印 options
3. `FdwRoutine` 基础 7 回调（核心）：
  - `GetForeignRelSize`
  - `GetForeignPaths`
  - `GetForeignPlan`
  - `BeginForeignScan`
  - `IterateForeignScan`
  - `ReScanForeignScan`
  - `EndForeignScan`

### 1.1 业务 SQL 与规划器名词的对照

从**业务 SQL** 看，`FROM` 子句里的**外表（`FOREIGN TABLE`）**就是一张**逻辑上的表**，可像普通表一样写 `SELECT ... WHERE ... JOIN ...`（具体能否下推、能否 join 取决于 FDW 能力）。**FDW** 则是这类表背后的访问协议与实现，由 `CREATE FOREIGN DATA WRAPPER` 注册，`SERVER` / `USER MAPPING` / 表级 `OPTIONS` 提供连接与行为参数。

从规划器/执行器内核看，后文出现的类型可先对齐为：


| 名词                     | 白话含义                                                                                 |
| ---------------------- | ------------------------------------------------------------------------------------ |
| `RelOptInfo`           | 对**当前查询**里某个关系（单表、join 结果、上层关系等）的**规划期描述**：行数估计、可选路径列表等都挂在这里。                        |
| `Path`                 | **候选访问方案**：一种“打算怎么读这份数据”的描述，带成本；优化器会从中挑便宜的。`ForeignPath` 是其中一种，表示走 FDW 扫描。           |
| `ForeignScan`（Plan 节点） | **执行期计划节点**：已经定稿的“用 FDW 扫这份关系”的方案，内含 `targetlist`、`qual`、以及 `fdw_private` 等执行时要用的信息。 |
| `ForeignScanState`     | **执行期状态**：对应正在跑的 `ForeignScan` 计划，`fdw_state` 给 FDW 挂私有数据。                           |


**规划阶段**在 `RelOptInfo` 向 `pathlist` 注册候选 `Path`（通常使用 `add_path`），选中后再落成 **`ForeignScan` 计划**。**执行阶段**则用 **`ForeignScanState`** 驱动 `Begin/Iterate/.../End` 回调。

---

## 2. 最小代码骨架

下面骨架在 **handler、各扫描回调**里加入了最基础的 `ereport(NOTICE, …)`，方便第一次跑通时对照调用顺序，确认规划/执行链路是否走到预期位置。**实际开发中通常不需要**这些打印。功能稳定后应去掉或改为受控的诊断开关，避免客户端/日志噪音与无谓开销（`Iterate` 路径上尤其不要常驻大段 `NOTICE`）。

```c
#include "postgres.h"            /* Datum, PG_FUNCTION_ARGS, PG_RETURN_*, ereport/errmsg */
#include "fmgr.h"                /* PG_FUNCTION_INFO_V1, PG_MODULE_MAGIC */
#include "foreign/fdwapi.h"      /* FdwRoutine, PlannerInfo/RelOptInfo/ForeignScanState 等回调签名 */
#include "optimizer/pathnode.h"  /* add_path, create_foreignscan_path (testGetForeignPaths 用) */
#include "optimizer/planmain.h"  /* make_foreignscan (testGetForeignPlan 用) */
#include "optimizer/restrictinfo.h" /* extract_actual_clauses (testGetForeignPlan 用) */
#include "executor/tuptable.h"   /* ExecClearTuple (testIterateForeignScan 用) */
#include "access/reloptions.h"   /* untransformRelOptions (validator 解析 text[] options) */
#include "commands/defrem.h"     /* defGetString (validator 把 DefElem value 转字符串) */

PG_MODULE_MAGIC;

PG_FUNCTION_INFO_V1(test_fdw_handler);
PG_FUNCTION_INFO_V1(test_fdw_validator);

static void testGetForeignRelSize(PlannerInfo *root, RelOptInfo *baserel, Oid foreigntableid);
static void testGetForeignPaths(PlannerInfo *root, RelOptInfo *baserel, Oid foreigntableid);
static ForeignScan *testGetForeignPlan(PlannerInfo *root, RelOptInfo *baserel, Oid foreigntableid,
                                       ForeignPath *best_path, List *tlist, List *scan_clauses, Plan *outer_plan);
static void testBeginForeignScan(ForeignScanState *node, int eflags);
static TupleTableSlot *testIterateForeignScan(ForeignScanState *node);
static void testReScanForeignScan(ForeignScanState *node);
static void testEndForeignScan(ForeignScanState *node);

Datum
test_fdw_handler(PG_FUNCTION_ARGS)
{
    FdwRoutine *fdwroutine = makeNode(FdwRoutine);

    ereport(NOTICE, (errmsg("test_fdw: handler called")));

    fdwroutine->GetForeignRelSize = testGetForeignRelSize;
    fdwroutine->GetForeignPaths = testGetForeignPaths;
    fdwroutine->GetForeignPlan = testGetForeignPlan;
    fdwroutine->BeginForeignScan = testBeginForeignScan;
    fdwroutine->IterateForeignScan = testIterateForeignScan;
    fdwroutine->ReScanForeignScan = testReScanForeignScan;
    fdwroutine->EndForeignScan = testEndForeignScan;

    PG_RETURN_POINTER(fdwroutine);
}

Datum
test_fdw_validator(PG_FUNCTION_ARGS)
{
    List     *options_list = untransformRelOptions(PG_GETARG_DATUM(0));
    Oid       catalog = PG_GETARG_OID(1);
    ListCell *cell;

    ereport(NOTICE, (errmsg("test_fdw: validator called, catalog=%u", catalog)));

    foreach(cell, options_list)
    {
        DefElem *def = (DefElem *) lfirst(cell);
        char    *val = defGetString(def);
        ereport(NOTICE, (errmsg("test_fdw: option key=%s, value=%s",
                                def->defname, val ? val : "<null>")));
    }

    PG_RETURN_VOID();
}

static void
testGetForeignRelSize(PlannerInfo *root, RelOptInfo *baserel, Oid foreigntableid)
{
    ereport(NOTICE,
            (errmsg("test_fdw: GetForeignRelSize foreigntableid=%u", foreigntableid)));
    baserel->rows = 1;
}

static void
testGetForeignPaths(PlannerInfo *root, RelOptInfo *baserel, Oid foreigntableid)
{
    ereport(NOTICE,
            (errmsg("test_fdw: GetForeignPaths foreigntableid=%u", foreigntableid)));
    add_path(baserel, (Path *) create_foreignscan_path(root, baserel,
                                                        NULL,			/* default pathtarget */
                                                        baserel->rows,
                                                        0,				/* disabled_nodes */
                                                        0,				/* startup_cost */
                                                        0,				/* total_cost */
                                                        NIL,			/* pathkeys */
                                                        NULL,			/* required_outer */
                                                        NULL,			/* fdw_outerpath */
                                                        NIL,			/* fdw_restrictinfo */
                                                        NIL));			/* fdw_private */
}

static ForeignScan *
testGetForeignPlan(PlannerInfo *root, RelOptInfo *baserel, Oid foreigntableid,
                   ForeignPath *best_path, List *tlist, List *scan_clauses, Plan *outer_plan)
{
    ereport(NOTICE,
            (errmsg("test_fdw: GetForeignPlan foreigntableid=%u", foreigntableid)));
    scan_clauses = extract_actual_clauses(scan_clauses, false);
    return make_foreignscan(tlist, scan_clauses, baserel->relid, NIL, NIL, NIL, NIL, outer_plan);
}

static void
testBeginForeignScan(ForeignScanState *node, int eflags)
{
    ereport(NOTICE, (errmsg("test_fdw: BeginForeignScan eflags=%d", eflags)));
}

static TupleTableSlot *
testIterateForeignScan(ForeignScanState *node)
{
    ereport(NOTICE, (errmsg("test_fdw: IterateForeignScan")));
    return ExecClearTuple(node->ss.ss_ScanTupleSlot);
}

static void
testReScanForeignScan(ForeignScanState *node)
{
    ereport(NOTICE, (errmsg("test_fdw: ReScanForeignScan")));
}

static void
testEndForeignScan(ForeignScanState *node)
{
    ereport(NOTICE, (errmsg("test_fdw: EndForeignScan")));
}
```

调试时在会话里执行 `SET client_min_messages TO notice;` 才能稳定看到上述 `NOTICE`。`IterateForeignScan` 在一条 `SELECT` 中可能被调用多次，日志会刷屏，属正常现象。

---

## 3. PG 函数模型

### 3.1 先用一个最小业务函数建立直觉：`add_one`

先看一个完整、可直接给用户 `SELECT` 调用的 C 函数：

```c
#include "postgres.h"
#include "fmgr.h"

PG_MODULE_MAGIC;
PG_FUNCTION_INFO_V1(add_one);

Datum
add_one(PG_FUNCTION_ARGS)
{
    int32 arg = PG_GETARG_INT32(0);
    PG_RETURN_INT32(arg + 1);
}
```

对应 SQL 注册：

```sql
CREATE FUNCTION add_one(int)
RETURNS int
AS 'MODULE_PATHNAME', 'add_one'
LANGUAGE C STRICT;
```

使用方式：

```sql
SELECT add_one(41);  -- 42
```

这个例子说明 PG 的函数模型分两层：

- C 层：统一写成 `Datum func(PG_FUNCTION_ARGS)`，通过 `PG_GETARG_*`/`PG_RETURN_*` 取参与返回
- SQL 层：声明参数类型、返回类型、语言和符号映射；`AS 'MODULE_PATHNAME', 'add_one'` 表示「当前扩展共享库里的符号 `add_one`」（安装时由扩展机制替换路径）

理解了 `add_one` 再看 FDW 的 `handler`/`validator`：它们也是 V1 函数，只是调用方通常是内核（加载 FDW / 校验 OPTIONS）而不是业务 `SELECT`。

### 3.2 handler 和 validator 的调用方式

写 FDW 时，两个入口如下：

```c
Datum test_fdw_handler(PG_FUNCTION_ARGS)
Datum test_fdw_validator(PG_FUNCTION_ARGS)
```

它们通常由服务端内核通过 fmgr（Function Manager，PostgreSQL 的函数调用管理器，负责按系统目录元数据定位函数符号并完成统一调用）在特定时机调用。下面看内核中的调用代码：

`handler` 的调用路径（`src/backend/foreign/foreign.c`）：

```c
FdwRoutine *
GetFdwRoutine(Oid fdwhandler)
{
    Datum      datum;
    FdwRoutine *routine;

    datum = OidFunctionCall0(fdwhandler);
    routine = (FdwRoutine *) DatumGetPointer(datum);

    if (routine == NULL || !IsA(routine, FdwRoutine))
        elog(ERROR, "foreign-data wrapper handler function %u did not return an FdwRoutine struct",
             fdwhandler);
    return routine;
}
```

`validator` 的调用路径（`src/backend/commands/foreigncmds.c`）：

```c
if (OidIsValid(fdwvalidator))
{
    Datum   valarg = result;

    if (DatumGetPointer(valarg) == NULL)
        valarg = PointerGetDatum(construct_empty_array(TEXTOID));

    OidFunctionCall2(fdwvalidator, valarg, ObjectIdGetDatum(catalogId));
}
```

`handler`、`validator` 虽非典型业务函数，但一样由 fmgr 调用，因此同样需要 `PG_FUNCTION_INFO_V1`。若对 `test_fdw_handler()` 做 `SELECT` 调用，返回的是 `fdw_handler` 内部伪类型（指针封装），无业务语义。

### 3.3 PG_MODULE_MAGIC：.so 加载时的安全门

在 `.c` 文件的最顶部（只能出现一次）：

```c
PG_MODULE_MAGIC;
```

它展开后（`src/include/fmgr.h`）会向 `.so` 中嵌入一个 `pg_magic_data` 结构，记录当前编译时的 PG 主版本号、`FUNC_MAX_ARGS`、`INDEX_MAX_KEYS` 等 ABI 关键参数：

```c
/* src/include/fmgr.h */
#define PG_MODULE_MAGIC \
extern PGDLLEXPORT const Pg_magic_struct *PG_MAGIC_FUNCTION_NAME(void); \
const Pg_magic_struct * \
PG_MAGIC_FUNCTION_NAME(void) \
{ \
    static const Pg_magic_struct Pg_magic_data = PG_MODULE_MAGIC_DATA; \
    ...
```

服务器加载 `.so` 时首先检查这个结构，若版本不匹配则拒绝加载并报错：

```
ERROR: incompatible library "...test_fdw.so": missing magic block
```

缺少 `PG_MODULE_MAGIC` 是开发新 FDW 时最常见的首次报错。

---

### 3.4 什么是 V1 调用约定，哪些函数需要声明为 V1

PostgreSQL 目前只支持一种 C 函数调用约定，称为 Version-1，所有经 fmgr 调用的动态库函数都必须遵守。声明方式是在函数定义之前写：

```c
PG_FUNCTION_INFO_V1(funcname);
```

它展开后（`src/include/fmgr.h` 第 415 行）同时做了两件事：

```c
/* src/include/fmgr.h */
#define PG_FUNCTION_INFO_V1(funcname) \
extern PGDLLEXPORT Datum funcname(PG_FUNCTION_ARGS); \
extern PGDLLEXPORT const Pg_finfo_record * CppConcat(pg_finfo_,funcname)(void); \
const Pg_finfo_record * \
CppConcat(pg_finfo_,funcname) (void) \
{ \
    static const Pg_finfo_record my_finfo = { 1 }; \
    return &my_finfo; \
}
```

1. 宏展开会生成对外导出声明：`extern PGDLLEXPORT Datum funcname(PG_FUNCTION_ARGS);`，供动态链接器找到符号。
2. **生成 `pg_finfo_funcname()` 查询函数**：返回 `{ 1 }` 这个结构，告诉 fmgr "此为 V1 约定"。fmgr 在加载符号时会先调用 `pg_finfo_funcname()` 确认调用版本，再按 V1 方式调用。

**哪些函数需要 V1？**

规则很简单：**任何通过 `CREATE FUNCTION ... LANGUAGE C` 注册到数据库、由 fmgr 负责调用的函数**，都需要 `PG_FUNCTION_INFO_V1`。

给用户 `SELECT` 的普通业务函数，例如 `add_one`：

```c
PG_FUNCTION_INFO_V1(add_one);
Datum
add_one(PG_FUNCTION_ARGS)
{
    int32 arg = PG_GETARG_INT32(0);
    PG_RETURN_INT32(arg + 1);
}
```

FDW 的 `handler` 和 `validator`，虽然不给用户主动调用，但一样由 fmgr 调用，所以一样需要：

```c
PG_FUNCTION_INFO_V1(test_fdw_handler);
PG_FUNCTION_INFO_V1(test_fdw_validator);
```

回调函数（`testGetForeignRelSize` 等）**不需要**。它们不通过 fmgr，而是由 handler 直接把函数指针填入 `FdwRoutine`，规划器和执行器直接通过指针调用，绕开了 fmgr 那套机制。

---

### 3.5 V1 函数的真实 C 签名：宏展开后是什么

宏展开后为：

```c
Datum test_fdw_validator(PG_FUNCTION_ARGS)
```

对 GCC 来说展开是：

```c
uintptr_t test_fdw_validator(FunctionCallInfo fcinfo)
```

再展开 `FunctionCallInfo`（`src/include/fmgr.h` 第 38 行）：

```c
typedef struct FunctionCallInfoBaseData *FunctionCallInfo;
```

GCC 最终看到的签名是：

```c
uintptr_t test_fdw_validator(struct FunctionCallInfoBaseData *fcinfo)
```

**这就是所有 V1 函数的统一 C 签名**，不管 SQL 层声明它有几个参数、是什么类型。

两个关键宏的定义位置：

`Datum` 定义在 `src/include/postgres.h`：

```c
/* src/include/postgres.h */
typedef uintptr_t Datum;
```

`PG_FUNCTION_ARGS` 定义在 `src/include/fmgr.h`：

```c
/* src/include/fmgr.h 第 193 行 */
#define PG_FUNCTION_ARGS    FunctionCallInfo fcinfo
```

编译 `.c -> .so` 时，编译器通过 `#include` 链找到这些定义，SQL 文件不参与编译。

---

### 3.6 同一个宏签名，如何体现不同的参数和返回值

既然所有 V1 函数签名都是 `uintptr_t f(struct FunctionCallInfoBaseData *fcinfo)`，那 `add_one(int)` 和 `test_fdw_validator(text[], oid)` 是如何区分参数的？

**参数全部打包在 `fcinfo->args[N].value` 里**，类型信息由 SQL 注册时写入 `pg_proc`，运行时由 fmgr 负责按正确类型传入。C 代码通过 `PG_GETARG_`* 系列宏按位置取出并做类型转换。

`PG_GETARG_`* 宏（`src/include/fmgr.h`）：

```c
#define PG_GETARG_DATUM(n)   (fcinfo->args[n].value)           /* 原始 Datum，不做转换 */
#define PG_GETARG_INT32(n)   DatumGetInt32(PG_GETARG_DATUM(n)) /* Datum -> int32 */
#define PG_GETARG_INT64(n)   DatumGetInt64(PG_GETARG_DATUM(n)) /* Datum -> int64 */
#define PG_GETARG_FLOAT8(n)  DatumGetFloat8(PG_GETARG_DATUM(n))/* Datum -> float8 */
#define PG_GETARG_BOOL(n)    DatumGetBool(PG_GETARG_DATUM(n))  /* Datum -> bool */
#define PG_GETARG_OID(n)     DatumGetObjectId(PG_GETARG_DATUM(n))/* Datum -> Oid */
#define PG_GETARG_TEXT_PP(n) DatumGetTextPP(PG_GETARG_DATUM(n))/* Datum -> text*（变长） */
#define PG_GETARG_POINTER(n) DatumGetPointer(PG_GETARG_DATUM(n))/* Datum -> void* */
```

同理，返回值统一是 `uintptr_t`（`Datum`），通过 `PG_RETURN_*` 宏把实际值打包进去：

```c
#define PG_RETURN_VOID()     return (Datum) 0              /* 无返回值 */
#define PG_RETURN_INT32(x)   return Int32GetDatum(x)       /* int32 -> Datum */
#define PG_RETURN_FLOAT8(x)  return Float8GetDatum(x)      /* float8 -> Datum */
#define PG_RETURN_BOOL(x)    return BoolGetDatum(x)        /* bool -> Datum */
#define PG_RETURN_POINTER(x) return PointerGetDatum(x)     /* 任意指针 -> Datum */
#define PG_RETURN_TEXT_P(x)  PG_RETURN_POINTER(x)          /* text* -> Datum（通过指针） */
```

以 handler 为例，它返回一个 `FdwRoutine *`，通过 `PG_RETURN_POINTER` 将指针地址打包成 `uintptr_t` 交给 fmgr，fmgr 再把这个 `Datum` 转回指针交给规划器使用：

```c
PG_RETURN_POINTER(fdwroutine);
/* 展开为：return PointerGetDatum(fdwroutine); */
/* 即：return (uintptr_t)(void *)(fdwroutine); */
```

`Datum`（`uintptr_t`）是 PG 的万能容器，足够大，既能装整数值（`int32`、`bool` 等按值传递类型），也能装指针地址（`text *`、`FdwRoutine *` 等引用类型）。V1 统一 ABI 靠它做通道，类型安全则靠宏在两端做转换。

**`PG_FUNCTION_ARGS`** 在不同函数里看起来相同，但参数内容不同。宏本身只展开为 `FunctionCallInfo fcinfo`，而 `FunctionCallInfo` 本质是一个指针类型，指向 `FunctionCallInfoBaseData`。它的定义（相对路径：`src/include/fmgr.h`）是：

```c
#define PG_FUNCTION_ARGS    FunctionCallInfo fcinfo
typedef struct FunctionCallInfoBaseData *FunctionCallInfo;

typedef struct FunctionCallInfoBaseData
{
    FmgrInfo   *flinfo;
    fmNodePtr   context;
    fmNodePtr   resultinfo;
    Oid         fncollation;
    bool        isnull;
    short       nargs;
    NullableDatum args[FLEXIBLE_ARRAY_MEMBER];
} FunctionCallInfoBaseData;
```

“同一个函数签名”只是 ABI 统一。真正的参数个数和值都在 `fcinfo->nargs` 与 `fcinfo->args[]` 里，由 fmgr 按 `pg_proc` 注册信息在运行时填充，C 代码只负责按位置解包（`PG_GETARG_*`）。

---

## 4. handler 的职责与 validator 的参数解析

### 4.1 handler 的职责：交出回调函数指针表

handler 函数只做一件事：分配一个 `FdwRoutine`，填好函数指针，经 `GetFdwRoutine` 交给规划器/执行器。

`FdwRoutine` 定义在 `src/include/foreign/fdwapi.h`，是一张回调函数指针表：

```c
typedef struct FdwRoutine
{
    NodeTag     type;   /* Node 类型标记，makeNode 自动设置 */

    /* 扫描必须实现的 7 个回调 */
    GetForeignRelSize_function  GetForeignRelSize;
    GetForeignPaths_function    GetForeignPaths;
    GetForeignPlan_function     GetForeignPlan;
    BeginForeignScan_function   BeginForeignScan;
    IterateForeignScan_function IterateForeignScan;
    ReScanForeignScan_function  ReScanForeignScan;
    EndForeignScan_function     EndForeignScan;

    /* 后续还有 JOIN、DML、并行、异步等可选回调 ... */
} FdwRoutine;
```

分配这个结构用 `makeNode(FdwRoutine)`，它会调用 `palloc0fast` 分配并清零内存，再设置 `type` 字段。

调用链（`src/include/nodes/nodes.h`）：

```c
#define makeNode(_type_)   ((_type_ *) newNode(sizeof(_type_), T_##_type_))

/* GCC/Clang：用语句表达式，_result 是局部临时变量，直接返回 */
#ifdef __GNUC__
#define newNode(size, tag) \
({  Node *_result; \
    AssertMacro((size) >= sizeof(Node)); \
    _result = (Node *) palloc0fast(size); \
    _result->type = (tag); \
    _result; \
})
/* 非 GCC（如 MSVC）：语句表达式不可用，改用全局临时变量 newNodeMacroHolder 承接中间值 */
#else
extern PGDLLIMPORT Node *newNodeMacroHolder;
#define newNode(size, tag) \
( \
    AssertMacro((size) >= sizeof(Node)), \
    newNodeMacroHolder = (Node *) palloc0fast(size), \
    newNodeMacroHolder->type = (tag), \
    newNodeMacroHolder \
)
#endif
```

两个分支做的事完全相同，区别仅在于 GCC/Clang 支持语句表达式（`({ ... })` 可作为表达式返回值），而 MSVC 等不支持，改用全局变量绕行。

结果：`palloc0fast` 分配并清零内存，所以 `FdwRoutine` 中所有函数指针字段初始均为 `NULL`，未手工赋值的回调不需要显式写 `= NULL`。

**为什么回调函数要写成 `static`？**

`testGetForeignRelSize` 这类回调只在本文件内部被 handler 拿去填指针，不需要对外导出符号。写 `static` 的好处有三点：

- 编译器不会为它生成导出符号，减小 `.so` 体积
- 多个 FDW 在同一进程中可以有同名回调而不冲突
- 明确表达"此函数仅本文件可见"的意图

规范命名：回调用驼峰加前缀（`testGetForeignRelSize`），对外暴露给 fmgr 的入口用下划线（`test_fdw_handler`、`test_fdw_validator`）。

---

### 4.2 SQL 注册：sql 文件做了什么

SQL 安装脚本（`test_fdw--1.0.sql`）内容：

```sql
CREATE FUNCTION test_fdw_handler()
RETURNS fdw_handler
AS 'MODULE_PATHNAME'
LANGUAGE C STRICT;

CREATE FUNCTION test_fdw_validator(text[], oid)
RETURNS void
AS 'MODULE_PATHNAME'
LANGUAGE C STRICT;

CREATE FOREIGN DATA WRAPPER test_fdw
  HANDLER test_fdw_handler
  VALIDATOR test_fdw_validator;
```

这些语句在 `CREATE EXTENSION test_fdw` 时执行，将函数对象写入 `pg_proc`，将 FDW 对象写入 `pg_foreign_data_wrapper`。它们记录了参数类型、返回类型、`.so` 路径、C 符号名等元数据，不参与 `.c -> .so` 编译，也不生成任何 C 函数签名。

扩展版本文件说明：

- `test_fdw.control` 中 `default_version = '1.0'` 决定 `CREATE EXTENSION` 默认执行哪个 SQL 脚本。
- 若默认版本改为 `2.0`，`CREATE EXTENSION test_fdw` 会直接执行 `test_fdw--2.0.sql`，不会自动先执行 `1.0`，也不会自动执行 `test_fdw--1.0--2.0.sql`。
- `test_fdw--1.0--2.0.sql` 仅用于“已安装 1.0 实例”的升级路径：`ALTER EXTENSION test_fdw UPDATE TO '2.0'` 时才会被使用。
- 实践建议：若系统里同时存在“全新安装 2.0”和“从 1.0 升级到 2.0”两类场景，应同时提供 `test_fdw--2.0.sql` 与 `test_fdw--1.0--2.0.sql` 两个脚本。

---

### 4.3 validator 的入参从哪来：看内核调用点

validator 在 SQL 层声明为 `(text[], oid)`，这两个参数从哪来？看内核实际调用处（`src/backend/commands/foreigncmds.c`）的代码：

```c
/* src/backend/commands/foreigncmds.c 第 192-202 行 */
if (OidIsValid(fdwvalidator))
{
    Datum   valarg = result;   /* result 是 options 序列化后的 text[] Datum */

    /* 空 options 时用空数组代替 NULL，确保 validator 不会因 STRICT 收到 NULL */
    if (DatumGetPointer(valarg) == NULL)
        valarg = PointerGetDatum(construct_empty_array(TEXTOID));

    OidFunctionCall2(fdwvalidator, valarg, ObjectIdGetDatum(catalogId));
}
```

两个参数的来源分别为：

- **第 1 个参数 `valarg`（对应 `text[]`）**：当前对象上的所有 OPTIONS 经过 ADD/SET/DROP 操作合并后，序列化为 `text[]` 格式的 `Datum`。
- **第 2 个参数 `ObjectIdGetDatum(catalogId)`（对应 `oid`）**：正在操作的系统目录 OID，例如 `ForeignDataWrapperRelationId`、`ForeignServerRelationId`、`UserMappingRelationId`、`ForeignTableRelationId`。validator 可以通过这个 OID 区分当前是在校验哪类对象的选项。

同一套 validator 在 `CREATE/ALTER` 四类对象时都会被调用，`catalogId` 负责区分当前在操作哪一类。

---

### 4.4 从 validator 打印所有 options：List、ListCell、DefElem

options 进来时是 `text[]` 的 `Datum`，需要先还原成链表再遍历。完整代码：

```c
Datum
test_fdw_validator(PG_FUNCTION_ARGS)
{
    List     *options_list = untransformRelOptions(PG_GETARG_DATUM(0));
    Oid       catalog = PG_GETARG_OID(1);
    ListCell *cell;

    ereport(NOTICE, (errmsg("test_fdw: validator called, catalog=%u", catalog)));

    foreach(cell, options_list)
    {
        DefElem *def = (DefElem *) lfirst(cell);
        char    *val = defGetString(def);
        ereport(NOTICE, (errmsg("test_fdw: option key=%s, value=%s",
                                def->defname, val ? val : "<null>")));
    }

    PG_RETURN_VOID();
}
```

下面逐个说明涉及的类型和函数，都附带实际源码定义：

**`untransformRelOptions`**（声明：`src/include/access/reloptions.h`）

```c
extern List *untransformRelOptions(Datum options);
```

将 `text[]` 格式的 `Datum` 还原成 `List *`，每个元素是一个 `DefElem *`，对应一个 `key=value` 的 option 条目。

**List** / **ListCell**（`src/include/nodes/pg_list.h`）

PG 内部通用链表。`List` 持有长度和元素数组，`ListCell` 是联合体，能存指针、int 或 Oid：

```c
typedef union ListCell
{
    void *ptr_value;
    int   int_value;
    Oid   oid_value;
} ListCell;

typedef struct List
{
    NodeTag   type;
    int       length;
    ListCell *elements;
} List;
```

**foreach** / **lfirst**（`src/include/nodes/pg_list.h`）

`foreach` 是 PG 自定义宏，不是 C 标准语法，展开后是一个遍历 `List` 的 for 循环：

```c
foreach(cell, options_list)        /* 遍历 list，cell 依次指向每个 ListCell */
{
    DefElem *def = (DefElem *) lfirst(cell);  /* lfirst 取 cell->ptr_value 并强转为指定类型 */
```

**DefElem**（`src/include/nodes/parsenodes.h`）

每个 option 用一个 `DefElem` 表示，含键名和值：

```c
typedef struct DefElem
{
    NodeTag         type;
    char           *defnamespace; /* NULL if unqualified name */
    char           *defname;
    Node           *arg;          /* typically Integer, Float, String, or TypeName */
    DefElemAction   defaction;    /* unspecified action, or SET/ADD/DROP */
    int             location;     /* token location, or -1 if unknown */
} DefElem;
```

`def->defname` 是 option 键名；`def->arg` 是值节点。提取值时不应直接 `(char *) def->arg`，必须走 `defGetString(def)` 这类解包函数。

为什么 `def->defname` 通常不做空判断，而 `val` 会做防御式判断？

1. `defname` 是一个合法 `DefElem` 的核心字段。很多内核代码直接把 `def->defname` 用于报错文案，默认它必须可用（见 `src/backend/commands/define.c`）：

```c
if (def->arg == NULL)
    ereport(ERROR,
            (errcode(ERRCODE_SYNTAX_ERROR),
             errmsg("%s requires a parameter", def->defname)));
```

2. `defGetString(def)` 的实现对 `arg` 会做严格检查：`arg == NULL` 直接报错，节点类型不支持也报错（见 `src/backend/commands/define.c`）：

```c
char *
defGetString(DefElem *def)
{
    if (def->arg == NULL)
        ereport(ERROR, ... errmsg("%s requires a parameter", def->defname));

    switch (nodeTag(def->arg))
    {
        case T_Integer: return psprintf("%ld", (long) intVal(def->arg));
        case T_Float:   return castNode(Float, def->arg)->fval;
        case T_Boolean: return boolVal(def->arg) ? "true" : "false";
        case T_String:  return strVal(def->arg);
        ...
        default:
            elog(ERROR, "unrecognized node type: %d", (int) nodeTag(def->arg));
    }
    return NULL; /* keep compiler quiet */
}
```

3. `defGetString(def)` 正常路径一般不会返回 `NULL`。代码里写 `val ? val : "<null>"` 更多是防御式写法，让调试输出更稳健。

**`defGetString`**（声明：`src/include/commands/defrem.h`）

```c
extern char *defGetString(DefElem *def);
```

将 `DefElem->arg` 中的值转换为 `char *`。支持的节点类型见 `define.c` 中 `defGetString` 的 `switch(nodeTag(def->arg))`；不支持的类型会直接报错，不会静默返回。

从 `DefElem` 提取内容的实践建议：

- 键名：直接用 `def->defname`，但若编写的是防御型代码（特别是自定义 parser/构造节点场景），可加一次 `if (def == NULL || def->defname == NULL) ereport(ERROR, ...)`。
- 值：优先用 `defGetString/defGetBoolean/defGetInt32/...` 这类 `defGet*`；不要直接解 `def->arg`。
- 打印：调试日志可保留 `val ? val : "<null>"`，但要知道这通常不是“业务上常见分支”。

---

## 5. 基础回调函数

这一节回到最小骨架，逐个说明 7 个基础回调：**角色 → 签名 → 调用点 → 参数要点（含文档/源码依据）→ `test_fdw` 最小实现 → 小结**。与业务 SQL、规划器类型的对照见 **§1.1**。

这里的正确说法是 **FDW 回调函数（callback）**，不是全局 hook。  
它们由 `handler` 返回的 `FdwRoutine` 结构体中的函数指针表承载，规划器和执行器在固定阶段按需调用。

本文 `test_fdw` 的目标是实现一个**最小可跑通骨架**：

- 规划阶段能正常返回一个最简单的 `ForeignScan Path/Plan`
- 执行阶段能进入 FDW 回调链路
- validator 能打印 options
- 扫描阶段先返回空结果，便于调试调用顺序

每个回调的说明分两层：

1. PostgreSQL 设计意图（文档/内核调用点）
2. 本文最小骨架实现到什么程度

---

### 5.1 `testGetForeignRelSize`

**角色**：规划**早期**调用，FDW 根据外表与约束修正**行数/宽度等估计**，供后续选路与成本使用。

函数签名（`fdwapi.h`）：

```c
typedef void (*GetForeignRelSize_function) (PlannerInfo *root,
                                            RelOptInfo *baserel,
                                            Oid foreigntableid);
```

调用点（`src/backend/optimizer/path/allpaths.c`）：

```c
static void
set_foreign_size(PlannerInfo *root, RelOptInfo *rel, RangeTblEntry *rte)
{
    set_foreign_size_estimates(root, rel);
    rel->fdwroutine->GetForeignRelSize(root, rel, rte->relid);
    rel->rows = clamp_row_est(rel->rows);
}
```

参数要点：

- `root`：本次查询的规划器全局状态（`PlannerInfo *`）。官方文档：`GetForeignRelSize` 在扫描外表的规划**开头**调用。
- `baserel`：当前被规划的关系对应的 `RelOptInfo`；最小场景下即 `FROM` 中那张外表。应更新如 `baserel->rows` 等估计。
- `foreigntableid`：外表在 `pg_class` 中的 OID，便于 `GetForeignTable` 等读 catalog；文档说明也可从规划结构推出，显式传入为省事。

官方文档（`fdw-callbacks`）：应更新 `baserel->rows` 为考虑过滤后的期望行数；可更新 `width`、`tuples` 等。

`test_fdw` 最小实现：

```c
static void
testGetForeignRelSize(PlannerInfo *root, RelOptInfo *baserel, Oid foreigntableid)
{
    ereport(NOTICE,
            (errmsg("test_fdw: GetForeignRelSize foreigntableid=%u", foreigntableid)));
    baserel->rows = 1;
}
```

只写 `rows = 1` 是为让规划继续；真实 FDW 会读 options、远端统计或 `EXPLAIN` 等。依赖 `foreign/fdwapi.h`（签名与类型）。教学骨架刻意不连远端，只把 `baserel->rows` 设成常数即可把规划链跑通。部分成熟的 FDW 在 `GetForeignRelSize` 内就会建立连接，做的也不限于「估行」一条线。

---

### 5.2 `testGetForeignPaths`

**角色**：为当前关系生成至少一条 **`ForeignPath`** 并 `add_path` 挂到 `baserel->pathlist`，否则优化器无路径可选。

函数签名（`fdwapi.h`）：

```c
typedef void (*GetForeignPaths_function) (PlannerInfo *root,
                                          RelOptInfo *baserel,
                                          Oid foreigntableid);
```

调用点（`src/backend/optimizer/path/allpaths.c`）：

```c
static void
set_foreign_pathlist(PlannerInfo *root, RelOptInfo *rel, RangeTblEntry *rte)
{
    rel->fdwroutine->GetForeignPaths(root, rel, rte->relid);
}
```

参数要点：`root` / `baserel` / `foreigntableid` 与 `GetForeignRelSize` 一致；`GetForeignPaths` 在 `GetForeignRelSize` **之后**调用（文档：parameters are the same as for `GetForeignRelSize`, which has already been called）。

`test_fdw` 最小实现：

```c
static void
testGetForeignPaths(PlannerInfo *root, RelOptInfo *baserel, Oid foreigntableid)
{
    ereport(NOTICE,
            (errmsg("test_fdw: GetForeignPaths foreigntableid=%u", foreigntableid)));
    add_path(baserel, (Path *) create_foreignscan_path(root, baserel,
                                                        NULL,			/* default pathtarget */
                                                        baserel->rows,
                                                        0,				/* disabled_nodes */
                                                        0,				/* startup_cost */
                                                        0,				/* total_cost */
                                                        NIL,			/* pathkeys */
                                                        NULL,			/* required_outer */
                                                        NULL,			/* fdw_outerpath */
                                                        NIL,			/* fdw_restrictinfo */
                                                        NIL));			/* fdw_private */
}
```

`create_foreignscan_path` 原型见 `src/include/optimizer/pathnode.h`（**PostgreSQL 18 / current**，查询日期 2026-07-17）：

```c
ForeignPath *create_foreignscan_path(PlannerInfo *root, RelOptInfo *rel,
                                     PathTarget *target,
                                     double rows, int disabled_nodes,
                                     Cost startup_cost, Cost total_cost,
                                     List *pathkeys,
                                     Relids required_outer,
                                     Path *fdw_outerpath,
                                     List *fdw_restrictinfo,
                                     List *fdw_private);
```

版本差异（易踩坑）：PG 16 无 `disabled_nodes`、无 `fdw_restrictinfo`；PG 17 有 `fdw_restrictinfo`、无 `disabled_nodes`。对照本地头文件再改调用实参。

**要点**：`NIL` 只适用于 **`List *`**（`pathkeys`、`fdw_restrictinfo`、`fdw_private`）。**`Path *` / `Relids`** 的空值用 **`NULL`**，不要写 `NIL`。

依赖 `optimizer/pathnode.h`（`add_path`、`create_foreignscan_path`）。真实 FDW 会算成本、多 path、`fdw_private` 等。

---

### 5.3 `testGetForeignPlan`

`GetForeignPlan` 把选中的 `ForeignPath` 落成可执行的 `ForeignScan`，并决定 quals 的远端/本地分工，把执行期私有信息写入节点字段。

此处 **quals** 主要指交给该 `ForeignScan` 的 restriction clauses，即参数 `scan_clauses`。

官方文档（`fdw-callbacks`）：

```text
... the target list to be emitted by the plan node, the restriction clauses
to be enforced by the plan node ...
```

调用点（`src/backend/optimizer/plan/createplan.c`）：

```c
/*
 * Let the FDW perform its processing on the restriction clauses and
 * generate the plan node.  Note that the FDW might remove restriction
 * clauses that it intends to execute remotely, or even add more (if it
 * has selected some join clauses for remote use but also wants them
 * rechecked locally).
 */
scan_plan = rel->fdwroutine->GetForeignPlan(root, rel, rel_oid,
                                            best_path,
                                            tlist, scan_clauses,
                                            outer_plan);
```

函数签名（`fdwapi.h`）：

```c
typedef ForeignScan *(*GetForeignPlan_function) (PlannerInfo *root,
                                                 RelOptInfo *baserel,
                                                 Oid foreigntableid,
                                                 ForeignPath *best_path,
                                                 List *tlist,
                                                 List *scan_clauses,
                                                 Plan *outer_plan);
```

调用点（`src/backend/optimizer/plan/createplan.c`）：

```c
scan_plan = rel->fdwroutine->GetForeignPlan(root, rel, rel_oid,
                                            best_path,
                                            tlist, scan_clauses,
                                            outer_plan);
```

参数要点：

- `root`：规划器全局上下文（`PlannerInfo *`）。
- `baserel`：当前 path 所属关系的 `RelOptInfo`（不只可表示 base rel，也可表示 join/upper rel 的关系节点）。
- `foreigntableid`：基表场景下是外表 `pg_class` OID；join path 场景为 `InvalidOid`。
- `best_path`：优化器最终选中的 `ForeignPath`。文档明确它是 selected `ForeignPath`，且可能由 `GetForeignPaths / GetForeignJoinPaths / GetForeignUpperPaths` 产生。
- `tlist`：该 plan 节点的目标列列表（target list）。文档写明是“the target list to be emitted by the plan node”。
- `scan_clauses`：当前关系由该 `ForeignScan` 负责约束的 restriction clauses（quals）。
- `outer_plan`：`ForeignScan` 的外层子计划，主要用于 `RecheckForeignScan` 重检场景。

`test_fdw` 的最小实现：

```c
static ForeignScan *
testGetForeignPlan(PlannerInfo *root, RelOptInfo *baserel, Oid foreigntableid,
                   ForeignPath *best_path, List *tlist, List *scan_clauses, Plan *outer_plan)
{
    ereport(NOTICE,
            (errmsg("test_fdw: GetForeignPlan foreigntableid=%u", foreigntableid)));
    scan_clauses = extract_actual_clauses(scan_clauses, false);
    return make_foreignscan(tlist, scan_clauses, baserel->relid, NIL, NIL, NIL, NIL, outer_plan);
}
```

这个最小实现做了两件事：

- `extract_actual_clauses(...)`：把 `List<RestrictInfo>` 拆成裸表达式（并过滤掉不需要项）。
- `make_foreignscan(...)`：构造 `ForeignScan`，并填入 `targetlist/qual/outer_plan` 及各 FDW 字段（`fdw_exprs`、`fdw_private`、`fdw_scan_tlist`、`fdw_recheck_quals`）。

相关源码：

```c
/* src/backend/optimizer/util/restrictinfo.c */
/* src/include/nodes/plannodes.h */ 
/*src/backend/optimizer/plan/createplan.c */
List *
extract_actual_clauses(List *restrictinfo_list,
                       bool pseudoconstant)
{
    ...
    if (rinfo->pseudoconstant == pseudoconstant &&
        !rinfo_is_constant_true(rinfo))
        result = lappend(result, rinfo->clause);
}

/* Plan */
List *targetlist;   /* target list to be computed at this node */
List *qual;         /* implicitly-ANDed qual conditions */
struct Plan *lefttree; /* input plan tree(s) */

/* ForeignScan */
List *fdw_exprs;        /* expressions that FDW may evaluate */
List *fdw_private;      /* private data for FDW */
List *fdw_scan_tlist;   /* optional tlist describing scan tuple */
List *fdw_recheck_quals;/* original quals not in scan.plan.qual */

/* make_foreignscan() 填充点 */
ForeignScan *
make_foreignscan(List *qptlist, List *qpqual, Index scanrelid,
                 List *fdw_exprs, List *fdw_private,
                 List *fdw_scan_tlist, List *fdw_recheck_quals,
                 Plan *outer_plan)
{
    ...
    plan->targetlist = qptlist;
    plan->qual = qpqual;
    plan->lefttree = outer_plan;
    node->fdw_exprs = fdw_exprs;
    node->fdw_private = fdw_private;
    node->fdw_scan_tlist = fdw_scan_tlist;
    node->fdw_recheck_quals = fdw_recheck_quals;
}
```

`GetForeignPlan` 在规划末期把已选 `ForeignPath` 落成 `ForeignScan`，明确 quals 的远端/本地分工，封装执行期私有信息（`fdw_exprs`、`fdw_private`、`fdw_scan_tlist`、`fdw_recheck_quals`、`outer_plan`）。

#### 常见疑问：`InvalidOid`、多类 Path 回调、`scan_clauses`、`outer_plan`

**1) 为什么 join relation 的 `ForeignPath` 会让 `foreigntableid = InvalidOid`？**

因为该 path 代表的是 join/upper 结果，不是单一外表，没有唯一 `pg_class` OID 可传，所以传 `InvalidOid`。

官方文档（`fdw-callbacks`）原文：

```text
(If the path is for a join rather than a base relation, foreigntableid is InvalidOid.)
```

**2) `best_path` 可能来自三类 Path 回调，是否表示会反复回调？**

会多次回调，但按阶段划分，并非无意义循环：

- 基表阶段：`GetForeignRelSize` -> `GetForeignPaths`
- join 阶段：`GetForeignJoinPaths`（同一 joinrel 可能多次）
- upper 阶段：`GetForeignUpperPaths`
- 最后选出 `best_path` 后，再调用 `GetForeignPlan`

**3) `scan_clauses` 在 SQL 里通常是什么？**

它是“当前关系”的 `ForeignScan` 要负责的条件集合。拥有者可以是 base rel、join rel、upper rel。

示例 A（单表）：

```sql
SELECT * FROM ft WHERE a > 10 AND b = 'x';
```

常见是：`scan_clauses = [a > 10, b = 'x']`（传给 FDW 时通常还包着 `RestrictInfo`）。

示例 B（带 join）：

```sql
SELECT *
FROM ft1
JOIN ft2 ON ft1.id = ft2.id
WHERE ft1.a > 10;
```

- 不做 join pushdown：`ft1` 常见 `[ft1.a > 10]`，`ft2` 常见 `NIL`，`ft1.id = ft2.id` 在本地 join 节点。
- 做 join pushdown：join rel 的 `scan_clauses` 可能包含 `ft1.id = ft2.id`、`ft1.a > 10`（或其子集），由 FDW 决定远端/本地分工。

**4) `outer_plan` 什么时候有值？**

普通单表 foreign scan 通常为 `NULL`。  
只有存在 `best_path->fdw_outerpath` 时，core 才会先构造 `outer_plan` 再传给 FDW，常见于需要重检等较复杂场景。

```c
Plan *outer_plan = NULL;
...
if (best_path->fdw_outerpath)
    outer_plan = create_plan_recurse(root, best_path->fdw_outerpath,
                                     CP_EXACT_TLIST);
```

---

### 5.4 `testBeginForeignScan`

`BeginForeignScan` 是执行器启动阶段的入口。它负责做扫描前初始化，但不应真正开始取数（真正取数在 `IterateForeignScan`）。

函数签名（`fdwapi.h`）：

```c
typedef void (*BeginForeignScan_function) (ForeignScanState *node,
                                           int eflags);
```

调用点（`src/backend/executor/nodeForeignscan.c`）：

```c
fdwroutine->BeginForeignScan(scanstate, eflags);
```

参数要点：

- `node`：当前执行期 `ForeignScanState`，用于读 plan 信息与保存 FDW 运行态（如 `fdw_state`）。
- `eflags`：执行模式标志位。

`test_fdw` 最小实现：

```c
static void
testBeginForeignScan(ForeignScanState *node, int eflags)
{
    ereport(NOTICE, (errmsg("test_fdw: BeginForeignScan eflags=%d", eflags)));
}
```

该实现刻意不做初始化，只验证回调可达。真实 FDW 通常在此建立连接、初始化游标和私有状态。若 `(eflags & EXEC_FLAG_EXPLAIN_ONLY)`，官方文档要求不做对外可见动作，只保证节点状态足以支撑 `ExplainForeignScan` / `EndForeignScan`。

---

### 5.5 `testIterateForeignScan`

`IterateForeignScan` 逐行取数：每调用一次产出一行。官方文档写明无更多行时返回 `NULL`；实现上也常见 `ExecClearTuple(slot)` 后返回该 slot。执行器经 `TupIsNull` 两者都当作 EOF。

函数签名（`fdwapi.h`）：

```c
typedef TupleTableSlot *(*IterateForeignScan_function) (ForeignScanState *node);
```

调用点（`src/backend/executor/nodeForeignscan.c`）：

```c
slot = node->fdwroutine->IterateForeignScan(node);
```

参数要点：

- `node`
  - 含义：当前扫描状态，可从中拿 `ScanTupleSlot`、FDW 私有状态、relation 信息。
  - 官方文档证据：要求结果写入 node 的 `ScanTupleSlot`，并在无更多行时返回空。
  - 源码证据：执行器把 `node` 传入回调，并消费其返回的 `slot`。

`test_fdw` 最小实现：

```c
static TupleTableSlot *
testIterateForeignScan(ForeignScanState *node)
{
    ereport(NOTICE, (errmsg("test_fdw: IterateForeignScan")));
    return ExecClearTuple(node->ss.ss_ScanTupleSlot);
}
```

这段代码的语义是：每次都清空并返回扫描 slot，相当于“立即 EOF、无数据行”。  
因此当前 `test_fdw` 能跑通执行链路，但 `SELECT` 不会返回实际数据。带 `NOTICE` 时，本回调仍可能被连续调用多次，属正常。

依赖头文件：

- `executor/tuptable.h`（`ExecClearTuple` 声明）

---

### 5.6 `testReScanForeignScan`

**角色**：同一计划节点需要**从头再扫一遍**时调用（如嵌套循环内层重扫、参数变化）。应重置游标/远端查询状态，使下一次 `IterateForeignScan` 从第一行开始。

函数签名（`fdwapi.h`）：

```c
typedef void (*ReScanForeignScan_function) (ForeignScanState *node);
```

调用点（`src/backend/executor/nodeForeignscan.c`）：

```c
node->fdwroutine->ReScanForeignScan(node);
```

参数要点：

- `node`：当前 `ForeignScanState`；可访问 `node->fdw_state`、计划节点等。

官方文档（`fdw-callbacks`）：`Restart the scan from the beginning.` 并说明参数可能已变，新扫描返回的行集未必与上次相同。

`test_fdw` 最小实现：

```c
static void
testReScanForeignScan(ForeignScanState *node)
{
    ereport(NOTICE, (errmsg("test_fdw: ReScanForeignScan")));
}
```

无远端状态时可为空；真实 FDW 需重绑/重执行远端语句。依赖：`foreign/fdwapi.h`。

---

### 5.7 `testEndForeignScan`

**角色**：扫描**结束**时释放资源（文件、连接、结果集等）；`palloc` 内存通常随查询结束回收，但**外部资源**应在此释放。

函数签名（`fdwapi.h`）：

```c
typedef void (*EndForeignScan_function) (ForeignScanState *node);
```

调用点（`src/backend/executor/nodeForeignscan.c`）：

```c
node->fdwroutine->EndForeignScan(node);
```

参数要点：

- `node`：当前 `ForeignScanState`；`fdw_state` 在 `BeginForeignScan` 中分配时，应在此清理。

官方文档（`fdw-callbacks`）：`End the scan and release resources.` 并说明 `palloc` 不必刻意释放，但打开的文件、远端连接等需要清理。

`test_fdw` 最小实现：

```c
static void
testEndForeignScan(ForeignScanState *node)
{
    ereport(NOTICE, (errmsg("test_fdw: EndForeignScan")));
}
```

骨架里 `EndForeignScan` 可为空；真实 FDW 在此关闭 statement、远端连接等。依赖：`foreign/fdwapi.h`。

---

### 5.8 这 7 个回调与当前最小骨架的关系

按执行顺序：

1. `GetForeignRelSize`：给规划器一个大小估计
2. `GetForeignPaths`：提供可选扫描路径
3. `GetForeignPlan`：把选中的路径变成 `ForeignScan` 计划节点
4. `BeginForeignScan`：执行前初始化
5. `IterateForeignScan`：逐行产出结果
6. `ReScanForeignScan`：需要重扫时重置状态
7. `EndForeignScan`：结束时释放资源

真实 FDW 的「最小 SELECT 扫描链路」主要就是这 7 个。本文 `test_fdw` 只验证骨架：规划阶段给最简 rows/path/plan；执行阶段证明回调能进来；用空 slot 返回、不做真实远端访问；调试优先 `NOTICE`，而不是一上来实现完整扫描。

骨架依赖的头文件：`foreign/fdwapi.h`、`optimizer/pathnode.h`、`optimizer/planmain.h`、`optimizer/restrictinfo.h`、`executor/tuptable.h`、`access/reloptions.h`、`commands/defrem.h`。

适用边界：本文只验证回调可达性与空结果返回。接入 catalog/options、远端连接与真实扫描需另行扩展，通常仍从同一 7 回调入手。

## 6. 官方参考

下列链接基于 **PostgreSQL 当前手册**（`/docs/current/`）。若本地有源码树，可同时对照 `doc/src/sgml/` 下同名 `.sgml`。

### C 语言与 SQL 注册

- C 语言函数（`Datum`、`PG_FUNCTION_ARGS`、`PG_RETURN_`* 等）：[xfunc-c](https://www.postgresql.org/docs/current/xfunc-c.html)
- `CREATE FUNCTION`（`LANGUAGE C`、符号与库路径）：[sql-createfunction](https://www.postgresql.org/docs/current/sql-createfunction.html)

### FDW 总览与回调

- 编写 FDW 章节总览：[fdwhandler](https://www.postgresql.org/docs/current/fdwhandler.html)
- **各回调签名与语义（与 `fdwapi.h` 一一对应）**：[fdw-callbacks](https://www.postgresql.org/docs/current/fdw-callbacks.html)
- 规划阶段补充说明（与 §58.4 交叉引用）：[fdw-planning](https://www.postgresql.org/docs/current/fdw-planning.html)
- 辅助函数（如 `make_foreignscan`、路径构造相关 helper 的文档入口）：[fdw-helpers](https://www.postgresql.org/docs/current/fdw-helpers.html)

### SQL 侧对象（与业务 `FROM`、连接信息对应）

- `CREATE FOREIGN DATA WRAPPER`：[sql-createforeigndatawrapper](https://www.postgresql.org/docs/current/sql-createforeigndatawrapper.html)
- `CREATE SERVER` / `ALTER SERVER`：[sql-createserver](https://www.postgresql.org/docs/current/sql-createserver.html)
- `CREATE USER MAPPING`：[sql-createusermapping](https://www.postgresql.org/docs/current/sql-createusermapping.html)
- `CREATE FOREIGN TABLE`：[sql-createforeigntable](https://www.postgresql.org/docs/current/sql-createforeigntable.html)

### 扩展与模块

- `CREATE EXTENSION`（安装扩展、脚本与版本）：[sql-createextension](https://www.postgresql.org/docs/current/sql-createextension.html)

### 本地源码树（与手册对应）

- `doc/src/sgml/fdwhandler.sgml`
- `src/include/foreign/fdwapi.h`

---

