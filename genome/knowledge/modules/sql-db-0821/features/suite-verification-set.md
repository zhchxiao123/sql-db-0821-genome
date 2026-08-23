---
id: suite-verification-set
confidence: high
---

被用于验证引擎的固定数据与政策:`suite/`(从上游 pin 下来)、`testdata/`(验收自测)、
`scripts/fetch-suite.sh`(抓取)与 README 记录的 waiver 政策/状态表。

- **承重不变量**:
  - 套件 pin:上游 `gregrahn/sqllogictest` tag `version-3.11.0` / commit
    `0b24fd28f7bbe2598fb87dab53cb17b8ddd77520`;抓取只取 `select1..5.test` 与
    `random/expr/slt_good_*.test`(0..119,120 个文件),其它目录(evidence/subquery/
    index 等)刻意不抓(`scripts/fetch-suite.sh:14-33`)。
  - waiver 政策是引擎可执行子集与上游套件的对齐机制:子集外构造的 record 报 waived
    而非 failed;`--strict` 翻转(`README.md:114`)。
  - `testdata/` 是对验收面的自测:`wrong.test` 故意写错预期值,跑它必须 exit 1(验证失败
    检测真的会响);`expr.test`/`orderby.test`/`groupby.test`/`transaction.test` 等各自
    钉一个子需求面;每条预期值注释"verified against sqlite3 3.51.0"(`README.md:180`)。
  - suite 目录是抓下来的产物,不是手写数据;改套件要改抓取脚本/README pin,不要直接改
    `suite/` 里的 .test 文件。
- **异常与降级路径**:网络取不到 pin 时会抓取失败(脚本 `set -euo pipefail`,
  `scripts/fetch-suite.sh:12`);没抓过套件时默认 glob 为空,runner exit 2 提示先
  `make suite`。
- **测试证据**:无 Go 测试直接覆盖数据文件本身;数据被 `make sqllogictest`(=
  `go run ./cmd/sqllogictest $(SUITE)`)消费,套件结果由 README "Current suite status"
  表记录(该表是快照:TOTAL 528001 records / 461118 passed / 0 failed / 66883 waived)。
- **坑点**:README 状态表是写文档时刻的快照——曾观察到 run gate 出现 66329 waived 的
  实测值,与表上的 66883 不同;改子集/套件会移动这些数字,别把表当常量。当前引擎已能
  通过本模块各验收面,但 JOIN/UNION/子查询仍属未实现——任何验收文件若碰这些,只会
  增加 waive 数,不是回归。