当我们学习一门新的语言时, 要查看一个变量的值, 一般是通过该语言提供的标准输出函数把变量的值打印在终端上查看. Elixir 和其他语言相比有不同的地方, 除了提供了 `IO.puts/2`, 以及 `IO.inspect/2` 标准的控制台输出函数输出变量的值. 还提供了其他调试方法. 

本文介绍了Elixir调试的三种工具, 分别为`IEx.pry/0`, `:debugger`和 `Erlyberly`客户端图形工具

## `IEx.pry`

`pry` 英文解释为: `刺探;侦察;窥探`, 不言而喻, 其作用是查看程序运行的内部状态, 在 Elixir 中它的基本作用是: 查看`IEx.pry`插入点位置所在的作用域的内部状态.

要使用`IEx.pry`, 需要在全局作用域或模块中`require`

```elixir
require IEx
defmodule Example
    def double_sum(x, y) do
        hard_work(x, y)
        IEx.pry  %% 插入点
    end
    def hard_work(x, y) do
        x = 2 * x
        y = 2 * y
        
        x + y
    end
end
```

输入`respawn`可退出`pry`, 另外, 如果调试代码有多个`IEx.pry`插入点`respawn`后会自动进入下一个. 如果`IEx.pry`插入点位于递归代码中, `respawn`会进入下一次递归, 直到`递归终止条件`满足为止

![图片描述][1]

## `:debugger`

如果你需要断点功能, 我们可以使用Erlang提供的`:debugger`模块, 现在来演示一下:

```elixir
defmodule Example
    def double_sum(x, y) do
        hard_work(x, y)
    end
    def hard_work(x, y) do
        x = 2 * x
        y = 2 * y
        
        x + y
    end
end
```

现在我们启动`:debugger`

```
$ iex -S mix
Erlang/OTP 18 [erts-7.3] [source] [64-bit] [smp:4:4] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Compiled lib/example.ex
Interactive Elixir (1.2.4) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> :debugger.start()
{:ok, #PID<0.87.0>}
iex(2)> :int.ni(Example)
{:module, Example}
iex(3)> :int.break(Example, 3)
:ok
iex(4)> Example.double_sum(1,2)
```

当你启动了`:debugger`, 一个图形界面同时会打开.

`:int.break(Example, 3)` 在第三行添加了一个断点, 在打开的图形界面中可以向其他断点调试工具类似的方法使用

## 问题

```
iex(2)> :int.ni(Example)
** Invalid beam file or no abstract code: 'Elixir.Example'
```

OTP 18还么有这个功能, 要等到 OTP 19

解决办法:

下载:`int.erl` https://raw.githubusercontent.com/josevalim/otp/c7e82c6b406b632a191c791a1bd2162bde08f692/lib/debugger/src/int.erl
编译: `erlc -o . int.erl`
覆盖: `lib/debugger/ebin/int.beam`

详细的说明参考 Erlang文档 [Debugger](http://erlang.org/doc/apps/debugger/debugger_chapter.html)

## `Erlyberly`

Erlyberly 的好处在于不会中断进程的执行, 在产品和开发环境中都可以使用, 在启动的时候要求你指定目标节点的名称和Cookie.

### 命名节点

要让`Erlyberly`能够工作, 需要指定`--name`或`--sname`参数. `Cookie`用于验证和连接节点. `Cookie`的值可以通过`Node.get_cookie/0`, 或 `~/.erlang.cookie`

使用`~/.erlang.cookie`文件中的`Cookie`值

```
iex --name "foo@127.0.0.1" -S mix
```

命令行指定`Cookie`

```
iex --name "foo@127.0.0.1" --cookie "my_cookie" -S mix
```

### 进程

![图片描述][2]

### 内存

内存图基于`:erlang.memory()`提供的信息, 表示在Erlang VM中内存使用情况. 并非操作系统相关的内存.


![图片描述][3]

### 模块和跟踪

![图片描述][4]

通过追踪函数可以看到函数的`参数值`, `返回值`

![图片描述][5]

关于 `Erlyberly` 的详细资料, 参考 [Erlyberly 项目主页](https://github.com/andytill/erlyberly)

## 补充

`:observer.start()`也是一个不错的工具, 可以追踪进程的状态, 进程之间的连接关系等. 

## 结语

每种工具提供不同的调试场景, 请按自己的需要选择. 本文旨在抛砖引玉, 细节如何操作, 请自己探索, 在这里就不多说了.

## 参考资料

[Debugging techniques in Elixir](http://blog.plataformatec.com.br/2016/04/debugging-techniques-in-elixir-lang)
[How to trace Elixir nodes with Erlyberly](http://blog.plataformatec.com.br/2016/04/how-to-trace-elixir-nodes-with-erlyberly)


  [1]: https://segmentfault.com/img/bVuZ0X
  [2]: https://segmentfault.com/img/bVuZYm
  [3]: https://segmentfault.com/img/bVuZYo
  [4]: https://segmentfault.com/img/bVuZ1U
  [5]: https://segmentfault.com/img/bVu0cd