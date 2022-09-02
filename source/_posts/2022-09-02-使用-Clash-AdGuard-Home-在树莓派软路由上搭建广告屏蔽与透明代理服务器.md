---
title: 使用 Clash + AdGuard Home 在树莓派软路由上搭建广告屏蔽与透明代理服务器
tags:
  - 树莓派
  - 软路由
  - Clash
  - AdGuard Home
categories:
  - 折腾
  - 树莓派
date: 2022-09-02 15:24:44
updated: 2022-09-02 15:24:44
toc: true
thumbnail: /2022/09/02/使用-Clash-AdGuard-Home-在树莓派软路由上搭建广告屏蔽与透明代理服务器/raspberry-pi-foundation-vector-logo.svg
---

之前 [【补档】树莓派折腾记录](/2021/07/11/【补档】树莓派折腾记录/#使用-USB-网卡) 中也记录过了相关内容，但是一年过来有些地方有些变动与改进，故单开一篇重新记录并长期更新。

<!-- more -->

## 配置 Clash 进行透明代理

TODO

## 配置 AdGuard Home 进行广告屏蔽

TODO

## 防火墙

作为一个挂在公网下 7×24h 运行的网关服务器，进行一定的防火墙配置是必不可少的。这里主要通过 iptables 和 ip6tables 实现。

由于使用了 iptables-persistent 进行 iptables 规则可持久化，方便起见下面就直接把保存的规则文件贴上来了。

### IPv4

```bash /etc/iptables/rules.v4
*filter
# INPUT 和 FORWARD 链上默认 DROP 掉
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]

# 防止外网使用内网 IP 欺骗
-A INPUT -i eth0 -s 192.168.0.0/16 -j DROP

# 允许本机、内网以及已建立的连接通过和转发
-A INPUT -i lo -j ACCEPT
-A INPUT -i docker0 -j ACCEPT
-A INPUT -i wlx1cbfceb110dc -j ACCEPT
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
-A FORWARD -i lo -j ACCEPT
-A FORWARD -i docker0 -j ACCEPT
-A FORWARD -i wlx1cbfceb110dc -j ACCEPT
-A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# 开放外网 SSH, HTTP, HTTPS 连接
-A INPUT -p tcp -m multiport --dport 22,80,443 -j ACCEPT

COMMIT


*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
# 新建 clash 链
:clash - [0:0]

# 将转发后的包源地址修改为本机地址
-A PREROUTING -s 192.168.0.0/16 -p tcp -j clash

# 内网 TCP 请求转发给 clash 链
-A POSTROUTING -s 192.168.0.0/16 -o eth0 -j MASQUERADE

# 访问本机和内网不经过 clash
-A clash -d 10.0.0.0/8 -j RETURN
-A clash -d 127.0.0.0/8 -j RETURN
-A clash -d 169.254.0.0/16 -j RETURN
-A clash -d 172.16.0.0/12 -j RETURN
-A clash -d 192.168.0.0/16 -j RETURN
-A clash -d 224.0.0.0/4 -j RETURN
-A clash -d 240.0.0.0/4 -j RETURN

# 访问校园网不经过 clash
-A clash -d 59.66.0.0/16 -j RETURN
-A clash -d 101.5.0.0/16 -j RETURN
-A clash -d 101.6.0.0/16 -j RETURN
-A clash -d 118.229.0.0/19 -j RETURN
-A clash -d 166.111.0.0/16 -j RETURN
-A clash -d 183.172.0.0/15 -j RETURN
-A clash -d 202.112.39.2/32 -j RETURN
-A clash -d 219.223.168.0/21 -j RETURN
-A clash -d 219.223.176.0/20 -j RETURN

# 其余请求重定向至 clash 端口
-A clash -p tcp -j REDIRECT --to-ports 7891

COMMIT
```

### IPv6

参考：<https://www.sixxs.net/wiki/IPv6_Firewalling>

关于为何 DHCPv6 相比 DHCPv4 要额外设置，参考：<https://unix.stackexchange.com/questions/452880/what-are-the-essential-iptables-rules-for-ipv6-to-work-properly>

[RFC 4890](https://www.rfc-editor.org/rfc/rfc4890) 给出了针对 ICMPv6 的防火墙配置建议，由于时间有限未能细读与实现。

[RFC 5095](https://www.rfc-editor.org/rfc/rfc5095) 废除了 Type 0 Routing Headers，防火墙中给予了实现。

```bash /etc/iptables/rules.v6
*filter
:INPUT DROP [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]

# Filter all packets that have RH0 headers. Refer to RFC 5095
-A INPUT -m rt --rt-type 0 -j DROP
-A FORWARD -m rt --rt-type 0 -j DROP
-A OUTPUT -m rt --rt-type 0 -j DROP

# Allow trusted link to INPUT and FORWARD
-A INPUT -i lo -j ACCEPT
-A INPUT -i docker0 -j ACCEPT
-A INPUT -i wlx1cbfceb110dc -j ACCEPT
-A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
-A FORWARD -i lo -j ACCEPT
-A FORWARD -i docker0 -j ACCEPT
-A FORWARD -i wlx1cbfceb110dc -j ACCEPT
-A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow DHCPv6
-A INPUT -p udp --dport 546 -d fe80::/10 -j ACCEPT

# Allow ICMPv6
-A INPUT -p icmpv6 -j ACCEPT

# Allow some server port
-A INPUT -p tcp -m multiport --dport 22,80,443 -j ACCEPT

COMMIT


*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]

# SNAT
-A POSTROUTING -s fd22:41b7:e060::/64 -o eth0 -j MASQUERADE

COMMIT
```
