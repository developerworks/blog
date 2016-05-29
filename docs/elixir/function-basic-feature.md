## 一级函数(First-Class Functions)

什么是一级函数? 下面的连接是知乎一篇问答给出的解释, 原文链接: 
https://www.zhihu.com/question/27460623/answer/36747015

> - 一级的(`first class`)
> 该等级类型的值可以传给子程序作为参数,可以从子程序里返回,可以赋给变量.大多数程序设计语言里,整型,字符类型等简单类型都是一级的. 
> - 二级的(`second class`)
> 该等级类型的值可以传给子程序作为参数,但是不能从子程序里返回,也不能赋给变量. 
> - 三级的(`third class`)
> 该等级类型的值连作为参数传递也不行.

有了这个基础知识, 下面对于高阶函数,闭包的理解就更容易了.

## 高阶函数(Higher-Order Functions)

Elixir不仅允许你把函数放入变量, 还允许你把函数作为参数传递给另一个函数. 在数学中, 一个高阶函数通常是一个接受一个或多个函数作为输入 `或者` 同时也返回一个函数作为输出的函数. 这个特性使Elixir语言非常强大.

```
iex> square = fn x -> x * x end
#Function<6.17052888 in :erl_eval.expr/5>
iex> Enum.map(1..10, square)
[1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
```

上面定义了一个匿名函数, 它接受一个数字, 求平方值并赋值给 `square` 变量. 然后使用 `Enum.map` 对数字列表 `1..10` 中的每一个数字计算平方值.

> 注: 关于 `1..10` 请参考 [Range](http://elixir-lang.org/docs/stable/elixir/Range.html). 
>
> 高阶函数参考资料:
> - http://chimera.labs.oreilly.com/books/1234000001642/ch08.html
> - Introducting Elixir 第八章
> - 以及网络中各种文档资料, 请自行Google,百度

## 闭包(Closures)

闭包有如下几个特性:

* 可以传递, 作为参数或作为返回值
* 它能记住`当闭包函数创建时作用域中的所有变量`(相当于一个符号表快照), 在闭包函数外部调用此函数时, 闭包函数还能够访问`创建时作用于中的变量`.

```shell
iex(1)> outside_var = 5
iex(2)> print = fn() -> IO.puts(outside_var) end 
iex(3)> outside_var = 6
iex(4)> print.()
5
```

可以看到, 即使修改了 `outside_var` 的值, 但结果任然是 `5`. 

## 不可修改的状态(Immutable )

正是由于这个特性使`Erlang`消除了并发编程中`同一时刻`访问全局变量的`竞太条件`, 这也是`Erlang`作为开发并发系统的首选语言之一.

看下面的例子:

```shell
iex> tuple = {:ok, "hello"}
{:ok, "hello"}
iex> put_elem(tuple, 1, "world")
{:ok, "world"}
iex> tuple
{:ok, "hello"}
```

是的, 你仍能够给一个变量重新赋值. 

```elixir
defmodule Assignment do
    def change_me(string) do
        string = 2
    end
end
```

## 关于`^`(Pin)操作符

正常情况下给变量绑定一个值(或者在其他语言中成为赋值)操作如下:

`name = "developerworks"`

这里的name为一个变量, "developworks"为其值.

那么关于 `^`, 一个更容易理解的方式, 可以认为是 `^name` 是获取 `name`的值. 并且 `^` 符号只能用在模式匹配中.