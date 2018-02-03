## 操作数据库

### 使用 Neo4j browser

### 创建节点和关系

### 修改现有的数据

### 标签和属性

**多标签**

```
CREATE (u:User:Inactive) RETURN u
```

这样,`u`节点上存在两个标签`User`,`Inactive`

**创建节点的时候同时创建节点的属性**

```
CREATE (u:User {name: "John", surname: "Doe"}) RETURN u
```

**创建多个模式**

这里的**模式** 代表节点, 关系, 路径等想要创建的对象.

```
CREATE (a:User {name: "Jane", surname: "Roe"}),
       (b:User {name: "Carlos", surname: "Garcia"}),
       (c:User {name: "Mei", surname: "Weng"})
```


**创建关系**

```
CREATE (:User {name: "Jack", surname: "Smith"})
          -[:Sibling]->
          (:User {name: "Mary", surname: "Smith"})
```

**创建完整的路径**

一条路径, 包含多个节点以及多个关系. 可以在一条CREATE语句中同时创建, 如下:

```
CREATE p = (jr:User {name: "Jack", surname: "Roe"})-[:Sibling]->
             (:User {name: "Mary", surname: "Roe"})-[:Friend]->
             (:User {name: "Jane", surname: "Jones"})-[:Coworker {company: "Acme Inc."}]->(jr)
RETURN p
```

这样在数据库中就新创建了3个节点, 3个关系, 注意我们使用了变量`jr`作为节点`:User {name: "Jack", surname: "Roe"}`的引用, 返回的消息如下:

> Added 3 labels, created 3 nodes, set 7 properties, created 3 relationships, started streaming 1 records after 12 ms and completed after 12 ms.

**路径可视化**
![创建完整的路径](../assets/create-path.png)
