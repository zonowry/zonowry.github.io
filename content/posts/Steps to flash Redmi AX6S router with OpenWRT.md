---
tags:
  - Handbook
name: Steps to flash Openwrt to Redmi AX6S router 
title: 红米 AX6S 路由器刷入纯净最新 ImmortalWrt 步骤
description: 如题，只是一个简单的操作手册

date: 2024-03-31
lastmod: 2024-04-01
toc: true
isCJKLanguage: true
keywords:
  - zonowry
  - AX6S
  - OpenWRT
  - ImmortalWrt
  - Router
  - Openclash
---



## 启用路由器的 telnet

为了启用 telnet，需要手动升级路由器固件到开发版，开发版固件可以在 [GitHub - YangWang92/AX6S-unlock](https://github.com/YangWang92/AX6S-unlock) 仓库下载。([miwifi\_rb03\_firmware\_stable\_1.2.7.bin](https://github.com/YangWang92/AX6S-unlock/blob/master/miwifi_rb03_firmware_stable_1.2.7.bin))

## 通过 SN 码获取路由器 root 密码

在网页 [Xiaomi Router Developer Guide & Tools](https://miwifi.dev/ssh) 内输入路由器 SN 码，计算出 root 密码。

## Telnet 登录路由器开启 ssh 等服务

登录 telnet，用户名 root，密码你刚刚拿到了。

```bash
telnet 192.168.31.1
```

执行以下三条命令，开启些服务，主要是 ssh，照做就是了。

```bash
nvram set ssh_en=1 && nvram set uart_en=1 && nvram set boot_wait=on && nvram set bootdelay=3 && nvram set flag_try_sys1_failed=0 && nvram set flag_try_sys2_failed=1

nvram set flag_boot_rootfs=0 && nvram set "boot_fw1=run boot_rd_img;bootm"

nvram set flag_boot_success=1 && nvram commit && /etc/init.d/dropbear enable && /etc/init.d/dropbear start
```

## 下载纯净（原版）的 ImmortalWrt 固件

[ImmortalWrt Firmware Selector](https://firmware-selector.immortalwrt.org/)

搜索自己的路由器型号，下载 **factory.bin**。~~如果没有话只能自己编译了~~

## 通过 scp 上传固件

scp 上传刚刚下载的原版固件。

```bash
scp path/to/file/factory.bin root@192.168.31.1:/tmp
```

然后 ssh 登录路由器，刷写 openwrt 固件。

```bash
# 刷入上一步 scp 传过来的底包
mtd -r write /tmp/factory.bin firmware
```

## 访问 openwrt 配置 lan 接口

原版的 openwrt 网址是 `192.168.1.1`。这个 IP 地址可能会和光猫冲突，所以首先编辑 `网络/接口/lan`，修改网段为你喜欢的，例如 `10.0.0.1`。

## 配置路由器拨号

配置好 lan 接口后，再配置下 wan 口的 PPPoE 信息，确保路由器可以联网~

## 安装 argon 管理界面主题

仓库地址： [GitHub - jerrykuku/luci-theme-argon](https://github.com/jerrykuku/luci-theme-argon)。

安装的版本：[2.3.1](https://github.com/jerrykuku/luci-theme-argon/releases/download/v2.3.1/luci-theme-argon_2.3.1_all.ipk) 

```bash
opkg install luci-compat
opkg install luci-lib-ipkg
#  scp path\to\file\luci-theme-argon_2.3.1_all.ipk root@10.0.0.1:/tmp
#  安装 scp 传递下载好的 ipk
opkg install /tmp/luci-theme-argon*.ipk
```

## 安装 openclash

仓库地址： [GitHub - OpenClash](https://github.com/vernesong/OpenClash)。

安装的版本：[v0.46.003-beta](https://github.com/vernesong/OpenClash/releases/tag/v0.46.003-beta)

```bash
# 同理先使用 scp 上传到路由器，然后 opkg 安装
opkg install /tmp/luci-app-openclash_0.46.003-beta_all.ipk
```

安装后重启路由器，就可以看到“服务”菜单了。按照自己需求配置 `openclash` ，接入**互联网**。

![preview](/images/blog/image-2024_04_01_20_17_39.png)



## 为什么不装整合包

整合包挺方便的，不过内核都比较老旧，直接更新还容易变砖。而且大多服务/插件都没什么用。

这样是获得了一个最新版本的 openwrt 内核，不会再碰见内核版本不兼容问题了，可以放心的安装各种新版插件了。

最主要的是它很**干净**哇！


## 参考

- [\[2-2\]AX6S 闭源无线驱动 Openwrt 刷机教程/固件下载-小米无线路由器及小米网络设备-恩山无线论坛](https://www.right.com.cn/forum/thread-8187405-1-1.html)
- [360T7安装immortalwrt官方原版固件（含Luci Web页面），供新手参考-360无线路由器及其他360网络设备-恩山无线论坛](https://www.right.com.cn/forum/thread-8290496-1-1.html)