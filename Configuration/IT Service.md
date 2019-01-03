# IT Service
## 概述
IT services 从较高的角度俯瞰IT基础架构. 很多时候，我们对于低级别的监控细节并不感兴趣：缺少磁盘空间、CPU负载高等。我们感兴趣的是IT部门对外提供的整体服务可用性。我们可能感兴趣的是辨别IT基础架构的薄弱环节, SLA有助于我们了解这些内容.
Rems2监控系统的 IT services 用来解决此类型问题.
IT services 其实就是监控数据的层次化展现。.
一个常见的 IT service 结构如下所示:
``` IT Service
```|
```|-Workstations
```| |
```| |-Workstation1
```| |
```| |-Workstation2
```|
```|-Servers

结构中的每一个节点都有其自身状态. 每个节点的状态，向上影响SLA的计算. IT Service最底层的实现依赖于触发器. 
可以定义运行维护时间——运维时间内，不计算IT Service SLA。