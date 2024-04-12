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
```Bash
yum groupinstall "Development Tools" -y && \
yum install -y \
    openssl-devel \
    rpm-build \
    dwarves \
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
    scl-utils
yum -y install centos-release-scl devtoolset-8-gcc*
yum -y install python3	# 没有python3的需要安装一下,有的话忽略
```

#### 仅在当前终端启用新版本GCC用于编译，不影响系统默认GCC版本
```Bash
scl enable devtoolset-8 bash
```

#### 编译安装git
编译内核会强制校验GIT仓库，此时会需要用到 git -C 指令，而yum源安装的git版本很低，不支持该参数，所以需要编译安装新版
```Bash
wget https://mirrors.edge.kernel.org/pub/software/scm/git/git-2.9.5.tar.xz
tar xvf git-2.9.5.tar.xz
cd git-2.9.5
./configure
make && make install
mv /usr/bin/git /usr/bin/git.bak
cp ./git /usr/bin/
```

#### 编译安装新版本 pahole，用于支持btf编译
```Bash
git clone https://github.com/acmel/dwarves.git
cd dwarves
git submodule update --init --recursive
mkdir build
cd build
cmake -D__LIB=lib ..
make install
```
#### 将编译好的so文件直接拷贝或移动到 /lib64/ 目录
```Bash
mv /usr/local/lib/libdwarves_emit.so.1.0.0 /usr/local/lib/libdwarves.so.1.0.0 /usr/local/lib/libdwarves_reorganize.so.1.0.0 /usr/local/lib/libdwarves.so.1 /usr/local/lib/libdwarves_emit.so.1 /usr/local/lib/libdwarves_reorganize.so.1 /lib64/
```

#### 测试一下pahole
```Bash
pahole --version	# 输出 v1.26 或者更高的版本信息表示成功
```

#### 下载并解压内核源码
```Bash
wget https://mirrors.tuna.tsinghua.edu.cn/kernel/v6.x/linux-6.5.2.tar.gz
tar -zxvf linux-6.5.2.tar.gz
```

#### 配置BTF
```Bash
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
```Bash
make -j 12 all
make INSTALL_MOD_STRIP=1 modules_install > /dev/null && make modules_install && make install
```

#### 设置默认新版本内核启动
```Bash
awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg && grub2-set-default 0 && grub2-mkconfig -o /boot/grub2/grub.cfg
```

---
## 其它安装方式
有些场景我们可能需要将编译好内核安装到其它主机，以下是两种方式

#### 1. 拷贝安装到其它主机

```Bash
scp {源码路径}/.config {目标主机}/boot/config-6.5.2
scp /boot/initramfs-6.5.2.img {目标主机}/boot/
scp /boot/System.map-6.5.2 {目标主机}/boot/
scp /boot/vmlinuz-6.5.2 {目标主机}/boot/
scp -r /lib/modules/6.5.2 {目标主机}/lib/modules/
grub2-mkconfig -o /boot/grub2/grub.cfg
```

#### 2. 打rpm包
要将高版本的内核源码在CentOS7上打成rpm包会有不少兼容性或者依赖版本的问题需要解决，需要做一些简单的配置
- 修改源码内的 `./scripts/package/mkspec` 文件,找到 `BuildRequires: (elfutils-libelf-devel or libelf-devel) flex` 修改为 `BuildRequires: elfutils-libelf-devel flex`
- 编译rpm包最低版本要求`4.13`，需要升级rpm，执行这一步的时候可能会有一些依赖问题，比如python3或者rpm-sign本身的依赖问题等等，直接yum将冲突的依赖卸载后重试，python3要记得升级完要装回来
  ```Bash
    yum install -y  popt-devel libarchive
    rpm -Uvh https://cbs.centos.org/kojifiles/packages/rpm/4.13.0.2/1.el7.c8/x86_64/rpm-4.13.0.2-1.el7.c8.x86_64.rpm \
    https://cbs.centos.org/kojifiles/packages/rpm/4.13.0.2/1.el7.c8/x86_64/rpm-plugin-selinux-4.13.0.2-1.el7.c8.x86_64.rpm \
    https://cbs.centos.org/kojifiles/packages/rpm/4.13.0.2/1.el7.c8/x86_64/rpm-plugin-ima-4.13.0.2-1.el7.c8.x86_64.rpm \
    https://cbs.centos.org/kojifiles/packages/rpm/4.13.0.2/1.el7.c8/x86_64/rpm-plugin-syslog-4.13.0.2-1.el7.c8.x86_64.rpm \
    https://cbs.centos.org/kojifiles/packages/rpm/4.13.0.2/1.el7.c8/x86_64/rpm-plugin-systemd-inhibit-4.13.0.2-1.el7.c8.x86_64.rpm \
    https://cbs.centos.org/kojifiles/packages/rpm/4.13.0.2/1.el7.c8/x86_64/rpm-libs-4.13.0.2-1.el7.c8.x86_64.rpm \
    https://cbs.centos.org/kojifiles/packages/rpm/4.13.0.2/1.el7.c8/x86_64/python2-rpm-4.13.0.2-1.el7.c8.x86_64.rpm \
    https://cbs.centos.org/kojifiles/packages/rpm/4.13.0.2/1.el7.c8/x86_64/rpm-devel-4.13.0.2-1.el7.c8.x86_64.rpm \
    https://cbs.centos.org/kojifiles/packages/rpm/4.13.0.2/1.el7.c8/x86_64/rpm-build-libs-4.13.0.2-1.el7.c8.x86_64.rpm \
    https://cbs.centos.org/kojifiles/packages/rpm/4.13.0.2/1.el7.c8/x86_64/rpm-build-4.13.0.2-1.el7.c8.x86_64.rpm
  ```
- 执行命令开始编译，编译前要确保内存充足
  ```Bash
  make rpm-pkg -j 12 # 如果需要压缩rpm大小可以增加 INSTALL_MOD_STRIP=1 参数, 如：make INSTALL_MOD_STRIP=1 rpm-pkg -j 12
  ```
- 编译完成后可以在 `~/rpmbuild/RPMS/x86_64/` 目录找到三个rpm包，可以通过下面的指令进行安装
  ```Bash 
   yum localinstall kernel* -y
  ```