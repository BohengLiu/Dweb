# DHT是什么

# DHT协议
BEP-5  
协议解析: https://blog.csdn.net/qq_40306953/article/details/78035215
## Kademlia协议
Kademlia协议(模型)是被电驴，BitTorrent所采用了的，基于异或距离算法的分布式散列表(DHT), 它实现了一个去中心化的信息储存与查询系统。

Kademlia将网络设计为一个具有160层的二叉树，树最末端的每个叶子便是一个节点，节点在树中的位置由同样是160bit的节点ID决定。
每个bit的两种可能值(0或1), 决定了节点在树中属于左面还是右面的子树，160层下来，每个节点ID便都有了一个确定的位置。

Kademlia使用独特的异或距离算法来计算节点间的距离，异或是一种简单的数学计算，它有很多独特的性质，这些性质在之后会为我们带来方便：
```
自己与自己的距离为0:
x ^ x = 0
不同的节点间必有距离:
x ^ y > 0
交换律，x到y的距离等于y到x的距离:
x ^ y = y ^ x
从a经b绕到c, 要比直接从a到c距离长:
a ^b + b ^ c >= a ^ c
下面两个是资料上提到的，似乎很重要，但我不大理解他们的含义:
a + b >= a ^ b
(a ^ b) ^ (b ^ c) = a ^ c
```
在Kademlia中，异或(距离)算法具有单向性(或者说一一对应关系)，即给定一个节点和一个距离，必定存在唯一一个相对应节点。包括距离算法在内的，Kademlia中大部分的概念，都既有算术上的意义，又可以在节点树上表现实际意义。
事实上，节点间距离反映的就是节点ID中比特的差异情况，而且越靠前的比特权值越大。或者说是反映节点在树中相隔了多少个分支，需要向上多少个树节点才能找到共同的祖先节点。

Kademlia中使用了名为K-桶的概念来储存其他(临近)节点的状态信息，这里的状态信息主要指的就是节点ID, IP, 和端口。
对于160bit的节点ID, 就有160个K-桶，对于每一个K-桶i, 它会储存与自己距离在区间 [2^i, 2^(i+1)) 范围内的节点的信息，每个K-桶中储存有k个其他节点的信息，在BitTorrent的实现中，k的取值为8.

下表反映了每个K-桶所储存的信息

K-桶|	储存的距离区间|	储存的距离范围|	储存比率
-|-|-|-|
0   |	[20, 21)    |	1         |	100%
1	  | [21, 22)    |	2-3       |	100%
2	  | [22, 23)    |	4-7       |	100%
3	  | [23, 24)	  | 8-15	    | 100%
4	  | [24, 25)	  | 16-31	    | 75%
5	  | [25, 26)	  | 32-63	    | 57%
10  |	[210, 211)  |	1024-2047	| 13%
i	  | [2i, 2i+1)  |	/	        |0.75<sup>i-3</sup>

放在节点树上，即每个节点都更倾向于储存与自己距离近的节点的信息，形成 储存的离自己近的节点多, 储存离自己远的节点少 的局面。
从上表可以看出，在1-15这个范围内的节点，只要发现，就会被100%地储存下来，而离自己距离在1000左右的节点，只会储存13%.

对于一个节点而言，K-桶就代表着节点树上那些未知的节点(其实除了自己都是未知的)构成的子树，160个K桶分别是具有1到160层的子树，由小至大。对于节点ID, 160个K-桶分别储存着节点ID前0到159个bit和自己一致的节点。

K-桶中的条目(其他临近节点的状态信息)排序的，每当收到一个消息(如查询)时，就要更新一次K桶。
首先计算自己与对方的距离，然后储存到对应的K-桶中，但如果K-桶已满(前面提到每个K-桶储存有k=8个条目), 则会倾向放弃储存，继续保持旧的节点信息(如果还有效的话). 除了距离外，Kademlia更倾向于与在线时间长，稳定的节点建立联系。
这是因为实践证明，累积在线时间越长的节点越稳定，越倾向于继续保持在线，即节点的失效概率和在线时长成反比。
这样还可以在一定程度上抵御攻击行为。因为当大量恶意的新节点涌入时，大家都会选择继续保持旧的节点信息，而不是接受新的。
除此之外，还需要定时检查K-桶中的节点是否任然在线，及时删去已失效节点。

Kademlia协议仅定义了四种操作：

- PING: 探测一个节点是否在线
- STORE: 令对方储存一份数据
- FIND NODE: 根据节点ID查找一个节点
- FIND VALUE: 根据键查找一个值(数据)

当查找一个节点时，首先计算自己与目标节点的距离d, 然后将 log2d 向下取整，找到对应的K-桶，从这个K-桶中选取a个节点(在BitTorrent的实现中取值为3), 向它们发送查询。
收到查询的节点同样计算距离后从自己的对应K-桶中选取a个节点返回给查询者。查询者不断重复这个过程，知道找到目标节点，或无法再找到更近的结果。
很多资料将这个过程描述成是递归的，但我觉得这里认为它是迭代的更为恰当。

因为每个节点都更倾向于储存距自己近的节点的信息，而整个网络又是一个二叉树，因此每次查询都会至少取得距离减半的结果，对于有N个节点的网络，至多只需要 log2N 次查询。

当进行 FIND VALUE 操作时，查询操作是类型的，每份数据都有一个同样是160bit的键，没分数据都倾向于储存在与键值距离较近的节点上。
当上传者上传一份数据时，上传者首先定位几个较为接近键值的节点，用STORE操作要求他们储存这份数据。
储存有数据的节点，每当发现比自己与键值距离更为接近的节点时，便将数据复制一份到这个节点上。

当一个新节点欲加入网络时，只需找到一个已在网络中的节点，借助它对自己的节点ID进行一次常规查询即可，这样便完成了对自己信息的广播，让距自己较近的节点获知自己的存在。而离开网络不必执行任何操作，一段时间后，你的信息会自动地从其他节点被删除。

Kademlia的精妙之处在于它选择了异或运算作为计算距离的依据，异或运算不仅具有算术的意义，在二叉树式的网络模型中，同样具有实际意义，同时任何情况下都在强调距离的概念，让节点间通过距离来聚合起来。

### DHT爬虫项目
node.js 的DHT 嗅探爬虫:　https://github.com/78/ssbc/

## DHT协议内容

### 前言
做了一个磁力链接和BT种子的搜索引擎 {Magnet & Torrent}，因此把 DHT 协议重新看了一遍。
```
BEP: 5
Title: DHT Protocol 
Version: 3dec52cb3ae103ce22358e3894b31cad47a6f22b
Last-Modified: Tue Apr 2 16:51:45 2013 -0700
Author: Andrew Loewenstern drue@bittorrent.com, Arvid Norberg arvid@bittorrent.com
Status: Draft
Type: Standards Track
Created: 31-Jan-2008
Post-History: 22-March-2013: Add “implied_port” to announce_peer message, to improve NAT support
```
BitTorrent 使用”分布式哈希表”(DHT)来为无 tracker 的种子(torrents)存储 peer 之间的联系信息。这样每个 peer 都成了 tracker。这个协议基于 Kademila[1] 网络并且在 UDP 上实现。

请注意本文档中使用的术语，以免混乱。

- “peer” 是在一个 TCP 端口上监听的客户端/服务器，它实现了 BitTorrent 协议。
- “节点” 是在一个 UDP 端口上监听的客户端/服务器，它实现了 DHT(分布式哈希表) 协议。

DHT 由节点组成，它存储了 peer 的位置。BitTorrent 客户端包含一个 DHT 节点，这个节点用来联系 DHT 中其他节点，从而得到 peer 的位置，进而通过 BitTorrent 协议下载。

### 概述 Overview
每个节点有一个全局唯一的标识符，作为 “node ID”。节点 ID 是一个随机选择的 160bit 空间，BitTorrent infohash[2] 也使用这样的 160bit 空间。 “距离”用来比较两个节点 ID 之间或者节点 ID 和 infohash 之间的”远近”。节点必须维护一个路由表，路由表中含有一部分其它节点的联系信息。其它节点距离自己越近时，路由表信息越详细。因此每个节点都知道 DHT 中离自己很”近”的节点的联系信息，而离自己非常远的 ID 的联系信息却知道的很少。

在 Kademlia 网络中，距离是通过异或(XOR)计算的，结果为无符号整数。distance(A, B) = |A xor B|，值越小表示越近。

当节点要为 torrent 寻找 peer 时，它将自己路由表中的节点 ID 和 torrent 的 infohash 进行”距离对比”。然后向路由表中离 infohash 最近的节点发送请求，问它们正在下载这个 torrent 的 peer 的联系信息。如果一个被联系的节点知道下载这个 torrent 的 peer 信息，那个 peer 的联系信息将被回复给当前节点。否则，那个被联系的节点则必须回复在它的路由表中离该 torrent 的 infohash 最近的节点的联系信息。最初的节点重复地请求比目标 infohash 更近的节点，直到不能再找到更近的节点为止。查询完了之后，客户端把自己作为一个 peer 插入到所有回复节点中离种子最近的那个节点中。

请求 peer 的返回值包含一个不透明的值，称之为”令牌(token)”。如果一个节点宣布它所控制的 peer 正在下载一个种子，它必须在回复请求节点的同时，附加上对方向我们发送的最近的”令牌(token)”。这样当一个节点试图”宣布”正在下载一个种子时，被请求的节点核对令牌和发出请求的节点的 IP 地址。这是为了防止恶意的主机登记其它主机的种子。由于令牌仅仅由请求节点返回给收到令牌的同一个节点，所以没有规定他的具体实现。但是令牌必须在一个规定的时间内被接受，超时后令牌则失效。在 BitTorrent 的实现中，token 是在 IP 地址后面连接一个 secret(通常是一个随机数)，这个 secret 每五分钟改变一次，其中 token 在十分钟以内是可接受的。

### 路由表 Routing Table
每个节点维护一个路由表保存已知的好节点。路由表中的节点是用来作为在 DHT 中请求的起始点。路由表中的节点是在不断的向其他节点请求过程中，对方节点回复的。

并不是我们在请求过程中收到得节点都是平等的，有的节点是好的，而另一些则不是。许多使用 DHT 协议的节点都可以发送请求并接收回复，但是不能主动回复其他节点的请求。节点的路由表只包含已知的好节点，这很重要。好节点是指在过去的 15 分钟以内，曾经对我们的某一个请求给出过回复的节点，或者曾经对我们的请求给出过一个回复(不用在15分钟以内)，并且在过去的 15 分钟给我们发送过请求。上述两种情况都可将节点视为好节点。在 15 分钟之后，对方没有上述 2 种情况发生，这个节点将变为可疑的。当节点不能给我们的一系列请求给出回复时，这个节点将变为坏的。相比那些未知状态的节点，已知的好节点会被给于更高的优先级。

路由表覆盖从 0 到 2^160 全部的节点 ID 空间。路由表又被划分为桶(bucket)，每个桶包含一部分的 ID 空间。空的路由表只有一个桶，它的 ID 范围从 min=0 到 max=2^160。当 ID 为 N 的节点插入到表中时，它将被放到 ID 范围在 min <= N < max 的 桶 中。空的路由表只有一个桶，所以所有的节点都将被放到这个桶中。每个桶最多只能保存 K 个节点，当前 K=8。当一个桶放满了好节点之后，将不再允许新的节点加入，除非我们自身的节点 ID 在这个桶的范围内。在这样的情况下，这个桶将被分裂为 2 个新的桶，每个新桶的范围都是原来旧桶的一半。原来旧桶中的节点将被重新分配到这两个新的桶中。如果一个新表只有一个桶，这个包含整个范围的桶将总被分裂为 2 个新的桶，每个桶的覆盖范围从 0..2^159 和 2^159..2^160。

当桶装满了好节点，新的节点会被丢弃。一旦桶中的某个节点变为了坏的节点，那么我们就用新的节点来替换这个坏的节点。如果桶中有在 15 分钟内都没有活跃过的节点，我们将这样的节点视为可疑的节点，这时我们向最久没有联系的节点发送 ping。如果被 ping 的节点给出了回复，那么我们向下一个可疑的节点发送 ping，不断这样循环下去，直到有某一个节点没有给出 ping 的回复，或者当前桶中的所有节点都是好的(也就是所有节点都不是可疑节点，他们在过去 15 分钟内都有活动)。如果桶中的某个节点没有对我们的 ping 给出回复，我们最好再试一次(再发送一次 ping，因为这个节点也许仍然是活跃的，但由于网络拥塞，所以发生了丢包现象，注意 DHT 的包都是 UDP 的)，而不是立即丢弃这个节点或者直接用新节点来替代它。这样，我们得路由表将充满稳定的长时间在线的节点。

每个桶都应该维持一个 lastchange 字段来表明桶中节点的”新鲜”度。当桶中的节点被 ping 并给出了回复，或者一个节点被加入到了桶，或者一个节点被新的节点所替代，桶的 lastchange 字段都应当被更新。如果一个桶的 lastchange 在过去的 15 分钟内都没有变化，那么我们将更新它。这个更新桶操作是这样完成的：从这个桶所覆盖的范围中随机选择一个 ID，并对这个 ID 执行 find_nodes 查找操作。常常收到请求的节点通常不需要常常更新自己的桶，反之，不常常收到请求的节点常常需要周期性的执行更新所有桶的操作，这样才能保证当我们用到 DHT 的时候，里面有足够多的好的节点。

在插入第一个节点到路由表并启动服务后，这个节点应试着查找 DHT 中离自己更近的节点，这个查找工作是通过不断的发出 find_node 消息给越来越近的节点来完成的，当不能找到更近的节点时，这个扩散工作就结束了。路由表应当被启动工作和客户端软件保存（也就是启动的时候从客户端中读取路由表信息，结束的时候客户端软件记录到文件中）。

### BitTorrent 协议扩展 BitTorrent Protocol Extension
BitTorrent 协议已经被扩展为可以在通过 tracker 得到的 peer 之间互相交换节点的 UDP 端口号(也就是告诉对方我们的 DHT 服务端口号)，在这样的方式下，客户端可以通过下载普通的种子文件来自动扩展 DHT 路由表。新安装的客户端第一次试着下载一个无 tracker 的种子时，它的路由表中将没有任何节点，这是它需要在 torrent 文件中找到联系信息。

peers 如果支持 DHT 协议就将 BitTorrent 协议握手消息的保留位的第 8 字节的最后一位置为 1。这时如果 peer 收到一个 handshake 表明对方支持 DHT 协议，就应该发送 PORT 消息。它由字节 0x09 开始，payload 的长度是 2 个字节，包含了这个 peer 的 DHT 服务使用的网络字节序的 UDP 端口号。当 peer 收到这样的消息是应当向对方的 IP 和消息中指定的端口号的节点发送 ping。如果收到了 ping 的回复，那么应当使用上述的方法将新节点的联系信息加入到路由表中。

### Torrent 文件扩展 Torrent File Extensions
一个无 tracker 的 torrent 文件字典不包含 announce 关键字，而使用 nodes 关键字来替代。这个关键字对应的内容应该设置为 torrent 创建者的路由表中 K 个最接近的节点。可供选择的，这个关键字也可以设置为一个已知的可用节点，比如这个 torrent 文件的创建者。请不要自动加入 router.bittorrent.com 到 torrent 文件中或者自动加入这个节点到客户端路由表中。

nodes = [["<host>", <port>], ["<host>", <port>], ...]
nodes = [["127.0.0.1", 6881], ["your.router.node", 4804]]

### KRPC 协议 KRPC Protocol
KRPC 协议是由 bencode 编码组成的一个简单的 RPC 结构，他使用 UDP 报文发送。一个独立的请求包被发出去然后一个独立的包被回复。这个协议没有重发。它包含 3 种消息：请求，回复和错误。对DHT协议而言，这里有 4 种请求：ping，find_node，get_peers 和 announce_peer。

一条 KRPC 消息由一个独立的字典组成，其中有 2 个关键字是所有的消息都包含的，其余的附加关键字取决于消息类型。每条消息都包含 t 关键字，它是一个代表了 transaction ID 的字符串类型。transaction ID 由请求节点产生，并且回复中要包含回显该字段，所以回复可能对应一个节点的多个请求。transaction ID 应当被编码为一个短的二进制字符串，比如 2 个字节，这样就可以对应 2^16 个请求。另外每个 KRPC 消息还应该包含的关键字是 y，它由一个字节组成，表明这个消息的类型。y 对应的值有三种情况：q 表示请求，r 表示回复，e 表示错误。

### 联系信息编码 Contact Encoding
Peers 的联系信息被编码为 6 字节的字符串。又被称为 “CompactIP-address/port info”，其中前 4 个字节是网络字节序的 IP 地址，后 2 个字节是网络字节序的端口。

节点的联系信息被编码为 26 字节的字符串。又被称为 “Compactnode info”，其中前 20 字节是网络字节序的节点 ID，后面 6 个字节是 peers 的 “CompactIP-address/port info”。

**联系信息编码 Contact Encoding**

Peers 的联系信息被编码为 6 字节的字符串。又被称为 “CompactIP-address/port info”，其中前 4 个字节是网络字节序的 IP 地址，后 2 个字节是网络字节序的端口。

节点的联系信息被编码为 26 字节的字符串。又被称为 “Compactnode info”，其中前 20 字节是网络字节序的节点 ID，后面 6 个字节是 peers 的 “CompactIP-address/port info”。

**请求 Queries**

请求，对应于 KPRC 消息字典中的 y 关键字的值是 q，它包含 2 个附加的关键字 q 和 a。关键字 q 是字符串类型，包含了请求的方法名字。关键字 a 一个字典类型包含了请求所附加的参数。

**回复 Responses**

回复，对应于 KPRC 消息字典中的 y 关键字的值是 r，包含了一个附加的关键字 r。关键字 r 是字典类型，包含了返回的值。发送回复消息是在正确解析了请求消息的基础上完成的。

**错误 Errors**

错误，对应于 KPRC 消息字典中的 y 关键字的值是 e，包含一个附加的关键字 e。关键字 e 是列表类型。第一个元素是数字类型，表明了错误码。第二个元素是字符串类型，表明了错误信息。当一个请求不能解析或出错时，错误包将被发送。下表描述了可能出现的错误码：

错误码 |	描述
-|-
201 |	一般错误
202 |	服务错误
203 |	协议错误，比如不规范的包，无效的参数，或者错误的 token
204 |	未知方法

**错误包例子 Example Error Packets:**

- generic error = {"t":"aa", "y":"e", "e":[201, "A Generic Error Ocurred"]}
- bencoded = d1:eli201e23:A Generic Error Ocurrede1:t2:aa1:y1:ee

### DHT 请求 DHT Queries
所有的请求都包含一个关键字 id，它包含了请求节点的节点 ID。所有的回复也包含关键字 id，它包含了回复节点的节点 ID。

**ping**

最基础的请求就是 ping。这时 KPRC 协议中的 "q" = "ping"。Ping 请求包含一个参数 id，它是一个 20 字节的字符串包含了发送者网络字节序的节点 ID。对应的 ping 回复也包含一个参数 id，包含了回复者的节点 ID。

- 参数: {"id" : "<querying nodes id>"}
- 回复: {"id" : "<queried nodes id>"}

**报文包例子 Example Packets**

- ping Query = {"t":"aa", "y":"q", "q":"ping", "a":{"id":"abcdefghij0123456789"}}
- bencoded = d1:ad2:id20:abcdefghij0123456789e1:q4:ping1:t2:aa1:y1:qe
- Response = {"t":"aa", "y":"r", "r": {"id":"mnopqrstuvwxyz123456"}}
- bencoded = d1:rd2:id20:mnopqrstuvwxyz123456e1:t2:aa1:y1:re

**find_node**  

find_node 被用来查找给定 ID 的节点的联系信息。这时 KPRC 协议中的 "q" == "find_node"。find_node 请求包含 2 个参数，第一个参数是 id，包含了请求节点的ID。第二个参数是 target，包含了请求者正在查找的节点的 ID。当一个节点接收到了 find_node 的请求，他应该给出对应的回复，回复中包含 2 个关键字 id 和 nodes，nodes 是字符串类型，包含了被请求节点的路由表中最接近目标节点的 K(8) 个最接近的节点的联系信息。

- 参数: {"id" : "<querying nodes id>", "target" : "<id of target node>"}
- 回复: {"id" : "<queried nodes id>", "nodes" : "<compact node info>"}

**报文包例子 Example Packets**

- find_node Query = {"t":"aa", "y":"q", "q":"find_node", "a": {"id":"abcdefghij0123456789", "target":"mnopqrstuvwxyz123456"}}
- bencoded = d1:ad2:id20:abcdefghij01234567896:target20:mnopqrstuvwxyz123456e1:q9:find_node1:t2:aa1:y1:qe
- Response = {"t":"aa", "y":"r", "r": {"id":"0123456789abcdefghij", "nodes": "def456..."}}
- bencoded = d1:rd2:id20:0123456789abcdefghij5:nodes9:def456...e1:t2:aa1:y1:re

**get_peers**

get_peers 与 torrent 文件的 infohash 有关。这时 KPRC 协议中的 "q" = "get_peers"。get_peers 请求包含 2 个参数。第一个参数是 id，包含了请求节点的 ID。第二个参数是 info_hash，它代表 torrent 文件的 infohash。如果被请求的节点有对应 info_hash 的 peers，他将返回一个关键字 values，这是一个列表类型的字符串。每一个字符串包含了 "CompactIP-address/portinfo" 格式的 peers 信息。如果被请求的节点没有这个 infohash 的 peers，那么他将返回关键字 nodes，这个关键字包含了被请求节点的路由表中离 info_hash 最近的 K 个节点，使用 "Compactnodeinfo" 格式回复。在这两种情况下，关键字 token 都将被返回。token 关键字在今后的 annouce_peer 请求中必须要携带。token 是一个短的二进制字符串。

- 参数: {"id" : "<querying nodes id>", "info_hash" : "<20-byte infohash of target torrent>"}
- 回复: {"id" : "<queried nodes id>", "token" :"<opaque write token>", "values" : ["<peer 1 info string>", "<peer 2 info string>"]}
- 或: {"id" : "<queried nodes id>", "token" :"<opaque write token>", "nodes" : "<compact node info>"}

**报文包例子 Example Packets:**

- get_peers Query = {"t":"aa", "y":"q", "q":"get_peers", "a": {"id":"abcdefghij0123456789", "info_hash":"mnopqrstuvwxyz123456"}}
- bencoded = d1:ad2:id20:abcdefghij01234567899:info_hash20:mnopqrstuvwxyz123456e1:q9:get_peers1:t2:aa1:y1:qe
- Response with peers = {"t":"aa", "y":"r", "r": {"id":"abcdefghij0123456789", "token":"aoeusnth", "values": ["axje.u", "idhtnm"]}}
- bencoded = d1:rd2:id20:abcdefghij01234567895:token8:aoeusnth6:valuesl6:axje.u6:idhtnmee1:t2:aa1:y1:re
- Response with closest nodes = {"t":"aa", "y":"r", "r": {"id":"abcdefghij0123456789", "token":"aoeusnth", "nodes": "def456..."}}
- bencoded = d1:rd2:id20:abcdefghij01234567895:nodes9:def456...5:token8:aoeusnthe1:t2:aa1:y1:re

**announce_peer**

这个请求用来表明发出 announce_peer 请求的节点，正在某个端口下载 torrent 文件。announce_peer 包含 4 个参数。第一个参数是 id，包含了请求节点的 ID；第二个参数是 info_hash，包含了 torrent 文件的 infohash；第三个参数是 port 包含了整型的端口号，表明 peer 在哪个端口下载；第四个参数数是 token，这是在之前的 get_peers 请求中收到的回复中包含的。收到 announce_peer 请求的节点必须检查这个 token 与之前我们回复给这个节点 get_peers 的 token 是否相同。如果相同，那么被请求的节点将记录发送 announce_peer 节点的 IP 和请求中包含的 port 端口号在 peer 联系信息中对应的 infohash 下。

- 参数: {"id" : "<querying nodes id>", "implied_port": <0 or 1>, "info_hash" : "<20-byte infohash of target torrent>", "port" : <port number>, "token" : "<opaque token>"}
- 回复: {"id" : "<queried nodes id>"}

**报文包例子 Example Packets:**

- announce_peers Query = {"t":"aa", "y":"q", "q":"announce_peer", "a": {"id":"abcdefghij0123456789", "implied_port": 1, "info_hash":"mnopqrstuvwxyz123456", "port": 6881, "token": "aoeusnth"}}
- bencoded = d1:ad2:id20:abcdefghij01234567899:info_hash20:<br /> mnopqrstuvwxyz1234564:porti6881e5:token8:aoeusnthe1:q13:announce_peer1:t2:aa1:y1:qe
- Response = {"t":"aa", "y":"r", "r": {"id":"mnopqrstuvwxyz123456"}}
- bencoded = d1:rd2:id20:mnopqrstuvwxyz123456e1:t2:aa1:y1:re

### References
[1] Peter Maymounkov, David Mazieres, “Kademlia: A Peer-to-peer Information System Based on the XOR Metric”, IPTPS 2002. http://www.cs.rice.edu/Conferences/IPTPS02/109.pdf  
[2] Use SHA1 and plenty of entropy to ensure a unique ID.

## Q&A
1. 整个dht的过程是建立在udp上的，而不是udp和tcp同时使用。