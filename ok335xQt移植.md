# OK335xS-II Qt5.7.0移植

## 环境
Host：Ubuntu 12.04 LTS
TI-SDK：[ti-sdk-am335x-evm-06.00.00.00-Linux-x86](http://software-dl.ti.com/sitara_linux/esd/AM335xSDK/06_00_00_00/index_FDS.html)
交叉编译器：[gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf](https://releases.linaro.org/components/toolchain/binaries/4.9-2017.01/arm-linux-gnueabihf/)
linux kernel：飞凌提供的 linux-kernel-3.2.0
文件系统：[busybox-1.31.0](https://busybox.net/downloads/busybox-1.31.0.tar.bz2)

参考文章：[为AM335x+Linux移植SGX+OpenGL+Qt5之完全开发笔记](https://www.eefocus.com/marianna/blog/15-04/311437_e4f0a.html)
## 设置编译器

```shell
export PATH=/home/xmark/00_app/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin/:$PATH
```<!-- slide -->


## kernel编译

```shell
mkdir kernel
tar xvf kernel.tar.bz2 –C kernel
cd kernel
cp arch/arm/configs/ok335xs2_evm_linux_defconfig .config
make CROSS_COMPILE=arm-linux-gnueabihf- ARCH=arm uImage
make CROSS_COMPILE=arm-linux-gnueabihf- ARCH=arm modules INSTALL_MOD_PATH=/home/xmark/01_am335x/build/rootfs/
```

内核错误
```shell
Can't use 'defined(@array)' (Maybe you should just omit the defined()?) at kernel/timeconst.pl line 373.

/root/working/Hi3520D_SDK_V2.0.3.0/osdrv/kernel/linux-3.0.y/kernel/Makefile:140: recipe for target 'kernel/timeconst.h' failed

make[1]: *** [kernel/timeconst.h] Error 255

Makefile:945: recipe for target 'kernel' failed

make: *** [kernel] Error 2
```
[解决办法](https://my.oschina.net/hanxiaodong/blog/755703)
```
将kernel/timeconst.pl中第373行的defined()去掉只留下@val就可以了
```

##dropbear ssh
https://wiki.beyondlogic.org/index.php?title=Cross_Compiling_BusyBox_for_ARM
https://blog.csdn.net/code_style/article/details/61928328
http://wiki.andreas-duffner.de/index.php/Ssh,_error:_openpty:_No_such_file_or_directory


## 文件系统
### build busybox
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- all
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- install

```shell
In file included from scripts/kconfig/lxdialog/checklist.c:24:0:
scripts/kconfig/lxdialog/dialog.h:31:20: fatal error: curses.h: No such file or directory
compilation terminated.
make[2]: *** [scripts/kconfig/lxdialog/checklist.o] Error 1
make[1]: *** [menuconfig] Error 2
make: *** [menuconfig] Error 2
```
原因：出现该错误的原因是在使用menuconfig时，需要ncurses库的支持。
解决办法：sudo apt-get install libncurses5-dev libncursesw5-dev

### 新建文件系统

```shell
#拷贝busybox编译生成文件
cp -r busybox-1.31.0/_install/ ./rootfs
#拷贝依赖动态库
cp -r ~/00_app/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/arm-linux-gnueabihf/libc/lib/ ./rootfs/lib
#新建etc目录
 mkdir etc etc/rc0.d etc/rc1.d etc/rc2.d etc/rc3.d etc/rc4.d etc/rc5.d etc/rc6.d etc/rcS.d
#从ti提供的文件系统中拷贝相关文件
cp ./ti-SDK-06/rootfs-sdk/etc/group etc/group
cp ./ti-SDK-06/rootfs-sdk/etc/passwd etc/passwd
cp ./ti-SDK-06/rootfs-sdk/etc/default/rcS etc/default/rcS
cp ./ti-SDK-06/rootfs-sdk/etc/inittab etc/inittab
cp ./ti-SDK-06/rootfs-sdk/etc/fstab etc/fstab
cp ./ti-SDK-06/rootfs-sdk/etc/init.d/rc etc/init.d/rc
cp ./ti-SDK-06/rootfs-sdk//etc/init.d/rcS etc/init.d/rcS
#创建其他目录
mkdir dev home home/root media mnt opt proc sys var lib/modules lib/modules/3.2.0

```

### SD卡启动系统

使用TI SDK中的create-sdcard.sh脚本格式化SD卡，分区选中boot和rootfs两个分区
```shell
#因为Ubuntu系统中 sfdisk版本不同在Ubuntu 18.04下运行TI-SDK-AM335x-evm-06下的create-sdcard脚本报错
sfdisk : Invaild arg  -D 
# Ubuntu 12.04 TLS版本不会有问题  
```

```shell
#拷贝飞凌提供的MLO u-boot.img logo.bmp 到SD卡boot区
#拷贝rootfs到SD卡rootfs区
```
SD卡启动系统错误：
```shell
[    3.545816] kjournald starting.  Commit interval 5 seconds
[    3.551706] EXT3-fs (mmcblk0p2): recovery complete
[    3.562661] EXT3-fs (mmcblk0p2): mounted filesystem with ordered data mode
[    3.569990] VFS: Mounted root (ext3 filesystem) readonly on device 179:2.
[    3.577295] devtmpfs: mounted<!-- slide -->
[    3.580982] Freeing init memory: 256K
Bad inittab entry at line 5
can't open /dev/si: No such file or directory
can't open /dev/~~: No such file or directory
can't open /dev/l0: No such file or directory

```
原因分析：busybox启动程序/sbin/init会读取`/etc/inittab`的初始化job然后依次执行命令。飞凌的inittab配置需要修改
[参考](https://www.cyberciti.biz/howto/question/man/inittab-man-page.php)
改为以下内容
```shell
# /etc/inittab: init(8) configuration.
# $Id: inittab,v 1.91 2002/01/25 13:35:21 miquels Exp $

# The default runlevel.
id:5:initdefault:

# Boot-time system configuration/initialization script.
# This is run first except when booting in emergency (-b) mode.

::sysinit:/etc/init.d/rcS
#::respawn:/sbin/getty  115200  ttyO0
ttyO0::askfirst:-/bin/sh #ttyO0是默认的控制串口
::restart:/sbin/init
::ctrlaltdel:/bin/umount -a -r
```


## build TI SGX

修改Graphics_SDK_4_09_00_0/Rules.make文件
```shell
#Graphics安装目录
GRAPHICS_INSTALL_DIR=${HOME}/Graphics_SDK_#_##_##_##
#交叉编译工具位置
CSTOOL_DIR = /home/<user_account>/toolchain/arm-2009q1
#toolchain prefix
CSTOOL_PREFIX=arm-linux-gnueabihf-
# linux内核位置 Set kernel installation path ( ex /home/user/linux-04.00.01.13 )
KERNEL_INSTALL_DIR=/home/xmark/01_am335x/build/kernel
# 目标文件系统位置 Set Target filesystem path ( ex /home/user/targetfs )
TARGETFS_INSTALL_DIR= /home/xmark/01_am335x/build/rootfs/
```
编译
```shell
#enviournment 
export ARCH=arm
#编译 debug出错会有信息打印出来便于调试
make BUILD=release OMAPES=8.x FBDEV=yes SUPPORT_XORG=0 PM_RUNTIME=1 CROSS_COMPILE=arm-linux-gnueabihf- all
#安装
make BUILD=release OMAPES=8.x FBDEV=yes SUPPORT_XORG=0 PM_RUNTIME=1 install
```
安装完成在rootfs/opt下会产生gfxlibraries、gfxsdkdemos两个目录拷贝这两个目录到SD卡文件系统
进入AM335x系统运行 /opt/gfxsdkdemos/335x-demo
```shell
Installation complete!
You may now reboot your target.

[  109.471125] ------------[ cut here ]------------
[  109.476108] WARNING: at fs/sysfs/dir.c:481 sysfs_add_one+0xa0/0xb0()
[  109.482808] sysfs: cannot create duplicate filename '/bus/platform/devices/pvrsrvkm'
[  109.490975] Modules linked in: pvrsrvkm(O+)
[  109.495403] Backtrace:
[  109.498017] [<c0018408>] (dump_backtrace+0x0/0x10c) from [<c04dee5c>] (dump_stack+0x18/0x1c)
[  109.506921]  r7:000001e1 r6:c05fbf90 r5:00000009 r4:c700bd38
[  109.512930] [<c04dee44>] (dump_stack+0x0/0x1c) from [<c003ea54>] (warn_slowpath_common+0x5c/0x6c)
[  109.522302] [<c003e9f8>] (warn_slowpath_common+0x0/0x6c) from [<c003ea9c>] (warn_slowpath_fmt+0x38/0x40)
[  109.532299]  r9:00000000 r8:00000001 r7:c5841000 r6:c5841000 r5:c76e3770
[  109.539205] r4:c05fbfa0
[  109.541977] [<c003ea64>] (warn_slowpath_fmt+0x0/0x40) from [<c01012c8>] (sysfs_add_one+0xa0/0xb0)
[  109.551357]  r3:c5841000 r2:c05fbfa0
[  109.555141]  r4:ffffffef
[  109.557818] [<c0101228>] (sysfs_add_one+0x0/0xb0) from [<c0101a14>] (sysfs_do_create_link+0xd4/0x1f8)
[  109.567540]  r7:c700bd90 r6:c7013db0 r5:c76e3770 r4:c76e3900
[  109.573541] [<c0101940>] (sysfs_do_create_link+0x0/0x1f8) from [<c0101b4c>] (sysfs_create_link+0x14/0x18)
[  109.583629]  r9:c076e918 r8:bf08c4c8 r7:00000000 r6:c071cd78 r5:bf08c4c8
[  109.590525] r4:bf08c4d0
[  109.593310] [<c0101b38>] (sysfs_create_link+0x0/0x18) from [<c0239ab4>] (bus_add_device+0xfc/0x1b8)
[  109.602866] [<c02399b8>] (bus_add_device+0x0/0x1b8) from [<c0237d5c>] (device_add+0x4d0/0x5e0)
[  109.611949]  r9:c076e918 r8:bf08c4c8 r7:bf08c5e8 r6:00000000 r5:c071cc48
[  109.618861] r4:bf08c4d0
[  109.621639] [<c023788c>] (device_add+0x0/0x5e0) from [<c023c160>] (platform_device_add+0x108/0x1ec)
[  109.631193] [<c023c058>] (platform_device_add+0x0/0x1ec) from [<c023c714>] (platform_device_register+0x28/0x2c)
[  109.641834]  r7:c7230564 r6:00000000 r5:00000000 r4:bf08c4c0
[  109.647969] [<c023c6ec>] (platform_device_register+0x0/0x2c) from [<bf09907c>] (PVRCore_Init+0x7c/0x164 [pvrsrvkm])
[  109.658982]  r5:00000000 r4:bf08e37c
[  109.662815] [<bf099000>] (PVRCore_Init+0x0/0x164 [pvrsrvkm]) from [<c0008770>] (do_one_initcall+0x12c/0x184)
[  109.673184]  r7:c7230564 r6:c7230540 r5:c700a028 r4:c074aa80
[  109.679184] [<c0008644>] (do_one_initcall+0x0/0x184) from [<c006cde0>] (sys_init_module+0x958/0x1bdc)
[  109.688927] [<c006c488>] (sys_init_module+0x0/0x1bdc) from [<c0014740>] (ret_fast_syscall+0x0/0x30)
[  109.698474] ---[ end trace ea945ba47dc9849a ]---
Module pvrsrvkm failed to load. Retrying.

```
`This means the kernel sources you are using has the SGX HWMOD support integrated but you are not building graphics SDK with PM_RUNTIME=1 command line option. Check the kernel sources to have the SGX HWMOD support. Check if the changes in below patch is already present in kernel sources you are using - http://gitorious.org/rowboat/kernel/commit/d3ab5e87304b45dc594351ed091a6ae451f40de7?format=patch

In this case, please build graphics SDK with PM_RUNTIME=1 command line option & that should help in solving the above error/issue.`
## 编译Qt

### 编译tslib1.4

```shell
./autogen.sh
echo "ac_cv_func_malloc_0_nonnull=yes" >arm-linux.cache
./configure --host=arm-linux --prefix=`pwd`/_install CC=arm-linux-gnueabihf-gcc --cache-file=arm-linux.cache
make && make install
```

### 不支持openGL的Qt编译

```shell
#修改qt-everywhere-opensource-src-5.6.3/qtbase/mkspace/linux-arm-guneabi-g++/qmake.conf如下

#
# qmake configuration for building with arm-linux-gnueabi-g++
#

MAKEFILE_GENERATOR      = UNIX
CONFIG                 += incremental
QMAKE_INCREMENTAL_STYLE = sublib

include(../common/linux.conf)
include(../common/gcc-base-unix.conf)
include(../common/g++-unix.conf)

QT_QPA_DEFAULT_PLATFORM = linuxfb  #使用linuxfb 软件渲染，SGX还未移植成功                                             
QMAKE_CFLAGS_RELEASE   += -O2 -march=armv7-a                 
QMAKE_CXXFLAGS_RELEASE += -O2 -march=armv7-a
#tslib相关
QMAKE_INCDIR += /home/xmark/am335x/ok335x/tslib/include
QMAKE_LIBDIR += /home/xmark/am335x/ok335x/tslib/lib

#../common/linux.conf中有如下内容 与 x11 nsl EGL opengl openvg udev等相关的 
# 在arm-linux不需要x11全部重新赋值为空
# QMAKE_LIBS              =
# QMAKE_LIBS_DYNLOAD      = -ldl
# QMAKE_LIBS_X11          = -lXext -lX11 -lm
# QMAKE_LIBS_NIS          = -lnsl
# QMAKE_LIBS_EGL          = -lEGL
# QMAKE_LIBS_OPENGL       = -lGL
# QMAKE_LIBS_OPENGL_ES2   = -lGLESv2
# QMAKE_LIBS_OPENVG       = -lOpenVG
# QMAKE_LIBS_THREAD       = -lpthread
# QMAKE_LIBS_LIBUDEV      = -ludev
QMAKE_LIBS_X11          = 
QMAKE_LIBS_NIS          = 
QMAKE_LIBS_EGL          = 
QMAKE_LIBS_OPENGL       = 
QMAKE_LIBS_OPENGL_ES2   = 
QMAKE_LIBS_OPENVG       = 
QMAKE_LIBS_THREAD       = -lpthread
QMAKE_LIBS_LIBUDEV      = 

# modifications to g++.conf
QMAKE_CC                = arm-arago-linux-gnueabi-gcc
QMAKE_CXX               = arm-arago-linux-gnueabi-g++
QMAKE_LINK              = arm-arago-linux-gnueabi-g++
QMAKE_LINK_SHLIB        = arm-arago-linux-gnueabi-g++

# modifications to linux.conf
QMAKE_AR                = arm-arago-linux-gnueabi-ar cqs
QMAKE_OBJCOPY           = arm-arago-linux-gnueabi-objcopy
QMAKE_NM                = arm-arago-linux-gnueabi-nm -P
QMAKE_STRIP             = arm-arago-linux-gnueabi-strip
load(qt_config)

```

### Qt 5.7
参考[TI wiki](http://processors.wiki.ti.com/index.php/Building_Qt_with_OpenGL_ES_accelerated_by_SGX#Building_Qt_with_OpenGL_ES)

```shell
cd qt-everywhere-opensource-src-5.7.0/qtbase/

./configure \
-release -opensource -confirm-license -shared \
-prefix /home/xmark/01_am335x/build/qt/5.7.0/arm-linux-gunebif-linaro-4.9.4 \
-platform linux-g++-64 \
-device linux-TIarmv7-sgx-g++ \
-device-option CROSS_COMPILE=/home/xmark/00_app/gcc-linaro-4.9.4-2017.01-x86_64_arm-linux-gnueabihf/bin/arm-linux-gnueabihf- \
-eglfs -opengl es2 -qreal float -v \
-qt-sql-sqlite \
-qt-libjpeg \
-qt-libpng \
-qt-zlib \
-tslib \
-nomake examples -nomake tests -no-iconv

#在ubuntu12.04下出现如下错误
create qmake.....
unrecognized command line option “-std=c++11”
#原因是ubuntu12.04自带的gcc版本不够高
#添加源（Ubuntu）
$ sudo add-apt-repository ppa:ubuntu-toolchain-r/test
$ sudo apt-get update
#安装gcc 4.8版本
$ sudo apt-get install gcc-4.8 g++-4.8
#查看本地安装版本
$ ls -lh /usr/bin/g++*
#这里应该可以看到本机安装了4.6和4.8两个版本。
#切换版本
$ sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.6 60 --slave /usr/bin/g++ g++ /usr/bin/g++-4.6
$ sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-4.8 40 --slave /usr/bin/g++ g++ /usr/bin/g++-4.8
$ sudo update-alternatives --config gcc
#**选择4.8版本的序号**

#再次查看g++版本，确认为 4.8 版本
$ g++ --version

```

制作UBI文件系统
```shell
#使用飞凌提供的mkfs和ubinize，使用ubuntu安装版本有问题
./mkfs.ubifs -F -r rootfs -m 2048 -e 126976 -c 1866 -o ubifs.img
./ubinize -o ubi.img -m 2048 -p 128KiB -s 2048 -O 2048 ubinize-256M.cfg
```

tslib配置https://doc.qt.io/qt-5/embedded-linux.html#eglfs

运行环境配置
```shell
export QTDIR=/mnt/zkkr/qt-5.7.0

export LD_LIBRARY_PATH=$QTDIR/lib:$LD_LIBRARY_PATH
export PATH=$QTDIR/bin:$PATH

export ZKKR_DIR=/mnt/zkkr
export QT_DIR=$ZKKR_DIR/qt-5.7.0
export TSLIB_ROOT=$ZKKR_DIR/tslib

export LD_LIBRARY_PATH=$QT_DIR/lib:$TSLIB_ROOT/lib:$LD_LIBRARY_PATH
export PKG_CONFIG_PATH=$QT_DIR/lib/pkgconfig/$PKG_CONFIG_PATH

export TSLIB_CONFFILE=$TSLIB_ROOT/etc/ts.conf
export TSLIB_PLUGINDIR=$TSLIB_ROOT/lib/ts
export TSLIB_CALIBFILE=/etc/pointercal
export TSLIB_CONSOLEDEVICE=none
export TSLIB_FBDEVICE=/dev/fb0
export TSLIB_TSDEVICE=/dev/input/event0
export TSLIB_TSEVENTTYPE=INPUT

export QT_QPA_PLATFORM=eglfs
export QT_QPA_EGLFS_FB=/dev/fb0
export QT_QPA_EGLFS_WIDTH=1024
export QT_QPA_EGLFS_HEIGHT=600
export QT_QPA_EGLFS_PHYSICAL_WIDTH=154.08
export QT_QPA_EGLFS_PHYSICAL_HEIGHT=85.92
export QT_QPA_EGLFS_TSLIB=1
#export QT_DEBUG_PLUGINS=1

export QML2_IMPORT_PATH=$QT_DIR/qml
export QT_QPA_FONTDIR=$QT_DIR/lib/fonts
export QT_QPA_PLATFORM_PLUGIN_PATH=$QT_DIR/plugins
export QT_QPA_GENERIC_PLUGINS=tslib:$TSLIB_TSDEVICE
```

OK335x Nand分区信息
```shell
nandrootfstype=yaffs2 rootwait=1
nandrootfstype=ubi rootwait=1

env set nandrootfstype "ubi rootwait=1"

led all on;
setenv TYPE 0;
nand erase.chip;
mmc rescan; 
setenv TYPE 1;
fatload mmc 0 80A00000 MLO;
setenv TYPE 2;
nand write.i 80A00000 0 ${filesize}; 
setenv TYPE 2;
nand write.i 80A00000 0x20000 ${filesize}; 
setenv TYPE 2;
nand write.i 80A00000 0x40000 ${filesize}; 
setenv TYPE 2;
nand write.i 80A00000 0x60000 ${filesize}; 
setenv TYPE 2;
nand write.i 80A00000 0x80000 ${filesize}; 
setenv TYPE 2;
nand write.i 80A00000 0x100000 ${filesize}; 
setenv TYPE 2;
nand write.i 80A00000 0x180000 ${filesize}; 
setenv TYPE 3;
fatload mmc 0 80A00000 u-boot.img;
setenv TYPE 4;
nand write.i 80A00000 800000 ${filesize}; 
setenv TYPE 4;
nand write.i 80A00000 400000 ${filesize}; 
setenv TYPE 5;
fatload mmc 0 80A00000 uImage;    
setenv TYPE 6;
nand write.i 80A00000 c00000 ${filesize}; 
setenv TYPE 7;
fatload mmc 0 80A00000 ubi.img;     
setenv TYPE 8;
nand write.i 80A00000 1400000 ${filesize};
setenv TYPE 9;
fatload mmc 0 80A00000 logo.bmp;  
setenv TYPE 10;
nand write.i 80A00000 600000 ${filesize};
```

