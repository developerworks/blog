> `问`: 为什么需要串口调试?
> `答`: 因为显示器直连Respberry Pi的HDMI的启动信息无法向上滚动, 无法查看完整的启动信息, 非常不方便硬件调试.

## 如何使用串口连接到目标板

默认 `iex` 控制台显示在连接到`HDMI`端口的显示器上. 这对于新手来说是比较友好的, 因为只需要用`HDMI线`把树莓派的`HDMI输出端口`和`显示器的输入端口`连接起来就可以了. 但是对于调试问题, 高级开发工流程, 通常期望通过[FTDI 线](https://www.sparkfun.com/products/9717)或`USB TTL线`把开发电脑连连接到目标板的串口. 这让我们能够通过终端模拟器(比如`screen`)与目标设备进行交互.

本文以上一篇文章 [Elixir 下开发嵌入式系统开发: 小试身手](https://segmentfault.com/a/1190000007733527) 为基础进行实际的上手操作, 如果你不了解, 可以先阅读上一篇文章.

默认IEx 终端显示输出到HDMI接口.

可以切换到`UART`(通用异步收发器), 下面讲述这个切换过程.

## 原材料准备

USB 转 TTL 调试线

![图片描述][1]

根据[Mac OS X上使用USB转串口线连接树莓派](http://shumeipai.nxez.com/2015/09/06/mac-os-x-rpi-serial-connection.html)安装驱动.

![图片描述][2]

除了OSX系统外, 还有Android和Windows的驱动可以下载. 选择合适自己的即可. Windows 用户请参考 [Windows下用串行连接控制树莓派](http://shumeipai.nxez.com/2014/05/04/under-windows-serial-connection-control-raspberry-pi.html).


## 步骤一: 配置

参考 [Elixir 下开发嵌入式系统开发: 小试身手](https://segmentfault.com/a/1190000007733527), 从Github Fork代码库.

配置文件覆盖, 该配置所指向的目录会覆盖系统文件对应的文件. 在 `hello_iot/apps/fw/config/config.exs` 配置文件中增加如下配置:

```
# ------------
# 增加覆盖目录, 覆盖默认系统文件
# ------------
config :nerves, :firmware,
  rootfs_additions: "config/rootfs-additions"
```

在 `hello_iot/apps/fw/config/rootfs-additions` 下创建 `erlinit.config` 文件, 该文件可以从 [这里](https://github.com/nerves-project/nerves_system_rpi3/blob/master/rootfs-additions/etc/erlinit.config) 下载.

![图片描述][3]

把 `-c tty1` 修改为 `-c ttyS0`

## 步骤二: 烧制

下载依赖, 编译, 制作固件, 烧制固件.

```
# 切换到固件目录
cd hello_iot/apps/fw
# 下载依赖
mix deps.get
# 编译
mix compile
# 固件打包
mix firmware
# 烧制固件
mix firmware.burn
```


## 步骤三: 连接

用USB串口线把Mac和Respberry Pi连接起来. 如下图:

Respberry Pi 3 的串口线连接线示意图
![图片描述][4]

GPIO针脚15, 接绿线TXD, 14针脚, 接白线RXD, 黑色为GND地线, 我用的Mini USB的外接电源, 所以这里`红色的供电针脚不接, 实际接线图如下`:

![图片描述][5]

把USB插入MAC笔记本的USB端口, 并执行如下命令:


```
screen /dev/tty.usbserial 115200
```

其中 `115200` 为波特率

重启树莓派就可以看到启动信息输出到了开发机的屏幕上了.

![ttys0](https://cloud.githubusercontent.com/assets/725190/21129549/25fb5e26-c13d-11e6-9687-2faba9a5d596.gif)

`screen` 的输出历史问题:

在 `~/.screenrc` 文件中添加下面一行:

```
defscrollback 10000
```

然后输入`Ctrl-a` `ESC`, 按翻页, 上下键即可查看输出历史.


## 参考资料

- [串口驱动下载地址](http://www.prolific.com.tw/US/ShowProduct.aspx?pcid=41&showlevel=0041-0041)
- [Mac OS X上使用USB转串口线连接树莓派](http://shumeipai.nxez.com/2015/09/06/mac-os-x-rpi-serial-connection.html)
- [Windows下用串行连接控制树莓派](http://shumeipai.nxez.com/2014/05/04/under-windows-serial-connection-control-raspberry-pi.html)

## 系列文章

- [Elixir 下开发嵌入式系统开发: 小试身手](https://segmentfault.com/a/1190000007733527)
- [Elixir 下开发嵌入式系统开发: 串口调试](https://segmentfault.com/a/1190000007785009)


  [1]: https://segmentfault.com/img/bVGPgp
  [2]: https://segmentfault.com/img/bVGPgK
  [3]: https://segmentfault.com/img/bVGPh9
  [4]: https://segmentfault.com/img/bVGPjc
  [5]: https://segmentfault.com/img/bVGPjJ
