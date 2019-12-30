<center> <font size=10>OLSR协议分析</font>  </center>


# 1. 概述

## 1.1 协议概述

最佳链路状态路由协议（Optimized Link State Routing Protocol）是一个针对Ad-hoc网络需求的先应式的链路状态算法的优化协议。

Ad-hoc网络是一种多跳，无中心的，自组织的无线网络，其不依赖于固定的网络核心，每个节点都是移动的。该网络的主要特点是自组织性，节点对等，分布式控制，临时性，网络拓扑结构变化快，带宽链路有限等。

针对Ad-hoc网络的特点，该协议的核心主要是是MPR(Multipoint Relay)多点中继，每个节点选择选择一组其邻居节点作为MPR节点，然后只由这些MPR节点负责洪泛网络控制信息。MPR通过减少流量的传输次数，减少了网络中的信息量开销。

该协议中，MPR节点负责周期性的广播路由更新，每个节点根据其他节点的控制信息，计算自己的网络拓扑，得到网络节点间的最短路径。该协议使用Dijkstra最短路径算法选择路径


## 1.2 主要算法描述

### 1.2.1 部分元组结构

**链接元组 ___(L\_local\_iface\_addr, L\_neighbor\_iface\_addr, L\_SYM\_time, L_ASYM\_time, Ltime)___** 

**邻居元组 ___(N\_neighbor\_main\_addr, N\_2hop\_addr,N\_time)___**

**MPR选择元组 ___(MS\_main\_addr, MS\_time)___** 

**拓扑元组 ___{T\_test\_addr,T\_lase\_addr, T\_seq, T\_time}___**

**路由表项元组__{R\_dest\_addr, R\_next\_addr, R\_dist, R\_iface\_addr}__**

### 1.2.2链路感知

在Ad-hoc这种无线自组织网络，由于网络的动态性，有些连接可能是单向连接。OLSR规定只有双向对称的链路才能传播信息，每个节点要建立与其他节点的对称链路，必须进行链路检测。

节点链路感知是要通过互相发送HELLO分组来实现的，本地链路信息库存储本节点到邻居节点的链接信息。当收到一个HELLO分组时，一个节点应该更新其链路集，收到一个HELLO分组后，首先计算该分组的有效事件，来确认该分组是否有效，然后更新链路信息，更新链路信息的主要流程如下：

* 收到一个HELLO分组，如果不存在如下一条链路记录元组：


​		<code>L\_neighbor\_iface\_addr == 该分组的Originator Address</code>

必须加入在链接信息库中新加入：

​		<code>  L\_beighbor\_iface\_addr  = 该分组的Originator Address</code>>

​        <code>L\_local\_iface\_addr          = 该分组的接口地址</code>

​		<code>L\_SYM\_time                     = Current time - 1</code>

​		<code>L\_time                               =current time + validity time</code>  

* 如果该元组已经存在,则按照如下规则修改链路信息集：

<code>L\_ASYM\_time = current time + validity time</code>

如果接口地址在HELLO分组链路信息列表中，则如下更新：

		* 如果链接状态是LOST_LINK,则：

​		<code>L\_SYM_time = current time + validity time</code>

 * 如果链接状态是SYM_LINK或者ASYM_LINK,则：

​		<code>L\_SYM\_time = current time + validity time</code>

​		<code>L\_time = L\_SYM_time+NEIGHB\_HELLO\_TIME</code>

<code> L\_time = max(L\_time, L\_ASYM\_time)</code>

### 1.2.3 邻居检测

邻居检测会通过改变邻居信息库来记录邻居信息，而且这种检测主要和节点与节点间的主地址相关。OLSR中的邻居检测机制是通过定期交换HELLO分组实现的。

#### 1. 扩充邻居集

节点基于链接元组集来维护邻居信息元组集，随着链接信息集的更新来更新这些信息。

链接集保存链路的链接信息，邻居集保存邻居信息，这两个集合之间有明确的关系，当一个节点是另一个节点的邻居节点时，这两个节点间至少应该有一条双向链接链路。

邻居和链接只间的正是对应关系如下:

* 如果一个链接元组中的关联邻居元组存在，邻居元组为：

  ​	<code>N\_neighbor\_main\_addr ==L\_neightbor\_iface \_addr的主地址</code> 

* 如果邻居元组的关联链接元组都是链接元组，并且：

  ​	<code>N\_neighbor\_main\_addr == L\_neighbot\_iface\_addr的主地址</code>	

  则邻居集必须通过维护链接元组和关联邻居元组之间对应关系来填充，填充规则如下：

  * 创建

    每次出现一个链接时，必须创建关联的邻居元组，如果邻居元组不存在，则：

    <code>N\_neighbor\_main\_addr = Lneighbor\_iface\_addr 的主地址</code>

  * 更新

    每次链接信息更改时，节点必须确保相关邻居元组的N_status遵循：

    ​		如果邻居有任何关联元组代表一个对称链接，则：

    ​		<code>N\_status = SYM</code>

    ​		否则：

    ​		<code>N\_status = NOT\_SYM</code>

  * 删除

    每次删除一个链接时，相关的链接元组必须被删除，相关的邻居元组也必须被删除。

  这些规则都是为了确保存在一个唯一的相关邻居元组对应一个链接元组，并且每个邻居元组至少对应一个关联元组。

* HELLO分组处理

  一个HELLO分组的Original Address是该分组的发出者地址。同时，根据HELLO的willingness字段重新计算willingness。收到一个HELLO分组后，节点需要更新自己的链接集和邻居集。
* 如果发出者地址是邻居集中邻居元组中的N\_neighbor\_main\_addr，则邻居元组应该更新：
  
  ​	<code>N\_willingness = HELLO分组中的willingness</code>

#### 2. 扩充二跳邻居集

二跳邻居集描述的是对称邻居的对称链接信息，其依然通过互相交互HELLO分组来维护。

从对称邻居接受到一个全新的HELLO分组后，一个节点应该更新其二跳邻居集。接受到一个HELLO分组，首先通过Vtime计算有效时，检测其是否有效。如果该分组的发出者地址是包含着链接集中链接元组中的L\_neighbor\_iface\_addr的一个主地址，如果 <code>L\_SYM\_time >= current time</code> 则该分组未过期。

* 对于每一个在HELLO中分组中且邻居类型为SYM\_NEIGH或MPR\_NEIGH的地址：

   * 如果一个二跳邻居节点的主地址不等于接受节点的主地址，则丢弃该二跳邻居。

   * 否则，新创建一个二跳邻居元组：

     <code>N\_neighbor\_main\_addr = 发出者地址</code>

     <code>N\_2hop\_addr = 二跳邻居的主地址</code>

     <code>N\_time = current time+validity time</code>

     这个新分组用于替换和其具有相同N\_neighbor\_main\_addr 和N\_2hop\_addr值的元组。

* 对于每一个在HELLO中分组中且邻居类型为NOT_NEIGH的地址，所有的二跳邻居应该是：

  ​	<code>N\_neighbor\_main\_addr == 发出者地址 and</code> 

  ​	<code>N\_2hop\_addr == 二跳邻居的主地址</code>

###  1.2.4 MPR

MPR用于洪泛网络控制分组，主要是减少分组传输过程中的重传次数，因此，该机制是对传统洪泛机制的优化。

网络中的每个节点独立地从一跳对称邻居中选择自己的MPR集，和邻居节点中MPR的对称链路在HELLO分组中的链路类型为MPR_NEIGH。

 每个节点的每个接口都计算自己的MPR集，所有接口的MPR集的并集为该节点的MPR集。

MPR集应满足以下两点要求：节点与MPR之间必须是一跳双向链路；节点能通过MPR集到达所有的严格两跳邻居节点。



#### 1. MPR计算

* 从一个节点的子集开始，当N\_willness等于WILL_ALWAYS该子集中成员属于一个MPR集。

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

     <code>MS\_main\_addr == 发出者地址</code>

     则创建一个新元组：

     <code>MS\_main\_addr = 发出者地址</code>

  2. 如果元组：

     <code>MS\_main\_addr == 发出者地址</code>

     则修改MS_time字段:

     <code>MS\_time = current time + validity time</code>

#### 3. 邻居和二跳邻居变化

​	邻居节点的变化情况将在以下情况下被检测到:

* 如果链接元组的L\_SYM\_Time字段到期。

* 在链接集中插入一个新的链接元组，其中L\_SYM\_Time字段未过期，或者L\_SYM\_Time被修改避免过期。

当邻居集或者二跳邻居集被检测到发生变化，则执行以下流程：

* 如果有邻居丢失，则所有<code>N\_neighbor\_main\_addr==Main Address</code>的二跳元组必须被删除。
* 如果有邻居丢失，所有<code>MS\_main\_addr == Main Address</code>的MPR selector元组必须被删除。
* 当有邻居或者二跳邻居改变，丢失，或者删除时，MPR集必须重新计算。
* 当MPR集发生改变时，可能发出一个额外的HELLO分组。

### 1.2.5 拓扑发现

前面的链路检测和邻居检测部分为每个节点提供了能够直接通信的邻居列表，并且合并了数据包的格式和通过MPR优化的前向路由洪泛算法。拓扑信息基于此在网络中得以传播。

网络的路由结构是通过广播链路来实现的。一个节点必须至少传播其本身和MPR selector集中的信息，让网络中有充分的信息构建路由表。

#### 1. 广播邻居集

一个节点发出TC分组声明链接集，称为广播链接集。该行为必须至少包含到其MPR selector集的所有链接信息。

与广播邻居集相关联的序列号(ANSN)也应该伴随发出。当链接从广播的邻居集中移除时，ANSN必须增加。当有链接加入邻居集时，ANSN也应该增加。

#### 2. TC分组生成

为了构建拓扑信息集，每一个被选为MPR的节点，必须广播TC分组。TC分组通过广播方式传播到网络中的每一个节点。MPR在拓扑信息的分散上有良好的扩展性。

地址是TC分组的一部分。所有TC分组的解析必须在一个确定的更新周期内完成。这些TC分组携带的信息将协助网络中所有节点完成路由表的计算。

当一个节点的可广播链接集为空时，其依然应该在之前发出TC分组的validity time时间内持续发送空的TC分组消息。以使之前发送的TC分组失效。然后其应该停止发送TC分组，直到有某个节点加入该节点的广播链接集。

一个节点能够传递附加的TC分组以增强对链路故障的反应性。当MPR selector集的变化被检测到，且该变化可能导致链路故障时，节点应该在短于TC_INTERVAL的时间内传递一个TC分组。

#### 3. TC分组转发

TC分组必须由MPR节点广播转发到整个网络。

#### 4. TC分组处理

* 当接受到一个TC分组后，必须先通过其头部的Vtime字段计算其有效时间然后，拓扑集根据以下规则更新：
  
* 如果此分组的发送者接口不在本节点的一跳对称邻居中，则丢弃该分组。
  
* 如果拓扑信息集中存在一些元组：

  <code>T\_last\_addr == 发出者地址 AND</code>

  <code>T\_seq > ANSN</code>

  则不对该分组做任何进一步处理，并且丢弃该分组。

* 如果拓扑集中所有的元组都是:

  <code>T\_last\_addr == 发出者地址 AND</code>

  <code>T\_seq < ANSN</code>

  则必须从拓扑集中删除。

* 对于每一个在TC分组中受到的被广播的邻居的主地址：

  * 如果在拓扑集中存在一些元组：

    <code> T\_test\_addr == 广播邻居的主地址 AND</code>

    <code>T\_last\_addr == 发出者地址</code>

    则该元组的有效时间必须被设为：

    <code>T\_time = current time + validity time</code>

  * 否则，必须在拓扑集中记录一条新的原则记录:

    <code>T\_dest\_addr = 被广播邻居主地址</code>

    <code>T\_last\_addr = 发出者地址</code>

    <code>T\_seq = ANSN</code>

    <code>T\_time = current time + validity time</code>

### 1.2.6 路由表计算

每个节点都拥有一个路由表，通过路由表节点能够让网络中其他节点发送信息.路由表是基于本地的链接信息集和拓扑集构建的，所以这两个集合一旦有任何一个发生变化，路由表都需要重新计算。

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

  <code>N\_status = SYM</code>

  并且，对于邻居节点的每个关联链接元组，都有<code>L\_time >= current time</code>,路由表中新加的表项为:

  <code>R\_dest_addr = 关联链接元组的L_neighbor_iface_addr</code>

  <code>R\_next\_addr = 关联链接元组的L\_neighbor\_iface\_addr</code>

  <code> R\_dist = 1</code>

  <code>R\_iface_addr = 关联链接元组的L\_local\_iface\_addr</code>

  如果按照上述情况，没有R\_dest\_addr等于邻居节点的主地址，则必须添加新的路由表项:

  <code> R\_dest\_addr = 邻居的Main address</code>

  <code>R\_next\_addr = L_time>0的关联链接元组的L\_neighbor\_iface\_addr</code>

  <code>R\_dist = 1</code>

  <code>R\_iface\_addr = 关联链接元组的L\_local\_iface</code>

* 对于严格二条邻居节点，如果二跳邻居集中至少存在一项记录中N\_neighbor\_main\_addr对于一个willingness不为WILL_NEVER的节点，选择一个二跳邻居节点，并在路由表中添加新表项：

  <code>R\_dest\_addr = 二跳邻居的Main Address</code>

  <code>R\_next\_addr = 路由表项中R\_dest\_addr等于二跳邻居N\_neighbor\_main\_addr的记录的R\_next\_addr</code>

  <code>R\_dist = 2 </code>

  <code>R\_ifacce\_addr = 路由表中R\_dest\_addr等于二跳元组中N\_neighbor\_main-addr的记录的R\_iface\_addr</code>

* 对于h+1跳的节点，按照Dijkstra算法加入路由表中

* 对于多接口关联信息集中的每个实体，如果不存在一个路由表项：

  <code>R\_dest\_addr == I\_main\_addr</code>

  并且，也没有表项：

  <code>R\_dest\_addr == I\_iface\_addr</code>

  则添加一个新表项：

  <code>R\_dext\_addr = I\_iface\_addr</code>

  <code>R\_dext\_addr = R\_next\_addr</code>

  <code>R\_dist = R\_dist</code>

  <code> R\_iface\_addr = R\_iface\_addr</code>

## 1.3 OLSR的优点和局限性
* OLSR协议是一种先应式路由协议，具有查找路由延时小的优点。

* OLSR用于移动自组织网络，MPR的实现极大的减小了网络中的控制流量，特别适用大型密集型移动网络，而且网络越大，效果越好。

* OLSR最初就是为以完全分布式方式工作，不依赖任何中心实体。
* OLSR不需要对IP数据包做任何修改。

# 2.代码分析

## 2.1 文件介绍

OLSR协议中共有123个源文件，以下对部分重要文件进行罗列。

![image-20191229224612199](C:\Users\yuxi\AppData\Roaming\Typora\typora-user-images\image-20191229224612199.png)

## 2.2 数据结构

### 2.2.1 OLSR头部(省略IP和UDP报头)

____

```c
55	struct olsr_common {
56	  uint8_t type;
57	  olsr_reltime vtime;
58	  uint16_t size;
59	  union olsr_ip_addr orig;
60	  uint8_t ttl;
61	  uint8_t hops;
62	  uint16_t seqno;
63	};
```

____

53-63：olsr\_common是OLSR协议的基本数据包首部。

对于与协议相关的所有数据，OLSR使用统一的数据包格式进行通信，这样做的目的是在不破坏向后兼容性的情况下促进协议的可扩展性。这也提供了一种简单的方法，将不同“类型”的信息汇集到一个单一的传输中。这些数据包嵌入在UDP数据报中，使用UDP通信，IANA将端口698分配给OLSR协议专用。每个分组封装一个或多个消息，这些消息共享一个通用的报头格式，使节点能够接受和重传未知类型的消息。OLSR协议分组的基本格式如图所示：

![image-20191229225147064](C:\Users\yuxi\AppData\Roaming\Typora\typora-user-images\image-20191229225147064.png)

**a.**数据包首部

**Packet** **Length：**数据包的长度（单位：字节）

**Packet** **Sequence** **Number：**数据包序列号，每当传送一个新的OLSR分组时，序列号加一。为每个接口维护一个单独的分组序列号，以便对通过接口发送的分组进行顺序枚举。

**b.**消息首部

**Message** **Type** ：消息类型，在0-127范围内的消息类型保留给本协议中的所有消息和以后可能的扩展。

**Vtime** ：该字段表明接收节点后多长时间必须将消息中包含的信息视为有效，除非接收到对信息的最新更新。有效时间由它的尾数a（Vtime字段的四个最高位）和它的指数b（Vtime字段的四个最低位）表示。有效期=C*（1a/16）*2^b[秒]，C为一系数。

**Message Size**：消息的大小，以字节为单位计算。并从“消息类型”字段的开头到下一个“消息类型”字段的开头（如果没有下一个“消息类型”，则到该消息的结尾为止)。

**Originator Address**：此字段包含最初生成此消息的节点的主地址，重传过程中地址不发生改变。此字段不应与IP标头中的源地址混淆，后者每次都更改为重传该消息的中间接口的地址。

**Time To Live**：此字段包含将发送消息的最大跳数。消息每次重传前TTL减1，当TTL为0或1时，则不再重传。通过此方式可以限制一个消息的洪泛范围。

**Hop Count**：一个消息获得的跳数的数量。每次重传消息前，跳数加一。初始值为0。

**Message Sequence Number**：由源节点产生的一个消息的唯一标识，用来保证一个消息不会被任何节点重传一次以上。

### 2.2.2 HELLO分组

OLSR协议采用一种通用的机制来填充本地链路信息库和邻居信息库，即周期性地交换HELLO消息。

____

```c
106	/* serialized LQ_HELLO */
107	struct lq_hello_info_header {
108	  uint8_t link_code;
109	  uint8_t reserved;
110	  uint16_t size;
111	};
112
113	struct lq_hello_header {
114	  uint16_t reserved;
115	  uint8_t htime;
116	  uint8_t will;
117	};
```

____

106-117：结构体lq\_hello\_info\_header和lq\_hello\_header共同组成HELLO消息数据包的首部。格式如下图：

![image-20191229225535823](C:\Users\yuxi\AppData\Roaming\Typora\typora-user-images\image-20191229225535823.png) 这一部分是将OLSR基本数据包首部的"Message Type"设为HELLO_MESSAGE，TTL设为1，Vtime设为NEIGHB_HOLD_TIME。 

**Reserved**:保留字段,必须设置设为“0000000000000

**Htime**:指定节点在特定接口上的HELLO发射间隔，即下一个HELLO传输前的时间。Hello发射间隔用尾数(Htime的四个最高位)和指数(Htime的四个最低位)来表示。即HELLO发射间隔=C*（1+a/16）*2^b[秒]。其中a是Htime字段的四个最高位表示的整数，b是Htime字段的四个最低位表示的整数。 

**willingness**:此字段指定节点为其他节点承载和转发流量的意愿。

Willingness有三种取值：

1. **WILL\_NEVER**：永远不会选择意愿为WILL_NEVER的节点作为MPR。

2. **WILL\_ALWAYS**：具有WILL_ALWAYS意愿的节点将始终被选择为MPR。

3. **WILL\_DEFAULT**：默认情况下，节点意愿为WILL_DEFAULT。

**Link Code**:此字段指定发送方的接口和邻居列表中邻居接口之间的链路类型。它还指定有关邻居状态的信息。链路类型不为节点所知的邻居信息被静默丢弃:

链路类型有以下三种： 

1. **ASYM\_LINK**:发送HELLO分组的节点与邻居列表中的节点间的链路是非对称的。表示可以收到邻居节点的消息,但不确定邻居节点能否收到本节点的消息。

2. **SYM\_LINK**:发送HELLO分组的节点与列表中的邻节点间的链路是对称的。表示链路已经被验证为双向的。

3. **MPR\_LINK**:表示列表中的节点已被发送该HELLO分组的节点选择为MPR。

**Link Message Size**：本链路消息的大小，以字节为单位，从Link Code字段开始到下一个Link Code字段之前（如果没有下一个Link Code，则到该消息尾部）。

**Neighbor Interface Address**：邻居节点的接口地址。每一种链路类型之后都紧跟一组邻居节点接口地址，表明该节点与这组邻居节点中的每一个节点链路类型都相同，都为前面给出的链路类型。

____

```c
49	struct hello_neighbor {
50	  uint8_t status;
51	  uint8_t link;
52	  union olsr_ip_addr main_address;
53	  union olsr_ip_addr address;
54	  struct hello_neighbor *next;
55	  olsr_linkcost cost;
56	  uint32_t linkquality[0];
57	};
58
59	struct hello_message {
60	  olsr_reltime vtime;
61	  olsr_reltime htime;
62	  union olsr_ip_addr source_addr;
63	  uint16_t packet_seq_number;
64	  uint8_t hop_count;
65	  uint8_t ttl;
66	  uint8_t willingness;
67	  struct hello_neighbor *neighbors;
68
69	};
```

____

49-57：结构体hello\_neighbor是HELLO消息邻居节点集。Status，记录邻居的状态；link指明链路类型；main_address是邻居的主地址；address是邻居的其他地址；cost，链路代价；linkquality，链路质量。

59-69：hello\_message是消息数据包。vtime，消息有效时间；htime，HELLO消息的发射间隔；source\_addr，发送消息的原地址；packet_seq_number，数据包的序列号；hop\_count，消息已经经历的跳数；ttl，数据包生命周期；willingness，节点进行转发的意愿；neighbors，下一个传递的邻居节点。

### 2.2.3 TC分组

该部分是将OLSR首部中“MessageType”设置为TC_Message。TTL设置为255(最大值)，以便将消息传播到网络中Vtime相应地设置为Top_hold_time的值。

![image-20191229230132208](C:\Users\yuxi\AppData\Roaming\Typora\typora-user-images\image-20191229230132208.png)

**ANSN**：序列号与公布的邻居集相关。每当节点检测到其邻居集中的更改时，它都会递增这个序列号。当节点接收到TC消息时，它可以根据该消息来判定所接收到的信息是否比现有消息更新。

**Advertised Neighbor Main Address**：此字段包含邻居节点（此处的邻居节点仅为该节点的MPR Selector集中的节点）的主地址。

**Reserved**：此字段是保留的，必须设置为“0000000000000000”。

____

```c
77	struct tc_message {
78	  olsr_reltime vtime;
79	  union olsr_ip_addr source_addr;
80	  union olsr_ip_addr originator;
81	  uint16_t packet_seq_number;
82	  uint8_t hop_count;
83	  uint8_t ttl;
84	  uint16_t ansn;
85	  struct tc_mpr_addr *multipoint_relay_selector_address;
86	};
```

____

77-86：tc\_message是TC消息数据包格式。OLSR协议利用TC拓扑表记录接收到的TC消息内容。TC拓扑表的介绍见“节点存储的表项”的“TC拓扑表”。

### 2.2.4 节点存储表项

1. 接口关联集

   网络中的每一个目的地节点存储了接口关联多元组。

   | I\_iface\_addr | I\_main\_addr | I\_time |
   

**i\_iface\_addr**是节点的接口地址；

**i\_main\_addr**是该节点的主地址；

**i\_time**指定此元组过期的时间，以及必须删除的时间。

2. 本地链路信息表

   |L\_local\_iface_addr | L_neighbor_iface_addr | L_SYM_time | L_ASYM_time | L_time 

​	  **L\_local\_iface\_addr**是本地节点(即链路的一个端点)的接口地址；

​	  **L\_neighbor\_iface\_addr**是相邻节点的接口地址；

​	  **L\_SYM\_time**是链路被视为对称的时间；

​	  **L\_ASYM\_time**是被认为听到邻居接口的时间；

 	 **L\_Time**指定了此记录过期的时间，并且必须被删除。当L\_SYM\_TIME和L\_ASYM\_TIME过期时，该链接被视为丢失。

3.  一跳邻居表

   | N_neighbot_main | N_status | N_willingness |

 	 **N\_neighbor\_main\_addr**：邻居的主地址；

  	**N\_status**：指定节点是NOT\_SYM还是SYM；

  	**N\_willingness**：指定节点的携带意愿，取值为0到7之间的整数。

4. 二跳邻居表

   | N_neighbor_main_addr | N_2hop_addr | N_time |

  	**N\_neighbor\_main\_addr**是邻居的主地址；

  	**N\_2hop\_addr**是具有与N\_neighbor\_main\_addr对称链接的2跳邻居的主地址，也就是说节点通过邻居节点N\_neighbor\_main\_addr到达其二跳邻居节点N\_2hop\_addr；

  	**N\_time**指定元组过期和必须删除的时间。

5. MPR表

      一个节点维护着其被选为MPR的邻居节点的集合，这些被选为MPR的邻居节点的主地址存放在MPR集中。

6. MPR selector表

   | MS_main_addr | MS_time |
   

**MS\_main\_addr**是节点的主地址，该节点选择本节点为MPR；

**MS\_time**指定元组过期的时间和必须删除的时间。

7. TC拓扑表

   网络中的每个节点维护有关网络的拓扑信息。此信息从TC消息中获取，并用于路由表计算。

   | T_dest_addr | T_last_addr | T_seq | T_time |
   

**T\_dest\_addr**目的地节点的主地址，它可以从地址为T\_last\_addr的节点经过一跳到达；

**T\_last\_addr**是T\_dest\_addr的MPR；

**T\_seq**是一个序列号；

**T\_time**指定此元组过期的时间，以及必须删除的时间。

8. 路由表

   | R_dest_addr | R_next_addr | R_dist | R_iface_addr |
   

**R\_dest\_addr**目的地节点地址

**R\_next\_addr**下一跳节点地址

**R\_dist**本节点到目的节点的距离

**R\_iface\_addr**该节点转发路由信息的接口

## 2.3 链路感知和邻居检测

链路感知的机制是HELLO消息的周期性交换。节点必须在每个接口上执行链路感知，以便检测接口和邻居接口之间的链路。因此，对于给定的接口，HELLO消息将包含该接口上的链路列表，以及整个邻居的列表。

### 2.3.1 HELLO分组生成

原则上，HELLO消息服务于三个独立的任务：链路感知，邻居检测和MPR选择。

------

```c
66	void
67	generate_hello(void *p)
68	{
69	  struct hello_message hellopacket;
70	  struct interface *ifn = (struct interface *)p;
71
72	  olsr_build_hello_packet(&hellopacket, ifn);
73
74	  if (queue_hello(&hellopacket, ifn))
75		net_output(ifn);
76
77	  olsr_free_hello_packet(&hellopacket);
78
79	}
```

------

66-79：generate\_hello()函数用来产生hello消息包。调用olsr\_build\_hello\_packet()函数为指定的接口生成要发送的HELLO数据包。如果创建成功，则通过net\_output()将该HELLO数据包通过指定的接口发送出去。最后释放该HELLO消息。

____

```c
88 	int
89 	olsr_build_hello_packet(struct hello_message *message, struct interface *outif)
90 	{
91 	  struct hello_neighbor *message_neighbor, *tmp_neigh;
92 	  struct link_entry *links;
93 	  struct neighbor_entry *neighbor;
94 
95 	#ifdef DEBUG
96 	  OLSR_PRINTF(3, "\tBuilding HELLO on interface \"%s\"\n", outif->int_name ? outif->int_name : "<null>");
97 	#endif
98 
99 	  message->neighbors = NULL;
100	  message->packet_seq_number = 0;
101
102	  //message->mpr_seq_number=neighbortable.neighbor_mpr_seq;
103
104	  /* Set willingness */
105
106	  message->willingness = olsr_cnf->willingness;
107	#ifdef DEBUG
108	  OLSR_PRINTF(3, "Willingness: %d\n", olsr_cnf->willingness);
109	#endif
110
111	  /* Set TTL */
112
113	  message->ttl = 1;
114	  message->source_addr = olsr_cnf->main_addr;
115
116	#ifdef DEBUG
117	  OLSR_PRINTF(5, "On link:\n");
118	#endif
119
120	  /* Walk all links of this interface */
121	  OLSR_FOR_ALL_LINK_ENTRIES(links) {
122	#ifdef DEBUG
123		struct ipaddr_str buf;
124	#endif
125		int lnk = lookup_link_status(links);
126		/* Update the status */
127
128		/* Check if this link tuple is registered on the outgoing interface */
129		if (!ipequal(&links->local_iface_addr, &outif->ip_addr)) {
130		  continue;
131		}
132
133		message_neighbor = olsr_malloc_hello_neighbor("Build HELLO");
134
135		/* Find the link status */
136		message_neighbor->link = lnk;
137
138		/*
139		 * Calculate neighbor status
140		 */
141		/*
142		 * 2.1  If the main address, corresponding to
143		 *      L_neighbor_iface_addr, is included in the MPR set:
144		 *
145		 *            Neighbor Type = MPR_NEIGH
146		 */
147		if (links->neighbor->is_mpr) {
148		  message_neighbor->status = MPR_NEIGH;
149		}
150		/*
151		 *  2.2  Otherwise, if the main address, corresponding to
152		 *       L_neighbor_iface_addr, is included in the neighbor set:
153		 */
154
155		/* NOTE:
156		 * It is garanteed to be included when come this far
157		 * due to the extentions made in the link sensing
158		 * regarding main addresses.
159		 */
160		else {
161
162		  /*
163		   *   2.2.1
164		   *        if N_status == SYM
165		   *
166		   *             Neighbor Type = SYM_NEIGH
167		   */
168		  if (links->neighbor->status == SYM) {
169			message_neighbor->status = SYM_NEIGH;
170		  }
171
172		  /*
173		   *   2.2.2
174		   *        Otherwise, if N_status == NOT_SYM
175		   *             Neighbor Type = NOT_NEIGH
176		   */
177		  else if (links->neighbor->status == NOT_SYM) {
178			message_neighbor->status = NOT_NEIGH;
179		  }
180		}
181
182		/* Set the remote interface address */
183		message_neighbor->address = links->neighbor_iface_addr;
184
185		/* Set the main address */
186		message_neighbor->main_address = links->neighbor->neighbor_main_addr;
187	#ifdef DEBUG
188		OLSR_PRINTF(5, "Added: %s -  status %d\n", olsr_ip_to_string(&buf, &message_neighbor->address), message_neighbor->status);
189	#endif
190		message_neighbor->next = message->neighbors;
191		message->neighbors = message_neighbor;
192
193	  }
194	  OLSR_FOR_ALL_LINK_ENTRIES_END(links);
195
196	  /* Add the links */
197
198	#ifdef DEBUG
199	  OLSR_PRINTF(5, "Not on link:\n");
200	#endif
201
202	  /* Add the rest of the neighbors if running on multiple interfaces */
203
204	  if (ifnet != NULL && ifnet->int_next != NULL)
205		OLSR_FOR_ALL_NBR_ENTRIES(neighbor) {
206
207	#ifdef DEBUG
208		struct ipaddr_str buf;
209	#endif
210		/* Check that the neighbor is not added yet */
211		tmp_neigh = message->neighbors;
212		//printf("Checking that the neighbor is not yet added\n");
213		while (tmp_neigh) {
214		  if (ipequal(&tmp_neigh->main_address, &neighbor->neighbor_main_addr)) {
215			//printf("Not adding duplicate neighbor %s\n", olsr_ip_to_string(&neighbor->neighbor_main_addr));
216			break;
217		  }
218		  tmp_neigh = tmp_neigh->next;
219		}
220
221		if (tmp_neigh) {
222		  continue;
223		}
224
225		message_neighbor = olsr_malloc_hello_neighbor("Build HELLO 2");
226
227		message_neighbor->link = UNSPEC_LINK;
228
229		/*
230		 * Calculate neighbor status
231		 */
232		/*
233		 * 2.1  If the main address, corresponding to
234		 *      L_neighbor_iface_addr, is included in the MPR set:
235		 *
236		 *            Neighbor Type = MPR_NEIGH
237		 */
238		if (neighbor->is_mpr) {
239		  message_neighbor->status = MPR_NEIGH;
240		}
241		/*
242		 *  2.2  Otherwise, if the main address, corresponding to
243		 *       L_neighbor_iface_addr, is included in the neighbor set:
244		 */
245
246		/* NOTE:
247		 * It is garanteed to be included when come this far
248		 * due to the extentions made in the link sensing
249		 * regarding main addresses.
250		 */
251		else {
252
253		  /*
254		   *   2.2.1
255		   *        if N_status == SYM
256		   *
257		   *             Neighbor Type = SYM_NEIGH
258		   */
259		  if (neighbor->status == SYM) {
260			message_neighbor->status = SYM_NEIGH;
261		  }
262
263		  /*
264		   *   2.2.2
265		   *        Otherwise, if N_status == NOT_SYM
266		   *             Neighbor Type = NOT_NEIGH
267		   */
268		  else if (neighbor->status == NOT_SYM) {
269			message_neighbor->status = NOT_NEIGH;
270		  }
271		}
272
273		message_neighbor->address = neighbor->neighbor_main_addr;
274		message_neighbor->main_address = neighbor->neighbor_main_addr;
275	#ifdef DEBUG
276		OLSR_PRINTF(5, "Added: %s -  status  %d\n", olsr_ip_to_string(&buf, &message_neighbor->address), message_neighbor->status);
277	#endif
278		message_neighbor->next = message->neighbors;
279		message->neighbors = message_neighbor;
280
281		}
282	  OLSR_FOR_ALL_NBR_ENTRIES_END(neighbor);
283
284	  return 0;
285	}
```

106和113：设置willingness和ttl。HELLO消息只能在一跳范围内传播，所以ttl值设为1.

120-180：遍历该接口的所有链路得到链路状态，用以计算邻居的状态。如果邻居的主地址位于该节点的MPR集合，则将邻居的状态更新为MPR\_NEIGH；如果邻居的主地址位于该节点的邻居列表（即不为MPR），则判断之间的链路状态是否对称并更新邻居的状态域。

182-186：设置邻居节点的接口地址以及主地址。

____

```c
58	struct link_entry {
59	  union olsr_ip_addr local_iface_addr;
60	  union olsr_ip_addr neighbor_iface_addr;
61	  const struct interface *inter;
62	  char *if_name;
63	  struct timer_entry *link_timer;
64	  struct timer_entry *link_sym_timer;
65	  uint32_t ASYM_time;
66	  olsr_reltime vtime;
67	  struct neighbor_entry *neighbor;
68	  uint8_t prev_status;
69
70	  /*
71	   * Hysteresis
72	   */
73	  float L_link_quality;
74	  int L_link_pending;
75	  uint32_t L_LOST_LINK_time;
76	  struct timer_entry *link_hello_timer; /* When we should receive a new HELLO */
77	  olsr_reltime last_htime;
78	  bool olsr_seqno_valid;
79	  uint16_t olsr_seqno;
80
81	  /*
82	   * packet loss
83	   */
84	  olsr_reltime loss_helloint;
85	  struct timer_entry *link_loss_timer;
86
87	  /* user defined multiplies for link quality, multiplied with 65536 */
88	  uint32_t loss_link_multiplier;
89
90	  /* cost of this link */
91	  olsr_linkcost linkcost;
92
93	  struct list_node link_list;          /* double linked list of all link entries */
94	  uint32_t linkquality[0];
```

____

58-62：local\_iface\_addr存储了节点接口的IP地址，neighbor\_iface\_addr存储邻居节点的接口IP地址。

63-68：各种时间的存储结构。link\_timer定时器，link\_sys\_timer定时器。Vtime表示接受该消息之后多少时间之内视其为有效。Neighbor以链表的形式存储邻居节点信息，pre\_status记录上一个节点的状态。

____

```c
58	struct neighbor_entry {
59	  union olsr_ip_addr neighbor_main_addr;
60	  uint8_t status;
61	  uint8_t willingness;
62	  bool is_mpr;
63	  bool was_mpr;                        /* Used to detect changes in MPR */
64	  bool skip;
65	  int neighbor_2_nocov;
66	  int linkcount;
67	  struct neighbor_2_list_entry neighbor_2_list;
68	  struct neighbor_entry *next;
69	  struct neighbor_entry *prev;
70	};
```

_____

58-69：结构体neighbor\_entry用来存储邻居节点的信息。分别存储了邻居的主地址、状态、转发意愿、覆盖两跳邻居的数量、节点连接的链路数量。其中还存储两个布尔类型的值用来判断该节点是否是MPR以及该MPR是否发生过改变。最后包含了指向两跳邻居列表的指针。

____

```c
49	struct neighbor_2_list_entry {
50	  struct neighbor_entry *nbr2_nbr;     /* backpointer to owning nbr entry */
51	  struct neighbor_2_entry *neighbor_2;
52	  struct timer_entry *nbr2_list_timer;
53	  struct neighbor_2_list_entry *next;
54	  struct neighbor_2_list_entry *prev;
55	};
```

____

49-54：结构体neighbor\_2\_list\_entry用来存储两跳节点列表中的信息。均采用指针的方式记录节点信息及结点的两跳邻居节点的信息以及列表的有效时间等。

### 2.3.2 节点操作

____

```c
119	struct link_entry *
120	lookup_link_entry(const union olsr_ip_addr *remote, const union olsr_ip_addr *remote_main, const struct interface *local)
121	{
122	  struct link_entry *link;
123
124	  OLSR_FOR_ALL_LINK_ENTRIES(link) {
125		if (ipequal(remote, &link->neighbor_iface_addr)
126			&& (link->if_name ? !strcmp(link->if_name, local->int_name) : ipequal(&local->ip_addr, &link->local_iface_addr))) {
127		  /* check the remote-main address only if there is one given */
128		  if (NULL != remote_main && !ipequal(remote_main, &link->neighbor->neighbor_main_addr)) {
129			/* Neighbor has changed it's main_addr, update */
130			struct ipaddr_str oldbuf, newbuf;
131			OLSR_PRINTF(1, "Neighbor changed main_ip, updating %s -> %s\n",
132						olsr_ip_to_string(&oldbuf, &link->neighbor->neighbor_main_addr), olsr_ip_to_string(&newbuf, remote_main));
133			link->neighbor->neighbor_main_addr = *remote_main;
134		  }
135		  return link;
136		}
137	  }
138	  OLSR_FOR_ALL_LINK_ENTRIES_END(link);
139
140	  return NULL;
```

____

160-168：该函数的功能是获取链路的状态，状态基于链路条目中的不同超时，根据不同的定时器值将返回不同的链路状态信息，如link\_sys\_timer对应链路状态为对称链路。

____

```c
179	static int
180	get_neighbor_status(const union olsr_ip_addr *address)
181	{
182	  const union olsr_ip_addr *main_addr;
183	  struct interface *ifs;
184
185	  /* Find main address */
186	  if (!(main_addr = mid_lookup_main_addr(address)))
187		main_addr = address;
188
189	  /* Loop trough local interfaces to check all possebilities */
190	  for (ifs = ifnet; ifs != NULL; ifs = ifs->int_next) {
191		struct mid_address *aliases;
192		struct link_entry *lnk = lookup_link_entry(main_addr, NULL, ifs);
193
194		if (lnk != NULL) {
195		  if (lookup_link_status(lnk) == SYM_LINK)
196			return SYM_LINK;
197		}
198
199		/* Get aliases */
200		for (aliases = mid_lookup_aliases(main_addr); aliases != NULL; aliases = aliases->next_alias) {
201
202		  lnk = lookup_link_entry(&aliases->alias, NULL, ifs);
203		  if (lnk && (lookup_link_status(lnk) == SYM_LINK)) {
204			return SYM_LINK;
205		  }
206		}
207	  }
208
209	  return 0;
210	}
211
```

____

182-197：通过查找main\_addr找到节点，然后通过lookup\_link\_status找到节点链路状态，并判断是否对称。

200-206：查找主地址并找出节点其他端口的IP，判断该节点其他端口的链路状态。并判断该IP所在的链路状态是否对称，并返回对称链路的信息。

### 2.3.3 操作邻居表

____

```c
56	void
57	olsr_init_neighbor_table(void)
58    {
59      int i;
60
61      for (i = 0; i < HASHSIZE; i++) {
62        neighbortable[i].next = &neighbortable[i];
63        neighbortable[i].prev = &neighbortable[i];
64      }
65    }
```

____

59-63：初始化邻居表。将每一个邻居表neighbortable[i]初始化为指向自身的仅有一个节点的链表。

____

```c
139	struct neighbor_2_list_entry *
140	olsr_lookup_my_neighbors(const struct neighbor_entry *neighbor, const union olsr_ip_addr *neighbor_main_address)
141	{
142	  struct neighbor_2_list_entry *entry;
143
144	  for (entry = neighbor->neighbor_2_list.next; entry != &neighbor->neighbor_2_list; entry = entry->next) {
145
146		if (ipequal(&entry->neighbor_2->neighbor_2_addr, neighbor_main_address))
147		  return entry;
148
149	  }
150	  return NULL;
151	}
```

____

142-150：该函数的功能是检查两跳邻居是否可以通过给定的邻居访问。遍历邻居节点的两跳邻居节点列表。如果找到列表中存在IP地址与给定的地址相匹配的两跳邻居节点，则返回该两跳邻居节点列表结构体。

____

```c
260	struct neighbor_entry *
261	olsr_lookup_neighbor_table(const union olsr_ip_addr *dst)
262	{
263	  /*
264	   *Find main address of node
265	   */
266	  union olsr_ip_addr *tmp_ip = mid_lookup_main_addr(dst);
267	  if (tmp_ip != NULL)
268		dst = tmp_ip;
269	  return olsr_lookup_neighbor_table_alias(dst);
270	}
```

____

271-274：该函数的功能是基于给定的地址在邻居表中查找邻居项。

____

```c
299	int
300	update_neighbor_status(struct neighbor_entry *entry, int lnk)
301	{
302	  /*
303	   * Update neighbor entry
304	   */
305
306	  if (lnk == SYM_LINK) {
307		/* N_status is set to SYM */
308		if (entry->status == NOT_SYM) {
309		  struct neighbor_2_entry *two_hop_neighbor;
310
311		  /* Delete posible 2 hop entry on this neighbor */
312		  if ((two_hop_neighbor = olsr_lookup_two_hop_neighbor_table(&entry->neighbor_main_addr)) != NULL) {
313			olsr_delete_two_hop_neighbor_table(two_hop_neighbor);
314		  }
315
316		  changes_neighborhood = true;
317		  changes_topology = true;
318		  if (olsr_cnf->tc_redundancy > 1)
319			signal_link_changes(true);
320		}
321		entry->status = SYM;
322	  } else {
323		if (entry->status == SYM) {
324		  changes_neighborhood = true;
325		  changes_topology = true;
326		  if (olsr_cnf->tc_redundancy > 1)
327			signal_link_changes(true);
328		}
329		/* else N_status is set to NOT_SYM */
330		entry->status = NOT_SYM;
331		/* remove neighbor from routing list */
332	  }
333
334	  return entry->status;
335	}
```

_____

该函数的功能为更新邻居的状态。

313-328：如果链路状态更新为SYM\_LINK时，原来非对称的链路通知网络重新选举MPR和更新路由表并删除通过该邻居节点到达的两跳邻居节点。

329-339：如果链路状态原先为SYS\_LINK，通知网络重新进行MPR的选取和路由表的更新。

____

```c
215	struct neighbor_entry *
216	olsr_insert_neighbor_table(const union olsr_ip_addr *main_addr)
217	{
218	  uint32_t hash;
219	  struct neighbor_entry *new_neigh;
220
221	  hash = olsr_ip_hashing(main_addr);
222
223	  /* Check if entry exists */
224
225	  for (new_neigh = neighbortable[hash].next; new_neigh != &neighbortable[hash]; new_neigh = new_neigh->next) {
226		if (ipequal(&new_neigh->neighbor_main_addr, main_addr))
227		  return new_neigh;
228	  }
229
230	  //printf("inserting neighbor\n");
231
232	  new_neigh = olsr_malloc(sizeof(struct neighbor_entry), "New neighbor entry");
233
234	  /* Set address, willingness and status */
235	  new_neigh->neighbor_main_addr = *main_addr;
236	  new_neigh->willingness = WILL_NEVER;
237	  new_neigh->status = NOT_SYM;
238
239	  new_neigh->neighbor_2_list.next = &new_neigh->neighbor_2_list;
240	  new_neigh->neighbor_2_list.prev = &new_neigh->neighbor_2_list;
241
242	  new_neigh->linkcount = 0;
243	  new_neigh->is_mpr = false;
244	  new_neigh->was_mpr = false;
245
246	  /* Queue */
247	  QUEUE_ELEM(neighbortable[hash], new_neigh);
248
249	  return new_neigh;
250	}
```

____

该函数的功能是在邻居节点表中插入一条邻居节点信息。如果已存在该信息返回插入成功则返回1。

230-233：检查表项是否存在。

237-249：添加邻居节点信息，将地址、意愿、状态等值初始化。

____

```c
160	int
161	olsr_delete_neighbor_table(const union olsr_ip_addr *neighbor_addr)
162	{
163	  struct neighbor_2_list_entry *two_hop_list, *two_hop_to_delete;
164	  uint32_t hash;
165	  struct neighbor_entry *entry;
166
167	  //printf("inserting neighbor\n");
168
169	  hash = olsr_ip_hashing(neighbor_addr);
170
171	  entry = neighbortable[hash].next;
172
173	  /*
174	   * Find neighbor entry
175	   */
176	  while (entry != &neighbortable[hash]) {
177		if (ipequal(&entry->neighbor_main_addr, neighbor_addr))
178		  break;
179
180		entry = entry->next;
181	  }
182
183	  if (entry == &neighbortable[hash])
184		return 0;
185
186	  two_hop_list = entry->neighbor_2_list.next;
187
188	  while (two_hop_list != &entry->neighbor_2_list) {
189		two_hop_to_delete = two_hop_list;
190		two_hop_list = two_hop_list->next;
191
192		two_hop_to_delete->neighbor_2->neighbor_2_pointer--;
193		olsr_delete_neighbor_pointer(two_hop_to_delete->neighbor_2, entry);
194
195		olsr_del_nbr2_list(two_hop_to_delete);
196	  }
197
198	  /* Dequeue */
199	  DEQUEUE_ELEM(entry);
200
201	  free(entry);
202
203	  changes_neighborhood = true;
204	  return 1;
205
206	}
```

____

165-211：该函数的功能是删除邻居表项，删除邻居表项的同时会删除掉该节点存储的两跳邻居列表。

## 2.4 MPR

OLSR采用MPR机制对路由信息进行选择性的洪泛。MPR作为OLSR协议的核心部分，算法描述已经在前面描述过了，此处不再赘述。

### 2.4.1 MPR生成

____

```c
357	static uint16_t
358	add_will_always_nodes(void)
359	{
360	  struct neighbor_entry *a_neighbor;
361	  uint16_t count = 0;
362
363	#if 0
364	  printf("\nAdding WILL ALWAYS nodes....\n");
365	#endif
366
367	  OLSR_FOR_ALL_NBR_ENTRIES(a_neighbor) {
368		struct ipaddr_str buf;
369		if ((a_neighbor->status == NOT_SYM) || (a_neighbor->willingness != WILL_ALWAYS)) {
370		  continue;
371		}
372		olsr_chosen_mpr(a_neighbor, &count);
373
374		OLSR_PRINTF(3, "Adding WILL_ALWAYS: %s\n", olsr_ip_to_string(&buf, &a_neighbor->neighbor_main_addr));
375
376	  }
377	  OLSR_FOR_ALL_NBR_ENTRIES_END(a_neighbor);
378
379	#if 0
380	  OLSR_PRINTF(1, "Count: %d\n", count);
381	#endif
382	  return count;
383	}
```

____

该函数的作用是添加willingness为WILLALWAYS的邻居节点到MPR集合中。

368-383：对于非对称以及转发意愿为WILLNEVER的邻居节点不进行处理，其余节点添加到MPR集合中，并返回添加的节点的数量。

____

```c
236	static void
237	olsr_clear_mprs(void)
238	{
239	  struct neighbor_entry *a_neighbor;
240	  struct neighbor_2_list_entry *two_hop_list;
241
242	  OLSR_FOR_ALL_NBR_ENTRIES(a_neighbor) {
243
244		/* Clear MPR selection. */
245		if (a_neighbor->is_mpr) {
246		  a_neighbor->was_mpr = true;
247		  a_neighbor->is_mpr = false;
248		}
249
250		/* Clear two hop neighbors coverage count/ */
251		for (two_hop_list = a_neighbor->neighbor_2_list.next; two_hop_list != &a_neighbor->neighbor_2_list;
252			 two_hop_list = two_hop_list->next) {
253		  two_hop_list->neighbor_2->mpr_covered_count = 0;
254		}
255
256	  }
257	  OLSR_FOR_ALL_NBR_ENTRIES_END(a_neighbor);
```

____

该函数的功能是清除被选为MPR的节点。

246-249：如果节点目前是MPR节点，那就将is\_mpr项设为false，was\_mpr设为true，表明该节点过去是MPR节点，现在不是了。

252-255：当删除一个MPR节点时，该节点存储的两跳邻居列表也应该清除。所以将邻居节点覆盖的两跳邻居节点的数量置0。

____

```c
140	static int
141	olsr_chosen_mpr(struct neighbor_entry *one_hop_neighbor, uint16_t * two_hop_covered_count)
142	{
143	  struct neighbor_list_entry *the_one_hop_list;
144	  struct neighbor_2_list_entry *second_hop_entries;
145	  struct neighbor_entry *dup_neighbor;
146	  uint16_t count;
147	  struct ipaddr_str buf;
148	  count = *two_hop_covered_count;
149
150	  OLSR_PRINTF(1, "Setting %s as MPR\n", olsr_ip_to_string(&buf, &one_hop_neighbor->neighbor_main_addr));
151
152	  //printf("PRE COUNT: %d\n\n", count);
153
154	  one_hop_neighbor->is_mpr = true;      //NBS_MPR;
155
156	  for (second_hop_entries = one_hop_neighbor->neighbor_2_list.next; second_hop_entries != &one_hop_neighbor->neighbor_2_list;
157		   second_hop_entries = second_hop_entries->next) {
158		dup_neighbor = olsr_lookup_neighbor_table(&second_hop_entries->neighbor_2->neighbor_2_addr);
159
160		if ((dup_neighbor != NULL) && (dup_neighbor->status == SYM)) {
161		  //OLSR_PRINTF(7, "(2)Skipping 2h neighbor %s - already 1hop\n", olsr_ip_to_string(&buf, &second_hop_entries->neighbor_2->neighbor_2_addr));
162		  continue;
163		}
164		//      if(!second_hop_entries->neighbor_2->neighbor_2_state)
165		//if(second_hop_entries->neighbor_2->mpr_covered_count < olsr_cnf->mpr_coverage)
166		//{
167		/*
168		   Now the neighbor is covered by this mpr
169		 */
170		second_hop_entries->neighbor_2->mpr_covered_count++;
171		the_one_hop_list = second_hop_entries->neighbor_2->neighbor_2_nblist.next;
172
173		//OLSR_PRINTF(1, "[%s](%x) has coverage %d\n", olsr_ip_to_string(&buf, &second_hop_entries->neighbor_2->neighbor_2_addr), second_hop_entries->neighbor_2, second_hop_entries->neighbor_2->mpr_covered_count);
174
175		if (second_hop_entries->neighbor_2->mpr_covered_count >= olsr_cnf->mpr_coverage)
176		  count++;
177
178		while (the_one_hop_list != &second_hop_entries->neighbor_2->neighbor_2_nblist) {
179
180		  if ((the_one_hop_list->neighbor->status == SYM)) {
181			if (second_hop_entries->neighbor_2->mpr_covered_count >= olsr_cnf->mpr_coverage) {
182			  the_one_hop_list->neighbor->neighbor_2_nocov--;
183			}
184		  }
185		  the_one_hop_list = the_one_hop_list->next;
186		}
187
188		//}
189	  }
190
191	  //printf("POST COUNT %d\n\n", count);
192
193	  *two_hop_covered_count = count;
194	  return count;
195
196	}
```

____

该函数的作用是用来处理已经选定的MPR节点，函数的返回值为一跳邻居节点中MPR的数量。

159-165：对second_hop_entries进行遍历

173：该邻居节点被MPR覆盖，将两跳邻居节点被MPR覆盖的数量+1。

181-188：如果两跳节点被MPR覆盖的数量大于全局变量mpr\_coverage，将count递减。

_____

```c
265	static int
266	olsr_check_mpr_changes(void)
267	{
268	  struct neighbor_entry *a_neighbor;
269	  int retval;
270
271	  retval = 0;
272
273	  OLSR_FOR_ALL_NBR_ENTRIES(a_neighbor) {
274
275		if (a_neighbor->was_mpr) {
276		  a_neighbor->was_mpr = false;
277
278		  if (!a_neighbor->is_mpr) {
279			retval = 1;
280		  }
281		}
282	  }
283	  OLSR_FOR_ALL_NBR_ENTRIES_END(a_neighbor);
284
285	  return retval;
286	}
```

____

该函数的作用是遍历所有的MPR节点判断其状态是否发生变化，若发生变化返回值为1，否则为0。

282-292：对于结点如果其was\_mpr值为true，表明它过去是MPR，is\_mpr值为false，表示现在不是MPR，所以它的状态发生了变化，将retval置为1

### 2.4.2 MPR选择

MPR的选择原则是MPR为对称邻居节点，通过MPR可以到达所有的两跳邻居节点，并且MPR的数量要尽可能少。接下来介绍有关两跳邻居节点的函数以及如何周到能够覆盖最多两跳节点的MPR。

____

```c
86 	static struct neighbor_2_list_entry *
87 	olsr_find_2_hop_neighbors_with_1_link(int willingness)
88 	{
89 
90 	  uint8_t idx;
91 	  struct neighbor_2_list_entry *two_hop_list_tmp = NULL;
92 	  struct neighbor_2_list_entry *two_hop_list = NULL;
93 	  struct neighbor_entry *dup_neighbor;
94 	  struct neighbor_2_entry *two_hop_neighbor = NULL;
95 
96 	  for (idx = 0; idx < HASHSIZE; idx++) {
97 
98 		for (two_hop_neighbor = two_hop_neighbortable[idx].next; two_hop_neighbor != &two_hop_neighbortable[idx];
99 			 two_hop_neighbor = two_hop_neighbor->next) {
100
101		  //two_hop_neighbor->neighbor_2_state=0;
102		  //two_hop_neighbor->mpr_covered_count = 0;
103
104		  dup_neighbor = olsr_lookup_neighbor_table(&two_hop_neighbor->neighbor_2_addr);
105
106		  if ((dup_neighbor != NULL) && (dup_neighbor->status != NOT_SYM)) {
107
108			//OLSR_PRINTF(1, "(1)Skipping 2h neighbor %s - already 1hop\n", olsr_ip_to_string(&buf, &two_hop_neighbor->neighbor_2_addr));
109
110			continue;
111		  }
112
113		  if (two_hop_neighbor->neighbor_2_pointer == 1) {
114			if ((two_hop_neighbor->neighbor_2_nblist.next->neighbor->willingness == willingness)
115				&& (two_hop_neighbor->neighbor_2_nblist.next->neighbor->status == SYM)) {
116			  two_hop_list_tmp = olsr_malloc(sizeof(struct neighbor_2_list_entry), "MPR two hop list");
117
118			  //OLSR_PRINTF(1, "ONE LINK ADDING %s\n", olsr_ip_to_string(&buf, &two_hop_neighbor->neighbor_2_addr));
119
120			  /* Only queue one way here */
121			  two_hop_list_tmp->neighbor_2 = two_hop_neighbor;
122
123			  two_hop_list_tmp->next = two_hop_list;
124
125			  two_hop_list = two_hop_list_tmp;
126			}
127		  }
128
129		}
130
131	  }
132
133	  return (two_hop_list_tmp);
134	}
135
```

____

该函数的作用是查找一条连接两跳邻居节点的链表。

92-95：定义了两个两跳邻居节点链表（two\_hop\_temp，two\_hop\_list），前者是函数的返回值。

Dup\_neighor记录邻居节点集合中已经存在的节点，two\_hop\_neighbor记录一个两跳邻居节点的局部变量。

97-112：遍历两跳邻居列表two\_hop\_neighbortable。

114-117：寻找邻居表中已经存在的邻居地址neighbor\_2\_addr，忽略与本节点不对称的邻居节点。

122-126：如果该两跳邻居节点不在两跳邻居链表中，且只有一个邻居节点，则将该两跳邻居节点加入链表。

____

```c
206	static struct neighbor_entry *
207	olsr_find_maximum_covered(int willingness)
208	{
209	  uint16_t maximum;
210	  struct neighbor_entry *a_neighbor;
211	  struct neighbor_entry *mpr_candidate = NULL;
212
213	  maximum = 0;
214
215	  OLSR_FOR_ALL_NBR_ENTRIES(a_neighbor) {
216
217	#if 0
218		printf("[%s] nocov: %d mpr: %d will: %d max: %d\n\n", olsr_ip_to_string(&buf, &a_neighbor->neighbor_main_addr),
219			   a_neighbor->neighbor_2_nocov, a_neighbor->is_mpr, a_neighbor->willingness, maximum);
220	#endif
221
222		if ((!a_neighbor->is_mpr) && (a_neighbor->willingness == willingness) && (maximum < a_neighbor->neighbor_2_nocov)) {
223
224		  maximum = a_neighbor->neighbor_2_nocov;
225		  mpr_candidate = a_neighbor;
226		}
227	  }
228	  OLSR_FOR_ALL_NBR_ENTRIES_END(a_neighbor);
229
230	  return mpr_candidate;
231	}
```

该函数的作用是找到能够覆盖最多两跳节点的MPR。

225-233：如果该一跳邻居节点不是MPR，而且该节点覆盖的两跳邻居节点的数量比maximum大，则更新maximum的值并将该一跳邻居节点作为候选MPR节点。

## 2.5 拓扑发现

网络中的MPR节点每隔一段时间会广播TC分组，用以维护网络的拓扑信息。在算法描述中已经说明，对于相同的TC分组,节点只有在第一次收到且其选择为MPR的情况下才转发，这样减少了网络中的广播包的数量，尽可能的避免了网络风暴。

### 2.5.1 TC分组生成

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

首先构建一个TC分组的结构体，然后使用olsr\_build\_tc\_packet（）函数对该结构体进行一些初始化和赋值操作，然后queue\_tc（）函数将该分组加入MID队列中，同时TIMED\_OUT()检测接口的时间的时间戳是否已满，调用set\_buffer\_timer()设置定时器。最后由接口ifn释放该分组，然后调用olsr\_free\_tc\_packet()函数释放内存。

### 2.5.2 拓扑信息集的初始化

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

6:avl\_init()初始化拓扑信息集为avl树。

11-18: 对拓扑表集合进行初始化，主要通过从cookie中获取的值为拓扑表集合的属性做初始化赋值。

23:调用olsr\_add\_tc\_entry（）配置一个entry,并将该entry加入avl树中。

### 2.5.3 TC分组处理

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

当节点接受到一个TC分组后，只考虑其消息类型。如果类型不等于LQ\_TC\_MESSAGE或者TC\_MESSAGE，则直接丢弃。

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

如果分组中的msg\_seq和外部变量msg\_seq相等，且ignored小于32，说明该分组已经处理过，所以丢弃该分组，返回false。

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

调用olsr\_calculate\_tc\_border()计算borderset的值，并且重置相关的定时器。

### 2.5.4 拓扑信息集的删除

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

该函数的功能是删除一个tc\_entry.

如果预定义了 LINUX\_NETLINK\_ROUTING(即在linux系统上运行)，则删除网关信息。删除时对网关的时间信息，网关协议等等先进行判断，判断这些信息是否为空后再进行删除。

首先删除本地路由表中的rt\_path；

清空所有的边，停止相应的计时器，将edge\_gc\_timer和validity\_time属性都置为空。

最后在avl树中删除相应节点。

## 2.6 路由表计算

### 2.6.1 相关结构体

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

rt\_metric:在路径选择时使用符合矩阵，矩阵中包括两个节点间的路径花销和跳数。

rt\_nexthop:该结构体表示吓一跳的网关和接口索引。

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

每一个RIB需要一个路由接口，这个接口中包含最佳路径的下一跳网关信息，同时该接口时rt\_path\_tree的根节点。同样也包含了所有路由信息中最佳的一个路径。rt\_dst包含该信息的路由地址和前缀长度。rt\_path\_tree是一个avl树，rt\_tree\_node表示最短路径的引用。

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

这个结构体主要描述了rt_path的成员，每接收到一个rt\_path就将其加入RIB。根据Dijkstra算法计算出的结果可以得到最优路径，同时可以得到一个最小的矩阵。rt\_path首先被加入到tc\_entry树中，如果根据Dijkstra计算可得当前tc\_entry是可达到，则在全局RIB树中加入下一跳地址。

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

### 2.6.2 路由表计算

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

首先调用avl\_init()将路由表初始化为avl树，然后维护一个版本号routingtree\_version用以检测每一个每一个rt\_entry和rt\_path，检查其是否过期，并将版本号初始化为0。

然后是为rt\_entry和rt\_path分配内存，并创建相应的cookie。

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

该函数功能是对于每个rt\_path创建一个路由表项并将其加入全局的RIB树中。

首先检查传入的参数是否合法，如果传入的tc\_entry的path\_cost为ROUTE\_COST\_BROKEN，或者传入的rtp的目的地址长度大于所设置的最大地址长度，直接返回NULL。

然后调用avl\_find()函数检查传入的rtp节点是否在路由表中，如果节点不在路由表中，将其加入avl树中。如果在，则将其从avl\_node类型转为rt\_entry类型。

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

然后将其从前缀树中删除，并解锁相应的tc\_entry。

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

首先调用avl\_walk\_first()函数从rt\_path\_tree中的到第一个条目并坚持其是否为0，然后把节点转为rt\_entry类型。

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

数功能是将一个前缀节点插入一颗前缀树。首先检查是否该rt\_path是已知的，如果不是，则创建，如果根据Dijkstra算法得到节点不可达，最后计算最短路径是不考虑。