[GraphQL][1] 是Facebook开发的一个应用层查询语言. 后端定义基于图的模式. 客户端可以按需查询需要的数据.

![clipboard.png][2]

上图所示, 查询流程分为几个步骤, 涉及多个组件, 包括客户端应用程序(Web, 手机, 桌面等App), 一个GraphQL服务器用于解析查询, 以及多个不同的数据来源.

再来看一下, GraphQL官方首页的说明

![clipboard.png][5]

- 描述数据
- 请求你想要的数据
- 获得预期的结果

客户端数据要求发生变化时, 不需要修改后端. 因此, 你不必因为客户端数据需求的变更而改变你的后端. 这解决了管理REST API中的最大的问题.

为什么解决了REST API的大问题, 看如下阐述:

> **注解:**

> 只要你的业务模型没有发生变化, 从数据模型不会发生变化, 那么我们就不需要修改后端API. 前端按照需要的字段进行查询即可. 如果业务发生了变化, 我们只需要修改GraphQL的模式定义, 并且实现对应的服务器端数据查询逻辑即可. 传统的REST查询那些字段是固定的, 客户端不能指定, GraphQL可以让客户端指定要获取那些字段的数据, 这给客户端带来了极大的灵活性, 让前后端进一步分离. 查询是可以嵌套的, 返回的JSON对象结构和GraphQL查询的结构是一样的, 这样更方便客户端自己定义数据的结构.


GraphQL同样能够让客户端程序高效地批量获取数据. 例如, 看一看下面这个GraphQL请求:

```graphql
{
  latestPost {
    _id,
    title,
    content,
    author {
      name
    },
    comments {
      content,
      author {
        name
      }
    }
  }
}
```

这个 GraphQL 请求获取了一篇博客文章和对应评论与作者信息的数据. 下面是请求的返回结果:

```
{
  "data": {
    "latestPost": {
      "_id": "03390abb5570ce03ae524397d215713b",
      "title": "New Feature: Tracking Error Status with Kadira",
      "content": "Here is a common feedback we received from our users ...",
      "author": {
        "name": "Pahan Sarathchandra"
      },
      "comments": [
        {
          "content": "This is a very good blog post",
          "author": {
            "name": "Arunoda Susiripala"
          }
        },
        {
          "content": "Keep up the good work",
          "author": {
            "name": "Kasun Indi"
          }
        }
      ]
    }
  }
}
```

如果你使用的是REST的话，你需要调用多个REST API的请求才能获取这些信息。

> ## 上手视频
> 打开[GraphQL沙箱][3], 然后跟着下面的视频练习:
> https://v.qq.com/x/page/r0387ut9y45.html

## GraphQL是一个规范

因此, 它可以用于任何平台或语言. 它有一个参考的实现 JavaScript,  由Facebook维护. 还有许多社区维护的实现有许多种语言。


之前我们用简短的描述说明了GraphQL是什么, 对其有了一个基本的映像, 现在我们通过实际的操作来感受GraphQL具体是一个什么东西.

首先在浏览器中打开: https://sandbox.learngraphql.com ,然后在左侧的查询窗口中输入下面的查询语句:

我们会看到下图的GraphiQL查询界面, 其界面窗口如下所示:

![GraphiQL查询界面][4]

```
{
  latestPost {
    title,
    summary
  }
}
```

我们将在右侧的窗口看到查询的结果如下:

```
{
  "data": {
    "latestPost": {
      "title": "New Feature: Tracking Error Status with Kadira",
      "summary": "Lot of users asked us to add a feature to set status for errors in the Kadira Error Manager. Now, we've that functionality."
    }
  }
}
```

现在我们体验了一下GraphQL是怎么工作的, 下面我们来扩展一下, 起始GraphQL的功能远比你现在看到的要强大.

我们可以打开右上角的`Docs`连接, 可以看到整个基于图的模式有哪些东西是我们可以使用的. 下面我通过一个视频来演示怎么样进一步深入使用GraphQL.




  [1]: http://graphql.org/
  [2]: https://segmentfault.com/img/bVLbZo
  [3]: https://sandbox.learngraphql.com
  [4]: https://segmentfault.com/img/bVLcT0
  [5]: https://segmentfault.com/img/bVLXcp
