# RFC: TiVP - Visual Plan For TiDB

- Author(s): [@eastfisher](https://github.com/eastfisher), [@mischaZhang](https://github.com/mischaZhang), [@yangwenmai](https://github.com/yangwenmai)
- Progress: NaN

## 项目介绍

该项目旨在为 TiCDC 用户提供插件的方式定制数据处理过程。

## 背景&动机

随着 TiDB 被运用到更加复杂的分析场景，客户场景通过 TiDB、TiFlash 满足不了业务需求，需要把数据引入到其他数据库上。

Pingcap 提供的把数据同步到异构数据库的解决方案就是通过 TiCDC 把数据打到 Kafka。而 Kafka 本身也是一个维护成本和消耗资源都足够昂贵的组件，Kafka之后还要对接其他服务消费这部分数据才能发到对应数据库。

在业务测，修改源代码实现 Go 相关接口、编译 TiFlow 代码是不合理的，因此我们希望 TiCDC 可以提供像同为 golang 实现的 ApiSix 微服务网关一样的插件能力，让有数据同步需求的用户可以自由地定制贴合业务的逻辑，而且减少组件依赖。

## 项目设计

用 Wasm，go plugin，lua 三种形式支持用户自定义数据同步逻辑。

实现除 MQ 和 MySQL 之外的 Plugin sink。https://github.com/pingcap/tiflow/tree/master/cdc/sink

未来扩展：

- 可以通过 TiCDC 插件，提供足够多的 AP 数据库发送模板让，以配置化的形式让用户不需要编译 TiFlow 代码或任何插件即可完成数据同步

[@eastfisher](https://github.com/eastfisher) focus on WASM

[@mischaZhang](https://github.com/mischaZhang) forcus on Lua and Go Plugins
