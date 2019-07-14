## 简介

在本文中, 你将会学习到切片具体内容. 我们使用 Forge 官方的 HL7®FHIR® 补充规范编辑器, 来创建我们的补充规范, 你可以 [免费下载][1] Forge


本文包含的内容有:

- 什么时候使用切片
- 定义一个鉴别器
- 在 Forge 中进行切片操作

## 什么时候使用切片

切片的目的是把原本应用于整个元素的约束切分出来, 可以对元素的分片(部分)进行单独的约束, 这有益于对单个元素的不同分片定义不同的约束, 以进一步自定义 Profile.

> 举例:

一个患者的电话通信方式(Patient资源的 telecom 元素), 有手机号码, 家庭号码, 或者工作号码, 我们需要手机号码是强制性的约束, 家庭号码和工作号码是可选的(患者注册的时候必须要填写手机号码, 家庭号码和工作号码选填). 

这时候我们就需要分片的功能, 先把 telecom 元素分片, 然后在对每种单独的联系方式进行约束(设置基数 Cardinality)

## 定义一个鉴别器

一个系统需要某些类型的指示器来判定切片的区别. 换言之,它需要知道选择那个元素才能进行区分. 鉴别器用于区分切片元素相互之间的差别

当您第一次切片一个元素时, 您将需要分配一个鉴别器. 为此, 需要在创建切片时指定路径(Path)和类型(Type). 路径指示可以在哪个元素中找到鉴别器, 类型定义应如何计算鉴别器. FHIR规范中定义了五种类型:

**表 2-1**

|类型|说明|
|-|-|
|**value**|切片在指定元素中具有不同的值|
|**exists**|切片通过指定元素的存在或不存在来区分|
|**pattern**|切片在指定元素中具有不同的值，这是通过根据适用的ElementDefinition.pattern[x]对其进行测试来确定的|
|**type**|切片按指定元素的类型与指定的补充规范(Profile)进行区分|
|**profile**|切片通过指定元素与指定补充规范(Profile)的一致性来区分|

> 举例:

假设您希望对元素 **Patient.identifier** 进行切片, 并为患者分别添加一个 **国家级** 的标识和 **医院级** 的标识. 鉴别器是用于注册标识符的命名系统的值(例如, 在荷兰, BSN 编号用作国家标识, 而医院使用本地患者编号作为医院标识). 在这种情况下, 鉴别器的类型(Type)是值(value), 其路径(Path)是系统(system).

### 切片(复数)和切片条目的基数

当在元素上定义了一个或多个切片后, 可以为每个切片和切片元素定义不同的基数. 在上一节的示例中, 患者至少应具有一个BSN编号和一个可选的本地患者编号. 例如, **identifier** 元素的基数为 **0..*** 或者如果不想允许两个以上的标识符, 可以将其限制为 **0..2**. BSN编号的切片的基数将为 **1..1**, 以使其成为必需的, 而本地患者编号的基数将为 **0..1**, 以使其成为可选的.

另一种可能是添加了两个可选切片(**0..1**), 并通过将**切片条目的基数**限制为 **1..1** 来指定其中一个切片为必须. 例如, 假设希望对元素 **MedicationStatement.subject** 进行切片, 并添加以下切片: 一个名为 **SubjectPatient** 的切片和名为 **SubjectGroup** 的切片. 两个切片都是可选的, 但是**切片条目基数**要求您至少指定一个主题(**二选一**, 不能两个都为空). 

```xml
<element id="MedicationStatement.subject">
    <path value="MedicationStatement.subject"/>
    <slicing>
        <discriminator>
            <type value="type"/>
            <path value="$this"/>
        </discriminator>
        <rules value="open"/>
    </slicing>
    <min value="1"/>
    <max value="1"/>
</element>
<element id="MedicationStatement.subject:SubjectPatient">
    <path value="MedicationStatement.subject"/>
    <sliceName value="SubjectPatient"/>
    <min value="0"/>
    <max value="1"/>
    <type>
        <code value="Reference"/>
        <targetProfile value="http://hl7.org/fhir/StructureDefinition/Patient"/>
    </type>
</element>
<element id="MedicationStatement.subject:SubjectGroup">
    <path value="MedicationStatement.subject"/>
    <sliceName value="SubjectGroup"/>
    <min value="0"/>
    <max value="1"/>
    <type>
        <code value="Reference"/>
        <targetProfile value="http://hl7.org/fhir/StructureDefinition/Group"/>
    </type>
</element>
```

> 上面的 XML 定义了一个元素, 以及与之对应的两个切片.

![clipboard.png][5]

回顾上一节, 介绍了 5 中类型的鉴别器, 上图中使用了 **type** 类型的鉴别器, 如**表 2-1**中定义的: **切片按指定元素的类型与指定的补充规范(Profile)进行区分**, 它表示: 切片需要对比其 `<path>` 所指定的类型和 `<type>.< targetProfile>` 来区分两个不同的切片. 这连个切片除了 元素ID, 和 **<type>.< targetProfile>** 的值不同其他的都一样.


### 切片规则

在评估资源实例时,切片规则定义用来如何解释切片. 在切片元素的时候, 可以使用这些规则定义来说明是否允许附加内容. 在切片元素的`rules`元素中, 可以选择以下选项之一:


![clipboard.png][2]


|代码|定义|
|-|-|
|**closed**|除此Profile中切片所描述的内容外,不允许其他内容|
|**open**|列表中的任何地方都允许添加内容|
|**openAtEnd**|允许附加内容,但仅限于列表末尾.注意,使用它需要对切片进行排序,这使得很难共享使用.这只能在绝对需要的情况下使用, 否则不建议使用该选项.|

例如: 假设您已经将 `Practitioner.identifier`. 在荷兰, 我们有不同的识别从业者的方法: UZI编号, AGB代码和BIG代码. 假设您为UZI编号添加了一个切片, 为AGB代码添加了一个切片. 当切片规则为 **open** 时, 您可以添加额外的标识符, 例如BIG代码. 但是, 当切片规则为 **closed** 时, 不允许添加BIG代码. 当切片规则为 **openAtEnd** 时, 可以添加一个BIG代码, 但只能在标识符列表的末尾添加.

## 在 Forge 中进行切片

**图 5-1: 元素切片定义**

![clipboard.png][3]

选择要切片的元素, 然后单击分片图标(如上图). 元素现在是 "Sliced", 可以通过单击带有加号的切片图标来添加切片. 可以根据需要创建任意多个切片, 方法是选择切片元素, 然后再次单击 "Add Slice" 图标. 添加的切片可以像任何其他元素一样定义和约束.


添加切片后, Forge 显示一条警告消息, 其中包含没有为切片元素定义鉴别器的消息. 鉴别器信息可以在切片元素的 **"Element Properties"** 中的 **"Slicing Details"** 下提供. 分配路径和类型给鉴别器, 请注意, 当对 **"name[x]"** 元素进行切片时, Forge 会自动初始化类型为 **type** 的鉴别器.

**图 5-2: 切片规则**

![clipboard.png][4]

默认情况下，**rules** 元素设置为 **open** (如图 5-2 中 2 处). 通过从下拉菜单中选择所需的值, 可以在切片元素(旁边有切片图标的元素, 如图 5-2 中 1 处, 此例中为 `Patient.identifier` 元素, 表示患者标识)中更改此值.

请记住, 切片还用于表示扩展定义和扩展元素. Forge用户界面试图向用户隐藏这种复杂性, 但是它在XML查看器中是清晰可见的.


  [1]: https://simplifier.net/forge/download
  [2]: https://segmentfault.com/img/bVbu4nT
  [3]: https://segmentfault.com/img/bVbu4nT
  [4]: https://segmentfault.com/img/bVbu4rp
  [5]: https://segmentfault.com/img/bVbu4n4