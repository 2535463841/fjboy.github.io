Nova是OpenStack云中的计算组织控制器。

支持OpenStack云中实例（instances）生命周期的所有活动都由Nova处理。这样使得Nova成为一个负责管理计算资源、网络、认证、所需可扩展性的平台。

<img src="https://gitee.com/zbw2535463841/images-bed/raw/master/2021/12/18/image-20211215191246850.png" alt="image-20211215191246850" style="zoom: 80%;" />

 

1. Nova API     ：HTTP服务，用于接收和处理客户端发送的HTTP请求

2. Nova Compute     ：Nova组件中最核心的服务，实现虚拟机管理的功能。

    实现了在计算节点上创建、启动、暂停、关闭和删除虚拟机、虚拟机在不同的计算节点间迁移、虚拟机安全控制、管理虚拟机磁盘镜像以及快照等功能。

3. Nova Cert     ：用于管理证书，为了兼容AWS。AWS提供一整套的基础设施和应用程序服务，使得几乎所有的应用程序在云上运行;

4. Nova Conductor     ：RPC服务，主要提供数据库查询功能。

5. Nova Scheduler     ：Nova调度子服务。当客户端向Nova 服务器发起创建虚拟机请求时，决定创建在哪个节点上。

6. Nova Console、Nova     Consoleauth、Nova VNCProxy ：Nova控制台子服务。

    功能是实现客户端通过代理服务器远程访问虚拟机实例的控制界面。



## Nova-scheduler

Openstack中会由多个Instance共享同一个Host，需要提供调度服务来协调和管理Instance之间的资源分配。nova-scheduler在创建实例的时候，为实例(Instance)选择出合适的主机（host）

> 调度器

调度器均继承 /nova/scheduler/driver.py中的Scheduler类

• ChanceScheduler(随机调度器)：从所有正常运行nova-compute服务的HostNode中随机选取来创建Instance
• FilterScheduler(过滤调度器)：根据指定的过滤条件以及权重来挑选最佳Instance的Host Node。
Caching(缓存调度器)：是过滤调度器的一种，在其基础上将Host资源信息缓存到本地的内存中，然后通过后台的定时任务从数据库中获取最新的Host资源信息。
