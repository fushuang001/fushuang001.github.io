---
layout:         post
title:          Linux 网卡 mode4 LACP 模式 failover 测试
subtitle:		    双网卡可以通过合理的设置和 failover 测试，来保障业务的高可用性。
date:           2021-07-18
author:         Forest
cover:          '/assets/img/post-Linux_bond_LACP-bg.png'
tags:           Linux, LACP, 802.3ad, Nexus VPC
---

# 基础信息介绍

客户使用思科 Nexus9000 交换机 VPC，对接 Linux 双网卡 server，Linux 网卡使用 bond mode=4, LACP/802.3ad 模式。

简易拓扑如下：

![](/assets/img/post-Linux_bond_LACP-topo.png)

## Nexus9000 VPC

思科的 VPC 是一个 layer2/二层概念，是将两台 nexus 交换机捆绑为逻辑上的一台设备，对外提供 port-channel 服务，此架构里面没有 STP block 接口。


    show vpc                                        # vpc 状态
    show port-channel summary                       # port-channel 状态
    show lacp counters                              # LACP PDU 收发 counter
    show lacp interface ethernet 1/1                # 接口 LACP 信息
    show lacp neighbor interface port-channel 1     # 对端 LACP 信息，比如 SA 表示 `slow_active`
    show lacp internal info interface ethernet 1/1 detail fsmlog    # `fsmlog` 会包含更多 event-history 的 log


## Linux 网卡 bond

通过 bond 将双网卡设置为不同模式，如主备、LACP 等，提供高可用性；


    cat /etc/sysconfig/network/ifcfg-bond0         # summary 信息
    cat /proc/net/bonding/bond0                    # slave 网卡接口信息

# 链路 failover 测试

期望状态下，当 LACP port-channel 中某一条 member link 断开，另一条 member link 会用来承载流量。在 member link failover 期间的丢包数量应该是**毫秒级**。

使用 client 长 ping Server bond0 ip address, 测试分为下面 3 个方式：

1. 直接拔掉 n9k 或者 server 端某一条物理链路

2. n9k 命令行 `shutdown` 接口

3. Linux server `ifdown eth<x>` 去 disableslave 接口

其中方式 1 和 2，流量切换在毫秒级完成，不丢包；

方式 3，若 `ifdown eth<active_slave>` 网卡，会观察到 90 秒的丢包；若 `ifdown eth<non-active_slave>` 网卡，没有发现丢包。

# Troubleshooting 过程

90 秒的流量中断时间，和默认 LACP(又叫做**Slow**) timeout 时间吻合：Slow LACP hello 30 seconds, timeout 90 seconds.

推测在`ifdown eth<active_slave>` 网卡之后，__Linux server__ 并没有真正断开这条 link；对端的 __n9k交换机__ 焦急等待 90 秒 LACP PDU timeout，发现 partner 一去不回头，才从逻辑上断开这条 link。那么在 90 秒时间之内，这条 link 就是流量黑洞，黑洞吞噬一切。

为了验证推测，需要找到 Linux server 监测链路状态的机制。既然拔线和 ifdown 有不同结果，大概是这个机制有某些 limitation。

Google 了一些帖子，发现 Linux 可以依赖 `ARP` or `MII` 去监测链路是否存活，故障切换基本在 100 毫秒级。

这里有一个 KB 关于[Troubleshooting bond interfaces on RHEL/CentOS 7.5 ](https://kb.juniper.net/InfoCenter/index?page=content&id=KB36182&cat=CONTRAIL&actp=LIST)，可以用来热身。

[Linux Ethernet Bonding Driver HOWTO](https://www.kernel.org/doc/Documentation/networking/bonding.txt) 这大概就是传说中的 Linux 源码了，有争议的地方不要紧，查源码。

作为一个合格的搬运工，highlight 一些信息：


      Link monitoring can be enabled via either the miimon or
      arp_interval parameters (described in the module parameters section,
      above).  In general, miimon monitors the carrier state as sensed by
      the underlying network device, and the arp monitor (arp_interval)
      monitors connectivity to another host on the local network.

      At the present time, due to implementation restrictions in the
      bonding driver itself, it is not possible to enable both ARP and MII
      monitoring simultaneously.

      In general, however, in a multiple switch topology, the ARP
      monitor can provide a higher level of reliability in detecting end to
      end connectivity failures (which may be caused by the failure of any
      individual component to pass traffic for any reason).

      Finally, the 802.3ad mode mandates the use of the MII monitor,
      therefore, the ARP monitor is not available in this mode.


简单来说(毕竟复杂的我也不懂😂)，Linux server bond 在使用了 `mode 4 LACP/802.3ad` 模式下，只能依赖 `MII` 来监测 slave 网卡是否存活。

而测试过程里面可以看到 `watch cat /proc/net/bonding/bond0`的变化过程：

1. 拔线以后，对应的 slave 网卡 MII: down

2. `ifdown`， 对应的 slave 网卡从这个文件被 **remove** 了，这真的是人走茶凉一把好手了，于是系统里面看不到 bond0 的被 ifdown 的 eth<x> MII 信息，也就不会通知上层去 shutdown 这个端口了。这就是后续 __n9k交换机__ 苦苦等待 90 秒的原因。


      [root@localhost ~]# cat /proc/net/bonding/bond0
      Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

      Bonding Mode: IEEE 802.3ad Dynamic link aggregation
      Transmit Hash Policy: layer2 (0)
      MII Status: up
      MII Polling Interval (ms): 100        # 监测间隔
      Up Delay (ms): 0
      Down Delay (ms): 0

      802.3ad info
      LACP rate: slow                      # LACP 模式，分为 slow(30s/90s) 和 fast(1s/3s)                    
      Min links: 0

      Slave Interface: ens1f0
      MII Status: up
      Link Failure Count: 0

      Slave Interface: ens1f1
      MII Status: up
      Link Failure Count: 0

# 解决方法

首先 `ifdown` 不是一个合理的 failover 测试工具，推荐使用`ifenslave -d bond 0 eth0` 方式，更多的讨论细节可以参考[Redhat How to test nic bonding?](https://access.redhat.com/discussions/669983)，搬运工 highlight：

      We've always found that using 'ifdown' to simulate a network failure is not a good practice.
      The best option obviously is to physically remove it or have the network team disable the port.
      If that's not possible, use "ifenslave" to detach an interface from a bond.
      For example, if bond0=eth0,eth1 and eth0 is active, use "ifenslave -d bond0 eth0".

如果客户还是喜欢 `ifdown`，那么就让 __n9k交换机__ 来帮帮忙，配置 `lacp rate fast`，即要求 Linux server 每 1s 发送一次 LACP PDU，若 3s 收不到就认为 link failure，弃用之。

    int e1/1
      lacp rate fast
      Cannot set lacp rate. Port is not admin down in port-channel
      shutdown
      lacp rate fast
      exit
      end
