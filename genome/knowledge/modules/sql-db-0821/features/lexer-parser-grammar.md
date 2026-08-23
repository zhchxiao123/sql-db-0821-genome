---
id: lexer-parser-grammar
confidence: high
---

词法与语句语法层:把 SQL 文本切成 token、按语句类型分发,并为 CREATE / INSERT /
UPDATE / DELETE / DROP 建立 AST 侧的动作。它决定了一条语句是"能执行"还是
"该回 typed 错误"。

- **承重不变量**:
  - 子集外构造(JOIN/UNION/INTERSECT/EXCEPT/CASE/EXISTS/INDEX/ALTER/VIEW/PRAGMA/
    VACUUM/ATTACH 等)必须返回 `UnsupportedError`,而**不是**普通语法错——runner 靠
    `errors.As(...UnsupportedError)` 把 record 判成 waive 而非 fail
    (`internal/engine/errors.go:18`,`internal/engine/parser.go::unsupportedKeywords`,
    分发点 `internal/engine/parser.go:155`)。
  - 列级约束 `PRIMARY KEY / NOT NULL / UNIQUE / AUTOINCREMENT` 被解析但**刻意忽略**,
    不留任何残留字段;这是 README 明说的"后续子需求",不是遗漏
    (`internal/engine/parser.go::skipColumnConstraints`)。
  - `tokenize` 必须接受三类 SQLite 字面量:单引号串(内嵌 `''` 转义)、`X'hex'` blob
    (hex 长度必须为偶数,否则 SQL 错)、`==` 视作 `=`
    (`internal/engine/parser.go:92`, `internal/engine/parser.go:172`)。
  - INSERT 的值在落库前按目标列 affinity 转换,转换与解析同函数完成
    (`internal/engine/parser.go::parseInsert`, `internal/engine/parser.go:377`)。
- **异常与降级路径**:`checkTrailing` 把语句尾部残留的 unsupported 关键字报成
  `UnsupportedError`,其余残留报 syntax error(`internal/engine/parser.go:222`);未闭合
  串/未闭合 blob/非 hex 都报 `SQLError`(`internal/engine/parser.go:100`,
  `internal/engine/parser.go:132`)——绝不让解析器 panic。
- **测试证据**:`TestUnsupportedConstructs`(engine_test.go:174)对 13 类子集外语句逐一
  断言 `UnsupportedError`;`TestSyntaxErrors`(engine_test.go:202)断言这些语法错是普通
  `SQLError` 而非 `UnsupportedError`(两类错误不能串台);`TestColumnConstraintsIgnored`
  (engine_test.go:239);`TestBlobLiterals`(engine_test.go:459,含奇数 hex / 非 hex 报错)。
- **坑点**:schema 约束的"解析即忽略"与 `internal/engine/expr.go:1390` 那一行把旧聚合
  实现整段粘进 doc comment 的陈旧注释,都是容易被误读为已实现/将实现的点。