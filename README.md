# 重要！纯萌新小白慎入！~~现在关闭网页还来得及~~


本文记录借鉴 [使用 RouterOS，OSPF 和树莓派为国内外 IP 智能分流](https://idndx.com/use-routeros-ospf-and-raspberry-pi-to-create-split-routing-for-different-ip-ranges/)，稍作调整，直接通过隧道从vps接收路由。

关于DNS分流，详见[mosdns-cn](https://github.com/allanchen2019/mosdns-cn-debian-install)。

## 环境概述：

本地Mikrotik设备（hapac2 v7.3）负责拨号，远程vps Debian 11 Linux，之间通过wireguard隧道连接，具体配置方法不再赘述，这里仅对和本文相关的配置做必要说明。


假设vps wg接口名称 `wg0`，地址：`10.0.1.0/31`,
网卡名 `eth0`，公网ip地址：`1.2.3.4`

本地ROS wg接口名称`wg-dc1` ，地址：`10.0.1.1/31`

## 0.配置VPS WireGuard：

在`[Interface]`段最后加入：
`Table = off`


`[Peer]`段修改：
`AllowedIPs = 0.0.0.0/0`

修改完毕wg0.conf如下所示，应该只有这些行，

其他行全部删除：

```
[Interface]
PrivateKey = ***=
Address = 10.0.1.0/31
ListenPort = 65535
Table = off

[Peer]
PublicKey = ***=
AllowedIPs = 0.0.0.0/0
```

执行`wg-quick down wg0 && wg-quick up wg0`重启接口。

## 1.配置VPS非cn路由表、iptables、bird

首先安装bird2:

`apt update && apt install bird2`

拉取生成!cn静态路由表的库：
```
git clone https://github.com/dndx/nchnroutes.git
cd nchnroutes
```
编辑Makefile文件，假设公网ip是1.2.3.4，在`python3 produce.py`后加入` --next eth0 --exclude 1.2.3.4/32`，保存退出。

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
debug protocols all;
router id 10.0.1.0;
protocol device {
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

开启ip转发,修改`/etc/sysctl.conf`,添加或修改为:

`net.ipv4.ip_forward = 1`

执行`sysctl -p`生效。


安装防火墙规则持久化包
`apt install iptables-persistent`

过程中询问是否保存当前规则，选两次No。


新建`/etc/iptables/rules.v4`文件，写入如下内容：

```
*nat
:PREROUTING ACCEPT
:INPUT ACCEPT
:OUTPUT ACCEPT
:POSTROUTING ACCEPT
-A POSTROUTING -o eth0 -j MASQUERADE
COMMIT

*filter
:INPUT ACCEPT
:FORWARD ACCEPT
:OUTPUT ACCEPT
-A FORWARD -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
COMMIT

*mangle
:PREROUTING ACCEPT
:INPUT ACCEPT
:FORWARD ACCEPT
:OUTPUT ACCEPT
:POSTROUTING ACCEPT
-A FORWARD -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
COMMIT
```
其中`-A POSTROUTING -o eth0 -j MASQUERADE`的 `eth0` 按vps网卡名实际情况替换。

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

## 2.配置ROS

打开ROS命令行依次执行下列命令：

Change MSS：
```
/ip firewall mangle add action=change-mss chain=forward new-mss=clamp-to-pmtu passthrough=yes protocol=tcp tcp-flags=syn
```
SRCNAT：

`/ip firewall nat add action=src-nat chain=srcnat out-interface=wg-dc1 to-addresses=10.0.1.1`

创建OSPF实例：
```
/routing ospf instance add name=dc1 router-id=10.0.1.1
/routing ospf area add instance=dc1 name=ospf-area-dc1
/routing ospf interface-template add area=ospf-area-dc1 hello-interval=10s interfaces=wg-dc1 type=ptp
```

运气好的话`/routing ospf neighbor pr`就可以看到邻居状态，过几十秒状态应该为full


`/ip route pr`可以看到vps端发来的路由：

```
 #       DST-ADDRESS        GATEWAY            DISTANCE
...
   DAo   1.0.0.0/24         10.0.1.1%wg-dc1          110
   DAo   1.0.4.0/22         10.0.1.1%wg-dc1          110
   DAo   1.0.16.0/20        10.0.1.1%wg-dc1          110
   DAo   1.0.64.0/18        10.0.1.1%wg-dc1          110
   DAo   1.0.128.0/17       10.0.1.1%wg-dc1          110
   DAo   1.1.1.0/24         10.0.1.1%wg-dc1          110
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

## 3.分流验证：

ROS:

`/tool/traceroute 1.1.1.1`第一跳为10.0.1.1

`/tool/traceroute 223.5.5.5`第一跳不是10.0.1.1

终于改完了，~~喝杯咖啡压压惊~~  你学废了吗:upside_down_face:
