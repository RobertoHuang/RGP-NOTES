# 文件配置
在`VirtualBox`的 管理->全局设定-> 常规中可以设定虚拟机保存的目录

# 网络配置

## `NAT`网络配置
网卡1配置如下

![网卡1配置](https://raw.githubusercontent.com/RobertoHuang/RGP-NOTES/master/00.%E7%9B%B8%E5%85%B3%E5%9B%BE%E7%89%87/%E8%BD%AF%E4%BB%B6%E9%85%8D%E7%BD%AE%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/%E7%BD%91%E5%8D%A11%E9%85%8D%E7%BD%AE.png)

修改`/etc/sysconfig/network-scripts/ifcfg-enp0s3`的配置，将`ONBOOT`改为`yes`，`BOOTPROTO`改为`dhcp`，重启网络(`service network restart`)即可`ping`通外网，修改后的配置文件如下
```
TYPE=Ethernet
BOOTPROTO=dhcp
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_FAILURE_FATAL=no
NAME=enp0s3
UUID=6e1437b5-44bf-4549-991e-545a8824b583
DEVICE=enp0s3
ONBOOT=yes
```
## `HOST ONLY`网络配置
网卡2配置如下

![网卡2配置](https://raw.githubusercontent.com/RobertoHuang/RGP-NOTES/master/00.%E7%9B%B8%E5%85%B3%E5%9B%BE%E7%89%87/%E8%BD%AF%E4%BB%B6%E9%85%8D%E7%BD%AE%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/%E7%BD%91%E5%8D%A12%E9%85%8D%E7%BD%AE.png)

配置静态`IP`地址(即虚拟机`IP`地址不会随宿主机`IP`改变而发生改变)，修改`/etc/sysconfig/network-scripts/ifcfg-enp0s8`的配置，将`BOOTPROTO`改为`static`，`IPADDR`可以自己指定，添加`NETMASK`配置，最后重启网络(`service network restart`)，修改后的配置文件如下
```
TYPE=Ethernet
BOOTPROTO=static
DEFROUTE=yes
PEERDNS=yes
PEERROUTES=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_FAILURE_FATAL=no
NAME=enp0s8
UUID=392dd639-73af-4a1a-a093-120f380fd85d
DEVICE=enp0s8
ONBOOT=yes
IPADDR=192.168.56.128
NETMASK=255.255.255.0
```
## 使用`ip addr`命令查看网络配置
```
[root@localhost ~]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:0b:ad:cb brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 86267sec preferred_lft 86267sec
    inet6 fe80::a00:27ff:fe0b:adcb/64 scope link 
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:24:6e:58 brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.128/24 brd 192.168.56.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe24:6e58/64 scope link 
       valid_lft forever preferred_lft forever
```
# 后台启动虚拟机
1.进入`VirtualBox`安装目录

```
e:
cd E:\Develop\VirtualBox
```
2.后台启动虚拟机 其中master是虚拟机名
```
.\VBoxManage.exe  startvm  master --type headless
```