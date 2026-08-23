---
id: sqllogictest-runner
confidence: high
---

`.test` 文件解析(`internal/sqllogictest/parse.go`)与逐 record 判定
(`internal/sqllogictest/runner.go`):每个文件跑在**全新**引擎上,record 按 full 或
hash 模式对比输出,产出 pass/fail/waive 三态统计。

- **承重不变量**:
  - **每个文件新建一个引擎**(`RunFile`, `internal/sqllogictest/runner.go:41`);同一文件内
    record 共享会话(所以文件内 BEGIN 与后续 COMMIT 能连上),跨文件不共享——这是参考
    跑分器的语义。
  - 判定三态里 waive **只能**源自 `UnsupportedError`(`isUnsupported`,
    `internal/sqllogictest/runner.go:198`);非 strict 时 waive 不计失败,strict 时计为
    失败(`RunFile`, `internal/sqllogictest/runner.go:57`)。
  - `query` 的 full 模式:先比结果长度、再逐值比(`internal/sqllogictest/runner.go:124`);
    hash 模式计算 md5(每个值后加换行)(`computeHash`, `internal/sqllogictest/runner.go:178`),
    hash 行解析失败即 fail(`internal/sqllogictest/runner.go:116`)。
  - `rowsort` 的列数 = `TypeStr` 长度、缺省 1(`internal/sqllogictest/runner.go:104-109`);
    `sortRows` 必须先把行拷出来再排,否则排序会在改写源切片时互相污染
    (`internal/sqllogictest/runner.go:155` 注释即回归说明)。
  - `statement error` / `query error` 记录:执行有错即 pass、无错即 fail
    (`runRecord`, `internal/sqllogictest/runner.go:71-100`)。
  - `skipif`/`onlyif` 整条记录被跳过(连 SQL 都不执行)(`internal/sqllogictest/parse.go:112`);
    `hash-threshold` 与 `halt` 属格式噪声,忽略/终止(`internal/sqllogictest/parse.go:108-111`)。
- **异常与降级路径**:未知 statement 期望、未知 record 类型 → ParseFile 报格式化错误;
  一个文件解析失败不影响其它文件(`cmd/sqllogictest` 对各文件独立 exit 2)。引擎返回
  非 UnsupportedError 的错误在 statement/query error 上是预期结果(可能 pass),在
  `statement ok`/普通 query 上是 fail——两层错误(waive vs 真失败)不能互相污染。
- **测试证据**:`runner_test.go` 整文件:TestRunFilePass(:20)、TestRunFileHashMode(:52,
  钉死 hash `6ddb4095...`)、TestRunFileWrongExpected(:76,错值必 fail 且定位行号)、
  TestRunFileStatementError(:100)、TestRunFileQueryError(:119)、
  TestRunFileWaiveUnsupported(:135,strict 翻转)、TestSortRows(:165)/TestSortRowsThreeRows
  (:178,别名回归)、TestComputeHash(:190)、TestExtractHash(:199)、
  TestParseFileSkipConditional(:208)。
- **坑点**:`TestSortRows` 注释明说的"别名腐蚀源切片"回归是这里最容易再犯的坑;凡是改
  `sortRows` 的人必须保住"先拷贝再排"。waive/失败的分流完全依赖 `errors.As` 命中
  `UnsupportedError`,若未来有人给引擎错误类型换包装,这条分流会静默毁掉整个套件判定。