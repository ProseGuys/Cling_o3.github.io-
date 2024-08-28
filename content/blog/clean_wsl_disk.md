---
title: WSL 虚拟磁盘清理
date: 2024-08-27T09:18:59+08:00
tags: ["WSL"]
series: []
featured: true
---
为什么在WSL里删除文件后，Windows的磁盘空间却纹丝不动？
WSL 磁盘清理计划，为你的磁盘保驾护航

<!--more-->
你有没有遇到过这样的情况：在WSL（Windows Subsystem for Linux）中勤勤恳恳地清理了一堆文件，结果转到Windows查看磁盘空间，发现那个数字一点都没变？这到底是哪里出了问题？今天我们来揭开这个令人困惑的谜团！

## 1. 原因分析

WSL2 运行在一个虚拟化的Linux文件系统之上，而这个文件系统本质上是一个存储在Windows中的虚拟硬盘文件（通常是ext4.vhdx）。当你在WSL中删除文件时，文件系统的元数据可能已经更新，但这个.vhdx文件的大小不会自动缩小。Windows并不知道你在WSL中发生了什么，它只是看着这个.vhdx文件的总大小，所以磁盘空间似乎没有变化。

## 2. 如何释放真正的空间

### 2.1 找到对应的WSL服务

我的 `WSL2` 有如下的 Linux distributions：

```bash
>wsl -l -v
  NAME            STATE           VERSION
* Ubuntu-22.04    Running         2
```

可以通过Windows的文件资源管理器搜索ext4.vhdx的位置，如下图所示：

![image.png](/images/clean_wsl_disk/image.png)

由于我这里磁盘空间不足主要是 `Ubuntu-20.04` 删除文件后 `ext4.vhd` 没有缩容引起的，所以只压缩了它的 `ext4.vhdx`。

如果出现删除 `Docker` 镜像、删除 `Docker` 容器后磁盘占用没有缩小，应该也可以类比操作。

### 2.2 压缩前的准备工作（可选）

如果你害怕在压缩虚拟磁盘时，因为异常导致的压缩失败，你可以先进行ext4.vhd的备份（当然，要是空间不够的情况下，还是算了吧）

在 `PowerShell` 中执行：

```bash
# 关闭 WSL2 中的 linux distributions
wsl --shutdown
# 备份指定的 Linux distribution 到指定的位置
wsl --export Ubuntu-20.04 D:\Ubuntu-20.04.tar
```

### 2.3 手动压缩虚拟磁盘

在 `PowerShell` 中执行：

```bash
# 关闭 WSL2 中的 linux distributions
wsl --shutdown
# 运行管理计算机的驱动器的 DiskPart 命令
diskpart
```

在新打开的 `DiskPart` 命令窗口中执行：

```bash
# 选择虚拟磁盘文件
select vdisk file="D:\Ubuntu_WSL\ext4.vhdx"
# 压缩文件
compact vdisk
# 压缩完毕后卸载磁盘
detach vdisk
```

上述操作执行完毕，`WSL2` 删除文件后空出来的磁盘空间就被释放了。

## 文件迁移方案：

当我们第一次使用WSL2的时候，大概率并不知道Windows会进行什么处理和操作。听之任之得让他将虚拟磁盘丢在了C盘，直到系统快要卡死时才明白C盘已经爆满了，我们还可以通过迁移ext4.vhdx的方式，将C的虚拟磁盘丢入D盘中。

### 导出镜像：

```bash
wsl --export <Image Name> <Export location file name.tar>
```

### 导入镜像：

```bash
wsl --import <Image Name> <Directory where you want to store the imported image> <Directory where the exported .tar file exists>
```

### 验证：

```bash
wsl --list
```

![image.png](/images/clean_wsl_disk/image_1.png)

详情可参考以下文档：https://www.virtualizationhowto.com/2021/01/wsl2-backup-and-restore-images-using-import-and-export/

## **总结**

在WSL中删除文件后，Windows磁盘空间未变的原因其实是由于WSL使用虚拟硬盘存储文件，而删除文件后这个虚拟硬盘并不会自动缩小。通过手动压缩虚拟硬盘，你可以有效释放被占用的磁盘空间。掌握这些小技巧，让你的系统运行更加高效流畅！