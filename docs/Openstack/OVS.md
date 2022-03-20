OpenFlow是用于管理交换机流表的协议，ovs-ofctl是Open vSwitch提供的命令行工具。

在OpenFlow白皮书中，Flow被定义为某个特定的网络流量。例如，一个TCP连接就是一个Flow，或者从某个IP地址发出来的数据包，都可以被认为是一个Flow。

支持OpenFlow协议的交换机应该包括一个或多个流表，流表中的条目包含：

数据包头的信息、匹配成功后要执行的指令和统计信息。

 当数据包进入OVS后，会将数据包和流表中的流表项进行匹配，如果发现了匹配的流表项，则执行该流表项中的指令集。相反，如果数据包在流表中没有发现任何匹配，OVS会通过控制通道把数据包发到OpenFlow控制器中。

在OVS中，流表项作为ovs-ofctl的参数，采用如下的格式：字段=值，如果有多个字段，可以用逗号或空格分开，一些常用的字段列举如下表所示：

![image-20211215190706940](https://gitee.com/fjboy/cdn/raw/image-bed/2021/12/18/image-20211215190706940.png)

##  OVS基本命令分类

 Open vSwitch中有多个命令，分别有不同的作用，大致如下：

- ovs-vsctl用于控制ovs db
- ovs-ofctl用于管理OpenFlow     switch 的 flow
- ovs-dpctl用于管理ovs的datapath
- ovs-appctl用于查询和管理ovs     daemon