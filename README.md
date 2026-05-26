# Kubuntu26.04 customized for IP++

[English](README_en.md) | [简体中文](README.md)

为方便大家使用IP++以及做相关的开发调试，这里专门基于Kubuntu26.04定制了一版操作系统。您可以直接下载镜像使用，或者仿照这里列出的步骤自己定制。

## 安装kubuntu26.04
在虚拟机中安装kubuntu26.04，按下面所示配置参数：
```
用户名   : ippp
密码     : ippp
内存大小 : 8192M
虚拟硬盘 : 128G
语言     : 英文
```
## 安装基础软件
```
sudo passwd root            #ippp
apt update && apt upgrade
apt install net-tools	    #ifconfig
apt install openssh-server	#sshd
apt install build-essential #gcc15.2
apt install bridge-utils uml-utilities
apt install wireshark
```
将[ippp.lua](https://github.com/changqing1975/Wireshark-extension-for-IPPP/blob/main/ippp.lua)拷贝至plugins目录(help >> about wireshark >> folders >> global lua plugins  xxx/4.6/)。

## 配置网络

将虚拟机设置>>网络>>连接方式 设置为桥接网卡

为方便使用，用手动方式将IP地址配置为固定模式，将文件[01-network-manager-all.yaml](https://github.com/changqing1975/IPPP-protocol-stack-for-linux/blob/main/samples/lesson1/01-network-manager-all.yaml)拷贝至/etc/netplan。

## 配置免密登录
客户端（windows）中执行

`ssh-keygen -t rsa`

会在目录C:\Users\Administrator\\.ssh中生成id_rsa.pub

虚拟机中执行

`ssh-keygen -t rsa`

会在/home/ippp/.ssh中生成密钥对文件。在该目录中创建文件authorized_keys，将客户端id_rsa.pub文件中内容拷贝至文件authorized_keys。

## 创建目录

目录/home/ippp下创建目录ippp，ippp下创建ippp_stack_linux、kernel。

## 设置环境变量

在文件/etc/profile中最后一行添加

`export PATH="$PATH:/home/ippp/ippp/ippp_stack_linux/tool/ippp"`

若要让其永久生效，在文件/root/.bashrc中最后一行添加

`source /etc/profile`

## Linux内核源码浏览

#### 1. 下载源码并解压

`https://github.com/torvalds/linux/tags`处下载Linux6.18.23，存放到指定目录并解压

`tar -Jxf linux-6.18.23.tar.xz`

#### 2. 安装global

先在虚拟机中安装global

`apt install global`

再在vscode中登录状态下安装GNU Global插件。

验证是否生效：在vscode中使用快捷键Ctrl + Shift + P，执行Show GNU Global Version，在vscode右下角显示global版本号，表示global配置生效。

#### 3. 指定相关路径

在vscode的settings.json（file>preferences>settings）里添加

```
"gnuGlobal.globalExecutable": "/usr/bin/global",
"gnuGlobal.gtagsExecutable": "/usr/bin/gtags",
"gnuGlobal.objDirPrefix": "/mnt/global"
```

注意："gnuGlobal.objDirPrefix"用于指明生成的符号表存放在哪个文件夹，其路径需手动创建好并设置好读写属性，否则会导致后续 Rebuild 的失败。

#### 4. 生成符号表

在vscode使用快捷键Ctrl + Shift + P，执行 Rebuild Gtags Database，等待数分钟，在 vscode 右下角显示 Build tag files successfully，说明符号表解析完成了。符号表生成成功会在"gnuGlobal.objDirPrefix"的路径里生成三个文件：GRTAGS、GTAGS、GPATH。

## 至此生成了一个可浏览内核源码的kubuntu系统，可直接下载[镜像](https://share.weiyun.com/SUACzzRC)使用。

## 安装内核

安装定制版内核的方法，可参阅项目[Linux-kernel-for-IPPP-protocol-stack](https://github.com/changqing1975/Linux-kernel-for-IPPP-protocol-stack)。

## 安装GDB
在/home/ippp/ippp下创建目录GDB
### 1. 安装依赖

#### 安装python：

`wget https://www.python.org/ftp/python/3.14.4/Python-3.14.4.tar.xz`

`tar -xvf Python-3.14.4.tar.xz`

`cd Python-3.14.4`

`./configure --with-python`

`make && make install`

`apt install python3-dev`

#### 安装GMP：

`wget ftp://gcc.gnu.org/pub/gcc/infrastructure/gmp-6.3.0.tar.bz2`

`tar -jxvf gmp-6.3.0.tar.bz2`

`cd gmp-6.3.0`

`./configure`

`make && make install`

`apt install libgmp-dev`

#### 安装MPFR：

`wget ftp://gcc.gnu.org/pub/gcc/infrastructure/mpfr-4.2.2.tar.bz2`

`tar -jxvf mpfr-4.2.2.tar.bz2`

`cd mpfr-4.2.2`

`./configure`

`make && make install`

#### 安装libdebuginfod

`apt install libdebuginfod-dev`

### 2. 下载源码并解压

`wget http://ftp.gnu.org/gnu/gdb/gdb-17.1.tar.gz`

`tar -zxvf gdb-17.1.tar.gz`

### 3. 修改源码

文件gdb/remote.c

```
/* Further sanity checks, with knowledge of the architecture.  */
// if (buf_len > 2 * rsa->sizeof_g_packet)
//   error (_("Remote 'g' packet reply is too long (expected %ld bytes, got %d "
//      "bytes): %s"),
//    rsa->sizeof_g_packet, buf_len / 2,
//    rs->buf.data ());
if (buf_len > 2 * rsa->sizeof_g_packet) {
    rsa->sizeof_g_packet = buf_len;
    for (i = 0; i < gdbarch_num_regs(gdbarch); i++) {
        if (rsa->regs[i].pnum == -1)
            continue;
        if (rsa->regs[i].offset >= rsa->sizeof_g_packet)
            rsa->regs[i].in_g_packet = 0;
        else
            rsa->regs[i].in_g_packet = 1;
}}
```
### 4. 编译

`./configure --with-python --with-debuginfod`

`make && make install`

`cp gdb/gdb /usr/bin/`

在内核根目录下执行

`make scripts_gdb`

### 5. 配置gdbinit启动脚本

在/root/.config/gdb/gdbinit文件中设置

`add-auto-load-safe-path /home/ippp/ippp/kernel/linux-6.18.23/.gdbinit`

在该.gdbinit文件中添加

```
add-auto-load-safe-path ./
source ./vmlinux-gdb.py
target remote :1234
b do_init_module
```

## Buildroot生成根文件系统

### 安装依赖

`apt install -y libcrypt-dev`

### 下载并解压

在/home/ippp/ippp下创建目录buildroot，去buildroot.org下载安装包。

`tar -zxvf buildroot-xxx`

`cd buildroot-xxx`

### 配置

`make qemu_x86_64_defconfig`

`make menuconfig`

```
System configuration  --->
    (ippp) Root password

Kernel  --->
    [ ] Linux Kernel

Target packages  --->
    Networking application  --->
[*] bridge-utils
[*] iperf3
[*] iproute2
[*] ipset
[*] iptables
[*] iputils
[*] openssh
[*] openswan
[*] openvpn
[*] tcpdump
    Shell and utilities  --->
[*] screen
    System tools  --->
[*] kmod

Filesystem images  --->
    [*] ext2/3/4 root filesystem
         ext2/3/4 variant  --->
            (X) ext4
    (128M) exact size
```

### 编译

`make`

编译完成后的目标文件存放在 output/images/目录下。

## 安装qemu

使用以下命令安装：

`apt install qemu-system-x86 libc6-dev-i386 -y`

使用类似下面的命令运行：
```
qemu-system-x86_64 \
  -kernel ./linux-6.18.23/arch/x86_64/boot/bzImage \
  -hda ../buildroot/buildroot-2025.02.13/output/images/rootfs.ext4 \
  -virtfs local,path=../shared,mount_tag=host0,security_model=passthrough,id=host0 \
  -append "root=/dev/sda nokaslr console=ttyS0 noapic" \
  -serial stdio \
  -s \
  -S \
  -net nic -net tap,ifname=tap0,script=no,downscript=no
```

### 配置共享文件
1. 准备目录

`mkdir ../shared`

2. 启动QEMU时的参数

`qemu-system-x86_64 ... -virtfs local,path=../shared,mount_tag=host0,security_model=passthrough,id=host0`

3. 在虚拟机内部，挂载共享文件夹：

`mkdir -p /ippp/shared`

`sudo mount -t 9p -o trans=virtio,version=9p2000.L host0 /ippp/shared`

要想每次重启QEMU虚拟机时能自动重新挂载，则将

`mount -t 9p -o trans=virtio,version=9p2000.L host0 /ippp/shared`

加入到etc/init.d/rcS的最后一行。

## 至此生成了一个支持IP++开发与调试的kubuntu系统，可直接下载[镜像](https://share.weiyun.com/XzZ7NRTK)使用。