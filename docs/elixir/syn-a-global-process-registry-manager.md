## 什么是进程注册表?

全局进程注册表, 用单个 `Key` 在集群中的所有节点上注册一个进程. 其作用类似于 DNS 服务器的作用: `通过域名获取一个IP地址`, 这里的进程注册表的作用是: 通过 `Key` 获取进程, 简单的说就是通过索引`Key` 去进程 Hash 表中查询对应的 `Pid`.

> 经典用例: 
> - 在系统上注册一个处理物理设备连接的进程(使用其序列号)
> - 聊天客户端


## 什么是进程组?

进程组是一个命名组, 包含多个最终运行在多个不同节点上的进程. 通过组名称, 可以获取运行在集群中任意节点上的所有进程, 或者发布一条消息给进程组中的所有进程. 该机制可以实现 `发布/订阅` 模式.

> 经典用例:
> - 聊天室
> - 发布/订阅模式的应用场景

## 什么是 Syn ?

- 一个全局进程注册表
- 一个进程组管理器
- 任何项式可以用作键和名称
- 消息可以发送给进程组中的所有成员进程(PubSub机制)
- 快速写
- 自动处理冲突决策(网络分区)
- 可配置回调
- 自动监控进程, 死亡时自动移除

在任何分布式系统中都要面对一致性的问题, 通常用一个主节点执行所有鞋操作, (通过[领导者选举](http://en.wikipedia.org/wiki/Leader_election)机制) 或通过[原子事务](http://en.wikipedia.org/wiki/Atomicity_(database_systems)).

Syn 诞生于物联网领域的应用. 在这个场景下, 用于标识一个进程的Key 一般是物理对象的唯一标识符(例如, 序列号,或Mac地址), 因此, 这些键已经保证了唯一性. 

写速度在Syn架构中是一个决定性因素, 因此, 可用性大于一致性, Syn 选择了[最终一致性](http://en.wikipedia.org/wiki/Eventual_consistency)

## 安装

### Rebar3

如果使用 [rebar3](), 在 `rebar.config` 配置文件添加 `syn` 作为依赖

```
{syn, {git, "git://github.com/ostinelli/syn.git", {tag, "1.3.1"}}}
```

或者, 如果使用 [Hex.pm](https://hex.pm/) 作为包管理器(和 [rebar3_hex](https://github.com/hexpm/rebar3_hex))插件

```
{syn, "1.3.1"}
```

然后, 编译

```
$ rebar3 compile
```

### Rebar2

如果使用 [rebar](), 在 `rebar.config` 配置文件中添加 `syn` 依赖:

```
{syn, ".*", {git, "git://github.com/ostinelli/syn.git", {tag, "1.3.1"}}}
```

然后, 获取依赖并编译:

```
$ rebar get-deps
$ rebar compile
```

## 使用

### 设置

确保Syn已经在你的应用程序中启动, 可以添加到 .app 文件中让它随你的应用自动启动, 也可以通过 `syn:start().` 手工启动.

你的应用程序在集群中运行, 一旦节点相互连接, 确保初始化 Sync (这回设置底层的 mnesia 后端)

```
syn:init().
```

> 请保证在一个节点上只初始化一次Syn, 即使该节点从集群中断开, 并重连到集群, 不要重新初始化它. 这回让 Syn 不能正确地自动决断冲突

推荐的启动位置为: 主应用模块的 `start/2` 函数.

```erlang
-module(myapp_app).
-behaviour(application).

-export([start/2, stop/1]).

start(_StartType, _StartArgs) ->
    %% connect to nodes
    connect_nodes(),
    %% init syn
    syn:init(),
    %% start sup
    myapp_sup:start_link().

connect_nodes() ->
    %% list of nodes contained in ENV variable `nodes`
    Nodes = application:get_env(nodes),
    %% connect to nodes
    [net_kernel:connect_node(Node) || Node <- Nodes].
```

### 进程注册

注册一个进程:

```erlang
syn:register(Key, Pid) -> ok | {error, Error}.

Types:
    Key = any()
    Pid = pid()
    Error = taken | pid_already_registered
```

注册一个进程并附加元数据:

```erlang
syn:register(Key, Pid, Meta) -> ok | {error, Error}.

Types:
    Key = any()
    Pid = pid()
    Meta = any()
    Error = taken | pid_already_registered
```

> 当一个进程注册之后, Syn 会自动地监控它. 因此需要的时候你可以处理进程的'DOWN'事件

通过Key获取进程的Pid:

```erlang
syn:find_by_key(Key) -> Pid | undefined.

Types:
    Key = any()
    Pid = pid()
```

获取一个进程的Pid, 及其元数据:

```erlang
syn:find_by_key(Key, with_meta) -> {Pid, Meta} | undefined.

Types:
    Key = any()
    Pid = pid()
    Meta = any()
```

通过Pid查找Key:

```erlang
syn:find_by_pid(Pid) -> Key | undefined.

Types:
    Pid = pid()
    Key = any()
```

通过Pid及其元数据查找Key:

```erlang
syn:find_by_pid(Pid, with_meta) -> {Key, Meta} | undefined.

Types:
    Pid = pid()
    Key = any()
    Meta = any()
```

注销一个进程:

```erlang
syn:unregister(Key) -> ok | {error, Error}.

Types:
    Key = any()
    Error = undefined
```

> 你不需要注销死亡进程的Key, 因此它被Syn自动监控, 并且被自动移除. 如果你在进程死亡前手动注销一个进程(看下面), 进程的退出回调函数可能不会调用.

获取集群中测注册进程数:

```erlang
syn:registry_count() -> non_neg_integer().
```

获取指定节点上的注册进程数:

```erlang
syn:registry_count(Node) -> non_neg_integer().

Types:
    Node = atom()
```

### 进程组

> 不需要手动创建/删除进程组, Syn 将会帮你管理这些.

添加一个进程到组:

```erlang
syn:join(Name, Pid) -> ok | {error, Error}.

Types:
    Name = any()
    Pid = pid()
    Error = pid_already_in_group
```

> 当一个进程加入一个组, Syn会自动地监控它.

从组中删除一个进程:

```erlang
syn:leave(Name, Pid) -> ok | {error, Error}.

Types:
    Name = any()
    Pid = pid()
    Error = undefined | pid_not_in_group
```

> 你不需要删除已经死亡的进程, 因此其被Syn监控, 并且在死亡的时候自动从组中删除.

获取进程组中的成员进程列表:

```erlang
syn:get_members(Name) -> [pid()].

Types:
    Name = any()
```

> 不管你是从哪个节点上调用 `syn:get_members(Name)`, Syn保证了其结果顺序是一样的. 但是并没有保证这个结果的顺序和进程加入的顺序一样. 

查询一个进程是否在一个组中:

```erlang
syn:member(Pid, Name) -> boolean().

Types:
    Pid = pid()
    Name = any()
```

发布一条消息给所有成员:

```erlang
syn:publish(Name, Message) -> {ok, RecipientCount}.

Types:
    Name = any()
    Message = any()
    RecipientCount = non_neg_integer()
```

调用所有的组成员, 并获取其响应:

```erlang
syn:multi_call(Name, Message) -> {Replies, BadPids}.
```

等同于 `syn:multi_call(Name, Message, 5000).`

```erlang
syn:multi_call(Name, Message, Timeout) -> {Replies, BadPids}.

Types:
    Name = any()
    Message = any()
    Timeout = non_neg_integer()
    Replies = [{MemberPid, Reply}]
    BadPids = [MemberPid]
      MemberPid = pid()
      Reply = any()
```

> 接收所有成员的响应的过程可能会超时, 超时值可通过 `Timeout()` 参数设定, 在 `Timeout()` 超时后, 所有没有及时获得回应的成员进程或者崩溃的进程会被移到 `BadPids` 列表中.

响应的格式:

```erlang
{syn_multi_call, CallerPid, Message}

Types:
    CallerPid = pid()
    Message = any()
```

To reply, every member must use the method:

```erlang
syn:multi_call_reply(CallerPid, Reply) -> ok.

Types:
    CallerPid = pid()
    Reply = any()
```
    
## 选项

选项可以设置在环境变量 `syn` 中, 可能最好的是使用应用程序配置文件(在发布包的 `sys.config` 配置文件中). 所有可用的选项包括:

```erlang
{syn, [
    %% define callback function on process exit
    {registry_process_exit_callback, [module1, function1]},

    %% define callback function on conflicting process (instead of kill)
    {registry_conflicting_process_callback, [module2, function2]}
]}
```

这些选项在下面解释.

### 注册表选项
    
运行设置的进程注册表选项包括:

- `registry_process_exit_callback`
- `registry_conflicting_process_callback`

#### 进程退出回调

`registry_process_exit_callback` 选项允许你指定 `module` 和 `function`作为注册进程退出的回调函数. 该回调仅在进程运行的节点上调用.

回调函数定义为:

```erlang
CallbackFun = fun(Key, Pid, Meta, Reason) -> any().

Types:
    Key = any()
    Pid = pid()
    Meta = any()
    Reason = any()
```

`Key` 和 `Pid` 为进程退出时的 `Key` 和 `Pid`, `Reason` 为该进程退出的原因.

例如, 当进程退出时, 如果你想要打印日志, 定义如下回调函数

```erlang
-define(my_callback).
-export([callback_on_process_exit /4]).

callback_on_process_exit() ->
    error_logger:info_msg(
        "Process with key ~p, Pid ~p and Meta ~p exited with reason ~p~n",
        [Key, Pid, Meta, Reason]
    ).
```

并设置选项:

```erlang
{syn, [
    %% define callback function
    {registry_process_exit_callback, [my_callback, callback_on_process_exit]}
]}
```

如果你不设置该选项, 不会触发回调.

> 如果进程因为冲突而死亡, 进程退出回调函数任然会被调用, 这时 `Key` 和 `Meta` 的值为 `undefined`.

#### 通过回调来决断冲突

在有竞态条件的情况下, 或者网络断开, 一个 `Key` 可能在两个不同的节点上同时注册, 当网络恢复或者竞态条件消失的时候, 进程注册表会导致一个名称冲突.

当这种情况发生时, `Syn` 会在冲突的进程中选择一个, 杀死另一个(互斥, 只能存在一个)来解决进程注册表冲突. Syn 将会抛弃运行在冲突决断的节点上的进程, 并且默认将会发送一个 `kill` 信号(`exit(Pid, kill`)来杀死它.

如果这不是你期望的, 你可以设置 `registry_conflicting_process_callback` 选项来让 Syn 触发一个回调, 这样你可以指向一些自定义的操作(比如 `graceful shutdown`). 在这种场景下, 进程不会被 Syn 杀掉, 你必须决定要做什么事情. 该回调仅在进程运行的节点上被调用.

> 注: 实际处理应该避免自动处理冲突. 应该重启节点, 让它去同步 Mnesia 集群的数据.

该回调函数定义为:

```erlang
CallbackFun = fun(Key, Pid, Meta) -> any().

Types:
    Key = any()
    Pid = pid()
    Meta = any()
```

`Key`, `Pid` 和 `Meta` 属于被抛弃的进程的. 例如, 如果你想发送一个 `shutdown` 时间给被抛弃的进程:

```erlang
-module(my_callback).
-export([callback_on_conflicting_process/3]).

callback_on_conflicting_process(_Key, Pid, _Meta) ->
    Pid ! shutdown
```

选项设置:

```erlang
{syn, [
    %% define callback function
    {registry_conflicting_process_callback, [my_callback, callback_on_conflicting_process]}
]}
```

> 重要事项: 冲突决断方法应该在所有集群的节点上以相同的的方式定义. 不同节点上如果存在不同的冲突决断方式, 将会导致意外的结果. 

### 进程组选项

当前没有进程组选项

## 内部机制


在底层, Syn 采用脏读/写到分布式的基于内存的Mnesia表中, 跨集群的多个节点进行复制.

要自动决断冲突, Syn 实现了一个[unsplit](https://github.com/uwiger/unsplit)机制的一个简化版本.


## Syn 的压力测试结果

|                                 | 1 Node | 2 Nodes | 3 Nodes | 4 Nodes
| --------------------------------| ------ | ------- | ------- | --------
| Reg / second                    | 106,324| 52,792  | 60,958  | 40,929
| Retrieve registered Key (ms)    | 0      | 0       | 0       | 56
| Unreg / second                  | 105,506| 50,591  | 67,042  | 42,896
| Retrieve unregistered Key (ms)  | 0      | 0       | 0       | 0
| Re-Reg / second                 | 106,424| 51,322  | 77,258  | 47,125
| Retrieve re-registered Key (ms) | 0      | 0       | 0       | 0
| Retrieve Key of killed Pid (ms) | 719    | 995     | 1,577   | 1,825

## 项目主页

https://github.com/ostinelli/syn