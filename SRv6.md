# SRv6
SRv6（Source Routing in IPv6）是一种IPv6扩展，结合了Segment Routing（SR）和IPv6两种技术。它允许数据包的发送者指定其路径，而不是依赖于网络的中间节点来确定数据包的路径。SRv6的核心原理是**在IPv6数据包头部的扩展标头中指定数据包的路径**，从而使网络节点可以根据这些指示来转发数据包。

## SR
Segment Routing（SR）是基于源路由理念而设计的在网络上转发数据包的一种协议。它将网络路径分成一个个段，并且为这些段和网络节点分配Segment ID（SID）。通过对SID进行有序排列（Segment List）， 就可以得到一条转发路径。

而传统路由中，数据包本身是不知道路径的，它被路由器转发时，路由器知道下一跳的地址。

> 举个例子，从南昌坐高铁到长沙，中途会经过宜春和萍乡北两站。传统路由模式需要在南昌买票到宜春，到达宜春后买票去萍乡北，到萍乡北后再买票到长沙。但在SRv6模式，可以直接买从南昌到长沙的票，可以预知所经过的节点，到达各站点只需出示一下我买的票即可。

## SRv6
SRv6的原理可以简要描述如下：
1. Segment Routing Header (SRH)：为了实现srv6，ipv6报文进行了一些扩展，新增了一种扩展头Segment routing header简称srh，分段路由头，用于指定数据包的路径。SRH包含一系列的"段"（Segments），每个段代表网络中的一个节点或一组节点。**网络中间节点可以不支持srv6。但头节点必须支持srv6。**
2. Segment Identifier (SID)：每个节点在网络中被分配一个唯一的标识符，称为Segment Identifier（SID）。这些SID用于标识网络中的节点或链路。
3. Instructions for Packet Processing：发送者通过在IPv6数据包头部的SRH中指定一系列的SID来定义数据包的路径。每个SID代表数据包在网络中经过的节点或链路。
4. Network Node Processing：网络中的节点根据SRH中指定的SID来转发数据包。当数据包到达一个节点时，该节点查找SRH中的下一个SID，并将数据包转发到对应的节点或链路。
5. Flexible Path Selection：SRv6允许发送者在数据包传输过程中动态地调整路径，而无需对网络进行特定配置。这使得网络具有更大的灵活性和可编程性。srv6的网络路径可以编程，业务可以编程，转发行为可以编程，srv6是完全基于架构的，能够跨越APP和网络之间的鸿沟。将APP的应用程序信息带入到网络之中。

**srv6报文格式**

![image](https://github.com/Cookie-ch/note/assets/79464052/91d7e3db-6086-4f04-96d4-1adfc2d63f87)

Net Header取值：4 ——IPv4封装，41——IPv6封装，43——IPv6-Route（IPv6路由扩展头），58——ICMPv6，59——Next Header为空

**srv6网络节点：**
头节点
![头节点](https://github.com/Cookie-ch/note/assets/79464052/d1618f3e-bc96-4322-a2be-8f882b70b21f)

不支持srv6的中间节点 
![不支持srv6的节点](https://github.com/Cookie-ch/note/assets/79464052/859c8a10-a91e-4d22-83a9-0bb580e11b44)

srv6节点
![image](https://github.com/Cookie-ch/note/assets/79464052/8f13ae82-5aab-459e-8360-d1cdaad00b56)

### SRH
Segment routing header简称srh，分段路由头。

srh有两个关键信息：
* 首先是ipv6地址形式的Segment list。这是一个由IPv6地址形式的有序列表，表示了数据包在网络中传输的显式路径。每个IPv6地址代表一个路由段（Segment），这些路由段按照列表中的顺序构成了数据包的传输路径。
* 另外一个关键字段是Segment left （sl）。Segment left是一个指针，指针它指向当前活跃的一个list，它的初始值是0，表示从列表的第一个路由段开始，每次经过一个节点，SL的值就减1，直到减到0为止。SL的最大值是Segment List长度减1，即最后一个路由段的位置。
在srv6里ipv6的目的地址sl字段是一个不断变化的字段。当指针指向当前活跃的list时，需要将list的ipv6地址复制到ipv6目的地址字段。

> * 在SRv6的网络节点处理过程中，如果节点支持SRv6，它将检查SL的值。如果SL指向的是当前节点所对应的路由段，节点将执行以下操作：
>   1. 将SL值减1，更新SL字段。
>   2. 将新的Segment List信息复制到IPv6目的地址字段，即将当前路由段的IPv6地址作为新的目的地。
>   3. 将数据包转发给下一个节点。
> * 如果有一个转发节点不支持ipv6，那么不需要处理ipv6报文里的srh信息。仅依据ipv6目的地址字段查找ipv6路由表，进行普通转发。

当Segment Left字段减为零时，节点会弹出SRH报文头，然后对报文进行进一步的处理，这可能包括查找传统的IPv6路由表中的下一跳地址，或者如果报文已经达到了最终目的地，则进行相应的终节点处理。

### SID
SID（Segment Identifier）由locator、function、argument三部分组成，这三个部分共同构成一个128位的ipv6地址，其中argument字段是可选的，可以不带。locator一般是手工配置，function可以手工配置，也可以由IGP/BGP动态分配

* locator（位置标识）：网络中分配给一个网络节点的标识，可以用于路由和转发数据包。Locator有两个重要的属性，可路由和聚合。在SRv6 SID中Locator是一个可变长的部分，用于适配不同规模的网络。
* Function（功能）：设备分配给本地转发指令的一个ID值，该值可用于表达需要设备执行的转发动作，相当于计算机指令的操作码。在SRv6网络编程中，不同的转发行为由不同的功能ID来表达。一定程度上功能ID和MPLS标签类似，用于标识VPN转发实例等。
* Args（变量）：转发指令在执行的时候所需要的参数，这些参数可能包含流，服务或任何其他相关的可变信息。

> **注：**
>  路由器收到SID，先判断Locator，如果发现Locator是本机发布的，那就去处理Function指令（可以将Locator理解为前缀，在转发时使用；Funcation只有到达目的节点（到达响应Locator前缀对应的节点）之后才会查看）
> 
> 严格意义上来说Segment List并不是IPv6地址，只是以IPv6地址的形式存在
> 
> 只有将Segment List拷贝到IPv6报文的Destination Address字段中的才是IPv6地址
> 
> 一台设备只有一个Locator，并且全网唯一
> 
> 一台设备可以有多个Function，并且只需要保证本地唯一就可以


![image](https://github.com/Cookie-ch/note/assets/79464052/dbbefcc6-007d-4d28-9a12-25e6f6b75d5a)

SID有很多种不同类型：

* End SID（Endpoind SID）：这种SID代表网络中的一个目的节点，他给设备的指令是处理srh，更新ipv6目的地址字段，然后查找ipv6路由表进行报文转发
* end.x SID：代表的是网络中的一个邻接，他给设备的指令是处理srh，更新ipv6目的地址字段，然后从end.x sid指定的出接口转发报文
* end.dt4（6） sid：是一种pe类型的sid，主要用于ipv4（6）场景，标识网络中的VPN实例，他给设备的指令是解封装报文，去除外层的srh和ipv4（6）报文头，然后根据剩余报文里的目的地址查找ipv4（6） vpn实例路由表进行转发
* end.dx4（6） sid：用于标识网络中的某个IPv4（6） CE，他给设备的指令是解封装报文，并将解封装后的IPv4（6）报文在该SID绑定的三层接口上转发。假如有多个接口绑定相同的IPv4（6）实例，可以通过End.DX4（6）来决定走那个接口，将其送到VRF对应的接口
* End.OP SID：是一个OAM类型的SID,主要用于测试（Ping、Tracert场景）。SRv6中，直接ping End SID是ping不同的，由于报文中的Flag中O没有置位，End SID收到后是不会响应的，此时就需要End.OP SID，直接ping End.OP SID是可以ping通的。End.OP SID一般手动配置

IPv4数据包本身不支持SRv6，但SRv6中可以承载IPv4报文。

### srv6报文转发过程
![image](https://github.com/Cookie-ch/note/assets/79464052/ec669dbc-a52b-43f1-a3f4-6850d8a6b720)

上图展示了SRv6转发的一个范例。在这个范例中，结点R1要指定路径（需要通过R2-R3、R4-R5的链路转发）转发到R6，其中R1、R2、R4、R6为有SRv6能力的的设备，R3、R5为不支持SRv6的设备。

1. 步骤一：Ingress结点处理：R1将SRv6路径信息封装在SRH扩展头，指定R2和R4的END.X SID，同时初始化SL = 2，并将SL指示的SID A2::11拷贝到外层IPv6头目的地址。R1根据外层IPv6目的地址查路由表转发到R2。
2. 步骤二：End Point结点处理：R2收到报文以后，根据外层IPv6地址A2::11查找本地Local SID表，命中END.X SID，执行END.X SID的指令动作：SL—，并SL指示的SID拷贝到外层IPv6头目的地址，同时根据END.X关联的下一跳转发。
3. 步骤三：Transit结点处理：R3根据A4::13查IPv6路由表进行转发，不处理SRH扩展头。具备普通的IPv6转发能力即可。
4. 步骤四：End Point结点处理：R4收到报文以后，根据外层IPv6地址A4::13查找本地Local SID表，命中END.X SID，执行END.X SID的指令动作：SL—，并将SL指示的SID拷贝到外层IPv6头目的地址，由于SL = 0, 弹出SRH扩展头，同时根据END.X关联的下一跳转发。
5. 步骤五：弹出SRH扩展头以后，报文就变成普通的IPv6头，由于A6::1是1个正常的IPv6地址，遵循普通的IPv6转发到R6。

## SRv6技术的特点
* 纯IP化：基于Native IPv6转发，SRv6通过扩展报文头实现，没有改变IPv6封装结构，SRv6报文依然是IPv6报文，兼容IPv6设备，网络拓扑信息和业务都被编码在数据包的包头中
* 可扩展性：可编程
* 简化：协议简化，去除LDP、RSVP-TE、VxLAN等
* SRv6与存量IPv6网络兼容：业务开通只需在头尾节点进行部署，缩短部署时间，提升部署效率
* 提升跨域体验：相对于传统的MPLS跨域技术来说，srv6跨域部署更加简单，srv6具有native ipv6的特质，在跨域场景中，只需要将一个域的ipv6路由通过BGP4+引到另外一个域，就可以开展跨域业务不熟，降低了业务部署的复杂性
* 提供高保护效率的FRR保护能力：在SRv6技术的基础上结合RLFA FRR实现高效的TI-LFA FRR算法（同SR-MPLS）

> 参考：
> 
> <https://blog.csdn.net/weixin_43991442/article/details/125324404>
> 
> <https://www.sdnlab.com/23735.html>
> 
> <https://blog.csdn.net/m0_49864110/article/details/123591943>
