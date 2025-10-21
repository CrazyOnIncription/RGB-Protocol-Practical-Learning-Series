# 7.macOS环境下Esplora区块链浏览器部署实践

在为RGB协议测试环境搭建Esplora服务时，需要在macOS上提供Bitcoin testnet的REST API查询服务。本文记录了从Docker方案到本机编译的完整部署过程，以及在资源优化方面的关键决策。

### Docker方案的网络限制

初期尝试使用Docker快速部署，但遇到了macOS特有的网络隔离问题。Docker容器中的`host.docker.internal`解析到`192.168.65.254`，而Bitcoin Core配置为监听`127.0.0.1:18332`和本机IP，导致容器无法访问宿主机的Bitcoin服务。

错误日志显示：

`WARN - failed to connect daemon at 192.168.65.254:18332: Connection refused`

尝试了多种Docker网络配置（host模式、自定义网络）均无法解决，最终决定采用本机编译方案。

### 版本选择的关键差异

系统中已有标准electrs，但发现其仅提供Electrum RPC协议，不支持HTTP REST API。需要使用Blockstream维护的esplora-electrs变体：

bash

`git clone <https://github.com/Blockstream/electrs.git> esplora-electrs cd esplora-electrs cargo build --release`

版本间的主要差异：

* 标准electrs：`-cookie-file`参数，仅支持Electrum协议
* esplora-electrs：`-cookie`参数，支持`-http-addr`选项

### 全节点模式的资源消耗挑战

初始尝试完整索引模式时遇到了严重的资源管理问题。监控数据显示数据库异常膨胀，从预期的150GB左右增长到337GB峰值，远超正常范围。关键问题包括：

**资源消耗异常**：

* 数据库大小异常增长到337GB（正常应为\~150GB）
* 系统内存使用率持续98%，频繁swap操作
* CPU负载维持在3.0-4.0，系统资源接近饱和
* 磁盘I/O达到18TB读取和4.6TB写入的异常水平

**同步过程不稳定**：

* 区块同步经常停滞在特定高度（如4,655,778）
* 历史索引进度频繁回退（从4,296,773回到3,518,999个区块）
* 进程被OOM killer反复终止，导致重复工作

**数据库管理问题**：

* RocksDB压缩过程无法正常完成
* 数据库文件碎片化严重
* 索引数据未能及时持久化，重启后丢失进度

这种异常的资源消耗模式表明全节点模式在当前硬件配置下难以稳定运行。

### Light Mode的性能优化

针对资源约束问题，启用light模式配置：

bash

`./target/release/electrs \\ --network testnet \\ --daemon-dir "/Volumes/MAC_Programs" \\ --daemon-rpc-addr "127.0.0.1:18332" \\ --db-dir "/Volumes/DEV/esplora-electrs-db" \\ --http-addr "127.0.0.1:3001" \\ --monitoring-addr "127.0.0.1:14225" \\ --lightmode \\ -vvv`

优化效果：

* 存储需求：从500GB降至300GB
* 数据库大小：从160GB压缩至90GB
* 内存占用：显著降低，避免OOM终止
* 功能影响：保留UTXO查询和地址历史功能

### 索引过程的技术分析

通过日志分析，识别出electrs的三个主要处理阶段：

**Stage 1 - Transaction Store**

`INFO 4,655,777 blocks were added`

存储所有交易数据到txstore数据库。

**Stage 2 - History Index**

`INFO 4,296,773 blocks were indexed`

构建地址到交易的映射关系。

**Stage 3 - Real-time Sync**

`INFO hash=... height=4656666 @ 2025-08-31T13:47:11Z (1 left to index)`

切换到实时处理模式，逐个处理新区块。

每个阶段完成后执行RocksDB压缩：

`DEBUG starting full compaction on RocksDB { path: "/Volumes/DEV/esplora-electrs-db/testnet/newindex/history" }`

### 部署架构设计

**存储分配策略**：

* Bitcoin Core数据：/Volumes/MAC\_Programs（\~200GB）
* Esplora索引数据：/Volumes/DEV（\~90GB）
* 分布在不同磁盘以优化I/O性能

**服务端口配置**：

* Bitcoin RPC：127.0.0.1:18332
* Esplora HTTP API：127.0.0.1:3001
* Electrum RPC：127.0.0.1:60001
* Prometheus监控：127.0.0.1:14225

### 服务状态验证

索引完成的关键指标：

* 处理模式从批量历史数据切换到实时单区块处理
* 数据源从"using BlkFiles"切换到"using Bitcoind"
* 开始活跃处理mempool交易

API服务验证：

bash

`curl -s <http://127.0.0.1:3001/api/blocks/tip/height> curl -s <http://127.0.0.1:3001/api/blocks/tip/hash> curl -s <http://127.0.0.1:3001/api/address/{address}`>

### 技术总结

在macOS环境下部署Esplora服务的关键经验：

1. **平台适配性**：本机编译比容器化方案更可靠，避免了网络层面的复杂性
2. **版本选择**：准确识别软件变体的功能差异，选择符合需求的版本
3. **资源优化**：light模式在保持核心功能的前提下大幅降低资源需求
4. **过程监控**：通过日志分析理解处理阶段，便于问题诊断和进度评估

完整的索引过程耗时约12小时，最终提供稳定的Bitcoin testnet REST API服务，为RGB协议开发测试提供了可靠的区块链数据查询基础设施。
