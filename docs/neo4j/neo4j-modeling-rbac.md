## 建模RBAC权限管理系统

![clipboard.png](https://segmentfault.com/img/bV3aG1)
![clipboard.png](https://segmentfault.com/img/bV3aNX)

对于CRUD操作, 角色和资源有4条关系. 分别是`CREATE`,`UPDATE`,`READ`,`DELETE`. 如果对应的操作权限不存在, 标识没有权限

这里用ID为 `c508b480-082e-11e8-9f0c-b8e8563f0d3a`的用于有两条操作权限记录. 这样我们就可以定义`具有某个角色的用户在指定的资源上拥有什么权限`这种判断, 来达到控制用户对资源的访问.

## Cypher 节点关系创建

```
// 创建角色(管理员,运维,普通用户)
CREATE (:Role {name: "Operator"}),(:Role {name: "Admin"}),(:Role {name: "User"});

// 创建资源节点
CREATE (resource:Resource {path: "/", name: "根目录"})
  RETURN resource;


// 普通用户在资源 / 上有, READ, CREATE权限
MATCH (role:Role {name: "User"}), (resource:Resource {path: "/"})
  CREATE (role)-[op:READ {created_at: timestamp()}]->(resource);

MATCH (role:Role {name: "User"}), (resource:Resource {path: "/"})
  CREATE (role)-[op:CREATE {created_at: timestamp()}]->(resource);


// 查询一个用户在某个资源山的操作权限列表
MATCH (u:User {name: "测试用户"})-[r:HAS_ROLE]->(role:Role {name: "User"})-[op]->(resource)
  WHERE u.roleId = role.uuid
  RETURN u.name,op.name, resource.path;
```

## 抽象表达

在实际的查询需要把用户(`u`), 角色(`r`), 资源(`s`)参数化. 我们用一个函数来表达

$$P = f(u, r, s)$$

> `P`为具有某个角色(`r`)的用户(`u`)在资源`s`上的权限集合.

## Cypher 查询语句


```
// 权限查询
MATCH (u:User)
        -[:HAS_ROLE]->(r:Role {name: "User"})
        -[op]->(resource:Resource {path: "/"})
RETURN u.uuid as user_id,
       r.name as role_name,
       resource.path as resource_path,
       type(op) as operation;
```

在IDEA中的Neo4j插件, 支持单一语句执行(在`.cypher`文件中可以保持多条Cypher查询语句, 点击一条语句会有绿色的框, 然后在点击执行按钮会执行这条被选中的Cypher语句).

![clipboard.png](https://segmentfault.com/img/bV3aQ7)

在Neo4j Browser中的效果

![clipboard.png](https://segmentfault.com/img/bV3aRP)

## 参考资料

- [RBAC权限管理模型](https://www.xiaoman.cn/detail/150)
