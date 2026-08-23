---
id: transaction-concurrency
confidence: high
---

`Engine` 共享存储 + `Conn` 连接的事务与并发:`internal/engine/conn.go` 实现
BEGIN/COMMIT/ROLLBACK、隐式 autocommit(语句级原子性)、快照隔离读与写锁互斥写。

- **承重不变量**:
  - 已提交的 `Table` 值**绝不允许原地变更**:所有写都在事务私有副本上做,COMMIT 时
    整表替换 `e.tables`(`internal/engine/conn.go:9` 文档、`cloneTables`
    `internal/engine/conn.go:79`)。reader 只拷 map 指针、不持锁读表(`snapshot`,
    `internal/engine/conn.go:67`),靠"已提交值不可变"保证无数据竞争。
  - 写锁是互斥的:同一时刻至多一个连接持有;第二个并发写者**立即**得到
    "database is locked",不是阻塞(`acquireWrite`, `internal/engine/conn.go:219`)——这是
    SQLite 无 busy_timeout 语义的刻意选择。
  - autocommit 下语句失败必须丢弃整份工作副本并释放写锁,不留部分修改
    (`execWrite`, `internal/engine/conn.go:176`,失败分支 `internal/engine/conn.go:184`);
    显式事务中失败语句同样不产生部分修改(行级 COW,`internal/engine/parser.go::parseUpdate`
    `:908`)。
  - 无事务 COMMIT 报错、事务内再 BEGIN 报错、无事务 ROLLBACK 是 no-op
    (`execBegin`/`execCommit`/`execRollback`, `internal/engine/conn.go:255-295`)。
  - 读不取写锁:`execRead` 在显式事务内读事务副本、否则读 `snapshot()`
    (`internal/engine/conn.go:165`)。
- **异常与降级路径**:写锁冲突是 `SQLError`(可预期),调用方应重试;`flush` 里持久化
  失败(写盘错误)会回给 COMMIT 的调用方(`internal/engine/conn.go:241`),不会静默。
- **测试证据**:`TestBeginCommitRollback`(engine_test.go:491)、
  `TestCommitWithoutTransactionErrors`(engine_test.go:512)、`TestCompoundRollback`
  (engine_test.go:524)、`TestImplicitAutocommit`(engine_test.go:538)、
  `TestStatementLevelAtomicity`(engine_test.go:550,UPDATE中途失败不留 a=9,COW 断言)、
  `TestConcurrencyReadIsolation`(engine_test.go:596)、`TestConcurrencyWriteLock`
  (engine_test.go:614)、`TestConcurrentReadersNoDataRace`(engine_test.go:633,8 goroutine
  并发读);另有 `testdata/transaction.test`。
- **坑点**:并发写是"抢到即用、抢不到即错",无排队;想在引擎外做排队会改变锁语义。
  该模块没有 WAL/journal,事务原子性只靠 COW 工作副本,别引入在已提交表上直接改的
  新路径。