# VXLAN
## 背景
传统的二层网络面临VLAN资源不足、虚机迁移业务部中断等挑战，并且多数据中心之间要通过实现三层网络连接的两个二层网络互通

## 什么是VXLAN
VXLAN（Virtual Extensible LAN）可扩展虚拟局域网。是基于IP网络、采用“MAC in UDP”封装形式的二层VPN技术，可以基于已有的网络，为分散的物理站点提供二层互联，并能够为不同的租户提供业务隔离

VXLAN是Overlay技术的一种，VXLAN通过隧道机制在现有网络上构建一个叠加的网络，从而绕过现有VLAN标签的限制。Overlay可以做到流量控制。

VLAN的作用是从逻辑上隔离局域网中的设备，VXLAN是将两个局域网打通

**VTEP**（Virtual Tunnel Endpoint）是VXLAN隧道端点，是VXLAN的边缘设备，VXLAN的相关处理都在VTEP上进行。它主要负责在物理网络与虚拟网络之间进行数据包的封装和解封装，从而实现跨物理网络的虚拟网络通信。如物理交换机、虚拟交换机、等，可以是独立的物理设备或虚机所在的服务器
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
  * Reserved，协议保留位
* 原始二层数据，虚机发送的原始以太报文

![image](https://github.com/Cookie-ch/note/assets/79464052/64a062f0-9114-4513-96eb-28be31d3c628)

虚机网关在物理交换机上时，报文会封装外层MAC头
![image](https://github.com/Cookie-ch/note/assets/79464052/e7694b2e-85f9-4909-858c-9e8b8ee3911c)

虚机网关在虚拟交换机上时，不会封装外层MAC头
![image](https://github.com/Cookie-ch/note/assets/79464052/a48839b5-18be-4bad-b557-e3c0509eb53e)

## VSI
VSI：虚拟交换实例（Virtual Switch Instance），是在虚拟交换机上创建的逻辑实体。可以把同一个VSI的VXLAN网络看做一个大二层VXLAN Domain

每个 VSI 可以看作是一个独立的逻辑交换机，具有自己的端口、VLAN 设置和交换规则。在虚拟化环境中，多个 VSI 可以共享同一个物理交换机，并且能够隔离不同的虚拟网络流量

VSI具备传统以太网交换机的所有功能，如MAC地址表、老化机制等。VSI与VXLAN需一一对应

## VSI-Interface
VSI ：虚拟交换接口（Virtual Switch Interface），这是指连接虚拟交换机和物理网络设备（如物理交换机或路由器）的逻辑接口。是VXLAN内虚拟机的网关，用于处理跨VXLAN网络的报文转发

一个VXLAN网络对应一个VSI-Interface

## AC
AC：接入链路（Attachment Circuit），就是接入链路，接入VTEP的逻辑链路。也称以太网服务实例（Service Instannce），指定该服务的感兴趣数据流（通常基于VLAN），指定感兴趣六在哪个VSI中转发
![image](https://github.com/Cookie-ch/note/assets/79464052/72b256fb-f65b-4e36-9bf3-d43b9d714eac)

## VXLAN的实现原理
![image](https://github.com/Cookie-ch/note/assets/79464052/9e3e5bec-843f-4f1e-a6d3-8826e709b97b)
![image](https://github.com/Cookie-ch/note/assets/79464052/3e733c41-fc9a-4aef-a336-39c9e26e7b7d)



## VXLAN运行机制
### VXLAN隧道工作模式
#### L2 Gateway
L2 Gateway二层转发模式，VTEP通过查找**MAC地址表项**对流量进行转发，用于VXLAN和**VLAN**之间的二层通讯

下图是个例子（左上源MAC和目的MAC没有写）
![image](https://github.com/Cookie-ch/note/assets/79464052/560ead03-37a3-4597-9eed-12729b2f07ae)


#### IP Gateway
IP Gateway三层转发模式，VTEP设备通过查找**ARP表项**对流量进行转发，用于VXLAN和外部**IP**网络之间的三层通讯


### VXLAN隧道的建立与关联
发现远端VTEP，在VTEP之间建立VXLAN隧道，并将VXLAN隧道与VXLAN关联
#### VXLAN隧道建立的方式
**手动** ：手动配置Tunnel接口，并指定隧道的源和目的IP地址分别为双方VTEP的IP地址

**自动** ：通过EVNP（以太网虚拟专用网络）发现对端VTEP后，自动建立VXLAN隧道

#### VXLAN对到与VXLAN关联的方式
**手动** ：手动将VXLAN隧道与VXLAN关联

**自动** ：通过EVPN协议自动将VXLAN隧道与VXLAN关联

### 识别VXLAN
识别接收到的VXLAN，将报文的源MAC地址学习到的VXLAN对应到VSI，并在该VSI内转发该报文

识别方式有两种：

**本地站点内接收到数据帧的识别方式：**
* 三层口与VSI关联：该三层口接收到的数据帧均属于指定的VSI，VSI内创建的VXLAN即为数据帧所属的VXLAN
* 以太网服实例与VSI关联：以太网服务实例定义一系列匹配规则
* VLAN与VXLAN关联：VTEP接收到的该VLAN的数据帧均属于执行的VXLAN

**VXLAN隧道上接收报文的识别方式：**
VTEP根据报文中携带的VXLAN ID判断该报文所属的VXLAN
