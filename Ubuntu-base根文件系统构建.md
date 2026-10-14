# 第 A3 章 Ubuntu-base 根文件系统构建

大家肯定会有疑问，能不能在 ALPHA 开发板上运行 Ubuntu 呢？答案肯定是可以的，Ubuntu 是 Linux 系统的一种，可以简单的将 Ubuntu 理解为一个根文件系统，和我们用 busybox、buildroot 制作的根文件系统一样。因此移植 Ubuntu 也就是将 Ubuntu 根文件系统移植到我们的开发板上，但是因为 I.MX6ULL 性能偏弱，本章我们只移植 Ubuntu 的最小根文件系统，不带图形界面。

## A3.1 ubuntu-base 获取

本章参考网上相关帖子与内容。 Ubuntu 的移植不需要编译源码，官方已经预制好根文件系统，我们只需要简单配置，适配开发板即可。本章选用 Ubuntu 16.04，也可以尝试移植 18.04 等其他版本。

官方下载地址：`http://cdimage.ubuntu.com/` ，下载界面如图 1-1 所示。进入 ubuntu-base 下载页面如图 1-2，打开 releases 版本页面如图 1-3。Ubuntu 提供 12.04 ~ 19.10 多个版本的 ubuntu-base。

![图 1-1 ubuntu镜像下载网站](photo/Ubuntu-base/1-1%20ubuntu镜像下载网站.png)
![图 1-2 ubuntu-base下载界面](photo/Ubuntu-base/1-2%20ubuntu-base下载界面.png)
![图 1-3 ubuntu base各版本](photo/Ubuntu-base/1-3%20ubuntu%20base各版本.png)

选择 16.04.5，进入 release 页面。Ubuntu 提供多种 CPU 架构包：amd64、armhf、i386、powerpc 等。I.MX6ULL 为 CortexA7，带硬件浮点单元，选择 armhf。 找到文件：ubuntu-base-16.04.5-base-armhf.tar.gz，下载。

光盘预置文件路径：开发板光盘 ->8、系统镜像 ->2、教程系统镜像 ->3、文件系统 ->1、Ubuntu base 根文件系统 ->ubuntu-base-16.04.5-base-armhf.tar.gz

## A3.2 ubuntu 根文件系统构建

### A3.2.1 解压缩 ubuntu base 根文件系统

在主机 Ubuntu 的 nfs 目录新建ubuntu_rootfs目录，存放文件系统：

```sh
cd /home/zuozhongkai/linux/nfs
mkdir ubuntu_rootfs
```

把下载好的ubuntu-base-16.04.5-base-armhf.tar.gz拷贝进ubuntu_rootfs，解压：

```sh
sudo tar -vzxf ubuntu-base-16.04.5-base-armhf.tar.gz
```

解压完成如图 2-1 所示，解压出来是标准 Linux 根目录，但还不能直接在开发板运行，需要进一步配置。

![图 2-1 解压缩以后的 ubuntu base 根文件系统](photo/Ubuntu-base/2-1%20解压缩以后的%20ubuntu%20base%20根文件系统.png)

### A3.2.2 安装 qemu

主机 Ubuntu 安装 qemu 静态模拟器，用于 chroot 到 armhf 文件系统里配置：

```sh
sudo apt-get install qemu-user-static
```

将qemu-arm-static拷贝到解压后的 ubuntu 根文件系统的usr/bin目录：

```sh
cd /home/zuozhongkai/linux/nfs/ubuntu_rootfs
sudo cp /usr/bin/qemu-arm-static ./usr/bin/
```

说明：qemu-arm-static作用：在 x86 主机上 chroot 进入 armhf 架构的 ubuntu-base，执行 arm 架构的命令，用来配置 rootfs（设置密码、软件包等）。

### A3.2.3 设置软件源

apt-get 安装软件需要软件源。先把主机的 DNS 配置复制到根文件系统，保证 chroot 内网络域名解析正常。

```sh
cd /home/zuozhongkai/linux/nfs/ubuntu_rootfs
sudo cp /etc/resolv.conf ./etc/resolv.conf
```

编辑根文件系统内 ubuntu_rootfs/etc/apt/sources.list，添加armhf 架构的 Ubuntu-ports 中科大源，不要使用 x86 的源。
```
#中科大源
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-backports main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-proposed main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-security main multiverse restricted universe
deb http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-updates main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-backports main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-proposed main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-security main multiverse restricted universe
deb-src http://mirrors.ustc.edu.cn/ubuntu-ports/ xenial-updates main multiverse restricted universe
```

### A3.2.4 在主机挂载并配置根文件系统

#### 1、编写挂载、卸载脚本

在ubuntu_rootfs目录新建 `mount.sh`：

```sh
#!/bin/bash
echo "MOUNTING"
sudo mount -t proc /proc /home/zuozhongkai/linux/nfs/ubuntu_rootfs/proc
sudo mount -t sysfs /sys /home/zuozhongkai/linux/nfs/ubuntu_rootfs/sys
sudo mount -o bind /dev /home/zuozhongkai/linux/nfs/ubuntu_rootfs/dev
sudo mount -o bind /dev/pts /home/zuozhongkai/linux/nfs/ubuntu_rootfs/dev/pts
sudo chroot /home/zuozhongkai/linux/nfs/ubuntu_rootfs
```

新建 `unmount.sh`：

```sh
#!/bin/bash
echo "UNMOUNTING"
sudo umount /home/zuozhongkai/linux/nfs/ubuntu_rootfs/proc
sudo umount /home/zuozhongkai/linux/nfs/ubuntu_rootfs/sys
sudo umount /home/zuozhongkai/linux/nfs/ubuntu_rootfs/dev/pts
sudo umount /home/zuozhongkai/linux/nfs/ubuntu_rootfs/dev
```

添加可执行权限：

```sh
sudo chmod 777 mount.sh unmount.sh
```

执行挂载脚本，进入 chroot 环境，如图 2-2 所示：

```sh
./mount.sh
```

![图 2-2 切换根文件系统](photo/Ubuntu-base/2-2%20切换根文件系统.png)

#### 2、安装常用命令软件（chroot 内部执行）

ubuntu-base 是最小系统，缺少大量工具：

```sh
apt update
apt install sudo
apt install vim
apt install kmod
apt install net-tools
apt install ethtool
apt install ifupdown
apt install language-pack-en-base
apt install rsyslog
apt install htop
apt install iputils-ping
```

#### 3、设置 root 密码

```sh
passwd root
```

设置密码为root，输入两次确认，如图 2-3 所示。

![图 2-3 设置root用户密码](photo/Ubuntu-base/2-3%20设置root用户密码.png)

#### 4、配置主机名与 hosts

```sh
echo "alientek_imx6ul" > /etc/hostname
echo "127.0.0.1 localhost" >> /etc/hosts
echo "127.0.0.1 alientek_imx6ul" >> /etc/hosts
```

#### 5、配置串口 ttymxc0 终端登录

I.MX6ULL 开发板串口为ttymxc0，创建 systemd 服务软链接，使串口可以登录：

```sh
ln -s /lib/systemd/system/getty@.service /etc/systemd/system/getty.target.wants/getty@ttymxc0.service
```

退出 chroot：

```sh
exit
```

执行卸载脚本，一定要卸载，防止文件残留占用：

```sh
./unmount.sh
```

到此 ubuntu-base 根文件系统制作完成。

## A3.3 ubuntu 根文件系统测试

### A3.3.1 nfs 挂载测试

U-Boot 设置 bootargs：

```sh
setenv bootargs 'console=tty1 console=ttymxc0,115200 root=/dev/nfs nfsroot=192.168.1.253:/home/zuozhongkai/linux/nfs/ubuntu_rootfs rw ip=192.168.1.251:192.168.1.253:192.168.1.1:255.255.255.0::eth0:off'
saveenv
```

重启开发板，通过 NFS 挂载 Ubuntu 根文件系统，如图 2-4 所示。 登录用户名：root，密码：root，登录成功后系统正常运行，如图 2-5 所示。

![图 2-4 进入ubuntu根文件系统](photo/Ubuntu-base/2-4%20进入ubuntu根文件系统.png)
![图 2-5 进入ubuntu根文件系统](photo/Ubuntu-base/2-5%20进入ubuntu根文件系统.png)

#### 实操小提示：
1. 执行完 chroot 必须运行unmount.sh，否则 proc/sys/dev 目录处于挂载状态，后续打包会出问题；
2. qemu-arm-static 必须放在 rootfs 的/usr/bin，否则 chroot 执行 arm 程序报错；
3. 软件源必须用ubuntu-ports，专门给 arm 架构使用。

### A3.3.2 ubuntu 根文件系统烧写

NFS 挂载测试正常后，使用 MfgTool 把 Ubuntu 根文件系统烧写到板载 EMMC/NAND，烧写流程参考《第三十九章 系统烧写》。

#### 1、打包 ubuntu 根文件系统

```sh
cd ubuntu_rootfs
tar -vcjf ubuntu_rootfs.tar.bz2 *
```

打包完成，得到ubuntu_rootfs.tar.bz2。

#### 2、烧写 ubuntu 根文件系统 将打包文件重命名：
- EMMC 版本：rootfs-alientek-emmc.tar.bz2
- NAND 版本：rootfs-alientek-nand.tar.bz2

运行对应的 vb 脚本：
- EMMC：mfgtool2-alientek-alpha-emmc.vbs
- NAND：mfgtool2-alientek-alpha-nand.vbs

也可以新增专属烧写脚本，完成 MfgTool 定制。烧写完毕，从 EMMC/NAND 启动开发板测试。

## A3.4 ubuntu 系统使用

### A3.4.1 添加新用户

#### 1、创建普通用户
root 直接操作容易误操作损坏系统，新建普通用户，需要高权限时使用 sudo。

```sh
adduser zuozhongkai
```

按提示设置密码，如图 3-1 所示。创建完成后可使用该账号登录。

![图 3-1 新用户添加](photo/Ubuntu-base/3-1%20新用户添加.png)

#### 2、赋予普通用户 sudo 权限
新用户默认不在 sudoers，执行 sudo 会报错：zuozhongkai is not in the sudoers file. This incident will be reported.，如图 3-2 所示。

![图 3-2 错误提示](photo/Ubuntu-base/3-2%20错误提示.png)

切换 root：

```sh
su
```

修改sudoers读写权限：

```sh
chmod u+w /etc/sudoers
vi /etc/sudoers
```

找到
root ALL=(ALL:ALL) ALL

如图 3-3 所示。

![图 3-3 sudoers文件](photo/Ubuntu-base/3-3%20sudoers文件.png)

保存退出，恢复只读权限：

```sh
chmod u-w /etc/sudoers
```

切换普通用户，测试 sudo：

```sh
sudo fdisk -l
```

能打印存储设备信息代表配置成功。

### A3.4.2 网络 DHCP 配置

配置网卡 eth0 开机自动 DHCP 获取 IP：

```sh
su
echo auto eth0 > /etc/network/interfaces.d/eth0
echo iface eth0 inet dhcp >> /etc/network/interfaces.d/eth0
/etc/init.d/networking restart
```

eth1 网卡把命令中 eth0 替换为 eth1 即可。重启开发板网卡自动启用。

### A3.4.3 安装软件

和 PC 端 Ubuntu 一致，直接 apt 安装，示例安装 mplayer：

```sh
sudo apt-get install mplayer
```

### A3.4.4 FTP 服务器搭建

安装vsftpd搭建 FTP 服务，配合 FileZilla 做主机-开发板文件互传，参考文档 4.1 小节。 配置完成后 FileZilla 连接，如图 3-4、图 3-5 所示。

![图 3-4 站点设置](photo/Ubuntu-base/3-4%20站点设置.png)
![图 3-5 FTP访问开发板](photo/Ubuntu-base/3-5%20FTP访问开发板.png)

### A3.4.5 gcc 和 make 工具安装

开发板本地编译，安装编译工具链：

```sh
sudo apt-get install gcc
sudo apt-get install make
```

版本查看：

```sh
gcc -v
make -v
```

如图 3-6 所示，gcc 版本为 5.4.0。

![图 3-6 gcc版本号](photo/Ubuntu-base/3-6%20gcc版本号.png)

简单测试代码 `main.c`：

```c
#include <stdio.h>

int main(void)
{
    printf("hello world!\r\n");
    return 0;
}
```

板上编译运行：

```sh
gcc main.c -o main
file main
./main
```

如图 3-7 所示，生成 32 位 ARM 可执行程序，打印hello world!。

![图 3-7 main文件属性](photo/Ubuntu-base/3-7%20main文件属性.png)

注意：I.MX6ULL 性能弱，只适合编译简单小程序，复杂代码编译速度极慢，不推荐板上本地编译。

到此 I.MX6ULL 移植无图形版 Ubuntu-base 全部完成。受硬件性能限制不运行 GUI，可根据项目需求继续定制。