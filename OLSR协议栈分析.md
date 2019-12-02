<center style:"font:100"> OLSR协议分析  </center>



# 概述

<p style= "text-indent:2em">最佳链路状态路由协议（Optimized Link State Routing Protocol）是一个针对Ad-hoc网络需求的先应式的链路状态算法的优化协议，该协议的核心是MPR(Multipoint Relay)多点中继，每个节点选择选择一组其邻居节点作为MPR节点，然后由这些MPR节点负责转发网络控制流量,MPR通过减少流量的传输次数，减少了网络中的信息量开销。</p>

<p style="text-indent:2em">该协议中，MPR节点负责周期性的广播路由更新，每个节点根据其他节点的控制信息，计算自己的网络拓扑，得到网络节点间的最短路径。该协议使用Dijkstra最短路径算法选择路径。</p>

### OLSR的优点和局限性：

* OLSR协议是一种先应式路由协议，具有查找路由延时小的优点。
* OLSR用于移动自组织网络，MPR的实现极大的减小了网络中的控制流量，特别适用大型密集型移动网络，而且网络越大，效果越好。
* OLSR最初就是为以完全分布式方式工作，不依赖任何中心实体。
* OLSR不需要对IP数据包做任何修改。

