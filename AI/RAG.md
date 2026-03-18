

# 🧠 一句话理解 RAG

**RAG（Retrieval-Augmented Generation）= 用“外部知识库”增强大模型回答能力**

👉 不再只靠模型记忆，而是：  
**先查资料 → 再回答**

---

# 🔁 传统 RAG 的核心流程

## 标准 4 步：

```text
query → embedding → top-k → answer
```

我们一层一层拆开👇

---

# 1️⃣ Query（用户问题）

用户输入：

> “FlashLabs 的 voice 产品怎么接入知识库？”

这是系统的起点。

---

# 2️⃣ Embedding（把问题变成向量）

系统会把 query 转成一个向量（数字数组）：

👉 本质是：**把语义编码成一个点**

你可以理解成：

```text
"voice 接入 知识库"
↓
[0.12, -0.98, 0.33, ...]
```

同时，系统已经提前做过：

- 把所有文档切成 chunks
    
- 每个 chunk 也变成向量
    

---

## 📌 关键理解

embedding 的作用是：

👉 **让“语义相似”的文本在向量空间中靠近**

例如：

- “接入知识库”
    
- “连接数据源”
    

→ 向量距离会很近

---

# 3️⃣ Top-k 检索（找最相关的文档片段）

现在有：

- query 向量
    
- 一堆文档 chunk 向量
    

系统会计算相似度：

```text
similarity(query, chunk_i)
```

然后选出最相似的 k 个：

```text
Top-3 或 Top-5 chunks
```

👉 这一步就是：**从知识库里“找资料”**

---

## 🔍 举个例子

检索结果可能是：

- chunk 1：voice API 接入说明
    
- chunk 2：知识库对接架构
    
- chunk 3：示例代码
    

---

# 4️⃣ Answer（生成答案）

把这些 chunk 拼起来，喂给大模型：

```text
[chunk1 + chunk2 + chunk3] + 用户问题
→ LLM 生成回答
```

👉 模型现在是“带着资料开卷答题”

---

# 🔁 整体流程图（你可以背这个）

```text
用户问题
   ↓
向量化（embedding）
   ↓
向量相似度检索（top-k）
   ↓
拿到相关文档片段
   ↓
LLM 阅读这些片段
   ↓
生成答案
```

---


# ⚙️ 工程视角（你笔试要用的）

你可以把传统 RAG 看成 3 个模块：

## ① Indexing（离线）

- 文档 → chunk
    
- chunk → embedding
    
- 存入向量库
    

## ② Retrieval（在线）

- query → embedding
    
- 向量检索 → top-k
    

## ③ Generation（在线）

- 拼 context
    
- LLM 生成答案
    

---
