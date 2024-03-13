基于IPv6转发平面的segment routine，即Segment routine IPV6，简称srv6。

产生背景：sdn，云网，nfv，ipv6

srv6的网络路径可以编程，业务可以编程，转发行为可以编程，srv6是完全基于架构的，能够跨越APP和网络之间的鸿沟。将APP的应用程序信息带入到网络之中。
srv6是基于igp和bgp扩展来实现的。没有mpls标签。
纯ip化。网络中间节点可以不支持srv6。

为了实现srv6，ipv6报文进行了一些扩展，新增了一种扩展头Segment routing header简称srh。
srh有两个关键信息，首先是ipV6地址形式的Segment list。进行有序的一个排列，就构成了srv6里面的显示路径。
另外一个关键字段是segment left （sl）。Segment left是一个指针，指针它指向当前活跃的一个list，初始值是0，它的最大值取值是seven miss的个数减1。
在srv6里ipv6的目的地址Da字段是一个不断变化的字段。当指针指向当前活跃的list时，需要将list的IPv6地址复制到IPv6目的地址字段。

如果有一个转发节点不支持ipV6，那么不需要处理IPv6报文里的srh信息。仅依据IPv6目的地址字段查找IPv6路由表，进行普通转发。
如果节点出现SrV6，且出现在了segment list中。那么就要处理rsh，将segment left进行-1操作。然后将新的segment list信息复制到IPv6目的地址字段。将报文向下一个节点进行转发。

当segment left字段减为零时，节点可以弹出srh报文头，然后对报文进行下一步处理。

## segment routing的技术特点
* 源路由：网络拓扑信息和业务都被编码在数据包的包头中
* 可扩展性：网络fabric不保留TE或NFV使用的任何flow状态
* 简化：协议简化，去除LDP、RSVP-TE、VxLAN等
* 端到端：DC、Metro、WAN
