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
![[Pasted image 20260202104506.png|500]]
router will update its routing table based on **routing algorithm** (e.g. A's table), when some "cost" in the network changed.
![[Pasted image 20260202105251.png]]
# **Routing（路由）笔记整理**

---

## **一、Routing Algorithm（路由算法）**

### **1. 什么是 Routing Algorithm**

- 决定 **到达路由器的数据包** 应该从 **哪个出口接口** 发出
    
- 依据：**Routing Table（路由表）**
    
- 可能对 **每一个到达的数据包** 做一次决定
    

> Routing algorithm = 决策规则

> Routing table = 决策结果

---

### **2. Routing vs. Forwarding（高频考点）**

#### **Forwarding（转发）**

- 功能：
    
    - 查路由表
        
    - 将数据包从正确的接口发出
        
    
- 特点：
    
    - 每个数据包都会发生
        
    - 速度快
        
    - 不做复杂计算
        
    
#### **Routing（路由）**

- 功能：
    
    - 初始化、维护、更新路由表
        
    
- 方式：
    
    - 路由器之间交换**控制报文（小数据包）**
        
    - 运行路由算法
        
    
- 特点：
    
    - 不一定每个数据包都发生
        
    - 关注全局路径选择
        
    

**一句话对比**：

- Forwarding：执行层（查表 + 转发）
    
- Routing：控制层（算路 + 建表）
    

---

### **3. 好的 Routing Algorithm 的性质**

- Correctness（正确性）
    
- Simplicity（简单性）
    
- Robustness（鲁棒性）
    
- Stability（稳定性，避免震荡）
    
- Fairness（公平性）
    
- Optimality（最优性）
    

  

> 重要取舍：**Fairness vs. Optimality**

---

### **4. Adaptive vs. Non-adaptive Routing**

#### **Adaptive Routing（动态路由）**

- 根据以下因素动态调整路由：
    
    - 网络拓扑
        
    - 当前流量
        
    - 链路状态
        
    
- 特点：
    
    - 灵活、智能
        
    - 实现复杂
        
    
- 例子：RIP、OSPF、BGP
    

  

#### **Non-adaptive Routing（静态路由）**

- 由管理员手工配置固定路径
    
- 可设置备份路由
    
- 特点：
    
    - 简单、稳定
        
    - 不适应网络变化
        
    

---

## **二、Inter-domain vs. Intra-domain Routing**

### **1. Domain（域）的概念**

- Domain：
    
    - 由同一管理者控制的一组网络
        
    - 实际中≈ AS（Autonomous System，自主系统）
        
    

---

### **2. Intra-domain Routing（域内路由）**

  

**定义**：在同一个 Domain 内部进行的路由

  

特点：

- Complete knowledge（对内部拓扑了解完整）
    
- 追求 optimal path（最优路径）
    
- 规模较小（~100 个网络）
    
- 主要依赖动态路由算法
    

  

常见协议：

- RIP
    
- OSPF
    
- IS-IS
    

  

例子：

- ISP 内部网络路由
    
- 企业内部网络
    

---

### **3. Inter-domain Routing（域间路由）**

**定义**：不同 Domain 之间的路由

特点：

- Aggregated knowledge（只知道方向，不知道内部细节）
    
- 规模巨大（整个 Internet）
    
- **由 Policy（策略）主导**
    
Policy 示例：

- 优先通过某个网络 X
    
- X 不可用时再走 Y
    
- 禁止某些方向的流量转发
    

特点总结：

- 决策依据不是最短路径，而是商业/管理策略
    
- 路由规则可能非常复杂
    

主要协议：

- BGP（Border Gateway Protocol）
    

---

### **4. Intra-domain vs. Inter-domain 对比表**

|**对比项**|**Intra-domain**|**Inter-domain**|
|---|---|---|
|范围|单一 Domain|多个 Domain|
|规模|小|极大|
|知识|完整|聚合|
|目标|最优路径|策略优先|
|路由方式|动态为主|策略控制|
|协议|RIP / OSPF|BGP|

---

## **三、整体框架速记**

```
Routing
 ├─ Routing vs. Forwarding
 ├─ Static vs. Dynamic Routing
 ├─ Intra-domain Routing（RIP / OSPF）
 └─ Inter-domain Routing（BGP, Policy）
```

---

📌 **复习建议**：

- 重点记忆概念区分（Routing vs Forwarding）
    
- 理解 Intra-domain 和 Inter-domain 的目标差异
    
- Policy 是 Inter-domain 的核心关键词