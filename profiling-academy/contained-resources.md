## 参考资料

- https://simplifier.net/guide/profilingacademy/Containedresources

## 简介

本文件涵盖的内容包括:

- 什么是被包含资源?
- 什么时候使用, 什么时候不用被包含资源
- 如何使用被包含资源
- 被包含资源的理解

## 什么是被包含资源

**被包含的资源** 是嵌入到另一个资源中的资源。被包含的资源只存在于包含的资源的上下文中，不能单独存在。这意味着被包含的资源在外部是不可识别的。

被包含的资源只有在从包含的资源引用时才有意义。因此，只有存在对所包含资源的引用时，才能包含资源。这是为了确保所包含资源的含义清晰，避免混淆其意义。对打包在包含资源中的资源的引用称为内部包含引用

## 什么时候使用被包含资源

当资源引用的内容除了包含它的资源之外没有独立存在时，可以使用包含的资源。通常，当源数据的第二个用户（如中间件引擎）正在组装资源时，会出现这种情况。例如，在某些情况下，可能没有足够的（标识）信息来创建唯一的、可解析的资源实例。没有任何标识信息（甚至是具有任意标识信息的资源）的资源决不能是包含资源上下文之外的事务的主题，应始终包含该主题。

> **示例 1**: 通常，医嘱只包含药物名称，没有任何识别信息。在这种情况下，您应该将药物资源作为包含在 `MedicationRequest` 资源中的资源发送

> **示例 2**: 假设您想获取诊断报告或条件资源中诊断患者的医生姓名。您没有该从业者的任何附加(识别)信息。您应该将 `Practioner` 资源作为诊断报告或条件资源的包含资源发送。

## 什么时候使用

当有足够的识别信息时，您应该谨慎使用包含的资源。一旦标识丢失，就很难恢复它。所包含的资源决不应简单地用作内联序列化内容的方法 - 参考 [Bundles][1], [Lists][2], 或 [Documents][3]. 

## 如何使用被包含资源(Contained resources)

在继承 **[DomanResurce][4]** 的任何资源中添加一个或多个包含的资源（除了Bundle之外，这将是最多的）来使用 `contained`元素。在XML表示中，所包含的资源应该在任何文本叙述之后和任何扩展之前添加到资源的顶部。包含的资源没有叙述，但它们内部可以有扩展（只是不在包含的元素本身上）。不允许嵌套包含的资源，因此无法将包含的资源添加到已经是包含的资源本身的资源中。

以下是XML中包含的 `Practitioner` (从业者) 资源的不完整示例：

```xml
<DiagnosticReport xmlns="http://hl7.org/fhir">
  <contained>
    <Practitioner>
      <id value="practitioner1"/>
      <name>
        <family value="Practitioner"/>
        <given value="James"/>
      </name>
    </Practitioner>
  </contained>
```

下一步是添加对包含的资源的引用（即内部包含的引用）。下面是一个XML代码的示例，用于内部包含对我们刚刚创建的包含的从业者资源的引用。内部包含的引用始终以 `#` 开头，后跟所包含资源的ID。请注意，此ID在包含资源之外不存在。因此，不可能对`Practitioner/practitioner1`运行 GET 请求以从服务器获得包含的资源。

```xml
<performer>
    <actor>
    	<reference value="#practitioner1" />
    </actor>  
</performer>
```

最终结果看起来像这样:

```xml
<DiagnosticReport xmlns="http://hl7.org/fhir">
  <contained>
    <Practitioner>
      <id value="practitioner1"/>
      <name>
        <family value="Practitioner"/>
        <given value="James"/>
      </name>
    </Practitioner>
  </contained>

  <!-- ... -->

  <performer>
    <actor>
    	<reference value="#practitioner1"/>
    </actor>  
  </performer>
```

## 被包含资源(Contained resources)的理解

所包含资源的含义取决于其上下文。通过查看包含的资源来解析引用。例如，假设一个诊断报告包含一个从业者和观察结果。在观察资源中引用从业者资源, 一次形成内部引用。

**被包含的资源** (Contained resources)不从**包含资源** (containing resource)继承上下文。例如，假设包含资源和被包含资源都有一个主题。在这种情况下，不能假定**被包含资源**与**包含资源**具有相同的主题。

## 关键词

- 被包含
- 外部不可识别
- 非独立存在(依附于包含他的外层资源)

  [1]: https://www.hl7.org/fhir/bundle.html
  [2]: https://www.hl7.org/fhir/list.html
  [3]: https://www.hl7.org/fhir/documents.html
  [4]: https://www.hl7.org/fhir/domainresource.html