---
title: "基于CentOS73.10内核版本编译安装6.5内核并开启vmlinux"
date: 2024-04-05T11:03:49+08:00
draft: false
---

## 目的
编译安装6.x版本的内核，用于支持ebpf程序，开启/sys/kernel/btf/vmlinux

## 编译环境
- CentOS Linux release 7.9.2009 (Core) 
- 3.10.0-1160.el7.x86_64

---

#### 编译环境准备
```bash
yum groupinstall "Development Tools" -y && \
yum install -y \
    openssl-devel \
    rpm-build \
    redhat-rpm-config \
    asciidoc \
    hmaccalc \
    perl-ExtUtils-Embed \
    pesign \
    xmlto \
    audit-libs-devel \
    binutils-devel \
    elfutils-devel \
    elfutils-libelf-devel \
    ncurses-devel \
    newt-devel \
    numactl-devel \
    pciutils-devel \
    python-devel \
    zlib-devel \
    cmake \
    centos-release-scl devtoolset-8-gcc* \
    rpm-build

yum -y install python3	# 没有python3的需要安装一下,有的话忽略
```

#### 仅在当前终端启用新版本GCC用于编译，不影响系统默认GCC版本
```bash
scl enable devtoolset-8 bash
```

#### 编译安装新版本 pahole，用于支持btf编译
```bash
git clone https://github.com/acmel/dwarves.git
cd dwarves
git submodule update --init --recursive
mkdir build
cd build
cmake -D__LIB=lib ..
make install
```
#### 将编译好的so文件直接拷贝或移动到 /lib64/ 目录
```bash
mv /usr/local/lib/libdwarves.so.1.0.0 /usr/local/lib/libdwarves_emit.so.1.0.0 /usr/local/lib/libdwarves.so.1.0.0 /usr/local/lib/libdwarves_reorganize.so.1.0.0 /usr/local/lib/libdwarves.so.1 /usr/local/lib/libdwarves_emit.so.1 /usr/local/lib/libdwarves_reorganize.so.1 /lib64/
```

#### 测试一下pahole
```bash
pahole --version	# 输出 v1.26 或者更高的版本信息表示成功
```

#### 下载并解压内核源码
```bash
wget https://mirrors.tuna.tsinghua.edu.cn/kernel/v6.x/linux-6.5.2.tar.gz
tar -zxvf linux-6.5.2.tar.gz
```

#### 配置BTF
```bash
cd linux-6.5.2
make menuconfig
```
> 1. 选择 **Kernel hacking** -> **Compile-time checks and compiler options** ->  **Debug information** 
> 2. 选中 **Generate DWARF Version 4 debuginfo**
> 3. 选中 **Generate BTF typeinfo (NEW)** 
> 4. 选中 **Save** 保存配置文件, 然后一直**Exit**退出**meunconfig**
> 5. 查看 **.config** 文件， 检查一下 **CONFIG_DEBUG_INFO_BTF** 配置是否存在，如果该项存在且 **=y** 那么就配置成功

#### 编译安装内核源码
`-j 12`  参数是并行编译的数量，可以根据自己的机器CPU核心情况修改
```bash
make -j 12 all
make INSTALL_MOD_STRIP=1 modules_install > /dev/null && make modules_install && make install
```

#### 设置默认新版本内核启动
```bash
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg && grub2-set-default 0 && grub2-mkconfig -o /boot/grub2/grub.cfg
```

---

## 安装到其它主机
#### 有些场景我们可能需要将编译好内核安装到其它主机，以下是操作步骤

```bash
scp {源码路径}/.config {目标主机}/boot/config-6.5.2
scp /boot/initramfs-6.5.2.img {目标主机}/boot/
scp /boot/System.map-6.5.2 {目标主机}/boot/
scp /boot/vmlinuz-6.5.2 {目标主机}/boot/
scp -r /lib/modules/6.5.2 {目标主机}/lib/modules/
grub2-mkconfig -o /boot/grub2/grub.cfg
```

