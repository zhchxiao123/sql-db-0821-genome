---
id: cli-entrypoints
confidence: high
---

两个命令行入口:`cmd/sqldb`(引擎 CLI:stdin SQL → 结果)与 `cmd/sqllogictest`(.test
跑分器),外加 `Makefile` 把它们串成 build/test/suite 目标。

- **承重不变量**:
  - `cmd/sqldb` 输出:每行结果用 `|` 连接、NULL 渲染为字面 `"NULL"`、非 SELECT 语句不
    打印任何行;第一个错误 → stderr 报错 + `exit 1`(`cmd/sqldb/main.go:47-58`)。
  - `splitStatements` 必须按 `;` 切分且尊重单引号串(含 `''` 转义),否则串内的分号会
    被误切(`cmd/sqldb/main.go:64`)。
  - `-db` 为空 → 内存库(进程退出即失);非空 → `engine.Open` 持久库
    (`cmd/sqldb/main.go:34`)。
  - `cmd/sqllogictest` 不带文件参数时,默认集合是 `suite/select*.test` +
    `suite/random/expr/*.test`(按字符串路径 `filepath.Glob`,`cmd/sqllogictest/main.go:25`);
    `--strict` 把 waived 记成 fail;总失败 > 0 → `exit 1`,runner 自身/解析错误 → `exit 2`
    (`cmd/sqllogictest/main.go:37,62`)。
  - `Makefile`:`build`/`test`/`suite`/`sqllogictest`/`check`(= build test sqllogictest)
    (`Makefile:9-28`);`sqllogictest` 目标默认跑 `$(SUITE)=suite/select*.test`。
- **异常与降级路径**:stdin 读失败、Open 失败都在 stderr 报错后 exit 1;套件文件不存在
  (没 `make suite`)时 exit 2 并提示先抓套件。
- **测试证据**:runner 的判定逻辑由 `runner_test.go` 整文件覆盖(见 sqllogictest-runner
  卡);CLI 本身无 Go 测试(`cmd/` 输出 `[no test files]`),其行为由 README 文档与
  `testdata/*.test` 的人工跑分维持。
- **坑点**:套件未抽取时默认 glob 找不到文件会直接 exit 2——这是 CI 第一步必须先
  `make suite` 的原因。README 的 \"Current suite status\" 表是一次跑分的快照,不是
  exit/waive 数字的来源。