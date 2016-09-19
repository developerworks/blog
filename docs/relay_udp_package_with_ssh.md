> 我们在阿里云开了一台服务器, 合作方那边的设备需要向服务器6000端口不停的发送UDP数据包, 我通过SSH和SOCAT把对服务器6000端口的数据包转发到我本地Macbook Pro的6000端口上以方便开发和测试工作. `目的是解决SSH仅支持TCP的问题`

## 网络

![图片描述][1]

其中`server_ip`为阿里云SSH服务器IP地址或域名. 使用时可以替换为你自己的服务器IP地址或域名.

## SSH端口转发需要的工具

SSH服务器, SOCAT工具

## SSH的TCP端口转发

在`client`端运行如下命令

```
ssh -vv -N -R 8000:127.0.0.1:8000 root@server_ip
```

上面的命令在客户端和SSH服务器之间建立了一根相互连接的隧道. 隧道客户端地址为本机地址 `127.0.0.1:8000`, 服务器端地址为 `0.0.0.0:8000`(监听在`0.0.0.0`表示绑定服务器的所有接口地址, 这样客户端能够访问服务器端的所有接口地址)


## SSH的UDP端口转发

> 由于SSH并不直接支持UDP, 因此我们用到了一个UDP中继工具 `socat` (SOcket CAT)

把服务器在UDP 6000端口上收到的UDP数据包转发到服务器本地8000 TCP端口上. 在 `server` 端运行如下命令:

```
socat -T15 udp4-recvfrom:6000,reuseaddr,fork tcp:localhost:8000
```

此时UDP数据包会通过SSH隧道到达我本机笔记本的8000端口上, 我们还需要开一个转换器, 让本地8000端口接收到的TCP数据包转换为UDP数据包, 并发送到本地 6000 UDP端口上. 在 `client` 端运行如下命令:

```
socat tcp4-listen:8000,reuseaddr,fork udp:127.0.0.1:6000
```

现在我的本机就看到了, 从服务器6000端口, 通过SSH隧道过来的UDP数据包了.

> 对于SSH的远程转发, 需要在SSH服务器端的`/etc/ssh/sshd_config`配置文件中增加`GatewayPorts yes`

下图为erlang编写的一个简单udp服务器接收到数据包时的输出.

![图片描述][2]

## 参考资料

[SSH Port Forwarding for TCP and UDP Packets](http://www.digitalinternals.com/network/ssh-port-forwarding-tcp-udp/365)
[Performing UDP tunneling through an SSH connection](http://www.qcnetwork.com/vince/doc/divers/udp_over_ssh_tunnel.html)


  [1]: https://segmentfault.com/img/bVyxzd
  [2]: https://segmentfault.com/img/bVyv59
