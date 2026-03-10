+++
date = '2025-12-11T08:01:06+08:00'
draft = false
ShowToc = true
TocOpen = true
title = 'Spring AI 全景指南（三）解构 RAG —— 从 BGE 向量模型到高维语义检索'
+++

在上一篇中，我们通过 ChatClient 实现了与大模型的基础对话。然而，通用LLM(如 Qwen、DeepSeek)存在一个致命缺陷：它是“无状态”的，且“知识截止”的。它不知道你公司内部的 2025年KPI制度，也无法实时读取最新的数据库变动。要解决这一问题，不能仅靠微调(Fine-tuning)，更轻量级、工程化的方案是 RAG(Retrieval-Augmented Generation，检索增强生成)。

本文将深入RAG的心脏——向量(Vector)，剖析Embedding模型与LLM的本质区别，并详解 Spring AI中向量检索的核心参数与SQL实现逻辑。

## 一、“制图师”与“作家”：Embedding 模型 vs LLM

在 RAG 架构中，我们通常需要部署两类完全不同的模型。很多开发者容易混淆它们的职责，例如将 BGE (BAAI General Embedding) 与 Qwen (通义千问) 混为一谈。

**Embedding模型 (如 BGE-M3, text-embedding-3)**
    
角色：它是制图师。

功能：它不负责说话，只负责将文本“压缩”成一串高维浮点数数组(向量)。它致力于将语义相似的文本映射到向量空间中距离相近的坐标点。

输入/输出： 输入文本->输出 \[0.012, -0.98, ...\]。

**LLM (如Qwen, DeepSeek-V3)**
    
角色：它是作家。

功能：它负责理解上下文，并基于概率预测生成下一个字。在RAG中，它负责阅读检索到的片段，并组织语言回答用户问题。

输入/输出： 输入提示词(Prompt)->输出自然语言。

**Spring AI 的策略**
    
使用EmbeddingClient调用BGE模型进行数据的向量化存储与查询；使用ChatClient调用Qwen进行最终答案的生成。

## 二、向量数据库的核心参数解析

当我们将数据存入向量数据库(如 PostgreSQL 的 pgvector 插件、Milvus、Chroma)时，必须理解以下关键参数，这些参数直接决定了检索的精度与性能：

**Dimensions (维度)**
    
这是向量数组的长度。例如，OpenAI的text-embedding-3-small是1536维，而某些轻量级BGE模型可能是768维。Spring AI 配置的 vector-store 维度必须与你选用的Embedding模型严格一致，否则会抛出维度不匹配错误。

**Index Type (索引类型)**
    
为了在百万级数据中毫秒级找到相似向量，不能暴力扫描，需要索引，最常用的有HNSW和IVF两种。

HNSW (Hierarchical Navigable Small World)：图索引。性能最强，适合内存充足、对延迟要求极高的场景。

IVF (Inverted File)： 倒排索引。将向量空间划分为多个聚类，检索时只搜索最近的几个聚类。适合数据量极大、显存/内存受限的场景。

**Distance Type (距离/度量方式)**

决定了如何计算两个向量像不像，常见计算方式包括Cosine、Euclidean、Inner Product三种。

Cosine (余弦相似度)： 衡量两个向量夹角的余弦值。最常用，侧重于语义方向的一致性，而非文本长度。

Euclidean / L2 (欧几里得距离)： 空间中两点的直线距离。

Inner Product (内积)： 适用于归一化后的向量。

## 三、检索逻辑：SQL 视角下的 Top-K 与 Threshold

在Spring AI的VectorStore 接口背后，实际发生的是一次基于数学运算的数据库查询。

假设我们使用 PostgreSQL (pgvector)，当用户发起查询时，涉及以下核心参数：

**Top-K**：截取最相似的前K个片段。

**Similarity Threshold (相似度阈值)**： 这是一个 0 到 1 之间的数值(通常)。只有相似度高于该值的片段才会被召回，用于过滤低相关性的噪声，通常Similarity与相似度成正比,其计算公式如下,具体实现视数据库算子定义而定：                              

```javascript
   Similarity = 1 - Distance
```




**SQL示例**：

Spring AI 会将自然语言问题转化为向量 \[0.1, ...\]，然后执行类似如下的 SQL：

```sql
-- 假设 embedding 列存储向量数据
-- <=> 操作符代表余弦距离计算
SELECT content, (1 - (embedding <=> '[0.1, 0.2, ... 0.9]')) as similarity FROM enterprise_knowledge_base 
  WHERE (1 - (embedding <=> '[0.1, 0.2, ... 0.9]')) > 0.75  -- 对应 similarityThreshold
  ORDER BY embedding <=> '[0.1, 0.2, ... 0.9]' ASC          -- 按距离排序
  LIMIT 5; 
```

## 四、Spring AI 完整 RAG 流程实现

理解了底层逻辑后，在 Spring AI 中实现 RAG 就变得顺理成章。代码逻辑分为三步：

**向量化**：将用户问题通过 Embedding Model 转为向量。

**检索**：拿着向量去数据库查 Top-K。

**生成**：将检索到的文本(Context)注入到Prompt中，发送给LLM。

![图片](/images/springai3-1.png)

## 五、痛点与思考：用户“说方言”怎么办

至此，我们已经搭建了一个标准的 RAG 系统。但在实际生产中，我们面临一个严峻的挑战：Garbage In, Garbage Out。

如果用户问得专业：“请列出2025考勤制度中关于年假的规定。”->检索精准。如果用户问得随性：“请假怎么弄？我想歇两天。”->语义模糊，可能检索到“年假”、“事假”甚至“调休”等一堆无关数据。如果检索的相关度无法得到物理保证，那么检索出的上下文是错误的，LLM就会一本正经地胡说八道。

向量检索高度依赖“语义相似性”，如何解决这个问题？

这就需要我们跳出简单的“检索-生成”线性思维，引入**智能体工作流 (Agentic Workflow)**的概念。

我们需要构建一个输入规整化的预处理环节：先调用一次 LLM 提取关键意图(“提取关键字”)，再用规整后的关键字去查向量库，拿到结果后再次调用LLM进行自校验(Self-Correction)。

这种“LLM->Vector->LLM”的链式思考与自我修正机制，正是我们下一篇要深入探讨的主题：从RAG走向Agent —— 神经符号AI与反思迭代的工程实践。
