# 补充规范工具

## 简介

本章我提供了一个关于创建补充规范工具的概述性说明, 本章仅仅是概述, 并不包含实际例子和实践.

本章涵盖了工具相关的以下内容:

- 探索现存的工具
- 构建补充规范和扩展
- 构建术语
- 创建示例
- 创建实现指南
- 验证你的工作成果
- 与 FHIR 服务器交互

对于进行 FHIR 补充规范开发的人员, 有几种工具可用. HL7 在 [FHIR Tools Registry][1] 提供关于工具的概述. 在本章节中, 我们将向您介绍常用的与分析相关的工具. 当进行分析时, 您可能会涉及上述几个活动, 对于这些活动中的每一项, 我们都将解释我们喜欢使用哪些工具以及如何使用它们.

## 探索现存的资料

- [FHIR 注册库][2]
- [Simplifier][3]
- [FHIR 扩展性注册库][4]
- hl7.org/fhir 上的 `Profiles` 标签

## 构建补充规范和扩展

[Forge][5] 是补充规范官方的遍及工具. Forge 具有图形化的界面, 能够相对容易地创建, 编辑以及验证补充规范(Profiles)和扩展. Forge 和 Simplifier 的集成可以让你能够直接导入导出补充规范到 Simplifier 项目中. 你可以在这里[下载 Forge][6].

使用 Forge, 你可以做如下的工作:

- 创建和编辑 FHIR 补充规范
- 创建和编辑 FHIR 扩展定义
- 创建和编辑 FHIR 符合性包
- 验证 FHIR 补充规范
- 从 FHIR 服务器获取, 以及发布补充规范到 FHIR 服务器
- 从 FHIR 服务器获取, 以及发布补充规范到 FHIR 注册库
- 定义正式的约束
- 定义切片
- 定义取值集合绑定
- 定义映射

下面是一系列的视频连接, 阐述了 Forge(STU3) 的基本用法:

- [编辑基数][7]
- [主窗口概述][8]
- [约束多个类型,并修改取值集合][9]
- [如何发布到 Simplifier][10]

关于 Forge 的详细文档, 可以参考 [Forge 文档页][11]

## 构建术语

Snapper Author（STU3）是一个方便的工具，我们喜欢使用它来创建术语资源，如代码系统、值集和 `ConceptMaps` . 您还可以使用此工具直接验证您创建的资源. 

Snapper Author 允许您:

- 单独定义代码或通过 CSV 批量上传
- 编辑所有相关元数据
- 代码查询术语服务器
- 创作 `ConceptMap`
- 验证工作成果
- 从 FHIR 服务器导入资源
- 将资源导出为 JSON 或上传到 FHIR 服务器

下面是一些视频链接, 说明如何使用 Snapper Author 构建代码系统和取值集合:

- [CodeSystem demonstration][12]
- [ValueSet demonstration][13]

## 创建示例

[SMART FRED][14] 是一个不错的快速创建 JSON 示例的工具. 你可以使用 DSTU2 或 STU3 版本.

SMART FRED 可以用来:

- 创建资源
- 在捆束中创建资源
- FHIR 模式感知的编辑
- 导出为 JSON

首先选择页面顶部的 `Open Resource`. 从空白资源开始，选择资源类型或从本地文件或网站URL打开现有资源. 从下拉菜单中选择值，开始添加（或编辑）元素. 完成后，选择页面顶部的 `Export JSON` 以获取示例的 JSON 代码.

## 创建实现指南

Forge 有构建实现指南的功能, 但是更加快捷的方式是使用 Simplifier 上的 **IG-editor**. 详细信息科参考: [blog post on creating Implementation Guides using Simplifier][15].

Simplifier 上的 IG-editor 可以让你:

- 非常容易的入门
- 上传补充规范, 取值集合以及其他 FHIR 工件
- 使用 Markdown 书写内容
- 使用 CSS 设置实现指南的样式
- 使用指令渲染资源

下面是一段视频的链接，说明如何在 Simplifier 上开始使用 IG-editor:

- [Simplifier feature movie on Implementation Guides][16]

To learn more about creating IGs, please follow the module on Publishing and validating your work.

关于创建实现指南的更多信息, 请参考: [发布和验证工作成果][17]

## 验证工作成果

我们描述的大多数工具都包含一个选项，可以根据 FHIR 规范验证您的工作. 我们使用 [.NET GUI验证器][18] 或 Simplifier 进行验证(STU3).

.NET 图形化验证器:

- 允许根据规范验证 FHIR XML 或 JSON
- 允许根据规范验证自定义补充规范
- 支持术语验证
- 同时作为规范和术语服务器

Simplifier:

- 允许根据规范验证 FHIR XML 或 JSON
- 允许根据规范验证自定义补充规范
- 在浏览器中运行

以下的视频解释如何使用.NET GUI验证器和 `Simplifier` 进行验证的视频链接:

- [Validation with the .NET GUI validator][19]
- [Validation in Simplifier][20]

## 与 FHIR 服务器交互

我们喜欢使用 Postman 与 FHIR 服务器进行交互。我们经常使用它，要么用于我们自己的 [Vonk][21] 服务器的测试目的，要么用于在真正的服务器上测试我们的分析工作。

使用 Postman 可以:

- 收发 FHIR JSON 和 XML
- 执行 CRUD 操作(GET, POST, UPDATE, DELETE)
- Easily make sample responses for IG
- Create, store and share collections of requests

下面的视频连接描述了如何使用 Postman:

- [Postman demonstration][22]
  
## 其他工具

- Trifolia workbench
  - 补充规范(Profile)编辑器
  - 取值集合编辑器
  - 实现指南导出
- HL 7 [IG-publisher][23]
  - FHIR like IG
  - Static export
- Java validator
- [ClinFHIR][24]

  [1]: http://wiki.hl7.org/index.php?title=FHIR_Tools_Registry
  [2]: http://registry.fhir.org
  [3]: https://simplifier.net/
  [4]: http://hl7.org/fhir/STU3/extensibility-registry.html
  [5]: https://simplifier.net/Forge
  [6]: https://simplifier.net/forge/download
  [7]: https://vimeo.com/242746255
  [8]: https://vimeo.com/242746249
  [9]: https://vimeo.com/242746242
  [10]: https://vimeo.com/242746237
  [11]: http://docs.simplifier.net/forge/index.html
  [12]: https://www.youtube.com/watch?feature=youtu.be&v=5VIqqiQ1UUU
  [13]: https://www.youtube.com/watch?feature=youtu.be&v=hVU9cskxo1Q
  [14]: https://github.com/smart-on-fhir/fred
  [15]: https://blog.fire.ly/2016/08/08/create-my-first-fhir-implementation-guide-using-simplifier/
  [16]: https://www.youtube.com/watch?v=aLQiDBxVXwM
  [17]: https://simplifier.net/guide/profilingacademy/Publishingandvalidatingyourwork
  [18]: http://docs.simplifier.net/fhirnetapi/
  [19]: https://www.youtube.com/watch?v=p35Sw_T6jvc
  [20]: https://www.youtube.com/watch?v=Vxqp_wdLpjM  
  [21]: https://simplifier.net/vonk
  [22]: https://www.youtube.com/watch?v=g9sFmXtZYjc&feature=youtu.be
  [23]: http://wiki.hl7.org/index.php?title=IG_Publisher_Documentation
  [24]: http://clinfhir.com/
