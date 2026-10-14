# 附录B Yocto构建嵌入式Linux根文件系统
> 前面我们学习了三种根文件系统构建方式：`busybox（纯手动）`、`buildroot（配置菜单编译）`、`ubuntu‑base（官方预编译arm根文件系统chroot改造）`。本章学习 **Yocto Project**，Yocto是一套更加专业、工业级嵌入式Linux构建框架，NXP i.MX系列芯片官方主推，适合产品级项目开发。

Buildroot是单配置菜单，而Yocto采用**层(Layer)** 机制，模块化管理BSP、软件包、自定义应用，支持定制U‑Boot、Linux内核、根文件系统、SDK工具链全套组件。

## B1 Yocto简介
Yocto Project不是源代码，是一套**编译构建框架**，基于BitBake任务引擎。NXP针对i.MX6ULL提供`fsl‑community‑bsp`社区BSP包，使用repo工具管理大量git仓库。

- busybox：适合学习原理，产品需要大量手动补齐；
- buildroot：配置菜单，上手简单，软件包整合；
- **Yocto**：工业产品，分层管理，可做SDK、镜像裁剪，NXP官方维护i.MX BSP。

> 硬件主机要求：Ubuntu 18.04/20.04，建议内存≥16GB，磁盘空闲空间≥150GB，编译全程需要网络下载源码包，**普通非root用户编译，不要使用sudo执行bitbake**。

## B2 主机环境准备
安装Yocto编译依赖包
```bash
sudo apt‑get update
sudo apt‑get install gawk wget git‑core diffstat unzip texinfo gcc‑multilib \
build‑essential chrpath socat cpio python3 python3‑pexpect python3‑jinja2 \
pylint3 rsync bc bison xz‑utils debianutils iputils‑ping libsdl1.2‑dev xterm
```

## B3 获取NXP i.MX Yocto源码（repo工具）
1. 安装repo工具
```bash
mkdir ~/bin
curl `https://storage.googleapis.com/git`‑repo‑downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
echo "export PATH=\$PATH:~/bin" >> ~/.bashrc
source ~/.bashrc
```

2. 创建工作目录，拉取fsl‑release‑yocto源码（imx‑linux‑zeus分支，适配i.MX6ULL）
```bash
mkdir -p ~/Linux/Yocto/fsl‑release‑yocto
cd ~/Linux/Yocto/fsl‑release‑yocto
repo init -u `https://github.com/nxp`‑imx/imx‑manifest.git -b imx‑linux‑zeus
repo sync
```
repo sync会下载所有meta‑xxx层、poky核心、BSP源码，耗时视网络而定。下载完成后目录下会存在`sources`文件夹，存放各个meta层。

## B4 初始化Yocto构建目录 imx‑setup‑release.sh
NXP提供`imx‑setup‑release.sh`脚本，用来生成build构建工作目录，自动生成两个核心配置文件：
1. `conf/bblayers.conf`：管理所有meta层（layer），告诉bitbake去哪里找配方recipes；
2. `conf/local.conf`：本地编译配置，定义`MACHINE`、`DISTRO`、编译线程、下载缓存目录。

> 关键两个环境变量
> - `MACHINE`：**目标硬件平台**，i.MX6ULL参考板：`imx6ull14x14evk`，代表硬件BSP配置；
> - `DISTRO`：**发行版类型**；`fsl‑imx‑fb` framebuffer无图形；还有`fsl‑imx‑xwayland`图形桌面版本。
> - `-b build`：指定生成的build工作目录名字。

执行初始化脚本：
```bash
cd ~/Linux/Yocto/fsl‑release‑yocto
DISTRO=fsl‑imx‑fb MACHINE=imx6ull14x14evk source imx‑setup‑release.sh -b build
```
脚本运行会弹出EULA许可协议，按`q`退出阅读，输入`y`接受许可。
脚本执行完毕，会自动进入`build`目录。

> ⚠️关闭终端之后，环境会失效，再次进入编译环境执行：
```bash
cd ~/Linux/Yocto/fsl‑release‑yocto
source setup‑environment build
```

### 两个核心配置文件说明
1. **bblayers.conf**
记录所有meta层路径，Yocto的Layer分层思想：
- `poky/meta`：Yocto核心基础层
- `poky/meta‑yocto`：Yocto参考配置
- `meta‑imx/meta‑bsp`：NXP硬件BSP（U‑Boot、内核、设备树）
- `meta‑imx/meta‑sdk`：SDK与发行版配置
- `meta‑openembedded/*`：大量第三方软件recipes

新增自定义软件或者板级修改，**不要直接修改官方meta层，新建自己meta‑xxx层，把路径添加进bblayers.conf**。

2. **local.conf**
- `MACHINE`：目标板卡；
- `DISTRO`：系统发行版；
- `BB_NUMBER_THREADS`：bitbake任务线程数；
- `PARALLEL_MAKE`：编译进程数量；
- `DL_DIR`：源码包下载缓存目录，多build目录可以共享，节省重复下载。

## B5 bitbake编译根文件系统镜像
进入build目录后，使用`bitbake`命令编译镜像。
常用镜像target：
| target |说明 |
|---|---|
| `core‑image‑minimal` | 极简控制台根文件系统，最小，无多余工具 |
| `fsl‑image‑core` | NXP官方基础镜像，包含i.MX外设工具 |
| `fsl‑image‑gui` | 带图形界面镜像 |

编译最小控制台镜像：
```bash
bitbake core‑image‑minimal
```
> ⚠️**第一次编译耗时非常久**，会下载、交叉编译工具链、内核、uboot、所有软件包。
> 禁止使用sudo执行bitbake。

编译成功后输出产物路径：
```
build/tmp/deploy/images/imx6ull14x14evk/
```
该目录下会生成：
1. `core‑image‑minimal‑imx6ull14x14evk.tar.bz2`：根文件系统压缩包，可以用于NFS挂载测试；
2. `core‑image‑minimal‑imx6ull14x14evk.wic`：完整SD卡镜像，包含uboot、dtb、zImage、rootfs，直接dd烧写到SD卡；
3. zImage内核、设备树dtb文件、u‑boot.bin。

## B6 NFS挂载测试Yocto根文件系统
1. 将生成的rootfs压缩包拷贝到nfs工作目录解压
```bash
cd ~/linux/nfs
mkdir yocto_rootfs
cd yocto_rootfs
tar -xvf ../../fsl‑release‑yocto/build/tmp/deploy/images/imx6ull14x14evk/core‑image‑minimal‑imx6ull14x14evk.tar.bz2
```
2. U‑Boot设置bootargs，和busybox/buildroot/ubuntu‑base测试方式完全一样：
```bash
setenv bootargs 'console=tty1 console=ttymxc0,115200 root=/dev/nfs nfsroot=192.168.1.253:/home/zuozhongkai/linux/nfs/yocto_rootfs rw ip=192.168.1.251:192.168.1.253:192.168.1.1:255.255.255.0::eth0:off'
saveenv
```
启动开发板，通过NFS挂载Yocto生成的rootfs，登录测试。

## B7 烧写镜像到SD卡 / EMMC
### 方式1：使用wic完整镜像直接烧SD卡
```bash
sudo dd if=core‑image‑minimal‑imx6ull14x14evk.wic of=/dev/sdb bs=1M status=progress
```
> /dev/sdb为主机的SD卡设备节点，千万不要写错，避免写坏主机硬盘。

### 方式2：使用MfgTool烧写EMMC
把`rootfs.tar.bz2`替换MfgTool中的rootfs压缩包，修改脚本，流程同buildroot、ubuntu‑base根文件系统烧写。

## B8 Yocto常用补充操作
1. 只编译单独一个软件包
```bash
bitbake alsa‑utils
```
2. 清理单个包编译产物
```bash
bitbake alsa‑utils -c clean
```
3. 编译生成交叉SDK（在主机上编译目标板应用程序）
```bash
bitbake core‑image‑minimal -c populate_sdk
```
SDK输出在 `tmp/deploy/sdk`，得到sh脚本，主机运行脚本安装交叉编译工具链。

## B9 Buildroot / BusyBox / Ubuntu‑base / Yocto对比总结
|构建方案 |原理 |适合场景 |特点 |
|---|---|---|---|
| BusyBox |手动创建目录+busybox命令，手动补齐库、配置 |**学习根文件原理** |工作量大，产品需要大量手动完善 |
| Buildroot |菜单配置，一键编译整套rootfs |中小型产品，快速开发 |上手简单，单一大工程，分层能力弱 |
| Ubuntu‑base |官方预编译arm根文件系统，qemu‑arm‑static chroot改造 |想要Debian/Ubuntu软件生态 |软件包丰富，体积大，裁剪麻烦 |
| Yocto |BitBake+Layer分层，每个软件为recipe配方 |**工业产品、NXP官方BSP** |模块化，可生成镜像/SDK，学习门槛高 |

> 注意：Yocto学习门槛最高，NXP i.MX系列官方产品推荐使用Yocto；学习阶段优先学busybox，做原型快速验证优先buildroot；想要apt包管理选ubuntu‑base。

> 常见踩坑：
> 1. 不要用sudo执行bitbake，会改变文件权限后续编译报错；
> 2. 磁盘空间一定要充足，第一次编译上百GB占用；
> 3. 关闭终端后必须重新source `setup‑environment build`恢复bitbake环境；
> 4. 不要修改官方meta层，新增功能新建自己meta层。
>