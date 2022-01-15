# ospf-over-wireguard

本文记录借鉴 [使用 RouterOS，OSPF 和树莓派为国内外 IP 智能分流](https://idndx.com/use-routeros-ospf-and-raspberry-pi-to-create-split-routing-for-different-ip-ranges/)文章做分流,并改为直接通过隧道从vps接收路由,旁路Linux仅作为DNS分流器，详见[mosdns-cn](https://github.com/allanchen2019/mosdns-cn-debian-install)。

#### 环境概述：

本地Mikrotik设备（hapac2 v7.2rc1）负责拨号，vps Debian 11 Linux，之间通过wireguard隧道连接，具体配置方法不再赘述，这里仅对和本文相关的配置做必要说明。

假设vps wg0 address：`10.0.1.1/24`

本地routeros wgdc1 address：`10.0.1.2/24`

wg配置中如有Post Up & Down和DNS条目先注释掉，在`[Interface]`最后加入 `Table = off`因为我们不需要vps插入wg路由。
在`[Peer]`中负责连接本地ros的AllowedIPs修改为如下形式：

`AllowedIPs = 10.0.1.2/32,224.0.0.5/32`

修改完毕后`g-quick down wg0 && wg-quick up wg0`重启接口。

#### 1.vps配置

首先安装bird2:

`apt update && apt install bird2`

拉取生成!cn静态路由表的库：
```
git clone https://github.com/dndx/nchnroutes.git
cd nchnroutes
```
编辑Makefile文件，假设vps网卡接口是eth0，公网ip是1.2.3.4，在`python3 produce.py`后加入` --next eth0 --exclude 1.2.3.4/32`，保存退出。

生成路由表并复制到bird配置目录：
```
make
cp routes4.conf /etc/bird/routes4.conf
cd /etc/bird
mv bird.conf bird.conf.orig
touch bird.conf
```
编辑/etc/bird.conf内容如下
```
log syslog all;

router id 10.0.1.1;

protocol device {
}



protocol static {
        ipv4;
        include "routes4.conf";
}

protocol ospf v2 {
        ipv4 {
                export all;
        };
        area 0.0.0.0 {

                interface "wg0" {
                        type ptp;
                        hello 5;
                        dead 40;
                };
        };
}

```
这里的router id要根据vps实际wg接口地址填写，其他保持默认即可。

开启ip转发：

`sed -i.bak 's/net.ipv4.ip_forward = 0/net.ipv4.ip_forward = 1/g' /etc/sysctl.conf && sysctl -p |grep forward`

返回`net.ipv4.ip_forward = 1`即为执行成功。

修改iptables：

安装防火墙规则持久化包：`apt install iptables-persistent`，过程中询问是否保存当前规则都选No。

编辑/etc/iptables/rules.v4写入如下内容：

```
*filter
:INPUT ACCEPT [0:0]
:FORWARD DROP [0:0]
:OUTPUT ACCEPT [0:0]
-A FORWARD -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
-A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i eth0 -o wg0 -j ACCEPT
COMMIT

*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A POSTROUTING -o wg0 -j MASQUERADE
COMMIT
```
保存退出，执行`iptables-restore < /etc/iptables/rules.v4`让配置生效。

#### 2.本地RouterOS配置

假设ros wireguard连接vps接口名称`wgdc1`接口地址`10.0.1.2/24`

Change MSS：
```
/ip firewall mangle add action=change-mss chain=forward new-mss=clamp-to-pmtu passthrough=yes protocol=tcp tcp-flags=syn
/ip firewall mangle add action=change-mss chain=output new-mss=clamp-to-pmtu passthrough=yes protocol=tcp tcp-flags=syn
/ip firewall mangle add action=change-mss chain=postrouting new-mss=clamp-to-pmtu passthrough=yes protocol=tcp tcp-flags=syn
```
创建OSPF实例：
```
/routing ospf instance add name=dc1 router-id=10.0.1.2 routing-table=ospf
/routing ospf area add instance=dc1 name=ospf-area-dc1
/routing ospf interface-template add area=ospf-area-dc1 hello-interval=5s interfaces=wgdc1 networks=10.0.1.0/24 type=ptp
```

