---
id: select-query-pipeline
confidence: high
---

单表 SELECT 执行管线:`parseSelect` 处理 FROM/WHERE/GROUP BY/HAVING/ORDER BY/
LIMIT/OFFSET/DISTINCT 与聚合,并把聚合查询重写为"先按组物化聚合值、再对组行求值"
的 slot 管线。无 FROM 的 SELECT 提供一行常量行。

- **承重不变量**:
  - 聚合查询中每个 `AggExpr` 必须被 `replaceAggregates` 重写成 `SlotExpr`,聚合值按组
    物化进组行尾部;`HAVING` 可以引用**未投影**的聚合(`internal/engine/parser.go:787-860`,
    `internal/engine/expr.go::replaceAggregates`, `internal/engine/expr.go::evalAgg`)。
  - GROUP BY 查询里,select 列表与 HAVING 中任何裸非聚合列若不是 GROUP BY 键列,
    必须报 "not a GROUP BY column"—这是相对 SQLite 的**刻意偏离**(SQLite 返回任意值),
    决策出处注释在 `internal/engine/expr.go:803`(`q6`);例外是聚合参数
    (`internal/engine/expr.go::checkBareCols`)。
  - NULL 语义三处一致:GROUP BY 键 NULL==NULL 并组、SELECT DISTINCT 行 NULL 行合并、
    ORDER BY 里 NULL ASC 最先 / DESC 最后(`internal/engine/expr.go::groupKeyString`,
    `internal/engine/expr.go::dedupRows`, `internal/engine/expr.go::orderCompare`)。
  - LIMIT 边界与 SQLite 一致:LIMIT 0 → 无行;OFFSET 越界 → 无行;负 LIMIT → 不限;
    裸 OFFSET(无 LIMIT)是语法错;`LIMIT m,n` 是 `LIMIT m OFFSET n`
    (`internal/engine/parser.go:678`)。
  - `condTrue` 只认"Int 且非零"为 true,NULL 与 0 都滤除(`internal/engine/expr.go:890`)。
- **异常与降级路径**:HAVING 出现在非聚合查询是语法错(`internal/engine/parser.go:764`);
  `sortRowsByKeys` 中表达式键评估失败时以 `false` 返回(吞错、排序退化),这是现存
  行为,改动时要意识到(见坑点)。SELECT * 必须有 FROM,否则语法错
  (`internal/engine/parser.go:723`)。
- **测试证据**:`testdata/orderby.test`(NULL 位置/序数键/表达式键)、
  `testdata/groupby.test`(多列分组、NULL 并组)、`testdata/limit-offset.test`、
  `testdata/distinct.test`、`testdata/aggregate*.test`;engine_test.go 的
  `TestSelectExpressions`(:430)、`TestCountStar`(:158)、`TestWhereComparisons`(:93)。
- **坑点**:JOIN/UNION/子查询在当前 HEAD(`eb1c889`)里一律 `UnsupportedError`
  (`internal/engine/parser.go:584` 逗号 FROM、`:587` 子查询 FROM);存在"多表查询已落地"
  的过时认知,别据此期望这条管线支持多表。ORDER BY 表达式键评估错误被静默吞掉的路径,
  是新套件 record 失败时首个要排查的地方。