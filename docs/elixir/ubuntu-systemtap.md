> 这一篇是 [Erlang/Elixir: 在OSX上通过DTrace对Erlang进行运行时的动态追踪](https://segmentfault.com/a/1190000005156906) 在Ubuntu上的姊妹篇

## SystemTap 的工作流

![SystemTap 的工作流][3]

## Ubuntu 添加 SystemTap 支持

> SystemTap 是监控和跟踪运行中的Linux 内核的操作的动态方法. 这句话的关键词是动态. 因为SystemTap 没有使用工具构建一个特殊的内核, 而是允许您在运行时动态地安装该工具.

### 安装 SystemTap

```
sudo apt-get update
sudo apt-get install -y gettext
sudo apt-get install -y systemtap
sudo apt-get install -y gcc
sudo apt-get install -y linux-headers-$(uname -r)
```

验证安装

```
sudo stap -e 'probe begin { printf("Hello, World!\n"); exit() }'
```

如果打印出了 `Hello, World!` 那就说明 SystemTap 以及相关的工具安装成功了. 但是这还不完整, 为什么, SystemTap 安装好了, 并不能追踪用户空间的应用程序. 需要UTrace的支持才能实现和Dtrace提供的的完整功能. 

> 后续过程可能有错误, 如果出现版本不匹配等错误, 可以删除系统自带的systemtap包, 并通过源码安装
> `sudo apt-get remove systemtap`

### 获取调试符号

14.04 的系统, 应该导入第二个 PUBLIC KEY, 但在 `apt-get update` 的时候总是报`NO_PUBKEY C8CAB6595FDFF622`的错误. 干脆两个一起都装了. 

```
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys C8CAB6595FDFF622
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys ECDCAD72428D7C01 
```

添加更新源到 `/etc/apt/sources.list.d/ddebs.list` 文件

```
codename=$(lsb_release -c | awk  '{print $2}')

sudo tee /etc/apt/sources.list.d/ddebs.list << EOF
deb http://ddebs.ubuntu.com/ ${codename}          main restricted universe multiverse
deb http://ddebs.ubuntu.com/ ${codename}-security main restricted universe multiverse
deb http://ddebs.ubuntu.com/ ${codename}-updates  main restricted universe multiverse
deb http://ddebs.ubuntu.com/ ${codename}-proposed main restricted universe multiverse
EOF
```

更新,安装`调试符号`

```
sudo apt-get update
sudo apt-get install linux-image-$(uname -r)-dbgsym
```

### 从源码编译 SystemTap

```
sudo apt-get remove systemtap
wget https://fedorahosted.org/releases/e/l/elfutils/0.166/elfutils-0.166.tar.bz2
wget https://sourceware.org/systemtap/ftp/releases/systemtap-3.0.tar.gz
tar jxf elfutils-0.166.tar.bz2
tar zxf systemtap-3.0.tar.gz
cd systemtap-3.0
./configure --with-elfutils=../elfutils-0.166
make
sudo make install
```

### 测试是否成功安装

```
ubuntu:/srv/telegram/telegram_server$ sudo stap -v -e 'probe vfs.read {printf("read performed\n"); exit()}'
[sudo] password for ycc: 
Pass 1: parsed user script and 113 library scripts using 62612virt/39028res/4332shr/35308data kb, in 90usr/0sys/91real ms.
Pass 2: analyzed script: 1 probe, 1 function, 4 embeds, 0 globals using 218784virt/196924res/6196shr/191480data kb, in 1000usr/140sys/2045real ms.
Pass 3: translated to C into "/tmp/stap4FXIa1/stap_a9e167fcc079dfdacd5a9dd79af71e91_1748_src.c" using 218784virt/197052res/6324shr/191480data kb, in 0usr/0sys/3real ms.
Pass 4: compiled C into "stap_a9e167fcc079dfdacd5a9dd79af71e91_1748.ko" in 820usr/20sys/1101real ms.
Pass 5: starting run.
read performed
Pass 5: run completed in 0usr/10sys/385real ms.
```

最后三行输出指出 System 成功地创建了指令来探测内核, 运行指令, 检测被探测的事件(这里, 我们只想了一个虚拟文件系统读操作), 并只想一个有效的处理程序(打印文本, 并关闭)

### 百科词条

- [DTrace](http://baike.baidu.com/view/3223769.htm)
- [SystemTap](http://baike.baidu.com/view/7128648.htm)




## 重新编译 Erlang 添加对 SystemTap 的支持

> Ubuntu 使用 SystemTap 比 OSX 使用DTrace 稍微复杂一些, 要求更多的配置, 需要在配置Erlang的时候添加 `--with-dynamic-trace=systemtap` 选项. 这里为了方便我使用了kerl作为Erlang构建的管理工具. 并在`~/.kerlrc`配置文件中指定了 Erlang 的编译选项, 其内容如下:

```
KERL_CONFIGURE_OPTIONS="--with-dynamic-trace=systemtap --disable-debug --without-javac --enable-shared-zlib --enable-dynamic-ssl-lib --enable-hipe --enable-smp-support --enable-threads --enable-kernel-poll --with-wx"
```

现在就可以开始构建了, 在命令行中输入下面的命令, 等待 kerl 下载 erlang 源码包并编译.

```
# 构建
$ kerl build 18.3 18.3_systemtap
# 安装
$ kerl install 18.3_systemtap ~/.kerl/installs/18.3_systemtap
# 激活,最好把这句添加到 .bashrc
$ . /home/ycc/.kerl/installs/18.3_systemtap/activate 
```

编译,安装,激活完成后, 我们可以启动 IEx 来验证 systemtap 是否已经成功编译到 erlang 中:

![图片描述][1]

## 推荐阅读

对于希望深入研究 SystemTap, 在工作中改善系统的运行效率的同学, 可以细读下面这几本参考资料:

- [SystemTap Beginners Guide](https://sourceware.org/systemtap/SystemTap_Beginners_Guide)
- [Dynamic Tracing with DTrace & SystemTap](http://myaut.github.io/dtrace-stap-book/index.html)
- [SystemTap Language Reference](https://sourceware.org/systemtap/langref)

## Erlang 常用的探测点

### message__send

```
/**
 * Fired when a message is sent from one local process to another.
 *
 * NOTE: The 'size' parameter is in machine-dependent words and
 *       that the actual size of any binary terms in the message
 *       are not included.
 *
 * @param sender the PID (string form) of the sender
 * @param receiver the PID (string form) of the receiver
 * @param size the size of the message being delivered (words)
 * @param token_label for the sender's sequential trace token
 * @param token_previous count for the sender's sequential trace token
 * @param token_current count for the sender's sequential trace token
 */
probe message__send(char *sender, char *receiver, uint32_t size,
                    int token_label, int token_previous, int token_current);
```

> 当一条消息从本地一个进程发送到本地的其他进程, 

参数

```
Name          | Description
------------- | ------------------------
sender        | 发送进程的PID(字符串形式)
receiver      | 接收进程的PID(字符串形式)
size          | 为以字长为单位的被投递消息的大小.
token_label   | 追踪相关的符号
token_previous| 追踪相关的符号
token_current | 追踪相关的符号
```

### message__send__remote

```
/**
 * Fired when a message is sent from a local process to a remote process.
 *
 * NOTE: The 'size' parameter is in machine-dependent words and
 *       that the actual size of any binary terms in the message
 *       are not included.
 *
 * @param sender the PID (string form) of the sender
 * @param node_name the Erlang node name (string form) of the receiver
 * @param receiver the PID/name (string form) of the receiver
 * @param size the size of the message being delivered (words)
 * @param token_label for the sender's sequential trace token
 * @param token_previous count for the sender's sequential trace token
 * @param token_current count for the sender's sequential trace token
 */
probe message__send__remote(char *sender, char *node_name, char *receiver,
                            uint32_t size,
                    int token_label, int token_previous, int token_current);
```

> 当一条消息从本地进程发送到远程进程时触发该探测点. 

参数

```
Name          | Description
------------- | ------------------------
sender        | 发送进程的PID
node_name     | 接收进程的 Erlang 节点名称(字符串形式)
size          | 为以字长为单位的被投递消息的大小.
token_label   | 追踪相关的符号
token_previous| 追踪相关的符号
token_current | 追踪相关的符号
```

### message__queued

```
/**
 * Fired when a message is queued to a local process.  This probe
 * will not fire if the sender's pid == receiver's pid.
 *
 * NOTE: The 'size' parameter is in machine-dependent words and
 *       that the actual size of any binary terms in the message
 *       are not included.
 *
 * @param receiver the PID (string form) of the receiver
 * @param size the size of the message being delivered (words)
 * @param queue_len length of the queue of the receiving process
 * @param token_label for the sender's sequential trace token
 * @param token_previous count for the sender's sequential trace token
 * @param token_current count for the sender's sequential trace token
 */
probe message__queued(char *receiver, uint32_t size, uint32_t queue_len,
                    int token_label, int token_previous, int token_current);
```

> 当一条消息被排队到一个本地进程时触发. 如果发送进程的PID == 接受进程的PID, 该探测点不会触发.

参数

```
Name          | Description
------------- | ------------------------
sender        | 发送进程的PID
size          | 为以字长为单位的被投递消息的大小
queue_len     | 接收进程的队列长度
token_label   | 追踪相关的符号
token_previous| 追踪相关的符号
token_current | 追踪相关的符号
```

### message__receive

```
**
 * Fired when a message is 'receive'd by a local process and removed
 * from its mailbox.
 *
 * NOTE: The 'size' parameter is in machine-dependent words and
 *       that the actual size of any binary terms in the message
 *       are not included.
 *
 * @param receiver the PID (string form) of the receiver
 * @param size the size of the message being delivered (words)
 * @param queue_len length of the queue of the receiving process
 * @param token_label for the sender's sequential trace token
 * @param token_previous count for the sender's sequential trace token
 * @param token_current count for the sender's sequential trace token
 */
probe message__receive(char *receiver, uint32_t size, uint32_t queue_len,
                    int token_label, int token_previous, int token_current);
```

> 当一条消息被一个本地进程接收, 并从它的mailbox中删除时触发.

参数

```
Name          | Description
------------- | ------------------------
receiver      | 接收进程的PID(字符串形式)
size          | 为以字长为单位的被投递消息的大小
queue_len     | 接收进程的队列长度
token_label   | 追踪相关的符号
token_previous| 追踪相关的符号
token_current | 追踪相关的符号
```

### process__spawn

```
/**
 * Fired when a process is spawned.
 *
 * @param p the PID (string form) of the new process.
 * @param mfa the m:f/a of the function
 */
probe process__spawn(char *p, char *mfa);
```

> 有进程被创建时触发

### process__exit

```
/**
 * Fired when a process is exiting.
 *
 * @param p the PID (string form) of the exiting process
 * @param reason the reason for the exit (may be truncated)
 */
probe process__exit(char *p, char *reason);
```

> 进程退出时触发

### process__exit_signal

```
/**
 * Fired when exit signal is delivered to a local process.
 *
 * @param sender the PID (string form) of the exiting process
 * @param receiver the PID (string form) of the process receiving EXIT signal
 * @param reason the reason for the exit (may be truncated)
 */
probe process__exit_signal(char *sender, char *receiver, char *reason);
```

> 进程退出信号投递到本地进程时触发

### process__exit_signal__remote

```
/**
 * Fired when exit signal is delivered to a remote process.
 *
 * @param sender the PID (string form) of the exiting process
 * @param node_name the Erlang node name (string form) of the receiver
 * @param receiver the PID (string form) of the process receiving EXIT signal
 * @param reason the reason for the exit (may be truncated)
 * @param token_label for the sender's sequential trace token
 * @param token_previous count for the sender's sequential trace token
 * @param token_current count for the sender's sequential trace token
 */
probe process__exit_signal__remote(char *sender, char *node_name,
                                   char *receiver, char *reason,
                    int token_label, int token_previous, int token_current);
```

> 进程退出信号被递送到远程进程

### 

```
/**
 * Fired when a process is scheduled.
 *
 * @param p the PID (string form) of the newly scheduled process
 * @param mfa the m:f/a of the function it should run next
 */
probe process__scheduled(char *p, char *mfa);
```

### process__unscheduled

```
/**
 * Fired when a process is unscheduled.
 *
 * @param p the PID (string form) of the process that has been
 * unscheduled.
 */
probe process__unscheduled(char *p);
```

### process__hibernate

```
/**
 * Fired when a process goes into hibernation.
 *
 * @param p the PID (string form) of the process entering hibernation
 * @param mfa the m:f/a of the location to resume
 */
probe process__hibernate(char *p, char *mfa);
```

> 进程休眠

### process__port_unblocked

```
/**
 * Fired when a process is unblocked after a port has been unblocked.
 *
 * @param p the PID (string form) of the process that has been
 * unscheduled.
 * @param port the port that is no longer busy (i.e., is now unblocked)
 */
probe process__port_unblocked(char *p, char *port);
```

> 端口解锁

### process__heap_grow

```
/**
 * Fired when process' heap is growing.
 *
 * @param p the PID (string form)
 * @param old_size the size of the old heap
 * @param new_size the size of the new heap
 */
probe process__heap_grow(char *p, int old_size, int new_size);
```

> 进程堆增长

### process__heap_shrink

```
/**
 * Fired when process' heap is shrinking.
 *
 * @param p the PID (string form)
 * @param old_size the size of the old heap
 * @param new_size the size of the new heap
 */
probe process__heap_shrink(char *p, int old_size, int new_size);                    
```

> 进程堆收缩

## 运行一个探测进程生命周期的脚本


```
# /tmp/process-trace.stp
global start_time

probe begin
{
    printf("%%\n");
    start_time = gettimeofday_ns();
}

probe process("beam.smp").mark("message__send")
{
    printf("%d|%d|send|%s|%s|%d|%d|%d|%d\n",
           cpu(),
           gettimeofday_ns() - start_time,
           user_string($arg1),
           user_string($arg2),
           $arg3,$arg4, $arg5, $arg6);
}

probe process("beam.smp").mark("message__queued")
{
    printf("%d|%d|queued|%s|%d|%d|%d|%d|%d\n",
           cpu(),
           gettimeofday_ns() - start_time,
           user_string($arg1), 
           $arg2, $arg3, $arg4, $arg5, $arg6);
}

probe process("beam.smp").mark("message__receive")
{
    printf("%d|%d|receive|%s|%d|%d|%d|%d|%d\n",
           cpu(),
           gettimeofday_ns() - start_time,
           user_string($arg1), 
           $arg2, $arg3, $arg4, $arg5, $arg6);
}

probe process("beam.smp").mark("process__scheduled")
{
    printf("%d|%d|schedule|%s|%s\n",
           cpu(),
           gettimeofday_ns() - start_time, 
           user_string($arg1), 
           user_string($arg2));
}

probe process("beam.smp").mark("process__unscheduled")
{
    printf("%d|%d|unschedule|%s\n", 
           cpu(),
           gettimeofday_ns() - start_time,
           user_string($arg1));
}

probe process("beam.smp").mark("process__hibernate")
{
    printf("%d|%d|hibernate|%s|%s\n",
           cpu(),
           gettimeofday_ns() - start_time,
           user_string($arg1), 
           user_string($arg2));
}

probe process("beam.smp").mark("process__spawn")
{
    printf("%d|%d|spawn|%s|%s\n", 
           cpu(),
           gettimeofday_ns() - start_time,
           user_string($arg1), 
           user_string($arg2));
}

probe process("beam.smp").mark("process__exit")
{
    printf("%d|%d|exit|%s|%s\n", 
           cpu(),
           gettimeofday_ns() - start_time,
           user_string($arg1), 
           user_string($arg2));
}

probe process("beam.smp").mark("process__exit_signal")
{
    printf("%d|%d|exit_signal|%s|%s|%s\n",
           cpu(),
           gettimeofday_ns() - start_time,
           user_string($arg1), 
           user_string($arg2), 
           user_string($arg3));
}

/*
probe process("beam.smp").mark("process__exit_signal__remote")
{
    printf("sender %s -> node %s pid %s reason %s\n",
       user_string($arg1), user_string($arg2), user_string($arg3), user_string($arg4));
}
*/
```

```
ubuntu@ubuntu:~$ iex
Erlang/OTP 18 [erts-7.3] [source] [64-bit] [smp:4:4] [async-threads:10] [hipe] [kernel-poll:false] [systemtap]

Interactive Elixir (1.2.5) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> :os.getpid
'12796'     # 进程ID
iex(2)> 
```

`-x` 参数指定进程ID进行监控

```
sudo stap /tmp/process-trace.stp -x 12796
```

可以把输出重定向到文本文件进行后期的分析,统计.

## Erlang探测点的详细定义

源码是包含最多信息的

请参考 `~/.kerl/builds/18.3/otp_src_18.3/erts/emulator/beam/erlang_dtrace.d` 或 [erts/emulator/beam/erlang_dtrace.d](https://github.com/erlang/otp/blob/e1489c448b7486cdcfec6a89fea238d88e6ce2f3/erts/emulator/beam/erlang_dtrace.d)


  [1]: https://segmentfault.com/img/bVvNGX
  [2]: https://segmentfault.com/img/bVvOc8
  [3]: https://segmentfault.com/img/bVwz1p

  