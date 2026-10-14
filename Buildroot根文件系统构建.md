# 第A1章 Buildroot 根文件系统构建

BusyBox 很多第三方软件需要我们自己移植，而且这些第三方还有很多依赖于其他库文件，导致移植过程繁琐。

## A1.1 何为 buildroot？

### A1.1.1 buildroot 简介

第三篇系统移植篇里我们使用 busybox 构建根文件系统。busybox 仅提供基础命令，lib 库、/etc 下各类配置文件都需要手动编写，默认也没有账号密码。后续如果要移植 alsa、iperf、mplayer 这类第三方库与软件，都需要开发者自己处理。

Buildroot 就是解决这个问题的工具，内置 busybox，同时集成大量第三方库与应用软件，按需勾选即可，类似自助餐，大幅简化嵌入式根文件系统制作。

- **busybox 手动构建**：适合学习根文件原理，但细节多，不适合产品开发。
- **buildroot**：自动处理依赖与配置，生成的 rootfs 更规范稳定，产品项目推荐使用；同类工具还有 Yocto。

Buildroot 使用流程类似 U-Boot、Linux 内核：**下载源码 → 配置（交叉编译器、目标 CPU、勾选软件包）→ 编译**，编译完成后输出目录直接生成根文件系统。

### A1.1.2 buildroot 下载

源码从官网获取：`https://buildroot.org/` ，打开官网界面如图 1-1 所示，点击 DOWNLOAD 进入下载页面。

![图 1-1 buildroot 官方界面](photo/buildroot/1-1%20buildroot官方界面.png)

正点原子已把源码放在开发板光盘，路径：**1、例程源码 → 8、buildroot 源码 → buildroot-2019.02.6.tar.bz2**，准备完毕就可以开始构建根文件系统。

## A1.2 buildroot 构建根文件系统

### A1.2.1 配置 buildroot

将 buildroot 源码 `buildroot-2019.02.6.tar.bz2` 拷贝到 Ubuntu 的 tool 目录。拷贝完成后解压，命令如下：

```sh
tar -vxjf buildroot-2019.02.6.tar.bz2
```

解压得到目录 `buildroot-2019.02.6`，进入该源码目录，目录文件如图 1-2 所示。

![图 1-2 buildroot 源码文件](photo/buildroot/1-2%20buildroot源码文件.png)

buildroot 和 uboot、Linux kernel 一样支持图形化配置，输入下面命令打开图形配置界面：

```sh
make menuconfig
```

图形化配置界面打开后如图 1-3 所示。

![图 1-3 buildroot 图形化配置界面](photo/buildroot/1-3%20buildroot图形化配置界面.png)

接下来依次配置 buildroot，配置完成后即可编译。

#### 1、配置 Target options

Target options 配置项：

- Target Architecture = ARM (little endian)
- Target Binary Format = ELF
- Target Architecture Variant = cortex-A7
- Target ABI = EABIhf
- Floating point strategy = NEON/VFPv4
- ARM instruction set = ARM

配置完成如图 1-4 所示。

![图 1-4 配置好的 Target options 选项](photo/buildroot/1-4%20配置好的Target%20options选项.png)

#### 2、配置 Toolchain

用来指定交叉编译工具链；buildroot 可自动下载编译器，但国内网络差，推荐使用本地预装的外部交叉编译器。

- Toolchain type = External toolchain
- Toolchain = Custom toolchain  // 用户自定义交叉编译器
- Toolchain origin = Pre-installed toolchain  // 使用预装编译器
- Toolchain path = /usr/local/arm/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf
- Toolchain prefix = $(ARCH)-linux-gnueabihf
- External toolchain gcc version = 4.9.x
- External toolchain kernel headers series = 4.1.x
- External toolchain C library = glibc/eglibc
- [\*] Toolchain has SSP support? (NEW)
- [\*] Toolchain has RPC support? (NEW)
- [\*] Toolchain has C++ support?
- [\*] Enable MMU support (NEW)

**说明：**
- `Custom toolchain`：使用自己的交叉编译器
- `Pre-installed toolchain`：编译器提前安装在主机上
- `Toolchain path`：交叉编译器的绝对路径
- `Toolchain prefix`：编译器前缀，对应 `arm-linux-gnueabihf-gcc`

#### 3、配置 System configuration

系统基础设置：主机名、欢迎语、初始化系统、root 账号密码等

- System hostname = alpha_imx6ull
- System banner = Welcome to alpha i.mx6ull
- Init system = BusyBox
- /dev management = Dynamic using devtmpfs + mdev
- [\*] Enable root login with password (NEW)
- Root password = 123456

#### 4、配置 Filesystem images

设置最终根文件系统镜像格式

- [\*] ext2/3/4 root filesystem
- ext2/3/4 variant = ext4
- [\*] ubi image containing an ubifs root filesystem  // NAND Flash 使用 ubifs

I.MX6U 开发板通过 Mfgtool 烧写 EMMC/SD 卡，此处可以不生成镜像，后续自己打包根文件系统。

#### 5、禁止编译 Linux 内核和 uboot

buildroot 可以自动编译内核和 uboot，但官方源码缺少板载驱动，编译器版本容易冲突，因此只让 buildroot 制作根文件系统。

- `Kernel ---> [ ] Linux Kernel`  // 取消选中，不编译内核，如图 1-5 所示

![图 1-5 不编译 Linux 内核](photo/buildroot/1-5%20不编译Linux内核.png)

- `Bootloaders ---> [ ] U-Boot`  // 取消选中，不编译 U-Boot，如图 1-6 所示

![图 1-6 不编译 U-Boot](photo/buildroot/1-6%20不编译U-Boot.png)

#### 6、配置 Target packages

用于勾选第三方库与应用（alsa-utils、ffmpeg、iperf 等）。

初次编译不要勾选任何额外软件，先生成基础最小根文件系统，验证编译没问题后，再回来添加第三方包。一次性选太多软件，编译报错很难定位问题。

### A1.2.3 buildroot 根文件系统测试

buildroot 制作出来的根文件系统已经准备好了，接下来就是对其进行测试。测试方法也是通过 nfs 挂载的方式，启动 u-boot，修改 bootargs 环境变量，设置 nfsroot 目录为 Ubuntu 中的 buildrootfs 目录，命令如下：

```sh
setenv bootargs 'console=tty1 console=ttymxc0,115200 root=/dev/nfs nfsroot=192.168.1.253:/home/zuozhongkai/linux/nfs/buildrootfs rw ip=192.168.1.251:192.168.1.253:192.168.1.1:255.255.255.0::eth0:off'
```

设置好以后启动系统，进入根文件系统以后如图 2-1 所示。

![图 2-1 buildroot 根文件系统](photo/buildroot/2-0%20buildroot根文件系统.png)

> **注意**：启动后 `/lib/modules` 目录默认不存在，需要手动自行创建。

buildroot 构建的根文件系统启动以后会输出前面配置的欢迎语 `Welcome to alpha i.mx6ull`。输入用户名 `root`，密码为配置 buildroot 时设置的 `123456` 即可登录系统。此时是最简根文件系统，还没有配置任何第三方库和软件，后面我们开启第三方组件。

## A1.3 buildroot 第三方软件和库的配置

以前需要手动移植 alsa-lib、alsa-utils 等库，buildroot 中直接图形化勾选即可。以 alsa 为例：

**1、使能 alsa-lib**

执行 `make menuconfig`，配置路径：
```
Target packages
-> Libraries
-> Audio/Sound
-> -*- alsa‑lib --->
```

将该配置项下子项全部选中。

**2、使能 alsa-utils**
```
Target packages
-> Audio and video applications
-> alsa‑utils
```

将该目录下软件全部选中。

配置完成后编译，编译完成，把 `buildroot/images/rootfs.tar` 拷贝到 nfs 的 buildrootfs 目录，重新解压：

```sh
tar -vxf rootfs.tar
```

> **注意**：解压不会删除旧的自定义添加文件。

解压完毕就可以使用 `alsamixer` 等工具，无需手动移植。可按需开启其它第三方库与软件。

## A1.4 buildroot 下的 busybox 配置

### A1.4.1 busybox 配置

buildroot 会自动下载 busybox 源码，下载的压缩包存放在 `dl` 目录；解压后的源码放在 `output/build/busybox-1.29.3`，如需修改 busybox 源码直接在此目录修改。

打开 busybox 图形配置界面命令：

```sh
sudo make busybox-menuconfig
```

打开后就是 busybox 原生配置界面，操作和之前手动构建根文件系统时一致。

![图 A1.4.1 busybox 源码](photo/buildroot/2-1%20busybox源码.png)
![图 A1.4.2 buildroot 下的 busybox 配置界面](photo/buildroot/2-2%20buildroot下的busybox配置界面.png)

### A1.4.2 busybox 中文字符的支持

busybox 对中文显示有限制，修改方法参考 38.2.2 小节，本节不再重复。

### A1.4.3 编译 busybox

进入 buildroot 源码目录，查看当前所有启用软件包：

```sh
sudo make show-targets
```

单独编译安装 busybox：

```sh
sudo make busybox
```

完成后整体重新打包根文件系统：

```sh
sudo make
```

编译后检查 `output/images` 下 `rootfs.tar` 时间戳，如果不是最新，删除旧包再执行一次 `make`，之后使用新 `rootfs.tar` 启动系统。

## A1.5 根文件系统测试

整体测试项目和第三十八章一致，不再复述。

**问题现象**：登录后提示符始终只有 `#`，进入目录也不显示当前路径，不直观。

**原因**：由环境变量 `PS1` 控制命令行提示符格式。

**PS1 常用参数：**

| 参数 | 含义 |
|------|------|
| `\$` | root 显示 `#`，普通用户显示 `$` |
| `\u` | 当前用户名 |
| `\h` | 主机名 |
| `\w` | 完整当前工作目录路径 |
| `\W` | 仅显示当前目录名 |

修改 `/etc/profile`，注释原有 PS1 设置：

```sh
# if [ "$PS1" ]; then
#   if [ "`id -u`" -eq 0 ]; then
#     export PS1='# '
#   else
#     export PS1='$ '
#   fi
# fi
```

添加新的提示符配置：

```sh
PS1='[\u@\h]:\w$:'
export PS1
```

如图 2-4 所示。

![图 2-4 修改后的 profile 文件](photo/buildroot/2-4%20修改后的profile文件.png)

保存后重启开发板，提示符会展示 `[用户名@主机名]:当前路径$:`，如图 2-5 所示。

![图 2-5 命令提示符](photo/buildroot/2-5%20命令提示符.png)

至此 buildroot 构建根文件系统全部完成。可根据产品需求继续开启其它第三方软件包。
