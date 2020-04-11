---
layout:         post
title:          将 Nexus9000 NXOS image 转换为 ACI image
subtitle:       默认 Nexus9000 预装了 NXOS image，如果是 ACI 环境，需要手动转换 image.
date:           2020-04-10
author:         黄福森
cover:          '/assets/img/bg-post-aci.png'
tags:           Cisco, ACI-Fabric-Discovery
---


# 将 Nexus9000 NXOS image 转换为 ACI image
[官方参考文档-英文](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus9000/sw/7-x/upgrade/guide/b_Cisco_Nexus_9000_Series_NX-OS_Software_Upgrade_and_Downgrade_Guide_Release_7x/Converting_from_Cisco_NX_OS_to_ACI_Boot_Mode.pdf)

> 提前确认设备是否支持 ACI image；
> 对于 N9500 机框式，是否所有板卡都支持 ACI image

**对于 RMA 硬件更换的 Nexus9000，建议转换的 ACI image 要比现网 ACI image 低一个版本，然后接入 ACI 环境，用 APIC 完成最终的 ACI image升级(否则可能有惊喜，比如光模块不能识别)**

1. 给 Nexus9000 的 mgmt0 接口配置 ACI 的管理网 IP，确认Nexus9000 mgmt0 接口和 APIC 之间可以通信(不需要接Nexus9000 业务口到 ACI). 如果不希望从 APIC copy ACI image，可以直接使用 U 盘。

2. 输入命令`feature scp` 开启 Nexus9000 的 SCP 服务

3. SSH 登录 APIC 命令行，输入命令`scp -r /firmware/fwrepos/fwrepo/switch-image-name admin@switch-ip-address:switch-image-name` 把 image 传给 Nexus9000

4. Nexus9000 输入命令 `no boot nxos`, `copy run startup` 来忽略 NXOS 启动镜像

5. Nexus9000 输入命令 `boot aci bootflash:aci-image-name`  来设置 ACI image 为重启之后的镜像源；这一步**不能**`copy run startup`

6. Nexus9000 输入命令`show file bootflash:aci-image-name md5sum` 检查 ACI image 的 MD5 值

7. Nexus9000 输入命令 `reload` 来重启，启动之后使用`admin` 登录；若无异常，可以将设备接入 ACI Fabric

**关于惊喜-FPGA/EPLD 需要单独升级**

1. 偶尔客户会发现 convert to ACI image 之后，leaf 上联口插入光模块，接口 down；但是确认在 NXOS image，接口好用

2. `show interface ex/y` 可能显示 `sfp-missing`, 接下来查看 Nexus9000 `moquery -c firmwareARunning` 如果发现类似于下面的输入，那么确认中奖
```
# firmware.CompRunning
childAction    :
descr          :
dn             : sys/ch/supslot-1/sup/fpga-1/running
expectedVer    : 0x20
internalLabel  :
modTs          : never
mode           : normal
monPolDn       : uni/fabric/monfab-default
operSt         : ver-mismatch  <<<<<<<<<<<<<<<< 期望 0x20, 实际 0x17
rn             : running
status         :
ts             : 1970-01-01T00:00:00.000+00:00
type           : controller
version        : 0x17
```

3. FPGA 是硬件可编程芯片，若 version mismatch，可能会导致接口问题。目前因为是手动 转换/升级 ACI image 的，FPGA 可能没有随着一起升级。可以按照下面方式恢复
	1. convert back to NXOS image, 手动升级 EPLD/FPGA

	2. 比如 ACI image 目标版本为 3.2.5d, 可以降级到 3.2.2o，然后用 APIC 给 leaf 升级

	3. 在 ACI image，leaf 输入命令 `/bin/check-fpga.sh FpGaDoWnGrAdE`, 然后 `/usr/sbin/chassis-power-cycle.sh` ，然后重启，看是否能解决

	4. 如果以上方式均不好用，don't worry，开 case 给 Cisco TAC，还有些其他方式可用，但是需要 root 权限

	5. 如果设备没有合同或者不想开 case 怎么办，可以试着联系我 🤔

---

# 将 ACI image 转换为 standalone NXOS image
[官方参考文档-英文](https://www.cisco.com/c/en/us/td/docs/switches/datacenter/nexus9000/sw/7-x/upgrade/guide/b_Cisco_Nexus_9000_Series_NX-OS_Software_Upgrade_and_Downgrade_Guide_Release_7x/Converting_from_Cisco_NX_OS_to_ACI_Boot_Mode.pdf)

直接查找 **Converting Back to Cisco NX-OS**

1. 一般来说，ACI image 的 Nexus9000 bootflash 中如果没有 NXOS 镜像，需要借助 U 盘，将 NXOS 镜像放入 U 盘；

2. 接上 console线，重启 ACI image 的 Nexus9000，按下 <kbd>Ctrl</kbd> + <kbd>C</kbd> 或者<kbd> Ctrl</kbd> + <kbd>]</kbd> 中断启动过程，进入 `loader>` 模式

3. 输入命令`cmdline recoverymode=1`

4. 输入命令 `dir`  来查看是否存在可用的 NXOS 启动镜像

5. 输入命令`boot nxos.7.0.3.I7.6.bin`启动 NXOS image，若从 bootflash 启动，输入命令`boot usb1:nxos.7.0.3.I7.6.bin`

6. 完成 NXOS image 启动之后，建议再断电重启一次，保证重启过程顺利进入 NXOS.

**举个栗子**

```

Converting ACI Back to Cisco NX-OS
bootflash NOT have any nxos image, need to recovery from USB disk

 ---------------------------- 
 Aborting config file read and autoboot
 ---------------------------- 
No autoboot or failed autoboot. falling to loader



                Loader Version 7.65

loader >

 ---------------------------- 
 loader > cmdline recoverymode=1
 loader > boot nxos.7.0.3.I7.6.bin 
 ---------------------------- 
Booting nxos.7.0.3.I7.6.bin
Trying diskboot
 Filesystem type is ext2fs, partition type 0x83
Boot failed

Error 9: Unknown boot failure

 ---------------------------- 
 loader > dir
 ---------------------------- 

usb1::
 nxos.7.0.3.I7.6.bin

bootflash::
  aci-n9000-dk9.13.1.2m.bin
loader >

 ---------------------------- 
 loader > boot usb1:nxos.7.0.3.I7.6.bin
 ---------------------------- 
Booting usb1:nxos.7.0.3.I7.6.bin
Trying diskboot
 Filesystem type is fat, partition type 0xc
Image valid

Saving image for img-sync ...
Found /mnt/pss/rtdbctrl.log, migrating to /mnt/pss/xlog/
Loading system software

Cisco Nexus Operating System (NX-OS) Software
TAC support: http://www.cisco.com/tac
Copyright (C) 2002-2019, Cisco and/or its affiliates.
All rights reserved.

switch(boot)#
switch(boot)#
---------------------------- 
switch(boot)# init system 

This command is going to erase your startup-config, licenses as well as the contents of your bootflash:.
Do you want to continue? (y/n)  [n] y
----------------------------

Initializing the system ...
Checking flash ...

Initialization completed.
switch(boot)#
switch(boot)#
---------------------------- 
switch(boot)# load-nxos
---------------------------- 

Unsquashing rootfs ...

---------------------------- 
after boot up with NXOS, suggest to reload again
---------------------------- 

Abort Power On Auto Provisioning [yes - continue with normal setup, skip - bypass password and basic configuration, no - continue with Power On Auto Provisioning] (yes/skip/no)[no]: yes
Disabling POAP.......Disabling POAP


         ---- System Admin Account Setup ----


Do you want to enforce secure password standard (yes/no) [y]:

  Enter the password for "admin":
  Confirm the password for "admin":
  
2019 May 23 16:59:35 switch %$ VDC-1 %$ %COPP-2-COPP_POLICY: Control-Plane is protected with policy copp-system-p-policy-strict.

User Access Verification
 login: admin
Password:
```
转载必须注明原文链接

<details>
	<summary> 微信打赏 </summary>
	<img src="https://tva1.sinaimg.cn/large/007S8ZIlgy1gdpnmfxj9ej30bh0hegnu.jpg" alt="WeChat-Money.png" />
</details>

