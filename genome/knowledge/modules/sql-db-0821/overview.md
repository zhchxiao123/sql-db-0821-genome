# sql-db-0821

Go 1.24 写的最小内存 SQL 引擎(`repos/sql-db-0821/`,HEAD `eb1c889`,来自 PR #4
的 -db 持久化合入)。模块带两个 CLI 与一个 sqllogictest 跑分器,目标是"骨架先
跑得动":实现一个小的 SQL 子集(解析→执行→渲染端到端),并用 pinned 套件持续对拍
SQLite 3.51.0。

## 现状(2026-08-22 通读代码,非文档快照)

- **语言/构建**:`go.mod` 声明 Go 1.24.0,零第三方依赖;`make build` / `make test` /
  `make suite` / `make sqllogictest` / `make check`(Makefile)。`go build ./...` 与
  `go test ./...` 于本次扫描验证全绿。
- **子系统**,各配一张功能卡:
  - `internal/engine/parser.go` + `errors.go` —— 词法与语句语法,UnsupportedError
    与 SQLError 的区分(子集外构造给出 typed error,永不 crash)。
  - `internal/engine/value.go` + `expr.go` —— SQLite 兼容的值/类型/表达式系统。
  - `internal/engine/parser.go::parseSelect` + `expr.go` —— SELECT 查询管线。
  - `internal/engine/conn.go` + `engine.go` —— 事务与并发。
  - `internal/engine/persist.go` —— -db 持久化与崩溃恢复。
  - `internal/sqllogictest/` —— .test 解析与判定。
  - `cmd/sqldb`、`cmd/sqllogictest`、`Makefile` —— 入口与命令。
- **已验证行为**:事务(快照隔离、写锁互斥、autocommit、语句级原子性)与持久化的
  单元测试全绿;表达式/类型/查询语义有大量与 sqlite3 3.51.0 逐值对拍的测试。
  README 状态表记录套件跑分 0 failed / 66883 waived(该表是 snapshot,数值会随
  套件抽取与子集扩展移动,别当常量)。

## 关键判定(改动波及面)

- **跨模块契约**:无。Workspace 只挂此一模块;对外暴露的只有 CLI 与磁盘 JSON 文件
  (见 `interfaces.yaml`)。改动 `cmd/` 或 `persist.go` 不会波及别的模块,但会改变
  别的入口可观测的 CLI/文件形态。
- **刻意"不实现"而非 bug**:列级约束(PRIMARY KEY/NOT NULL/UNIQUE)被解析并忽略
  (`parser.go::skipColumnConstraints`);JOIN/UNION/子查询/CASE/EXISTS/函数调用/
  CREATE INDEX/VIEW/ALTER/PRAGMA 一律 `UnsupportedError`。任何"启用这些"的改动都
  是新增子需求,不是修 bug。
- **对拍可信度**:整个表达式/查询语义子集以 sqlite3 3.51.0 为参考实现;改 `value.go`
  的数值/affinity/渲染规则必须保持 suite `random/expr/*.test` 0 failed。

## 坑点(来自代码与历史)

- `internal/engine/expr.go:1390` 有一整行(~4KB)陈旧注释:旧版顺序聚合实现被粘在
  `resolveColumnAffinity` 的 doc comment 里,行内用字面 `\n` 分隔。能编译,但属噪音。
- 内存里有 `sql-db-0821` 模块更早的"空骨架"认知(2026-08-21),已被本次代码通读取代;
  也有"多表查询(JION/UNION)已落地"的认知,但当前 HEAD 里 JOIN/UNION/子查询全是
  UnsupportedError——那条认知对应的是另一个未合入本 repo 的分支。