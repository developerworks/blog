![HdC9YlO.png][1]

## 创建 Elixir 项目

```
$ mix new simple_statistics
$ cd simple_statistics
$ mix test
```

Mix 生成了如下目录结构

```
|-- _build
|-- config/
  |-- config.exs
|-- lib/
  |-- simple_statistics.ex
|-- test/
  |-- simple_statistics_test.exs
  |-- test_helper.exs
|-- mix.exs
|-- mix.lock
|-- README.md
|-- .gitignore
```

## 开发代码

在 `lib` 下创建 `simple_statistics` 子目录, 和包名称一样, 用于存放其他模块.

```
|-- lib/
  |-- simple_statistics/
    |-- mean.ex
  |-- simple_statistics.ex
```

```elixir
# lib/simple_statistics/mean.ex
defmodule SimpleStatistics.Mean do
  def mean([]), do: nil
  def mean(list) do
    Enum.sum(list) / Kernel.length(list)
  end
end
```

## 编写文档

使用 `@moduledoc` 和 `@doc` 编写函数和模块的文档. 应该在模块中编写每一个主要函数的文档.

```elixir
# lib/simple_statistics/mean.ex
defmodule SimpleStatistics.Mean do
  @moduledoc false

  @doc ~S"""
  The mean is the sum of all values over the number of values.
  """
  def mean([]), do: nil
  def mean(list) do
    Enum.sum(list) / Kernel.length(list)
  end
end
```

## 添加文档测试

```elixir
# lib/simple_statistics/mean.ex
defmodule SimpleStatistics.Mean do
  @moduledoc false

  @doc ~S"""
  The mean is the sum of all values over the number of values.

  ## Examples

      iex> SimpleStatistics.Mean.mean([])
      nil
      iex> SimpleStatistics.Mean.mean([1,2,3,4,5])
      3.0
      iex> SimpleStatistics.Mean.mean([1.5,-2.1,3,4.5,5])
      2.38

  """
  def mean([]), do: nil
  def mean(list) do
    Enum.sum(list) / Kernel.length(list)
  end
end
```

把 `doctest` 添加到测试集

```elixir
# test/simple_statistics_test.ex
defmodule SimpleStatisticsTest do
  use ExUnit.Case
  doctest SimpleStatistics.Mean
end
```

运行测试  `mix test`

```
$ mix test
.

Finished in 0.07 seconds (0.07s on load, 0.00s on tests)
1 test, 0 failures
```

## 添加类型注解

类型注解可以使用 `dialyzer` 进行静态分析. 

```elixir
@spec mean(nonempty_list(number)) :: float()
def mean(list) do
  Enum.sum(list) / Kernel.length(list)
end
```

更新 `mix.exs` 文件, 添加依赖:

```elixir
defp deps do
  [{:ex_doc, "~> 0.11", only: :dev},
   {:earmark, "~> 0.1", only: :dev},
   {:dialyxir, "~> 0.3", only: [:dev]}]
end
```

运行 `dialyzer` 静态分析工具:

```
$ mix dialyzer
Starting Dialyzer
dialyzer --no_check_plt --plt /Users/yosriady/.dialyxir_core_18_1.2.0.plt -Wunmatched_returns -Werror_handling -Wrace_conditions -Wunderspecs /Users/yosriady/simple_statistics/_build/dev/lib/simple_statistics/ebin
  Proceeding with analysis... done in 0m1.68s
done (passed successfully)
```

## 生成文档

添加如下依赖到 `mix.exs` 文件

```elixir
defp deps do
  [{:ex_doc, "~> 0.11", only: :dev},
   {:earmark, "~> 0.1", only: :dev}]
end
```

运行 `mix deps.get` 和 `mix deps.compile` 获取和安装这些依赖, 最后用 `mix docs` 生成文档:

```
$ mix docs
$ cd docs
$ open index.html
```

![生成ExDoc文档][2]

## 发布库

现在这边把开发好的库发布到Hex, 以方便自己和别人使用.

### 注册Hex用户账号

```
$ mix hex.user register
```

更具提示注册即可. 稍后会受到激活链接, 点击激活, 就可以登陆了. 

### 设置项目元数据

```elixir
# mix.exs
def project do
  [app: :decision_tree,
   version: "0.0.1",
   elixir: "~> 1.2",
   build_embedded: Mix.env == :prod,
   start_permanent: Mix.env == :prod,
   description: description,
   package: package,
   deps: deps]
end

defp description do
  """
  A few sentences (a paragraph) describing the project.
  """
end

defp package do
  [
   files: ["lib", "mix.exs", "README.md"],
   maintainers: ["Yos Riady"],
   licenses: ["Apache 2.0"],
   links: %{"GitHub" => "https://github.com/Leventhan/simple_statistics",
            "Docs" => "http://hexdocs.pm/simple_statistics/"}
   ]
end
```

元数据和依赖添加到 `mix.exs` 文件后, 我们就准备好发布这个包了, 使用 `mix hex.publish` 发布:

```
$ mix hex.publish
Publishing simple_statistics 0.0.1
  Dependencies:
  Files:
    lib/simple_statistics.ex
    lib/simple_statistics/mean.ex
    mix.exs
    README.md
  App: simple_statistics
  Name: simple_statistics
  Description: Statistics toolkit for Elixir.
  Version: 0.0.1
  Build tools: mix
  Licenses: Apache 2.0
  Maintainers: Yos Riady
  Links:
    Docs: http://hexdocs.pm/simple_statistics/
    GitHub: https://github.com/Leventhan/simple_statistics
  Elixir: ~> 1.2
  WARNING! Excluded dependencies (not part of the Hex package):
    ex_doc
    earmark
    dialyxir
Before publishing, please read Hex Code of Conduct: https://hex.pm/docs/codeofconduct
[#########################] 100%
Published at https://hex.pm/packages/simple_statistics/0.0.1
Don't forget to upload your documentation with `mix hex.docs`
```

现在我们的库就已经发布为 `mix.exs` 中指定的版本了. 

> 在一个小时内, 一个发布版本可以通过 `--revert` 参数进行修改或退回(取消这个版本的发布). 
> 如果你想回退(revert)超过一小时的发布, 需要联系管理员 

## 发布文档

可以把文档发布到 [Hex Docs](https://hexdocs.pm/), 文档可以用任务 `mix docs` 生成.

```
$ mix hex.docs
Docs successfully generated.
View them at "doc/index.html".
[#########################] 100%
Published docs for simple_statistics 0.0.1
Hosted at https://hexdocs.pm/simple_statistics/0.0.1
```

文档可以通过 `https://hexdocs.pm/simple_statistics/0.0.1` 查看, `https://hexdocs.pm/simple_statistics` 总是重定向到最新的发布版本. 

## 版本化

可以通过修改 `mix.exs` 文件的 `version` 值, 并运行 `mix hex.publish` 发布一个新的版本.

注意, 使用 [Git tags](https://git-scm.com/book/en/v2/Git-Basics-Tagging) 标注版本变更!

```
$ git tag -a v0.0.1 -m "Version 0.0.1"
$ git push origin v0.0.1 
```

## 结语

现在已经成功开发和发布了一个版本! 开发和发布Elixir库是简单和直接的. 现在其他开发者就可以把你的库添加到他们自己项目的依赖了.


  [1]: https://segmentfault.com/img/bVvL1c
  [2]: https://segmentfault.com/img/bVvpYC