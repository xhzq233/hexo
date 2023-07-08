---
title: linux远程启动
date: 2023-01-18 12:58:52
tags:
- linux
- wol
- Wake On Lan
---

# Wake on Lan on Linux

<!--more-->

## linux上

>  reference https://help.ubuntu.com/community/WakeOnLan

ifconfig看网卡

```
enp3s0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.0.233  netmask 255.255.255.0  broadcast 192.168.0.255
        ether 1c:1b:0d:ae:0e:66  txqueuelen 1000  (Ethernet)
```

找到所在局域网的网卡名称和MAC

`sudo ethtool <ifname>`

注意这两行

```
	Supports Wake-on: ***
	Wake-on: g
```

如果显示d就说明被禁用了，使用`sudo ethtool -s enp3s0 wol g`来启用。

### Enable on every booting

不过这每次shutdown后WOL功能都会关闭，需要每次boot都操作一遍很麻烦，这里创建一个system service来enable WOL。

* `vim /etc/systemd/system/wol@.service`

```ini
[Unit]
Description=Wake-on-LAN (%i)
Requires=network.target
After=network.target

[Service]
ExecStart=/sbin/ethtool -s %i wol g
Type=oneshot

[Install]
WantedBy=multi-user.target
```

* `systemctl enable wol@<ifname>`

接下来可以通过 `systemctl status wol@<ifname>` 来查看状态。

## 你的个人机器上

安装wakeonlan

首先试试在局域网内wake，`wakeonlan <MAC_ADDR>`

成功唤醒，抓包

![Screenshot 2023-01-18 at 1.15.36 PM](../../../Library/Application%20Support/typora-user-images/Screenshot%202023-01-18%20at%201.15.36%20PM.png)

然后试试从WAN唤醒

先看linux主机公网IP `nslookup xhzq233.tpddns.cn`

在路由器添加规则将magic pkt转发到linux主机。

```sh
wakeonlan -i <WAN_IP> -p <OPEN_PORT> <MAC_ADDR>
```

成功，结束，（需要注意每次断电WOL服务都会失效）。
