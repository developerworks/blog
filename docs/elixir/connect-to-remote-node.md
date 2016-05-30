### Erlang内置的SSH应用

通过 Erlang 提供的 SSH应用程序连接到远程 Erlang 控制台

生成客户端/服务器秘钥

```
mkdir client
ssh-keygen -t rsa -f /tmp/client/id_rsa

mkdir server
ssh-keygen -t rsa -f /tmp/server/ssh_host_rsa_key
```

把客户端的公钥复制到服务器秘钥目录下

```
cp /tmp/client/id_rsa.pub /tmp/server/authorized_keys
```

启动服务器, 同时设置SSH的系统目录和用户目录. 如果不设置, 系统目录默认为`/etc/ssh`(需要root权限), 用户目录为`~/.ssh`

```
ssh:start().
ssh:daemon(11111, [{system_dir, "/tmp/server"}, {user_dir, "/tmp/server"}]).
```

![图片描述][1]


连接到Erlang节点

```
mkdir .ssh_erlang

ssh 127.0.0.1 -p 11111 -i \
    /tmp/client/id_rsa -o UserKnownHostsFile=.ssh_erlang/known_hosts
```

### 通过SSH端口转发监控远程节点

在本地节点启动 observer 启动监视远程节点的运行时信息, 本地启动SSH端口转发, 把 remote_ip 替换成实际的远程节点IP地址.

```
ssh -vv -N -L 9001:localhost:9001 -L 4369:localhost:4369 ubuntu@192.168.8.129
```

启动本地 observer

```
erl -sname debug@localhost -setcookie 123456 -run observer
```


注意, 远程应用程序需要添加 `:runtime_tools` 作为依赖, 否则会出现如下错误信息:

```
14:15:12.548 [error] [node: :test_server@localhost, call: {:observer_backend, > :sys_info, []}, reason: {:badrpc, :nodedown}]
```

`:observer_backend` 模块存在于应用程序`runtime_tools`中.

![图片描述][2]

## 参考

- https://www.erlang-solutions.com/blog/secure-shell-for-your-erlang-node.html


  [1]: https://segmentfault.com/img/bVvKr0
  [2]: https://segmentfault.com/img/bVvJ2b