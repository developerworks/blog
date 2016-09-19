## 显示硬件信息

### Dmidecode
Dmidecode 可以读取硬件信息, 直接输入 `sudo dmidecode` 会输出大量信息, 可以按照下面的表格分类显示

```
# 显示BIOS信息
sudo dmidecode -t 0
```

### 支持的类型

```
Type | Descritpion
---- | --------------------------
0    | BIOS
1    | System
2    | Base Board
3    | Chassis
4    | Processor
5    | Memory Controller
6    | Memory Module
7    | Cache
8    | Port Connector
9    | System Slots
10   | On Board Devices
11   | OEM Strings
12   | System Configuration Options
13   | BIOS Language
14   | Group Associations
15   | System Event Log
16   | Physical Memory Array
17   | Memory Device
18   | 32-bit Memory Error
19   | Memory Array Mapped Address
20   | Memory Device Mapped Address
21   | Built-in Pointing Device
22   | Portable Battery
23   | System Reset
24   | Hardware Security
25   | System Power Controls
26   | Voltage Probe
27   | Cooling Device
28   | Temperature Probe
29   | Electrical Current Probe
30   | Out-of-band Remote Access
31   | Boot Integrity Services
32   | System Boot
33   | 64-bit Memory Error
34   | Management Device
35   | Management Device Component
36   | Management Device Threshold Data
37   | Memory Channel
38   | IPMI Device
39   | Power Supply
```

### lshw

lshw(Hardware Lister)是另外一个可以查看硬件信息的工具, 不仅如此, 它还可以用来做一些硬件的benchmark. 这个工具其实就是用`/proc`里面读取一些文件来显示相关的信息, 它用到了如下文件和目录:

Path            | Description
--------------- | --------------
`/proc/cpuinfo` | 显示CPU信息
`/proc/bus/pci` | 显示pci信息
`/proc/scsi`    | 显示scsi信息
`/proc/net/dev` | 显示网络设备信息
`/proc/kcore`   | 从内存映像读取相关信息
`/proc/ide`     | 显示IDE设备信息
`/proc/devices` | 其他设备
`/proc/mounts`  | 当前挂载信息
`/proc/fstab`   | 启动时的挂载点

显示帮助

```
ubuntu@ubuntu:~$ sudo lshw -h
Hardware Lister (lshw) - B.02.16
usage: lshw [-format] [-options ...]
       lshw -version

	-version        print program version (B.02.16)

format can be
	-html           output hardware tree as HTML
	-xml            output hardware tree as XML
	-short          output hardware paths
	-businfo        output bus information

options can be
	-class CLASS    only show a certain class of hardware
	-C CLASS        same as '-class CLASS'
	-c CLASS        same as '-class CLASS'
	-disable TEST   disable a test (like pci, isapnp, cpuid, etc. )
	-enable TEST    enable a test (like pci, isapnp, cpuid, etc. )
	-quiet          don't display status
	-sanitize       sanitize output (remove sensitive information like serial numbers, etc.)
	-numeric        output numeric IDs (for PCI, USB, etc.)
```

显示总线信息

```
ubuntu@ubuntu:~$ sudo lshw -businfo
Bus info          Device     Class          Description
=======================================================
                             system         B85M-D2V (To be filled by O.E.M.)
                             bus            B85M-D2V
                             memory         64KiB BIOS
                             memory         256KiB L1 cache
                             memory         1MiB L2 cache
                             memory         6MiB L3 cache
                             memory         16GiB System Memory
...
...
...
                  scsi0      storage        
scsi@0:0.0.0      /dev/sda   disk           120GB KINGSTON SV300S3
scsi@0:0.0.0,1    /dev/sda1  volume         95GiB EXT4 volume
scsi@0:0.0.0,2    /dev/sda2  volume         15GiB Extended partition
...
...
...
```

```
# 显示简短的硬件摘要
ubuntu@ubuntu:~$ sudo lshw -short
H/W path        Device     Class          Description
=====================================================
...
...
...
/0/1/0.0.0      /dev/sda   disk           120GB KINGSTON SV300S3
/0/1/0.0.0/1    /dev/sda1  volume         95GiB EXT4 volume
/0/1/0.0.0/2    /dev/sda2  volume         15GiB Extended partition
/0/1/0.0.0/2/5  /dev/sda5  volume         15GiB Linux swap / Solaris partition
...
...
...
```

### LS系列命令

CPU 信息

```
ubuntu@ubuntu:~$ lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                4
On-line CPU(s) list:   0-3
Thread(s) per core:    1
Core(s) per socket:    4
Socket(s):             1
NUMA node(s):          1
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 60
Stepping:              3
CPU MHz:               842.691
BogoMIPS:              6185.68
Virtualization:        VT-x
L1d cache:             32K
L1i cache:             32K
L2 cache:              256K
L3 cache:              6144K
NUMA node0 CPU(s):     0-3
```

块设备信息

```
ubuntu@ubuntu:~$ lsblk 
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0 111.8G  0 disk 
|-sda1   8:1    0  95.9G  0 part /
|-sda2   8:2    0     1K  0 part 
`-sda5   8:5    0  15.9G  0 part [SWAP]
sdb      8:16   0 465.8G  0 disk 
|-sdb1   8:17   0    50G  0 part 
|-sdb2   8:18   0     1K  0 part 
|-sdb5   8:21   0   139G  0 part 
|-sdb6   8:22   0   139G  0 part 
`-sdb7   8:23   0 137.7G  0 part 
```

PCI设备

```
ubuntu@ubuntu:~$ lspci
00:00.0 Host bridge: Intel Corporation 4th Gen Core Processor DRAM Controller (rev 06)
...
...
...
```

USB设备

```
ubuntu@ubuntu:~$ lsusb
Bus 004 Device 002: ID 8087:8000 Intel Corp. 
Bus 004 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
...
...
...
```

SCSI 磁盘信息

```
ubuntu@ubuntu:~$ lsscsi
[0:0:0:0]    disk    ATA      KINGSTON SV300S3 BBF0  /dev/sda 
[1:0:0:0]    disk    ATA      ST500DM002-1BD14 KC48  /dev/sdb 
```

## 软件信息

Linux 分发信息

```
ubuntu@ubuntu:~$ lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 14.04.4 LTS
Release:	14.04
Codename:	trusty
```

内核版本信息

```
ubuntu@ubuntu:~$ uname -a
Linux ubuntu 4.2.0-27-generic #32~14.04.1-Ubuntu ...
```

```
ubuntu@ubuntu:~$ cat /proc/version
Linux version 4.2.0-27-generic (buildd@lcy01-23) ...
```

## 杀手级工具

```
apt-get install inxi
inxi -Fx
```

inxi 还有很多参数组合可以用, 具体参考 `inxi -h`, 下面我举几个栗子:

显示分区的UUID

```
ubuntu@ubuntu:~$ inxi -plu
Partition: ID: / size: 95G used: 1.4G (2%) fs: ext4 dev: /dev/sda1 
           label: N/A uuid: ********-b3*a-4*04-b*c8-1b0cbf4a****
           ID: swap-1 size: 17.06GB used: 0.00GB (0%) fs: swap dev: /dev/sda5 
           label: N/A uuid: ********-99*b-4*52-8*dd-85c181e9****
```

显示RAID阵列信息

```
ubuntu@ubuntu:~$ inxi -xx -R
RAID:      System: supported: N/A
           No RAID devices detected - /proc/mdstat and md_mod kernel raid module present
           Unused Devices: none
```

磁盘序列号

```
inxi -xx -D
```

芯片供应商

```
inxi -xx -G
```

网络

```
inxi -xx -i
```
