Ranch 是一个很牛逼的Acceptor pool, 它让Phoenix跑到了[200W的并发](http://www.phoenixframework.org/blog/the-road-to-2-million-websocket-connections).

## 监听器

监听器(`listener`)是一组进程,它的用途是在一个指定的端口上监听`新的连接`. 它管理一组`acceptor`进程, 每个`acceptor`无限地等待接受客户端的`连接请求`. 当接受一个连接时, 它启动一个新的进程执行`协议处理代码`(一般是二进制数据格式的处理). 

监听器监控所有的`acceptor`进程和连接进程, 让开发者只关注应用逻辑的开发.

### 启动监听器

监听器可以启动和停止

当启动一个监听器时, 需要设置一些选项, 包括:

- 一个监听器的名字, 可以通过名字与其交互
- 在acceptor池中的`acceptors`进程的数量
- 传输处理模块(`ranch_ssl`或`ranch_tcp`)以及相关的选项
- 协议处理器以及相关的选项

监听器通过调用`ranch:start_listener/6`启动, 在这之前, 需要启动`ranch`应用.

启动`ranch`应用程序

```erlang
ok = application:start(ranch).
```


## 协议

协议处理程序启动一个`连接进程`, 并用于定义在`该进程`中执行的逻辑.

下面是一个用`Elixir`编写的示例:

### 启动监听器

```elixir
require Logger
defmodule EchoServerAcceptor do
  @config Application.get_env(:tcp_server, :ranch)
  def start_link do
    {:ok, _} = :ranch.start_listener(
      @config[:listener_name],
      @config[:acceptors],
      @config[:transport_type],
      @config[:transport_options],
      TcpServer.ConnectionWorker,
      []
    )
  end
end
```

### 协议处理模块

```elixir
defmodule EchoServerConnectionWorker do
  @behaviour :ranch_protocol
  def start_link(ref, socket, transport, opts) do
    {ip, port} = get_address(socket)
    pid = spawn_link(__MODULE__, :init, [ref, socket, transport, opts])
    Logger.info 'accepted connection from: "#{ip}:#{port}"'
    {:ok, pid}
  end
  def init(ref, socket, transport, _opts = []) do
    :ok = :ranch.accept_ack(ref)
    transport.setopts(socket, [nodelay: :true])
    :erlang.process_flag(:trap_exit, true)
    loop(socket, transport)
  end
  def loop(socket, transport) do
    case transport.recv(socket, 0, 5000) do
      {:ok, data} ->
        transport.send(socket, data)
        loop(socket, transport)
      _ ->
        :ok = transport.close(socket)
    end
  end
end
```

### 编写一个协议处程序

所有的协议处理程序都必须实现`ranch_protocol`行为, 它定义了一个回调函数`start_link/4`, 该回调函数负责创建一个新的进程用于处理这个连接. 它接受4个参数, 分别是:

1. 监听器的名字
2. 套接字
3. 使用的处理程序
4. 以及调用`ranch:start_listener/6`时传递的协议选项

该回调函数必须返回`{ok, Pid}`, `Pid`为新创建进程的进程ID.

## 死锁的问题

> `ranch:accept_ack(ref)` 是为了保证套接字的控制权交给调用者

> 对于`gen_server`或`gen_fsm`行为的特殊性, `start_link`在子进程的`init`返回之前是不会返回的. 这是有问题的, 因此不能够在`init`回调中调用`ranch:accept_ack/1`, 由此产生`循环调用`, 因此导致死锁.

> 为什么会导致死锁,因为 `ranch:accept_ack/1` 是一个阻塞操作, 在`init`中调用`ranch:accept_ack/1`会导致`init`一直无法返回, 又因为`gen_server`的机制, `start_link`一直要等待`init`返回, `start_link`自身才会返回, 由此导致了死锁.

在Elixir中标准的GenServer为, 

```elixir
defmodule Stack do
  use GenServer
  # Client
  def start_link(default) do
    GenServer.start_link(__MODULE__, default)
  end
  def push(pid, item) do
    GenServer.cast(pid, {:push, item})
  end
  def pop(pid) do
    GenServer.call(pid, :pop)
  end
  def handle_call(:pop, _from, [h|t]) do
    {:reply, h, t}
  end
  def handle_call(request, from, state) do
    # Call the default implementation from GenServer
    super(request, from, state)
  end
  def handle_cast({:push, item}, state) do
    {:noreply, [item|state]}
  end
  def handle_cast(request, state) do
    super(request, state)
  end
end
```

这是在Elixir [GenServer文档](http://elixir-lang.org/docs/stable/elixir/GenServer.html#callbacks) 的 `Client / Server APIs`部分给出的(可以在浏览器中搜索"Client / Server APIs")

上述代码我们可以看, 调用的是 `GenServer.start_link`, 为了避免循环调用, 有两种方法来解决这个问题:

#### 1. 使用`proc_lib:start_link`, `proc_lib:init_ack`, `gen_server:enter_loop`绕过`gen_server`的默认行为

> 这种方式是推荐的方式

规避这个问题需要使用 `gen_server:enter_loop`, 该方法让`已经存在的独立进程`(通过 `proc_lib:start_link` 创建的进程)转换为 `gen_server` 进程, 在进入`gen_server`的循环之前执行如下逻辑:

监听进程调用 `proc_lib:start_link` 创建一个进程`my_protocol`, `my_protocol`执行 `init` 函数, 并调用 `proc_lib:init_ack` 告诉监听进程, 进程`my_protocol`已经启动完成, 此时监听进程就可以返回了, 然后进程`my_protocol`再执行`accept_ack`函数, 最后调用`gen_server:enter_loop`进入`gen_server`的循环.

```erlang
-module(my_protocol).
-behaviour(gen_server).
-behaviour(ranch_protocol).
-export([start_link/4]).
-export([init/4]).
%% Exports of other gen_server callbacks here.
start_link(Ref, Socket, Transport, Opts) ->
    proc_lib:start_link(?MODULE, init, [Ref, Socket, Transport, Opts]).

init(Ref, Socket, Transport, _Opts = []) ->
    ok = proc_lib:init_ack({ok, self()}),
    %% Perform any required state initialization here.
    ok = ranch:accept_ack(Ref),
    ok = Transport:setopts(Socket, [{active, once}]),
    gen_server:enter_loop(?MODULE, [], {state, Socket, Transport}).
%% Other gen_server callbacks here.
```

#### 2. 触发超时

> 这种方式可能产生竞态条件不推荐使用.

第二种方法是在`gen_server:init`结束的时候立即触发超时, 如果返回超时值为`0`, 那么`gen_server`会立即调用`handle_info(timeout, _, _)`

```erlang
module(my_protocol).
-behaviour(gen_server).
-behaviour(ranch_protocol).

init([Ref, Socket, Transport]) ->
    {ok, {state, Ref, Socket, Transport}, 0}.
handle_info(timeout, State={state, Ref, Socket, Transport}) ->
    ok = ranch:accept_ack(Ref),
    ok = Transport:setopts(Socket, [{active, once}]),
    {noreply, State};
```

## Elixir 代码实现

项目中有处理断包, 连包的问题, 使用到了`Ranch`网络库. 并用`GenServer`行为实现了回调函数对断包, 连包的处理.

```elixir
require Logger

defmodule ProtocolGenServer do
  use GenServer
  use Types

  @behaviour :ranch_protocol
  @timeout Application.get_env(:server, :protocol, 5000)

  def start_link(ref, socket, transport, opts \\ []) do
    :proc_lib.start_link(__MODULE__, :init, [ref, socket, transport, opts])
  end

  @doc """
  这里是最重要的部分, 不能直接使用`GenServer.start_link/4`, 
  要绕过`GenServer`的默认行为. 否则会进入死循环.
  """
  def init(ref, socket, transport, opts \\ []) do
    add_socket(socket)
    :erlang.process_flag(:trap_exit, true)
    Logger.debug "#{__MODULE__}:init/4 called. options: #{inspect opts}"
    # 通知父进程
    :ok = :proc_lib.init_ack({:ok, self()})
    # 移交套接字控制权
    :ok = :ranch.accept_ack(ref)
    # 主动接收一次,然后切换到被动
    :ok = transport.setopts(socket, [{:active, :once}])
    # 初始化进程状态
    state = %{
      socket: socket,
      transport: transport,
      fragments: [],
      packet_type: 0
    }
    # 进入循环
    :gen_server.enter_loop(__MODULE__, [], state)
  end

  def handle_info({:tcp, _socket, data}, %{
    socket: socket,
    transport: transport,
    packet_type: packet_type,
    fragments: fragments
  } = state) do
    :ok = transport.setopts(socket, [{:active, :once}])
    case packet_type do
      0 ->
        case {is_binary(data), byte_size(data) >= 64} do
          {true, true} ->
            <<_head::binary-size(64), tail::binary>> = data
            {:noreply, get_payload(tail, state)}
          {_, _} ->
            Logger.error "#{__ENV__.file}:#{__ENV__.line} Unsupported packet."
            {:stop, "Unsupported packet", state}
        end
      1 ->
        {:noreply, get_payload(data, state)}
      2 ->
        merged_list = fragments ++ :erlang.binary_to_list(data)
        %{state | fragments: []}
        merged_binary = :erlang.list_to_binary(merged_list)
        {:noreply, get_payload(merged_binary, state)}
    end
  end

  def handle_info({:tcp_closed, _socket}, state) do
    quit()
    Logger.debug "Connection closed."
    # {:stop, state}
    {:stop, :normal, state}
  end

  @doc """
  Invoked when the server is about to exit. It should do any cleanup required.
  """
  def terminate(reason, state) do
    Logger.warn "process exit with reason: #{inspect reason}"
    Logger.warn "process state #{inspect state}"
    :ok
  end

  def get_payload(data, state) do
    {len, bin1} = get_len(data)
    cond do
      byte_size(bin1) == len ->
        received(state.socket, bin1)
        %{state | packet_type: 1}
      byte_size(bin1) >= len ->
        <<full_packet::binary-size(len), fragments::binary>> = data
        received(state.socket, full_packet)
        %{state | fragments: fragments ++ state.fragments, packet_type: 2}
        get_payload(state.fragments, state)
      byte_size(bin1) <= len ->
        %{state | fragments: bin1 ++ state.fragments, packet_type: 2}
      true ->
        state
    end
  end

  def get_len(bin) do
    <<len0::unsigned_int(8), bin1::binary>> = bin
    case len0 < 127 do
      true ->
        {len0 * 4, bin1}
      false ->
        <<len2::unsigned_int(24), bin2::binary>> = bin1
        {len2 * 4, bin2}
    end
  end
end
```

## 补充

代码中使用自定义宏

`types.ex`

```elixir
defmodule Types do
  @moduledoc """
  Macros of data type specifiers
  """
  defmacro unsigned_int(size) do
    quote do
      unsigned-little-integer-size(unquote(size))
    end
  end
  defmacro signed_int(n) do
    quote do
      signed-little-integer-size(unquote(n))
    end
  end
end
```

## 修订

2016-04-10 增加`gen_fsm`的 [gen_fsm协议处理模块的erlang实现][1]

## 参考资料

- https://github.com/ninenines/ranch/issues/137
- http://wudaijun.com/2015/08/erlang-server-design1-cluster-server/
- http://youthyblog.com/2015/09/28/ranch%E7%AC%94%E8%AE%B0/
- http://youthyblog.com/2015/07/31/erlang-question-gen-server-and-init/


  [1]: http://wot123.github.io/erlang/2015/05/12/using-ranch-to-accept-connections.html