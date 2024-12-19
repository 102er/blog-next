## **介绍**

哨兵（sentinel），是一个分布式系统，用于对主从结构中的每一台服务器进行监控，当master节点出现故障后通过投票机制来挑选新的master节点 ，并且讲所有的salve节点连接到master节点上。

redis哨兵（sentinel）是运行在特殊模式下的redis服务器，不支持读写操作，它的作用是配合redis的复制功能，实现对主从节点的监控，对下线的master节点进行故障转移和通知，确保整个redis系统的可用性。具备几个能力：

- 监控：持续监控master、slave是否处于预期工作状态
- 自动切换主库：当master节点运行故障，哨兵启动自动故障恢复流程：从slave中选择一台作为新的master
- 通知：让slave执行replicaof，和新的master同步，并且通知客户端和新master建立连接

## **架构**

<img src="redis-sentinel.svg" alt="架构图" style="zoom:50%;" />

## 哨兵

### **监控**

sentinel定期(1s)向master、slave和其他sentinel发送ping命令，回复有两种情况：

- 有效回复：返回+PONG、
- 无效回复：有效回复之外的回复，或者指定啥时间内返回任何回复

sentinel对探测结果的处理

- slave没有在规定时间响应sentinel的ping命令，sentinel就认为slave节点嗝屁了，就会将他记录为【下线状态】
- master没有在规定时间内响应sentinel的ping命令，sentinel就判定master下线，开始执行【自动切换master】的流程

为了防止master节点假死，sentinel设计了【主观下线】和【客观下线】两个状态。

#### **主观下线**

哨兵对节点进行探测，没有得到有效回复的时候，就会对节点标记【主观下线】，执行**<u>+sdown</u>**

如果是slave节点，对集群没有影响，不会进行下一步动作。

如果是master节点，哨兵会确认master节点是否真的下线，并执行故障转移。

#### **客观下线**

哨兵确认master节点是否真的宕机，需要进行投票，所有哨兵达成共识（哨兵投票，如果票数大于一半，且大于等于**`quorum`**设置的数量，就是投票成功）认为master节点【主观下线】了，哨兵才能确认master节点【客观下线】，执行**<u>+odown</u>**

```
# sentinel.conf
sentinel monitor <master-name> <ip> <redis-port> <quorum>

# sentinel monitor：标识哨兵要监控哪个master节点
# master-name：redis的名称，自定义
# ip：redis master节点ip
# redis-port：redis master节点端口
# quorum：法定数量，代表>=N个哨兵认为主节点不可用的时候，才会把master节点设置为客观下线状态
```

**客观下线投票过程**



*如果多个哨兵在同一时间内发现master节点主观下线，那么每个发现的哨兵都会发起投票，投票的结果真是让发起投票的哨兵能够确认master节点是否下线，并不会与其他哨兵共享。因此这个下线确认的动作可能是多个哨兵同时发起进行的。*

#### **主观下线和客观下线的区别**

主观下线是哨兵认为节点宕机了，而客观下线是不但哨兵认为节点宕机了，而且该哨兵和其他哨兵达成了共识，一致认为节点宕机了。

