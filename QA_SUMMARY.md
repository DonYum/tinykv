# QA 汇总

## TinyKV、Raft 与客户端
- **有没有比 TinyKV 更简单、适合作为入门阶梯的实现？**  
  可以先练习 MIT 6.824 提供的 Raft/KV skeleton、仅保留 Project1/2 的 TinyKV fork，或社区的 tinyraft、tinykv-raft 等轻量仓库；这些代码量少、功能聚焦，理解后再迁移到完整 TinyKV。
- **我要在本地启动 TinyKV 并连上 TinySQL，应按什么顺序执行命令？**  
  安装 Go1.13+，在仓库根目录运行 `make` 生成 `bin/tinykv-server` 与 `bin/tinyscheduler-server`，分别在不同终端启动；随后运行 `./tinysql-server --store=tikv --path="127.0.0.1:2379"`，最后通过 `mysql -u root -h 127.0.0.1 -P 4000` 进入 SQL 层。
- **部署了多个 TinyKV/PD 服务后，客户端应填哪些地址、如何感知 leader？**  
  客户端配置所有 PD/TinyScheduler 地址（例如 `--path="pd-0:2379,pd-1:2379,pd-2:2379"`），只与 PD 对话获取 region 拓扑和 leader 位置；若 leader 切换，PD 或 TinyKV 会返回 NotLeader，客户端根据提示刷新缓存并重试。
- **PD/TinyScheduler 属于有状态组件吗？如何做高可用？**  
  它们要持久化元数据与 timestamp，通常部署 3、5 等奇数节点自成 Raft 集群：Raft 只需多数派即可选主，3 节点能容忍 1 台故障，4 节点仍只能容忍 1 台却多占资源。节点之间开放 2379/2380，并把所有 PD 地址提供给客户端，剩余节点即可自动续主。
- **TiKV 的 Region 概念是什么、为何重要？**  
  Region 是按字典序切分的连续 key 区间，属性包括 `start_key/end_key/RegionID`，每个 Region 独立运行 Raft 复制并可自动 split/merge，是调度、容灾、路由的最小单元。
- **Follower 默认不回读，是否会浪费副本？**  
  为保证线性一致性，读写默认进入 leader；但可开启 Replica Read，在确认 follower 追上日志（ReadIndex/leader lease）后执行只读，甚至使用 learner 作为专用读副本。

## KVrocks 与数据同步
- **KVrocks 的 Raft 复制里是否提供 learner/observer？**  
  是的，2.0 之后可以将节点设为 learner，它只追日志不投票，适合异地只读或 CDC，需要时可用 `cluster promote` 转为正式 follower。
- **KVrocks 仓库的代码量大概是多少？**  
  统计 master 分支约 18–20 万行，其中 C++ 约 16 万行、C/Shell/Python/Go 合计约 2 万行，可使用 `cloc --exclude-dir=thirdparty,docs,tests .` 自行验证。
- **KVrocks 能把数据直接存到 S3/OSS 等对象存储吗？**  
  不行，RocksDB 依赖低延迟随机写和 WAL，因此必须用本地块存储；通常把数据写入 SSD，再用备份工具把 SST/WAL 上传对象存储实现冷备。
- **WAL 在 KVrocks 场景中的意义？**  
  WAL（Write-Ahead Log）先记录每个写操作，再刷新 LSM 树文件，宕机后可用 WAL 重放到最新状态，保证原子性与持久性。
- **同城双活但机房之间不互联时，如何同步 KVrocks 数据？**  
  只能离线复制：在主机房完成批量写入后导出 SST/WAL 或用 `SCAN` 导出前缀数据，通过对象存储或介质转移到备机房，再离线导入，切换前需冻结主端写入。
- **按前缀导出 KVrocks 数据的方式有哪些？**  
  可用 Redis 协议执行 `kvrocks-cli --scan --pattern prefix:*` 批量导出；如需更快，可用 RocksDB `sst_dump --from=prefix --to=prefix\xFF` 或 `ldb dump --key_prefix` 直接过滤 SST。
- **KVrocks 在 Kubernetes 中是否能用多 Pod 形式部署？**  
  可以但必须 StatefulSet + PVC，确保每个实例有独立数据目录；再通过 Raft 复制或备份机制实现高可用，不能当作无状态服务随意扩缩。
- **如何让多个同步消费者（MySQL 队列）避免重复消费？**  
  在任务表设计 `status/locked_at/worker_id`，使用 `SELECT ... FOR UPDATE SKIP LOCKED` 或 `UPDATE ... WHERE status='pending'` 抢锁，成功者置为 processing 并处理；超时任务再重置状态，处理逻辑需幂等。

## Loki、对象存储与可用性
- **Loki 是否原生支持将日志和索引写到对象存储？**  
  支持，chunk 和 boltdb-shipper 索引都能配置 S3/GCS/OSS/Swift 等后端；只要在 `storage_config` 中指定 bucket、region、凭证即可落地到对象存储。
- **用了对象存储后是否还依赖本地磁盘？**  
  仍需小量本地或 PVC 保存 ingester WAL、临时 boltdb 文件和缓存，但长期日志都在对象存储，单节点磁盘压力很小。
- **如何在生产环境部署高可用 Loki？**  
  为 distributor/ingester/querier/query-frontend/compactor/ruler 分别部署多个副本，ingester 开启 WAL 并在 ring 中设置至少 3 个副本，数据与索引写对象存储，再结合缓存和监控即可撑住故障。
- **对象存储方案是否经过大规模验证？**  
  是 Loki 的默认架构，Grafana Cloud 以及大量企业自建集群都在线性扩展到 PB 级，说明 S3/GCS/OSS 后端已经成熟可行。
- **使用 S3 会带来多大性能损耗？**  
  chunk 上传和历史查询会增加几十到几百毫秒延迟，取决于网络与 S3 延迟；开启 chunk/index cache、合理设置 chunk 大小与并发，可将开销控制在可接受范围。

## Kubernetes 相关
- **Headless Service 的概念与用途是什么？**  
  它把 `spec.clusterIP` 设为 None，不提供虚拟 IP，而是直接把 DNS 解析为各个 Pod 的 IP；常与 StatefulSet 搭配，让客户端依 Pod 名直连分布式节点。
- **想学习 Headless Service，在《Kubernetes 权威指南》中该看哪部分？**  
  参考介绍 Service 的章节（通常是第 6 章“服务发现与负载均衡”）关于 Service 类型与 DNS 的小节，里面会详细说明 ClusterIP=None 的配置与场景。
- **《Kubernetes 权威指南》当前最新版本是多少？**  
  最新是第 5 版（2022 年，覆盖 1.20+ 及云原生生态），相比你手头的第 2 版，章节结构和案例都已有多轮更新。
