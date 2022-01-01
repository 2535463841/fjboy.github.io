## 架构

Neutron 采用的是分布式架构，包括 Neutorn Server、各种 plugin/agent、database 和 message queue。

1. Neutron server 接收 api 请求。
2. plugin/agent 实现请求。
3. database 保存 neutron 网络状态。
4. message queue 实现组件之间通信。

> 5 类组件

- Neutron-server可以理解为一个专门用来接收Neutron     REST API调用的服务器，然后负责将不同的rest api分发到不同的neutron-plugin上。
- Neutron-plugin可以理解为不同网络功能实现的入口，各个厂商可以开发自己的plugin。Neutron-plugin接收neutron-server分发过来的REST     API，向neutron database完成一些信息的注册，然后将具体要执行的业务操作和参数通知给自身对应的neutron agent。
- Neutron-agent可以直观地理解为neutron-plugin在设备上的代理，接收相应的neutron-plugin通知的业务操作和参数，并转换为具体的设备级操作，以指导设备的动作。当设备本地发生问题时，neutron-agent会将情况通知给neutron-plugin。
- Neutron     database，顾名思义就是Neutron的数据库，一些业务相关的参数都存在这里。
- Network     provider，即为实际执行功能的网络设备，**一般为虚拟交换机**（OVS或者Linux Bridge）。

> 两类Plugin

- Core-plugin，Neutron中即为ML2（Modular     Layer 2），负责管理L2的网络连接。ML2中主要包括network、subnet、port三类核心资源，对三类资源进行操作的REST     API被neutron-server看作Core API，由Neutron原生支持。其中：

| 资源    | 说明                                                         |
| ------- | ------------------------------------------------------------ |
| Network | 代表一个隔离的二层网段，是为创建它的租户而保留的一个广播域。subnet和port始终被分配给某个特定的network。Network的类型包括Flat，VLAN，VxLAN，GRE等等。 |
| Subnet  | 代表一个IPv4/v6的CIDR地址池，以及与其相关的配置，如网关、DNS等等，该subnet中的  VM 实例随后会自动继承该配置。Sunbet必须关联一个network。 |
| Port    | 代表虚拟交换机上的一个虚机交换端口。VM的网卡VIF连接 port 后，就会拥有 MAC  地址和 IP 地址。Port 的 IP 地址是从 subnet 地址池中分配的。 |

- Service-plugin，即为除core-plugin以外其它的plugin，包括l3     router、firewall、loadbalancer、VPN、metering等等，主要实现L3-L7的网络服务。这些plugin要操作的资源比较丰富，对这些资源进行操作的REST     API被neutron-server看作Extension API，需要厂家自行进行扩展。

## 控制端的实现

从neutron-server的启动开始说起。neutron-server的启动入口在neutron.server.__init__中，主函数中主要就干了两件事，

1. 启动wsgi服务器监听Neutron REST API，
2. 启动rpc服务，用于core     plugin与agent间的通信，两类服务作为绿色线程并发运行。从SDN的角度来看，wsgi负责Neutron的北向接口，而Neutron的南向通信机制主要依赖于rpc来实现（当然，不同厂家的plugin可能有其它的南向通信机制）。
2. Neutron内部，plugin与数据库交互，获取业务的全局参数，然后通过rpc机制将操作与参数传给设备上的Agent（某些plugin和ML2 Mechanism Driver通过别的方式与Agent通信，比如REST API、NETCONF等）。

​		RPC机制就可以理解为Neutron的南向通信机制，Neutron的RPC实现基于AMPQ模型，plugins和agents之间通常采用“发布——订阅”模式传递消息，

agents收到相应plugins的***NotifyApi后，会回调设备本地的***CallBack来操作设备，完成业务的底层部署。

> Northbound Interface/Southbound Interface
> 北向接口：提供给其他厂家或运营商进行接入和管理的接口，即向上提供的接口。
> 南向接口：管理其他厂家网管或设备的接口，即向下提供的接口。

## 2. 设备端的实现

​		控制端neutron-server通过wsgi接收北向REST API请求，neutron-plugin通过rpc与设备端进行南向通信。设备端agent则向上通过rpc与控制端进行通信，向下则直接在本地对网络设备进行配置。
​    Neutron-agent的实现很多，彼此之间也没什么共性的地方，下面选取比较具有代表性的ovs-neutron-agent的实现进行简单的介绍。

​		Ovs-neutron-agent的启动入口为/neutron/plugins/openvswitch/agent/ovs-neutron-agent.py中的main方法，其中负责干活的两行代码在l 1471和l 1476。L 1471实例化了OVSNeutronAgent类，负责在本地配置OVS，而l 1476则启动了与控制端的rpc通信。

OVSNeutronAgent的实例化过程中依次干了6个工作：
1. 启动ovs     br-int网桥；
2. 启动rpc；
3. 启动ovs br-eth1；
4. 启动ovs br-tun；
5. 实例化OVSSecurityGroupAgent；
6. 开始rpc轮询与监听。

rpc_loop做的工作主要就是轮询一些状态，根据这些状态，进行相应的操作。
比如一旦探测到本地的OVS重启了（l 1295，l 1309），就重新创建本地的网桥（l 1294-1300），并重新添加port（l 1336）；
再比如一旦rpc监听到update_port事件（l 1309），则在本地使能相应的port（l 1336）。

ovs-neutron-agent的启动也就是这些工作了，启动完毕后，便开始了与相应plugin（OVS Plugin或者OVS Mechanism Driver）的rpc通信。

##  NuetronServer

 neutron-server的本质是一个Python Web Server Gateway Interface（WSGI），是通过eventlet lib来实现服务的异步并发模型的。实际工作时，通过'serve_wsgi'启动点(Entry point)来构造了一个NeutronApiService实例，通过该实例生成Eventlet，Greenpool来运行WSGI app并回应客户端请求。

研究之前，首先需要先了解下Entry point的概念。

```ini
[entry_points]
console_scripts =
    neutron-db-manage = neutron.db.migration.cli:main
    neutron-debug = neutron.debug.shell:main
    neutron-dhcp-agent = neutron.cmd.eventlet.agents.dhcp:main
```

## GRE模式

<img src="https://gitee.com/zbw2535463841/images-bed/raw/master/2021/12/18/image-20211215184611700.png" alt="image-20211215184611700" style="zoom:80%;" />

隧道桥（br-tun）根据 OpenFlow 规则将 VLAN 标记的流量从集成网桥转换为隧道 ID。

隧道桥允许不同网络的实例彼此进行通信。隧道有利于封装在非安全网络上传输的流量，它支持两层网络，即 GRE 和 VXLAN。

```bash
**# virsh dumpxml instance-000008ed |grep bridge**
  <interface type='bridge'>
   <source bridge='**qbrc68f8b26-23**'/>
  <interface type='bridge'>
   <source bridge='**qbr7cff5be4-5c**'/>
# brctl show
bridge name   bridge id        STP enabled   interfaces
qbrc68f8b26-23     8000.663c2f1e30b4    no       qvbc68f8b26-23
                                               tapc68f8b26-23   ◆instance port id
qbr7cff5be4-5c     8000.8a66ef285ffe    no       qvb7cff5be4-5c
                                            tap7cff5be4-5c     ◆ instance port id

#ovs-vsctl show
Bridge br-int
    Port "qvoc68f8b26-23"
      tag: 7
      Interface "qvoc68f8b26-23"
    Port "qvo7cff5be4-5c"
     tag: 6
      Interface "qvo7cff5be4-5c"
    Port patch-tun
      Interface patch-tun
       type: patch
        options: {peer=patch-int}

  Bridge br-tun
    Port br-tun
      Interface br-tun
        type: internal
    Port patch-int
      Interface patch-int
        type: patch
        options: {peer=patch-tun}
```

## vlan模式

<img src="https://gitee.com/zbw2535463841/images-bed/raw/master/2021/12/18/image-20211215190542910.png" alt="image-20211215190542910" style="zoom: 50%;" />

## 几个典型流程案例

> 同一个host上同一个子网内虚机之间的通信过程

<img src="https://gitee.com/zbw2535463841/images-bed/raw/master/2022/01/01/image-20211215184942367.png" alt="image-20211215184942367" style="zoom:50%;" />

因为br-int是个虚拟的二层交换机，所以同一个host上的同一个子网内的虚机之间的通信只是经过 br-int 桥，不需要经过 br-tun 桥。

> 不同主机上同一个子网内的虚机之间的通信过程

<img src="https://gitee.com/zbw2535463841/images-bed/raw/master/2022/01/01/image-20211215185016555.png" alt="image-20211215185016555" style="zoom:80%;" />

1. 从左边的虚机1出发的packet，经过Linux bridge到达br-int，被打上 VLAN ID Tag

2. 到达br-tun，将VLAN ID转化为Tunnel ID，从GRE Tunnel 发出，到达另一个compute节点

3. 在另一个compute节点上经过相反的过程，到达右边的虚机

> 虚机访问外网

<img src="https://gitee.com/zbw2535463841/images-bed/raw/master/2022/01/01/image-20211215185056636.png" alt="image-20211215185056636" style="zoom:80%;" />

1. Packet离开虚机，经过Linux bridge， 到达br-int，打上 VLAN ID Tag
2. 达到 br-tun，将 VLAN ID转化为 Tunnel ID
3. 从物理网卡进入GRE通道
4. 从GRE通道达到 Neutron 节点的网卡
5. 达到跟物理网卡相连的br-tun，将 Tunnel ID 转化为 VLAN ID
6. 达到 br-int，再达到 router，router的NAT 表 将 fixed IP 地址 转化为 floatiing IP 地址，再被route 到br-ex
7. 从br-ex相连的物理网卡上出去到外网
外网IP访问虚机是个相反的过程。

> 虚机发送DHCP请求

<img src="https://gitee.com/zbw2535463841/images-bed/raw/master/2022/01/01/image-20211215185221201.png" alt="image-20211215185221201" style="zoom:80%;" />

1. 虚机的packet -> br-int -> br-tun -> GRE Tunnel -> eth2 ------>eth2->br-tun->br-int->qDHCP
2. qDHCP返回其fixed IP地址，原路返回

 

> 不同tenant内虚机之间的通信

Neutron Tenant网络是为tenant中的虚机之间的通信。如果需要不同tenant内的虚机之间通信，需要在两个subnet之间增加Neutron路由。

## Neutron的租户隔离实现方案

> 数据面

![image-20211215185357857](https://gitee.com/zbw2535463841/images-bed/raw/master/2021/12/18/image-20211215185357857.png)

​    	br-ethx/tun、br-int分别只有一个用例，这个是属于：用“多租户共享”的方案，来实现多租户隔离。比如br-int、br-ethx通过VLAN来隔离各个租户网络数据流量，br-tun通过相应的tunnel来隔离各个租户网络的流量。
​	    qbr跟VM一一对应，这个属于：用“单租户独占”的方案，来实现“多租户隔离”。qbr由于绑定了安全组，它在原生的数据面租户隔离技术的基础上又叠加了一层“安全层”来保证租户隔离。原生的数据面转发（br-ethx/tun、br-int）负责“正常行为”的租户隔离，而安全技术（qbr）负责“异常行为”（非法访问）的租户隔离。
​	   Router/DVR跟租户相对应，而且每个Router/DVR运行在一个namespace中，这个属于：用“单租户独占”（用namespace隔离）的方案，来实现多租户隔离的目的。Router/DVR除了可以保证租户间网络不会互相访问以外，还解决了逻辑资源（IP地址）冲突的问题

> 管理面

共享数据库，共享表，通过表中字段（比如tenant_id来分区不同的租户）。
Neutron所采取就是第3种，所以说，它在数据库层面的租户隔离方案是比较弱的。
