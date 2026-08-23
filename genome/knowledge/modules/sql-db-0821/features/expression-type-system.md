---
id: expression-type-system
confidence: high
---

SQLite 兼容的值/类型/表达式系统:`internal/engine/value.go` 定义 5 种 storage
class(Null/Int/Float/Text/Blob)与 affinity 规则,`internal/engine/expr.go` 定义
表达式树及求值。整个子集以 sqlite3 3.51.0 为参考实现逐值对拍。

- **承重不变量**:
  - 整数溢出必须升为 REAL,禁止回绕;除零→NULL;NaN 结果→NULL(`intMath`/`arith`,
    `internal/engine/value.go:516`, `internal/engine/value.go:469`)。这是
    `-9223372036854775808` 相关用例的全部正确性来源。
  - 布尔三值逻辑:`AND` 里 0 压过 NULL(0 AND NULL = 0),`OR` 里 1 压过 NULL
    (1 OR NULL = 1),`NOT NULL = NULL`(`andValues`, `internal/engine/value.go:430`;
    `orValues`, `internal/engine/value.go:442`)。
  - 比较按 storage-class 序 NULL < numbers < TEXT < BLOB,且比较前先施加 comparison
    affinity(数值列 vs 文本字面量时文本转数字)(`compareValues`,
    `internal/engine/value.go:564`;`applyComparisonAffinity`, `internal/engine/expr.go:927`)。
  - `CAST` 是强制转换,允许丢信息(`castValue`, `internal/engine/value.go:271`);
    `-9223372036854775808` 是合法的最小 int64,裸 `9223372036854775808` 是 REAL 大数
    (`internal/engine/expr.go:23` bigInt 字段、`internal/engine/expr.go:1208`
    unary-minus 特判)。
  - `formatFloatSQLite` 是浮点渲染契约:15 位有效数字、恒有小数点、两位带符号指数、
    NaN 渲染为空串、Inf 渲染为 `Inf`/`-Inf`(`internal/engine/value.go:124`)。
- **异常与降级路径**:表达式求值失败向上冒 `SQLError`(如 `ESCAPE` 非单字符);
  NULL 传播按 SQLite 规则——算术/比较遇 NULL 得 NULL(IS/IS NOT 除外)。render 层是
  纯函数,不吞错误。
- **测试证据**:`TestConstantExpressions`(engine_test.go:283,约 30 条与 sqlite3 逐值
  对拍)、`TestOverflowPromotesToReal`(engine_test.go:340)、`TestAffinityConversion`
  (engine_test.go:359,INSERT affinity`48.00`→`48` 等)、`TestFloatRendering`
  (engine_test.go:249,含 1.0/3.0 与 2 的幂边界)、`TestComparisonAffinity`
  (engine_test.go:413)。
- **坑点**:sqllogictest `RenderSLT` 里列类型串驱动渲染——`'R'` 浮点截成 `%.3f`,
  `'T'` 用 `formatFloatSQLite`;给浮点结果写错的类型串,对拍必然失败(这正是
  random/expr 套件里部分 record 的 waive 来源)。