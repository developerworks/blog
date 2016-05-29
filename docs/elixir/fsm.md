## OTP 原理

有限状态机被描述为如下形式的一组关系集合.

![有限状态机被描述为如下形式的一组关系集合][1]

> 含义可以解释为:
> 如果在状态`S`的时候发生了事件`E`, 那么执行动作`A`并且使状态`S`过渡( transition )到状态`S'`.

对于使用 `gen_fsm` 行为的有限状态机来说, 状态`过渡规则`被实现为一些 Erlang 函数, 他们遵循如下的约定:

```erlang
StateName(Event, StateData) ->
    .. code for actions here ...
    {next_state, StateName', StateData'}
```

对于这种形式的函数, 状态机存在多少个状态, 就应该定义多少个这样的状态处理函数


> ### 初始化回调

![初始化回调][2]

当使用 `gen_fsm:start/3,4` 或 `gen_fsm:start_link/3,4` 启动的时候, 该函数被新的进程调用来初始化. 如果初始化成功, 该函数应该返回 `{ok,StateName,StateData}, {ok,StateName,StateData,Timeout} or {ok,StateName,StateData,hibernate}`, 其中 `StateName` 为该状态机的初始状态名, `StateData` 为该状态机的初始状态数据. 如果提供了一个整数超时值, 当在此时间范围内没有接受到任何消息时, 触发一个超时. 超时以原子 `timeout` 标识, 并且应当被 `Module:StateName/2` 回调函数处理. 原子 `infinity` 表示无线超时值, 这是默认的.

> ### 事件的产生

![事件的产生][3]

当一个 `gen_fsm` 进程接收到使用 `gen_fsm:send_all_state_event/2` 发送的一个事件时, 该回调函数被调用来处理该事件. `StateName` 为 `gen_fsm` 的当前状态名称.  该函数接收3个参数, 分别是事件名称(`term`), 状态名称(`atom`) 和一个状态数据(`term`), 返回一个结果(`term`)

结果包括4种:

```
{next_state,NextStateName,NewStateData}             # 进入下一个状态
{next_state,NextStateName,NewStateData,Timeout}     # 带超时设置进入下一个状态
{next_state,NextStateName,NewStateData,hibernate}   # 进入下一个状态并休眠
{stop,Reason,NewStateData}                          # 停止, 并调用terminate/3, 终止状态机进程
```

![事件的产生][4]

异步地发送一个事件给 `FsmRef`状态机进程, 并立即返回 `ok`. 状态机进程调用 `Module:handle_event/3` 来处理该事件.

参数说明请参考 `send_event/2`

`send_event` 和 `send_all_state_event` 的区别是: 哪一个事件处理函数来处理事件. 当每个状态以相同的方式处理的时候, 使用该函数发送事件, 并且只需要一个`handle_event`子句, 而不需要每一个状态名函数都需要一个`handle_event`子句.

> ### 事件的处理


![事件的处理][5]

对于每一个可能的状态都应该有一个这样的函数实例. 当 `gen_fsm` 使用 `gen_fsm:send_event/2`发出的一个事件时, 该函数的一个和挡墙状态名相同的同名函数实例被调用来处理这个事件. 如果发生超时也可以被调用.

如果发生超时, Event 是一个原子 `timeout`, 否则为传递给 `send_event/2` 的参数.

`StateData` 为 `gen_fsm` 的状态数据.

如果函数返回 `{next_state, NextStateName, NewStateData}`, {next_state, NextStateName, NewStateData, Timeout}`, `{next_state, NextStateName, NewStateData, hibernate}`状态机继续执行


启动状态机进程

```
start_link(Module, Args, Options) -> Result
start_link(FsmName, Module, Args, Options) -> Result
```

> Option = {debug,Dbgs} | {timeout,Time} | {spawn_opt,SOpts}

关于选项, 如果给定了 `{timeout,Time}` 参数, 状态机初始化必须在 `Time` 毫秒内完成, 否则, 进程终止, 启动函数返回 `{error,timeout}` 错误

如果成功初始化, 该函数返回 `{ok,Pid}`, 其中 `Pid` 为该状态机进程的进程ID. 如果已经存在一个名为 `FsmName` 的进程, 函数返回 `{error, {already_started, Pid}}`



## 实践

我们这里使用Elixir作为示例来演示如何创建一个状态机来解决实际的问题. 这里我要解决的问题是把服务器端处理网络协议的进程归纳为在多个状态之间过渡的这么一个状态机.


### 创建项目

第一步: 创建一个项目

```
➜  /tmp mix new ex_fsm_example --sup
* creating README.md
* creating .gitignore
* creating mix.exs
* creating config
* creating config/config.exs
* creating lib
* creating lib/ex_fsm_example.ex
* creating test
* creating test/test_helper.exs
* creating test/ex_fsm_example_test.exs

Your Mix project was created successfully.
You can use "mix" to compile it, test it, and more:

    cd ex_fsm_example
    mix test

Run "mix help" for more commands.
```

第二步: 创建子目录, 并增加一个模块

```
cd ex_fsm_example/lib
mkdir ex_fsm_example
cd ex_fsm_example
touch worker.ex
```

第三步: ExFsmExample.Worker 基本实现

```
defmodule ExFsmExample.Worker do
  @behaviour :gen_fsm
  def start_link() do
    :gen_fsm.start_link({:local, __MODULE__}, __MODULE__, [], [])
  end
end
```

第四步: 编译

```
mix compile
lib/ex_fsm_.../worker.ex:1: warning: undefined behaviour function code_change/4 (for behaviour :gen_fsm)
lib/ex_fsm_.../worker.ex:1: warning: undefined behaviour function handle_event/3 (for behaviour :gen_fsm)
lib/ex_fsm_.../worker.ex:1: warning: undefined behaviour function handle_info/3 (for behaviour :gen_fsm)
lib/ex_fsm_.../worker.ex:1: warning: undefined behaviour function handle_sync_event/4 (for behaviour :gen_fsm)
lib/ex_fsm_.../worker.ex:1: warning: undefined behaviour function init/1 (for behaviour :gen_fsm)
lib/ex_fsm_.../worker.ex:1: warning: undefined behaviour function terminate/3 (for behaviour :gen_fsm)
Compiled lib/ex_fsm_example/worker.ex
```

输出告诉我们, `gen_fsm` 行为的哪些函数还么有实现. 依次添加函数实现即可.

第五步: 启动,并测试

模块基本结构

```elixir
defmodule ExFsmExample.Worker do
  @behaviour :gen_fsm

  def start_link() do
    :gen_fsm.start_link({:local, __MODULE__}, __MODULE__, [], [])
  end

  def init(_args) do
    state = %{socket: :undefined}
    {:ok, :on, state}
  end

  def handle_event(event, state_name, state_data) do
    {:next_state, state_name, state_data}
  end

  def handle_sync_event(event, from, state_name, state_data) do
    {:next_state, state_name, state_data}
  end

  def handle_info(info, state_name, state_data) do
    {:next_state, state_name, state_data}
  end

  def terminate(reason, state_name, state_data) do
    nil
  end

  def code_change(_old_vsn, state_name, state_data, _extra) do
    {:ok, state_name, state_data}
  end
end
```

有限状态机的进程信息

![有限状态机的进程信息][6]

### 一个门禁的例子

这个例子是根据 [code_lock](http://erlang.org/doc/design_principles/fsm.html#id69827) 用 Elixir 重写的

```elixir
require Logger
defmodule ExFsmExample.CodeLock do
  @moduledoc """
  一个经典的密码锁状态机.
  应用场景:
  1. 比如出入办公室的自动门, 输入密码门打开, 10秒钟后自动关闭
  """

  @doc """
  用一个密码初始化这个状态机, 反转密码的顺序
  """
  def start_link(password) do
    Logger.debug "门禁的密码为: #{inspect password}"
    :gen_fsm.start_link({:local, __MODULE__}, __MODULE__, Enum.reverse(password), [])
  end

  def button(digit) do
    Logger.debug "您输入了 #{digit}"
    :gen_fsm.send_event(__MODULE__, {:button, digit})
  end

  @doc """
  初始化状态包含一个字符输入队列, 和一个密码作为初始状态
  """
  def init(password) do
    Logger.debug "密码的逆序值为: #{inspect password}"
    {:ok, :locked, {[], password}}
  end

  @doc """
  当外部调用button/1函数输入数字的时候, 执行这个状态函数
  """
  def locked({:button, digit}, {sofar, password}) do
    now = [digit | sofar]
    Logger.debug "Now: #{inspect now}, password #{inspect password}"
    case [digit | sofar] do
      ^password ->
        # do_unlock()
        Logger.debug "已打开, 3秒后自动关闭"
        {:next_state, :open, {[], password}, 3000}
      incomplete when length(incomplete) < length(password) ->
        Logger.debug "#{inspect incomplete}"
        {:next_state, :locked, {incomplete, password}}
      _wrong ->
        Logger.debug "密码错误"
        {:next_state, :locked, {[], password}}
    end
  end

  def open(:timeout, state) do
    # do_lock()
    Logger.debug "超时, 自动关闭"
    {:next_state, :locked, state}
  end
end
```

  [1]: https://segmentfault.com/img/bVvRtA
  [2]: https://segmentfault.com/img/bVvRsJ
  [3]: https://segmentfault.com/img/bVvRrt
  [4]: https://segmentfault.com/img/bVvRrK
  [5]: https://segmentfault.com/img/bVvRsm
  [6]: https://segmentfault.com/img/bVvREN
