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
  
  路由表的更新既不在网络中，也不在一跳邻居中生成或触发任何消息。

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

# 2.代码分析

### 2.1 文件介绍

### 2.2 数据结构

### 2.3 邻居检测

### 2.4 MPR

### 2.5 拓扑发现

网络中的MPR节点每隔一段时间会广播TC分组，用以维护网络的拓扑信息。在算法描述中已经说明，对于相同的TC分组,节点只有在第一次收到且其选择为MPR的情况下才转发，这样减少了网络中的广播包的数量，尽可能的避免了网络风暴。

#### 2.5.1 TC分组生成

___

```c
81	void
82	generate_tc(void *p)
83	{
84	  struct tc_message tcpacket;
85	  struct interface *ifn = (struct interface *)p;
86
87	  olsr_build_tc_packet(&tcpacket);
88
89	  if (queue_tc(&tcpacket, ifn) && TIMED_OUT(ifn->fwdtimer)) {
90		set_buffer_timer(ifn);
91	  }
92
93	  olsr_free_tc_packet(&tcpacket);
94	}
```

____

首先构建一个TC分组的结构体，然后使用olsr_build_tc_packet（）函数对该结构体进行一些初始化和赋值操作，然后queue_tc（）函数将该分组加入MID队列中，同时TIMED_OUT()检测接口的时间的时间戳是否已满，调用set_buffer_timer()设置定时器。最后由接口ifn释放该分组，然后调用olsr_free_tc_packet()函数释放内存。

#### 2.5.2 拓扑信息集的初始化

____

```c
185	void
186	olsr_init_tc(void)
187	{
188	  OLSR_PRINTF(5, "TC: init topo\n");
189
190	  avl_init(&tc_tree, avl_comp_default);
191
192	  /*
193	   * Get some cookies for getting stats to ease troubleshooting.
194	   */
195	  tc_edge_gc_timer_cookie = olsr_alloc_cookie("TC edge GC", OLSR_COOKIE_TYPE_TIMER);
196	  tc_validity_timer_cookie = olsr_alloc_cookie("TC validity", OLSR_COOKIE_TYPE_TIMER);
197
198	  tc_edge_mem_cookie = olsr_alloc_cookie("tc_edge_entry", OLSR_COOKIE_TYPE_MEMORY);
199	  olsr_cookie_set_memory_size(tc_edge_mem_cookie, sizeof(struct tc_edge_entry) + active_lq_handler->tc_lq_size);
200
201	  tc_mem_cookie = olsr_alloc_cookie("tc_entry", OLSR_COOKIE_TYPE_MEMORY);
202	  olsr_cookie_set_memory_size(tc_mem_cookie, sizeof(struct tc_entry));
203
204	  /*
205	   * Add a TC entry for ourselves.
206	   */
207	  tc_myself = olsr_add_tc_entry(&olsr_cnf->main_addr);
208	}
```

____

6:avl_init()初始化拓扑信息集为avl树。

11-18: 对拓扑表集合进行初始化，主要通过从cookie中获取的值为拓扑表集合的属性做初始化赋值。

23:调用olsr_add_tc_entry（）配置一个entry,并将该entry加入avl树中。

#### 2.5.3 TC分组处理

____

```c
809	  /* We are only interested in TC message types. */
810	  pkt_get_u8(&curr, &type);
811	  if ((type != LQ_TC_MESSAGE) && (type != TC_MESSAGE)) {
812		return false;
813	  }
814
815	  /*
816	   * If the sender interface (NB: not originator) of this message
817	   * is not in the symmetric 1-hop neighborhood of this node, the
818	   * message MUST be discarded.
819	   */
820	  if (check_neighbor_link(from_addr) != SYM_LINK) {
821		OLSR_PRINTF(2, "Received TC from NON SYM neighbor %s\n", olsr_ip_to_string(&buf, from_addr));
822		return false;
823	  }
```

____

当节点接受到一个TC分组后，只考虑其消息类型。如果类型不等于LQ_TC_MESSAGE或者TC_MESSAGE，则直接丢弃。

如果检测到该消息的发送者接口不是本节点的对称一跳邻居，则丢弃该分组。

____

```c
847    if (olsr_seq_inrange_high((int)tc->msg_seq - TC_SEQNO_WINDOW, tc->msg_seq, msg_seq)
848        && olsr_seq_inrange_high((int)tc->ansn - TC_ANSN_WINDOW, tc->ansn, ansn)) {
849
850      /*
851       * Ignore already seen seq/ansn values (small window for mesh memory)
852       */
853      if ((tc->msg_seq == msg_seq) || (tc->ignored++ < 32)) {
854        return false;
855      }
856
857      OLSR_PRINTF(1, "Ignored to much LQTC's for %s, restarting\n", olsr_ip_to_string(&buf, &originator));
```

____

如果分组中的msg_seq和外部变量msg_seq相等，且ignored小于32，说明该分组已经处理过，所以丢弃该分组，返回false。

____

```c
878	/*
879	   * Generate a new tc_entry in the lsdb and store the sequence number.
880	   */
881	  if (!tc) {
882		tc = olsr_add_tc_entry(&originator);
883	  }
884
885	  /*
886	   * Update the tc entry.
887	   */
888	  tc->msg_hops = msg_hops;
889	  tc->msg_seq = msg_seq;
890	  tc->ansn = ansn;
891	  tc->ignored = 0;
892	  tc->err_seq_valid = false;
893
894	  OLSR_PRINTF(1, "Processing TC from %s, seq 0x%04x\n", olsr_ip_to_string(&buf, &originator), tc->msg_seq);
```

____

如果现有的拓扑集中没有当前收到的TC分组中的Main Address，则添加一条新纪录，用TC分组中获取到的信息为该记录赋值。

____

```c
915	 /*
916	   * Calculate real border IPs.
917	   */
918	  if (borderSet) {
919		borderSet = olsr_calculate_tc_border(lower_border, &lower_border_ip, upper_border, &upper_border_ip);
920	  }
921
922	  /*
923	   * Set or change the expiration timer accordingly.
924	   */
925	  olsr_set_timer(&tc->validity_timer, vtime, OLSR_TC_VTIME_JITTER, OLSR_TIMER_ONESHOT, &olsr_expire_tc_entry, tc,
926					 tc_validity_timer_cookie);
927
928	  if (emptyTC && lower_border == 0xff && upper_border == 0xff) {
929		/* handle empty TC with border flags 0xff */
930		memset(&lower_border_ip, 0x00, sizeof(lower_border_ip));
931		memset(&upper_border_ip, 0xff, sizeof(upper_border_ip));
932		borderSet = 1;
933	  }
```

____

调用olsr_calculate_tc_border()计算borderset的值，并且重置相关的定时器。

#### 2.5.4 拓扑信息集的删除

____

```c
278	void
279	olsr_delete_tc_entry(struct tc_entry *tc)
280	{
281	  struct tc_edge_entry *tc_edge;
282	  struct rt_path *rtp;
283	#if 0
284	  struct ipaddr_str buf;
285	  OLSR_PRINTF(1, "TC: del entry %s\n", olsr_ip_to_string(&buf, &tc->addr));
286	#endif
287
288	  /* delete gateway if available */
289	#ifdef LINUX_NETLINK_ROUTING
290	  olsr_delete_gateway_entry(&tc->addr, FORCE_DELETE_GW_ENTRY);
291	#endif
292	  /*
293	   * Delete the rt_path for ourselves.
294	   */
295	  olsr_delete_routing_table(&tc->addr, olsr_cnf->maxplen, &tc->addr);
296
297	  /* The edgetree and prefix tree must be empty before */
298	  OLSR_FOR_ALL_TC_EDGE_ENTRIES(tc, tc_edge) {
299		olsr_delete_tc_edge_entry(tc_edge);
300	  } OLSR_FOR_ALL_TC_EDGE_ENTRIES_END(tc, tc_edge);
301
302	  OLSR_FOR_ALL_PREFIX_ENTRIES(tc, rtp) {
303		olsr_delete_rt_path(rtp);
304	  } OLSR_FOR_ALL_PREFIX_ENTRIES_END(tc, rtp);
305
306	  /* Stop running timers */
307	  olsr_stop_timer(tc->edge_gc_timer);
308	  tc->edge_gc_timer = NULL;
309	  olsr_stop_timer(tc->validity_timer);
310	  tc->validity_timer = NULL;
311
312	  avl_delete(&tc_tree, &tc->vertex_node);
313	  olsr_unlock_tc_entry(tc);
314	}
```

____

该函数的功能是删除一个tc_entry.

如果预定义了 LINUX_NETLINK_ROUTING(即在linux系统上运行)，则删除网关信息。删除时对网关的时间信息，网关协议等等先进行判断，判断这些信息是否为空后再进行删除。

首先删除本地路由表中的rt_path；

清空所有的边，停止相应的计时器，将edge_gc_timer和validity_time属性都置为空。

最后在avl树中删除相应节点。

### 2.6 路由表计算

#### 2.6.1 相关结构体

____

```c
85	/* a composite metric is used for path selection */
86	struct rt_metric {
87	  olsr_linkcost cost;
88	  uint32_t hops;
89	};
90
91	/* a nexthop is a pointer to a gateway router plus an interface */
92	struct rt_nexthop {
93	  union olsr_ip_addr gateway;          /* gateway router */
94	  int iif_index;                       /* outgoing interface index */
95	};
```

____

rt_metric:在路径选择时使用符合矩阵，矩阵中包括两个节点间的路径花销和跳数。

rt_nexthop:该结构体表示吓一跳的网关和接口索引。

____

```c
78	/*
79	 * Every prefix in our RIB needs a route entry that contains
80	 * the nexthop of the best path as installed in the kernel FIB.
81	 * The route entry is the root of a rt_path tree of equal prefixes
82	 * originated by different routers. It also contains a shortcut
83	 * for accessing the best route among all contributing routes.
84	 */
85	struct rt_entry {
86	  struct olsr_ip_prefix rt_dst;
87	  struct avl_node rt_tree_node;
88	  struct rt_path *rt_best;             /* shortcut to the best path */
89	  struct rt_nexthop rt_nexthop;        /* nexthop of FIB route */
90	  struct rt_metric rt_metric;          /* metric of FIB route */
91	  struct avl_tree rt_path_tree;
92	  struct list_node rt_change_node;     /* queue for kernel FIB add/chg/del */
93	};
```

____

每一个RIB需要一个路由接口，这个接口中包含最佳路径的下一跳网关信息，同时该接口时rt_path_tree的根节点。同样也包含了所有路由信息中最佳的一个路径。rt_dst包含该信息的路由地址和前缀长度。rt_path_tree是一个avl树，rt_tree_node表示最短路径的引用。

____

```c
106	struct rt_path {
107	  struct rt_entry *rtp_rt;             /* backpointer to owning route head */
108	  struct tc_entry *rtp_tc;             /* backpointer to owning tc entry */
109	  struct rt_nexthop rtp_nexthop;
110	  struct rt_metric rtp_metric;
111	  struct avl_node rtp_tree_node;       /* global rtp node */
112	  union olsr_ip_addr rtp_originator;   /* originator of the route */
113	  struct avl_node rtp_prefix_tree_node; /* tc entry rtp node */
114	  struct olsr_ip_prefix rtp_dst;       /* the prefix */
115	  uint32_t rtp_version;                /* for detection of outdated rt_paths */
116	  uint8_t rtp_origin;                  /* internal, MID or
117	  HNA */
118	};
```

____

这个结构体主要描述了rt_path的成员，每接收到一个rt_path就将其加入RIB。根据Dijkstra算法计算出的结果可以得到最优路径，同时可以得到一个最小的矩阵。rt_path首先被加入到tc_entry树中，如果根据Dijkstra计算可得当前tc_entry是可达到，则在全局RIB树中加入下一跳地址。

____

```c
182	union olsr_kernel_route {
183	  struct {
184		struct sockaddr rt_dst;
185		struct sockaddr rt_gateway;
186		uint32_t metric;
187	  } v4;
188
189	  struct {
190		struct in6_addr rtmsg_dst;
191		struct in6_addr rtmsg_gateway;
192		uint32_t rtmsg_metric;
193	  } v6;
194	};
```

____

该联合体分别定义了IPV4和IPV6的olsr核心路由表结构。路由表项中主要包含目的地和网关。

____

```c
129	enum olsr_rt_origin {
130	  OLSR_RT_ORIGIN_MIN,
131	  OLSR_RT_ORIGIN_INT,
132	  OLSR_RT_ORIGIN_MID,
133	  OLSR_RT_ORIGIN_HNA,
134	  OLSR_RT_ORIGIN_MAX
135	};
```

____

OLSR中有三种不同的路由类型，INT（internal route）有简单的TC分组接收生成，MID由MID分组生成并且HNA路由由HNA公告生成。

#### 2.6.2 路由表计算

____

```c
167	void
168	olsr_init_routing_table(void)
169	{
170	  OLSR_PRINTF(5, "RIB: init routing tree\n");
171
172	  /* the routing tree */
173	  avl_init(&routingtree, avl_comp_prefix_default);
174	  routingtree_version = 0;
175
176	  /*
177	   * Get some cookies for memory stats and memory recycling.
178	   */
179	  rt_mem_cookie = olsr_alloc_cookie("rt_entry", OLSR_COOKIE_TYPE_MEMORY);
180	  olsr_cookie_set_memory_size(rt_mem_cookie, sizeof(struct rt_entry));
181
182	  rtp_mem_cookie = olsr_alloc_cookie("rt_path", OLSR_COOKIE_TYPE_MEMORY);
183	  olsr_cookie_set_memory_size(rtp_mem_cookie, sizeof(struct rt_path));
184	}
```

____

该函数的主要功能是初始化路由表。

首先调用avl_init()将路由表初始化为avl树，然后维护一个版本号routingtree_version用以检测每一个每一个rt_entry和rt_path，检查其是否过期，并将版本号初始化为0。

然后是为rt_entry和rt_path分配内存，并创建相应的cookie。

_____

```c
231	static struct rt_entry *
232	olsr_alloc_rt_entry(struct olsr_ip_prefix *prefix)
233	{
234	  struct rt_entry *rt = olsr_cookie_malloc(rt_mem_cookie);
235	  if (!rt) {
236		return NULL;
237	  }
238
239	  memset(rt, 0, sizeof(*rt));
240
241	  /* Mark this entry as fresh (see process_routes.c:512) */
242	  rt->rt_nexthop.iif_index = -1;
243
244	  /* set key and backpointer prior to tree insertion */
245	  rt->rt_dst = *prefix;
246
247	  rt->rt_tree_node.key = &rt->rt_dst;
248	  avl_insert(&routingtree, &rt->rt_tree_node, AVL_DUP_NO);
249
250	  /* init the originator subtree */
251	  avl_init(&rt->rt_path_tree, avl_comp_default);
252
253	  return rt;
254	}
```

____

该函数的功能是创建一个路由表项，并作一些初始化操作后将其插入avl树中。

首先是为新的路由表项申请内存空间，并且将空间清零。

标识该表项为新建表项，并将目的地址设置为参数提供的入口地址。

把该表项的树节点插入avl树，并初始化该树。

____

```c
292	void
293	olsr_insert_rt_path(struct rt_path *rtp, struct tc_entry *tc, struct link_entry *link)
294	{
295	  struct rt_entry *rt;
296	  struct avl_node *node;
297
298	  /*
299	   * no unreachable routes please.
300	   */
301	  if (tc->path_cost == ROUTE_COST_BROKEN) {
302		return;
303	  }
304
305	  /*
306	   * No bogus prefix lengths.
307	   */
308	  if (rtp->rtp_dst.prefix_len > olsr_cnf->maxplen) {
309		return;
310	  }
311
312	  /*
313	   * first check if there is a route_entry for the prefix.
314	   */
315	  node = avl_find(&routingtree, &rtp->rtp_dst);
316
317	  if (!node) {
318
319		/* no route entry yet */
320		rt = olsr_alloc_rt_entry(&rtp->rtp_dst);
321
322		if (!rt) {
323		  return;
324		}
325
326	  } else {
327		rt = rt_tree2rt(node);
328	  }
329
330	  /* Now insert the rt_path to the owning rt_entry tree */
331	  rtp->rtp_originator = tc->addr;
332
333	  /* set key and backpointer prior to tree insertion */
334	  rtp->rtp_tree_node.key = &rtp->rtp_originator;
335
336	  /* insert to the route entry originator tree */
337	  avl_insert(&rt->rt_path_tree, &rtp->rtp_tree_node, AVL_DUP_NO);
338
339	  /* backlink to the owning route entry */
340	  rtp->rtp_rt = rt;
341
342	  /* update the version field and relevant parameters */
343	  olsr_update_rt_path(rtp, tc, link);
```

____

该函数功能是对于每个rt_path创建一个路由表项并将其加入全局的RIB树中。

首先检查传入的参数是否合法，如果传入的tc_entry的path_cost为ROUTE_COST_BROKEN，或者传入的rtp的目的地址长度大于所设置的最大地址长度，直接返回NULL。

然后调用avl_find()函数检查传入的rtp节点是否在路由表中，如果节点不在路由表中，将其加入avl树中。如果在，则将其从avl_node类型转为rt_entry类型。

把新节点加入avl树，然后改变相应的参数，更新路由表。

____

```c
211	void
212	olsr_update_rt_path(struct rt_path *rtp, struct tc_entry *tc, struct link_entry *link)
213	{
214
215	  rtp->rtp_version = routingtree_version;
216
217	  /* gateway */
218	  rtp->rtp_nexthop.gateway = link->neighbor_iface_addr;
219
220	  /* interface */
221	  rtp->rtp_nexthop.iif_index = link->inter->if_index;
222
223	  /* metric/etx */
224	  rtp->rtp_metric.hops = tc->hops;
225	  rtp->rtp_metric.cost = tc->path_cost;
226	}
```

该函数的主要功能是更新路由路径的网关，接口和路由跳数和路径的版本信息。

____

```c
349	void
350	olsr_delete_rt_path(struct rt_path *rtp)
351	{
352
353	  /* remove from the originator tree */
354	  if (rtp->rtp_rt) {
355		avl_delete(&rtp->rtp_rt->rt_path_tree, &rtp->rtp_tree_node);
356		rtp->rtp_rt = NULL;
357	  }
358
359	  /* remove from the tc prefix tree */
360	  if (rtp->rtp_tc) {
361		avl_delete(&rtp->rtp_tc->prefix_tree, &rtp->rtp_prefix_tree_node);
362		olsr_unlock_tc_entry(rtp->rtp_tc);
363		rtp->rtp_tc = NULL;
364	  }
365
366	  /* no current inet gw if the rt_path is removed */
367	  if (current_inetgw == rtp) {
368		current_inetgw = NULL;
369	  }
370
371	  olsr_cookie_free(rtp_mem_cookie, rtp);
372	}
```

____

该函数功能是删除并释放一条路由路径。

首先将rtp所指的树节点从所在的树里面删除供将指向该树根节点的指针置空。

然后将其从前缀树中删除，并解锁相应的tc_entry。

最后把rtp所指向的树节点从rtp树里面删除，释放cookie所占用的内存。

____

```c
435	static bool
436	olsr_cmp_rtp(const struct rt_path *rtp1, const struct rt_path *rtp2, const struct rt_path *inetgw)
437	{
438	  olsr_linkcost etx1 = rtp1->rtp_metric.cost;
439	  olsr_linkcost etx2 = rtp2->rtp_metric.cost;
440	  if (inetgw == rtp1)
441		etx1 *= olsr_cnf->lq_nat_thresh;
442	  if (inetgw == rtp2)
443		etx2 *= olsr_cnf->lq_nat_thresh;
444
445	  /* etx comes first */
446	  if (etx1 < etx2) {
447		return true;
448	  }
449	  if (etx1 > etx2) {
450		return false;
451	  }
452
453	  /* hopcount is next tie breaker */
454	  if (rtp1->rtp_metric.hops < rtp2->rtp_metric.hops) {
455		return true;
456	  }
457	  if (rtp1->rtp_metric.hops > rtp2->rtp_metric.hops) {
458		return false;
459	  }
460
461	  /* originator (which is guaranteed to be unique) is final tie breaker */
462	  if (memcmp(&rtp1->rtp_originator, &rtp2->rtp_originator, olsr_cnf->ipsize) < 0) {
463		return true;
464	  }
465
466	  return false;
467	}
```

____

该函数的功能是比较两个路由路径，如过第一个路径更好，返回TRUE，否则，返回False。

首先比较路径花销，花销小的路径更优。

如果路径花销相同，则比较路径跳数，跳数更小的路径更优。

如果前两项比较结果相同，则比较源地址，源地址小的更优。

____

```c
485	void
486	olsr_rt_best(struct rt_entry *rt)
487	{
488	  /* grab the first entry */
489	  struct avl_node *node = avl_walk_first(&rt->rt_path_tree);
490
491	  assert(node != 0);            /* should not happen */
492
493	  rt->rt_best = rtp_tree2rtp(node);
494
495	  /* walk all remaining originator entries */
496	  while ((node = avl_walk_next(node))) {
497		struct rt_path *rtp = rtp_tree2rtp(node);
498
499		if (olsr_cmp_rtp(rtp, rt->rt_best, current_inetgw)) {
500		  rt->rt_best = rtp;
501		}
502	  }
503
504	  if (0 == rt->rt_dst.prefix_len) {
505		current_inetgw = rt->rt_best;
506	  }
507	}
```

____

运行最优路径，首先得到第一个条表项后遍历所有表项，找到一条最有路径并修改当前网关路径到最优路径。

首先调用avl_walk_first()函数从rt.rt_path_tree中的到第一个条目并坚持其是否为0，然后把节点转为rt_entry类型。

然后遍历整棵avl树，并比较当前路径和当前最优路径，获取最优路径。

____

```c
523	struct rt_path *
524	olsr_insert_routing_table(union olsr_ip_addr *dst, int plen, union olsr_ip_addr *originator, int origin)
525	{
526	#ifdef DEBUG
527	  struct ipaddr_str dstbuf, origbuf;
528	#endif
529	  struct tc_entry *tc;
530	  struct rt_path *rtp;
531	  struct avl_node *node;
532	  struct olsr_ip_prefix prefix;
533
534	  /*
535	   * No bogus prefix lengths.
536	   */
537	  if (plen > olsr_cnf->maxplen) {
538		return NULL;
539	  }
540
541	  /*
542	   * For all routes we use the tc_entry as an hookup point.
543	   * If the tc_entry is disconnected, i.e. has no edges it will not
544	   * be explored during SPF run.
545	   */
546	  tc = olsr_locate_tc_entry(originator);
547
548	  /*
549	   * first check if there is a rt_path for the prefix.
550	   */
551	  prefix.prefix = *dst;
552	  prefix.prefix_len = plen;
553
554	  node = avl_find(&tc->prefix_tree, &prefix);
555
556	  if (!node) {
557
558		/* no rt_path for this prefix yet */
559		rtp = olsr_alloc_rt_path(tc, &prefix, origin);
560
561		if (!rtp) {
562		  return NULL;
563		}
564	#ifdef DEBUG
565		OLSR_PRINTF(1, "RIB: add prefix %s/%u from %s\n", olsr_ip_to_string(&dstbuf, dst), plen,
566					olsr_ip_to_string(&origbuf, originator));
567	#endif
568
569		/* overload the hna change bit for flagging a prefix change */
570		changes_hna = true;
571
572	  } else {
573		rtp = rtp_prefix_tree2rtp(node);
574	  }
575
576	  return rtp;
```

____

数功能是将一个前缀节点插入一颗前缀树。首先检查是否该rt_path是已知的，如果不是，则创建，如果根据Dijkstra算法得到节点不可达，最后计算最短路径是不考虑。