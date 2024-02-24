---
tags:
  - article
  - blog
  - iptables
  - network
name: "build a proxy gateway based on iptables and clash"
title: "iptables + clash 透明网关实践与总结"
descriptions: "一种纯净、科学的网络路由编排手段（"
creation date: 2023-03-01

description: "一种纯净、科学的网络路由编排手段（"
date: 2023-03-01
toc: true
isCJKLanguage: true
keywords:
  - zonowry
  - iptables + clash 透明网关实践与总结
  - clash
  - iptables
  - gateway
---

# 前言

尝试用 `clash tun` 模式来实现过网关，虽然过程很流畅也比较“新潮“，但对于我来说有点魔法了，因为比较难搞清楚 `clash` 帮我们做了哪些工作，出现问题不好找原因。也可能是我比较“洁癖” ，所以我采用了 `iptables + tproxy` 这种更加“简单“的方式，`clash` 只作为流量中继，流量包的路由都依靠 linux 内核的 `netfilter` 模块实现，这样搭建的网关会更加“可控”一点。

然后我看了不少 `clash + linux netfilter(iptables/nftables) 搭建“富强”网关` 的教程文章。步骤都是很简单的，照着做就能实现。但每个人总会有点特殊需求，不去理解这些步骤的奥秘，很难解决一些特殊问题。

我就是遇到了公网上无法访问我网关上的 `docker` 服务，debug 排查了好久，虽然最后凭感觉解决了。但一直没有理顺流量是怎么路由的，只是稍有眉目、模棱两可。所以我去尝试理解了过程中每个操作（命令）的底层逻辑，现在写篇文章梳理一下这些知识。

# linux 网络之 netfilter

首先说说这一切的基石：linux 的 `netfilter` 模块及延伸工具 `iptables`。

`iptables` 只是个命令行工具，依赖 `netfilter` 内核模块，也即真正实现防火墙功能的是 linux 内核的 `netfilter` 模块。可惜不仅 `iptables` 的命令宛若天书，`netfilter` 的链路也错综复杂，很难去使用。想要理解使用这些工具或命令，必须得先了解一些 `netfilter` 与 `iptables` 的基础知识。

## iptables 的链

`netfilter` 提供了 **5 个 hook** 点，`iptables` 根据这些 **hook** 点，搞出了 `链 (chain)` 的概念，也就**内置**了 **5 个默认链**。可以看出 5 个 `iptables chian` 和 5 个 `netfilter hook` 一一对应。当然，我们可以添加自定义链，不过想要某个自定义链生效，需要追加一条从**内置链**跳转到这个自定义链的规则。因为内核的 5 个 hook 点只会触发这 5 个内置链。

| netfilter hook     | iptables chain | netfilter hook 解释                                                            |
| ------------------ | -------------- | ------------------------------------------------------------------------------ |
| NF_IP_PRE_ROUTING  | PREROUTING     | 接收到的包进入协议栈后立即触发此 hook，在进行任何路由判断 （将包发往哪里）之前 |
| NF_IP_LOCAL_IN     | INPUT          | 接收到的包经过路由判断，如果目的是本机，将触发此 hook                          |
| NF_IP_FORWARD      | FORWARD        | 接收到的包经过路由判断，如果目的是其他机器，将触发此 hook                      |
| NF_IP_LOCAL_OUT    | OUTPUT         |   本机产生的准备发送的包，在进入协议栈后立即触发此 hook                        |
| NF_IP_POST_ROUTING | POSTROUTING    | 本机产生的准备发送的包或者转发的包，在经过路由判断之后， 将触发此 hook         |

### iptables 的表与动作

`iptables` 为了更颗粒度的管理流量，又设计出 `table` 的概念。用 `table` 来组织这些链，可以理解为每个 `table` 根据其用处包含了不同的链。每个 `table` 都支持一些“**动作**“。例如 `nat` 表的 `DNAT` 动作支持重写目标地址。不过有些动作只在特定的 `chain`（或者说 `hook`）上才有意义。例如向 `INPUT` 链添加 `DNAT` 动作时，内核会抛出这个错误：`ip_tables: DNAT target: used from hooks INPUT, but only usable from PREROUTING/OUTPUT`。另一个例子是 `mangle` 表不允许添加 `SNAT` 等动作，所以一个**动作**需要 `table` + `chain` 都允许才能被添加。

| 表       | 支持的内置链                                | 支持的动作（部分，仅供参考）         |
| -------- | :------------------------------------------ | :----------------------------------- |
| mangle   | 支持全部 5 个内置链                         | `RETURN` `TPROXY`                    |
| raw      | `PREROUTING` `OUTPUT`                       | `TRACE`                              |
| nat      | `PREROUTING` `INPUT` `OUTPUT` `POSTROUTING` | `SNAT` `DNAT` `REDIRECT` `MASQURADE` |
| filter   | `INPUT` `FORWARD` `OUTPUT`                  | 略                                   |
| security | 略                                          | 略                                   |

每个 `table` 的 `chain` 当然也是有触发顺序的，具体顺序可以参考那张著名的 `netfilter 流程图` ，或[这篇文章](https://arthurchiao.art/blog/deep-dive-into-iptables-and-netfilter-arch-zh/)的介绍 。

<div style="background: #fff">
<img src="https://arthurchiao.art/assets/img/deep-dive-into-iptables-netfilter/Netfilter-packet-flow.svg" title="netfilter 流程图"/>
</div>

## 流量方向 与 iptables 规则

### 开启内核转发功能

要想把一台 linux 机器配置成有路由转发功能的机器，第一步需要用以下命令开启内核转发功能。

```bash
sysctl -w net.ipv4.ip_forward=1
```

单单这条命令只是将 linux 机器做成中继路由，一般情况下没太大意义。我们还需要处理途径机器的流量。即设定规则将途径流量“路由（转发）”到本机某些程序上（常用如 `clash` 或者 `v2ray` ），经代理中转后再原路返回。达成“加速网络”的目的。 `iptables` 等相关工具就登场了。

### 局域网流量跳过处理，直连主路由

linux 系统是可以作为**主路由**的，但一般的机器没有多个网口，所以都是作为**旁路由**来辅助主路由。既然作为旁路由来使用，我们只想代理加速公网流量，局域网内机器的流量肯定还是希望通过**主路由**来直连，没必要再来来回回途径一次旁路由了。所以需要添加一些**转发规则**，让旁路由跳过局域网内流量，原封不动转出去，让主路由继续去路由。

结合 netfilter 段落的知识，逆向思考一下要怎么做。首先我们要添加一些**路由规则**，这些规则最终肯定是注入到 netfilter hook 里的，可以通过 **iptables chain** 操作 netfikter hook。所以规则要添加到一个**合适**的 **chain** 里，**iptables** 又是通过 **table** 来组织管理 **chain** 的。我们还需要找一个**合适**的 **table** 来添加 **chain**（或者说规则）。思考了这些后，我们再回头看命令：

```bash
# clash 链负责处理转发流量
iptables -t mangle -N clash

# 让所有流量通过 clash 链进行处理
iptables -t mangle -A PREROUTING -j clash

# 目标地址为局域网或保留地址的流量跳过处理
iptables -t mangle -A clash -d 0.0.0.0/8 -j RETURN
iptables -t mangle -A clash -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A clash -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A clash -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A clash -d 192.168.0.0/16 -j RETURN
iptables -t mangle -A clash -d 169.254.0.0/16 -j RETURN
iptables -t mangle -A clash -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A clash -d 240.0.0.0/4 -j RETURN
```

- 首先我们新建了一个自定义链管理规则：`iptables -t mangle -N clash`
- 然后从内置链 `PREROUTING` 跳转而来：`iptables -t mangle -A PREROUTING -j clash`
  - 当然我们可以直接不写这两句，直接将规则添加到 `PREROUTING` 链。但那样写不是很规范，不推荐直接向内置链（这里是 `PREROUTING` ）添加规则。
- 然后追加局域网 IP 直连规则到 `clash` 表中
  我们使用的表是 mangle 表，链是 链。
  总而言之，最终实现了局域网机器流量发到**旁路由**时，旁路由发现目标地址是局域网内 ip，跳过处理，转发出去给到主路由，就是主路由和源主机直接通信了，之后的网络传输本网关就不会参与了。

### 中转外网流量，clash 透明代理

由于上一步我们跳过了内部（局域网内）流量，剩下的流量基本就是外部（互联网）流量了。这些**外部流量**应该要转发到 `clash` 中进行透明代理。

虽然可以简单的通过 `REDIRECT` 动作将流量转发到 `7893` 端口。但 `REDIRECT` 不能很好的支持 `UDP` 流量。所以采用 `TPROXY` 方式，这样 `TCP` 和 `UDP` 都能支持。

```bash
# tproxy 7893（clash） 端口，并打上 mark 666 命中策略，走 666 路由表
iptables -t mangle -A clash -p tcp -j TPROXY --on-port 7893 --tproxy-mark 666
iptables -t mangle -A clash -p udp -j TPROXY --on-port 7893 --tproxy-mark 666

# 转发所有 DNS 查询到 1053 端口
# 此操作会导致所有 DNS 请求全部返回虚假 IP(fake ip 198.18.0.1/16)
iptables -t nat -I PREROUTING -p udp --dport 53 -j REDIRECT --to 1053

# 添加策略与路由表（）
ip rule add fwmark 666 lookup 666
ip route add local 0.0.0.0/0 dev lo table 666
```

前两句 `iptables` 命令，追加了两条 `TPROXY` 规则。将 `tcp` & `udp` 流量转发到 `clash` 的 `7893` 端口，且打了 `666` 标记。

因为 `TPROXY` 不会修改 IP 数据包，数据包的 dest ip 一般都是外网地址，所以数据包下一跳会直接 forward 转出到下一跳机器上。因此 `TPROXY` 大部分情况都需要搭配 `ip route` 策略路由一起使用。比如我们这里就是新建了一个名为 `666` 的路由表，此路由表会将所有数据包发到本地回环上。这样就阻断了 forward 过程，相当于让（ `tproxy` 过的）数据包重新走一边网络栈流程。这样数据包就可以转发到 `7893` 端口上了，然后我们只让有 `666` 标记的数据包经过此路由表。

### 代理网关本机的流量

经过以上步骤，局域网内的其它机器已可以正常使用本网关了。当然，一台 llinux 机器只用来当一个网关太浪费了，还可以跑各种服务以及日常使用。顺便将本机的流量也代理一下，也即代理本机发出（经过 `OUTPUT` 链）的数据包。

首先与上一步类似的步骤，将本机发出的流量（OUTPUT）打上标记，触发重新路由。这样本机发出的流量就和局域网内其它机器进入的流量相同了，路由的流程也就一样了。不过 `OUTPUT` 上的数据包也会包含 clash 发出流量，这样会出现数据包死循环，得处理一下。只需要跳过 clash 程序发出的数据包，避免死循环。用 clash 用户启动 clash 程序，根据 uid 跳过数据包即可。。

```bash
# clash_local 链负责处理网关本身发出的流量
iptables -t mangle -N clash_local

# nerdctl 容器流量重新路由
#iptables -t mangle -A clash_local -i nerdctl2 -p udp -j MARK --set-mark 666
#iptables -t mangle -A clash_local -i nerdctl2 -p tcp -j MARK --set-mark 666

# 跳过内网流量
iptables -t mangle -A clash_local -d 0.0.0.0/8 -j RETURN
iptables -t mangle -A clash_local -d 127.0.0.0/8 -j RETURN
iptables -t mangle -A clash_local -d 10.0.0.0/8 -j RETURN
iptables -t mangle -A clash_local -d 172.16.0.0/12 -j RETURN
iptables -t mangle -A clash_local -d 192.168.0.0/16 -j RETURN
iptables -t mangle -A clash_local -d 169.254.0.0/16 -j RETURN
iptables -t mangle -A clash_local -d 224.0.0.0/4 -j RETURN
iptables -t mangle -A clash_local -d 240.0.0.0/4 -j RETURN

# 为本机发出的流量打 mark
iptables -t mangle -A clash_local -p tcp -j MARK --set-mark 666
iptables -t mangle -A clash_local -p udp -j MARdocK --set-mark 666

# 跳过 clash 程序本身发出的流量, 防止死循环(clash 程序需要使用 "clash" 用户启动)
iptables -t mangle -A OUTPUT -p tcp -m owner --uid-owner clash -j RETURN
iptables -t mangle -A OUTPUT -p udp -m owner --uid-owner clash -j RETURN

# 让本机发出的流量跳转到 clash_local
# clash_local 链会为本机流量打 mark, 打过 mark 的流量会重新回到 PREROUTING 上
iptables -t mangle -A OUTPUT -j clash_local
```

## 外网访问内网 docker 问题

也可以说外网访问局域网内机器（非网关机器）的问题。我们这样配置好后，会发现无法从外网访问内网的 docker 服务（设置路由器端口转发）。可以通过手机流量访问测试。

我是参考该 [github issue](https://github.com/Dreamacro/clash/issues/432#issuecomment-571634905) 受到了启发，最终解决了。

```bash
# 跳过 docker0 的 ip 范围。即跳过 docker 服务的出站数据包
sudo iptables -t mangle -A clash -p tcp -s 172.18.0.0/16 -j RETURN
```

然后以下是个人的推测，可能有误，仅供参考。

首先手机入站数据包经过路由器，`NAT` 到 `docker` 服务（网关机器）上。此时因为 **dest ip** 是内网 ip，**clash 链** 会跳过。**DOCKER 链** 接手处理，通过 `DNAT` 转发到了 **docker0 bridge** 网卡上，这几步都很正常。顺利到达 docker 容器。

随后是 docker 容器的出站数据包，此时数据包会从 **docker0 bridge** 发到宿主机的物理网卡 **eth** 网卡。这时数据包之于宿主机来说，是一个入站数据包。数据包会经过 `PREROUTING` 链，jump 到 **clash** 链，而此时的 **dest ip** 为手机的 ip 。会被转发到 clash 上处理，但这个数据包只在出站时转发给 clash 处理。入站的时候跳过了。估计 clash 无法处理这个数据包，可能就丢弃了。就出现了外网无法访问内网 docker 容器的问题。

所以根据 source ip 判断， 将 docker 容器的数据包也跳过。跳过后就解决了～

## 参考

- [第一篇万字长文：围绕透明代理的又一次探究](https://moecm.com/something-about-v2ray-with-tproxy/)
- [「译」深入理解 iptables 和 netfilter 架构](https://arthurchiao.art/blog/deep-dive-into-iptables-and-netfilter-arch-zh/)
- [树莓派 Clash 透明代理(TProxy)\_](https://mritd.com/2022/02/06/clash-tproxy/)
- [tpclash wiki - 2、进阶流量控制](https://github.com/mritd/tpclash/wiki/2%E3%80%81%E8%BF%9B%E9%98%B6%E6%B5%81%E9%87%8F%E6%8E%A7%E5%88%B6) 。
- [iptables 的四表五链与 NAT 工作原理  \_](https://tinychen.com/20200414-iptables-principle-introduction/)
- https://www.zhaohuabing.com/learning-linux/docs/tproxy/
