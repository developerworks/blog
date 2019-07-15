# Profiling Academy: Custom constraints(自定义约束)

## 原文

- [Profiling Academy: Custom constraints][1]

## 简介

本文将说明如何以及何时创建基于路径导航的自定义约束. 本文假设你已经了解了 Profiling 的基本知识, 如果你想了解其基础可以参考 [Introduction to FHIR and profiling][2] 和 [Start profiling][3] 两篇入门级文章.

在本文中, 你将会学习到自定义约束具体内容. 我们使用 Forge 官方的 HL7®FHIR® 补充规范编辑器, 来创建我们的补充规范, 你可以 [免费下载][4] Forge.

本文涵盖的内容包括:

- 路径导航: 路径导航是什么意思, 如何使用它, 什么时候需要它？
- FHIRPath 基础, 为 FHIR 设计的路径导航语言
- 何时以及如何在 Profiles 中使用不变量(invariants)
- 如何在搜索参数中添加元素路径

## 路径导航

FHIR资源包含称为元素的属性的分层列表.在许多情况下, 需要引用层次结构中的特定元素.例如, 当您要定义**鉴别器的路径**, **搜索参数中特定元素的路径**或**Profile**中元素的规则时.

在FHIR中, 使用两种路径表达式语言来浏览资源及其元素.第一种是XPath, 它是一种XML路径语言, 用于选择XML文档中的节点.然而, 这种语言的采用率很低, 主要原因是它是XML特有的, 并且没有被很好地采用.第二种是FHIRPath, 一种基于路径的导航和提取语言, 类似于XPath, 但专门为FHIR设计.操作以层次数据模型的逻辑内容表示, 并支持数据的遍历、选择和过滤.

在本文中, 我们将主要关注**FHIRPath**及其在数据分析项目中的用途.我们将从对FHIRPath的简要说明开始.在下一章中, 您将学习如何在称为不变量(invariants)的元素上创建规则, 以及如何在搜索参数中定义元素路径.

## FHIRPath

**FHIRPath** 允许您在资源的元素树中导航.例如, 可以使用表达式 `name.given` 来检索`given`元素的所有值.请注意, 此表达式将浏览所有类型的元素, 并生成每个类型元素中存在的所有`given`元素的集合.不需要在表达式中提供根节点（例如 `Patient.name.given`）.两个表达式对患者资源的结果相同, 但后者仅在对类型为 **Patient** 的资源进行评估时返回结果.

### FHIRPath 表达式的输出(集合)

请记住, FHIRPath表达式的结果始终是一个集合, 即使返回的值是单个项.此集合是有序的、非唯一的、索引的和可计数的.这些特征解释如下.

|类型|说明|
|-|-|
|**Ordered**|尽可能保持集合中元素的顺序.|
|**Non-unique**|集合可能包含重复元素.尽管允许, 但有一些函数可用于从集合中剔除重复元素.|
|**Indexed**|集合中的元素索引从0开始, 第一项索引为 0,第二个元素索引为1, 依此类推.|
|**Countable**|可以使用函数`count()`计算集合中包含多少个元素.|

注意在FHIRPath中没有空值(`null`)这样的东西, 相反, 如果没有可用的数据, 这将产生一个空集合（表示为`{}`）.

### 符号和操作符

#### 符号

在 FHIRPath 表达式中允许的符号如下:

|符号|说明|例子|
|-|-|-|
|`()`|定义表达式分组|(`name.given` &#124; `name.family`)|
|`[]`|字符串或列表索引|`name[0]`|
|`{}`|列表定界符|`{value1, value2}`|
|`.`|限定符、访问器和调用|`name.given`|
|`,`|用于分隔句法列表中项目的逗号|`{value1, value2}`|
|`= != ~ !~ < > <= >=`|比较操作符|`name = "Chalmers"`|
|`+ - * /`|数学操作符|`1+3`|
|`&`|字符串连接|`name.given & " " & name.family`|

#### 逻辑操作符

下表列巨额了所有的在 FHIRPath 表达式中支持的逻辑操作符

|操作符|说明|例子|
|-|-|-|
|**and**||`a and b`|
|**or**||`a or b`|
|**xor**||`a xor b`|
|**implies**||`a implies b`|

#### 数据类型操作符

如前所述, 根类型在 FHIRPath 表达式中是可选的.同样, 您可以忽略表达式中某个元素的类型（导致集合包含一个或多个数据类型的项）, 或者在向下导航到树之前筛选类型上的集合（即, 使用 `ofType` 元素）.以下数据类型运算符可用于确定项是属于特定类型还是将项视为特定类型.

|操作符||例子|
|-|-|-|
|**is**||`name.given is "Peter"`|
|**as**||`Observation.value as Quantity` |

#### 集合操作符

|操作符||例子|
|-|-|-|
|**&#124;**|**合并**两个集合, 并**剔除**重复元素|`name.given` &#124; `name.family`|
|**in**|检查是否左侧元素存在于右侧集合中|`"Peter" in name.given`|
|**contains**| `in`的反向操作, 检测左侧的值是否在右侧的集合中|`name.given contains {"Peter","James"}`|

### FHIRPath 调用

FHIRPath表达式可以包含字面量和/或函数调用.

#### 字面量

|字面量|值|
|-|-|
|**boolean**|true, false|
|**string**|'test string', 'urn:oid:3.4.5.6.7.8'|
|**integer**|0, 45|
|**decimal**|0.0, 3.141592653589793236|
|**dateTime**|@2015-02-04T14:34:28Z (@ followed by ISO8601 compliant date/time)|
|**time**|@T14:34:28+09:00 (@ followed by ISO8601 compliant time beginning with T)|
|**quantity**|10 'mg', 4 days|

#### 函数

有关完整的功能列表, 请访问FHIRPath规范.下面是FHIRPath表达式中经常使用的函数示例列表.

|函数|返回类型|说明|
|-|-|-|
|`empty()`|Boolean|如果集合为空返回 true, 否则为 false|
|`not()`|Boolean|如果不是一个集合返回 true, 否则返回 true, 其他情况返回空集合|
|`exists([expression])`|Boolean|如果集合为空返回 false,否则返回 true, 求值前可以应用可选的条件,等同于`(criteria).exists()`|
|`first()`|Collection|如果不为空, 则返回集合中的第一个项, 否则返回空集合.此函数与`item(0)`等效.
|`last()`|Collection|如果不为空,则返回集合中的最后一项,否则返回空集合.|
|`where(expression)`|Collection|返回根据给定条件筛选的集合.包含计算结果为true的项, 排除计算结果为false或空的项.|
|`iif(expression, true-result, [otherwise-result])`|Collection|如果`expression`为真, 则返回`true-result`的值；如果`expression`为假或集合为空, 则返回（可选）`otherwise-result`的值.如果没有其他结果, 则返回空集合.|
|`contains(string)`|Boolean|如果给定字符串是集合中单个字符串项的子字符串, 或者如果子字符串是空字符串, 则返回true.|
|`combine(collection)`|Collections|将集合与给定集合组合, 但不删除任何重复项.|

## 不变量

不变量可用于基于XPath或FHIRPath表达式对元素施加附加约束.

例如, 假设您希望强制在不用药时添加原因.不能通过将`reasonNotGiven`元素的最小基数更改为1来完成此操作, 因为只有满足特定条件时, 元素才是必需的(**注: 也就是在不用药的情况下`reasonNotGiven`才是强制性的, 有前置条件**).在这种情况下, 可以使用以下表达式:

```shell
notGiven implies reasonNotGiven.exists()
```

这个表达式的第一部分是`notGiven`元素的布尔值.只有当此元素的值为`true`时, 才会计算表达式的第二部分.当`notGiven`元素的值为`false`时, 将不计算表达式的第二部分, 并且整个表达式的值将为`true`.当`notGiven`元素为空时, 表达式将返回空集合.

当`reasonNotGiven`元素具有值时, 表达式的第二部分返回`true`, 否则返回`false`.所以这个表达式的意义是, 当`notGiven`元素的值为true`时`, `reasonNotGiven`元素应该存在.

### 向 ElementDefinition 中添加约束

ElementDefinition 有一个名为`constraint`的子元素, 可用于添加表达式.
![clipboard.png][5]

必须提供键(`Key`)、严重性(`Severity`)（错误或警告）、约束的人工描述和实际 FHIRPath 表达式.相应的 XPath 表达式是可选的.

下面是一个示例, 说明了在上面解释的`reasonNotGiven`元素上添加约束时XML代码的样子.

```xml
<element id="MedicationAdministration.reasonNotGiven">
    <path value="MedicationAdministration.reasonNotGiven"/>
    <constraint>
        <key value="ele-2"/>
        <severity value="error"/>
        <human value="If medication is not given the reason should be present"/>
        <expression value="notGiven implies reasonNotGiven.exists()"/>
    </constraint>
</element>
```

### 在 Forge 中添加约束

如果要使用 Forge 向元素添加约束, 请向下导航元素属性, 然后单击 `Constraints` 旁边的`+`键.添加键并选择严重度（警告或错误）.在表达式字段中添加FHIRPath表达式.在"描述"字段中, 添加对表达式的可读描述.

![clipboard.png][6]

## 搜索参数中的元素路径

在 `SearchParameter` 资源中, 需要定义用于评估搜索参数的元素的路径.可以使用 `expression` 元素定义FHIRPath表达式, 也可以使用 `xpath` 元素定义 XPath 表达式.请注意, 当使用 `xpath` 元素指示应如何解释 XPath 语句时, `xpathUsage`元素是必需的.

下面是一个搜索参数的示例, 可用于筛选 **Condition** 的临床状态.

```xml
<SearchParameter xmlns="http://hl7.org/fhir">
  <id value="clinicalstatus" />
  <url value="http://hl7.org/fhir/SearchParameter/Condition-clinicalstatus" />
  <name value="clinicalstatus" />
  <status value="active" />
  <code value="clinicalstatus" />
  <base value="Condition" />
  <type value="token" />
  <description value="The clinical status of the condition" />
  <expression value="Condition.clinicalStatus" />
  <xpath value="f:Condition/f:clinicalStatus" />
  <xpathUsage value="normal" />
</SearchParameter>
```

请注意, 通过在表达式中省略根类型并在基值中选择多个资源类型, 搜索参数可以变得更通用.如果要将搜索参数应用于所有资源类型, 请使用 `Resource` 作为值和根类型.

```xml
<base value="Resource" />
<type value="token" />
<description value="Logical id of this artifact" />
  <expression value="Resource.id" />
  <xpath value="f:Resource/f:id" />
  <xpathUsage value="normal" />
</SearchParameter>
```

  [1]: https://simplifier.net/guide/profilingacademy/Customconstraints
  [2]: https://simplifier.net/guide/profilingacademy/IntroductiontoFHIRandprofiling
  [3]: https://simplifier.net/guide/profilingacademy/Startprofiling
  [4]: https://simplifier.net/guide/profilingacademy/IntroductiontoFHIRandprofiling
  [5]: https://segmentfault.com/img/bVbu6zS
  [6]: https://segmentfault.com/img/bVbu6Ba
