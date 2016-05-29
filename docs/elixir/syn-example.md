文章 [Elixir: Syn,一个分布式进程注册表和进程组管理器](https://segmentfault.com/a/1190000005093730) 阐述了如何使用 `Mnesia` 的`仅内存`表来运行分布式系统全局进程注册简介, 本文用一个实际的例子演示如何使用 `Syn` 进行分布式全局进程注册和查询

本文以一个机遇OpenCV的图形处理项目为例子演示怎么进行全局进程注册和查询.
https://github.com/developerworks/opencv_thumbnail_server

首先启动两个节点并连接

```
iex --name node1@192.168.8.104 -S mix
iex --name node2@192.168.8.104 -S mix
```

在 `node2@192.168.8.104` 终端中输入:

```
Node.connect :"node1@192.168.8.104"
```

运行 `:syn.start` 启动 `syn` 应用程序, 执行 `:syn.init` 初始化分布式 Mnesia 表

![图片描述][1]

查看 `Mnesia` 数据库的信息

![图片描述][2]

其中 `running db nodes` 指出了 `Mnesia` 集群包含那些节点, `ram_copies` 指出了, 哪些表分布在那些节点上, 这里为 `schema`, `syn_registry_table`, 和 `syn_groups_table`

在 `node1` 上注册进程, 在 `node2` 上查询:

```
iex(node1@192.168.8.104)2> worker = :poolboy.checkout(:opencv_thumbnail_server_pool)
iex(node1@192.168.8.104)3> :syn.register "process_1234567890", worker
:ok
iex(node1@192.168.8.104)4> :syn.find_by_key "process_1234567890"
#PID<0.293.0>
```

切换到 `node2` 执行查询

```
iex(node2@192.168.8.104)9> :syn.find_by_key "process_1234567890"
#PID<16898.293.0>
```

返回的 `#PID<16898.293.0>` 说明了这个进程是一个非本地进程.

把一个进程添加到一个组

```
iex(node2@192.168.8.104)11> pid = :syn.find_by_key "process_1234567890"
#PID<16898.293.0>
iex(node2@192.168.8.104)12> :syn.join "group1", pid                    
:ok
```

获取组 `group1` 的所有进程

```
iex(node2@192.168.8.104)13> :syn.get_members "group1"
[#PID<16898.293.0>]
```

从组中删除一个进程

```
iex(node2@192.168.8.104)14> :syn.leave "group1", pid
:ok
iex(node2@192.168.8.104)15> :syn.get_members "group1"
[]
```

  [1]: https://segmentfault.com/img/bVxwCc
  [2]: https://segmentfault.com/img/bVxwCu