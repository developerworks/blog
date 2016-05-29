## Map 的嵌套匹配

`添加时间: 2016-04-30`

```elixir
def background_check(%{manager: employee} = company) do
  %{name: full_name} = employee
  from(c in Criminals, where: c.full_name == ^full_name, select: c)
  |> Repo.one
  |> case do
    nil -> congratulate(employee)
    criminal -> notify(company)
  end
end
```

- 对参数 `company` 进行匹配, 析构出其中的 `manager` 字段
- 对 `manager` 再次析构出其中的 `name` 字段
- 然后查询 `Criminals` 表进行对比

在单个表达式中一次匹配

```elixir
def check_company(%{manager: %{name: full_name} = employee} = company) do
  from(c in Criminals, where: c.full_name == ^full_name, select: c)
  |> Repo.one()
  |> case do
    nil -> congratulate(employee)
    criminal -> notify(company)
  end
end
```

## Elixir单元测试超时

`添加时间: 2016-04-18`

```
** (ExUnit.TimeoutError) test timed out after 60000ms. You can change the timeout:
     
       1. per test by setting "@tag timeout: x"
       2. per case by setting "@moduletag timeout: x"
       3. globally via "ExUnit.start(timeout: x)" configuration
       4. or set it to infinity per run by calling "mix test --trace"
          (useful when using IEx.pry)
     
     Timeouts are given as integers in milliseconds.
```

上面是一个超时的错误提示, 通过上面的描述, 我们可以很清楚的了解如何设置单元测试的超时, 对于需要长时间运行的测试, 请把`timeout`设置的更大一点.

上面的说明标号分别为

- 单独设置一个测试`@tag timeout: x`
- 整个测试用例`@moduletag timeout: x`
- 全局设置`ExUnit.start(timeout: x)`
- 第四个不常见, 有空再研究一下.

## 获取和替换进程状态

`添加时间: 2016-04-18`

进程状态可以在任何时候获取和替换, 一般用于热更的时候. 

```elixir
# 创建一个GenServer模块
iex(1)> defmodule Test do
...(1)>   use GenServer
...(1)> end
{:module, Test,
 <<70, 79, 82, 49, 0, 0, 10, 40, 66, 69, 65, 77, 69,
120, 68, 99, 0, 0, 2, 60, 131, 104, 2, 100, 0, 14, 
101, 108, 105, 120, 105, 114, 95, 100, 111, 99, 115, 
95, 118, 49, 108, 0, 0, 0, 4, 104, 2, ...>>,
:ok} 

# 启动进程 
iex(2)> GenServer.start_link Test, %{name: "developerworks"}, name: :server
{:ok, #PID<0.72.0>}

# 获取进程状态   
iex(3)> :sys.get_state :server
%{name: "developerworks"}

# 替换状态
iex(4)> :sys.replace_state :server, fn(state)-> %{name: "new_state"} end      
%{name: "new_state"}

# 新状态
iex(5)> :sys.get_state :server
%{name: "new_state"}
```

[http://www.hoterran.info/erlang-otp-sys-sourcecode](http://www.hoterran.info/erlang-otp-sys-sourcecode)
[http://stackoverflow.com/questions/1840717/achieving-code-swapping-in-erlangs-gen-server](http://stackoverflow.com/questions/1840717/achieving-code-swapping-in-erlangs-gen-server)

## 布尔运算需要注意的地方

`||`,`&&`和`!` 两边支持任何数据类型
`and`, `or`, `not` 第一个参数必须为布尔类型

## 进制数的表示

`添加时间: 2016-04-06`

二进制: 前缀`0b`, 例如:

```
0b100000000 # 256
```

八进制: 前缀`0o`, 例如:

```
0o400 # 256
```

十六进制: 前缀`0x`, 例如:

```
0xFF # 255
```

## 关键字列表转换为Map

`添加时间: 2016-04-05`

```elixir
keywords = [a: 1, b: 2]
Enum.into(keywords, %{})
```

```shell
iex(1)> keywords = [a: 1, b: 2] 
[a: 1, b: 2]
iex(2)> Enum.into(keywords, %{})
%{a: 1, b: 2}
iex(3)> map = Enum.into(keywords, %{})
%{a: 1, b: 2}
iex(4)> map[:a]
1
iex(5)> map.a  
1

# 动态访问
iex(6)> map[:c]
nil
# 严格语法
iex(7)> map.c  
** (KeyError) key :c not found in: %{a: 1, b: 2}
```

如果Key不存在, 严格语法抛出`KeyError`异常, 动态访问方式返回`nil`, 可以参考`Jose Valim`的这篇文章[Writing assertive code with Elixir](http://blog.plataformatec.com.br/2014/09/writing-assertive-code-with-elixir)

## .ex用于编译, .exs用于解释.

`添加时间: 2016-04-05`

对于ExUnit测试, 用`.exs`文件, 因此不需要在每次修改的时候重新编译. 如果编写脚本或测试, 使用`.exs`文件, 否则使用`.ex`后缀.

解释代码需要消耗更多的时间来执行(解析, tokenize等), 但不要求编译. 如果灵活性比执行速度更重要, 使用`.exs`后缀, 否则使用`.ex`.

## IEx的历史记录

`添加时间: 2016-04-05`

Elixir 的IEx默认是不带历史命令支持的, 退出Erlang VM后,之前的命令全部么有了, 要跨多个IEx Session保存命令历史, 可以使用下面这个开源的工具:

```shell
git clone https://github.com/ferd/erlang-history.git
cd erlang-history
make install
```

安装完成后启动IEx就可以使用了.

这里还有一篇文章 [Erlang 和 Elixir shell 历史记录设置](http://blog.fnil.net/blog/erlang-he-elixir-shell-li-shi-ji-lu-she-zhi)

## 通过结构来创建新的类型

`添加时间: 2016-04-02`

创建一个工具模块包括常用的辅助函数

```elixir
defmodule Util do
  @mdoc """
  工具模块
  """

  @doc """
  用于获取UNIX时间戳
  """
  def get_unixtime() do
    {megas, secs, _micros} = :os.timestamp
    megas * 1000 * 1000 + secs
  end
end
```

定义一个`User`类型模块

```
defmodule User do
  @mdoc """
  用户类型
  """

  defstruct username: "", tel: "", email: "", created_at: Util.get_unixtime
  @type t :: %__MODULE__{
    username: String.t,
    tel: String.t,
    email: String.t,
    created_at: Integer.t
  }
end
```

测试代码

```
# 测试
require Logger
defmodule UserTest do
  def main() do
    # 创建一个新的User结构类型
    user = %User{}
    # 打印
    Logger.info "#{inspect user}"
  end
end

# 调用
UserTest.main
```

## 编译选项

打开 `warnings_as_errors` 编译器选项, 让你的代码没有警告, 在`config.exs`中设置编译选项

```elixir
Code.compiler_options([
  debug_info: true,
  docs: true,
  ignore_module_conflict: false,
  warnings_as_errors: true
])
```

小心使用`Enum.map/2`

在集合过大的时候不要直接使用该函数, 请使用 `Stream` 模块, 要获取结果的时候再用 `Enum` 模块中的函数计算最终结果.

[枚举类型和流](http://www.ituring.com.cn/article/123191)

## 二进制速构

http://elixir-lang.org/docs/stable/elixir/Kernel.SpecialForms.html#for/1

```
pixels = <<213, 45, 132, 64, 76, 32, 76, 0, 0, 234, 32, 15>>
rgb = for <<r::8, g::8, b::8 <- pixels >> do
  {r, g, b}
end
IO.inspect rgb
```

## 字符过滤

```
for <<c <- " hello world ">>, c != ?\s, into: "", do: <<c>>
"helloworld"
```

## 函数的`动态定义`

通过函数名称列表, 动态的创建函数的定义. 

这种技巧通常用于批量的定义类似的函数`一般参数数量相同, 函数名称有规律的重复, 多用于开发API模块`

```elixir
defmodule Reader do
  [:doc, :voice, :video] |> Enum.each(fn(name)->
    def unquote(:"read_#{name}")(id) do
      IO.inspect("Call function #{unquote(name)}(#{id})")
    end
  end)
end

Reader.read_doc 1     # Call function doc(1)
Reader.read_video 2   # Call function doc(2)
Reader.read_voice 3   # Call function doc(3)
```

## 函数的`动态调用`

使用`apply/3`动态调用模块

```elixir
defmodule RiakPoolEx.Handlers.MediaHandler do
  @mdoc """
  You can call functions in the module liks this:

  Exmaples:

    alias RiakPoolEx.QueryHandler.MediaHandler
    MediaHandler.read_doc(1)
    MediaHandler.read_voice(2)
    MediaHandler.read_video(3)
    MediaHandler.read_picture(4)
  """
  alias RiakPoolEx.QueryHandler
  [:doc, :voice, :video, :picture] |> Enum.each(fn(name)->
    def unquote(:"read_#{name}")(id) do
      apply(RiakPoolEx.QueryHandler, unquote(:"read_#{name}", [id]))
    end
  end)
end
```