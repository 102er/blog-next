---
title: 服务部署nginx+keepalived
date: 2020-05-19 14:27:22
categories:
    - 服务部署
tags:	
  - 负载均衡
---

## 介绍

使用nginx+keepalived实现服务的高可用和负载均衡，解决服务负载以及服务可用性

- nginx：充当着网络流中“交通指挥官”的角色，“站在”服务器前处理所有服务器端和客户端之间的请求，从而最大程度地提高响应速率和容量利用率，同时确保任何服务器都没有超负荷工作。如果单个服务器出现故障，负载均衡的方法会将流量重定向到其余的集群服务器，以保证服务的稳定性。nginx支持http代理，也支持udp代理。
- Keepalived：保证nginx服务高可用，会实时检测服务器的状态，如果有一台web服务器死机，或工作出现故障，Keepalived将检测到，并将有故障的服务器从系统中剔除，同时使用其他服务器代替该服务器的工作，当服务器工作正常后Keepalived自动将服务器加入到服务器群中

服务部署如图：

![osi](https://102er.github.io/uploads/nginx-keepalived.png)

<!-- more -->

## nginx负载均衡算法

- **轮询**：负载均衡中较为基础也较为简单的算法，它不需要配置额外参数。假设配置文件中共有 $M$ 台服务器，该算法遍历服务器节点列表，并按节点次序每轮选择一台服务器处理请求。当所有节点均被调用过一次后，该算法将从第一个节点开始重新一轮遍历。
- **加权轮询**：为了避免普通轮询带来的弊端，加权轮询应运而生。在加权轮询中，每个服务器会有各自的 `weight`。一般情况下，`weight` 的值越大意味着该服务器的性能越好，可以承载更多的请求。该算法中，客户端的请求按权值比例分配，当一个请求到达时，优先为其分配权值最大的服务器。
- I**P哈希**：`ip_hash` 依据发出请求的客户端 IP 的 hash 值来分配服务器，该算法可以保证同 IP 发出的请求映射到同一服务器，或者具有相同 hash 值的不同 IP 映射到同一服务器。

### KeepAlived

Keepalived是以VRRP协议为实现基础的，VRRP全称`Virtual Router Redundancy Protocol`，即`虚拟路由冗余协议`。可以认为是实现路由器高可用的协议，即将N台提供相同功能的路由器组成一个路由器组，这个组里面有一个master和多个backup，master上面有一个对外提供服务的vip，master会发组播，当backup收不到vrrp包时就认为master宕掉了，这时就需要根据VRRP的优先级来选举一个backup当master。选举策略是根据 VRRP 协议，完全按照权重大小，权重最大（0～255）的是 MASTER 机器，下面几种情况会触发选举：

1. keepalived 启动的时候
2. master 服务器出现故障（断网，重启，或者本机器上的 keepalived crash 等，而本机器上其他应用程序 crash 不算）
3. 有新的备份服务器加入且权重最大

配置keepalived需要注意：priority和weight值的设定应遵循: abs(MASTER priority - BAKCUP priority) < abs(weight)。一般情况下，初始值MASTER的priority值应该比较BACKUP大，但不能超过weight的绝对值。 另外，当网络中不支持多播(例如某些云环境)，或者出现网络分区的情况，keepalived BACKUP节点收不到MASTER的VRRP通告，就会出现脑裂(split brain)现象，此时集群中会存在多个MASTER节点。

keepalived主要有三个模块:

- core模块为keepalived的核心，负责主进程的启动、维护以及全局配置文件的加载和解析。
- check负责健康检查，包括常见的各种检查方式。
- vrrp模块是来实现VRRP协议的。

#### 脑裂问题

同时在keepalived高可用集群中，出现了两个虚拟IP地址信息，这种情况就称为脑裂

### 1. 脑裂情况出现原因：

- 网络问题，心跳线出现问题，交换设备问题，防火墙限制等。
- virtual_router_id配置数值不正确

总之：只要备服务器收不到组播包，就会成为主，而主资源没有释放，就会出现脑裂
