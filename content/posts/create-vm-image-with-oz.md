---
title: "使用Oz工具自动化虚拟机镜像"
subtitle: "一次Oz工具的学习及使用实践"
draft: false
date: "2018-05-16T10:22:57+08:00"
categories: "OpenStack"
tags: ["oz","image","vm"]
---



## Oz 简介

镜像制作一般分为手动制作和自动制作。

手动制作主要流程：

1. 利用OS镜像启动一个虚拟机实例，启动镜像的工具包括virt-install、virt-manager、qemu等；


2. 进入虚拟机实例，安装定制化软件包；


3. 压缩磁盘镜像并保存。

利用自动化工具制作：

自动化工具主要有： diskimage-builder, oz, image-bootstrap 等，它们主要通过配置文件完成虚拟机镜像制作。利用配置文件的好处主要有：

1.  一次手动配置保存为配置文件，以后反复使用，避免机械式的重复工作;
2.  避免手动步骤的遗忘。

在此介绍利用oz 工具进行自动化制作的方法。

Oz是一款基于Python的镜像制作工具，通过libvirt 和KVM完成操作系统，及定制化软件包的安装，支持多个操组作系统的安装。基于每一种系统，Oz都分三个阶段以完成定制化操作系统，这三个阶段是：操作系统安装、定制化软件安装、生成操作系统的软件包清单。

利用oz自动创建镜像需要两个配置文件：模板文件 *.tdl  和 kickstart (Red Hat-based systems) 或者preseed((Debian-based systems)。tdl文件是一个xml格式的文件，它确定了OS镜像的来源。 通常，我们在安装操作系统的过程中，需要大量的和服务器交互操作，为了减少这个交互过程，kickstart(简称ks)就诞生了。在ks 文件中记录了人工干预时填写的各种参数，如果在自动安装过程中出现要填写参数的情况，安装程序首先会去查找ks文件，如果找到合适的参数，就采用所找到的参数，找不到就合适的参数，使用默认值。其次，在ks 文件中，我们还可以添加需要安装的定制化软件包及其他脚本文件，这些定制化的步骤将在安装的第二个阶段完成。我们要做的就是告诉oz 从何处拿到tdl 文件和ks 文件，然后就去忙自己的事，等安装完毕。

更多Oz 介绍请参考：

Oz 介绍：https://github.com/clalancette/oz/wiki/Oz-architecture

Oz 支持的操作系统：https://github.com/clalancette/oz/wiki

下面以制作一个简单的虚拟机镜像为例，说明定制化虚拟机镜像的制作流程。



## 准备制作环境

本次制作镜像的HOST 机器是在VMware上建立的一个虚拟机。安装虚拟机用到的镜像版本为：CentOS-7-x86_64-DVD-1708.iso。省略HOST 虚拟机安装步骤。

Note：安装虚拟机时，点击”SOFTWARE SELECTION”, 选择 Vitalization Host 后，系统会自动配好KVM相关的虚拟机化软件，以便支持嵌套虚拟机化。

![img](/make-vm-image/os-install.jpg) 

该制作环境称为节点A。

**确认磁盘空间**

确保节点A的磁盘剩余空间约5G。

**配置节点yum 源**

配置节点的yum 源为阿里或清华的源

**安装基础软件包**

在制作镜像的的时候，我们将要在节点A上**嵌套地**启动一个虚拟机。

使用如下命令确认节点A的 CPU 是否支持 KVM 虚拟化

``` # egrep -c  "(vmx|svm)" /proc/cpuinfo  ``` 

<u>输出 0 表示不支持，非0 为支持。</u>

``` # lsmod | grep  kvm   # 查看是否加载kvm内核模块```

![img](/make-vm-image/ls-mod.jpg) 

确认节点A上libvirtd 进程处于运行状态。

**安装Oz 工具及依赖包**

```
# yum install epel-release
# yum makecache
# yum install oz  
```

OZ安装完后:

1. kickstart文件所在目录:  /usr/lib/python2.7/site-packages/oz/auto 
2. tdl文件模板所在目录： /usr/share/doc/oz-0.15.0/examples 

安装虚拟机机启动工具

```# yum install qemu-system-x86  #安装此工具，稍后用于自验制作的镜像是否能启动```

##  配置Oz

oz默认生成的镜像为raw格式，我们切换为qcow2。qcow2 格式的文件虽然在性能上比Raw 格式的有一些损失（主要体现在对于文件增量上，qcow2 格式的文件为了分配 cluster 多花费了一些时间），但是 qcow2 格式的镜像比 Raw 格式文件更小，只有在虚拟机实际占用了磁盘空间时，其文件才会增长，能方便的减少迁移花费的流量，更适用于云计算系统，同时，它还具有加密，压缩，以及快照等 raw 格式不具有的功能。

```
# vim /etc/oz/oz.cfg
[paths]
output_dir = /var/lib/libvirt/images
data_dir = /var/lib/oz
screenshot_dir = /var/lib/oz/screenshots
# sshprivkey = /etc/oz/id_rsa-icicle-gen

[libvirt]
uri = qemu:///system
image_type = qcow2
#image_type = raw
# type = kvm
# bridge_name = virbr0
# cpus = 1
# memory = 1024

[cache]
original_media = yes
modified_media = no
jeos = no

[icicle]
safe_generation = no
```

Oz 配置介绍：<https://github.com/clalancette/oz/wiki/oz-install>



## 制作镜像

**编写TDL文件**

TDL 文件为一个xml格式的文件，其完整的元素组成如下图所示：

![TDL-structure](/make-vm-image/TDL-structure.png)

如上图（图片来自：<http://imgfac.org/documentation/tdl/TDL.html>）所示，TDL文件中name 和 os 为必须元素，分别定义了模板的名称和os 镜像的描述，其他的为可选元素。详尽的模板元素说明请参考：https://github.com/clalancette/oz/wiki/Oz-template-description-language。

尽管可以通过TDL 文件的可选元素定义安装过程中的其他任务，例如：通过package元素定义需要安装的三方包，但是通常在TDL文件中只用到了其必须的元素用来定义一个操作系统的安装任务，其他定制化的任务在kickstart 中完成。

Tdl文件支持两种镜像来源：一种是iso，另一种是url，它们之间的差异是：采用iso类型时oz需要能够获取源镜像文件；URL类型则是从远程服务器获取iso解压后的文件。

以下是制作RVM 镜像时，用到的TDL文件

```
<template>
  <name>centos7_cloud_x86_64</name>
  <description>CentOS 7 x86_64 template</description>
  <os>
   <name>CentOS-7</name>
   <version>4</version>
   <arch>x86_64</arch>
   <install type='iso'>
     <iso>http://10.127.2.8/iso/CentOS-7-x86_64-Minimal-1708.iso</iso>
   </install>
  </os>
  <disk>
   <size>30</size>
  </disk>
</template>
```

tdl文件模板：https://github.com/rcbops/oz-image-build/tree/master/templates 

tdl文件裁剪示例：https://github.com/clalancette/oz/wiki/oz-examples#EXAMPLE_3__Guest_with_additional_packages

**编写Kickstart**

对于Redhat系列的OS来说，已经有内置的kickstart脚本，不过这些默认的ks文件，可能无法满足用户定制化的需求，我们可以自己制定自己的ks文件。kickstart文件支持自定义分区、自定义安装包和自定义脚本等。

通常，一个ks 文件由三部分组成：

1. 选项指令段，用于自动应答图形界面安装时除包选择外的所有手动操作

2. package选择段，定义需要安装的第三方包。 从’%packages’引导该功能

3. 脚本执行段，该段可有可无，分为两种：

   - %pre  预安装脚本段，在安装系统之前就执行的脚本，该段很少使用，因为可    	   用的命令太少

   - %post 后安装脚本段，在系统安装完成后执行的脚本

在虚拟机OS安装完成以后，Oz将调用 libvirt 使用 KVM 启动虚拟机，然后通过 ssh 等远程连接方式连接到虚拟机完成ks 文件的第二部分和第三部分内容。

更详细的Kickstart 文件结构说明: https://www.cnblogs.com/f-ck-need-u/archive/2017/08/10/7342022.htmll

以下是试验用到的kickstart文件，用注释对相关字段进行说明。对于一个简单的实践，只需要定义ks 文件的第一部分即可。

```
install    # 安装系统
Text        # 文本模式安装

# System language
lang en_US.UTF-8     # 系统安装阶段的必选项 （*）
keyboard us           #（*）
# Network information
network  --bootproto=dhcp
network  --hostname=localhost.localdomain
firewall --enabled --service=ssh
firstboot --disable  # 禁用首次启动后的手动配置界面
auth --enableshadow --passalgo=sha512  #启用shadow （*）
rootpw --iscrypted thereisnopasswordanditslocked  #使用加密密码 (*)
selinux --disabled
services --disabled="kdump" --enabled= "network,sshd,rsyslog,chronyd, docker"
timezone --utc Asia/Shanghai

# Disk
ignoredisk --only-use=vda
bootloader --append="console=tty0" --location=mbr --timeout=1 --boot-drive=vda     #（*）
zerombr   #清楚磁盘mbr
clearpart --all --initlabel #清除所有分区信息、创建标签
# --grow：使用所有可用空间，即为其分配所有剩余空间
# --size 单位是MB
part / --fstype ext4 --size=5000 --grow  

#repo  设置repo, 指定其他yum源
repo --name "os" --baseurl="http://10.127.2.8/centos/7/os/x86_64/" --cost=100
repo --name "updates" --baseurl="http://10.127.2.8/centos/7/updates/x86_64/" --cost=100
repo --name "extras" --baseurl="http://10.127.2.8/centos/7/extras/x86_64/" --cost=100
repo --name "epel" --baseurl="http://10.127.2.8/epel/7/x86_64/" --cost=100
repo --name "docker-ce" --baseurl="http://10.127.2.8/docker-ce/linux/centos/7/x86_64/stable/" --cost=100
reboot   # 安装结束后重启

--log=/mnt/sysimage/root/postinstall_stage1.log

%packages
chrony                    # 需要安装的包
cloud-init
cloud-utils-growpart
dracut-config-generic
dracut-norescue
firewalld
grub2
kernel
rsync
tar
yum-utils
wget
docker-ce
nginx
-NetworkManager   # 最前面使用横杠表示取反，即不选择
-aic94xx-firmware
-alsa-firmware
-alsa-lib
-alsa-tools-firmware
-biosdevname
-iprutils
-ivtv-firmware
-iwl100-firmware
-iwl1000-firmware
-iwl105-firmware
-iwl135-firmware
-iwl2000-firmware
-iwl2030-firmware
-iwl3160-firmware
-iwl3945-firmware
-iwl4965-firmware
-iwl5000-firmware
-iwl5150-firmware
-iwl6000-firmware
-iwl6000g2a-firmware
-iwl6000g2b-firmware
-iwl6050-firmware
-iwl7260-firmware
-libertas-sd8686-firmware
-libertas-sd8787-firmware
-libertas-usb8388-firmware
-plymouth
%end


%post --erroronfail --log=/root/postinstall.log

# passwd related

passwd -d root
passwd -l root

# 通过cloud-init 命令在第一次开机强制修改密码
cat > /etc/cloud/cloud.cfg.d/change_user_password.cfg << "EOF"
users:
   - name: default
chpasswd:
   list: |
      centos:changeme  #第一次开机登录时用户名/密码
   expire: True
ssh_pwauth: True
EOF

cat > /etc/sysconfig/network << "EOF"
NETWORKING=yes
NOZEROCONF=yes
EOF

cat > /etc/sysconfig/network-scripts/ifcfg-eth0 << "EOF"
DEVICE="eth0"
BOOTPROTO="dhcp"
ONBOOT="yes"
TYPE="Ethernet"
USERCTL="yes"
PEERDNS="yes"
IPV6INIT="no"
PERSISTENT_DHCLIENT="1"
EOF
%end
```

 **制作镜像**

```# oz-install -p -u -d3 -a centos7.ks centos7.tdl```

##  验证镜像

镜像创建完成后，镜像会保存在/var/lib/libvirt/images/下。我们可以通过以下命令先在节点A尝试启动镜像，验证制作是否成功。启动前，先备份镜像，并且使用qemu-system-x86_64 启动竟像时，需要在本地配置VNC viewer client 及在节点A配置VNC server。

```qemu-system-x86_64 -m 1024 -enable-kvm centos7_x86_64.qcow2  --vnc :1```

虚拟机启动初始化完成以后，进入登录界面。第一次登录默认用户名/密码为在ks 文件中定义的centos/changeme.

运行截图如下：

![vm-start](/make-vm-image/vm-start.png)

## 遇到的问题

**问题一：**

运行```oz install```命令时，出现qemu 运行失败的问题

![error1](/make-vm-image/error1.png)

没找到有效的解决办法，直接重换节点A的系统镜像，即选择了CentOS-7-x86_64-DVD-1708.iso。

**问题二：**

虚拟机初始化后，安装系统或包的过程中出现问题

![error2](/make-vm-image/error2.png)

这个阶段出现问题一般是ks 文件出错了，错误中提示查看后缀为ppm 的文件。 该文件是一个图片，需把改文件转换为jpg 格式后，才能查看到具体原因。上图转换后打开后如下

![error2-translate](/make-vm-image/error2-translate.png)

即错误原因为ks 文件中的```zerombr ```命令不需要任何参数，而我开始传参了（这估计和具体的oz 版本有关系）

*附图片格式转化代码*：

```
#coding=utf-8 

from PIL import Image
import os

path="/var/lib/oz/screenshots"
for file in os.listdir(path):
   img = Image.open(path+"/"+file)
   name = os.path.splitext(file)[0]
   file_name = name +".jpg"
   img.save(file_name)
```



**问题三：**

启动制作的镜像中，其中遇到cloud-init需要从默认的datasource 获取Metadata 数据。由于网络问题， 每次获取都失败，导致cloud-init 会反复尝试多个地址，这一过程，比较耗时（大约3分钟），但整体不影响虚拟机启动起来。

![error3](/make-vm-image/error3.png)

类似现状及解决办法： http://xcodest.me/cloud-init-cause-vm-boot-slow.html。