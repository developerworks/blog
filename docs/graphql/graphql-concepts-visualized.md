https://cdn-images-1.medium.com/max/1600/1*9ZFclOcI4mvp4j4bymi8cQ.png

GraphQL 概念可视化

GraphQL经常被解释为: `访问不同来源数据的统一接口`. 虽然这个解释是精确的, 但是它并没有说清楚底层的思路和GraphQL后面的动机, 甚至是为什么被称为`GraphQL` - 你可以看到星辰和夜晚, 却不是一个`繁星闪烁的夜晚`.

GraphQL实际上就是一个数据的「图」. 在本文中, 我将会介绍应用程序数据图. 谈一谈GraphQL如何在这个数据图上进行操作, 以及我们如何缓存GraphQL查询结果, 并且探索其树状结构.

## 应用程序数据图

在现代应用程序中, 许多数据可以用节点和边来标识一个图, 节点表示对象, 边标表示对象间的关系. 例如我们要构造一个分联系系统, 为了简化, 我们的分类有一个「图书」分支, 和「作者」分支, 每本书至少有一个作者. 作者还可能有合著者.

如果我们以图的方式对这种关系进行可视化, 他们看起来就像这样:

![](https://cdn-images-1.medium.com/max/1600/1*EmhOknzZEu9Q6U3q5NmT9Q.png)

上面这张图表示了各种数据片段之间的关系, 几乎所有应用程序在这类图上进行操作: 读取和写入, 这就是GraphQL名称的由来.

> GraphQL允许我们从应用程序数据图中提取子树.


## 使用GraphQL遍历图

现在我们来看一个GraphQL查询例子, 了解其如何从应用程序数据图中提取一个子树.

```
query {
  book(isbn: "9780674430006") {
    title
    authors {
      name
    }
  }
}
```

查询结果:

```
{
  book: {
    title: “Capital in the Twenty First Century”,
    authors: [
      { name: ‘Thomas Piketty’ },
      { name: ‘Arthur Goldhammer’ },
    ]
  }
}
```

更具应用程序数据图, 我们绘制了如下对象间的关系.

![](https://cdn-images-1.medium.com/max/1600/1*9ZFclOcI4mvp4j4bymi8cQ.png)

## 查询路径

> GraphQL 是一个用于遍历数据图, 以产生查询结果树的查询语言.

### 同样的路径, 同样的对象

假设, 我们有如下两个查询, 先后执行

```
# 查询一: 获取一个名称为"Thomas Piketty"作者信息
query particularAuthor {
  author(name: "Thomas Piketty") {
    name
    age
  }
}
# 查询二: 获取图书信息和作者信息
query authorAndBook {
  book(isbn: "9780674430006") {
    title
  }
  author(name: "Thomas Piketty") {
    name
    age
  }
}
```

通过上面代码的第二个查询中的作者字段在第一个查询中已经执行过并且被缓存了, 这个时候Apollo Client 会直接使用第一个查询在缓存中的结果, 而不是重新请求一次服务器获取作者信息.

Apollo会基于这个逻辑删除已经被缓存的查询. Apollo的这种行为就是基于之前的路径假设. 它假设路径`RootQuery→author(id: 6)→name`和之前的第一个查询获取到的相同的信息, 当然如果该假设不成立, 你可以使用`forceFetch`完全绕过缓存.



