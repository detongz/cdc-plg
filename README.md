# RFC: TiValue - Plugin System for TiCDC

- Author(s): [@eastfisher](https://github.com/eastfisher), [@mischaZhang](https://github.com/mischaZhang), [@yangwenmai](https://github.com/yangwenmai)

## 项目介绍

该项目旨在为 TiCDC 用户提供可扩展插件的方式，定制数据处理过程，提升 TiCDC 的扩展性。

## 背景&动机

随着 TiDB 的使用场景愈加广泛（实时数据分析、实时监听业务数据变更等），面向 TiDB 的实时数据导入导出能力、以及实时流式数据处理能力也提出了更高的要求。

TiDB 官方开发了 TiCDC 项目以解决 TiDB 实时数据同步的问题，通过拉取上游 TiKV 的数据变更日志，TiCDC 可以将数据解析为有序的行级变更数据输出到下游，并默认提供了两种 Sink 可将变更数据输出到 MySQL 协议兼容的数据库和 Kafka 消息队列。

然而，仅通过 MySQL 协议和 Kafka 显然不能满足下游灵活多样的 Sink 场景，比如：

- 实现不同于默认 Kafka Sink 特定的 Partition 路由策略
- 针对其他数据仓库 Sink 的特定优化，如：使用MySQL协议将数据写入 Doris 时的批量写入优化
- 将数据输出到其他数据源，如：Nats、Pulsar、S3，其他不支持 MySQL 协议的数据库等
- 其他 Sink 过程中的特定业务需求，如：针对某些敏感字段的数据脱敏，等等

针对以上每一种特定的业务测需求，通过修改 TiCDC 源代码，实现 Go 相关接口、重新编译 TiCDC 代码，往往是通过拉断代码的形式自己维护代码。用户需求的业务相关需求，不一定适合合并到上游；另一方面，企业用户通过修改 TiCDC 源码来增加的功能在 TiDB 版本更新时需要不断做 cherry pick和适配。用户自改逻辑自行编译 TiCDC 也由于编译环境与社区的偏差导致 TiCDC 稳定性与社区版本有出入。

因此我们希望通过这次 Hackathon 进行尝试，让 TiCDC 可以提供灵活、方便、安全的插件化 Sink 开发能力，让有定制化数据同步需求的用户可以自由地针对自身的业务场景，开发自定义 Sink 插件，让数据真正 Flow 起来~

## 项目设计

支持多种插件形式，用户可自定义数据同步逻辑。

实现除 MySQL协议 和 Kafka 之外的 Plugin Sink。https://github.com/pingcap/tiflow/tree/master/cdc/sink

### 插件形式

#### 插件和TiCDC在一个进程内运行

- Wasm (支持多语言)
- 用 gopher-lua 在程序里运行lua脚本 (lua语言)
- **.so** 插件程序 (支持可以编译成.so的任意语言)

#### TiCDC和插件在不同进程运行

该方式可以支持多种编程语言。TiCDC提供RPC或基于HTTP的接口

- 提供模板runner(类似https://github.com/apache/apisix-python-plugin-runner)或RPC IDL
- 提供SDK(类似https://pkg.go.dev/github.com/Kong/go-pdk)或定义HTTP Restful API

### 针对插件的管理


TiCDC 集群采用 Master-Worker 工作模式，每个 TiCDC 进程是无状态的，基于 pd 内置的 etcd 选举出全局唯一 Owner 节点负责对 Changefeed 进行统一调度，将 Changefeed 划分为 TablePipeline 这个最小同步单元之后，分发到不同的 Processor 节点上执行，执行过程中的状态信息保存到 pd 上。整个集群的所有角色节点都是高可用的。

无插件时，TiCDC 集群中的所有 TiCDC 进程运行的程序一致，此时 TiCDC 不同节点之间不存在数据处理不一致问题。引入插件后，有必要保证所有包含插件的 TiCDC 进程运行程序的一致。

我们提供了一种基于 2PC 的插件变更的实现思路，整个插件变更流程共分为 Prepare、Pause、Commit 三个阶段。

Prepare 阶段：通过 cdc cli 或 OpenAPI 调用上传插件接口，将插件文件上传至 Owner 节点（如果请求打到 Processor 节点上，会被路由至 Owner 节点）。Owner 节点此时开始执行 Prepare 操作，向所有 Processor 节点分发插件文件，直到所有节点返回成功后，执行下一步 Pause。

<img src="https://github.com/eastfisher/tivalve/raw/main/docs/assets/plugin_prepare.png">

Pause 阶段：此阶段会暂停所有正在运行的 Changefeed 任务，等待下一步 Commit 阶段做真正的插件切换操作。

<img src="https://github.com/eastfisher/tivalve/blob/main/docs/assets/plugin_pause.png">

Commit 阶段：对所有的 Changefeed 中的所有 TablePipeline 的 Sink 模块执行 Reload 操作（也可以优化下，只针对使用了 WasmPluginSink 实现的 TablePipeline 执行 Reload），重新初始化 Wasm Instance 时使用新版本的 Wasm 插件文件，从而实现版本更新。

<img src="https://github.com/eastfisher/tivalve/blob/main/docs/assets/plugin_commit.png">

以上三个阶段都是幂等的。

### 未来扩展：

- 目前只尝试针对 TiCDC Sink 这一个扩展点进行插件化改造，事实上 TiCDC 或者 DM 还有一些其他的扩展点可以挖掘插件能力，在引入更多插件扩展之后，TiFlow 可形成围绕 TiDB 数据库的功能强大的数据流处理生态，在一定场景上取代传统的流处理平台，简化技术架构。

- 可以通过 TiCDC 插件，提供足够多的 AP 数据库发送模板让，以配置化的形式让用户不需要编译 TiFlow 代码或任何插件即可完成数据同步
