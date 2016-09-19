> Rustler 项目还不是很成熟, 基本可用. 有兴趣的可以给作者提 Issue.

Rustler 是一个在安全的用 Rust 编写 Erlang NIF 的库. 这里安全的含义是, 它不会导致 BEAM(Erlang 虚拟机)的崩溃. 该库提供了一个设施用于生成与BEAM交互的模板, 处理Erlang Term的编码和解码. Rustler 适用于 Erlang 和 Elixir, Elixir 是首选的.

## 功能

- 安全性 - 用 Rust 编写的NIF绝不会导致BEAM崩溃.
- 互操作性 - 编码和解码Rust值到Erlang Term就像调用函数一样容易.
- 复合类型 - 可以用属性指定某个 Rust 结构为 Encodable, Decodable 
- 资源对象 - 可以安全地传递Rust结构到Erlang代码, 该结构不再被引用时自动删除.

## 入门

最简单的入门方法是, 使用Mix项目生成器

运行下面的命令安装Rustler的Mix项目生成器
```
mix archive.install https://github.com/hansihe/rustler_archives/raw/master/rustler_installer.ez
```
运行下面的命令创建一个rustler项目
```
mix rustler.new
```

需要安装 Nightly 版本的 Rust. 首先安装 [rustup](https://www.rustup.rs/). 并在项目目录下执行 `rustup override add nightly-2016-05-07-x86_64-apple-darwin`.

## Rust NIF

下面是一个最小的Rust NIF实现, 它只是简单的实现了一个加法函数返回两个数字的和.

```rust
#![feature(plugin)]
#![plugin(rustler_codegen)]

#![feature(link_args)]
#[link_args = "-flat_namespace -undefined suppress"]
extern {}

#[macro_use]
extern crate rustler;
use rustler::{ NifEnv, NifTerm, NifResult, NifEncoder };

rustler_export_nifs!(
    "Elixir.Test",
    [("add", 2, add)],
    None
);

fn add<'a>(env: &'a NifEnv, args: &Vec<NifTerm>) -> NifResult<NifTerm<'a>> {
    let num1: i64 = try!(args[0].decode());
    let num2: i64 = try!(args[1].decode());
    Ok((num1 + num2).encode(env))
}
```

OSX下面的编译问题参考 https://github.com/hansihe/Rustler/issues/16#issuecomment-224259032

## 与 Mix 项目集成

对于使用Mix的项目, 在Hex上有一个[助手包](https://hex.pm/packages/rustler). 这个包包含一个Mix编译器, 进行自动环境检查, crate 编译和nif加载. 如果使用mix,极其推荐使用这个包, 它使你基于Rust开发Erlang NIF更加方便.

要启动用Rust NIF的自动编译, 完成如下步骤:

- 添加 [rustler](https://hex.pm/packages/rustler) 到 mix.exs 依赖
- 添加 :rustler 编译器到项目的 `compilers` 列表. `compilers: [:rustler] ++ Mix.compilers`
- 添加一个 `rustler_crates: ["CRATE_PATH"]`到 `project` 函数, `CRATE_PATH` 应该是一个到包含 `Cargo.toml`文件的相对于mix项目更目录的相对路径.

完成后 `mix.exs` 文件看起来像这样:

```elixir
defmodule YourProject.Mixfile do
  use Mix.Project
 
  def project do
    [app: :your_project,
     [...]
     compilers: [:rustler] ++ Mix.compilers,
     rustler_crates: ["."],
     deps: deps]
  end

  [...]
 
  defp deps do
    [{:rustler, "~> 0.0.7"}]
  end
end
```

然后可以通过 `mix compile`编译项目, 如果环境设置有任何问题, rustler编译器插件应该能够提示相关的说明如何解决.

## 加载NIF
加载一个Rust NIF和普通的NIF没什么区别. 实际的加载是通过调用 `Rustler.load_nif(<LIBRARY_NAME>)`完成的, 通常在Elixir用`@on_load`钩子来实现.

```elixir
defmodule MyProject.NativeModule do
  @on_load :load_nif

  defp load_nif do
    :ok = Rustler.load_nif("<LIBRARY_NAME>")
  end

  // When loading a NIF module, dummy clauses for all NIF functions are required.
  // NIF dummies usually just error out when called when the NIF is not loaded, as that should
  // never normally happen.
  def my_native_function(_arg1, _arg2), do: exit(:nif_not_loaded)
end
```

注意 `<LIBRARY_NAME>` 是`Cargo.toml`文件中`[lib]`中的库名字.

## 关于Rustler 的依赖

打开 `Rustler` 的 `Cargo.toml` 文件我们看到下面的代码

```
[dependencies]
ruster_unsafe = ">=0.4"
libc = ">=0.1"
lazy_static = "0.1.*"
```

它依赖 `ruster_unsafe` 了去实现用 Rust 开发 NIF, `ruster_unsafe` 是一个底层的Erlang NIF的语言绑定, Rustler 只是一个集成到 Elixir 的助手工具.
