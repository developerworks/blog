#Defsql: 作为Elixir函数的SQL查询

Elixir的世界是函数的世界. 这里没有对象, 没有实例. 在这种情况下, 我问我自己一个问题.

> 我真的需要ORM吗?

我的答案是: NO.

我需要数据. 直接来自于数据库的纯粹的数据. 另一个我思考的问题是:

> 我需要一个查询数据库的DSL吗?

答案同样是否定的: NO.

其实, 我们已经有了一个现成的数据库查询语言, SQL, 记得么?

如果我创建一个包含SQL的Elixir函数. 如果我们像使用其他函数一样使用这个函数.

我们可以使用Elixir的宏系统来达到这个目的. 同时也是一个学习Elixir宏的好机会.

为此我开始编写 [Defql](https://github.com/fazibear/defql). [Defql](https://github.com/fazibear/defql)是一个简单的Elixir包, 它提供了一个方式用SQL语言来定义函数. 它是如何工作的?

## 查询数据库

这个包提供了一个`defquery`宏. 用它可以定义包含SQL查询的Elixir函数. 比如:


```elixir
defmodule UserQuery do
  use Defql

  defquery get_by_blocked(blocked) do
    "SELECT * FROM users WHERE blocked = $blocked"
  end
end
```

看起来非常干净整洁, 易于阅读. 并且最有用的是, 它支持参数.

调用这类函数也是非常简单的. 取决于给定的参数, 我们可以锁定/解锁用户.

```elixir
UserQuery.get_user(false) # => {:ok, []}
UserQuery.get_user(true) # => {:ok, []}
```

## CRUD

这个包还包含其他的宏, 看一下这个例子:

```elixir
defmodule UserQuery do
  use Defql
  defselect get(conds), table: :users
  definsert add(params), table: :users
  defupdate update(params, conds), table: :users
  defdelete delete(conds), table: :users
end
```

这些宏将会床公共的CRUD函数. 因此不必编写每一个SQL查询. 如何调用他们?

```elixir
UserQuery.get(id: "3")                        # => {:ok, [%{...}]}
UserQuery.add(name: "Smbdy")                  # => {:ok, [%{...}]}
UserQuery.update([name: "Other"], [id: "2"])  # => {:ok, [%{...}]}
UserQuery.delete(id: "2")                     # => {:ok, [%{...}]}
```

## 配置

配置也是一个简单的工作. 现在你可以用两种方式使用它.

使用现有的Ecto连接:

```elixir
config :defql, connection: [
  adapter: Defql.Adapter.Ecto.Postgres,
  repo: Taped.Repo
]
```

作为一个独立的连接

```elixir
config :defql, connection: [
  hostname: "localhost",
  username: "username",
  password: "password"
  database: "dbname",
  pool: DBConnection.Poolboy,
  pool_size: 1
]
```

## 却什么?

当然, 它还在非常早期的开发阶段. 但是`postgres`支持得很好. 下面是缺少的部分:

- MySQL支持
- `IN`的支持
- 清理 ECTO 适配器
- 支持数据库错误

## 摘要

使用 [Defql](https://github.com/fazibear/defql), 我们有一个非常干净, 简单和强大的方式与数据库交互.
