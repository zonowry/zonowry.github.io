---
title: Arch + Wayland 安装手册
date: 2022-12-24
tags: [archlinux,linux,wayland,折腾]
categories: 教程
toc: true
descriptions: "先分区，需要创建一个 `fat` 格式分区，存放 `efi` 引导文件。和 `ext4` 主分区，存放系统文件。以及一个交换分区（可选），其它分区就看个人需要了。"
---

	 本文填坑中... 
	 * wayland 桌面环境内容待补充中



## live 内步骤

### 硬盘分区、格式化、挂载

先分区，需要创建一个 `fat` 格式分区，存放 `efi` 引导文件。和 `ext4` 主分区，存放系统文件。以及一个交换分区（可选），其它分区就看个人需要了。

```bash
fdisk /dev/{sda/sdb/nvme...} # 选择一块硬盘进行分区
> g # gpt 分区表，uefi 启动
> n # 新建分区
> # ... 其它步骤
```

格式化磁盘

```bash
mkfs.fat /dev/sda1
mkswap /dev/sda2 
mkfs.ext4 /dev/sda3
```

挂载分区

```bash
mount /dev/sda3 /mnt
swapon /dev/sda2 # 启用交换分区
```

![分区案例](/images/blog/arch/fdisk-list.png)


### 替换 pacman 安装源

用 reflector 自动生成 pacman 源

```bash
reflector --age 24 --country China --sort rate --verbose --protocol http --protocol https --save /etc/pacman.d/mirrorlist
```


### 安装系统与基础软件

下载安装系统内核，以及网络管理、ssh之类的必备软件。我常用的软件列表以后可能会整理一篇 linux 日常使用的文章，到时候再罗列介绍吧。

如果安装时遇到签名错误，我是通过更新 archlinux-keyring 解决的，仅供参考，网上还有很多其它方法。

```bash
pacman -Sy archlinux-keyring 
```

> 如果是当作日常桌面系统使用，系统内核推荐安装 `linux-zen` ，看[linux-lqx(zen) 官网介绍](https://liquorix.net/)是比较适合桌面/游戏用户的。

安装一些系统必备软件，其它输入法之类的日用软件后续步骤安装，每个人喜好/环境不同，部分我安装的软件当作一种选择即可。

```bash
pacstrap /mnt base linux-zen linux-firmware sudo neovim networkmanager openssh git zsh grub efibootmgr intel-ucode
```

### 进入系统

```bash
arch-chroot /mnt
```

### grub 启动项配置

```bash
mkdir /efi
mount /dev/sda1 /efi # 挂载 fat 分区
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

### 添加用户

设置用户密码，配置普通用户 sudo 权限。

```bash
useradd -m kasyoumi
passwd kasyoumi
passwd root
# 配置suders
nvim /etc/sudoers
# 添加
kasyoumi ALL=(ALL) ALL
```


### fstab 配置

设置系统启动时自动挂载的盘，退出 `arch-chroot` ，通过脚本自动生成 fstab 配置。

```bash
exit # 退出 chroot /mnt 环境
# ---
genfstab /mnt > /mnt/etc/fstab
```

建议验证下，生成的是否正确

```bash
cat /mnt/etc/fstab
```

![cat-fstab](/images/blog/arch/cat-fstab.png)


## 重启进入系统

上边的步骤做完，重启应该就可以引导并进入系统了。

### 初始化网络

```bash
systemctl enable NetworkManager
systemctl start NetworkManager
```

### 时区本地化配置

```bash
timedatectl set-ntp true # ntp 同步时间
timedatectl set-timezone Asia/Shanghai
timedatectl # 查看验证
```

### 语言本地化配置

可能影响一些系统的提示语言，货币和时间的显示格式。

> 程序员的话建议就用 en_US.utf8 吧，一些命令报错，英文的话比较好搜索解决方法。

```bash
vim /etc/locale.gen # 解除注释，保存退出
locale-gen
localectl set-locale LANG=en_US.UTF-8
```

## wayland 桌面设置

这一步就不过多解释了，桌面环境涉及太多因素了（显卡、cpu、蓝牙、声音等等），有问题得具体沟通，这里我就简单记录一下我的操作步骤了。

### 安装

```bash
pacman -S wayland sway 
```


### sway 配置

窗口管理器我用的是 sway，一款平铺桌面管理器。

```bash

```


### 一些日用软件

```bash
pacman -S fcitx5 fcitx5-rime fcitx5-configtool zsh 
```


```bas
wget https://github.com/Morganamilo/paru/releases/download/v1.11.2/paru-v1.11.2-x86_64.tar.zst

pacman -U paru-v1.11.2-x86_64.tar.zst
```


## 注意事项

