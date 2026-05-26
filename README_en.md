# Kubuntu26.04 customized for IP++

[English](README_en.md) | [简体中文](README.md)

To facilitate the usage, development and debugging of IP++, a customized operating system based on Kubuntu 26.04 is provided. You can directly download the system image or follow the detailed steps below to customize the system manually.

## Install Kubuntu 26.04
Install Kubuntu 26.04 in a virtual machine with the following configuration parameters:
```
Username   : ippp
Password   : ippp
Memory     : 8192M
Virtual Disk : 128G
Language   : English
```
## Install Basic Software
```
sudo passwd root            #ippp
apt update && apt upgrade
apt install net-tools	    #ifconfig
apt install openssh-server	#sshd
apt install build-essential #gcc15.2
apt install bridge-utils uml-utilities
apt install wireshark
```
Copy [ippp.lua](https://github.com/changqing1975/Wireshark-extension-for-IPPP/blob/main/ippp.lua) to the global Lua plugins directory(Help >> About Wireshark >> Folders >> Global Lua Plugins xxx/4.6/).

## Network Configuration

Set the virtual machine's network connection mode to Bridged Networking.

For convenience, configure a static IP address manually by copying the file [01-network-manager-all.yaml](https://github.com/changqing1975/IPPP-protocol-stack-for-linux/blob/main/samples/lesson1/01-network-manager-all.yaml) to /etc/netplan.

## Configure Passwordless SSH Login

On the client (Windows), execute:

`ssh-keygen -t rsa`

Public key id_rsa.pub will be generated under C:\Users\Administrator\\.ssh.

On the virtual machine, execute:

`ssh-keygen -t rsa`

This generates the key pair in /home/ippp/.ssh. Create a file named authorized_keys in that directory and paste the contents of id_rsa.pub from the client into it.

## Create Directories

Create directories ippp under /home/ippp, Create directories ippp_stack_linux, and kernel under ippp.

## Set Environment Variables

Add the following line to the end of /etc/profile:

`export PATH="$PATH:/home/ippp/ippp/ippp_stack_linux/tool/ippp"`

To make it permanent, add the following line to the end of /root/.bashrc:

`source /etc/profile`

## Linux Kernel Source Code Browsing

#### 1. Download and Extract Source Code

Download Linux 6.18.23 from `https://github.com/torvalds/linux/tags`, store it in the specified directory, and extract:

`tar -Jxf linux-6.18.23.tar.xz`

#### 2. Install global

First, install global in the virtual machine:

`apt install global`

Then, install the GNU Global plugin in VScode while logged in.

Verify if it works: In VScode, press Ctrl + Shift + P, execute "Show GNU Global Version", and display the global version number in the bottom-right corner of VScode, indicating the global configuration is active.

#### 3. Specify Related Paths

Add the following to VScode's settings.json (File >> Preferences >> Settings):

```
"gnuGlobal.globalExecutable": "/usr/bin/global",
"gnuGlobal.gtagsExecutable": "/usr/bin/gtags",
"gnuGlobal.objDirPrefix": "/mnt/global"
```

Note: "gnuGlobal.objDirPrefix" specifies the folder where the generated symbol table is stored. The path must be created manually with read/write permissions; otherwise, subsequent rebuilds will fail.

#### 4. Generate Symbol Table

In VS Code, press Ctrl + Shift + P, execute "Rebuild Gtags Database", and wait several minutes. When "Build tag files successfully" appears in the bottom-right corner of VS Code, the symbol table parsing is complete. The three files GRTAGS, GTAGS, and GPATH will be generated in the path specified by gnuGlobal.objDirPrefix.

## A fully functional kernel source code browsing system is ready. You can download the pre-built [image](https://share.weiyun.com/SUACzzRC).

## Install the Kernel

For methods to install the customized kernel, refer to the project [Linux-kernel-for-IPPP-protocol-stack](https://github.com/changqing1975/Linux-kernel-for-IPPP-protocol-stack).

## Install GDB

Create a GDB directory under /home/ippp/ippp.

### 1. Install Dependencies

#### Install python：

`wget https://www.python.org/ftp/python/3.14.4/Python-3.14.4.tar.xz`

`tar -xvf Python-3.14.4.tar.xz`

`cd Python-3.14.4`

`./configure --with-python`

`make && make install`

`apt install python3-dev`

#### Install GMP：

`wget ftp://gcc.gnu.org/pub/gcc/infrastructure/gmp-6.3.0.tar.bz2`

`tar -jxvf gmp-6.3.0.tar.bz2`

`cd gmp-6.3.0`

`./configure`

`make && make install`

`apt install libgmp-dev`

#### Install MPFR：

`wget ftp://gcc.gnu.org/pub/gcc/infrastructure/mpfr-4.2.2.tar.bz2`

`tar -jxvf mpfr-4.2.2.tar.bz2`

`cd mpfr-4.2.2`

`./configure`

`make && make install`

#### Install libdebuginfod

`apt install libdebuginfod-dev`

### 2. Download Source Code and Extract

`wget http://ftp.gnu.org/gnu/gdb/gdb-17.1.tar.gz`

`tar -zxvf gdb-17.1.tar.gz`

### 3. Modify Source Code

File gdb/remote.c

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
### 4. Compile

`./configure --with-python --with-debuginfod`

`make && make install`

`cp gdb/gdb /usr/bin/`

Execute in the kernel root directory:

`make scripts_gdb`

### 5. Configure gdbinit Startup Script

In file /root/.config/gdb/gdbinit, set:

`add-auto-load-safe-path /home/ippp/ippp/kernel/linux-6.18.23/.gdbinit`

Add the following to file .gdbinit:

```
add-auto-load-safe-path ./
source ./vmlinux-gdb.py
target remote :1234
b do_init_module
```

## Buildroot for Root Filesystem Generation

### Install Dependencies

`apt install -y libcrypt-dev`

### Download and Extract

Create a buildroot directory under /home/ippp/ippp, and download the package from buildroot.org.

`tar -zxvf buildroot-xxx`

`cd buildroot-xxx`

### Configure

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

### Compile

`make`

The target files after compilation are stored in the output/images/ directory.

## Install qemu

Install using the following command:

`apt install qemu-system-x86 libc6-dev-i386 -y`

Run using a command similar to:

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

### Configure Shared Files

1. Prepare Directory

`mkdir ../shared`

2. QEMU Startup Parameters

`qemu-system-x86_64 ... -virtfs local,path=../shared,mount_tag=host0,security_model=passthrough,id=host0`

3. In the Virtual Machine, Mount Shared Folder:

`mkdir -p /ippp/shared`

`sudo mount -t 9p -o trans=virtio,version=9p2000.L host0 /ippp/shared`

To automatically remount each time the QEMU virtual machine restarts, add:

`mount -t 9p -o trans=virtio,version=9p2000.L host0 /ippp/shared`

to the end of etc/init.d/rcS.

## You can now directly download the [image](https://share.weiyun.com/XzZ7NRTK) of the Kubuntu system supporting IP++ development and debugging.