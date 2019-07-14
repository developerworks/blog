# 可行性分析

## 消息编码格式

支持的目标消息格式

- Protobuf
- JSON

对于JSON来说, Spring有对Jackson内置的支持, Spring 的各个模块都在使用. 是一个比较成熟的方案.
对于Protobuf的映射问题, 可以使用 MapStruct 来解决. MapStruct 是编译时动态生成字节码. 性能比运行时动态生成要好很多.

支持 Protobuf 消息编码的要求

- 要能够通过Java POJO生成对应 proto 文件.
- 要能够在 POJO 和 Protobuf 的 Builder 模式之间能够映射.

## 数据分析工具开发分析

通过股票系统案例实现一个实时数据显示和分析的工具

- 股票列表
- 可以从股票列表中选择一只股票, 加入到监视列表, 监视列表中实时显示股票数据.
- 可设置刷线间隔(5秒, 10秒, 15秒等间隔)
- 数据从 Redis 中读取, 通过 Websocket 发送到客户端显示