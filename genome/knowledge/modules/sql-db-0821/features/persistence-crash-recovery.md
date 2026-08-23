---
id: persistence-crash-recovery
confidence: high
---

`-db <path>` 模式下的持久化与崩溃恢复:`internal/engine/persist.go` 把每次
COMMIT/隐式提交后的已提交状态全量序列化为一份 JSON 文件,重启时 `engine.Open`
加载回来。目标:已提交事务在 `kill -9` 下不丢。

- **承重不变量**:
  - 持久化链路必须是 temp 文件 → `f.Sync()` → 原子 `os.Rename` → 目录 `Sync()`,且
    任一步失败都要清理 temp(`persist`, `internal/engine/persist.go:135-171`);已提交的
    字节必须先落盘,才能撑起 "committed survives kill -9"。
  - 只有 `flush()`(显式 COMMIT 与 autocommit 成功路径)触发持久化;未提交事务永远不会
    出现在文件里(`internal/engine/conn.go::flush` `:241`,先换 `e.tables` 再 `persist`)。
  - 文件缺失 = 空库;文件存在但不可解析 = 报 "corrupt database file" 错误,**不静默**
    回落成空库(`loadTables`, `internal/engine/persist.go:101-112`)。
  - 序列化输出必须稳定(map 键排序,`marshalTables` `internal/engine/persist.go:78`,
    JSON 自带键排序)。

- **异常与降级路径**:写盘任一步出错都向上回报、清 temp、已提交的内存状态不动——持久化
  失败不会回滚已完成的事务,代价是重启后丢失,调用方看到错误可以自己选择重试。
  文本/blob 以 base64 封存(`toPValue` `internal/engine/persist.go:25`),任意字节可往返。
- **测试证据**:`TestPersistCrashRecovery`(engine_test.go:568)——会话一 COMMIT+autocommit
  后 ROLLBACK 一条,会话二(模拟重启)`Open(path)` 只见已提交的 2 行。
- **坑点**:这份文件是本模块自有的 JSON 转储,不是 sqlite 或任何标准格式;别手工编辑它,
  别指望外部工具可读。历史坑(见项目记忆 `persist-constraint-reload-gap`):一旦给列加了
  表达式 DEFAULT/CHECK/约束而 `expr` 渲染又把它们塌缩成 `(expr)`,reload 会重建出残缺
  约束——当前仓库约束解析即忽略所以不触发,但实现约束子需求的第一天就要重查这条链路。