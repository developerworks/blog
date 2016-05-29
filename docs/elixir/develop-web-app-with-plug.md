需要一些 `Elixir` 的基础. 

对于没有`Erlang`背景知识的同学, 有比较陡峭的学习曲线. 但是Elixir语言提供了一个库: [Plug](https://github.com/elixir-lang/plug), 用它我们能够开发基于Erlang VM的Web应用.

本文采用 [Cowboy](https://github.com/ninenines/cowboy) 作为Web服务器, 来构造一个超级简单的Web应用. 它唯一的功能就是在浏览器上显示一个纯文本: "Hello World!". `Plug` 不是一个Web框架, 它不是用来替代 `Phoenix` 或 `Sugar` 的, 相反,这两个框架都使用了Plug来与底层的HTTP服务器(Cowboy)交互. 

## 快速上手

通过命令`mix new --sup plug_sample` 创建一个项目.

在`mix.exs`文件中添加依赖

```elixir
defp deps do
  [{:cowboy, "~> 1.0.0"},
   {:plug, "~> 0.12"}]
end
```

安装依赖

```
mix deps.get
```

把`:cowboy`和`:plug`添加到`application`函数

```
def application do
  [applications: [:logger, :cowboy, :plug],
   mod: {PlugSample, []}]
end
```

### 启动`Plug`

在`PlugSample.Worker`启动`Plug`, `PlugSample.Worker`由`PlugSample.start/2`启动

```
defmodule PlugSample.Worker do
  def start_link do
    Plug.Adapters.Cowboy.http PlugSample.MyPlug, []
  end
end

```

### 路由 `Plug.Router`

创建路由模块 `lib/elixir_plug_examples/router.ex`

```elixir
defmodule ElixirPlugExamples.Router do
  use Plug.Router
  if Mix.env == :dev do
    use Plug.Debugger
  end
  plug :match
  plug :dispatch
  # Root path
  get "/" do
    send_resp(conn, 200, "This entire website runs on Elixir plugs!")
  end
end
```

路由模块创建完成后, 就可以通过`iex -S mix`来启动这个简单的Web应用程序了. 访问 [http://localhost:4000/](http://localhost:4000/).

使用 `curl -v http://localhost:4000` 连接到服务器, `-v` 选项让我们可以看到响应头信息.

响应头如下:

```
> GET / HTTP/1.1
> Host: localhost:4000
> User-Agent: curl/7.43.0
> Accept: */*
> 
< HTTP/1.1 200 OK
< server: Cowboy
< date: Tue, 12 Apr 2016 08:25:38 GMT
< content-length: 11
< cache-control: max-age=0, private, must-revalidate
< content-type: text/plain; charset=utf-8
< 
Hello world
```

### 完整的代码

本文网站的示例代码

https://github.com/developerworks/plug_sample

## Plug 是什么?

- `Plug` 是一段代码片段, 它一般插入到请求处理链条中的一个特定位置.
- 多个`Plug`构成一个完整的处理链条
- 请求首先被转换成`%Plug.Conn{}`结构, 并传递个链条中的第一个`Plug`, 第一个`Plug`处理(修改,增加,删除)完成后, 传递给后续的`Plug`

![plug.png][1]


## 两种类型的`Plug`

- 函数的`Plug`
  函数`Plug`是一个接受`%Plug.Conn`和一个选项列表作为参数, 并返回一个`%Plug.Conn`的`函数`, 例如：
    ```elixir
    def my_plug(conn, opts) do
      conn
    end
    ```

- 模块的`Plug`
  模块`Plug`是一个实现了`init/1`, 和`run/2`函数的模块:
    ```elixir
    module MyPlug do
      def init(opts) do
        opts
      end
    
      def call(conn, opts) do
        conn
      end
    end
    ```

模块`Plug`有一个特点是: `init/1`在编译时运行, `run/2`在运行时运行. 由`init/1`返回的值会被传递给`run/2`. 因此尽量吧繁重的工作安排到`init/1`去执行对Plug的性能有非常大的提升.

在Phoenix中, 可以使用管道`pipeline`把多个`Plug`组合到一起. 

比如:

针对`浏览器`的管道(输出`HTML`):

```elixir
pipeline :browser do
  plug :accepts, ["html"]
  plug :fetch_session
  plug :fetch_flash
  plug :protect_from_forgery
  plug :put_secure_browser_headers
end
```

针对`API接口`的管道(输出`JSON`)

```elixir
pipeline :api do
  plug :accepts, ["json"]
end
```

  [1]: /img/bVu7eA

## Plug 的测试

`Plug`提供了一个`Plug.Test`模块简化`Plug`的测试, 下面是一个例子

```elixir
defmodule MyPlugTest do
  use ExUnit.Case, async: true
  use Plug.Test
  @opts AppRouter.init([])
  test "returns hello world" do
    # Create a test connection
    conn = conn(:get, "/hello")
    # Invoke the plug
    conn = AppRouter.call(conn, @opts)
    # Assert the response and status
    assert conn.state == :sent
    assert conn.status == 200
    assert conn.resp_body == "world"
  end
end
```

## 可用的`Plugs`

| Plug Types | Description |
| --------- | ----------- |
| `Plug.CSRFProtection` | 添加跨站点请求保护, 如果要使用`Plug.Session`,<br> 这个是必须的;
| `Plug.Head` | 把`HEAD`请求转换为`GET`请求;
| `Plug.Logger` | 记录请求;
| `Plug.MethodOverride` | 重写指定在请求头中指定的方法;
| `Plug.Parsers` | 根据`Content-Type`负责解析请求体;
| `Plug.RequestId` | 设置一个请求ID, 用在日志中;
| `Plug.Session` | 处理会话管理和存储;
| `Plug.SSL` | 强制请求通过SSL;
| `Plug.Static` | 处理静态文件;
| `Plug.Debugger` | 调试页面
| `Plug.ErrorHandler` | 允许开发者定制错误页面, 而不是发送一个空页面.


