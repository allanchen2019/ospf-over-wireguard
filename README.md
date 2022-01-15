# ospf-over-wireguard

本文记录借鉴 [使用 RouterOS，OSPF 和树莓派为国内外 IP 智能分流](https://idndx.com/use-routeros-ospf-and-raspberry-pi-to-create-split-routing-for-different-ip-ranges/)文章做分流,并改为直接通过隧道从vps接收路由。

DNS部分在旁路linux上部署[mosdns-cn](https://github.com/allanchen2019/mosdns-cn-debian-install)。

#### 环境概述：

vps和本地Mikrotik设备之间通过wireguard隧道连接，具体配置方法不再赘述，这里仅对和本文相关的配置做必要说明。
vps wg0 address：10.0.1.1
routeros wgdc1 address：10.0.1.2

#### 1.vps配置

首先安装bird2:

`apt update && apt install bird2`

生成!cn静态路由表：

```
git clone https://github.com/dndx/nchnroutes.git
cd nchnroutes
```
