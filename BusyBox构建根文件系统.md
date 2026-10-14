# 第三十八章 根文件系统构建

Linux 三巨头 U-Boot、Linux 内核、rootfs，前两个已经完成，本章构建根文件系统 rootfs，是嵌入式 Linux 移植最后一步。做好之后得到最小可运行系统，后续在此基础上开发驱动、移植组件。

## 38.1 根文件系统简介

rootfs 不是 EXT4/YAFFS 这类内核存储文件系统，它本质是一套特殊目录集合，存放系统运行必需：**命令、库、设备文件、配置脚本**。

> **根文件系统**：内核启动后第一个挂载的文件系统，内核读取 `inittab`、`rcS` 等初始化脚本。没有 rootfs，内核会报 `Kernel panic` 内核崩溃。

> **注意**：嵌入式 Linux 内核镜像不放在 rootfs，存放在 EMMC/NAND 指定分区。

### 根目录关键子目录（嵌入式常用）

1. **`/bin`**：普通用户可用基础命令（`ls`、`mv`）可执行程序
2. **`/dev`**：设备文件，一切皆文件，例如 `/dev/ttymxc0` 串口设备
3. **`/etc`**：系统配置文件，嵌入式版本内容精简
4. **`/lib`**：共享库，程序运行依赖的动态库
5. **`/mnt`**：临时挂载点，挂 SD 卡、U盘
6. **`/proc`**：虚拟 proc 文件系统，查看系统运行信息，无实际存储介质
7. **`/usr`**：Unix 软件资源目录，存放大量应用程序
8. **`/var`**：存放会变化的数据
9. **`/sbin`**：管理员专用系统命令
10. **`/sys`**：sysfs 虚拟文件系统，导出内核硬件设备信息
11. **`/opt`**：用户可选安装软件目录

构建 rootfs，核心就是创建以上目录，并填充必要文件。

## 38.2 BusyBox 构建根文件系统

### 38.2.1 BusyBox 简介

BusyBox 是一个集成了大量的 Linux 命令和工具的软件，像 `ls`、`mv`、`ifconfig` 等命令 BusyBox 都会提供。

BusyBox 就是一个大的工具箱，这个工具箱里面集成了 Linux 的许多工具和命令。一般下载 BusyBox 的源码，然后配置 BusyBox，选择自己想要的功能，最后编译即可。

BusyBox 可以在其官网下载到，官网地址为：`https://busybox.net/` ，如图 1-1 所示。

![图 1-1 BusyBox 官网](photo/BusyBox/1-1%20BusyBox官网.png)

在官网左侧的"Get BusyBox"栏有一行"Download Source"，点击"Download Source"即可打开 BusyBox 的下载页。正点原子的文件夹目录下也有放置相应的 busybox，如路径：**1、例程源码 → 6、BusyBox 源码 → busybox-1.29.0.tar.bz2**，BusyBox 准备好以后就可以构建根文件系统了。

### 38.2.2 编译 BusyBox 构建根文件系统

一般我们在 Linux 驱动开发的时候都是通过 nfs 挂载根文件系统的，在 ubuntu 里自己设置的 nfs 服务器目录中，创建一个名为 rootfs 的子目录：

```sh
mkdir rootfs
```

创建好的子目录下存放压缩包并 cd 进入：

```sh
tar -vxjf busybox-1.29.0.tar.bz2
```

busybox 目录内容如图 1-2 所示。

![图 1-2 busybox 目录内容](photo/BusyBox/1-2%20busybox目录内容.png)

#### 1）修改 Makefile 添加编译器

同 Uboot 和 Linux 移植一样，打开 busybox 顶层 Makefile，添加 ARCH 和 CROSS_COMPILE 的值：

**示例代码 38.2.2.1 Makefile 代码段**

```makefile
164 CROSS_COMPILE ?= /usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-\
    linux-gnueabihf/bin/arm-linux-gnueabihf-
......
190 ARCH ?= arm
```

在示例代码 38.2.2.1 中 CROSS_COMPILE 使用了**绝对路径**！主要是为了防止编译出错。

#### 2）busybox 中文字符支持

默认编译 busybox 时中文字符显示不正常的，会显示为 `?`，因此需要取消对中文的显示限制。打开文件 `busybox-1.29.0/libbb/printable_string.c`，找到函数 `printable_string`，缩减后的函数内容如下：

**示例代码 38.2.2.3 libbb/printable_string.c 代码段**

```c
12 const char* FAST_FUNC printable_string(uni_stat_t *stats, const char *str)
13 {
14     char *dst;
15     const char *s;
16
17     s = str;
18     while (1) {
......
30         if (c < ' ')
31             break;
32         /* 注释掉下面这个两行代码 */
33         /* if (c >= 0x7f)
34         break; */
35         s++;
36     }
37
38 #if ENABLE_UNICODE_SUPPORT
39     dst = unicode_conv_to_printable(stats, str);
40 #else
41     {
42         char *d = dst = xstrdup(str);
43         while (1) {
44             unsigned char c = *d;
45             if (c == '\0')
46                 break;
47             /* 修改下面代码 */
48             /* if (c < ' ' || c >= 0x7f) */
49             if( c < ' ')
50                 *d = '?';
51             d++;
52         }
......
59     }
60 #endif
61     return auto_string(dst);
62 }
```

红色部分的代码就是被修改以后的，主要就是禁止字符大于 `0X7F` 以后 break 和输出 `?`。

接着打开文件 `busybox-1.29.0/libbb/unicode.c`，找到如下内容：

**示例代码 38.2.2.5 libbb/unicode.c 代码段**

```c
1003 static char* FAST_FUNC unicode_conv_to_printable2(uni_stat_t *stats,
1004     const char *src, unsigned width, int flags)
1005 {
1006     char *dst;
1007     unsigned dst_len;
1008     unsigned uni_count;
1009     unsigned uni_width;
1010
1011     if (unicode_status != UNICODE_ON) {
1012         char *d;
1013         if (flags & UNI_FLAG_PAD) {
1014             d = dst = xmalloc(width + 1);
......
1022         /* 修改下面一行代码 */
1023         /* *d++ = (c >= ' ' && c < 0x7f) ? c : '?'; */
1024         *d++ = (c >= ' ') ? c : '?';
1025         src++;
1026         }
1027         *d = '\0';
1028     } else {
1029         d = dst = xstrndup(src, width);
1030         while (*d) {
1031             unsigned char c = *d;
1032             /* 修改下面一行代码 */
1033             /* if (c < ' ' || c >= 0x7f) */
1034             if(c < ' ')
1035                 *d = '?';
1036             d++;
1037         }
1038     }
......
1044     return dst;
1045 }
......
1048     return dst;
1049 }
```

红色部分的代码就是被修改以后的，同样主要是禁止字符大于 `0X7F` 的时候设置为 `?`。busybox 中文字符支持跟代码修改有关的就改好了，最后还需要配置 busybox 来使能 unicode 码，这个稍后我们配置 busybox 的时候再设置。

#### 3）配置 busybox

跟编译 uboot 和 kernel 一样要先对 busybox 进行默认配置：

1. **defconfig**：缺省配置，即默认配置选项
2. **allyesconfig**：全选配置，也就是选中 busybox 所有功能
3. **allnoconfig**：最小配置

我们一般使用默认配置即可：

```sh
make defconfig
```

busybox 也支持图形化配置，如图 1-3 所示。

![图 1-3 busybox 图形化配置](photo/BusyBox/1-3%20busybox图形化配置.png)

配置路径如下：
```
Location: 
    -> Settings 
        -> Build static binary (no shared libs)
```

选项"Build static binary (no shared libs)"用来决定是静态编译 busybox 还是动态编译。静态编译的话就不需要库文件，但是编译出来的库会很大。动态编译的话要求根文件系统中有库文件，但是编译出来的 busybox 会小很多。**不能采用静态编译！** 因为采用静态编译的话 DNS 会出问题！无法进行域名解析。

继续配置如下路径配置项：
```
Location: 
    -> Settings 
        -> vi-style line editing commands
```

继续配置如下路径配置项：
```
Location: 
    -> Linux Module Utilities 
        -> Simplified modutils 
```

默认会选中"Simplified modutils"。

继续配置如下路径配置项：
```
Location: 
    -> Linux System Utilities 
        -> mdev (16 kb) //确保下面的全部选中，默认都是选中的

最后就是使能 busybox 的 unicode 编码以支持中文，配置路径如下：
```
Location: 
-> Settings 
-> Support Unicode //选中 
-> Check $LC_ALL, $LC_CTYPE and $LANG environment variables  //选中
```

#### 4、编译 busybox

```sh
make
make install CONFIG_PREFIX=/home/zuozhongkai/linux/nfs/rootfs
```

`CONFIG_PREFIX` 指定编译结果的存放目录，指定 rootfs 目录后编译完成以后 busybox 的所有工具和文件就会被安装到 rootfs 目录中。查看目录下发现有 `bin`、`sbin` 和 `usr` 以及 `linuxrc` 这个文件。

之前说过内核 init 进程最后会查找用户空间的 init 程序，找到以后就会运行这个用户空间的 init 程序，从而切换到用户态。如果 bootargs 设置 `init=/linuxrc`，那么 linuxrc 就可以作为用户空间的 init 程序，所以用户态空间的 init 程序是 busybox 来生成的。

### 38.2.3 向根文件系统添加 lib 库

#### 1、向 rootfs 的 `/lib` 目录添加库文件

Linux 中的应用程序一般都是需要动态库的，在 rootfs 中创建一个名为 `lib` 的文件夹：

```sh
mkdir lib
```

lib 文件夹创建好了，库文件从哪里来呢？lib 库文件从交叉编译器中获取。前面我们搭建交叉编译环境的时候将交叉编译器存放到了 `/usr/local/arm/` 目录中。

进入如下路径对应的目录：
```
/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/arm-linux-gnueabihf/libc/lib
```

此目录下有很多的 `*so*`（`*` 是通配符）和 `.a` 文件，这些就是库文件。将此目录下所有的 `*so*` 和 `.a` 文件都拷贝到 rootfs/lib 目录中，拷贝命令如下：

```sh
cp *so* *.a /home/zuozhongkai/linux/nfs/rootfs/lib/ -d
```

后面的 `-d` 表示拷贝符号链接。这里有个比较特殊的库文件：`ld-linux-armhf.so.3`，此库文件也是个符号链接，相当于 Windows 下的快捷方式。

`ld-linux-armhf.so.3` 后面有个 `->`，表示其是个软连接文件，链接到文件 `ld-2.19-2014.08-1-git.so`。因为其是一个"快捷方式"，因此大小只有 24B。但是，**ld-linux-armhf.so.3 不能作为符号链接**，否则的话在根文件系统中执行程序无法执行！所以我们需要让 `ld-linux-armhf.so.3` 完成逆袭，由"快捷方式"变为"本尊"。

方法很简单，先将 rootfs/lib 中的 `ld-linux-armhf.so.3` 文件删除掉：

```sh
rm ld-linux-armhf.so.3
```

然后重新进入到交叉编译器的 lib 目录中，重新拷贝 `ld-linux-armhf.so.3`，注意这次**不加 `-d` 参数**：

```sh
cp ld-linux-armhf.so.3 /home/zuozhongkai/linux/nfs/rootfs/lib/
```

拷贝完成以后再到 rootfs/lib 目录下查看 `ld-linux-armhf.so.3` 文件详细信息，此时 `ld-linux-armhf.so.3` 已经不是软连接了，而是实实在在的一个库文件，而且文件大小为 724392B。

继续进入如下目录中：
```
/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/arm-linux-gnueabihf/lib 
```

此目录下也有很多的 `*so*` 和 `.a` 库文件，我们将其也拷贝到 rootfs/lib 目录中：

```sh
cp *so* *.a /home/zuozhongkai/linux/nfs/rootfs/lib/ -d
```

rootfs/lib 目录的库文件就这些了。

#### 2、向 rootfs 的 `/usr/lib` 目录添加库文件

在 rootfs 的 usr 目录下创建一个名为 lib 的目录，将如下目录中的库文件拷贝到 rootfs/usr/lib 目录下：
```
/usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/arm-linux-gnueabihf/libc/usr/lib
```

将此目录下的 so 和 .a 库文件都拷贝到 rootfs/usr/lib 目录中：

```sh
cp *so* *.a /home/zuozhongkai/linux/nfs/rootfs/usr/lib/ -d
```

至此，根文件系统的库文件就全部添加好了。可以使用 `du` 命令来查看一下 rootfs/lib 和 rootfs/usr/lib 这两个目录的大小：

```sh
cd rootfs                    # 进入根文件系统目录
du ./lib ./usr/lib/ -sh      # 查看 lib 和 usr/lib 这两个目录的大小
```

### 38.2.4 创建其他文件夹

在根文件系统中创建其他文件夹，如 `dev`、`proc`、`mnt`、`sys`、`tmp` 和 `root` 等。

## 38.3 根文件系统测试

使用 NFS 挂载来测试。uboot 里面的 bootargs 环境变量会设置 `root` 的值，所以我们将 `root` 的值改为 NFS 挂载即可。在 Linux 内核源码里面有相应文档讲解如何设置，文档为 `Documentation/filesystems/nfs/nfsroot.txt`，格式如下：
```
root=/dev/nfs nfsroot=[<server-ip>:]<root-dir>[,<nfs-options>] ip=<client-ip>:<server-ip>:<gw 
-ip>:<netmask>:<hostname>:<device>:<autoconf>:<dns0-ip>:<dns1-ip>
```

| 参数 | 说明 |
|------|------|
| `<server-ip>` | 服务器 IP 地址，也就是存放根文件系统主机的 IP 地址，即 Ubuntu 的 IP 地址，比如 `192.168.1.250` |
| `<root-dir>` | 根文件系统的存放路径，比如 `/home/zuozhongkai/linux/nfs/rootfs` |
| `<nfs-options>` | NFS 的其他可选选项，一般不设置 |
| `<client-ip>` | 客户端 IP 地址，也就是开发板的 IP 地址。此地址一定要和 Ubuntu 主机在同一个网段内，并且没有被其他设备使用。比如设置为 `192.168.1.251` |
| `<gw-ip>` | 网关地址，比如 `192.168.1.1` |
| `<netmask>` | 子网掩码，比如 `255.255.255.0` |
| `<hostname>` | 客户机的名字，一般不设置，此值可以空着 |
| `<device>` | 设备名，也就是网卡名，一般是 `eth0`、`eth1`……。这里我们使用 ENET2，所以网卡名就是 `eth0` |
| `<autoconf>` | 自动配置，一般不使用，所以设置为 `off` |
| `<dns0-ip>` | DNS0 服务器 IP 地址，不使用 |
| `<dns1-ip>` | DNS1 服务器 IP 地址，不使用 |

根据上面的格式，bootargs 环境变量的 root 值如下：
```
root=/dev/nfs nfsroot=192.168.1.250:/home/zuozhongkai/linux/nfs/rootfs,proto=tcp rw 
ip=192.168.1.251:192.168.1.250:192.168.1.1:255.255.255.0::eth0:off
```

`proto=tcp` 表示使用 TCP 协议，`rw` 表示 nfs 挂载的根文件系统为可读可写。

启动开发板，进入 uboot 命令行模式，然后重新设置 bootargs 环境变量：

```sh
setenv bootargs 'console=ttymxc0,115200 root=/dev/nfs nfsroot=192.168.1.250:/home/zuozhongkai/linux/nfs/rootfs,proto=tcp rw ip=192.168.1.251:192.168.1.250:192.168.1.1:255.255.255.0::eth0:off'
saveenv
```

设置好以后使用 `boot` 命令启动 Linux 内核。我们进入了根文件系统，说明我们的根文件系统工作了！如果没有启动进入根文件系统的话可以重启一次开发板试试。

输入 `ls` 命令测试一下，发现很多文件夹都有了，但是在进入根文件系统的时候会有下面这一行错误提示：can't run '/etc/init.d/rcS': No such file or directory 

提示很简单，说是无法运行 `/etc/init.d/rcS` 这个文件，因为这个文件不存在。

## 38.4 完善根文件系统

### 38.4.1 创建 `/etc/init.d/rcS` 文件

rcS 是个 shell 脚本，Linux 内核启动以后需要启动一些服务，而 rcS 就是规定启动哪些文件的脚本文件。

在 rootfs 中创建 `/etc/init.d/rcS` 文件，然后在 rcS 中输入如下所示内容：

**示例代码 38.4.1.1 /etc/init.d/rcS 文件**

```sh
#!/bin/sh

PATH=/sbin:/bin:/usr/sbin:/usr/bin:$PATH
LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/lib:/usr/lib
export PATH LD_LIBRARY_PATH

mount -a
mkdir /dev/pts
mount -t devpts devpts /dev/pts

echo /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s
```

- **第 3 行**，`PATH` 环境变量保存着可执行文件可能存在的目录，这样我们在执行一些命令或者可执行文件的时候就不会提示找不到文件这样的错误。
- **第 4 行**，`LD_LIBRARY_PATH` 环境变量保存着库文件所在的目录。
- **第 5 行**，使用 `export` 来导出上面这些环境变量，相当于声明一些"全局变量"。
- **第 7 行**，使用 `mount` 命令来挂载所有的文件系统，这些文件系统由文件 `/etc/fstab` 来指定，所以我们一会还要创建 `/etc/fstab` 文件。
- **第 8 和 9 行**，创建目录 `/dev/pts`，然后将 devpts 挂载到 `/dev/pts` 目录中。
- **第 11 和 12 行**，使用 mdev 来管理热插拔设备，通过这两行，Linux 内核就可以在 `/dev` 目录下自动创建设备节点。关于 mdev 的详细内容可以参考 busybox 中的 `docs/mdev.txt` 文档。

示例代码 38.4.1.1 中的 rcS 文件内容是最精简的，大家如果去看 Ubuntu 或者其他大型 Linux 操作系统中的 rcS 文件，就会发现其非常复杂。因为我们是初次学习，所以不用搞这么复杂的，而且这么复杂的 rcS 文件也是借助其他工具创建的，比如 buildroot 等。

创建好文件 `/etc/init.d/rcS` 以后一定要给其**可执行权限**！

```sh
chmod 777 rcS
```

设置好以后就重新启动 Linux 内核，提示找不到 `/etc/fstab` 文件，还有一些其他的错误。我们先把 `/etc/fstab` 这个错误解决了。前面我们说了 `mount -a` 挂载所有根文件系统的时候需要读取 `/etc/fstab`，因为 `/etc/fstab` 里面定义了该挂载哪些文件。

### 38.4.2 创建 `/etc/fstab` 文件

在 rootfs 中创建 `/etc/fstab` 文件。fstab 在 Linux 开机以后自动配置哪些需要自动挂载的分区，格式如下：
```
<file system> <mount point> <type> <options> <dump> <pass>
```

| 字段 | 说明 |
|------|------|
| `<file system>` | 要挂载的特殊的设备，也可以是块设备，比如 `/dev/sda` 等等 |
| `<mount point>` | 挂载点 |
| `<type>` | 文件系统类型，比如 `ext2`、`ext3`、`proc`、`romfs`、`tmpfs` 等等 |
| `<options>` | 挂载选项，一般使用 `defaults`，包含了 `rw`、`suid`、`dev`、`exec`、`auto`、`nouser` 和 `async` |
| `<dump>` | 为 1 表示允许备份，为 0 不备份，一般设置为 0 |
| `<pass>` | 磁盘检查设置，为 0 表示不检查。根目录 `/` 设置为 1，其他的都不能设置为 1，其他的分区从 2 开始。一般不在 fstab 中挂载根目录，因此这里一般设置为 0 |

按照上述格式，在 fstab 文件中输入如下内容：
```
示例代码 38.4.2.1 /etc/fstab 文件
示例代码 38.4.2.1 /etc/fstab 文件 
1 #<file system>   <mount point>   <type>     <options> <dump> <pass> 
2 proc               /proc           proc       defaults    0      0 
3 tmpfs              /tmp            tmpfs      defaults    0      0 
4 sysfs              /sys            sysfs      defaults    0      0 

fstab 文件创建完成以后重新启动 Linux，启动成功，而且没有任何错误提示。但是我们还需要创建一个文件 `/etc/inittab`。

### 38.4.3 创建 `/etc/inittab` 文件

inittab 的详细内容可以参考 busybox 下的文件 `examples/inittab`。init 程序会读取 `/etc/inittab` 这个文件，inittab 由若干条指令组成。每条指令的结构都是一样的，由以 `:` 分隔的 4 个段组成，格式如下：
```
<id>:<runlevels>:<action>:<process/></action></runlevels></id>
```

- **`<id>`**：每个指令的标识符，不能重复。但是对于 busybox 的 init 来说，`<id>` 有着特殊意义——用来指定启动进程的控制 tty，一般我们将串口或者 LCD 屏幕设置为控制 tty。
- **`<runlevels>`**：对 busybox 来说此项完全没用，所以空着。
- **`<action>`**：动作，用于指定 `<process>` 可能用到的动作。busybox 支持的动作如表 38.4.3.1 所示。
- **`<process>`**：具体的动作，比如程序、脚本或命令等。

**表 38.4.3.1 busybox 支持的 action**

| 动作 | 描述 |
|------|------|
| `sysinit` | 在系统初始化的时候 process 才会执行一次 |
| `respawn` | 当 process 终止以后马上启动一个新的 |
| `askfirst` | 和 respawn 类似，在运行 process 之前在控制台上显示 "Please press Enter to activate this console."。只要用户按下 "Enter" 键以后才会执行 process |
| `wait` | 告诉 init，要等待相应的进程执行完以后才能继续执行 |
| `once` | 仅执行一次，而且不会等待 process 执行完成 |
| `restart` | 当 init 重启的时候才会执行 process |
| `ctrlaltdel` | 当按下 ctrl+alt+del 组合键才会执行 process |
| `shutdown` | 关机的时候执行 process |

参考 busybox 的 `examples/inittab` 文件，我们也创建一个 `/etc/inittab`，在里面输入如下内容：

**示例代码 38.4.3.1 /etc/inittab 文件**

```sh
# /etc/inittab
::sysinit:/etc/init.d/rcS
console::askfirst:-/bin/sh
::restart:/sbin/init
::ctrlaltdel:/sbin/reboot
::shutdown:/bin/umount -a -r
::shutdown:/sbin/swapoff -a
```

- **第 2 行**，系统启动以后运行 `/etc/init.d/rcS` 这个脚本文件。
- **第 3 行**，将 `console` 作为控制台终端，也就是 `ttymxc0`。
- **第 4 行**，重启的话运行 `/sbin/init`。
- **第 5 行**，按下 `ctrl+alt+del` 组合键的话就运行 `/sbin/reboot`。
- **第 6 行**，关机的时候执行 `/bin/umount`，也就是卸载各个文件系统。
- **第 7 行**，关机的时候执行 `/sbin/swapoff`，也就是关闭交换分区。

`/etc/inittab` 文件创建好以后就可以重启开发板即可，至此！根文件系统要创建的文件就已经全部完成了。

接下来就要对根文件系统进行其他的测试，比如我们自己编写的软件运行是否正常、是否支持软件开机自启动、中文支持是否正常以及能不能联网等。

## 38.5 根文件系统其他功能测试

### 38.5.1 软件运行测试

我们编写一个小小的测试软件来测试一下库文件是否工作正常。在根文件系统下创建一个名为 `drivers` 的文件夹，以后我们学习 Linux 驱动的时候就把所有的实验文件放到这个文件夹里面。

在 ubuntu 下使用 vim 编辑器新建一个 `hello.c` 文件，在 hello.c 里面输入如下内容：

**示例代码 38.5.1.1 hello.c 文件**

```c
#include <stdio.h>

int main(void)
{
    while(1) {
        printf("hello world!\r\n");
        sleep(2);
    }
    return 0;
}
```

hello.c 内容很简单，就是循环输出 "hello world"。`sleep` 相当于 Linux 的延时函数，单位为秒，所以 `sleep(2)` 就是延时 2 秒。

编写好以后就是编译，因为我们是要在 ARM 芯片上运行的，所以要用交叉编译器去编译：

```sh
arm-linux-gnueabihf-gcc hello.c -o hello
```

使用 `file` 命令查看文件类型以及编码格式：

```sh
file hello
```

输出信息如下：
```
hello: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), statically linked, not stripped
```

hello 是个 32 位的 LSB 可执行文件，ARM 架构的，并且是动态链接的。所以我们编译出来的 hello 文件没有问题。

将其拷贝到 `rootfs/drivers` 目录下，在开发板中输入如下命令来执行这个可执行文件：

```sh
cd /drivers    # 进入 drivers 目录
./hello        # 执行 hello
```

运行软件的时候加上 `&` 即可让程序在后台运行，比如 `./hello &`。

在后台运行的软件可以使用 `kill -9 pid`（进程 ID）命令来关闭掉。首先使用 `ps` 命令查看要关闭的软件 PID 是多少，然后用 `kill -9` 结束它。比如 hello 对应的 PID 为 166：

```sh
kill -9 166
```

### 38.5.2 中文字符测试

#### 1、设置 SecureCRT 使用 UTF-8 编码

因为 Linux 使用的编码格式为 UTF-8，因此要先设置 SecureCRT 的编码格式。打开 `Options → Session Options…`，打开"Session Options"对话框，选择左侧的"Appearance"，然后在右侧的"Character encoding:"栏选择 UTF-8 编码，设置好以后点击下方的"Ok"按钮即可。

#### 2、创建中文文件

在 ubuntu 中向 rootfs 目录新建一个名为"中文测试"的文件夹，然后在 SecureCRT 下查看中文名能不能显示正确。接着用 `touch` 命令在"中文测试"文件夹中新建一个名为"测试文档.txt"的文件，并且使用 vim 编辑器在其中输入"这是一个中文测试文件"，借此来测试一下中文文件名和中文内容显示是否正常。

### 38.5.3 开机自启动测试

在 38.5.1 小节测试 hello 软件的时候都是等 Linux 启动进入根文件系统以后手动输入命令 `./hello` 来完成的。我们一般做好产品以后都是需要开机自动启动相应的软件，本节我们就以 hello 这个软件为例，讲解一下如何实现开机自启动。

因为进入根文件系统的时候会运行 `/etc/init.d/rcS` 这个 shell 脚本，因此我们可以在这个脚本里面添加自启动相关内容。添加完成以后的 `/etc/init.d/rcS` 文件内容如下：

**示例代码 38.5.3.1 rcS 文件代码**

```sh
#!/bin/sh
PATH=/sbin:/bin:/usr/sbin:/usr/bin
LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/lib:/usr/lib
runlevel=S
umask 022
export PATH LD_LIBRARY_PATH runlevel

mount -a
mkdir /dev/pts
mount -t devpts devpts /dev/pts

echo /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s

# 开机自启动
cd /drivers
./hello &
cd /
```

- **第 16 行**，进入 drivers 目录，因为要启动的软件存放在 drivers 目录下。
- **第 17 行**，以后台方式执行 hello 这个软件。
- **第 18 行**，退出 drivers 目录，进入到根目录下。

自启动代码添加完成以后就可以重启开发板，看看 hello 这个软件会不会自动运行。

### 38.5.4 外网连接测试

这里说的外网不是外国那些 404 网站的连接测试，而是百度、淘宝等这些网站的测试。也就是说看看我们的开发板能不能上网，能不能和我们的局域网外的这些网站进行通信。

测试方法很简单，就是通过 `ping` 命令来 ping 一下百度的官网 `www.baidu.com`：

```sh
ping www.baidu.com
```