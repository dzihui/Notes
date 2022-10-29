## 一、 Install fedora
使用U盘安装fedora36 workstation。

## 二、 Install softwares
### 1. 安装基础软件和第三方库
```
dnf install -y ncurses-devel
dnf install -y vim
dnf install -y gcc
dnf install -y gcc-c++
dnf install -y gdb
dnf install -y make
dnf install -y ctags
dnf install -y cscope
dnf install -y ntfs-3g
dnf install -y nfs-utils
dnf install -y git
dnf install -y p7zip
dnf install -y unrar
dnf install -y zip unzip
dnf install -y bzip2
dnf install -y bison
dnf install -y wget
dnf install -y net-tools
dnf install -y tcpdump
dnf install -y bind-utils
dnf install -y usbutils
dnf install -y nmap
dnf install -y ftp
dnf install -y zlib
dnf install -y zlib-devel
dnf install -y terminator
dnf install -y minicom
dnf install -y cloc
dnf install -y dwarves
dnf install -y gawk
dnf install -y flex
dnf install -y bison
dnf install -y openssl
dnf install -y openssl-devel
dnf install -y dkms
dnf install -y elfutils-libelf-devel
dnf install -y autoconf
```

### 2. 配置vim

### 3. Install google-chrome-stable
dnf install -y google-chrome-stable

### 4. 搭建ARM64交叉编译环境
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

### 5. 安装qemu
dnf install -y qemu

### 6. 安装atom
现在atom官网 https://atom.io/下载atom rpm安装包atom.x86_64.rpm。
然后执行sudo yum install atom.x86_64.rpm -y

## 三、Upgrade fedora system
dnf upgrade --refresh
dnf install dnf-plugin-system-upgrade -y
dnf system-upgrade download --releasever=32 -y
dnf system-upgrade reboot
