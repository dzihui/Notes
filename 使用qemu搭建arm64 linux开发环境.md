# 使用qemu搭建arm64 linux开发环境
## 1. 简介
本文描述了使用qemu-system-aarch64搭建arm64 linux开发环境的过程，主要包括如下内容：
* 安装qemu
* 安装aarch64交叉编译工具链
* 编译busybox并制作根文件系统
* 编译Linux内核镜像
* 通过qemue虚拟机运行内核镜像
* 设置Host主机与qemu虚拟机间文件共享
* 内核module交叉编译和在qemu虚拟机上运行。

用来搭建实验环境的Host主机OS以及交叉编译工具链等信息如下：
* 主机环境：fedor34 x86_64
* aarch64交叉编译工具链：gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu
* qemu-system-aarch64版本：V5.2.0
* busybox版本：busybox-1.32.1
* linux内核版本：linux-5.4

搭建过程主要参考了如下博客：
https://blog.csdn.net/21cnbao/article/details/116810865

## 2. 安装qemu
```
$ sudo dnf install -y qemu
$ qemu-system-aarch64 --version
QEMU emulator version 5.2.0 (qemu-5.2.0-7.fc34)
Copyright (c) 2003-2020 Fabrice Bellard and the QEMU Project developers
```

## 3. 安装aarch64交叉编译工具
* 在ARM官网下载交叉编译工具链  
https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a/downloads

* 解压缩文件到/usr/local/bin目录下
```
sudo cp gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu.tar.xz /usr/local/bin/
cd /usr/local/bin
sudo xz -d gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu.tar.xz
sudo tar xvf gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu.tar
```
解压后生成 /usr/local/bin/gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu/目录。
* 添加环境变量
```
sudo vim /etc/profile
```
在文件最后添加
```
export PATH=$PATH:/usr/local/bin/gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu/bin
```

* 使能环境变量
```
source /etc/profile
```

* 重启计算机
```
sudo reboot
```

* 查看arm-none-eabihf-gcc版本信息
```
aarch64-none-linux-gnu-gcc --version
```

## 4. 编译busybox并制作根文件系统
### 下载busybox
https://busybox.net/downloads/  
下载busybox-1.32.1.tar.bz2

### 编译busybox
* 解压缩busybox-1.32.1.tar.bz2
```
$ mkdir ~/qemu_arm64
$ cp busybox-1.32.1.tar.bz2 ~/qemu_arm64
$ cd ~/qemu_arm64
$ tar xjvf busybox-1.32.1.tar.bz2
$ cd busybox-1.32.1
```
```
$ export ARCH=arm64
$ export CROSS_COMPILE=aarch64-none-linux-gnu-
$ make menuconfig //设置CONFIG_STATIC=y
Settings --->
 [*] Build static binary (no shared libs)
 ```
```
$ make
$ make install
```
编译完成后会在busybox-1.32.1/_install目录下生成如下文件和目录
```
$ cd _install/
$ ls
bin  linuxrc  sbin  usr
```

### 完善根文件目录
执行如下命令在_install目录下创建子目录
```
$ mkdir dev etc lib sys proc tmp var home root mnt
$ ls
bin  dev  etc  home  lib  linuxrc  mnt  proc  root  sbin  sys  tmp  usr  var
```
### 更新etc目录
```
$ cd etc
```
* 添加profile文件
$ vim profile
，文件内容如下：
```
#!/bin/sh
export HOSTNAME=qemu
export USER=root
export HOME=/home
export PS1="[$USER@$HOSTNAME \W]\# "
PATH=/bin:/sbin:/usr/bin:/usr/sbin
LD_LIBRARY_PATH=/lib:/usr/lib:$LD_LIBRARY_PATH
export PATH LD_LIBRARY_PATH
```

* 添加inittab文件
$ vim inittab
文件内容如下：
```
::sysinit:/etc/init.d/rcS
::respawn:-/bin/sh
::askfirst:-/bin/sh
::ctrlaltdel:/bin/umount -a -r
```

* 添加fstab文件
$ vim fstab
文件内容如下：
```
#device  mount-point    type     options   dump   fsck order
proc /proc proc defaults 0 0
tmpfs /tmp tmpfs defaults 0 0
sysfs /sys sysfs defaults 0 0
tmpfs /dev tmpfs defaults 0 0
debugfs /sys/kernel/debug debugfs defaults 0 0
kmod_mount /mnt 9p trans=virtio 0 0
```
启动qemu虚拟机后，执行mount -a命令时，系统会根据/etc/fstab文件中的配置挂载文件系统。
* 创建init.d目录
```
mkdir  init.d
```
在init.d目录下添加rcS文件
$ vim rcS
rcS文件内容如下：
```
mkdir -p /sys
mkdir -p /tmp
mkdir -p /proc
mkdir -p /mnt
/bin/mount -a
mkdir -p /dev/pts
mount -t devpts devpts /dev/pts
echo /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s
```
完成上述操作后，_install/etc/目录如下：
```
$ tree
.
├── fstab
├── init.d
│   └── rcS
├── inittab
└── profile
```

### 更新dev目录
进入_install/dev/目录执行如下命令
```
sudo mknod console c 5 1
```
执行上述命令后在_install/dev/目录下创建了console文件。
```
$ ls
console
```

### 更新lib目录
将交叉编译工具链的lib复制到_install/lib/目录下
```
$ cd lib
$ cp /usr/local/bin/gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu/aarch64-none-linux-gnu/lib64/*.so* -a .
```
至此，我们完成了根文件系统的制作。下一步就是编译内核。

## 5. 编译内核镜像
### 解压缩内核源码
```
$ cp linux-5.4.tar ~/qemu_arm64
$ cd ~/qemu_arm64
$ tar xvf linux-5.4.tar
$ cd linux-5.4
````
### 将根文件系统放到内核源码目录
```
sudo cp ../busybox-1.32.1/_install _install_arm64 -a
```
### 配置内核
* 添加hotplug支持  
````
vim arch/arm64/configs/defconfig
在文件结尾加上
CONFIG_UEVENT_HELPER=y
CONFIG_UEVENT_HELPER_PATH="/sbin/hotplug"
`````````

* 添加initramfs的支持：
```
vim arch/arm64/configs/defconfig
在文件结尾加上
CONFIG_INITRAMFS_SOURCE="_install_arm64"
```

### 编译内核
```
export ARCH=arm64
export CROSS_COMPILE=aarch64-none-linux-gnu-
make defconfig
make all -j8
```

##6 运行qemu
在~/qemu_arm64目录下新建share共享目录和qemu_start.sh文件。
```
$ cd ～/qemu_arm64
$ mkdir share
$ vim qemu_start.sh
```
qemu_start.sh文件内容如下
```
#!/bin/bash
qemu-system-aarch64 \
                -machine virt \
                -cpu cortex-a72 \
                -machine type=virt \
                -m 2048 \
                -smp 4 \
                -kernel linux-5.4/arch/arm64/boot/Image \
                --append "rdinit=/linuxrc root=/dev/vda rw console=ttyAMA0 loglevel=8"  \
                -nographic \
                --fsdev local,id=kmod_dev,path=$PWD/share,security_model=none \
                -device virtio-9p-device,fsdev=kmod_dev,mount_tag=kmod_mount
```
设置qemu_start.sh可执行权限
```
$ chmod +x qemu_start.sh
```
执行qemu_start.sh脚本运行虚拟机
```
$ qemu_start.sh
```
虚拟机启动成功后会打印命令行提示符，执行uname -ra命令可以查看那linux内核版本。
```
Please press Enter to activate this console.
[root@localhost ]#
[root@localhost ]#
[root@localhost ]# uname -ra
Linux (none) 5.4.0 #1 SMP PREEMPT Sun Aug 8 10:26:39 CST 2021 aarch64 GNU/Linux
[root@localhost ]#
```

## 7. 模拟磁盘
```
$ cd ～/qemu_arm64
$ dd if=/dev/zero of=rootfs_ext4.img bs=1M count=1024
$ mkfs.ext4 rootfs_ext4.img
$ mkdir -p tmpfs
$ sudo mount -t ext4 rootfs_ext4.img tmpfs/ -o loop
$ sudo cp -af linux-5.4/_install_arm64/* tmpfs/
$ sudo umount tmpfs/
$ rm -rf tmpfs/
```
修改qemu_start.sh脚本内容为：
```
#!/bin/bash
qemu-system-aarch64 \
                -machine virt \
                -cpu cortex-a72 \
                -machine type=virt \
                -m 4096 \
                -smp 4 \
                -kernel linux-5.4/arch/arm64/boot/Image \
                --append "noinitrd root=/dev/vda rw console=ttyAMA0 loglevel=8"  \
                -nographic \
                -drive if=none,file=rootfs_ext4.img,id=hd0 \
                -device virtio-blk-device,drive=hd0 \
                --fsdev local,id=kmod_dev,path=$PWD/share,security_model=none \
                -device virtio-9p-device,fsdev=kmod_dev,mount_tag=kmod_mount
```
执行./qemu_start.sh启动虚拟机。
进入虚拟机后执行df -mh发现会报如下错误：
```
Please press Enter to activate this console.
[root@localhost ]# df -mh
Filesystem                Size      Used Available Use% Mounted on
df: /proc/mounts: No such file or directory
```
执行mount -a命令手动挂载后再查看。
```
root@localhost ]# mount -a
[root@localhost ]# df -mh
Filesystem                Size      Used Available Use% Mounted on
/dev/root               975.9M     60.6M    848.1M   7% /
devtmpfs                  1.9G         0      1.9G   0% /dev
tmpfs                     1.9G         0      1.9G   0% /tmp
kmod_mount              390.3G     34.9G    335.5G   9% /mnt
[root@localhost ]#
```

## 7. 主机向虚拟机共享文件
### 共享shell脚本
在主机侧~/qemu_arm64/share/目录下新建hello.sh文件，文件内容如下：
```
#!/bin/sh
echo "hello qemu"
```
这时在虚拟机的 /mnt/目录下即可看到该为文件。
```
[root@localhost ]# cd /mnt
[root@localhost mnt]# ls
hello.sh
[root@localhost mnt]# chmod +x hello.sh
[root@localhost mnt]# ./hello.sh
hello qemu
[root@localhost mnt]#
```

### 共享交叉编译的可执行文件
* 在主机侧~/qemu_arm64/share/目录下编写hello.c文件，内容如下：
```
#include <stdio.h>
int main(void)
{
        printf("hello qemu\n");

        return 0;
}
```
* 静态交叉编译hello.c文件
```
aarch64-none-linux-gnu-gcc hello.c  -static -o hello
```
__备注：__  
交叉编译编译时需要加上-static选项进行静态编译。

编译完成后在主机侧~/qemu_arm64/share/目录下和虚拟机侧/mnt/目录下可以看到hello文件。
* 在虚拟机侧执行hello文件
```
[root@localhost mnt]# cd /mnt
[root@localhost mnt]# ls
hello     hello.c   hello.sh
[root@localhost mnt]# chmod +x hello
[root@localhost mnt]# ./hello
hello qemu
[root@localhost mnt]#
```
### 退出qemu linux
按\<CTRL\> + a 组合键，再按x，退出qemu。
