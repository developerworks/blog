原文: http://elixir-lang.org/crash-course.html

> **水平有限, 翻译有误或不恰当的地方, 请在回复中指出**. 

## 函数调用

Elixir允许你调用函数的时候省略括号, Erlang不行.

| Erlang             | Elixir          |
|--------------------+-----------------|
| `some_function().` | `some_function` |
| `sum(A,B)`         | `sum a,b`       |

从模块中调用一个函数, 使用不同的语法, 在Erlang, 你可以写:

```erlang
lists:last([1,2]).
```

从List模块中调用last函数. 在Elixir中使用`.`符号代替`:`符号.

```elixir
List.last([1,2])
```

注意. 因为Erlang模块以原子的形式标识, 你可以在Elixir中以如下方式调用Erlang函数.

```elixir
:lists.sort [1,2,3]
```

所有Erlang内置函数存在于`:erlang`模块中.

## 数据类型

### 原子

在Erlang中, 原子是以任意小写字母开头的标识符号. 例如`ok`, `tuple`, `donut`. 大写字母开头的任意标识被作为变量名称.


Erlang

```erlang
im_an_atom.
me_too.
Im_a_var.
X = 10.
```

Elixir

```elixir
:im_an_atom
:me_too

im_a_var
x = 10

Module # 原子别名, 扩展为:`Elixir.Module`
```

还可以创建非小写字母开头的原子, 两种语言的语法差异如下:

Erlang

```erlang
is_atom(ok).                %=> true
is_atom('0_ok').            %=> true
is_atom('Multiple words').  %=> true
is_atom('').                %=> true
```

Elixir

```elixir
is_atom :ok                 #=> true
is_atom :'ok'               #=> true
is_atom Ok                  #=> true
is_atom :"Multiple words"   #=> true
is_atom :""                 #=> true
```

### 元组

元组的语法两种语言相同, 但API有却别.

## 列表和二进制

Erlang

```erlang
is_list('Hello').        %=> false
is_list("Hello").        %=> true
is_binary(<<"Hello">>).  %=> true
```

Elixir

```elixir
is_list 'Hello'          #=> true
is_binary "Hello"        #=> true
is_binary <<"Hello">>    #=> true
<<"Hello">> === "Hello"  #=> true
```

Elixir `string` 标识一个UTF-8编码的二进制, 同时有一个`String`模块处理这类书籍.
Elixir假设你的源文件是以UTF-8编码的. Elixir中的`string`是一个字符列表, 同时有一个`:string`模块处理这类数据.


Elixir 支持多行字符串(`heredocs`):

```elixir
is_binary """
This is a binary
spanning several
lines.
"""
#=> true
```

### 关键字列表

包含两个元素的元组列表

Erlang

```erlang
Proplist = [{another_key, 20}, {key, 10}].
proplists:get_value(another_key, Proplist).
%=> 20
```

Elixir

```elixir
kw = [another_key: 20, key: 10]
kw[:another_key]
#=> 20
```

### Map

Erlang

```erlang
Map = #{key => 0}.                  % 创建Map
Updated = Map#{key := 1}.           % 更新Map中的值
#{key := Value} = Updated.          % 匹配
Value =:= 1.
%=> true
```

Elixir

```elixir
map = %{:key => 0}                  # 创建Map
map = %{map | :key => 1}            # 更新Map中的值
%{:key => value} = map              # 匹配
value === 1
#=> true
```


注意:

1.`key:` 和 `0`之间一定要有一个空格, 如下:

```elixir
iex(2)> map = %{key:0}               
**(SyntaxError) iex:2: keyword argument must be followed by space after: key:
    
iex(2)> map = %{key: 0}
%{key: 0}
```

2.所有的key必须是原子类型才能这么用.


### 正则表达式

Elixir支持正则表达式的字面语法. 这样的语法允许正则表达式在编译时被编译而不是运行时进行编译, 并且`不要求`转义特殊的正则表达式符号:

Erlang

```erlang
{ ok, Pattern } = re:compile("abc\\s").
re:run("abc ", Pattern).
%=> { match, ["abc "] }
```

Elixir:

```elixir
Regex.run ~r/abc\s/, "abc "
#=> ["abc "]
```

正则表达式还能用在`heredocs`当中, 提供了一个定义多行正则表达式的便捷的方法:

```elixir
Regex.regex? ~r"""
This is a regex
spanning several
lines.
"""
```

## 模块

每个erlang模块保存在其自己的文件中, 并且有如下结构:

```erlang
-module(hello_module).
-export([some_fun/0, some_fun/1]).

% A "Hello world" function
some_fun() ->
  io:format('~s~n', ['Hello world!']).

% This one works only with lists
some_fun(List) when is_list(List) ->
  io:format('~s~n', List).

% Non-exported functions are private
priv() ->
  secret_info.
```

这里我们创建一个名为`hello_module`的模块. 其中我们定义了三个函数, 前两个用于其他模块调用, 并通过`export`指令导出. 它包含一个要导出的函数列表, 其中没个函数的书写格式为`<function name>/<arity>`. `arity`标书参数的个数.

上面的Elixir等效代码为:

```elixir
defmodule HelloModule do
  # A "Hello world" function
  def some_fun do
    IO.puts "Hello world!"
  end

  # This one works only with lists
  def some_fun(list) when is_list(list) do
    IO.inspect list
  end

  # A private function
  defp priv do
    :secret_info
  end
end
```

在Elixir中, 还可以在一个文件中定义多个模块, 以及嵌套模块.


```elixir
defmodule HelloModule do
  defmodule Utils do
    def util do
      IO.puts "Utilize"
    end

    defp priv do
      :cant_touch_this
    end
  end

  def dummy do
    :ok
  end
end

defmodule ByeModule do
end

HelloModule.dummy
#=> :ok

HelloModule.Utils.util
#=> "Utilize"

HelloModule.Utils.priv
#=> ** (UndefinedFunctionError) undefined function: HelloModule.Utils.priv/0
```


## 函数语法

Erlang 图书的[这章](http://learnyousomeerlang.com/syntax-in-functions)提供了模式匹配和函数语法的详细描述. 这里我简单的覆盖一些要点并提供Erlang和Elixir的代码示例:

### 模式匹配

Erlang

```erlang
loop_through([H|T]) ->
  io:format('~p~n', [H]),
  loop_through(T);

loop_through([]) ->
  ok.
```

Elixir

```elixir
def loop_through([h|t]) do
  IO.inspect h
  loop_through t
end

def loop_through([]) do
  :ok
end
```

当定义一个同名函数多次的时候, 没个这样的定义称为`分句`. 在Erlang中, `分句`总是紧挨着的, 并且由分号`;`分割. 最后一个分句通过点`.`终止.

Elixir并不要求使用标点符号来分割分句, 但是他们必须分组在一起(`同名函数必须上下紧接着`)

### 标识函数

在Erlang和Elixir中, 函数不仅仅由名字来标识, 而是由`名字`和`参数的个数`共同来标识一个函数. 在下面两个例子中, 我们定义了四个不同的函数(全部名为`sum`, 不同的参数个数)

Erlang

```erlang
sum() -> 0.
sum(A) -> A.
sum(A, B) -> A + B.
sum(A, B, C) -> A + B + C.
```

Elixir

```elixir
def sum, do: 0
def sum(a), do: a
def sum(a, b), do: a + b
def sum(a, b, c), do: a + b + c
```

基于某些条件, Guard表达式(Guard expressions), 提供了一个精确的方式定义接受特定取值的函数

Erlang

```erlang
sum(A, B) when is_integer(A), is_integer(B) ->
    A + B;
sum(A, B) when is_list(A), is_list(B) ->
    A ++ B;
sum(A, B) when is_binary(A), is_binary(B) ->
    <<A/binary,  B/binary>>.
sum(1, 2).
%=> 3
sum([1], [2]).
%=> [1,2]
sum("a", "b").
%=> "ab"
```

Elixir

```elixir
def sum(a, b) when is_integer(a) and is_integer(b) do
    a + b
end

def sum(a, b) when is_list(a) and is_list(b) do
    a ++ b
end

def sum(a, b) when is_binary(a) and is_binary(b) do
    a <> b
end

sum 1, 2
#=> 3

sum [1], [2]
#=> [1,2]

sum "a", "b"
#=> "ab"
```

### 默认值

Erlang不支持默认值

Elixir 允许参数有默认值

```elixir
def mul_by(x, n \\ 2) do
    x * n
end

mul_by 4, 3 #=> 12
mul_by 4    #=> 8
```

### 匿名函数

匿名函数以如下方式定义:

Erlang

```erlang
Sum = fun(A, B) -> A + B end.
Sum(4, 3).
%=> 7

Square = fun(X) -> X * X end.
lists:map(Square, [1, 2, 3, 4]).
%=> [1, 4, 9, 16]
```

当定义匿名函数的时候也可以使用模式匹配.

Erlang

```erlang
F = fun(Tuple = {a, b}) ->
        io:format("All your ~p are belong to us~n", [Tuple]);
        ([]) ->
        "Empty"
    end.

F([]).
%=> "Empty"

F({a, b}).
%=> "All your {a,b} are belong to us"
```

Elixir

```elixir
f = fn
      {:a, :b} = tuple ->
        IO.puts "All your #{inspect tuple} are belong to us"
      [] ->
        "Empty"
    end

f.([])
#=> "Empty"

f.({:a, :b})
#=> "All your {:a, :b} are belong to us"
```

### 一类函数

匿名函数是第一类值, 他们可以作为参数传递给其他函数, 也可以作为返回值. 这里有一个特殊的语法允许命名函数以相同的方式对待:

Erlang

```erlang
-module(math).
-export([square/1]).

square(X) -> X * X.

lists:map(fun math:square/1, [1, 2, 3]).
%=> [1, 4, 9]
```

Elixir

```elixir
defmodule Math do
  def square(x) do
    x * x
  end
end

Enum.map [1, 2, 3], &Math.square/1
#=> [1, 4, 9]
```

### Partials in Elixir

Elixir supports partial application of functions which can be used to define anonymous functions in a concise way:

```elixir
Enum.map [1, 2, 3, 4], &(&1 * 2)
#=> [2, 4, 6, 8]

List.foldl [1, 2, 3, 4], 0, &(&1 + &2)
#=> 10
```

Partials also allow us to pass named functions as arguments.

```elixir
defmodule Math do
  def square(x) do
    x * x
  end
end

Enum.map [1, 2, 3], &Math.square/1
#=> [1, 4, 9]
```

## 控制流

The constructs `if` and `case` are actually expressions in both Erlang and Elixir, but may be used for control flow as in imperative languages.

### Case

The case construct provides control flow based purely on pattern matching.

Erlang

```erlang
case {X, Y} of
  {a, b} -> ok;
  {b, c} -> good;
  Else -> Else
end
```

Elixir

```elixir
case {x, y} do
  {:a, :b} -> :ok
  {:b, :c} -> :good
  other -> other
end
```

### If

Erlang

```erlang
Test_fun = fun (X) ->
  if X > 10 ->
       greater_than_ten;
     X < 10, X > 0 ->
       less_than_ten_positive;
     X < 0; X =:= 0 ->
       zero_or_negative;
     true ->
       exactly_ten
  end
end.

Test_fun(11).
%=> greater_than_ten

Test_fun(-2).
%=> zero_or_negative

Test_fun(10).
%=> exactly_ten
```

Elixir

```elixir
test_fun = fn(x) ->
  cond do
    x > 10 ->
      :greater_than_ten
    x < 10 and x > 0 ->
      :less_than_ten_positive
    x < 0 or x === 0 ->
      :zero_or_negative
    true ->
      :exactly_ten
  end
end

test_fun.(44)
#=> :greater_than_ten

test_fun.(0)
#=> :zero_or_negative

test_fun.(10)
#=> :exactly_ten
```

区别:

1. `cond`允许左侧为任意表达式, erlang只允许`guard`子句.
2. `cond`中的条件只有在表达式为`nil`和`false`的时候为false, 其他情况一律为true, Erlang为一个严格的布尔值

Elixr还提供了一个简单的`if`结构

```elixir
if x > 10 do
  :greater_than_ten
else
  :not_greater_than_ten
end
```

### 发送和接受消息

Erlang

```erlang
Pid = self().

Pid ! {hello}.

receive
  {hello} -> ok;
  Other -> Other
after
  10 -> timeout
end.
```

Elixir 


```elixir
pid = Kernel.self

send pid, {:hello}

receive do
  {:hello} -> :ok
  other -> other
after
  10 -> :timeout
end
```

## 添加Elixir到现有的Erlang程序

### Rebar集成

如果使用Rebar,可以把Elixir仓库作为依赖.

```
https://github.com/elixir-lang/elixir.git
```

Elixir实际上是一个Erlang的app. 它被放在`lib`目录中, rebar不知道这样的目录结构. 因此需要制定Elixir的库目录.

```
{lib_dirs, [
  "deps/elixir/lib"
]}.
```

### 手动集成

If you are not using rebar, the easiest approach to use Elixir in your existing Erlang software is to install Elixir using one of the different ways specified in the [入门指南](http://elixir-lang.org/getting-started/introduction.html) and add the `lib` directory in your checkout to `ERL_LIBS`
