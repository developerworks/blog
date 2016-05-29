嵌入模式允许你把 `Ranch` 监听器直接插入到你的监控树中. 如果整个应用程序其他部分挂掉了, 可以通过关闭监听器的方法, 来提供更好的容错控制.

## 嵌入(Embedding)

要嵌入 Ranch 到你自己的应用程序中, 只需要简单地把子进程规范添加到监控树中即可. 在应用程序的一个(一般在顶层Supervisor, 如果应用程序比较复杂, 也可能是其他层) `Supervisor` 的 `init/1` 函数中完成这个过程.

对于嵌入, Ranch 要求最少两种类型的子进程规范. 首先,需要添加 `ranch_sup` 到监控树, 只需要一次, 不管使用多少个监听器. 然后需要为每个监听器添加子进程规范.

> 可以添加多个监听器, 比如80端口的HTTP监听器, 和443端口的HTTPS监听器.

Ranch 提供了一个简便的辅助函数 `ranch:child_spec/6` 获取监听器的子进程规范, 其工作方式类似于 `ranch:start_listener/6`, 只是它不启动任何进程, 只是返回子进程规范.

对于 `ranch_sup`, 子进程规范足够的简单, 不需要辅助函数.

下面的例子添加 `ranch_sup` 和一个监听器到另一个应用程序的监控树中.

直接嵌入 Ranch 到监控树中

```erlang
init([]) ->
    RanchSupSpec = {
        ranch_sup, 
        {ranch_sup, start_link, []}, 
        permanent, 
        5000, 
        supervisor, 
        [ranch_sup]
    },
    ListenerSpec = ranch:child_spec(
        echo, 
        100, 
        ranch_tcp, 
        [{port, 5555}], 
        echo_protocol, 
        []
    ),
    {ok, {{one_for_one, 10, 10}, [RanchSupSpec, ListenerSpec]}}.
```

> 记住, 可以按需要添加多个监听器, 但是只能有一个 `ranch_sup` 子进程规范!

## Elixir 的 Supervisor 实现  

嵌入前的 `Supervisor`

```elixir
require Logger
defmodule RanchEmbededMode.Supervisor do
  use Supervisor

  def start_link do
    Logger.debug "Start supervisor."
    Supervisor.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init([]) do
    children = [
      worker(RanchEmbededMode.TcpAcceptor, []),
    ]
    Logger.debug "supervisor child spec #{inspect children}"
    opts = [strategy: :one_for_one, max_restarts: 3]
    Logger.debug "strategy #{inspect opts}"
    supervise(children, opts)
  end
end
```

嵌入前的监控树结构

> 嵌入前 `Ranch` 是一个独立的 `Application`

![嵌入前 Ranch 是一个独立的 Application][1]

> 嵌入前 `RanchEmbededMode` 应用程序监控树结构

![嵌入前 RanchEmbededMode 应用程序监控树结构][2]

嵌入后的 `SupervisorEmbed`

```elixir
require Logger
defmodule RanchEmbededMode.SupervisorEmbed do
  use Supervisor

  def start_link do
    Logger.debug "Start supervisor."
    Supervisor.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init([]) do
    children = [
      ranch_sup(),
      ranch_embeded_mode_listener()
    ]
    Logger.debug "supervisor child spec #{inspect children}"
    opts = [strategy: :one_for_one, name: __MODULE__]
    Logger.debug "strategy #{inspect opts}"
    supervise(children, opts)
  end

  def ranch_sup do
    supervisor(:ranch_sup, [], [shutdown: 5000])
  end

  @doc """
  Ranch 提供了一个辅助函数能够更便捷的创建监听器的 Child Spec
  """
  def ranch_embeded_mode_listener do
    :ranch.child_spec(
      :ranch_embeded_mode_listener,
      10,
      :ranch_tcp,
      [port: 5555],
      RanchEmbededMode.TcpProtocolHandler, []
    )
  end
end
```

嵌入后的监控树结构, 独立的 Ranch Application 不在了.

![嵌入后的监控树结构][3]

## 需要特别注意的地方

> 当 Ranch 作为独立的 Application 时, 请确保 Ranch 在当前应用程序之前启动
> 当 Ranch 作为嵌入模式运行时, 请确保`不要`在 `mix.exs` 或其他位置启动 Ranch, 当前应用程序的 Supervisor 会负责启动 Ranch, 并作为当前应用程序监控树的子树运行.

![图片描述][4]

独立模式下, 如果没有在 mix.exs 提前启动 ranch, 报如下错误:

![图片描述][5]

嵌入模式下, 如果在 mix.exs 中提前启动了 ranch, 报如下错误:

![图片描述][6]

> **建议**: 
> 在设计监控树的结构方面, 确保当 `ranch_sup` 挂掉时, 重启所有监听器. 详情请参考 Ranch 官网 Guide 的 [Internal 章节](https://github.com/ninenines/ranch/blob/master/doc/src/guide/internals.asciidoc)了解如何这样做.

## 示例程序

本文的示例代码位于 https://github.com/developerworks/ranch_embeded_mode

另外, 如果你想要给你的项目取个你喜欢的名字, 请执行下面的步骤

```
git clone https://github.com/developerworks/ex_ranch_server_tasks.git
cd ex_ranch_server_tasks
mix archive.build
mix archive.install                # 输入Y确认
mix ranch.new <project_name> --sup # 用实际的项目名称替换 <project_name>
```

独立模式使用`RanchEmbededMode.Supervisor.start_link`启动
嵌入模式使用`RanchEmbededMode.SupervisorEmbed.start_link`启动

## 参考资料

https://github.com/ninenines/ranch/blob/master/doc/src/guide/embedded.asciidoc


  [1]: https://segmentfault.com/img/bVvy81
  [2]: https://segmentfault.com/img/bVvy87
  [3]: https://segmentfault.com/img/bVvzls
  [4]: https://segmentfault.com/img/bVvzdl
  [5]: https://segmentfault.com/img/bVvzf9
  [6]: https://segmentfault.com/img/bVvzgd