---
title: 透明网关之重生之我是 iptables 大师
date: 2023-01-02
tags: [linux,网络,iptables,clash]
categories: 知识
toc: true
author: zonowry
description: "尝试用 clash tun 模式来实现网关，虽然过程很流畅也比较“新潮“，但对于我来说有点魔法了，因为比较难搞清楚 clash 帮我们做了哪些工作，出现问题不好找原因。也可能是我比较“洁癖” ，所以我采用了 `iptables + tproxy` 这种更加“简单“的方式，clash 只作为流量中继，流量的路由都依靠 l`inux 内核` 的 `netfilter` 功能实现，这样搭建的网关会更加“可控”一点。"
---

> **本文尚处于填坑中**
> 
> 本文非 clash 网关搭建教程，而是借*富强* 网关的例子来学习了解 linux 网络的一些知识。不过也许能给正在搭建透明网关的你一些启发。
>
> 本文提到的旁路由、网关，若无特别说明指代的是同一台机器。


## 前言

尝试用 `clash tun` 模式来实现过网关，虽然过程很流畅也比较“新潮“，但对于我来说有点魔法了，因为比较难搞清楚 `clash` 帮我们做了哪些工作，出现问题不好找原因。也可能是我比较“洁癖” ，所以我采用了 `iptables + tproxy` 这种更加“简单“的方式，`clash` 只作为流量中继，流量包的路由都依靠 linux 内核的 `netfilter` 模块实现，这样搭建的网关会更加“可控”一点。

然后我看了不少 `clash + linux netfilter(iptables/nftables) 搭建“富强”网关` 的教程文章。步骤都是很简单的，照着做就能实现。但每个人总会有点特殊需求，不去理解这些步骤的奥秘，很难解决一些特殊问题。

我就是遇到了公网上无法访问我网关上的 `docker` 服务，debug 排查了好久，虽然最后凭感觉解决了。但一直没有理顺流量是怎么路由的，只是稍有眉目、模棱两可。所以我去尝试理解了过程中每个操作（命令）的底层逻辑，现在写篇文章梳理一下这些知识。

***

## linux 网络之 netfilter

首先说说这一切的基石：linux 的 `netfilter` 模块及延伸工具 `iptables`。

iptables 只是个命令行工具，依赖 netfilter 内核模块，也即真正实现防火墙功能的是 linux 内核的 netfilter 模块。不仅 iptables 的命令宛若天书，netfilter 的链路也错综复杂，很难去使用。想要理解使用这些工具或命令，必须得先了解一些 `netfilter` 与 `iptables` 的基础知识。

###  iptables 的链

`netfilter` 提供了 **5 个 hook** 点，`iptables` 根据这些 **hook** 点，搞出了 `链 (chain)` 的概念，也就**内置**了 **5 个默认链**。可以看出 5 个 `iptables chian` 和 5 个 `netfilter hook` 一一对应。当然，我们可以添加自定义链，不过想要某个自定义链生效，需要追加一条从**内置链**跳转到这个自定义链的规则。因为内核的 5 个 hook 点只会触发这 5 个内置链。

| netfilter hook | iptables chain | netfilter hook 解释 |
| --- |---| --- |
| NF_IP_PRE_ROUTING | PREROUTING | 接收到的包进入协议栈后立即触发此 hook，在进行任何路由判断 （将包发往哪里）之前|
| NF_IP_LOCAL_IN| INPUT | 接收到的包经过路由判断，如果目的是本机，将触发此 hook
|NF_IP_FORWARD | FORWARD | 接收到的包经过路由判断，如果目的是其他机器，将触发此 hook
| NF_IP_LOCAL_OUT | OUTPUT  |  本机产生的准备发送的包，在进入协议栈后立即触发此 hook
| NF_IP_POST_ROUTING | POSTROUTING |  本机产生的准备发送的包或者转发的包，在经过路由判断之后， 将触发此 hook


### iptables 的表与动作

`iptables` 为了更颗粒度的管理流量，又设计出 `table` 的概念。用 `table` 来组织这些链，可以理解为每个 `table` 根据其用处包含了不同的链。每个 `table` 都支持一些“**动作**“。例如 `nat` 表的 `DNAT` 动作支持重写目标地址。不过有些动作只在特定的 `chain`（或者说 `hook`）上才有意义。例如向 `INPUT` 链添加 `DNAT` 动作时，内核会抛出这个错误：`ip_tables: DNAT target: used from hooks INPUT, but only usable from PREROUTING/OUTPUT`。另一个例子是 `mangle` 表不允许添加 `SNAT` 等动作，所以一个**动作**需要 `table` + `chain` 都允许才能被添加。

| 表  | 支持的内置链 | 支持的动作（本人知道（用过）的，仅供参考） |
| --- | --- | --- |
| mangle  |  支持全部 5 个内置链 | `RETURN` `TPROXY` |
| raw | `PREROUTING`  `OUTPUT` | `TRACE` | 
| nat | `PREROUTING` `INPUT` `OUTPUT` `POSTROUTING` | `SNAT` `DNAT` `REDIRECT` `MASQURADE`| 
| filter | `INPUT` `FORWARD`  `OUTPUT` | 略 |
| security | 略 | 略 | 

每个 `table` 的 `chain` 当然也是有触发顺序的，具体顺序可以参考那张著名的 `netfilter 流程图` ，或[这篇文章](https://arthurchiao.art/blog/deep-dive-into-iptables-and-netfilter-arch-zh/)的介绍 。

<div style="background: #fff">
<img src="https://arthurchiao.art/assets/img/deep-dive-into-iptables-netfilter/Netfilter-packet-flow.svg" title="netfilter 流程图"/>
</div>
*** 

## 流量方向 与 iptables 规则

### 开启内核转发功能

要想把一台 linux 机器配置成有路由转发功能的机器，第一步需要用以下命令开启内核转发功能。

```bash
sysctl -w net.ipv4.ip_forward=1
```

单单这条命令只是将 linux 机器做成中继路由，一般情况下没太大意义。我们还需要处理途径机器的流量。即设定规则将途径流量“路由（转发）”到本机某些程序上（常用如 `clash` 或者 `v2ray` ），经代理中转后再原路返回。达成“加速网络”的目的。 `iptables` 等相关工具就登场了。

### 局域网流量跳过处理，直连主路由

linux 系统是可以作为**主路由**的，但一般的机器没有多个网口，所以都是作为**旁路由**来辅助主路由。既然作为旁路由来使用，我们只想代理加速公网流量，局域网内机器的流量肯定还是希望通过**主路由**来直连，没必要再来来回回途径一次旁路由了。所以我们就会添加以下规则，让旁路由跳过局域网内流量，原封不动转出去，让主路由继续去路由。

```bash
# clash 链负责处理转发流量
iptables -t mangle -N clash

# 目标地址为局域网或保留地址的流量跳过处理
iptables -t mangle -A clash -d 0.0.0.0/8 -j RETURN
iptables -t mangle -A clash -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A clash -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A clash -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A clash -d 192.168.0.0/16 -j RETURN
iptables -t mangle -A clash -d 169.254.0.0/16 -j RETURN
iptables -t mangle -A clash -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A clash -d 240.0.0.0/4 -j RETURN
  

# 最后让所有流量通过 clash 链进行处理
iptables -t mangle -A PREROUTING -j clash
```

可以结合 [linux 网络之 netfilter](#linux\ 网络之\ netfilter) 段落内容理解这些命令。我们先逆向思维来推导一下：“*我们要添加一些**路由规则**，这些规则最终肯定是注入到 **netfilter hook** 里的。我们使用 **iptables chain** 来注入，所以规则要添加到一个**合适**的 **chain** 里，**iptables** 是通过 **table** 来组织管理 **chain** 的。我们还需要找一个**合适**的 **table** 来添加 **chain**（或者说规则）*”。思考了这些后，我们再回头看命令。

首先 `iptables -t mangle -N clash`、`iptables -t mangle -A PREROUTING -j clash` 这两行是因为不推荐直接向内置链（这里是 `PREROUTING` ）添加规则，所以我们新建一个自定义链管理规则，然后从内置链 `PREROUTING` 跳转而来，这样更加规范一点。直译就是创建一个名为 `clash` 的链，选的表是 `mangle` 表，追加到内置链 `PREROUTING` 中。

>为什么是 **mangle** 表？
>
>仅目前这些规则来看，放到 **nat** 表似乎更合适。考虑到下一节我们要介绍 **TPROXY** 动作，放到一个链里看起来会更直观一点。

总而言之，最终实现了局域网机器流量发到**旁路由**时，旁路由发现目标地址是局域网内ip，跳过处理，转发出去给到主路由，之后就是主路由和源主机通信了。

### 中转外网流量，clash 透明代理

搞定了内部（局域网内）流量，接下来处理**外部**（互联网）流量。这些“外部流量”应该要转发到 `clash` 中进行透明代理。

> 如下所示，记得在 `clash` 配置中开启 `tproxy` 功能
> ```yaml
> tproxy-port: 7893
> ```

上一步我们跳过了（局域网内）流量，其余的流量**一般、基本**就是外部（互联网）流量了。通过以下命令，我们指定 `tcp` ，`udp` 类型的流量转到 `tproxy` 。

```bash
# 其他所有流量转向到 7893 端口，并打上 mark
iptables -t mangle -A clash -p tcp -j TPROXY --on-port 7893 --tproxy-mark 666
iptables -t mangle -A clash -p udp -j TPROXY --on-port 7893 --tproxy-mark 666
```


***

## 引用

赞美开放的互联网，感谢大佬们。

* [第一篇万字长文：围绕透明代理的又一次探究](https://moecm.com/something-about-v2ray-with-tproxy/)
* [「译」深入理解 iptables 和 netfilter 架构](https://arthurchiao.art/blog/deep-dive-into-iptables-and-netfilter-arch-zh/)
* [树莓派 Clash 透明代理(TProxy)_](https://mritd.com/2022/02/06/clash-tproxy/)
* [tpclash wiki - 2、进阶流量控制](https://github.com/mritd/tpclash/wiki/2%E3%80%81%E8%BF%9B%E9%98%B6%E6%B5%81%E9%87%8F%E6%8E%A7%E5%88%B6) 。
* [iptables的四表五链与NAT工作原理 _](https://tinychen.com/20200414-iptables-principle-introduction/)

