> **Edeliver 是一个基于 [deliver](https://github.com/gerhard/deliver) 的构建和部署工具.**
> **它提供了一个 Bash 脚本来帮助你构建, 部署, 以及执行热代码升级**

![Edeliver 部署关系图示][1]
**Edeliver 部署关系图示**
  
本文我们来聊一聊关于 Erlang/Elixir 的部署问题. 最原始的 Erlang, 我们要使用一大堆工具, 比如 reltool, 后来有了rebar, 还后来 relx 也出来了, Elixir 生态链中还有个 Exrm 打包工具, 是基于 relx 创建发布包的,  打包好了呢, 需要部署到服务器上去运行.

Exrm 打好的包, 一般情况, 我直接用 `scp` 复制到服务器上的某个目录, 然后在登录到远程服务器解压, 升级等操作. 而 Edeliver 让这个过程完全自动化, 提高了效率, 并且更加适用于中等规模的应用程序部署.


构建 Erlang/Elixir 发布, 并部署到产品服务器

```
mix edeliver build release --branch=feature
mix edeliver deploy release to production
mix edeliver start production
```

把上面三个单独的命令合并为一个

```
mix edeliver update production --branch=feature --start-deploy
```

## 配置

在项目目录下创建一个 `.deliver` 目录, 然后在其中添加 `config` 文件.

```
#!/usr/bin/env bash

APP="your-erlang-app" # name of your release

# 构建主机的地址, 用户名, 和构建目录
BUILD_HOST="build-system.acme.org"
BUILD_USER="build"
BUILD_AT="/tmp/erlang/my-app/builds"

# 测试目标主机的地址, 用户名, 和部署目录, 多个目标服务器地址用空格分隔开
STAGING_HOSTS="test1.acme.org test2.acme.org"
STAGING_USER="test"
TEST_AT="/test/my-erlang-app"

# 产品目标主机的地址, 用户名, 和部署目录, 多个目标服务器地址用空格分隔开
PRODUCTION_HOSTS="deploy1.acme.org deploy2.acme.org"
PRODUCTION_USER="production"
DELIVER_TO="/opt/my-erlang-app"
```

本质上, Edeliver 使用 SSH 和 scp 来部署发布. 如果需要无密码登陆, 可通过 `ssh-keygen` 工具创建密钥对, 把公钥添加到服务器的 `~/.ssh/authorized_keys` 文件中即可.



### 不同的部署目标不同的配置

对于一个需要实现 `Failover/Takeover` 分布式应用程序来讲, 每个节点的配置是不一样的, 这样在部署的时候就要去编写不同的配置文件.

Edeliver 提供了这样一个功能



## 构建

**必须在和目标系统类似 (建议完全一样的系统) 的系统上构建. ** 如果想在 Linux 上部署产品系统, 那么发布也应该在 Linux 系统上构建(建议私用相同厂商的 Linux 发型版). 不要求在产品系统上安装 Erlang/OTP 和 Elixir 运行时, 他们会随打包系统自动包含进发布包分发到产品系统上. 下面的配置变量是必须的:

Name        | Description 
----------- | ----------- 
`APP`       | 发布的应用程序名称
`BUILD_HOST`| 构建发布包的主机名称或IP地址
`BUILD_USER`| 构建主机上的本地用户名称
`BUILD_AT`  | 在构建主机上构建发布包的目录.

构建完成后, 发布包会被复制到你的本地 `.deliver/releases` 目录. 然后通过下面的命令发布到产品服务器. 如果成功编译和生成发布包. 发布包会从构建主机复制到发布存储, 默认发布存储的位置在项目根目录下的 `.deliver` 目录的 `releases` 子目录, 但可以通过配置环境变量来自定义, 例如:

> 准确的讲, 是复制到 `RELEASE_STORE` 环境变量指定的位置, 如果设置了该环境变量的话. 如果没有设置 `RELEASE_STORE` 环境变量, 默认为项目根目录的 `.deliver/releases` 子目录.

```
RELEASE_STORE=/tmp/.deliver                                       # 本地目录
RELEASE_STORE=user@releases.acme.org:/releases                    # 远程主机
RELEASE_STORE=s3://AWS_ACCESS_KEY_ID@AWS_SECRET_ACCESS_KEY:bucket # 亚马逊S3
```

发布使用`RELEASE_DIR=`环境变量来确定发布归档包的位置. 如果未设置, 默认目录包含`RELEASES`文件的目录. 比如: 如果`$APP=myApp`, `RELEASES`的位置为`rel/myApp/myApp/releases/RELEASE`. 关于完整的 edeliver 命令, 可以通过运行 `mix help edeliver` 查看:

## 使用 Edeliver 

别自己创建 `$BUILD_AT`, `$TEST_AT`, `$DELIVER_TO` 目录. `config/*.secret.exs` 应该放在`$CONFIG_AT`目录, `*.vm.args` 文件处理节点名称, 集群等配置参数


### 开始构建

```
mix edeliver build release --version=0.0.1
```

![图片描述][2]

### 部署第一个发布版本

```
mix edeliver deploy release to production --version=0.0.1 --start-deploy
```

### 构建一个热升级版本

提升版本号, 在 `mix.exs` 提升项目的版本号

```
git commit -m 'Bump version to 0.0.2'              # 提交
git tag 0.0.2                                      # 打TAG
mix edeliver build upgrade --from=0.0.1 --to=0.0.2 # 构建
```

### 部署一个升级版本

```
mix edeliver deploy upgrade to production --version=0.0.2
```

```
# 构建从 v1.0 到 v2.0 的升级包, 并部署升级到产品环境

mix edeliver build upgrade --from=v1.0 --to=v2.0
mix edeliver deploy upgrade to production

# 或者, 如果在发布存在有旧的发布版本, 可以用那个旧的发布版本来构建升级而不是Git的旧版修订/Tag

mix edeliver build upgrade --with=v1.0 --to=v2.0
mix edeliver deploy upgrade to production

# 运行Ecto 移植

mix edeliver migrate production

# 或者使用 --run-migrations 选项在升级期间自动运行
```

## 排错

构建主机需要安装好 Erlang/Elixir 的构建环境, 推荐使用 `kerl` 编译 Erlang, `kiex` 安装 Elixir. 然后把下面两行脚本添加到用户目录的 `.profile` 文件中. 目的是让 Edeliver 能够找到 Elixir 工具的位置

```
. /home/ycc/.kerl/installs/18.3_systemtap/activate
test -s "$HOME/.kiex/scripts/kiex" && source "$HOME/.kiex/scripts/kiex"
kiex use 1.2.5 > /dev/null 2>&1
```

## 关于 `-V`, `--verbose` 参数

我们在构建发布包的时候, 有时候会出错, 可以通过构建是添加 `-V` 参数来看到构建过程的大量信息, 包括 git 本地工作区的 `HEAD` 的指向, 编译的详细输出(`mix compile`的输出), 默认情况 Edeliver 会从拉取 `HEAD` 指向的分支来构建发布.

## 集群设置

> 在`vm.args`设置`节点名称`和 `cookie`

![参数配置][3]

> 在配置文件 `prod.exs` 中配置 `:kernel` 模块的 `:sync_nodes_mandator` 和 `:sync_nodes_optional` 选项

```elixir
use Mix.Config
config :kernel, distributed: [{:opencv_thumbnail_server, 5000, [
  :"node1@192.168.212.45",
  :"node2@192.168.212.45",
  :"node3@192.168.212.45",
  :"node4@192.168.212.45",
  :"node5@192.168.212.45"
]}]
config :kernel, inet_dist_listen_min: 9100
config :kernel, inet_dist_listen_max: 9105
config :kernel, sync_nodes_mandatory: [
  :"node2@192.168.212.45",
  :"node3@192.168.212.45"
]
config :kernel, sync_nodes_optional: [
  :"node4@192.168.212.45",
  :"node5@192.168.212.45"
]
config :kernel, sync_nodes_timeout: 10000
```

上传到服务器的压缩包解压后, 需要修改,下面两个文件的节点名字和 `distributed` 配置.

```
./releases/0.0.1/sys.config
./releases/0.0.1/vm.args
```

## 其他管理命令

```
# 显示发布版本清单
$ mix edeliver show releases
```

![图片描述][4]

## 总结

大体上部署过程就是三个命令

```
mix edeliver build release --verbose                # 构建发布包
mix edeliver deploy release to production --verbose # 部署(上传, 解压)到产品服务器
mix edeliver start production --verbose             # 启动产品服务器
```

如果上线前还需要部署到 staging 服务器进行测试, 或者灰度发布

## 参考资料

[Fast Continuous Deployment of an Elixir Gameserver](https://www.youtube.com/watch?v=RoT8RnQHvgo)
[Real World Elixir Deployment](http://www.slideshare.net/petegamache/real-world-elixir-deployment)
[Edeliver API Reference](https://hexdocs.pm/edeliver/api-reference.html)


  [1]: https://segmentfault.com/img/bVwAPt
  [2]: https://segmentfault.com/img/bVwAwp
  [3]: https://segmentfault.com/img/bVwAvk
  [4]: https://segmentfault.com/img/bVxFzO