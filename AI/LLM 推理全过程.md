下面是我帮你整理的一份**结构清晰、适合放进 Obsidian 的笔记文档（Markdown 格式）**，把我们整个讨论从 Agent → Prefill → Attention → QKV → h → Decode 全部串起来了👇

---

# 🧠 LLM 推理全过程（从输入到输出）

## 🎯 一句话总览

> **模型先通过 Prefill 理解上下文（构建 h + KV cache），再通过 Decode（自回归 + attention）逐步生成输出。**

---

# 🧩 1. 整体流程

```text
用户输入
   ↓
Agent 拼接上下文
   ↓
Prefill（理解输入）
   ↓
TTFT（第一个 token）
   ↓
Decode（逐 token 生成）
   ↓
最终输出
```

---

# 🧠 2. 核心概念

## 2.1 Token

- 模型处理的最小单位（不是字/词）
    
- 示例：
    

```text
“怎么快速赚钱”
→ ["怎么", "快速", "赚钱"]
```

---

## 2.2 Hidden State（h）

> **h = 模型当前对一个 token 的理解**

### 特点：

- 每个 token 都有一个 h
    
- 每一层都会更新 h
    
- h 是“长期状态”（会被保留）
    

### 示例：

```text
“苹果很好吃”

h₀：水果/公司（不确定）
h₁：可吃的东西
h₂：好吃的水果
h_final：用户在评价食物
```

---

## 2.3 Q / K / V

|符号|全称|作用|
|---|---|---|
|Q|Query|当前想找什么|
|K|Key|每个 token 的标签|
|V|Value|每个 token 的内容|

---

## 2.4 QKV 的来源

```text
h → 通过矩阵变换 → Q / K / V

Q = h × Wq  
K = h × Wk  
V = h × Wv
```

👉 同一个 token 会变成三种“视角”

---

# 🧩 3. Attention 机制

## 🎯 一句话

> **attention = 用 Q 匹配 K → 得到权重 → 加权 V**

---

## 分三步：

### ① 匹配（Q vs K）

```text
Q × K → 得到相关性分数
```

---

### ② 变成权重

```text
softmax → 概率分布
```

---

### ③ 加权 V

```text
输出 = Σ(权重 × V)
```

---

## 🔥 关键点

- K ❌ 不加权
    
- V ❌ 本身不变
    
- ✅ attention 输出 = 加权后的 V
    

---

# 🧩 4. Prefill 阶段

## 🎯 一句话

> **Prefill = 把所有输入 token 过一遍模型，并建立 KV cache**

---

## 做了什么？

对于每个 token：

```text
h → Q/K/V
   ↓
attention（看历史）
   ↓
更新 h
   ↓
存 K/V（KV cache）
```

---

## 特点

|特性|描述|
|---|---|
|计算方式|全量|
|复杂度|O(N²)|
|输出|所有 token 的 h|
|KV cache|构建|

---

## 🔥 为什么慢？

```text
token N → 看 N-1 个 token
→ 总计算量 ≈ N²
```

👉 Context 越大 → TTFT 越慢

---

# 🧩 5. KV Cache

## 🎯 一句话

> **KV cache = 缓存所有历史 token 的 Key 和 Value**

---

## 存的结构（按层）

```text
Layer1: K/V
Layer2: K/V
Layer3: K/V
...
```

---

## 作用

- 避免重复计算历史
    
- 加速 Decode
    

---

# 🧩 6. Decode（自回归生成）

## 🎯 一句话

> **每次生成一个 token，用 Q 去查 KV cache**

---

## 每一步：

```text
新 token：
  → 生成 Q
  → 匹配历史 K
  → 加权 V
  → 得到 h
  → 预测下一个 token
```

---

## 特点

|特性|描述|
|---|---|
|计算方式|增量|
|复杂度|O(N)|
|是否更新历史|❌ 不更新|
|使用 KV cache|✅|

---

# 🧩 7. Prefill vs Decode（核心对比）

||Prefill|Decode|
|---|---|---|
|计算对象|所有 token|新 token|
|attention 范围|全历史|新 → 历史|
|是否更新历史|✅|❌|
|KV cache|构建|使用|
|复杂度|O(N²) 🔥|O(N)|
|作用|理解输入|生成输出|

---

# 🧩 8. Transformer “层”的含义

## 🎯 一句话

> **一层 = attention + MLP 的组合模块**

---

## 结构

```text
h
 ↓
Attention（QKV）
 ↓
h'
 ↓
MLP
 ↓
h''
```

---

## 多层作用

```text
h₀ → h₁ → h₂ → ... → h_final
```

👉 每一层都在：

> **重新理解上下文**

---

# 🧩 9. 完整流程（结合例子）

## 输入：

```text
“怎么快速赚钱”
```

---

## Prefill：

```text
每个 token：
  h → Q/K/V
     ↓
  attention（融合上下文）
     ↓
  更新 h
```

---

## Decode：

```text
第1步：
→ “可以”

第2步：
→ “通过”

第3步：
→ “以下方式”
```

---

# 🎯 最终总结（核心理解）

> **LLM 的本质就是：**
> 
> - 用 attention 不断“理解上下文”（更新 h）
>     
> - 再用这些理解一步步生成输出
>     

---

# 🧩 最后一行人话总结

> **模型先把你说的话“想明白”（Prefill），然后一边回忆一边往下说（Decode）。**

---
