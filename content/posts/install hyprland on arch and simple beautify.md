---
tags:
  - article
  - arch
  - hyprland
creation date: 2024-02-16 04:12:28
name: install hyprland on arch and simple beautify
title: Arch + Hyprland 安装美化手册
description: 对 CJK 输入法用户最友好的一集
tags:
  - article
  - nat
  - network
date: 2024-09-20
keywords:
  - zonowry
  - hyprland
  - arch
  - linux
  - btrfs
  - theme
  - fcitx
isCJKLanguage: true
toc: true
---

首先展示成果

![preview](/images/blog/arch/image-2024_02_25_03_47_36.png)

## 进入 Arch LiveCD 开始安装

### 硬盘分区

先使用 `fdisk` 对磁盘分区，至少需要创建一个 `fat` 格式分区，存放 `efi` 引导文件。和一个 `btrfs` 主分区，存放系统文件。

> 下文使用 /dev/nvme0n1 作为例子。

```bash
fdisk /dev/nvme0n1 # 选择一块硬盘进行分区
> g # gpt 分区表，uefi 启动
> n # 新建分区
> ...
> w # 保存退出
```

### 格式化磁盘

```bash
# 主分区
mkfs.vfat /dev/nvme0n1p1
mkfs.btrfs -L arch-btrfs /dev/nvme0n1p2 # 建立 btrfs 分区并命名 Label
```

### 挂载分区 & 初始化 btrfs 

```bash
mkdir -p /mnt/efi
mount /dev/sda1 /mnt/efi
```

挂载主分区安装系统到磁盘。

> btrfs 压缩算法区别 [Compression](https://btrfs.readthedocs.io/en/latest/Compression.html#incompressible-data)

```bash
mount -t btrfs -o compress=zstd /dev/nvme0n1p2 /mnt
btrfs subvol create /mnt/@
btrfs subvol create /mnt/@home
btrfs subvol create /mnt/@var-cache
btrfs subvol create /mnt/@var-temp
btrfs subvol create /mnt/@opt
btrfs subvol create /mnt/@swap

umount /mnt
mount -t btrfs -o compress=zstd,subvol=@ /dev/nvme0n1p2 /mnt
mount --mkdir -t btrfs -o subvol=@home /dev/nvme0n1p2 /mnt/home
mount --mkdir -t btrfs -o subvol=@opt /dev/nvme0n1p2 /mnt/opt
mount --mkdir -t btrfs -o subvol=@var-cache /dev/nvme0n1p2 /mnt/var/cache
mount --mkdir -t btrfs -o subvol=@var-temp /dev/nvme0n1p2 /mnt/var/temp

btrfs fi mkswapfile /mnt/swap/swapfile --uuid clear --size 16G
swapon /swap/swapfile
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

> PS：如果是当作日常桌面系统使用，系统内核推荐安装 `linux-zen` ，看 [linux-lqx(zen) 官网介绍](https://liquorix.net/) 是比较适合桌面/游戏用户的。但会遇到因为不是官方标准内核，所以会碰见部分驱动需要用 `dkms` 自己编译的情况。例如 `nvidia` ，简单起见就先用 `linux` 标准内核吧，以后可以再安装切换到 `linux-zen`，也不麻烦。

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
devtools \
btrfs-progs
```

### chroot 进入系统

安装完成后，进入系统，方便后续做一些处理。

```bash
arch-chroot /mnt
ln -s /usr/bin/nvim /usr/bin/vim
```

### grub 启动项配置

```bash
mkdir /efi
# 挂载第一步创建的 vfat 分区
mount /dev/sda1 /efi
# grub uefi 启动项
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=GRUB

# TODO
vim /etc/default/grub

# loglevel=5, nowatchdog

# grub 引导/菜单配置
grub-mkconfig -o /boot/grub/grub.cfg
```

### 添加用户

设置用户密码，配置普通用户 sudo 权限。

> 用户名「zonowry」改成自己的即可。

```bash
useradd -m zonowry
# 为新用户设置密码
passwd zonowry
# 为 root 设置密码
passwd root
# 允许新用户使用 sudo 命令， 这里我设置了 sudo 免密码，可以去掉
echo "zonowry ALL=(ALL) NOPASSWD:NOPASSWD:ALL" >> /etc/sudoers
```

### fstab 配置

设置系统启动时自动挂载的分区，退出 `arch-chroot` ，通过 `arch` 安装脚本自动生成 `fstab` 配置。

```bash
exit # 退出 chroot /mnt 环境
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

systemctl enable sshd
systemctl start sshd
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
vim /etc/locale.gen
# 解除 en_US.UTF-8 的注释，
# :wq 保存退出

# 生成
locale-gen
localectl set-locale LANG=en_US.UTF-8
```

### Aur 客户端

一般使用 `yay` ，不过我图新鲜用的 `paru` 。其实也不知道具体啥区别，到底哪个好用。感觉除了开发语言不同，功能上都差不多吧。

去 `paru` 的 `github` release 页面下载二进制包，然后解压创建 `/usr/bin/paru` 软链接就好了。

```bash
# 或者直接 `wget` ，需要自己判断版本链接。
wget https://github.com/Morganamilo/paru/releases/download/v1.11.2/paru-v1.11.2-x86_64.tar.zst

# tar 解压 paru-v1.11.2-x86_64.tar.zst
tar xvf paru-v1.11.2-x86_64.tar.zst
mkdir /opt/paru
mv ./paru /opt/paru
chmod +x paru
ln -s /opt/paru/paru  /usr/bin/paru

paru -Syyu
```

## hyprland 安装

`hyprland`，一款平铺桌面管理器。动画流畅，丰富的自定义选项。最好配合 `hyprland` 官网食用。

```bash
pacman -Sy hyprland
```

安装后可以命令行执行 `Hyprland` 启动小窥一眼，默认配置比较简陋，`win+M` 退出继续折腾。 

hyprland 的配置例如动画样式、窗口规则、按键绑定、屏幕设置等不是很重要，默认配置也够用了，且个人差异太大，找个自己喜欢的 dotfiles 参考即可。比较重要的配置是环境变量

### byprland 环境变量

亲身感受下来是很多疑难杂症都可能和某个环境变量有关，只能靠多搜索多踩坑。

> 每个人环境不一样，所以也仅供参考。

```bash
# Some default env vars.
env = XDG_CURRENT_DESKTOP,Hyprland
env = XCURSOR_SIZE,24
env = QT_QPA_PLATFORMTHEME,qt6ct # change to qt6ct if you have that
# env = LIBVA_DRIVER_NAME,nvidia
env = XDG_SESSION_TYPE,wayland
# env = GBM_BACKEND,nvidia-drm
env = GDK_BACKEND,wayland
# env = __GLX_VENDOR_LIBRARY_NAME,nvidia
env = WLR_NO_HARDWARE_CURSORS,1
env = OZONE_PLATFORM,wayland
# env = GLFW_IM_MODULE,ibus
env = QT_IM_MODULE,fcitx
env = XMODIFIERS,@im=fcitx
env = LANG,zh_CN.UTF-8
env = LANGUAGE,zh_CN:en_US
env = JDK_JAVA_OPTIONS,-Dawt.useSystemAAFontSettings=on -Dswing.aatext=true -Dswing.defaultlaf=com.sun.java.swing.plaf.gtk.GTKLookAndFeel
env = _JAVA_AWT_WM_NONREPARENTING,1
env = AWT_TOOLKIT,MToolkit
env = JAVA_HOME,/opt/apps/jdk/jdk1.8.0_261
```


## 桌面配置 & 常用应用

以下软件大多都和 dotfiles 配置相关，可以在 [dotfiles](https://github.com/zonowry/dotfiles) 找到各个软件对应的配置。例如 `~/.config/fontconfig/fonts.conf` 字体配置。

### Must Have

> 中文字体是必备的，但也看个人喜好。不过字体对 waybar 等应用影响比较大，很容易出现在编辑器里很好看，但在 GUI 界面上很难看（对不齐之类的）。需要自己调整或者采用别人的挑选好的字体。

```bash
# 权限
pacman -Sy polkit-kde-agent \
dunst \
pipewire \
wireplumber \
xdg-desktop-portal-hyprland \
xdg-desktop-portal-wlr \
xdg-desktop-portal-gtk \
qt5-wayland \
qt6-wayland 
# 字体
ttf-lxgw-wenkai \
ttf-iosevka-nerd \
wqy-microhei



paru -S ttf-cascadia-code-nerd \
ttf-lxgw-wenkai-screen \
noto-fonts-emoji \
ttf-firecode-nerd \
ttf-material-design-icons-extended

fc-cache -vf # 刷新字体缓存
```

### 输入法

得益于 Hyprland 作者好像也是日语输入法用户，所以 Hyprland 的 input-method 相关协议较其它 wayland 合成器更为完善。现在 Hyprland 安装 chrome，fcitx5 之后，基本是开箱即用了，~~（只需要设置 fcitix5 环境变量与 electron flags）~~好起来了！

```bash
pacman -S fcitx5-im \
fcitx5-rime \
fcitx5-configtool \
fcitx5-material-color
```

强推下雾凇拼音 https://github.com/iDvel/rime-ice

```bash
paru -S rime-ice-git
```

### 常用软件

> 下列应用不一定适合所有人，大多都有平替，仅供参考。

```bash
pacman -Sy kitty \
wofi \
waybar \
thunar \
nfs-utils
```

```bash
paru -Sy visual-studio-code-bin \
google-chrome
```

## 美化

### 前置软件

```bash
paru -Sy kvantum \
qt5ct \
qt6ct \
nwg-look
```

以下是我采用的主题图标，若不喜欢。也可以去 [pling.com](https://pling.com) 找自己喜欢的，安装方式大同小异。
### 主题

```bash
git clone --depth=1 https://github.com/vinceliuice/Orchis-kde.git
cd ./Orchis-kde
./install.sh
```
### 图标

```bash
git clone --depth=1 https://github.com/vinceliuice/Tela-icon-theme.git
cd ./Tela-icon-theme
./install.sh -a
```

###  指针

[phinger-cursors](https://github.com/phisch/phinger-cursors)

```bash
paru -S phinger-cursors
# 打开 nwg-look 设置指针即可
```

安装好主题图标后，依次使用 `kvantum`、`nwg-look`、`qt5ct`、`qt6ct` 软件应用下主题图标即可，qt5ct/qt6ct 的主题选择 kvantum。最后重启 Hyprland 让主题生效。

## 备份

一系列繁琐操作后，得到了一个纯净的系统，当务之急是备份当前环境，避免重装。得益于 btrfs，备份十分简单。

> timeshift 只会备份名为 @、@home 的子卷，够用了。

```bash
sudo pacman -S timeshift grub-btrfs
```

> timeshift 需要 root， wofi 调起 timeshift，polkit 输入密码后，没有反应。懒得排查原因了，只能 sudo -E timeshift-gtk 启动了，问题不大。

```bash
sudo -E timeshift-gtk
```

按照操作来即可创建一个快照了。


## 小技巧

- `wev`
	- 命令行工具，查看键码
- `hyprctl clients`
	- 查看当前客户端列表信息，可以用来排查是否是 xwayland 启动。
