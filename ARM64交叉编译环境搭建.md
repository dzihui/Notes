## linux kernel ARM64交叉编译
### Host信息
* OS：Fedora release 33
* arm-gcc：aarch64-none-linux-gnu-gcc_10.2.1
* kernel：linux-5.4

### 安装交叉编译工具
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

### 安装依赖库
在编译过程中会因为缺少各种依赖库而报错，解决办法是根据报错的提示信息安装对应的库。以下是安装过的库：
```
sudo dnf install -y ncurses
sudo dnf install -y ncurses-devel
(sudo dnf install -y  *curses*)

sudo dnf install -y dwarves
sudo dnf install -y gawk
sudo dnf install -y flex
sudo dnf install -y bison
sudo dnf install -y openssl
sudo dnf install -y  openssl-devel
sudo dnf install -y dkms
sudo dnf install -y  elfutils-libelf-devel
sudo dnf install -y autoconf
```
### 下载kernel源代码
到内核官网kernel.org下载kernel源代码（例如linux-5.4），网址：  
https://mirrors.edge.kernel.org/pub/linux/kernel/  

### 编译kernel镜像
#### 生成defconfig
```
cd linux-5.4
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
```
#### 编译镜像
```
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- Image -j8
```

编译过程中可能会报如下错误：
```
/usr/bin/ld: scripts/dtc/dtc-parser.tab.o:(.bss+0x20): multiple definition of `yylloc'; scripts/dtc/dtc-lexer.lex.o:(.bss+0x0): first defined here
collect2: error: ld returned 1 exit status
make[1]: *** [scripts/Makefile.host:116: scripts/dtc/dtc] Error 1
make: *** [Makefile:1263: scripts_dtc] Error 2
```
查看代码会发现scripts/dtc/dtc-parser.tab.c和scripts/dtc/dtc-lexer.lex.c文件中确实都有定义yylloc，解决办法是将scripts/dtc/dtc-parser.tab.c中yyloc定义的地方注释掉，然后重新编译。编译完成后会在arch/arm64/boot/目录下生成Image文件。

### 其他
* 通过make help可以查看编译帮助信息，包括make命令的各种参数，例如clean targer和configuration target。
* make mrproper可以删除生成的中间文件和config文件。
