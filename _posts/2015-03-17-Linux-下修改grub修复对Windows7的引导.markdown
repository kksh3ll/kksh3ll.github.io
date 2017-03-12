---
layout: post
title:  Linux下修改grub修复对Windows7的引导
date:   2015-03-17 16:44:21
categories: Linux
---
## Linux下修改grub

进入Linux系统中，在终端中输入：`sudo vi /boot/grub/grub.cfg`回车，输入root密码，打开`grub.cfg`文件。在文件的末尾找到`### BEGIN ／etcgrub.d/40_custom ###`代码段：在下面加入如下代码：

{% highlight shell %}
memuentry "Windows7" {
  insmod part_msdos
  insmod ntfs
  set root = '(hd0,msdos1)'
  chainloader +1
}
{% endhighlight %}

保存后重启即可。

代码中的`（hd0,msdos1）`为Windows系统所在分区，其中的“Windows7”就是开机引导中的名字。修改配置文件注意Windows系统所在分区。
