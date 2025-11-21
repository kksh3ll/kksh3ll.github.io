---
layout: post
title: "linux下安全策略selinux"
date: 2021-04-25
categories: [Linux]
---

> 

## SELinux

SELinux阻止你运行二进制文件位于用户主目录（或者在你的情况下为根用户的主目录）中的系统服务。

要解决此问题，请将二进制文件复制到适当的目录，例如，/usr/local/bin然后从那里调用它。

请注意，如果将服务文件从主目录移动到“适当的目录”，则它们的SELinux上下文仍为“主目录”。

您可以通过运行重置sudo restorecon -rv /path/to/moved/directory


## 关闭selinux

#vi /etc/selinux/config

将SELINUX=enforcing改为SELINUX=disabled