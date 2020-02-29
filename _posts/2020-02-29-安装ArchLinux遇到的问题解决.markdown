---
layout: post
title: "安装ArchLinux系统"
date: 2020-02-29
categories: [Linux]
---

## 双系统，保留Widows7

安装的过程中主要是系统引导和磁盘格式化的问题，大致步骤如下：

* 通过easyBCD添加I启动项，我把iso文件放到了C盘，提取出目录 `ARCH\BOOT\X86_64\`下的`vmlinuz`和`archiso.img`两个文件同样也放到C盘根目录。在menu.lst文件输入一下内容：

{% highlight shell %}
title Install ArchLinux
root (hd0,0)
kernel /vmlinuz archisolabel=archlinux
initrd /archiso.img
boot
{% endhighlight %}

* 重启，选择启动项，进入安装界面rootfs的shell。

* 输入以下命令手动加载iso文件：

{% highlight shell %}
mkdir /tmpmnt
mount -r -t ntfs /dev/sda1 /tmpmnt
modprobe loop
losetup /dev/loop6 /tmpmnt/archlinux.iso
ln -s /dev/loop6 /dev/disk/by-label/archlinux
exit
{% endhighlight %}

* 进入ArchLinux的root系统中，现在可以分区格式化把ArchLinux系统装入磁盘中。

* 使用fdisk分区的时候可以分成3个区，一个是boot分区，512M大小，一个是swap分区，大小为2G，另一个是／分区，大小为剩余的空间。

* boot分区的时候格式必须是FAT格式，`mkfs.fat -FAT32 /dev/sda5`。

* 格式化主分区`mkfs.ext4 /dev/sda6`。

* 格式化交换分区`mkswap /dev/sda7`和`swapon /dev/sda7`。

* mount主分区`mount /dev/sda6 /mnt`并创建一个boot目录`mkdir /mnt/boot`并挂载`mount /dev/sda5 /mnt/boot`。

* 安装的时候需要联网，最简便的方法是用手机USB共享网络，有的无线网卡驱动不识别，修改`/etc/pacman.d/mirrorlist`。

* 开始安装`pacstrap -i /mnt base`。

* 多系统引导的话需要安装grub，然后新建`/boot/grub`目录。

* 执行命令`grub-mkconfig > /boot/grub/grub.cfg`生成配置。

* 安装grub，命令`grub-install --target=i386-pc /dev/sda`。

## 遇到的问题汇总

1. 分区的时候，执行命令`mkfs.ext4 /dev/sda5`发生错误：'没有那个文件或目录'，找不到分区设备的情况下，执行此命令`partprobe`，这个时候就可以正常运行了。

2. 安装完成后，无法进入Windows7，只能在grub引导的时候输入'c'进入grub命令行，然后执行如下命令：

{% highlight shell %}
set root=(hd0,1)
chainloader +1
boot
{% endhighlight %}

进入windows7系统！






























