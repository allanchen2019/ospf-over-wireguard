# ospf-over-wireguard

本文记录借鉴 [使用 RouterOS，OSPF 和树莓派为国内外 IP 智能分流](https://idndx.com/use-routeros-ospf-and-raspberry-pi-to-create-split-routing-for-different-ip-ranges/)改为直接通过隧道从vps接收路由,

旁路Linux作为DNS分流器，完成本文IP分流后部署，详见[mosdns-cn](https://github.com/allanchen2019/mosdns-cn-debian-install)。

## 环境概述：

本地Mikrotik设备（hapac2 v7.2rc3）负责拨号，远程vps Debian 11 Linux，之间通过wireguard隧道连接，具体配置方法不再赘述，这里仅对和本文相关的配置做必要说明。
本地ROS如果为CHR并且不负责拨号，请自行完成dst 0.0.0.0/0的网关设置。

假设vps wg-ospf address：`10.0.1.0/31`

本地routeros wgdc1 address：`10.0.1.1/31`

wg配置中如有Post Up & Down和DNS条目先注释掉，在`[Interface]`最后加入 `Table = off`因为我们不需要vps插入wg路由。
在`[Peer]`中AllowedIPs修改为如下形式：

`AllowedIPs = 0.0.0.0/0`

修改完毕后`wg-quick down wg-ospf && wg-quick up wg-ospf`重启接口。

## 1.vps配置

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
router id 10.0.1.0;
protocol device {
        scan time 60;
}

protocol static {
        ipv4;
        include "routes4.conf";
}

protocol ospf v2 {
        ipv4 {
                export all;
                import none;
        };
        area 0.0.0.0 {
                interface "wg*" {
                        type ptp;
                        hello 10;
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
-A FORWARD -i eth0 -o wg-ospf -j ACCEPT
COMMIT

*nat
:PREROUTING ACCEPT [0:0]
:INPUT ACCEPT [0:0]
:POSTROUTING ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A POSTROUTING -o wg-ospf -j MASQUERADE
COMMIT
```
保存退出，执行`iptables-restore < /etc/iptables/rules.v4`让配置生效。

启动BIRD：

`birdc c`

检查BIRD状态：

`birdc s p a`

可以在static1协议下看到如下消息：

```
static1    Static     master4    up     2022-01-06
  Channel ipv4
    State:          UP
    Table:          master4
    Preference:     200
    Input filter:   ACCEPT
    Output filter:  REJECT
    Routes:         12888 imported, 0 exported, 12888 preferred
    Route change stats:     received   rejected   filtered    ignored   accepted
      Import updates:          12893          0          0          0      12893
      Import withdraws:            5          0        ---          0          5
      Export updates:              0          0          0        ---          0
      Export withdraws:            0        ---        ---        ---          0
```
说明静态路由条目已经注入BIRD。

## 2.本地RouterOS配置

假设ros wireguard连接vps接口名称`wgdc1`接口地址`10.0.1.1/31`

Change MSS：
```
/ip firewall mangle add action=change-mss chain=forward new-mss=clamp-to-pmtu passthrough=yes protocol=tcp tcp-flags=syn
/ip firewall mangle add action=change-mss chain=output new-mss=clamp-to-pmtu passthrough=yes protocol=tcp tcp-flags=syn
```
NAT：

`/ip firewall nat add action=src-nat chain=srcnat out-interface=wgdc1 to-addresses=10.0.1.1`

创建路由表：

`/routing table add disabled=no fib name=ospf`

创建OSPF实例：
```
/routing ospf instance add name=dc1 router-id=10.0.1.1 routing-table=ospf
/routing ospf area add instance=dc1 name=ospf-area-dc1
/routing ospf interface-template add area=ospf-area-dc1 hello-interval=10s interfaces=wgdc1 networks=10.0.1.0/31 type=ptp
```

运气好的话`/routing ospf neighbor pr`就可以看到邻居状态，过几十秒状态应该为full

` 0  D instance=dc1 area=ospf-area-dc1 address=10.0.1.1 router-id=10.0.1.1 state="Full" state-changes=5 adjacency=6h11m6s timeout=37s`

`/ip route pr`可以看到vps端发来的路由：

```
 #       DST-ADDRESS        GATEWAY            DISTANCE
...
   DAo   1.0.0.0/24         10.0.1.1%wgdc1          110
   DAo   1.0.4.0/22         10.0.1.1%wgdc1          110
   DAo   1.0.16.0/20        10.0.1.1%wgdc1          110
   DAo   1.0.64.0/18        10.0.1.1%wgdc1          110
   DAo   1.0.128.0/17       10.0.1.1%wgdc1          110
   DAo   1.1.1.0/24         10.0.1.1%wgdc1          110
...
```
同时vps端执行`birdc s p a`也能看到ospf1协议已经UP状态：
```
ospf1      OSPF       master4    up     2022-01-06    Running
  Channel ipv4
    State:          UP
    Table:          master4
    Preference:     150
    Input filter:   ACCEPT
    Output filter:  ACCEPT
    Routes:         1 imported, 12888 exported, 1 preferred
    Route change stats:     received   rejected   filtered    ignored   accepted
      Import updates:              2          0          0          0          2
      Import withdraws:            1          0        ---          0          1
      Export updates:          12895          2          0        ---      12893
      Export withdraws:            6        ---        ---        ---          5
```

最后在Routing Rule里加入要分流的内网ip（段），比如`192.168.2.0/24`的所有设备都执行分流：

`/routing rule add action=lookup comment=science disabled=no src-address=192.168.2.0/24 table=ospf`

终于写完了，喝杯咖啡压压惊:upside_down_face:
