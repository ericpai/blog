---
author: "Eric Pai"
date: 2017-10-18
linktitle: DIY CentOS7 安装镜像
title: DIY CentOS7 安装镜像
tags: ["CentOS7", "Linux"]
draft: false
---

最近接手了一些关于私有化部署方面的工作。私有化部署中最基础的内容就是操作系统的自动安装了。在之前接触过的这类工作中，无论是 Windows 还是 Linux，都基本上是通过 GUI 交互式完成。然而，在大量的服务器安装的环境下，人工干预每台服务器的系统安装就变得不太现实了，因此需要能够实现自动安装操作系统。CentOS 的安装是支持自动方式的，只是需要对官方镜像做一定的修改。这篇文章总结了如何 DIY 一个 CentOS7 的自动安装镜像。

<!--more-->

## 目标

首先，要明确 DIY 的镜像的目标：

- 镜像版本：*CentOS-7-Minimal-1611* 。一般情况下，服务器基本都是安装 Minimal 版本。镜像可以在任意的 Linux 镜像源下载到，因为我这边下载较慢，是在[清华的镜像源](https://mirrors.tuna.tsinghua.edu.cn/)中下载它的种子，然后用 P2P 的方式加速。
- 安装过程：完全自动化。完全自动化包括了前期的系统分区、格式化、网络初始化、基本软件包的安装。这些过程应该能够自动正确完成，并且每台服务器都要有同构的配置。
- 安装的软件包：我们的私有化部署要求操作系统要提前安装好 [ansible](https://www.ansible.com/)，[dig](https://www.isc.org/downloads/bind/)，[tcpdump](http://www.tcpdump.org/) 等一系列常用工具。同时，为了之后方便部署应用，需要安装 [docker](https://www.docker.com/)。
- 一键设置 IP 地址：机房中服务器的网络基本都是固定 IP 地址。因此安装完操作系统后，需要有方便的脚本工具设置服务器的物理 IP，使其能够加入内网。
- 可插拔 yum 源：有的部署环境比较特殊，服务器无法访问外网，且内网中也没有 yum 源服务，这使得之后其他的软件包安装会变得非常麻烦。因此还需要让服务器通过插入刻录好 *CentOS-7-Everything-1611* 镜像的 U 盘作为临时 yum 源解决软件包安装的问题。

在这里，先列出默认安装时会遇到的问题，后面可以看到是如何解决这些问题的：

- 网卡名不固定：使用默认安装方式，不同的服务器的网卡名会不一致，类似 `enp0s3` 这种。网卡名不固定会使得后面的一些运维脚本无法正常工作。因此需要在安装系统时固定网卡名为默认的 `eth0`。
- SSH 登录时 DNS 反解问题：从 CentOS7 开始，`/etc/sshd/sshd_config` 中 `UseDNS` 的默认值为 `yes`，即有客户端发起登录请求时，服务器都需要去 nameserver 反解该 IP 地址。由于新安装系统的服务器一般是没有配置 nameserver 的，因此该过程会使得登录变得非常慢，需要禁用该配置。
- 公钥认证登录：在执行 ansible 任务时，需要将主节点的公钥（`/root/.ssh/id_rsa.pub`）加到每台目标节点上的 `/root/.ssh/authorized_keys` 中。如果在操作系统安装时不完成这项工作，那么后面需要手工 scp，费时费力。

## 步骤

### 1. 准备环境

DIY CentOS7 镜像的工作本身也需要在同版本的 CentOS 系统中进行，这是因为之后会有一大部分工作是在处理安装软件包的依赖问题，而解决该问题的最快方式就是下载其依赖的其他软件包。当然，前提是能够十分顺利安装或更新 yum 源。如果一直在 `determin fastest mirrors` 就不算顺利，这个时候有必要看一下 `/etc/yum.repos.d/` 里面源的配置。如果没有特别一致的环境，建议可以起一个 CentOS7.3-1611 的 docker 容器，在容器里面操作也是 OK 的，只是需要将工作目录 volume 到宿主机上，方便后面同步文件。

*CentOS-7-Minimal-1611* 的镜像大小为 680M+，因此最好确保自己的工作目录有不低于 1G 的硬盘空间。CPU 和内存则没有太大要求，正常即可。

### 2. 整理工作目录

假设我们的工作目录在 `~/my_linux`。

在 `~/my_linux` 下创建一些必要目录，命令如下：

```bash
cd ~/my_linux
mkdir isolinux
mkdir isolinux/images
mkdir isolinux/LiveOS
mkdir isolinux/Packages
```

`isolinux` 实际上就是之后打包成镜像时，镜像内部的根目录。对于里面创建的三个子目录，简单说明如下：

- `images`：主要包含了压缩了模块化的内核的可执行文件 [`vmlinuz`](https://zh.wikipedia.org/wiki/Vmlinux) 和临时文件系统 [`initrd`](https://www.systutorials.com/docs/linux/man/4-initrd/#index)。由于在安装时没有根文件系统，因此需要引导程序在内存中临时启动一个 Linux 系统并动态加载内核来执行后面的安装过程。
- `LiveOS`：包含了 `squashfs.img`，是一个完整的根文件系统的 Linux 镜像。在引导菜单中可以选择 *体验 CentOS* ，这时进入的系统就是来自于 `squashfs.img`。
- `Packages`：一个包含**完整**的 rpm 软件包的目录。**完整**指的是里面的 rpm 不存在任何依赖问题。如果有依赖问题可能会造成安装过程中断。

### 3. 拷贝必要文件

下载好 *CentOS-7-Minimal-1611* 镜像后，需要将其挂载到本机，然后拷贝一些里面的文件到工作目录。

假设下载的文件在 `~/my_linux/CentOS-7-x86_64-Minimal-1611.iso`。

挂载 iso 到 `/mnt/centos`：

```bash
mkdir /mnt/centos
mount ~/my_linux/CentOS-7-x86_64-Minimal-1611.iso /mnt/centos
```

此时该 iso 以只读形式挂载到了 `/mnt/centos`。开始拷贝文件：

```bash
cp -r /mnt/centos/images/* ~/my_linux/isolinux/images/
cp /mnt/centos/LiveOS/* ~/my_linux/isolinux/LiveOS/
cp /mnt/centos/Packages/* ~/my_linux/isolinux/Packages/
cp /mnt/centos/isolinux/* ~/my_linux/isolinux/
cp /mnt/centos/.discinfo ~/my_linux/isolinux/
```

最后还需要拷贝包含仓库信息的 `comp.xml.gz` 文件，后面 `createrepo` 时会使用到（不同版本的文件名不相同，请注意）：

```bash
cp /mnt/centos/repodata/d4de4d1e2d2597c177bb095da8f1ad794d69f76e8ac7ab1ba6340fdd0969e936-c7-minimal-x86_64-comps.xml.gz ~/my_linux/
cd ~/my_linux
gunzip d4de4d1e2d2597c177bb095da8f1ad794d69f76e8ac7ab1ba6340fdd0969e936-c7-minimal-x86_64-comps.xml.gz
mv d4de4d1e2d2597c177bb095da8f1ad794d69f76e8ac7ab1ba6340fdd0969e936-c7-minimal-x86_64-comps.xml comps.xml
```

### 4. 创建自定义的安装仓库

以 docker 为例，展示如何加入并解决 rpm 依赖问题。

1. `cd ~/my_linux/isolinux/Packages`。
1. 安装 yum 工具包 `yum install -y yum-utils`。
1. 建立临时仓库配置 `rpm --initdb --dbpath /tmp/testdb`，后面检查依赖完整性时会用到。
1. 确认一下当前的 yum 源中是否有符合要求的 docker 版本。

     ```bash
[root@ericpai ~]# yum info docker
已加载插件：fastestmirror, langpacks
Repository epel is listed more than once in the configuration
Loading mirror speeds from cached hostfile
可安装的软件包
名称    ：docker
架构    ：x86_64
时期       ：2
版本    ：1.12.6
发布    ：55.gitc4618fb.el7.centos
大小    ：15 M
源    ：extras/7/x86_64
简介    ： Automates deployment of containerized applications
网址    ：https://github.com/docker/docker
协议    ： ASL 2.0
描述    ： Docker is an open-source engine that automates the deployment of any
         : application as a lightweight, portable, self-sufficient container that will
         : run virtually anywhere.
         :
         : Docker containers can encapsulate any payload, and will run consistently on
         : and between virtually any server. The same container that a developer builds
         : and tests on a laptop will run at scale, in production*, on VMs, bare-metal
         : servers, OpenStack clusters, public instances, or combinations of the above.
     ``` 
 说明当前可安装的 docker 版本为 1.12.6，满足需要。

1. 下载 docker rpm 包及其依赖包 `yumdownloader --resolve docker`。
1. 检查依赖完整性 `rpm --test --dbpath /tmp/testdb -Uvh *.rpm`。
  当执行后的输出中没有错误时，说明各个 rpm 的依赖是完整的，否则需要根据错误信息，反复执行步骤 4-6 下载被依赖的 rpm 并测试。
  
  ```bash
[root@ericpai Packages]# rpm --test --dbpath /tmp/testdb -Uvh *.rpm
警告：acl-2.2.51-12.el7.x86_64.rpm: 头V3 RSA/SHA256 Signature, 密钥 ID f4a80eb5: NOKEY
警告：jq-1.5-1.el7.x86_64.rpm: 头V3 RSA/SHA256 Signature, 密钥 ID 352c64e5: NOKEY
准备中...                          ################################# [100%]
  ```

依赖解决完成后，可以在这些 package 的基础上初始化仓库了。在初始化前，首先确认环境中是否安装了 `createrepo`，如果没有则还需要先执行 `yum install -y createrepo`。

初始化仓库的过程很简单，如下：

```bash
cd ~/my_linux/isolinux
createrepo -g ~/my_linux/comps.xml .
```

这样，安装系统时需要的仓库已经准备就绪了，下一步是写系统安装的配置文件。

### 5. 配置 kickstart

*kickstart* 文件定义了在安装系统时的一系列配置，包括安装形式、安装源、网卡配置、根分区格式化配置、软件包安装以及安装后续操作等。详细语法可以见[官方文档](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/installation_guide/sect-kickstart-syntax)。

在安装好的系统中 `/root/` 下可以找到 `anaconda-ks.cfg`，这个其实就是用户在交互式安装过程中的各项选择形成的 *kickstart* 文件。一般情况下，只需对其进行少量修改，就可以满足需求。

先拷贝一份到工作目录：`cp /root/anaconda-ks.cfg ~/my_linux/isolinux/ks.cfg`。然后，用编辑器打开该文件，对部分配置进行修改。

这里先给出 *kickstart* 文件中需要注意的配置，后面会对每一行的含义做出解释（注释内容已经去掉）。

```bash
cdrom
cmdline
ignoredisk --only-use=sda
network  --bootproto=static --device=eth0 --onboot=true --noipv6 --no-activate
rootpw --iscrypted $6$WPyFZmvvw8OGV82j$TQulytgHsfZfbUyb2fxVBhQrLPJM9x8yVtgvD9VdmxpyB6lP4AMTQO09N7BJh9OFuw73fOjMPiA/WSH7lAnK2.
autopart --type=lvm
clearpart --drives=sda --all --initlabel

%packages
@^minimal
@core
-biosdevname
chrony
kexec-tools
docker

%end

%post --nochroot

#!/bin/sh

set -x -v
exec 1>/mnt/sysimage/root/kickstart.log 2>&1

cp -r /run/install/repo/tools /mnt/sysimage/root/setup_tools
mkdir -p /mnt/sysimage/root/.ssh
cp -f /run/install/repo/keys/* /mnt/sysimage/root/.ssh/

sed -i 's/#UseDNS yes/UseDNS no/g' /mnt/sysimage/etc/ssh/sshd_config

rm -rf /mnt/sysimage/etc/yum.repos.d/*
cp /run/install/repo/CentOS-Local.repo /mnt/sysimage/etc/yum.repos.d/


# Mount your Everything .iso at /mnt/localyum
mkdir -p /mnt/sysimage/mnt/localyum

%end
```

- `cdrom`：指定了从光盘安装系统。
- `cmdline`：自动安装模式。如果之前是通过 GUI 交互完成的安装，此处的值应该是 `graphical`，需要改为 `cmdline`。
- `ignoredisk`：忽略其他的磁盘设备。因为在给服务器安装系统时事先肯定是不知道服务器的磁盘配置的，因此只需要管系统盘就可以了，所以这里加了参数 `--only-use=sda`，即将系统只安装在 `/dev/sda` 上。
- `network`：指定了系统网卡的配置。此时注意，需要指定网卡名 `--device=eth0` 才能使对应的网卡配置生效，这个时候就体现出固定网卡名的作用了。
- `rootpw`：设置了 root 用户及其密码。配置的密码值应该是加密后的。这里的密码是 `root`。
- `autopart`：自动分区，一般都使用 lvm 方式。
- `clearpart`：如果 `/dev/sda` 上原来已经有系统了，使用该配置可以直接格掉原来的系统。这里注意的是，加上 `--all` 参数后，如果这个服务器是多系统，则它也会将其他系统删掉。
- `%packages...%end`：指定要安装的软件包。默认的 *kickstart* 都会安装 `minimal` 和 `core` 这两个组的 rpm。组的定义可以在 `~/my_linux/comps.xml`  里看到，其实就是一堆 rpm 包的集合。这里我选择不安装 `biosdevname`（不安装以 `-` 开头），防止网卡名被改。同时，在后面加了一行 `docker` 表示要在安装系统时安装 `docker`。
- `%post...%end`：指定安装成功后执行的命令。从内容上可以看出是执行了一个 shell 脚本，后面会有对其的具体解释。

从上面的介绍中可以看出，对 *kickstart* 文件要做的就是：

- 确认安装来源是 `cdrom`。
- 修改安装模式为 `cmdline`。
- 修改网卡配置。
- 设置 root 密码。
- 调整安装的软件包。
- 配置安装后执行的命令。

最后在这里解释一下 `%post...%end` 中的内容：

1. `%post` 后面指定了 `--nochroot` 参数。指定该参数通俗来讲是使用了"镜像视角"。即镜像中的 `a` 文件在 shell 中的路径为 `/run/install/repo/a`。而新安装的操作系统以文件系统的形式挂载在 `/mnt/sysimage`，即操作系统中的 `/` 为 `/mnt/sysimage/`。如果不指定则为"系统视角"，所有的命令中的路径都是新系统中的，但是此时就无法访问镜像中的文件了。因此，如果执行的命令比较复杂，可以拆分成多个 `%post...%end` 块，执行顺序由前至后，每一块是相互独立的，可以自行决定是否使用 `chroot`。

1. 将执行的日志保存在 `/root/kickstart.log` 中，这样即使执行过程某一步出错了，也可以事后通过日志检查。这里需要知道的是，`%post` 执行出错并不会导致系统安装失败，因为此时系统已经安装好了。

1. 将镜像的 `tools` 目录里的工具脚本拷贝到了系统的 `/root/setup_tools`。

1. 将事先生成好的密钥对以及 `authorized_keys` 拷贝到系统的 `/root/.ssh`，这样后面就可以实现服务器之间的公钥认证登录了。

1. 禁用 sshd 对客户端 IP 的反解。

1. 删掉默认的 yum 源，拷贝本地文件系统的 yum 源配置文件。

1. 提前创建好 yum 源的挂载点，这样后面如果需要用 Everything 的 yum 源，则直接插入刻好的U盘后执行 `mount /dev/sdb /mnt/sysimage/mnt/localyum` 就可以使用了（`/dev/sdb` 应该为你的U盘的设备地址）。

修改好 *kickstart* 后，再准备一下我们需要的工具脚本、密钥对以及 yum 源配置文件。

修改 IP 地址的脚本，默认子网掩码长度为 24：

```bash
[root@ericpai isolinux]# cat ~/my_linux/isolinux/tools/setip.sh
#!/bin/sh

ip=$1
gw=`echo $1 | awk -F. '{print $1"."$2"."$3".1"}'`
bc=`echo $1 | awk -F. '{print $1"."$2"."$3".255"}'`

sed -i "s/IPADDR=\"\"/IPADDR=$ip/g" /etc/sysconfig/network-scripts/ifcfg-eth0
sed -i "s/GATEWAY=\"\"/GATEWAY=$gw/g" /etc/sysconfig/network-scripts/ifcfg-eth0
sed -i "s/BROADCAST=\"\"/BROADCAST=$bc/g" /etc/sysconfig/network-scripts/ifcfg-eth0

service network restart
```

yum 源配置文件：

```bash
[root@ericpai isolinux]# cat ~/my_linux/isolinux/CentOS-Local.repo
[c7-local]
name=CentOS-$releasever - Local
baseurl=file:///mnt/localyum/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
```

keys 目录内容：

```bash
[root@ericpai isolinux]# ll ~/my_linux/isolinux/keys/
总用量 12
-rw-r--r--. 1 root root  408 10月 17 10:55 authorized_keys
-rw-------. 1 root root 1679 10月 17 10:50 id_rsa
-rw-r--r--. 1 root root  408 10月 17 10:50 id_rsa.pub
```

最后，还需要在对引导菜单配置做一点修改。

### 6. 修改 `isolinux.cfg`

`isolinux/isolinux.cfg` 文件定义了在引导时我们看到的那个选择菜单，找到里面的 `label linux` 一行以及下面的执行命令，修改 `init.ks` 参数以及增加 `net.ifnames=0`：

```bash
label linux
  menu label ^Install CentOS Linux 7
  kernel vmlinuz
  append initrd=initrd.img inst.stage2=hd:LABEL=CentOS\x207\x20x86_64 inst.ks=cdrom:/dev/cdrom:/ks.cfg net.ifnames=0 quiet
```

- `init.ks` 参数指定了安装系统使用的 *kickstart* 文件的位置，通过 `cdrom` 方式安装的都是 `/dev/cdrom`。

- `net.ifnames=0` 参数则是禁用网卡的自动命名，这也是固定网卡命名的一个必需步骤。

修改好了 `isolinux.cfg`，可以开始创建镜像了。

### 7. 创建镜像

创建镜像需要使用 `genisoimage` 工具，如果没有可以通过 yum 安装 `yum install -y genisoimage`。

创建过程如下：

```bash
cd ~/my_linux
mkisofs -o CentOS-7-x86_64-Minimal-1611-diy.iso -b isolinux.bin -c boot.cat -no-emul-boot \
-V 'CentOS 7 x86_64' -boot-load-size 4 -boot-info-table -R -J -v -T isolinux
```

镜像创建完成，可以放到虚拟机中安装测试了。

## 测试

我使用的是 [VirtualBox](https://www.virtualbox.org/) 启动虚拟机并测试的。创建一台 Linux 配置的服务器，然后将 `CentOS-7-x86_64-Minimal-1611-diy.iso` 加载到虚拟机的光驱中，调整好启动顺序并开机。在启动界面，选择 **Install CentOS Linux 7**，就可以开始自动安装了。

![启动菜单](/links/diy-centos7-install/boot_menu.png)

安装成功后，按照提示点击回车，即可结束安装过程。此时要注意，需要重新调整启动顺序为硬盘启动，否则会重新进入安装菜单。

![自动安装完成](/links/diy-centos7-install/auto_install_finished.png) 

## 总结

可以看出，DIY CentOS7 的自动安装镜像并不难，只要按照步骤一步一步即可实现。然而重要的点在于，需要确定我们的 DIY 的目标：系统需要什么版本，需要预安装什么软件包，以及需要拷贝好什么样的工具脚本等。目标明确了，DIY 的工作也就事半功倍了。

## 参考资料

1. https://zh.wikipedia.org/wiki/Vmlinux
1. https://zh.wikipedia.org/wiki/Initrd
1. https://thornelabs.net/2016/07/23/kickstart-centos-7-with-eth0-instead-of-predictable-network-interface-names.html
1. http://www.smorgasbork.com/2014/07/16/building-a-custom-centos-7-kickstart-disc-part-1/