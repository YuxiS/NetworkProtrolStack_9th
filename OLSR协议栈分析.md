<center> <font size=10>OLSR协议分析</font>  </center>
[TOC]

# 1. 概述

## 1.1 协议概述

<p style= "text-indent:2em">最佳链路状态路由协议（Optimized Link State Routing Protocol）是一个针对Ad-hoc网络需求的先应式的链路状态算法的优化协议。
<p style= "text-indent:2em">Ad-hoc网络是一种多跳，无中心的，自组织的无线网络，其不依赖于固定的网络核心，每个节点都是移动的。该网络的主要特点是自组织性，节点对等，分布式控制，临时性，网络拓扑结构变化快，带宽链路有限等。</p>
<p style="text-indent:2em">针对Ad-hoc网络的特点，该协议的核心主要是是MPR(Multipoint Relay)多点中继，每个节点选择选择一组其邻居节点作为MPR节点，然后只由这些MPR节点负责洪泛网络控制信息。MPR通过减少流量的传输次数，减少了网络中的信息量开销。</p>
<p style="text-indent:2em">该协议中，MPR节点负责周期性的广播路由更新，每个节点根据其他节点的控制信息，计算自己的网络拓扑，得到网络节点间的最短路径。该协议使用Dijkstra最短路径算法选择路径。</p>

## 1.2 主要算法描述

### 1.2.1 部分元组结构

**链接元组 ___(L_local_iface_addr, L_neighbor_iface_addr, L_SYM_time, L_ASYM_time, Ltime)___** 

**邻居元组 ___(N_neighbor_main_addr, N_2hop_addr,N_time)___**

**MPR选择分组 ___(MS_main_addr, MS_time)___** 

**拓扑分组 ___{T_test_addr,T_lase_addr, T_seq, T_time}___**

### 1.2.2链路感知

<p style="text-indent:2em">在Ad-hoc这种无线自组织网络，由于网络的动态性，有些连接可能是单向连接。OLSR规定只有双向对称的链路才能传播信息，每个节点要建立与其他节点的对称链路，必须进行链路检测。</p>
<p style="text-indent:2em">节点链路感知是要通过互相发送HELLO分组来实现的，本地链路信息库存储本节点到邻居节点的链接信息。当收到一个HELLO分组时，一个节点应该更新其链路集，收到一个HELLO分组后，首先计算该分组的有效事件，来确认该分组是否有效，然后更新链路信息，更新链路信息的主要流程如下：</p
* 收到一个HELLO分组，如果不存在如下一条链路记录元组：

​		<code>L_neighbor_iface_addr == 该分组的Originator Address</code>

必须加入在链接信息库中新加入：

​		<code>  L_beighbor_iface_addr  = 该分组的Originator Address</code>>

​        <code>L_local_iface_addr          = 该分组的接口地址</code>

​		<code>L_SYM_time                     = Current time - 1</code>

​		<code>L_time                               =current time + validity time</code>  

* 如果该元组已经存在,则按照如下规则修改链路信息集：

<code>L_ASYM_time = current time + validity time</code>

如果接口地址在HELLO分组链路信息列表中，则如下更新：

		* 如果链接状态是LOST_LINK,则：

​		<code>L_SYM_time = current time + validity time</code>

 * 如果链接状态是SYM_LINK或者ASYM_LINK,则：

​		<code>L_SYM_time = current time + validity time</code>

​		<code>L_time = L_SYM_time+NEIGHB_HELLO_TIME</code>

<code> L_time = max(L_time, L_ASYM_time)</code>

### 1.2.3 邻居检测

<p style="text-indent:2em">邻居检测会通过改变邻居信息库来记录邻居信息，而且这种检测主要和节点与节点间的主地址相关。OLSR中的邻居检测机制是通过定期交换HELLO分组实现的。</p>
#### 1. 扩充邻居集

<p style="text-indent:2em">节点基于链接元组集来维护邻居信息元组集，随着链接信息集的更新来更新这些信息。</p>
<p style="text-indent:2em">链接集保存链路的链接信息，邻居集保存邻居信息，这两个集合之间有明确的关系，当一个节点是另一个节点的邻居节点时，这两个节点间至少应该有一条双向链接链路。</p>
<p style="text-indext:2em">邻居和链接只间的正是对应关系如下:</p>

* 如果一个链接元组中的关联邻居元组存在，邻居元组为：

  ​	<code>N_neighbor_main_addr ==L_neightbor_iface _addr的主地址</code> 

* 如果邻居元组的关联链接元组都是链接元组，并且：

  ​	<code>N_neighbor_main_addr == L_neighbot_iface _addr的主地址</code>	

  则邻居集必须通过维护链接元组和关联邻居元组之间对应关系来填充，填充规则如下：

  * 创建

    每次出现一个链接时，必须创建关联的邻居元组，如果邻居元组不存在，则：

    <code>N_neighbor_main_addr = Lneighbor_iface_addr 的主地址</code>

  * 更新

    每次链接信息更改时，节点必须确保相关邻居元组的N_status遵循：

    ​		如果邻居有任何关联元组代表一个对称链接，则：

    ​		<code>N_status = SYM</code>

    ​		否则：

    ​		<code>N_status = NOT_SYM</code>

  * 删除

    每次删除一个链接时，相关的链接元组必须被删除，相关的邻居元组也必须被删除。

  这些规则都是为了确保存在一个唯一的相关邻居元组对应一个链接元组，并且每个邻居元组至少对应一个关联元组。

* HELLO分组处理

  <p style="text-indent:2em">一个HELLO分组的Original Address是该分组的发出者地址。同时，根据HELLO的willingness字段重新计算willingness。收到一个HELLO分组后，节点需要更新自己的链接集和邻居集。</p>

  * 如果发出者地址是邻居集中邻居元组中的N_neighbor_main_addr，则邻居元组应该更新：

    ​	<code>N_willingness = HELLO分组中的willingness</code>

#### 2. 扩充二跳邻居集

	<p style="text-indent:2em">二跳邻居集描述的是对称邻居的对称链接信息，其依然通过互相交互HELLO分组来维护。</p>

<p style="text-indent:2em">从对称邻居接受到一个全新的HELLO分组后，一个节点应该更新其二跳邻居集。接受到一个HELLO分组，首先通过Vtime计算有效时，检测其是否有效。如果该分组的发出者地址是包含着链接集中链接元组中的L_neighbor_iface_addr的一个主地址，如果 <code>L_SYM_time >= current time</code> 则该分组未过期。</p>

* 对于每一个在HELLO中分组中且邻居类型为SYM_NEIGH或MPR_NEIGH的地址：

   * 如果一个二跳邻居节点的主地址不等于接受节点的主地址，则丢弃该二跳邻居。

   * 否则，新创建一个二跳邻居元组：

     <code>N_neighbor_main_addr = 发出者地址</code>

     <code>N_2hop_addr = 二跳邻居的主地址</code>

     <code>N_time = current time+validity time</code>

     这个新分组用于替换和其具有相同N_neighbor_main_addr 和N_2hop_addr值的元组。

* 对于每一个在HELLO中分组中且邻居类型为NOT_NEIGH的地址，所有的二跳邻居应该是：

  ​	<code>N_neighbor_main_addr == 发出者地址 and</code> 

  ​	<code>N_2hop_addr == 二跳邻居的主地址</code>

###  1.2.4 MPR

<p style="text-indent:2em">MPR用于洪泛网络控制分组，主要是减少分组传输过程中的重传次数，因此，该机制是对传统洪泛机制的优化。</p>

<p style="text-indent:2em">网络中的每个节点独立地从一跳对称邻居中选择自己的MPR集，和邻居节点中MPR的对称链路在HELLO分组中的链路类型为MPR_NEIGH。</p>

 <p style="text-indent:2em">每个节点的每个接口都计算自己的MPR集，所有接口的MPR集的并集为该节点的MPR集。</p>

## 1.3 OLSR协议优缺点

* OLSR协议是一种先应式路由协议，具有查找路由延时小的优点。
* OLSR用于移动自组织网络，MPR的实现极大的减小了网络中的控制流量，特别适用大型密集型移动网络，而且网络越大，效果越好。
* OLSR最初就是为以完全分布式方式工作，不依赖任何中心实体。
* OLSR不需要对IP数据包做任何修改。

# 代码介绍

