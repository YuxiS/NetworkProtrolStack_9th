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

**MPR选择元组 ___(MS_main_addr, MS_time)___** 

**拓扑元组 ___{T_test_addr,T_lase_addr, T_seq, T_time}___**

**路由表项元组__{R_dest_addr, R_next_addr, R_dist, R_iface_addr}__**

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
<p style="text-indent:2em">MPR集应满足以下两点要求：节点与MPR之间必须是一跳双向链路；节点能通过MPR集到达所有的严格两跳邻居节点。</p>
#### 1. MPR计算

* 从一个节点的子集开始，当N_willness等于WILL_ALWAYS该子集中成员属于一个MPR集。

* 计算该子集中所有节点的一跳邻居集。

* 添加节点，当且仅当该节点对另一子集中节点可达时。例如另一个子集中的b节点通过对称链路可访问到节点该子集中的a节点时，向MPR集中添加a节点，同时去除另一子集中的节点。

* 当另一子集中存在未被当前MPR中节点覆盖节点时:

  * 计算当前子集中的每一个节点的可达性。即另一子集中尚未被当前子集中的节点覆盖，并且通过一跳邻居能到达的节点。
  * 在当前子集中选择一个拥有最大N_willingness，且可达节点不为0的节点。在这种情况下，选择的几点应该具有对另一子集节点的最大可达性。(即拥有对另一子集可达节点最多)在这种当多个节点拥有相同最大可达性的情况下，选择属于MPR集，且拥有最大一跳邻居集的节点。然后删除被该节点覆盖掉的另一子集中的节点。

* 从已有的MPR集中生成每个节点的MPR集。

  对于上述算法，还可以有部分优化。

#### 2. MPR selector扩充计算

​	节点的MPR selector集通过其他将同一子集选为初始MPR集的节点的主地址来扩充。MPR选择过程通过HELLO消息的交互来实现。

* HELLO消息处理

  一个节点接受一个HELLO分组后，如果发现本节点的其中一个接口地址在邻居类型等于MPR_NEIGH的列表中时，该HELLO分组携带的信息必须被重新记录在MPR Selector集中。

  Validity time必须通过HELLO分组的Vtime字段重新计算。MPR selector集更新规则如下：

  1. 如果不存在具有以下内容MPR selector元组：

     <code>MS_main_addr == 发出者地址</code>

     则创建一个新元组：

     <code>MS_main_addr = 发出者地址</code>

  2. 如果元组：

     <code>MS_main_addr == 发出者地址</code>

     则修改MS_time字段:

     <code>MS_time = current time + validity time</code>

#### 3. 邻居和二跳邻居变化

​	邻居节点的变化情况将在以下情况下被检测到:

* 如果链接元组的L_SYM_Time字段到期。

* 在链接集中插入一个新的链接元组，其中L_SYM_Time字段未过期，或者L_SYM_Time被修改避免过期。

当邻居集或者二跳邻居集被检测到发生变化，则执行以下流程：

* 如果有邻居丢失，则所有<code>N_neighbor_main_addr==Main Address</code>的二跳元组必须被删除。
* 如果有邻居丢失，所有<code>MS_main_addr == Main Address</code>的MPR selector元组必须被删除。
* 当有邻居或者二跳邻居改变，丢失，或者删除时，MPR集必须重新计算。
* 当MPR集发生改变时，可能发出一个额外的HELLO分组。

### 1.2.5 拓扑发现

<p style="text-indent:2em">前面的链路检测和邻居检测部分为每个节点提供了能够直接通信的邻居列表，并且合并了数据包的格式和通过MPR优化的前向路由洪泛算法。拓扑信息基于此在网络中得以传播。</p>
<p style="text-indent:2em">网络的路由结构是通过广播链路来实现的。一个节点必须至少传播其本身和MPR selector集中的信息，让网络中有充分的信息构建路由表。</p>
#### 1. 广播邻居集

<p style="text-indent:2em">一个节点发出TC分组声明链接集，称为广播链接集。该行为必须至少包含到其MPR selector集的所有链接信息。</p>
<p style="text-indent:2em">与广播邻居集相关联的序列号(ANSN)也应该伴随发出。当链接从广播的邻居集中移除时，ANSN必须增加。当有链接加入邻居集时，ANSN也应该增加。</p>
#### 2. TC分组生成

<p style="text-indent:2em">为了构建拓扑信息集，每一个被选为MPR的节点，必须广播TC分组。TC分组通过广播方式传播到网络中的每一个节点。MPR在拓扑信息的分散上有良好的扩展性。</p>
<p style="text-indent:2em">地址是TC分组的一部分。所有TC分组的解析必须在一个确定的更新周期内完成。这些TC分组携带的信息将协助网络中所有节点完成路由表的计算。</p>
<p style="text-indent:2em">当一个节点的可广播链接集为空时，其依然应该在之前发出TC分组的validity time时间内持续发送空的TC分组消息。以使之前发送的TC分组失效。然后其应该停止发送TC分组，直到有某个节点加入该节点的广播链接集。</p>
<p style="text-indent:2em">一个节点能够传递附加的TC分组以增强对链路故障的反应性。当MPR selector集的变化被检测到，且该变化可能导致链路故障时，节点应该在短于TC_INTERVAL的时间内传递一个TC分组。</p>
#### 3. TC分组转发

<p style="text-indent:2em">TC分组必须由MPR节点广播转发到整个网络。</p>
#### 4. TC分组处理

* 当接受到一个TC分组后，必须先通过其头部的Vtime字段计算其有效时间然后，拓扑集根据以下规则更新：
  
* 如果此分组的发送者接口不在本节点的一跳对称邻居中，则丢弃该分组。
  
* 如果拓扑信息集中存在一些元组：

  <code>T_last_addr == 发出者地址 AND</code>

  <code>T_seq > ANSN</code>

  则不对该分组做任何进一步处理，并且丢弃该分组。

* 如果拓扑集中所有的元组都是:

  <code>T_last_addr == 发出者地址 AND</code>

  <code>T_seq < ANSN</code>

  则必须从拓扑集中删除。

* 对于每一个在TC分组中受到的被广播的邻居的主地址：

  * 如果在拓扑集中存在一些元组：

    <code> T_test_addr == 广播邻居的主地址 AND</code>

    <code>T_last_addr == 发出者地址</code>

    则该元组的有效时间必须被设为：

    <code>T_time = current time + validity time</code>

  * 否则，必须在拓扑集中记录一条新的原则记录:

    <code>T_dest_addr = 被广播邻居主地址</code>

    <code>T_last_addr = 发出者地址</code>

    <code>T_seq = ANSN</code>

    <code>T_time = current time + validity time</code>

  ### 1.2.6 路由表计算

  <p style="text-indent:2em">每个节点都拥有一个路由表，通过路由表节点能够让网络中其他节点发送信息.路由表是基于本地的链接信息集和拓扑集构建的，所以这两个集合一旦有任何一个发生变化，路由表都需要重新计算。</p>
  * 当以下任何一个有变化时，都需要更新路由表:
    * 链路集
    * 邻居集
    * 拓扑集
    * 二条邻居集
    * 多接口关联信息集

  路由表计算方法如下：

  * 删除路由表中所有条目

  * 新增加路由表项从以对称邻居作为目的节点开始。因此，对于邻居集中的每个邻居元组，都有:

    <code>N_status = SYM</code>

    并且，对于邻居节点的每个关联链接元组，都有<code>L_time >= current time</code>,路由表中新加的表项为:

    <code>R_dest_addr = 关联链接元组的L_neighbor_iface_addr</code>

    <code>R_next_addr = 关联链接元组的L_neighbor_iface_addr</code>

    <code> R_dist = 1</code>

    <code>R_iface_addr = 关联链接元组的L_local_iface_addr</code>

    如果按照上述情况，没有R_dest_addr等于邻居节点的主地址，则必须添加新的路由表项:

    <code> R_dest_addr = 邻居的Main address</code>

    <code>R_next_addr = L_time>0的关联链接元组的L_neighbor_iface_addr</code>

    <code>R_dist = 1</code>

    <code>R_iface_addr = 关联链接元组的L_local_iface</code>

  * 对于严格二条邻居节点，如果二跳邻居集中至少存在一项记录中N_neighbor_main_addr对于一个willingness不为WILL_NEVER的节点，选择一个二跳邻居节点，并在路由表中添加新表项：

    <code>R_dest_addr = 二跳邻居的Main Address</code>

    <code>R_next_addr = 路由表项中R_dest_addr等于二跳邻居N_neighbor_main_addr的记录的R_next_addr</code>

    <code>R_dist = 2 </code>

    <code>R_ifacce_addr = 路由表中R_dest_addr等于二跳元组中N_neighbor_main-addr的记录的R_iface_addr</code>

  * 对于h+1跳的节点，按照Dijkstra算法加入路由表中
  
  * 对于多接口关联信息集中的每个实体，如果不存在一个路由表项：
  
    <code>R_dest_addr == I_main_addr</code>
  
    并且，也没有表项：
  
    <code>R_dest_addr == I_iface_addr</code>
  
    则添加一个新表项：
  
    <code>R_dext_addr = I_iface_addr</code>
  
    <code>R_dext_addr = R_next_addr</code>
  
    <code>R_dist = R_dist</code>
  
    <code> R_iface_addr = R_iface_addr</code>

## 1.3 OLSR的优点和局限性
* OLSR协议是一种先应式路由协议，具有查找路由延时小的优点。

* OLSR用于移动自组织网络，MPR的实现极大的减小了网络中的控制流量，特别适用大型密集型移动网络，而且网络越大，效果越好。

* OLSR最初就是为以完全分布式方式工作，不依赖任何中心实体。
* OLSR不需要对IP数据包做任何修改。

# 2.代码概述

