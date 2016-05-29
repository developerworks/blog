> DTrace 是一把瑞士军刀

这一篇是 [Erlang/Elixir: 在Ubuntu上通过SystemTap对Erlang进行运行时的动态追踪](https://segmentfault.com/a/1190000005163074) 的姊妹篇

![DTrace 是一把瑞士军刀][1]

## 动态系统追踪的工作流

![动态系统追踪的工作流][2]

## DTrace 的工作流

![dtrace.png][3]

## 基本配置

OSX是默认支持DTrace的, 所以不需要什么修改就可以使用, 只需要在Erlang的编译选项添加`--with-dynamic-trace=dtrace` 即可.  这里为了方便我使用了`kerl`作为Erlang构建的管理工具. 并在`~/.kerlrc`配置文件中指定了Erlang的编译选项, 其内容如下:

```
KERL_CONFIGURE_OPTIONS="--with-dynamic-trace=dtrace --disable-debug --without-javac --enable-shared-zlib --enable-dynamic-ssl-lib --enable-hipe --enable-smp-support --enable-threads --enable-kernel-poll --with-wx"
```

关于`kerl`的详细使用说明,请参考Github上的[项目主页](https://github.com/kerl/kerl). 或[Erlang版本管理工具: Kerl](https://segmentfault.com/a/1190000004909357)这篇文章

开始构建

```
$ kerl build 18.3 18.3_dtrace
```

安装

```
$ kerl install 18.3_dtrace ~/.kerl/installs/18.3_dtrace
```

激活, 把下面的代码添加到`.bashrc`,或`.zshrc`(如果使用了`oh-my-zsh`)

```
$ . ~/.kerl/installs/18.3_dtrace/activate
```

输入 iex 验证一下, 我们看到了标记行上面多了一个`[dtrace]`

![DTrace 标记][4]

## 试着开始第一步


下面我们来创建一个名为 `process_signal_trace.d` DTrace文件, 内容如下

```
#!/usr/sbin/dtrace -qs
erlang*:::process-spawn
{
  printf("pid %s mfa %s\n", copyinstr(arg0), copyinstr(arg1));
}
erlang*:::process-exit
{
  printf("pid %s reason %s\n", copyinstr(arg0), copyinstr(arg1));
}
erlang*:::process-exit_signal
{
  printf("sender %s -> pid %s reason %s\n", copyinstr(arg0), copyinstr(arg1), copyinstr(arg2));
}
```

修改执行权限

```
$ chmod +x /tmp/process_signal_trace.d
```

运行该脚本(运行dtrace需要root权限)

```
$ sudo /tmp/process_signal_trace.d 
```

然后在另一个终端启动IEx会话, 并输入一些创建进程的代码:

```
iex(8)> f = fn() -> :timer.sleep(2000) end   
#Function<20.50752066/0 in :erl_eval.expr/5>
iex(9)> spawn(fn() -> :timer.sleep(2000) end)
#PID<0.71.0>
iex(10)> spawn(fn() -> :timer.sleep(2000) end)
#PID<0.73.0>
iex(11)> spawn(fn() -> :timer.sleep(2000) end)
#PID<0.75.0>
```

然后回到运行 DTrace 的终端, 我们看到了探测点的输出了. 

![DTrace 探测点输出][5]

## 深入一点: 系统调用追踪

> 对于Erlang 的探测点, 可以通过命令 `sudo dtrace -l|grep beam.smp` 列举出来

有了上面的基础之后, 我们来分析与一下, [这篇](http://www.evanmiller.org/elixir-ram-and-the-template-of-doom.html)几天前我没怎么看懂的文章

首先下载文章中使用的脚本, 并修改执行权限, 并启动 DTrace:

```
# 下载

wget https://raw.githubusercontent.com/evanmiller/tracewrite/master/tracewrite.d -O /tmp/tracewrite.d

# 修改执行权限

chmod +x /tmp/tracewrite.d

# 启动, 其中-p参数是IEx的进程ID, 可以通过 ps aux|grep iex 查找
# 更好的方式是在IEx中执行 :os.getpid 的到IEx的操作系统进程ID

sudo -s /tmp/tracewrite.d -p 60838
```

在IEx会话中执行

```elixir
{:ok, file} = :file.open("/tmp/something.txt", [:write, :raw])
:file.write(file, :re.replace("Hello & Goodbye", "&", "&amp;"))
```

代码本身没什么特别, 只是打开一个文件,向其中写入一些文字而已. 回到 DTrace 输出终端, 我们看到:

```
iex(5)> :re.replace("Hello & Goodbye", "&", "&amp;")
["Hello ", ["&", "amp;"] | " Goodbye"]
```

`:re.replace("Hello & Goodbye", "&", "&amp;")` 会创建一个嵌套列表 `["Hello ", ["&", "amp;"] | " Goodbye"]`, 因此,对于`write(0x3, "Hello & goodbye\0", 0x13)` 会被展开为 `writev(0x1A, 0x1A5405F8, 0x4)`, **0x1A5405F8** 这个十六进制的东西是个什么鬼?  它实际上是这个向量的地址, 这个地址包含了另外4个地址, 如下所示:

![通过 DTrace 追踪 Elixir 对 `writev` 的系统调用][6]

> 关于内存地址

```
# amp;
0x0b49 'a' 
0x0b4a 'm'
0x0b4b 'p'
0x0b4c ';'

# Hello & Goodbyte
0x0d78 'H'
0x0d79 'e'
0x0d7a 'l'
0x0d7b 'l'
0x0d7c 'o'
0x0d7d ' '
0x0d7e '&'
0x0d7f ' '
0x0d80 'G'
0x0d81 'o'
0x0d82 'o'
0x0d83 'o'
0x0d84 'd'
0x0d85 'b'
0x0d86 'y'
0x0d87 'e'
```

从地址我们可以看出, 这个`:re`生成的嵌套列表, 只是包含三个`原始字符串`, 和一个额外的`替换串`的地址指针, 它并没有创建或修改字符串. 这样的数据结构被称为 I/O 列表, 它的设计目的是: 当向磁盘或网络写入的时候, 利用 `writev` 系统调用来最小化数据的复制.

> 我们在Erlang看到了很多函数参数或返回类型为 `iolist()` 的类型, 我们在这里正好说明了, `iolist` 具体表示了什么.

当然, 指针并不是万能药, 有的时候, 复制要比指针的连接更高效. 让我用 DTrace 来探索 Erlang 虚拟机的实现. 

## 追踪限制(Limit)

```
:file.write(file, Enum.map(1..14, fn(_) -> "Foobar" end))
:file.write(file, Enum.map(1..15, fn(_) -> "Foobar" end))
```

两行代码都是向文件中写入一个I/O列表. 第一行代码重复14次, 第二行重复15次, 你可能猜这两行代码可能会调用相同的系统调用, 实际上却不是这么回事的. 第一行代码用14个元素的向量调用 writev, 第二行代码把单个 I/O 列表扁平化为一个单独的内存大对象, 然后调用 `write` 写入数据.

DTrace 的输出可以证明这一点

![writev 的限制][7]

> 如果你试着增加或减小单个字符串的长度, 你会发现触发这种系统调用差异的只是简单的IO列表中的元素数量, 而不是结果字符串的总体大小. 如果在列表中大于等于了15个元素, Erlang 会吧IO列表中的每个元素连接成为一个连续的内存块, 并调用 `write` 系统调用去写数据. 如果你在探究一下Erlang虚拟机的源码, 你会看到这是通过常量 [SMALL_WRITE_VEC](https://github.com/erlang/otp/search?utf8=%E2%9C%93&q=SMALL_WRITE_VEC) 进行控制的. 

> 短小字符串是在其出现的位置直接初始化的

在来看一下 14 个 "Foobar" 字符串的地址

```
0    393                    writev:return Writev data 1/14: (6 bytes): 0x0000000021942ec0 Foobar
0    393                    writev:return Writev data 2/14: (6 bytes): 0x0000000021940f50 Foobar
0    393                    writev:return Writev data 3/14: (6 bytes): 0x0000000021941710 Foobar
0    393                    writev:return Writev data 4/14: (6 bytes): 0x0000000021941750 Foobar
0    393                    writev:return Writev data 5/14: (6 bytes): 0x0000000021941790 Foobar
0    393                    writev:return Writev data 6/14: (6 bytes): 0x0000000021942fb8 Foobar
0    393                    writev:return Writev data 7/14: (6 bytes): 0x0000000021942ff8 Foobar
0    393                    writev:return Writev data 8/14: (6 bytes): 0x0000000021941988 Foobar
0    393                    writev:return Writev data 9/14: (6 bytes): 0x00000000219419c8 Foobar
0    393                    writev:return Writev data 10/14: (6 bytes): 0x0000000021941a08 Foobar
0    393                    writev:return Writev data 11/14: (6 bytes): 0x0000000021941a48 Foobar
0    393                    writev:return Writev data 12/14: (6 bytes): 0x0000000021943038 Foobar
0    393                    writev:return Writev data 13/14: (6 bytes): 0x0000000021943078 Foobar
0    393                    writev:return Writev data 14/14: (6 bytes): 0x00000000219430b8 Foobar
```

注意到内存地址了么? 14 个字符串 "Foobar" 分布在内存的不同位置, 如果我们把字符串 "Foobar" 移动到 Elixir 闭包的外面, 又是如何呢?

```
# 把 foobar 移到闭包外
foobar = "Foobar"
 :file.write(file, Enum.map(1..14, fn(_) -> foobar end))
```

```
0    393                    writev:return Writev data 1/14: (6 bytes): 0x0000000021940b98 Foobar
0    393                    writev:return Writev data 2/14: (6 bytes): 0x0000000021940b98 Foobar
0    393                    writev:return Writev data 3/14: (6 bytes): 0x0000000021940b98 Foobar
0    393                    writev:return Writev data 4/14: (6 bytes): 0x0000000021940b98 Foobar
0    393                    writev:return Writev data 5/14: (6 bytes): 0x0000000021940b98 Foobar
0    393                    writev:return Writev data 6/14: (6 bytes): 0x0000000021940b98 Foobar
0    393                    writev:return Writev data 7/14: (6 bytes): 0x0000000021940b98 Foobar
0    393                    writev:return Writev data 8/14: (6 bytes): 0x0000000021940b98 Foobar
0    393                    writev:return Writev data 9/14: (6 bytes): 0x0000000021940b98 Foobar
0    393                    writev:return Writev data 10/14: (6 bytes): 0x0000000021940b98 Foobar
0    393                    writev:return Writev data 11/14: (6 bytes): 0x0000000021940b98 Foobar
0    393                    writev:return Writev data 12/14: (6 bytes): 0x0000000021940b98 Foobar
0    393                    writev:return Writev data 13/14: (6 bytes): 0x0000000021940b98 Foobar
0    393                    writev:return Writev data 14/14: (6 bytes): 0x0000000021940b98 Foobar
```

嗯哈,14 个同样的地址, 你应该懂了. 对于闭包内的字符串, 闭包每次执行的时候创建一个新的字符串, 当在闭包内引用外部的变量时, 每次执行闭包, 它只是创建了指向同一个字符串的引用.

Erlang 把字符串长度大于等于64字节的认为是比较大的二进制对象, (这是由 [ERL_ONHEAP_BIN_LIMIT](https://github.com/erlang/otp/blob/e1489c448b7486cdcfec6a89fea238d88e6ce2f3/erts/emulator/beam/erl_binary.h#L31) 魔数控制的. 如果大常量出现在已编译模块中, 他们将会被分配在`共享堆`上面, 并重新计数(re-counted)

## 少就是多

`ERL_SMALL_IO_BIN_LIMIT` 被硬编码为 `ERL_ONHEAP_BIN_LIMIT` 的4倍, 即为: 256字节.

在 writev 向量中的元素小于 `ERL_SMALL_IO_BIN_LIMIT`  的会被连接为一个大的字符串, 大于 `ERL_SMALL_IO_BIN_LIMIT` 会保留不变. 比如如向量中的每个元素的字节长度刚好小于 256 字节, 那么该向量会被连接成一个更大的单个字符串, 否则保持不变, 据此可以对代码中的二进制处理进行优化.

## 探测点

这里是对[官方文档](http://erlang.org/doc/apps/runtime_tools/DTRACE.html#id59725)探测点的的简单说明.  详细的探测点信息可以通过命令 `sudo dtrace -l | grep erlang` 得到.

列表比较长, 首先通过 `sudo dtrace -l | head -n 10` 命令得到四个列的名字.

![首先通过 `sudo dtrace -l | head -n 10` 命令得到四个列的名字][8]

在我的系统上, 目前提供了 1086 个探测点, 通过下面的命令可以得到

```
sudo dtrace -l|grep beam.smp |wc -l
```

### 探测点列表

```
# 所有非用户追踪的探测点, 目前有 133 个, 可以通过下面的命令查看完整的列表
➜ sudo dtrace -l | grep beam.smp | head -n 133
```

```
# 用户追踪的探测点 953 个, 可以通过下面的命令获取
➜ sudo dtrace -l | grep beam.smp | grep user_trace
```

```
# 进程相关的探测点
➜ sudo dtrace -l | grep beam.smp | grep process- | head -n 133

35811 erlang60838          beam.smp              erts_do_exit_process process-exit
35812 erlang60838          beam.smp                  send_exit_signal process-exit_signal
35813 erlang60838          beam.smp            erts_dsig_send_exit_tt process-exit_signal-remote
35814 erlang60838          beam.smp                     grow_new_heap process-heap_grow
35815 erlang60838          beam.smp                          do_minor process-heap_grow
35816 erlang60838          beam.smp                  major_collection process-heap_grow
35817 erlang60838          beam.smp                   shrink_new_heap process-heap_shrink
35818 erlang60838          beam.smp                    erts_hibernate process-hibernate
35819 erlang60838          beam.smp                begin_port_cleanup process-port_unblocked
35820 erlang60838          beam.smp            erts_port_resume_procs process-port_unblocked
35821 erlang60838          beam.smp                      process_main process-scheduled
35822 erlang60838          beam.smp                erl_create_process process-spawn
35823 erlang60838          beam.smp                          schedule process-unscheduled
```

```
# 分布式相关的探测点

➜ sudo dtrace -l | grep beam.smp | grep dist- | head -n 133

35697 erlang60838          beam.smp               send_nodes_mon_msgs dist-monitor
35698 erlang60838          beam.smp                 dist_port_command dist-output
35699 erlang60838          beam.smp                dist_port_commandv dist-outputv
35700 erlang60838          beam.smp                    erts_dsig_send dist-port_busy
35701 erlang60838          beam.smp           erts_dist_port_not_busy dist-port_not_busy
35702 erlang60838          beam.smp                    erts_dsig_send dist-port_not_busy
```

```
# 消息相关的探测点

➜ sudo dtrace -l | grep beam.smp | grep message- | head -n 133

35792 erlang60838          beam.smp                     queue_message message-queued
35793 erlang60838          beam.smp           erts_queue_dist_message message-queued
35794 erlang60838          beam.smp                      process_main message-receive
35795 erlang60838          beam.smp            erts_dsig_send_reg_msg message-send
35796 erlang60838          beam.smp                erts_dsig_send_msg message-send
35797 erlang60838          beam.smp                 erts_send_message message-send
35798 erlang60838          beam.smp            erts_dsig_send_reg_msg message-send-remote
35799 erlang60838          beam.smp                erts_dsig_send_msg message-send-remote
```

```
# 端口相关的探测点

➜ sudo dtrace -l | grep beam.smp | grep port- | head -n 133

35802 erlang60838          beam.smp                     set_busy_port port-busy
35803 erlang60838          beam.smp                set_port_connected port-command
35804 erlang60838          beam.smp            call_deliver_port_exit port-command
35805 erlang60838          beam.smp                  erts_port_output port-command
35806 erlang60838          beam.smp                set_port_connected port-connect
35807 erlang60838          beam.smp               call_driver_control port-control
35808 erlang60838          beam.smp            erts_deliver_port_exit port-exit
35809 erlang60838          beam.smp                     set_busy_port port-not_busy
35810 erlang60838          beam.smp                         open_port port-open
```

### 关于Erlang探测点的详细说明

源码是包含最多信息的
 
- 请参考`~/.kerl/builds/18.3/otp_src_18.3/erts/emulator/beam/erlang_dtrace.d`
- 或 [erts/emulator/beam/erlang_dtrace.d](https://github.com/erlang/otp/blob/e1489c448b7486cdcfec6a89fea238d88e6ce2f3/erts/emulator/beam/erlang_dtrace.d)

## 参考资料

- [Elixir RAM and the Template of Doom](http://www.evanmiller.org/elixir-ram-and-the-template-of-doom.html)
- [Profiling Erlang Applications using DTrace](https://speakerdeck.com/mrallen1/profiling-erlang-applications-using-dtrace)
- http://dtracehol.com/
- http://dtrace.org/guide
- [dtracebook](http://dtracebook.com/index.php/Main_Page)
- [/doc/apps/runtime_tools/DTRACE.html](http://erlang.org/doc/apps/runtime_tools/DTRACE.html)
- [/doc/apps/runtime_tools/SYSTEMTAP.html](http://erlang.org/doc/apps/runtime_tools/SYSTEMTAP.html)


  [1]: https://segmentfault.com/img/bVvLvi
  [2]: https://segmentfault.com/img/bVvOc8
  [3]: https://segmentfault.com/img/bVvOQu
  [4]: https://segmentfault.com/img/bVvLlb
  [5]: https://segmentfault.com/img/bVvLHL
  [6]: https://segmentfault.com/img/bVvLDk
  [7]: https://segmentfault.com/img/bVvLHf
  [8]: https://segmentfault.com/img/bVvLWz