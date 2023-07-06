---
title: 基于 Proxmox VE 的 All in One 服务器搭建
tags:
  - PVE
  - 软路由
categories:
  - 折腾
date: 2023-06-17 19:12:49
updated: 2023-07-06 22:14:04
toc: true
thumbnail: /2023/06/17/基于-Proxmox-VE-的-All-in-One-服务器搭建/proxmox-logo.svg
---

树莓派挂在宿舍当软路由已经两年了，大部分情况下都挺好用，透明代理体验也还算尚可。

然而由于使用的 USB 无线网卡驱动支持较差，无线峰值速率只能跑到 200+ Mbps，且不大稳定。并且树莓派性能不高，无法开设一些高负载服务。另外，树莓派作为路由常年开启，需要考虑散热问题，虽然给使用的小风扇写了启停功能，但是启动运转时还是会有一定的噪音，较为恼人。

<!-- more -->

正好快到暑假了，需要开设一台 MC 服务器，于是打算换用一台较高性能的软路由，再配合 Wi-Fi 6 无线路由器作为 AP，提供高速率、高稳定性的有线无线网络接入。

## 整体配置概览

### 软路由

选用畅网奔腾 8505 软路由，1 大核 + 4 小核，大核的单核性能较高，CPU-Z 单核跑分相比我的笔记本（i5-1135G7）高出约 40%，可以用来开设一些吃单核性能的服务。

内存使用了两条光威 8GB DDR4 3200 内存，时序 CL-22-22-22-52。使用 16 GB 内存也主要是为了开设 MC 服务器考虑。

存储方面安装了西数 SN570 1T 固态作为系统盘，也暂时承担一部分文件存储功能。

作为一台路由器，这台软路由有 6 个 Intel i226-V 2.5G 网卡，即使以后有更多设备需要有线接入，也不一定需要增设交换机。

功耗方面，实测待机时输入功率约为 10 W。

温度方面，室温 24 度环境下，不加风扇待机时 CPU 约 40 度，NVME 固态约 50 度，软路由表面摸起来较热；加上赠送的 USB 12cm 风扇吹顶部铝制散热片后，待机时 CPU 约 30 度，NVME 固态约 40 度，外壳很凉快。

### 无线 AP

目前廉价的 Wi-Fi 6 路由器均使用千兆有线网口，单口有线速率甚至可能不及无线速率，故考虑需要能和软路由之间做链路聚合提高内网性能。

调查后发现，TP-Link 系列的路由器似乎原厂固件就有着端口聚合功能，其子品牌水星也有着相同的功能，且便宜几十元。

于是选用水星 X306G 路由器，AX3000 规格，计划只使用其 5G 频段 Wi-Fi 和端口聚合功能，运行在 AP 模式。

## 安装和调试 PVE

### 安装

参考官方教程或网上教程，一路 next 即可。

其中将 ETH5 对应的网卡设为管理口，静态 IP 设为 `192.168.22.100`（22 网段）。

### 更换内核版本

由于 8505 CPU 是大小核架构，建议使用较新的内核获取大小核调度优化。

先换源，参考：<https://mirrors.tuna.tsinghua.edu.cn/help/proxmox/>，并删除企业源：`rm /etc/apt/sources.list.d/pve-install-repo.list`

`apt update` 后搜索可用的内核版本，我这儿选择 6.2 版本：`apt install pve-kernel-6.2`，安装完重启后生效。

可以使用 `proxmox-boot-tool` 管理安装的以及用于启动的 kernel 版本。

### 管理页面添加温度显示

偷懒使用了恩山论坛的脚本，添加了温度，CPU频率，硬盘信息的显示：<https://www.right.com.cn/forum/thread-6754687-1-1.html>

### 删除 lvm-thin 并扩容 lvm

由于使用单盘搭建 All in One 服务器，计划在 PVE 系统中使用 samba 共享数据文件，因此需要较大的 lvm 空间。同时，由于不需要开大量的虚拟机，也用不到 lvm-thin 的特性。故将 lvm-thin 的空间全部合入 lvm 中。

参考：<https://foxi.buduanwang.vip/virtualization/pve/1434.html/>

删除 lvm-thin：`lvremove /dev/pve/data`

扩容 lvm：`lvextend -rl +100%FREE /dev/pve/root`

### 部署 samba

安装：`apt update && apt install samba`

添加用户：`useradd thx`

将用户添加到 samba：`smbpasswd -a thx`

编辑 samba 配置文件 `/etc/samba/smb.conf`，添加共享文件夹的配置：

``` conf
[RaspCloud]
    path = /data/RaspCloud
    writeable = yes
    create mask = 0777
    directory mask = 1777
    public = yes
    guest ok = yes

[Private]
    path = /data/Private
    writeable = yes
    create mask = 0744
    directory mask = 1744
    public = no
    guest ok = no
    valid users = thx
    browseable = yes
```

其中 `/data/RaspCloud` 为公开共享目录，内部有 `read-only` 目录通过改变目录所有者来限制 guest 的写入。

`/data/Private` 为私密目录，使用白名单限制访问用户。

如果需要在共享目录中软链接到系统里的其它目录中，则需要在 `[global]` 段中添加：

``` conf /etc/samba/smb.conf
[global]
   follow symlinks = yes
   wide links = yes
   unix extensions = no
```

参考：<https://blog.csdn.net/humanking7/article/details/85058471>

`smb.conf` 中的配置项可查阅：<https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html>

### 更改 CPU 电源策略

作为一个桌面服务器，需要考虑到功耗与发热的问题。

参考：<https://pve.sqlsec.com/4/6/>

``` plain
# 查看支持的 CPU 电源模式
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_available_governors

# 查看当前的 CPU 电源模式
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
```

| 电源模式     | 解释说明                                                     |
| :----------- | :----------------------------------------------------------- |
| performance  | 性能模式，将 CPU 频率固定工作在其支持的较高运行频率上，而不动态调节。 |
| userspace    | 系统将变频策略的决策权交给了用户态应用程序，较为灵活。       |
| powersave    | 省电模式，CPU 会固定工作在其支持的最低运行频率上。           |
| ondemand     | 按需快速动态调整 CPU 频率，没有负载的时候就运行在低频，有负载就高频运行。 |
| conservative | 与 ondemand 不同，平滑地调整 CPU 频率，频率的升降是渐变式的，稍微缓和一点。 |
| schedutil    | 负载变化回调机制，后面新引入的机制，通过触发 schedutil `sugov_update` 进行调频动作。 |

安装工具：`apt install cpufrequtils`

设为 `ondemand` 模式：`cpufreq-set -g ondemand`

### 无显示器启动问题

发现如果不接显示器，则无法正常启动。初步判断在启动 grub 或之后出现问题。

然而，对 BIOS 和 grub 配置文件做各种修改之后也无法解决问题。

最终打算网上买个 HDMI 诱骗器插上得了。

插上后也没用。

经仔细排查（偶然将接地良好的显示器的 Type-C 线的外壳接触到软路由 Type-C 口的外壳上，发现问题就消失了），发现是供电接地问题。

先前为了监测功耗，我将电源的 DC 转为了 USB 母口，插上了 USB 电压电流测量设备，再将测量设备的 USB 口转为 DC 口，接到软路由的 DC 供电口。在这个过程中可能丢失了地线。

直接接电源就不再有问题。

### 挂载硬盘

再接上一块老机械盘，用来挂 PT。

安装好硬盘后，先使用 `lsblk -f` 或者 `ls -l /dev/disk/by-uuid` 找到分区的 UUID，使用 UUID 进行挂载可以避免 `/dev/` 下设备名的改变导致的挂载问题。

然后在 `/etc/fstab` 中添加挂载信息：

``` plain /etc/fstab
UUID=1f4a2672-3039-594b-808c-a5d3913b0fde /data/hdd ext4 defaults 0 0
```

然后 `mount -a` 生效，开机后也会自动挂载。

### 硬盘维护

#### 缩减数据盘的文件系统预留空间

参考：<https://askubuntu.com/questions/249387/df-h-used-space-avail-free-space-is-less-than-the-total-size-of-home>

默认情况下，`df -h` 查看硬盘空间会发现，`Size` 会大于 `Used` + `Avail`，如：

``` plain
Filesystem            Size  Used Avail Use% Mounted on
/dev/sda1             294G  261G   19G  94% /data/hdd
```

这是由于 `ext2/3/4` 文件系统默认会保留 5% 的空间只供 root 使用，而数据盘就没这个必要了，可以 `tune2fs -m 0 /dev/sda1` 来取消预留空间。

#### S.M.A.R.T. 信息

在 WebUI 中，可以在节点的磁盘一栏中查看 S.M.A.R.T. 信息。

命令行中，可以使用 `smartctl -a /dev/sda` 查看。

S.M.A.R.T. 信息的解析，可以参考：<https://blog.csdn.net/MrSate/article/details/88564764>。

使用 `journalctl -u smartmontools.service` 可以查看 smartmontools 守护进程的监测日志。

#### 配置 smartd 守护程序

还可以在 `/etc/smartd.conf` 中配置守护程序，初始时配置如下：

``` conf /etc/smartd.conf
DEVICESCAN -d removable -n standby -m root -M exec /usr/share/smartmontools/smartd-runner
```

添加如下参数：

- 基于默认修改的监测与报警参数 `-H -f -l error -l selftest -C 197 -U 198`，相比默认的 `-a` 参数减少了等效的 `-t` 参数，否则每半小时都会在日志中输出 S.M.A.R.T. 信息的变化情况（详见 `/etc/smartd.conf` 中的说明）
- 每周六凌晨三点短自检，每月二十号凌晨三点长自检：`-s (S/../../6/03/|L/../20/./03)`
- 监控温度，在温度变化 5 度时记录，达到 40 度时记录，达到 45 度时警告（0 为关闭）：`-W 5,40,45`

修改后如下：

``` conf /etc/smartd.conf
DEFAULT -H -f -l error -l selftest -d removable -n standby -m root -M exec /usr/share/smartmontools/smartd-runner
/dev/nvme0 -W 0,60,70
DEVICESCAN -C 197 -U 198 -s (S/../../6/03/|L/../20/./03) -W 0,40,50
```

重启服务后生效：`systemctl restart smartd.service`

如果守护程序检测到了出现问题，也会给安装 PVE 时填写的邮箱发邮件，实测 QQ 邮箱可以收到。

参考：

- `man smartd.conf`
- <https://blog.kahosan.top/2022/06/20/%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8%20smartd%20%E7%9B%91%E6%8E%A7%E4%BD%A0%E7%9A%84%E7%A1%AC%E7%9B%98/>
- <https://forum.proxmox.com/threads/seagate-smart-prefailure-attribute.57454/>
- <https://wiki.archlinux.org/title/S.M.A.R.T.#smartd>

#### 配置硬盘休眠

可能需要先关闭 `pvestatd` 对硬盘的扫描，参考：<https://www.ippa.top/954.html>

先 `ls -l /dev/disk/by-uuid` 找到硬盘的 UUID `1f4a2672-3039-594b-808c-a5d3913b0fde`，然后编辑 `/etc/lvm/lvm.conf`，在 `global_filter` 中添加 `"r|/dev/disk/by-uuid/1f4a2672-3039-594b-808c-a5d3913b0fde.*|"`，结果如下：

``` conf /etc/lvm/lvm.conf
devices {
    global_filter=["r|/dev/zd.*|", "r|/dev/disk/by-uuid/1f4a2672-3039-594b-808c-a5d3913b0fde.*|"]
}
```

`pvestatd restart` 后生效。

查看硬盘当前状态：`smartctl -i -n standby /dev/sda | grep "mode"|awk '{print $4}'`，ACTIVE 或者 IDLE 都为运转状态。

查询当前电源管理参数：`hdparm -B /dev/sda`

进入 `standby mode`：`hdparm -y /dev/sda`

进入 `sleep mode`：`hdparm -Y /dev/sda`

然后编辑 `/etc/hdparm.conf`，添加需要休眠的硬盘设置，比如十分钟后 standby 如下配置：

``` conf /etc/hdparm.conf
/dev/disk/by-uuid/1f4a2672-3039-594b-808c-a5d3913b0fde {
        apm = 127
        acoustic_management = 127
        spindown_time = 120
}
```

`/usr/lib/pm-utils/power.d/95hdparm-apm resume` 后生效（参考 `man hdparm.conf`）

其中 `spindown_time` 参考 `man hdparm` 中的 `-S` 选项说明来设置。

不过我的这块硬盘似乎并不按照设定的 `spindown_time` 来休眠，于是打算观察一段时间 `Load_Cycle_Count` 的增长情况来决定是否开启休眠功能。

`smartctl -a /dev/sda | grep Load_Cycle_Count` 即可查看。

## 基于 LXC 安装 OpenWrt

LXC 开销小，故尝试使用 LXC 安装 OpenWrt

### 安装 OpenWrt

安装过程参考：<https://virtualizeeverything.com/2022/05/23/setting-openwrt-in-proxmox-lxc/>

下载地址：<https://downloads.openwrt.org/>

我选择了 20.03.5 版本：<https://downloads.openwrt.org/releases/22.03.5/targets/x86/64/>

下载 `rootfs.tar.xz` 即可，可以在下载时选择哈希校验。

创建容器的 UI 的文档：<https://pve.proxmox.com/pve-docs/chapter-pct.html#pct_settings>

似乎由于 Web 页面无法添加 `--ostype unmanaged` 参数，需要进入命令行创建：

``` plain
pct create 101 /var/lib/vz/template/cache/openwrt-22.03.5-x86-64-rootfs.tar.gz --arch amd64 --hostname RaspCloud --rootfs local:1 --memory 1024 --cores 6 --cpuunits 200 --ostype unmanaged --unprivileged 1
```

然后在 Web UI 中添加一个虚拟网卡 veth0，桥接到 PVE 网桥 vmbr0 上，用于 OpenWrt 和 PVE 的交互。

此时连接在管理口上的设备便和 OpenWrt 连在同一个网桥上了，就可以通过这个虚拟网卡的地址访问 OpenWrt 了。

有可能启动后 veth0 并没被正确配置，可以手动 up 之后使用 IPv6 地址访问。

然后到配置文件 `/etc/pve/lxc/101.conf` 里添加直通网卡，比如：

``` conf
lxc.net.0.type: phys
lxc.net.0.link: enp2s0    --- 需要使用的 host 中的网卡名
lxc.net.0.name: eth0      --- LXC 中显示的网卡名
lxc.net.0.flags: up
```

需要注意 net.id 的 id 不能和 Web UI 中添加的相同。

最终的 conf 文件：

``` conf
arch: amd64
cores: 6
cpuunits: 200
hostname: RaspCloud
memory: 1024
net0: name=veth0,bridge=vmbr0,firewall=1,hwaddr=92:81:93:06:8A:7E,type=veth
onboot: 1
ostype: unmanaged
rootfs: local:101/vm-101-disk-1.raw,size=1G
startup: order=1
swap: 512
unprivileged: 1
lxc.net.5.type: phys
lxc.net.5.link: enp2s0
lxc.net.5.name: eth0
lxc.net.5.flags: up
lxc.net.6.type: phys
lxc.net.6.link: enp3s0
lxc.net.6.name: eth1
lxc.net.6.flags: up
lxc.net.7.type: phys
lxc.net.7.link: enp4s0
lxc.net.7.name: eth2
lxc.net.7.flags: up
lxc.net.8.type: phys
lxc.net.8.link: enp5s0
lxc.net.8.name: eth3
lxc.net.8.flags: up
lxc.net.9.type: phys
lxc.net.9.link: enp6s0
lxc.net.9.name: eth4
lxc.net.9.flags: up
```

### 配置 OpenWrt

#### 连接互联网

由于我需要 OpenWrt 通过 PVE 管理口连接电脑，通过电脑将无线网共享给有线网来访问互联网，所以需要先为 veth0 配置一些上网功能。

添加 Interface veth0，类型为静态地址，网卡设为 veth0，配置好对应的地址，网关和 DNS 服务器地址即可。

之后应该就可以访问网络了。

#### 换源

参考：<https://mirrors.tuna.tsinghua.edu.cn/help/openwrt/>

#### 添加中文

``` bash
opkg update
opkg install luci-i18n-base-zh-cn
```

#### 更改主题

不大喜欢 OpenWrt 的默认主题，换为 [Argon](https://github.com/jerrykuku/luci-theme-argon/)

下载对应的 ipk：`https://github.com/jerrykuku/luci-theme-argon/releases/download/v2.3.1/luci-theme-argon_2.3.1_all.ipk`

然后安装：

``` bash
opkg install luci-compat
opkg install luci-lib-ipkg
opkg install luci-theme-argon*.ipk
```

#### 配置路由功能

启动后，只有 wan 和 wan6 两个 Interface，对应 eth0 网口（device）。

计划把其它网口都划入 LAN 中。

首先添加网桥设备 `br-lan`，把网口都加上去。

然后添加 `lan` 接口，使用静态地址，把 `br-lan` 加上去，配置地址即可。

然后发现，似乎 dnsmasq 出现了些问题，无法正常启动。

根据 <https://github.com/openwrt/openwrt/issues/9064> 中的方法，`opkg remove procd-ujail` 可以解决。

dnsmasq 正常后，在接口中配置 DHCP / DNS 的相关配置，然后 lan 口上接的设备就能获取到分配的地址了。

校园网只会分配 /128 地址，故 IPv6 也需要做个 NAT。OpenWrt 中 IPv6 默认并不会配置 SNAT，需要做一些手动配置，参考：<https://openwrt.org/docs/guide-user/network/ipv6/ipv6.nat6>

先启用 IPv6 masquerading，其中我这儿的 `zone[1]` 为 wan 域：

``` bash
uci set firewall.@zone[1].masq6="1"
uci commit firewall
/etc/init.d/firewall restart
```

然后关闭 wan6 口的 sourcefilter：

``` bash
uci set network.wan6.sourcefilter="0"
uci commit network
/etc/init.d/network restart
```

这样 lan 口上接的设备也能获取到 DHCPv6 分配的 IPv6 地址了，OpenWrt 也会为其提供 NAT6 服务。

#### 配置 OpenClash

先卸载 dnsmasq：

``` bash
opkg remove dnsmasq
mv /etc/config/dhcp /etc/config/dhcp.bak
```

再按照发布页的说明，先安装依赖，再下载安装 ipk 即可：<https://github.com/vernesong/OpenClash/releases>

之后按个人喜好配置即可，我选用的是 Meta 核心的 redir-host 模式。

可以将校园网网段设为绕过，减少校内网络访问开销：

IPv4：

``` plain
59.66.0.0/16
101.5.0.0/16
101.6.0.0/16
118.229.0.0/19
166.111.0.0/16
183.172.0.0/15
202.112.39.2/32
219.223.168.0/21
219.223.176.0/20
```

IPv6：

``` plain
2402:f000::/32
```

或者可以打开 `实验性：绕过中国大陆 IP` 功能，减少国内网络访问开销。

遇上 OpenClash 的一个 bug。（**Updated 2023.07.02：** 随着 `0.45.128` 版本的发布，该问题已得到修复）

在 LuCI 中，虽然已经关闭了 `路由本机代理`，但是还是会 nftables 中添加处理 Output 链的规则，见 <https://github.com/vernesong/OpenClash/blob/9ee0f02ed7615a62f960c9ee2f951dd1b47e2411/luci-app-openclash/root/etc/init.d/openclash#LL1649C1-L1672C9>：

``` bash
if [ "$enable_redirect_dns" != "2" ] || [ "$router_self_proxy" = "1" ]; then
    nft 'add chain inet fw4 openclash_output' 2>/dev/null
    nft 'flush chain inet fw4 openclash_output' 2>/dev/null
    ...
    nft add rule inet fw4 openclash_output ip protocol tcp skuid != 65534 counter redirect to "$proxy_port" 2>/dev/null
    nft 'add chain inet fw4 nat_output { type nat hook output priority -1; }' 2>/dev/null
    nft 'add rule inet fw4 nat_output ip protocol tcp counter jump openclash_output' 2>/dev/null
fi
```

这里的判断中 **或** 上了不使用 Dnsmasq 转发，所以在我的工况下会被启用。暂时未明白为何要这样，故提了 Issue：<https://github.com/vernesong/OpenClash/issues/3354>。

两天后作者在 [d499374](https://github.com/vernesong/OpenClash/commit/d49937415c00c6d3f2519a382cd13be54d531e8b) 中修复了该 bug，发布于 `0.45.125` 版本中，等待合入 master。

#### 配置校园网认证

使用 [GoAuthing](https://github.com/z4yx/GoAuthing)。

新版本的 OpenWrt 配置起来似乎有些不一样的地方，参考 <https://github.com/z4yx/GoAuthing/issues/30> 进行修改。

将 `goauthing@` 脚本放到 `/etc/init.d/goauthing` 目录下，并 `chmod +x`。

``` shell /etc/init.d/goauthing
#!/bin/sh /etc/rc.common
# Authenticating utility for auth.tsinghua.edu.cn
# This init script is used explicitly with OpenWRT

USE_PROCD=1
START=96
PROG="/usr/bin/goauthing"
SERV=goauthing  # UCI config at /etc/config/goauthing

start_instance() {
  local config=$1
  local username password
  config_get username $config username
  config_get password $config password
  local args="-u $username -p $password"

  sleep 10 # Wait for link up
  "$PROG" $args deauth
  "$PROG" $args auth
  "$PROG" $args login

  procd_open_instance
  procd_set_param command "$PROG"
  procd_append_param command $args online
  procd_set_param stderr 1
  procd_set_param respawn
  procd_close_instance
}

logout() {
  local config=$1
  local username password
  config_get username $config username
  config_get password $config password
  local args="-u $username -p $password"

  "$PROG" $args logout
}

start_service() {
  config_load "$SERV"
  config_foreach start_instance "$SERV"
}

stop_service() {
  config_load "$SERV"
  config_foreach logout "$SERV"
}
```

其中更改了启动优先级，添加了 `sleep 10` 来等待 wan 口配置完成，否则会因为无法完成 auth / login 就直接进入 online 守护程序，从而一直无法完成登录认证。

然后下载 <https://mirrors.tuna.tsinghua.edu.cn/github-release/z4yx/GoAuthing/LatestRelease/auth-thu.linux.x86_64>，并移动至脚本中填写的位置（默认为 `/usr/bin/goauthing`）。

然后配置并启动服务：

``` bash
touch /etc/config/goauthing
uci add goauthing goauthing
uci set goauthing.@goauthing[0].username='<YOUR-TUNET-ACCOUNT-NAME>'
uci set goauthing.@goauthing[0].password='<YOUR-TUNET-PASSWORD>'
uci commit

/etc/init.d/goauthing enable
/etc/init.d/goauthing start
```

#### 配置链路聚合 AP

另购置了一台 Wi-Fi 6 无线路由器作为 AP，支持 2x2 MU-MIMO，160 MHz 频宽，理论带宽可达 2402 Mbps。

然而，路由器上只有四个千兆口，连接软路由后跑不到这么高。

不过，这款路由器支持两个端口链路聚合，故尝试配置一下。

首先在路由器上设置两个端口为聚合口，提示：

> 端口聚合使用IEEE 802.3ad动态聚合模式，请确保对端设备支持并配置为动态聚合模式。

然后在 OpenWrt 上安装需要的软件包：`opkg install kmod-bonding proto-bonding luci-proto-bonding`

然后将以下内容添加到 `/etc/rc.local` 的 `exit 0` 前：

``` bash
ip link add bond-lan type bond mode 802.3ad # 添加 bond 类型的虚拟接口 名称为 bond-lan
ip link set eth3 down
ip link set eth4 down
ip link set eth3 type bond_slave            # 配置网卡模式
ip link set eth4 type bond_slave
ip link set eth3 master bond-lan            # 加入名称为 bond-lan 的 bond 类型网卡
ip link set eth4 master bond-lan
ip link set bond-lan up                     # 启动该网卡
ip link set eth3 up
ip link set eth4 up
```

将 `eth3` 和 `eth4` 从原来的 `br-lan` 中移除，添加上 `bond-lan` 即可。

实测无线可以跑到 1.6 Gbps 左右。

#### 配置防火墙

官方原版 OpenWrt 默认有一套防火墙策略，简略微调即可。

#### 配置端口转发

两种方式，一种是基于 iptables 实现，另一种是使用 socat 来转发。

##### **基于 iptables 实现**

OpenWrt 默认的端口转发基于 iptables / nftables 实现，然而，配置后发现，在内网无法使用外网地址访问对应端口，初步探索后推测是 NAT 环回时出现问题。

于是在 nftables 中进行调试。

先新建一个表为符合规则的包启用跟踪调试：

``` bash
nft add table inet trace_debug
nft add chain inet trace_debug trace_pre { type filter hook prerouting priority -200000; }
nft insert rule inet trace_debug trace_pre ip saddr 192.168.22.118 ip daddr ??.??.??.?? tcp limit rate 1/second meta nftrace set 1
```

然后 `nft monitor trace` 就可以跟踪了。

跟踪检查后发现，包在 `prerouting policy accept` 后消失了。

一番摸索后发现，当对应网卡（`br-lan`）开启混杂模式后，就能正常工作了。

怀疑是 prerouting 后发现目标地址为本地链路地址，于是就修改了目的 mac 地址，导致非混杂模式的网卡将其丢弃。不过简单搜索后也没找到相关的资料。

##### **使用 socat**

**Updated 2023.06.29：** 使用 socat 遇到了一些问题，故弃用：

- 会修改源地址，丢失地址数据
- UDP 的转发使用 fork 参数存在线程泄漏问题，似乎每个 UDP 包都会 fork 出一个进程且不释放，导致产生大量进程占满内存

安装：

``` bash
opkg update
opkg install socat
```

然后配置端口转发。

例如，配置名为 `mc-tcp` 的策略，开启，监听本机所有地址的 25565 端口并转发到 192.168.22.3:25565 的配置如下（`fork` 允许多个连接，`reuseaddr` 允许 socket 的快速重用，`TCP6-LISTEN` 也同时监听 IPv4）：

``` bash
uci set socat.mc-tcp=socat
uci set socat.mc-tcp.enable=1
uci set socat.mc-tcp.SocatOptions='TCP6-LISTEN:25565,fork,reuseaddr TCP:192.168.22.3:25565'
uci commit
```

UDP 也类似：

``` bash
uci set socat.mc-udp=socat
uci set socat.mc-udp.enable=1
uci set socat.mc-udp.SocatOptions='UDP6-LISTEN:25565,fork,reuseaddr UDP:192.168.22.3:25565'
uci commit
```

然后重启 socat 服务即可生效：`/etc/init.d/socat restart`

对外网开放还需在防火墙中允许对应端口的输入。

#### 配置 DDNS

我的 DNS 解析提供商为 CloudFlare，故安装 `ddns-scripts-cloudflare`。

```bash
opkg install ddns-scripts-cloudflare luci-i18n-ddns-zh-cn
```

安装完成后在 LuCI 中配置即可，没太大难度。IPv4 和 IPv6 地址需要分别配置。

#### 配置 AdGuard Home

安装 AdGuard Home：

``` bash
opkg update
opkg install adguardhome
```

在 DHCP/DNS 的高级设置中，将 DNS 服务器端口改为 53 以外的端口，如 5353。

然后登入 <http://ip:3000> 配置 AdGuard Home，将上游 DNS 服务器设为 OpenClash 设置的 DNS 服务地址，并停用 OpenClash 的 DNS 劫持。

然后应该就可以开始运作了，再添加屏蔽列表即可。

如果需要在 OpenClash 没有正常启动成功的情况下仍可以进行 DNS 服务，则需要设置 fallback DNS 服务器地址。然而目前 AdGuard Home 中并没有这个功能，相关功能 <https://github.com/AdguardTeam/AdGuardHome/issues/3701> 被设为了 `v0.108.0` 的目标，希望有生之年能等到 `v0.108.0` 出来获得这一功能。

（也是因为这个原因，校园网的认证脚本需要配置为 OpenClash 启动完成后再进行认证，感觉怪怪的，所以也暂时放弃了 AdGuard Home）

#### 配置 WireGuard

安装 `WireGuard`：

``` bash
opkg update
opkg install luci-i18n-wireguard-zh-cn
```

重启后，在添加接口中就可以找到 WireGuard VPN 了，我添加了一个名为 `wg0` 的接口。

作为公网 IP 下供其它端来主动连接的一端，我特殊指定了这个接口的监听端口，并在防火墙中运行了该端口的 UDP 访问。

该接口的 IP 地址需要配置为单独的子网，用于各端 VPN 接口间的互相连接。我为其配置了 `192.168.23.1/24` 和 `fd23:41b7:e060::1/64` 地址，与 22 网段区别开。

为了方便配置，我没有将这个接口划入单独的区域，而是划入了 lan 区域，共享 lan 区域的防火墙配置。

然后就可以添加对端了。注意 `允许的 IP` 的这一项中填写 “对端的隧道 IP 地址和对端经由隧道的网络”，对于对端为非路由设备的情况，这一项只填隧道 IP 地址就行，比如 `192.168.23.102/32`（/32 可省略），但不能填写 VPN 接口间的网段，即不能填写 `192.168.23.102/24`。

各项设置可能需要重启后生效。

#### 之后的计划

- 配置内网 IP 的域名
- 配置 MosDNS 优化 DNS 解析（参考：<https://rushb.pro/article/router-dns.html>）
- 配置 Grafana 可视化路由运行状态、MosDNS 运行数据等
- 配置 UDP 转发以及游戏优化

## 基于 LXC 的其它功能服务器

其它杂七杂八的服务以及 Docker 就另外开在一个虚拟机上吧。

选用 Debian 12 系统，可以直接从 CT 模板中下载。

参考 <https://pve.proxmox.com/wiki/Unprivileged_LXC_containers#Using_local_directory_bind_mount_points>，挂载宿主机的共享目录：`pct set 100 -mp0 /host/dir,mp=/container/mount/point`

添加一个虚拟网卡 `eth0` 桥接到 vmbr0 上，IPv4 选择 DHCP 接收 OpenWrt 的地址分发；而如果 IPv6 选择 DHCP 的话，DHCPv6 是不会通告默认路由的，所以建议选择 SLAAC。

### 初始配置

换源，参考：<https://mirrors.tuna.tsinghua.edu.cn/help/debian/>

添加 sudo，先 `apt install sudo`，再 `echo "username  ALL=(ALL) ALL" | sudo tee /etc/sudoers.d/username`

默认下终端可能会有乱码，需要配置 UTF-8 语言，`sudo dpkg-reconfigure locales` 然后选中 `en_US.UTF-8` 即可。

### 安装 Docker

由于并不想在 PVE 中直接装 Docker，故在 Debian 虚拟机中安装 Docker。

参考：

- <https://docs.docker.com/engine/install/debian/#install-using-the-convenience-script>
- <https://yeasy.gitbook.io/docker_practice/install/debian>

安装：

``` bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

加入 Docker 用户组，在无 root 权限下使用 Docker：`sudo usermod -aG docker $USER`

验证安装正确性：`docker run --rm hello-world`

如没有方便的网络接入，配置镜像参考：<https://yeasy.gitbook.io/docker_practice/install/mirror>

在 LXC 容器启动后，Docker 会过一两分钟才会启动，暂时不知道原因为何。

### Docker 启用 IPv6 支持

由于挂的 PT 需要有 IPv6 接入，故需要给 Docker 开启 IPv6。

参考：<https://docs.docker.com/config/daemon/ipv6/>

编辑配置文件：

``` json /etc/docker/daemon.json
{
  "experimental": true,
  "ip6tables": true
}
```

然后重启 Docker：`sudo systemctl restart docker`

启动容器时，需要额外的配置。如果使用 Docker Compose，则添加如下内容，并在对应 service 配置中添加 networks 即可：

``` yml
networks:
  ip6net:
    enable_ipv6: true
    subnet: 2001:0DB8::/112
```

`netstat -tunlp` 可以查看监听端口，若对应的端口只有 tcp6 在监听也不用慌张，若 `cat /proc/sys/net/ipv6/bindv6only` 为 0 则表明已在双栈上监听，参考（<https://unix.stackexchange.com/questions/496137/does-80-in-netstat-output-means-only-ipv6-or-ipv6ipv4>）

### 配置 PT 客户端

使用 [linuxserver/transmission](https://hub.docker.com/r/linuxserver/transmission) Docker 镜像。

使用 Docker Compose 来配置容器，按说明编写配置文件：

``` yml docker-compose.yml
---
version: "2.1"
services:
  transmission:
    image: lscr.io/linuxserver/transmission:latest
    container_name: transmission
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
      - TRANSMISSION_WEB_HOME=/config/transmission-web-control/src #optional
      - USER=thx #optional
      - FILE__PASS=/config/password #optional
      - WHITELIST= #optional
      - PEERPORT= #optional
      - HOST_WHITELIST= #optional
    volumes:
      - /home/thx/Service/transmission/config:/config
      - /data/RaspCloud/read-only/PT:/downloads
      - /data/RaspCloud/read-only/PT/torrentwatch:/watch
    ports:
      - 9091:9091
      - 51413:51413
      - 51413:51413/udp
    restart: unless-stopped
    networks:
      - ip6net
networks:
  ip6net:
    enable_ipv6: true
    ipam:
      config:
        - subnet: 2001:0DB8::/112
```

其中 UID / GID 可以参考 `id $user` 的结果设置。

生成密钥文件时不能有行末符，可以这样生成：`echo -n password_in_clear_text > password`

Web UI 使用 [transmission-web-control](https://github.com/ronggang/transmission-web-control)，在 `/config` 对应的目录下 git clone 即可。

创建 container 并启动后台运行：`docker compose up -d`

停止并删除 container 和对应的网络：`docker compose down`

启动 / 停止 对应的 container：`docker compose start` / `docker compose stop`

迁移只需要将之前的 `config` 目录移过来即可，如果时裸机安装，目录可能在 `/var/lib/transmission-daemon/info`

### 配置蓝牙监听服务

之前 [使用树莓派和小米蓝牙温湿度计可视化宿舍温湿度变化](/2022/09/07/使用树莓派和小米蓝牙温湿度计可视化宿舍温湿度变化) 中配置了蓝牙接收温湿度计数据，也把这个服务迁移过来。

花费十元购入了 BR8651 芯片的 USB 蓝牙 5.1 适配器，据说该芯片在 Linux 下有驱动。

可能是因为芯片较新的原因，各方面的支持似乎都还不太好，尝试了几个方法都没能正常地使用脚本获取 BLE Advertising，这里记录了几次失败的过程。

#### 配置 LXC 的 USB 直通

本来想在 LXC 容器中配置蓝牙服务，但是 `hciconfig` 会报错 `Can't open HCI socket.: Address family not supported by protocol`，查阅资料后发现，由于蓝牙将自身注册为网络接口，所以并不能像使用 USB 那样将蓝牙设备传给 LXC 容器，参考：<https://forum.proxmox.com/threads/assign-a-bluetooth-dongle-to-a-ct.67577/>。

所以以下部分只是记录如何直通 USB 设备。

参考：<https://medium.com/@konpat/usb-passthrough-to-an-lxc-proxmox-15482674f11d>

直接在 LXC 中 `lsusb` 是可以看到各个 USB 设备的，但是无法使用。

``` plain
Bus 004 Device 002: ID 174c:3074 ASMedia Technology Inc. ASM1074 SuperSpeed hub
Bus 004 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 003 Device 003: ID 05e3:0751 Genesys Logic, Inc. microSD Card Reader
Bus 003 Device 004: ID 0a12:0001 Cambridge Silicon Radio, Ltd Bluetooth Dongle (HCI mode)
Bus 003 Device 002: ID 174c:2074 ASMedia Technology Inc. ASM1074 High-Speed hub
Bus 003 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
Bus 002 Device 001: ID 1d6b:0003 Linux Foundation 3.0 root hub
Bus 001 Device 001: ID 1d6b:0002 Linux Foundation 2.0 root hub
```

关注其中的 Bluetooth 设备，位于 `Bus 003 Device 004`。

查看其主次设备号：

``` plain
root@pve:/etc/pve/lxc# ls -al /dev/bus/usb/003/004
crw-rw-r-- 1 root root 189, 259 Jul  1 00:28 /dev/bus/usb/003/004
```

主设备号为 189，向配置文件中添加以下内容，将设备映射到容器内：

``` conf /etc/pve/lxc/102.conf
lxc.cgroup.devices.allow: c 189:* rwm
lxc.mount.entry: /dev/bus/usb/003/004 dev/bus/usb/003/004 none bind,optional,create=file
```

或者直接将目录映射过去也行，防止设备名发生变化：

``` conf /etc/pve/lxc/102.conf
lxc.mount.entry: /dev/bus/usb/003 dev/bus/usb/003 none bind,optional,create=dir
```

如果容器中 `ls -al /dev/bus/usb/003/004` 权限不对（`nobody` / `nogroup`），可以在 PVE 中 `chown 100000:100000 /dev/bus/usb/003/004`，这样容器中就为 `root` 权限了。

#### 在宿主机中直接配置蓝牙

由于不想再单独开一台虚拟机，故打算在宿主机中直接运行蓝牙监听服务。

正好宿主机中也有一个非 root 用户，使用这个用户来运行服务，尽量减小对系统的影响。

先验证蓝牙功能是否正常，在 `bluetoothctl` 中运行

``` bash
menu scan
transport le
back
scan on
```

应该能够看到一些广播和数据。

（我这儿 `hcitool lescan` 会报错 `Set scan parameters failed: Input/output error`，参考 <https://stackoverflow.com/questions/70777475/hcitool-lescan-returns-an-i-o-error-on-manjaro> 发现如上使用 `bluetoothctl` 就能正常工作）

##### 非特权安装 pip3

``` bash
wget https://bootstrap.pypa.io/get-pip.py
python3 get-pip.py --user
```

如果报错 `ModuleNotFoundError: No module named 'distutils.cmd'`，则需要安装 `python3-distutils`

##### 安装运行 MiTemperature2

按照 [MiTemperature2](https://github.com/JsBergbau/MiTemperature2) 文档进行安装。

安装 `bluepy` 前需要先安装 `libglib2.0-dev`

安装 `pybluez` 时可能遇到 `error in PyBluez setup command: use_2to3 is invalid.` 的问题，参考 <https://github.com/pybluez/pybluez/issues/467>，先 `pip3 install setuptools==58` 再安装即可。

之后遇到类似 <https://github.com/JsBergbau/MiTemperature2/issues/106> 的问题，以及类似 <https://stackoverflow.com/questions/75175755/not-seeing-ble-device-advertising-unless-set-bluetoothctl-transport-le> 的问题，都暂时没有被解决。

于是也放弃了。

### 配置 MC 服务器

#### 安装 Java 8

由于该整合包版本需要 Java 8，而 Debian 官方源中没有，故使用第三方源安装。

准备工作：`sudo apt install apt-transport-https ca-certificates wget dirmngr gnupg software-properties-common`

添加第三方源：

``` bash
wget -qO - https://adoptopenjdk.jfrog.io/adoptopenjdk/api/gpg/key/public | sudo tee /etc/apt/trusted.gpg.d/adoptopenjdk.asc
sudo add-apt-repository --yes https://adoptopenjdk.jfrog.io/adoptopenjdk/deb/
```

由于 adoptopenjdk 可能还没加上 bookworm 源，可能需要手动将源中的 `bookworm` 改为 `bullseye`。

安装 Java 8 JRE：

``` bash
sudo apt update
sudo apt install adoptopenjdk-8-hotspot-jre
```

#### 添加为服务
