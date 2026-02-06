## Network Layer's job

transfer packets from source subnet to destination subnet

## Routing

### Store-and-Forward Packet Routing(存储转发式路由)

router **fully** receives the packet --> check route table --> transmit

In contrast to **traditional** switch, which receives frames and sends frames **at the same time**

### Routing Table

Packets routed individually & No advance set up (not be like wire)
**IP 的无连接特性**

content of routing table: **destination --> next hop**
![[images/Pasted image 20260202104506.png|500]]
router will update its routing table based on **routing algorithm** (e.g. A's table), when some "cost" in the network changed.
![[images/Pasted image 20260202105251.png]]
### Routing vs Forwarding

Forwarding: **Check routing table** to find appropriate outgoing interface for packet

Routing: **update routing tables** based on routing algorithm

### Dynamic Routing

Two routing algorithms
- Distance Vector (Bellman-Ford Shortest Path Algorithm)
- Link State (Dijkstra-Prim Shortest Path Algorithm)
#### Distance Vector

![[images/Pasted image 20260204101848.png]]

![[images/Pasted image 20260204101909.png]]

problem -- **count-to-infinity**

- **1. 路由器只信邻居**
- **2.邻居可能信息过期**
- **3.坏消息没有“刹车机制”**
