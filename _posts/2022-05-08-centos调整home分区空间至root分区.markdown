---
layout: post
title: "centos调整home分区空间至root分区"
date: 2020-05-08
categories: [Linux]
---

* 备份/home目录下文件

`tar -czvf /root/home.tgz -C /home .`
* 测试备份文件

`tar -tvf /root/home.tgz`
* 解除/home挂载

`umount /dev/mapper/centos-home`
有可能有target is busy错误，使用

`umount -l /dev/mapper/centos-home`
* 删除逻辑分区

lvremove /dev/mapper/centos-home
若出现Logical volume centos/home contains a filesystem in use，则先运行

`fuser -m -v -k /home`
 此时可能会自动注销登录，重新登录，继续运行

`lvremove -f /dev/mapper/centos-home`
* 创建新的/home逻辑分区（这里以400G为例），并重新挂载

`lvcreate -L 400GB -n home centos`
`mkfs.xfs /dev/centos/home`
`mount /dev/mapper/centos-home`
* 把其余的空间分配给/分区

`lvextend -r -l +100%FREE /dev/mapper/centos-root`
* 还原/home分区数据

`tar -xzvf /root/home.tgz -C /home`
* 最后重启先电脑（可选）

`reboot`
