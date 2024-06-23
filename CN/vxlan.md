# VXLAN
## 背景
传统的二层网络面临VLAN资源不足、虚机迁移业务部中断等挑战，并且多数据中心之间要通过实现三层网络连接的两个二层网络互通

## 什么是VXLAN
VXLAN（Virtual Extensible LAN）可扩展虚拟局域网。是基于IP网络、采用“MAC in UDP”封装形式的二层VPN技术，可以基于已有的网络，为分散的物理站点提供二层互联，并能够为不同的租户提供业务隔离

VXLAN是Overlay技术的一种，VXLAN通过隧道机制在现有网络上构建一个叠加的网络，从而绕过现有VLAN标签的限制。Overlay可以做到流量控制。

VLAN的作用是从逻辑上隔离局域网中的设备，VXLAN是将两个局域网打通

**VTEP**（Virtual Tunnel Endpoint）是VXLAN隧道端点，是VXLAN的边缘设备，用于VXLAN报文的封装和解封装。如物理交换机、虚拟交换机、等，可以是独立的物理设备或虚机所在的服务器
VTEP只支持VXLAN二层转发功能的设备，只能在相同VXLAN内进行二层转发

## VXLAN的优势
1. 支持大量的租户：声依永24位标识符，最多可支持2的24磁盘（16777216）个VXLAN，解决了传统二层网络VLAN资源不足的问题
2. 虚机迁移IP、MAC不变：采用MAC in UDP的封装方式，实现原始二层报文在IP网络中的透明传输（支持虚机跨网络迁移）
3. 易于维护：基于IP网络组建大二层网络，可以充分利用现有的IP网络技术；只有IP核心网络的边缘设备需要进行VXLAN处理，网络中间设备只需根据IP头转发报文

## VXLAN报文结构
* 外层以太头
  * 14字节，若包含VLAN tag则为18字节
  * 源MAC为源VM所属VTEP的MAC地址
  * 目的MAC为到达目标VTEP路径上下一跳设备的MAC地址
* 外层IP头
  * 可以是ipv4（20字节）也可以是ipv6（40字节）
  * 源IP为源VTEP的IP地址，目标IP为目标VTEP的IP地址
* 外层UDP头（8字节）
  * 目的端口号缺省为4789，表示内层封装报文为VXLAN报文。源端口为本地随机获取
  * 可用于VTEP之间的多路负载分担的计算
  * 基于UDP来实现VXLAN隧道，VXLAN头部对于网络而言是一个服务，可以避免NAT、防火墙的影响，做到穿越NAT
  * 迁移虚机是做内存迁移，所以传输的数据小，时间够快，做到轻量级
* VXLAN头（8字节）
  * 标记位8bit，其中I位为1时，表示VXLAN头中的VXLAN ID有效
  * VXLAN ID（24bit），又称VNI，网络标识符，不同VXLAN网络中的用户终端不能二层互通
  * Reserved（8bit），协议保留位
* 原始二层数据，虚机发送的原始以太报文

![image](https://github.com/Cookie-ch/note/assets/79464052/64a062f0-9114-4513-96eb-28be31d3c628)

虚机网关在物理交换机上时，报文会封装外层MAC头
![image](https://github.com/Cookie-ch/note/assets/79464052/e7694b2e-85f9-4909-858c-9e8b8ee3911c)

虚机网关在虚拟交换机上时，不会封装外层MAC头
![image](https://github.com/Cookie-ch/note/assets/79464052/a48839b5-18be-4bad-b557-e3c0509eb53e)

## 一些特定名词
### BD
BD：桥域（Bridge Domain），类似传统网络中采用VLAN划分广播域，在VXLAN网络中一个BD标识一个大二层广播域

### VBDIF
VBDIF：类似于VLANIF。VBDIF接口在VXLAN三层网关上配置，是基于BD创建的三层逻辑接口

通过VBDIF接口可以实现不同网段的用户通过VXLAN网络通信，以及VXLAN网络和非VXLAN网络之间的同学，也可以实现二层网络接入三层网络

### VAP
VAP：虚拟接入点（Virtual Access Point），实现VXLAN的业务接入。

VAP有两种配置方式：
二层子接口方式接入，例如在sw1创建二层子接口关联BD10，则这个子接口下的特定流量会被注入到BD10
VLAN绑定方式接入，例如在sw2配置VLAN10与广播域BD10关联，则所有VLAN10的流量会被注入到BD10

### VSI
VSI：虚拟交换实例（Virtual Switch Instance），是在虚拟交换机上创建的逻辑实体。可以把同一个VSI的VXLAN网络看做一个大二层VXLAN Domain

每个 VSI 可以看作是一个独立的逻辑交换机，具有自己的端口、VLAN 设置和交换规则。在虚拟化环境中，多个 VSI 可以共享同一个物理交换机，并且能够隔离不同的虚拟网络流量

VSI具备传统以太网交换机的所有功能，如MAC地址表、老化机制等。VSI与VXLAN需一一对应

### VSI-Interface
VSI ：虚拟交换接口（Virtual Switch Interface），这是指连接虚拟交换机和物理网络设备（如物理交换机或路由器）的逻辑接口。是VXLAN内虚拟机的网关，用于处理跨VXLAN网络的报文转发

一个VXLAN网络对应一个VSI-Interface

### AC
AC：接入链路（Attachment Circuit），就是接入链路，接入VTEP的逻辑链路。也称以太网服务实例（Service Instannce），指定该服务的感兴趣数据流（通常基于VLAN），指定感兴趣六在哪个VSI中转发
![image](https://github.com/Cookie-ch/note/assets/79464052/72b256fb-f65b-4e36-9bf3-d43b9d714eac)

## VXLAN的实现原理
![image](https://github.com/Cookie-ch/note/assets/79464052/9e3e5bec-843f-4f1e-a6d3-8826e709b97b)
![image](https://github.com/Cookie-ch/note/assets/79464052/3e733c41-fc9a-4aef-a336-39c9e26e7b7d)

### VXLAN MAC地址表项
VXLAN实现的是在overlay网络中进行二层转发，转发单播数据帧依赖的是MAC地址表项

VTEP接收到BD内来自本地的数据帧，将数据帧的源MAC地址添加到该BD的MAC地址表中，出接口为收到数据帧的接口

该表项用于指导发往本VTEP下连接终端的数据帧的转发

![image](https://github.com/Cookie-ch/note/assets/79464052/c147d821-df55-424f-a935-d83f6f77257f)
![image](https://github.com/Cookie-ch/note/assets/79464052/1b7ed03d-c8b5-408f-b495-7e18cb41257b)

### MAC地址动态学习
转发属于远端VTEP下所连接设备的数据帧，需要先学习到远端设备的MAC地址
该过程与传统MAC地址表形成过程类似
![image](https://github.com/Cookie-ch/note/assets/79464052/3dc93f3d-87ef-4f88-b794-8286c78a7c92)

![image](https://github.com/Cookie-ch/note/assets/79464052/63991d2b-12bd-476c-aeca-081996fc5810)


## VXLAN运行机制
### VXLAN隧道工作模式
#### L2 Gateway
L2 Gateway二层转发模式，VTEP通过查找**MAC地址表项**对流量进行转发，用于VXLAN和**VLAN**之间的二层通讯

下图是个例子（左上源MAC和目的MAC没有写）
![image](https://github.com/Cookie-ch/note/assets/79464052/560ead03-37a3-4597-9eed-12729b2f07ae)


#### IP Gateway
IP Gateway三层转发模式，VTEP设备通过查找**ARP表项**对流量进行转发，用于VXLAN和外部**IP**网络之间的三层通讯


### VXLAN隧道的建立与关联
VXLAN隧道由一堆VTEP确定，报文在VTEP设备进行封装之后再VXLAN隧道中依靠路由进行传输。只要VXLAN隧道的两端VTEP是三层路由可达的，VXLAN隧道就可以建立成功

#### VXLAN隧道建立的方式
**静态隧道：** 通过用户手动配置本端和对端的VNI、VTEP地址和头端复制列表来完成
![image](https://github.com/Cookie-ch/note/assets/79464052/d0d5b05b-7da5-4a44-85fd-d449e9193655)


**动态隧道：** 通过BGP EVNP（以太网虚拟私有网络）方式，在VTEP之间建立BGP EVPN对等体，然后对等体之间利用BGP EVPN路由来互相传递VNI和VTEP IP，自动建立VXLAN隧道

#### VXLAN对到与VXLAN关联的方式
**手动** ：手动将VXLAN隧道与VXLAN关联

**自动** ：通过EVPN协议自动将VXLAN隧道与VXLAN关联

## 配置
> 静态隧道实验：<https://www.bilibili.com/video/BV11G411r7Ux?p=4&spm_id_from=pageDriver&vd_source=9a5a23788869349258fb86060e91a4a9>
### VXLAN接入配置
创建BD
```
bridge-domain 10
```

创建子接口:G1/0/2收到vlan10的报文，则将报文关联到BD10
```
interface gigabitethernet 1/0/2
port link-type trunk
interface gigabitethernet 1/0/2.1 mode l2       // 创建子接口
encapsulation dot1q vid 10                      //子接口和vlan10关联
bridge-domain 10                                //子接口和BD关联
```

### VXLAN隧道配置
属于BD10的报文，将打上VNI100的VXLAN标识，并封装上源地址为1.1.1.1，目的地址为2.2.2.2的新IP头部，发到对端VNE
```
bridge-domian 10    //进入BD10配置视图
vxlan vni 100       //将BD10和VNI100进行关联
interface vne 1     //创建VXLAN隧道，编号为1
source 1.1.1.1
vni 100 head-end peer-list 2.2.2.2    //隧道目的IP
```
