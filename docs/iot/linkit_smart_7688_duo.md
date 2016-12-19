> `LinkIt™ Smart 7688 Duo` 平台主要针对如下两个应用场景:
> - 智能家居的家电控制
> - 办公室设备控制

什么是联发科技LinkIt™ Smart 7688 开发平台
http://labs.mediatek.com/site/znch/developer_tools/mediatek_linkit_smart_7688/whatis_7688/index.gsp

## 连接到 LinkIt™ Smart 7688 Duo

下面介绍3种连接到 LinkIt™ Smart 7688 Duo 并执行管理任务的方法:

> ### 串口连接

还是一个USB转TTL串口线

![][1]

LinkIt™ Smart 7688 Duo 开发板一块, 价格100多一点.

![](https://cloud.githubusercontent.com/assets/725190/21251562/9947fb02-c389-11e6-947e-8bd8853d210c.png)

Breakout for LinkIt Smart 7688 扩展板: 为接入到本地局域网提供以太网接口.

![](https://cloud.githubusercontent.com/assets/725190/21251560/99463e3e-c389-11e6-82d4-c0257c36b25e.png)

针脚连接表

![图片描述](https://cloud.githubusercontent.com/assets/725190/21251557/99435228-c389-11e6-9828-b0164776fa79.png)

连接图

![图片描述](https://cloud.githubusercontent.com/assets/725190/21251559/99450e42-c389-11e6-8a94-fa2f0105c31f.png)

连接到串口

```
screen /dev/cu.usbserial 57600
```

下面是启动到内置的OpenWrt系统控制台的截图

![1c179820-3d41-4160-b600-848af2d00213](https://cloud.githubusercontent.com/assets/725190/21251529/69ae6f0c-c389-11e6-8807-4e1cbc6a6ba2.jpeg)

> ### 通过WIFI连接

![图片描述][2]

在地址栏输入 `http://mylinkit.local`, 按照提示执行密码设置等操作.

> ### 通过SSH

可以通过 `ssh root@mylinkit.local` 远程登录到 LinkIt™ Smart 7688 Duo. 密码就是刚才通过 `http://mylinkit.local` 设置的密码.  也可以通过串口线直接连接.


## WiFi LED状态灯

`LinkIt™ Smart 7688 Duo`没有显示输出接口, 因此用LED灯的闪烁来表示系统的状态, 下图说明了LED灯随着系统的状态变化而变化的过程. WIFI LED灯是橙色的.

![WiFi LED状态灯][3]

在 `AP` 模式下, Wi-Fi 等有两个状态:

- `LED 灯关闭`. 标识没有客户端连接到 LinkIt™ Smart 7688.
- `每秒闪烁3次, 间隔0.5秒, 不断重复`. 标识至少有一个客户端设备连接到 `LinkIt™ Smart 7688`.

在 `Station` 模式下, 有3个Wi-Fi LED灯状态:

- `LED 灯关闭`. LinkIt Smart 7688 无法连接到无线路由器, 或者超时.
- `每秒闪烁2次, 持续不断`. 成功连接到无线路由器.
- `根据数据传输状态进行闪烁`. LinkIt Smart 7688 成功连接到无线路由器, 并且在有数据传输的时候闪烁.
看这个视频: https://v.qq.com/x/page/f0356iwjvkd.html

## 示例程序

`LinkIt™ Smart 7688` 支持`C`, `Python`, 和`Node.js`的开发. 在`/IoT/examples`目录下有几个例子.

## 固件和Bootloader

固件可以通过Web UI和USB的方式更新, 更新过程, 成熟LED灯会闪烁约3分钟, 更新完成后系统会重启, 重启过程橙色LED等常量约30秒.

固件将开始上传至开发板. 请确认板子电源在固件更新过程完毕前无终断, 请注意Wi-Fi LED 将闪烁约 3 分钟 (固件更新中), 然后板子会重新启动, 这时LED 将点亮约 30 秒钟 (启动中). 最后, 板子进入AP 模式时, 就可以接受连接了.
用 Wi-Fi 搜索 LinkIt_Smart_7688_XXXXXX AP 并将其连接. 请注意, 当开发板连接上一个client设备时, Wi-Fi LED 将每秒闪烁3次. 现在重新加载mylinkit.local 网页, 设置新密码并登录, 您将在 Software information 看见新的固件版本

Youtube 的视频, 你懂得.

- 使用Web UI更新固件
https://www.youtube.com/watch?v=GtmbgN0VbyM
- 使用USB优盘更新固件
https://www.youtube.com/watch?v=FFPtL2ZKKD8
- 恢复工厂设置
https://www.youtube.com/watch?v=Y0r5c0mwm5Y
- 构建Bootloader
https://labs.mediatek.com/site/global/developer_tools/mediatek_linkit_smart_7688/training_docs/firmware_and_bootloader/build_bootloader/index.gsp
- 构建内核包
http://labs.mediatek.com/site/znch/developer_tools/mediatek_linkit_smart_7688/training_docs/firmware_and_bootloader/rebuild_existing_kernel_packages/index.gsp
- Bootloader 和内核Console
http://labs.mediatek.com/site/znch/developer_tools/mediatek_linkit_smart_7688/training_docs/firmware_and_bootloader/kernel_console/index.gsp
- 批量修改AP的SSID
https://www.youtube.com/watch?v=IwM6nlKeu2U
- 复制文件到`LinkIt Smarty 7688`
`scp ./helloworld root@mylinkit.local:/example/helloworld`


## 网络连接模式

> ### **AP模式**
>
> 作为一个路由器使用, 其他设备可以连接到 LinkIt™ Smart 7688 Duo, 请注意当连接上 LinkIt Smart 7688 AP 后, 您的计算机会无法访问因特网, 因为您的计算机加入了 LinkIt Smart 7688 Duo 形成的网络了, 如下图:


![](http://labs.mediatek.com/images/linkit7688/get_started/sign_in/gs_2_sign_in_both_apmode.png)

**LinkIt Smart 7688 Duo 开发板AP 模式**

从`AP`模式切换到`Station`模式

```
uci set wireless.sta.ssid=AAA
uci set wireless.sta.key=12345678
uci set wireless.sta.encryption=psk
uci set wireless.sta.disabled=0
# 提交修改
uci commit
# 重启WIFI
wifi
# 验证外网是否连通
ping –c 5 www.mediatek.com
```

> ### **Station 模式**
>
> 作为一个普通的电脑连接到其他路由器, 并可以连接到Internet.


从 `Station` 模式切换回 `AP` 模式:
```
# 禁用Station模式后自动转换为AP模式
uci set wireless.sta.disabled=1
# 提交修改
uci commit
# 重启WIFI
wifi
```

## 访问优盘和SD卡

当你插入优盘或者SD卡的时候, 他们的设备符号会出现在 `/Media/SD*` 或 `/Media/USB*`, 可以切换到其中访问数据.


## 扩展内存

内置的Flash 只有32MB的存储空间, 为了增加LinkIt 7688 的存储空间, 我们增加一个SD卡来存储数据.
https://mediateklabs.hackster.io/akashchandran30/expand-the-memory-in-linkit-7688-e37aa5

## 使用Web UI切换网络模式

切换为Station模式, 输入AAA的密码并重启系统

![图片描述][4]


## 开源项目

- 传感器监控项目
https://www.hackster.io/smerkousdavid/linkit-smart-one-sensor-monitoring-7e2741

## 链接

- [连接到系统](http://wiki.seeed.cc/LinkIt_Smart_7688_Duo/#connecting-to-the-embedded-operating-system)
- [小体积大能量——LinkIt Smart 7688 Duo评测](http://www.21ic.com/eva/MCU/201605/676931_4.htm)
- http://labs.mediatek.com/site/global/developer_tools/mediatek_linkit_smart_7688/whatis_7688/index.gsp
- http://www.seeed.cc/discover.html?t=linkit
- [通过 Arduino 控制外设和传感器](http://blog.csdn.net/c80486/article/details/51407564)


## 开发

- [开发者指南](http://labs.mediatek.com/fileMedia/download/87c801b5-d1e6-4227-9a29-b5421f2955ac)
- [LinkIt Smart 7688 配置工具](http://labs.mediatek.com/fileMedia/download/87c801b5-d1e6-4227-9a29-b5421f2955ac)

  [1]: https://segmentfault.com/img/bVGPgp
  [2]: https://segmentfault.com/img/bVGZ3Y
  [3]: https://segmentfault.com/img/bVG1FZ
  [4]: https://segmentfault.com/img/bVG1FW
