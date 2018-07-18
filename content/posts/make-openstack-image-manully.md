---
title: "手动制作OpenStack 虚拟机镜像"
draft: false
date: "2018-06-07T19:05:57+08:00"
categories: "OpenStack"
tags: ["openstack, image"]
---




最近对制作OpenStack 虚拟机镜像起了兴趣，主要是因为自己在OpenStack集群上创建了一些虚拟机并安装了常用的软件，但是这个OpenStack环境有时又会拆毁重建，为了避免在集群搭建好之后，重新从零开始创建虚拟机及安装软件包，自己将之前装了基础软件包的虚拟机做成了镜像，保存备用。

制作镜像时，大概参考https://docs.openstack.org/image-guide/create-images-manually.html ，但是该手册中还有当时没有读明白，或者没提及到的地方，直接按照手册也遇到了许多坑，特别是在虚拟机根分区自动扩容那块，花费了很长时间来研究，这也是写此笔记的原因之一。

该指导制作的虚拟机镜像具备：

1. 新创建虚拟机根分区第一次启动时自动扩容
2. 虚拟机hostname 注入，IP 注入
3. 不支持创建虚拟机时密码注入

#### 准备工作：

1. 一套OpenStack 环境（Pike 版本）

2. 制作镜像的ISO镜像：CentOS-7-x86_64-Minimal-1804.iso

#### 安装操作系统：

在此，利用OpenStack 从ISO启动一个虚拟机，具体步骤如下：

- 上传ISO 镜像到glance

```
openstack image create --file ~/CentOS-7-x86_64-Minimal-1804.iso --public --container-format=bare --disk-format=raw centos-7.iso
```

- 创建5G 大小的空卷，并设置卷可启动属性

```
openstack volume create --bootable --size 5 lzg-bootable-volume
```
- 创建虚拟机

   在运行以下命令时，需提前创建好其他资源，如网络，安全组，flavor 等。

   ```
   nova boot --nic net-name=test-net --security-groups test-security-group  \
   --block-device id=aaeabee6-1361-4bfd-8937-03bcee60ddd3, \
   source=image,dest=volume,bus=ide,type=cdrom,size=1,bootindex=1 \
   --block-device id=43c0f8a1-d8a1-493c-89e1-cf596ada1777, \
   source=volume,dest=volume,size=5,type=disk,bootindex=0 \
   --flavor m1.medium  test-vm
   ```

   ![img](/make-vm-image/nova-boot-cmd.png) 

- 安装操作系统

   - 虚拟机启动之后，便可通过VNC方式进入虚拟机，到达安装系统界面，系统安装和平时差不多，需要注意是选择手动分区，不要选择系统默认的LVM 方式，并且根分区要位于块设备分区的最后，选择 ext4 或 xfs 文件系统，不设置swap 分区，以便软件自动化扩展根分区。
     不设置swap分区的原因是在该系统上手动分区好多次，数次把swap置于vda2 ，根分区置于vda3 时，切换根分区文件系统格式，对根分区做"update setting"操作后，swap 分区又自动变回 vda3。
     亲测ext4,xfs 方式都支持根分区自动扩容，这一点与参考手册Partition the disks 一节做的有点不一样，但是原文这句话也很重要：having the partition that you want to dynamically grow at the end of the list will allow it to grow without crossing another partition’s boundary. ）例如，我的分区方式是：

     ![img](/make-vm-image/disk-partition.png) 

   - 开启网络连接，以便后边在系统中手动设置。

     ![img](/make-vm-image/network-setting.png) 

系统安装完毕之后，重启系统，进入系统，安装相关软件 


#### 系统设置

- 安装必要软件包

  ```
  yum install cloud-init    // 版本是：0.7.9-24.el7.centos
  yum install cloud-utils-growpart //  该包便是自动扩容的支持软件,版本是：0.29-2.el7 
  yum install acpid
  systemctl enable acpid
  ```


- 修改配置文件`/etc/default/grub`

  为了nova console-log才能获取虚拟机启动时的日志，修改配置文件`/etc/default/grub`，删掉```GRUB_CMDLINE_LINUX``` 一行中 ``rhgb quiet ``,添加 ```console=tty0 console=ttyS0,115200n8``` , 修改后的结果如下：

  ```
  GRUB_CMDLINE_LINUX="crashkernel=auto console=tty0 console=ttyS0,115200n8"
  ```

  运行以下命令，保存修改

  ```
   grub2-mkconfig -o /boot/grub2/grub.cfg
  ```

- 修改cloud-init 配置文件 ```/etc/cloud/cloud.cfg```

  默认情况下，cloud-init 在虚拟机启动时，会关闭ssh 的密码认证方式，使用key 方式连接。修改cloud-init 的配置文件，使其打开ssh 密码认证方式。方法如下：

  ```
  vi /etc/cloud/cloud.cfg
  
  users:
   - default
     
  disable_root: 1
  ssh_pwauth:   1 // 将默认的0 ,变为1。
  ```

- 安装其他需要的基础软件, 如vim, docker  等。

​  
####   制作镜像

- 关闭机器，上传卷到glance 镜像库

```
cinder upload-to-image 43c0f8a1-d8a1-493c-89e1-cf596ada1777 my-image --disk-format qcow2 \
--force
```

稍等一会，待到上传的image 状态是"active" 状态

- 利用该镜像，选择合适的flavor,启动该镜像。

```
openstack server create --flavor m1.small --image my-image   --network test-net --security-group test-security-group   --config-drive true  scaled-vm
```

此处,m1.small 的disk 大小为20。VNC登录，使用```df -h`` 命令，可以看到根分区文件系统已经扩大。

- 从glance 中导出镜像到本地，等待片刻，导出结束。  

  ```
  openstack image save  my-image --file my-image  // 导出的image 还是qcow2格式 
  ```
​
#### 其他实验

据上边引用的原文那段话，猜想cloud-utils-growpart 是否支持其他最后一个分区自动扩容，自己也测了一下，算是对cloud-utils-	growpart 的了解。特地将/var 作为vd3, 重新做以上工作，对比制作镜像前后启动的虚拟机分区情况，可以发现根分区以外的分区不能自动扩容。

实验结果如下：

![img](/make-vm-image/error-test1.png) 

![img](/make-vm-image/error-test2.png) 

  

   

   





   

   