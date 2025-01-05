---
author: Huang Yanchen
title: 再建Arch Linux | Reinstall Arch Linux
date: 2024-08-07
description: A brief guide to setup KaTeX
math: true
ShowBreadCrumbs: false
tags: ["Linux", "VMWare Workstation Pro"]
categories: ["Linux"]
showToc: true
TocOpen: false
UseHugoToc: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
---
## 和 Arch Linux 的渊源纠葛

在大一入学后，我开始接触到了 Linux 操作系统，觉得这种命令行交互的方式很酷炫（程序员不就应该咔咔敲键盘吗）。但是最开始我也只知道 Ubuntu、Centos 这种发行版本，对于 Arch Linux 几乎没有了解过。恰好我有个学长在某一天无意间给我展示了他的桌面，酷似苹果的 UI 瞬间给我种了草（虽然桌面 UI 和发行版没啥关系），他的系统正是 Arch Linux。于是乎我就想开始尝试安装自己安装一下 Arch Linux，这个号称最难安装、可玩性最高、软件包维护方便的 Linux 版本。

其实在去年几乎同一时间，我已经在 VMWare Workstation Pro 上成功安装过 Arch Linux，但由于虚拟机性能有限，再加上当时只是无脑跟着教程走了一遍（~~虽然这遍也是跟着教程无脑走的~~），再加上当时 pacman 包给我留下了鱼龙混杂的印象，总的来说使用体验不如我另一台 Ubuntu20.04 的虚拟机。为了再体验一下 Arch Linux 的魅力（其实只是下午摸鱼，但弄得我头昏脑胀，已经远远超出摸鱼的范畴了），我今天就又尝试安装一个 Arch Linux 的虚拟机，给我将来刷物理机做准备。

好吧，不得不事先声明一下，其实今天安装失败了说是，属于是白忙活一下午，看到这里的读者已经可以退出了==

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121254887.jpg)

## 动机

除了上述原因之外，这两天在 Westlake University 的服务器上下 python 包的时候，连接 pypl 官方镜像很不稳定，网上常用的镜像源是清华源、中科大源、阿里源。但我突然想到，我们南大不也有镜像仓库吗，为什么我们南大学生清一色都是换的清华源或中科大源呢？本着物尽其用的原则，于是我就给服务器换上了南大源，让我自豪地挺起了胸膛。在浏览南大镜像仓库的时候发现还有对应的 Arch Linux 的软件仓库，这一下子勾起了我遥远的 Arch Linux 的回忆，所以我就想这次用南大镜像配一下 Arch Linux（很莫名其妙的动机了属于是）。

再次声明，本教程为失败教程，看到这里的读者已经可以退出了==

## 流程

### 镜像下载

从[南京大学镜像仓库](https://mirror.nju.edu.cn/)中下载 Arch Linux x86-64 的镜像（~~南大人不就该用南大镜像仓库吗~~）

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121254798.png)

### 验证镜像签名

这一步就先跳过了，好像是可有可无的步骤

### 虚拟机的创建

在 VMWare Station 中创建一个新的虚拟机，版本就选择图中的 Ubuntu 64 位。磁盘我直接分了 100GB，内存分配了 8GB，其余配置均是默认配置

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121254128.png)
![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121254814.png)

虚拟机配置完成，启动虚拟机，启动成功（接下来每当我以为系统安装完成重启之后还会反复看到这个画面，今晚梦里都是这个了）

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121254079.png)

### Arch Linux 安装

#### 键盘布局和终端字体配置

键盘配置的是 US，终端字体选择 ter-132b

#### 网络连接配置

还没有配置网络连接就是正常的，能够 ping 通 archlinux.org（貌似物理机里网络这里是比较复杂的，虚拟机的好处就体现出来了）

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121254829.png)

但不知道为什么这里我的 root 变成 130 root 了 （[reddit 帖文](https://www.reddit.com/r/archlinux/comments/bl55hd/what_are_the_numbers_preceding_rootarchiso/)在这里找到了有同样的疑问，但是没有人回答）

#### 更新系统时间

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121254998.png)

#### 进行分盘操作

最开始硬盘分布是这样的

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121254286.png)

创建 GPT 分区

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121254149.png)

创建根分区（第一次只分配了 10G，然后删除分区重新分了 40G 上去）

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121254835.png)

创建 home 分区

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121254697.png)

分区状态最终打印如下

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121254214.png)

#### 挂载分区

挂载分区时失败了

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121255578.png)

原因是没有将分区格式化为 ext4 文件系统，之后再挂载就没有报错了

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121255271.png)

接着给仓库镜像换成南大源 `https://mirror.nju.edu.cn/archlinux/$repo/os/$arch` 然后更新软件包缓存

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121255002.png)

#### 安装三件套（姑且称呼为三件套吧）

安装 base linux linux-firmware 软件包

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121255019.png)

安装成功，但是中间出现了很多 WARNING，没有 ERROR 就先不管了

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121255497.png)

生成 fstab

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121255378.png)

#### 进入 arch-chroot

设置时区和硬件时间

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121255757.png)

在新系统中安装 vim 后，编辑/etc/locale.conf

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121255231.png)

#### 使用 pacman 安装 Networkmanager

```shell
Pacman -S networkmanager
```

将之作为网络管理工具，并配置开机自启动

#### 设置 root 密码

在此步骤中配置了 root 密码

#### 安装 Boot Loader

这里安装的是 intel-ucode

#### 退出 arch-chroot 并重启

按照官方教程，在上一步骤完成后 `exit` 退出 arch-chroot，取消挂载后，重启就可以了

然而刚刚一通操作下来，一直无法正确引导启动，重启之后界面居然还是这样的：

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121255604.png)

于是我找到了去年这个时候我看的教程，按照教程继续尝试了一下

#### 尝试挣扎 做了一些官方教程中没有的步骤

首先是配置 host

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121255194.png)

安装 EFI 启动管理器

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121255825.png)

配置/boot/loader/loader.conf 启动文件

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121255095.png)

配置/boot/loader/entries/arch/conf

![](https://yukino13.oss-cn-hangzhou.aliyuncs.com/blog/202408121255631.png)

运行以下命令

```shell
blkid -s PARTUUID -o value /dev/sda3 >> /boot/loader/entries/arch.conf
```

然后再次尝试开机，再次失败了

## 总结

感觉如果要刷物理机的话，关于刷 linux 的很多坑都得好好填一下，关于分区操作，关于挂载的机制，关于启动引导......

但是接下来我还有另外的大坑要补，对于深度学习的很多内容我的理解并不深刻，对于一些数学方面的如概率统计、微分方程等都需要好好加强一下，不然看一些论文很难有深层次的理解，也正是因为这种不理解，让我在这个阶段对读论文并没有特别的兴趣，导致这几天一直在划水摸鱼......

对于 linux 刷机相关的内容，或许等到我有新的设备后，可以尝试将我这台 y9000p 2021 刷成纯血的 Arch Linux，这段时间就先不折腾了。
