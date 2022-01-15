# ospf-over-wireguard

本文记录借鉴 [使用 RouterOS，OSPF 和树莓派为国内外 IP 智能分流](https://idndx.com/use-routeros-ospf-and-raspberry-pi-to-create-split-routing-for-different-ip-ranges/)文章做分流,并改为直接通过隧道从vps接收路由。

DNS部分在旁路linux上部署[mosdns-cn](https://github.com/allanchen2019/mosdns-cn-debian-install)。
