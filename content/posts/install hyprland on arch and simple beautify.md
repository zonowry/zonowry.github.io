---
tags:
  - article
  - arch
  - hyprland

creation date: 2024-02-16 04:12:28
name: "install hyprland on arch and simple beautify"
title: Arch + Hyprland 安装手册
descriptions: wayland 发展了许久，生态感觉可以了。遂使用了一个月，除了一些无法解决的小“毛刺”，日常使用还是可以接受的。坚持使用下去原因：hyprland 的平铺桌面好用又好看。

description: wayland 发展了许久，生态感觉可以了。遂使用了一个月，除了一些无法解决的小“毛刺”，日常使用还是可以接受的。坚持使用下去原因：hyprland 的平铺桌面好用又好看。
date: 2024-02-16
toc: true
isCJKLanguage: true
keywords:
  - zonowry
  - Arch + Hyprland 安装手册
  - linux
  - wayland
---

首先展示成果

![preview](/images/blog/arch/image-2024_02_25_03_47_36.png)

## 首先安装 Arch

进入 live 。

### 硬盘分区

先使用 `fdisk` 对磁盘分区，至少需要创建一个 `fat` 格式分区，存放 `efi` 引导文件。和 `ext4` 主分区，存放系统文件。以及一个交换分区（可选），其它分区就看个人需要了。

```bash
fdisk /dev/{sda/sdb/nvme...} # 选择一块硬盘进行分区，以下使用 /dev/sda 作为例子
> g # gpt 分区表，uefi 启动
> n # 新建分区
> # ... 其它步骤
```

划分出：

- `/dev/sda1` vfat 分区
- `/dev/sda2` 主分区
- `/dev/sda{n}` 其他分区

### 格式化磁盘

```bash
# 主分区
mkfs.ext4 /dev/sda2
```

### 挂载分区

挂载主分区安装系统到磁盘。

```bash
mount /dev/sda2 /mnt
mkdir -p /mnt/efi
mount /dev/sda1 /mnt/efi
```

### 替换 pacman 安装源

加速 `pacman` 安装速度。用 `reflector` 自动配置 `pacman` 使用国内镜像源。

```bash
reflector --age 24 --country China --sort rate --verbose --protocol http --protocol https --save /etc/pacman.d/mirrorlist
```

~~也可以编辑 mirrorlist 文件，解开 `[China]` 下的镜像源注释并注释掉官方源，或者用手机上网搜索最新的国内 archlinuxcn 镜像源吧，然后手动敲上。~~

### pacman 配置

如果安装时遇到签名错误，应该可以通过以下方式解决。~~如果你的 `live` 不是很新那么大概率会签名错误，所以推荐先执行以下，也没坏处。~~

```bash
pacman-key --init
pacman-key --populate archlinux
# Syy 刷新下
pacman -Syy archlinux-keyring
```

### 开始安装：

下载安装系统内核，以及网络管理、ssh 之类的**必备软件**。

> PS：如果是当作日常桌面系统使用，系统内核推荐安装 `linux-zen` ，看[linux-lqx(zen) 官网介绍](https://liquorix.net/)是比较适合桌面/游戏用户的。但会遇到因为不是官方标准内核，所以会碰见部分驱动需要用 `dkms` 自己编译的情况。例如 `nvidia` ，简单起见就先用 `linux` 标准内核吧，以后可以再安装切换到 `linux-zen`，也不麻烦。

> PPS：如果你是 intel 就安装 intel-ucode，amd 就安装 amd-ucode（大概是这个名字，参考 archlinux wiki 吧）。

```bash
pacstrap /mnt \
linux \
linux-firmware \
base \
sudo \
neovim \
networkmanager \
openssh \
git \
zsh \
grub \
efibootmgr \
intel-ucode \
base-devel \
devtools
```

### chroot 进入系统

安装完成后，进入系统，方便后续做一些处理。

```bash
arch-chroot /mnt
```

### grub 启动项配置

```bash
mkdir /efi
# 挂载第一步创建的 vfat 分区
mount /dev/sda1 /efi
# grub uefi 启动项
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB
# grub 引导/菜单配置
grub-mkconfig -o /boot/grub/grub.cfg
```

### 添加用户

设置用户密码，配置普通用户 sudo 权限。

```bash
useradd -m {username}
# 为新用户设置密码
passwd {username}
# 为 root 设置密码
passwd root

# 允许新用户使用 sudo 命令
echo "{username} ALL=(ALL) ALL" >> /etc/sudoers


```

### fstab 配置

设置系统启动时自动挂载的分区，退出 `arch-chroot` ，通过 `arch` 安装脚本自动生成 `fstab` 配置。

```bash
exit # 退出 chroot /mnt 环境
# ---
genfstab /mnt > /mnt/etc/fstab
```

建议验证下，生成的是否正确

```bash
cat /mnt/etc/fstab
```

![[image-2023_09_25_16_44_46.png]]

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
nvim /etc/locale.gen
# 解除 en_US.UTF-8 的注释，
# :wq 保存退出

# 生成
locale-gen
localectl set-locale LANG=en_US.UTF-8
```

### ”滚“一下

`Arch` 新系统怎么能不更新一下呢。

```bash
pacman -Syyu
```

### Aur 客户端

一般使用 `yay` ，不过我图新鲜用的 `paru` 。其实也不知道具体啥区别，到底哪个好用。感觉除了开发语言不同，功能上都差不多吧。

去 `paru` 的 `github` release 页面下载二进制包，然后解压创建 `/usr/bin/paru` 软链接就好了。

```bash
# 或者直接 `wget` ，需要自己判断版本链接。
wget https://github.com/Morganamilo/paru/releases/download/v1.11.2/paru-v1.11.2-x86_64.tar.zst

# tar 解压 paru-v1.11.2-x86_64.tar.zst
```

```bash
paru -Syyu
```

## hyprland 安装

`hyprland`，一款平铺桌面管理器。动画流畅，丰富的自定义选项。最好配合 `hyprland` 官网食用。

```bash
pacman -Sy hyprland
```

## 一些桌面日用软件

日用软件因为在 `linux` 下的”开源“性，由于各自环境差异导致碰到问题各异。以下软件、配置可以作为一些参考、思路吧。

按照步骤一步步做，也可能会遇到的问题，就需要自己分析了。。。

### 输入法

```bash
pacman -S fcitx5 fcitx5-rime fcitx5-configtool
```

强推下雾凇拼音 https://github.com/iDvel/rime-ice

```bash
paru -S rime-ice-git
```

### 多媒体/音频/录屏

- pipewire
- wireplusember
- pipewire-pluse
- pipewire...

### 蓝牙

使用 bluectl 来 connect device，也可以用 `blueman` gui 链接。可参考 archlinux wiki 的 蓝牙相关页面。

- bluez
- bluez-utils
- blueman-applet

### 主题

这个其实就很有必要单独讲讲。感觉可以出一篇 `linux` 的 `GUI` 美化配置文章了，这里就简单罗列我美化时用到的软件，可以先自行扩展搜索下相关资料，或许就知道该如何配置、美化了。

- qt5ct
  - 为基于 qt 开发的程序进行配置的程序
- kvantum
  - 将 gtk 主题应用到 qt 程序上
- orchis-theme
  - 一个 gtk 主题
- kvantum-orichis-theme
  - kvantum （qt）的主题
- uwg-look
  - wayland 下代替 lxapprence 的程序
  - 为基于 gtk 开发的程序进行配置的程序
- tela-icon-theme
  - 一套图标
- 还有个鼠标图标
  - 忘记名字了。。。

### zsh 终端美化配置

zsh 一般不许要自己手动配置，或者配置过一次后，就不需要重新配置了。

直接去下载别人的 `dotfiles` ，复制到自己 `home` 目录下即可。例如我的：[https://github.com/zonowry/dotfiles](https://github.com/zonowry/dotfiles)

- oh-my-zsh
- powerlvel10k 主题
- fzf
- 以及其他一些插件
- zsh shift 选择文字
  - https://github.com/jirutka/zsh-shift-select

### 一些杂项软件

- wev
  - 查找键盘键码的工具
