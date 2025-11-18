# Repository Guidelines

## 项目结构与模块划分
TinyKV 复刻 TiKV 的分层设计：`kv` 负责存储引擎与事务管线，`raft` 提供共识算法及配套测试，`scheduler` 实现 TinyScheduler 的集群管理与调度策略，`proto` 存放所有 gRPC 协议及生成的 Go 代码，`log` 提供日志工具，`doc` 包含实验说明、图示，`scripts` 维护辅助脚本。构建产物写入 `bin/`，请勿提交二进制或临时数据。

## 构建、测试与开发命令
默认使用 `make` 或 `make kv scheduler` 在 `bin/` 下生成 `tinykv-server` 与 `tinyscheduler-server`。接口变更后运行 `make proto` 重新生成协议代码（需 `protoc` 和 `protoc-gen-go`）。`make dev` 执行构建+测试的完整流程，`make test` 会以 `LOG_LEVEL=fatal` 串行运行 `go test`，模拟评测环境。针对各实验阶段可使用 `make project2a`、`make project3b` 等快捷方式，其内部会根据测试名调用 `go test ./module -run PATTERN` 并清理 `/tmp/*test-raftstore*`。

## 代码风格与命名规范
项目要求 Go 1.13+，所有源文件保持 `GO111MODULE=on`。缩进使用制表符，提交前运行 `gofmt -s`（可通过 `make format` 或 `make ci` 自动校验）。导出类型/函数遵循 Go 的驼峰命名，包内测试、工具可用首字母小写。`.proto` 字段沿用 snake_case，调整后务必重新生成对应的 Go 绑定。

## 测试指南
测试依赖标准库 `testing`，未引入第三方断言。推送前至少执行对应阶段的用例，如 `go test ./kv/test_raftstore -run ^TestBasic2B$` 或直接调用 `make project3c`。新测试命名以 `Test` + 阶段标识（如 `TestSnapshotRecover2C`），方便 CI 精确过滤。长时间运行的套件前后记得清理 `/tmp/*test-raftstore*`，避免残留持久化状态。

## 提交与 Pull Request 规范
Git 历史多为简洁祈使句 + 关联信息的格式，例如 `fix(to issue#455): fix a typo (#456)`。提交信息建议控制在 60 字符以内，并在尾部引用 Issue/实验编号。Pull Request 描述应包含：问题背景、解决方案要点、执行过的命令（如 `make test`、`make project2b`），以及协议或配置的额外说明。涉及文档截图时附上图片，代码改动优先贴日志或命令输出。

## 安全与配置提示
不要提交生成的二进制、`/tmp/*test-raftstore*` 数据或课堂凭证。重新生成协议时将 `$(pwd)/bin` 加入 `PATH` 以保持与 CI 一致。启动服务前先创建数据目录，例如 `mkdir -p data && ./tinykv-server -path=data`。若在共享环境运行多节点，请自行管理 TLS/口令；仓库脚本默认本地回环部署，不会替你加固安全配置。
