---
tags:
  - Handbook
name: Steps to flash Openwrt to Redmi AX6S router 
title: 红米 AX6S 路由器刷机 OpenWRT 步骤
description: 如题，只是一个简单的操作手册

date: 2024-03-31
toc: true
isCJKLanguage: true
keywords:
  - zonowry
  - AX6S
  - OpenWRT
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

## 通过 scp 上传固件

以下用到的 Openwrt 的固件可以在这里下载：[ax6s-22.zip - 蓝奏云](https://sssddddff.lanzouo.com/ibIns0me47mf)。

开启了 ssh 后，就可以 scp 上传固件了。把 openwrt 的底包传上去。

```bash
scp path/to/file/factory.bin root@192.168.31.1:/tmp
```

然后 ssh 登录路由器，刷写 openwrt 固件。

```bash
# 刷入上一步 scp 传过来的底包
mtd -r write /tmp/factory.bin firmware
```

## 升级 openwrt

openwrt 已经安装完成，浏览器访问 `192.168.6.1`，用户名：`root`，密码：`password`。
> PS：确保自己电脑的 ip 是自动获取，否则由于网段可能访问不到 192.168.6.1。

然后升级固件，大佬可能针对 AX6S 优化了一下，并捆绑了一些常用插件，升级固件也在上一步下载的压缩包里 `ax6s-full.bin` 或者 `ax6s-mini.bin` 。


## Openwrt 拨号

最后配置下 wan 口的 PPPoE 信息，连接互联网~

打完收工，Openwrt router! Getto Daze!


## 参考

- [\[2-2\]AX6S 闭源无线驱动 Openwrt 刷机教程/固件下载-小米无线路由器及小米网络设备-恩山无线论坛](https://www.right.com.cn/forum/thread-8187405-1-1.html)