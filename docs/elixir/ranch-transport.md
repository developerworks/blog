## 传输

> 传输(Transports) 定义了和套接字交互的接口

Ranch 的传输层抽象了两个协议处理模块出来, 一个是用于普通的TCP传输层套接字 `ranch_tcp`, 另一个是SSL加密传输层套接字处理模块 `ranch_ssl`

传输可用于连接(`connecting`), 监听(`listenig`), 以及接受(`accepting`)连接, 也可以接收(`receiving`)和发送(`sending`)数据.

支持主动(`active`)和被动(`passive`)模式

## TCP 传输

TCP 传输是一个 `gen_tcp` 的轻量级封装.

## SSL 传输 

SSL 传输是一个 `ssl` 的轻量级封装. 它依赖于 Erlang 提供的 `crypto`,`asn1`,`public_key`和`ssl` 应用程序, 因此必须在 Ranch 之前启动. 当启动一个 SSL 监听器的时候, Ranch 会自动的启动他们. 监听器删除的时候, 不会停止他们,

启动 SSL 应用

```erlang
ssl:start().
```

在一个正确的 OTP 设置中, 你需要保证你的应用程序依赖 `crypto`, `public_key`和 `ssl`应用程序. 当启动你的 release 时, 他们会被自动启动.

SSL 传输的 `accept/2` 函数同时执行传输(`transport`)和SSL 接受(`SSL accepts`)的操作. 在SSL接受阶段如果发生错误, 将返回 `{error, {ssl_accept, atom()}}` 以区分问题发生在哪个套接字上.

## 发送和接收数据

可以通过 `Transport:send/2` 函数给套接字发送数据. 数据的类型为 `iodata()`, 它包含两种子类型 `binary() | iolist()`

通过套接字发送数据

```erlang
Transport:send(Socket, <<"Ranch is cool!">>).
Transport:send(Socket, "Ranch is cool!").
Transport:send(Socket, ["Ranch", ["is", "cool!"]]).
Transport:send(Socket, ["Ranch", [<<"is">>, "cool!"]]).
```

可以通过`被动模式`或`主动模式`接收数据. 被动模式执行阻塞的 `Transport:recv/3` 调用. 主动模式把数据作为消息接收.

默认所有数据是作为二进制接收的. 也可以把接收的数据当成字符串. 

用被动模式接收数据要求一个函数调用. 第一个参数为套接字, 第三个参数为读取超时值(超时后返回`{error, timeout}`. 

第二个参数为想要接收的数据字节数. 该函数将会等待数据, 直到它接收到了指定长度的数据. 如果不指定精确的值, 也可以指定为0 , 它会尽快的返回, 而不管数据的大小.

被动模式下,从套接字读取数据

```erlang
{ok, Data} = Transport:recv(Socket, 0, 5000).
```

主动模式要求你告知套接字你想要把数据作为消息接收, 并且编写代码来接收消息.

主动模式有两种类型 `{active, once}` 和 `{active, true}`, 前者发送一个消息后就立即回到被动模式. 后者无限地发送消息. 不推荐使用 `{active, true}` 模式, 这种模式会很快的把进程邮箱塞满. 更好的是, 把数据留在套接字中, 仅当需要的时候再读取.

可以接收三种不懂的消息

- `{OK, Socket, Data}`
- `{Closed, Socket}`
- `{Error, Socket, Reason}`

取决于选择的传输模块, `OK`, `Closed`, 和 `Error` 的值有可能不同. 要能够正确的匹配他们, 必须首先调用 `Transport:messages/0`函数.

获取传输的主动消息标识

```erlang
{OK, Closed, Error} = Transport:messages().
```

要开始接收消息, 你需要调用 `Transport:setopts/2` 函数, 并且每次再接收消息的时候都要这样做.

主动模式下, 从套接字接收消息

```erlang
{OK, Closed, Error} = Transport:messages(),
Transport:setopts(Socket, [{active, once}]),
receive 
    {OK, Socket, Data} ->
        io:format("data received: ~p~n", [Data]);
    {Closed, Socket} ->
        io:format("socket got closed!~n");
    {Error, Socket, Reason} ->
        io:format("error happended: ~p~n", [Reason])
end.
```

可以很容易的把主动套接字集成到 Erlang 代码中, 当接收消息时, 仅仅需要不多的代码.

## 发送文件

通过套接字发送名为 `Filename` 的文件:

通过文件名发送文件

```erlang
{OK, SentBytes} = Transport:sendfile(Socket, Filename).
```

如果是文件的一部分, 使用大于等于0的`Offset`, `Bytes` 字节数, 以及 `ChunkSize` 块大小:

发送文件的一部分块

```erlang
Opts = [{chunk_size, ChunkSize}]
{ok, SentBytes} = Transport:sendfile(Socket, Filename, Offset, Bytes, Opts).
```

要改善发送同一个文件的多个部分, 可以使用以`raw`模式打开的文件描述符:

发送一个以 raw 模式打开的文件

```erlang
{ok, RawFile} = file:open(Filename, [raw, read, binary]),
{ok, SentBytes} = Transport:sendfile(Socket, RawFile, Offset, Bytes, Opts).
```

## 编写传输处理模块


传输处理模块是一个实现了 `ranch_transport` 行为的模块(比如(`ranch_tcp`, `ranch_ssl`). 

> `ranch_tcp.erl`文件头两行说明了这个事实

```erlang
-module(ranch_tcp).
-behaviour(ranch_transport).
```

为了能够透明的使用传输处理模块, 需要行为中定义的一系列回调函数.

当打开套接字时, 该行为没有定义可用的套接字选项. 因为对于使用不同的传输, 编写不同的初始化函数是相当容易的, 因此不需要有共同的传输选项设置, 一个例外是, `setopts/2` **必须** 实现 `{active, once}` 和 `{active, true}` 选项

如果传输没有实现 `sendfile/5`, 将使用 `ranch_transport:sendfile/6` 替代. 多的第一个参数是传输模块. 例子可以参考 `ranch_ssl` 模块.