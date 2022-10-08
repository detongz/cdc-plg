# RFC: TiValue - Plugin System for TiCDC

- Author(s): [@eastfisher](https://github.com/eastfisher), [@mischaZhang](https://github.com/mischaZhang), [@yangwenmai](https://github.com/yangwenmai)
- Progress: NaN

## 项目介绍

该项目旨在为 TiCDC 用户提供插件的方式定制数据处理过程。

## 背景&动机

随着 TiDB 的使用场景愈加广泛（实时数据分析、实时监听业务数据变更等），面向 TiDB 的实时数据导入导出能力、以及实时流式数据处理能力也提出了更高的要求。

TiDB 官方开发了 TiCDC 项目以解决 TiDB 实时数据同步的问题，通过拉取上游 TiKV 的数据变更日志，TiCDC 可以将数据解析为有序的行级变更数据输出到下游，并默认提供了两种 Sink 可将变更数据输出到 MySQL 协议兼容的数据库和 Kafka 消息队列。

然而，仅通过 MySQL 协议和 Kafka 显然不能满足下游灵活多样的 Sink 场景，比如：

- 实现不同于默认 Kafka Sink 特定的 Partition 路由策略
- 针对其他数据仓库 Sink 的特定优化，如：使用MySQL协议将数据写入 Doris 时的批量写入优化
- 将数据输出到其他数据源，如：Nats、Pulsar、S3，其他不支持 MySQL 协议的数据库等
- 其他 Sink 过程中的特定业务需求，如：针对某些敏感字段的数据脱敏，等等

针对以上每一种特定的业务测需求，通过修改 TiCDC 源代码，实现 Go 相关接口、重新编译 TiCDC 代码是不合理的。一方面，这些特定需求可能是业务相关的，并不适合放入官方仓库；另一方面，通过修改 TiCDC 源码来增加这类非常灵活的 Sink 功能，既不利于 TiCDC 发版的稳定性，更不利于 TiCDC 内核本身的安全性。

因此我们希望通过这次 Hackathon 进行尝试，让 TiCDC 可以提供灵活、方便、安全的插件化 Sink 开发能力，让有定制化数据同步需求的用户可以自由地针对自身的业务场景，开发自定义 Sink 插件，让数据真正 Flow 起来~

提供类似的插件化能力的相关项目：

- [Apache APISIX](https://apisix.apache.org/)

## 项目设计

支持多种插件形式，用户可自定义数据同步逻辑。

实现除 MySQL协议 和 Kafka 之外的 Plugin Sink。https://github.com/pingcap/tiflow/tree/master/cdc/sink

### 插件形式

#### 插件和TiCDC在一个进程内运行

- 用 gopher-lua 在程序里运行lua脚本
- **.so** 插件程序

#### TiCDC和插件在不同进程运行

该方式可以支持多种编程语言。TiCDC提供RPC或基于HTTP的接口

- 提供模板runner(类似https://github.com/apache/apisix-python-plugin-runner)或RPC IDL
- 提供SDK(类似https://pkg.go.dev/github.com/Kong/go-pdk)或定义HTTP Restful API

### 未来扩展：

- 可以通过 TiCDC 插件，提供足够多的 AP 数据库发送模板让，以配置化的形式让用户不需要编译 TiFlow 代码或任何插件即可完成数据同步
