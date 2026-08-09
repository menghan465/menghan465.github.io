+++
date = '2026-08-09T20:30:00+08:00'
draft = false
title = '宿优圈 RAG 管线：从旅游 PDF 与社区文章到 pgvector 检索和流式回答'
description = '记录宿优圈如何把 LvBanGPT 的旅游资料作为离线导入源，经过文本清洗、分块、Embedding、pgvector 检索和 AI SDK 流式生成，形成一条可追踪但仍有明确边界的 RAG 管线。'
tags = ['RAG', 'pgvector', 'PostgreSQL', 'AI SDK', 'Next.js', 'Embedding']
categories = ['AI 应用', '全栈开发']
slug = 'showyou-circle-rag-pgvector-pipeline'
+++

“接入一个大模型”不等于实现了 RAG。真正需要解释的是：资料从哪里来，如何被清洗和切分，向量是否与数据库维度一致，检索结果如何进入提示词，以及模型不可用时系统是否能给出可理解的失败状态。

宿优圈的旅游助手没有继续嵌套 Gradio 页面，也没有在 Web 请求中启动一个常驻 Python 服务。当前实现选择了一条相对简单、容易审计的链路：Next.js 负责接口和流式响应，AI SDK 负责 Embedding 与文本生成，PostgreSQL 的 pgvector 负责存储和近似检索。`E:\LvBanGPT\dataset` 中的 PDF 只是一次性知识来源，不是运行时依赖。

这篇文章只描述当前代码已经实现的内容，不把它包装成“生产级 RAG”。真实检索质量、召回率和线上延迟还需要独立的评估和压测。

## 先划清 LvBanGPT 的边界

旧项目里的 LvBanGPT 包含 PDF 数据集、Flask/Gradio 入口和 Python 算法代码。重构后没有把整个目录复制到 Next.js 进程里，而是把它拆成两种性质不同的能力：

```text
LvBanGPT/dataset/*.pdf
        │  离线导入时使用 PyMuPDF 提取文本
        ▼
knowledge_documents
        │  清洗、分块、Embedding
        ▼
knowledge_chunks.embedding vector(1536)
```

导入命令是显式的：

```bash
npm run knowledge:import -- "E:\LvBanGPT\dataset"
npm run knowledge:index
```

这里仍然会在导入阶段通过 Node 的 `execFile` 调用 `scripts/extract-pdf.py`，由 PyMuPDF 提取 PDF 文本。但它是短生命周期的离线工具调用；Next.js 启动和 `/api/chat` 请求不会启动 `LvBanGPT/app.py`、Gradio 或常驻 Python 子进程。这个区别很重要：系统不是“完全不使用 Python”，而是把 Python 限制在明确的批处理边界内。

## 导入阶段：先做可重复的数据准备

`importKnowledgeDocuments` 对每个 PDF 做四件事：提取文本、规范化内容、计算 SHA-256、写入知识库元数据。

SHA-256 不是为了安全认证，而是为了判断“这份文件是否已经导入过”。`sourceHash` 在数据库中建立唯一约束，已处于 `INDEXED` 状态的相同文件默认跳过；使用 `--force` 才会重新放回 `PENDING` 状态。这样可以重复执行导入命令，而不因为每次启动都产生重复文档。

导入后的状态流转如下：

```text
PDF 文件
  ├─ 文本提取失败       → 本次记录失败
  ├─ 文本少于 200 字     → 本次导入失败 / PDF_TEXT_TOO_SHORT
  └─ 导入成功             → PENDING
                              │
                              ├─ 向量化成功 → INDEXED
                              └─ 向量化失败 → FAILED + lastError
```

文本提取失败或少于 200 字时，当前导入器只记录本次失败；只有文档进入索引阶段后，向量化异常才会写入 `FAILED` 和 `lastError`。

文本规范化目前是有意保持保守的：删除空字符、零宽字符和分页控制符，统一换行，压缩连续空行和明显的空白噪声。它没有尝试对中文进行复杂语义重写，也没有自动“修复” PDF 中可能存在的错别字。对于旅游资料来说，宁可保留原文，也不应该在导入阶段无依据地改写价格、日期或路线。

文件名目前用于推导标题和城市，`旅游攻略`、`攻略` 等后缀会被移除；它不是可靠的地理信息抽取器。如果 PDF 文件名不规范，知识库中的城市字段也可能只是一个不完整的标题，这属于数据质量问题，不应被提示词掩盖。

## 分块：简单，但明确承认是近似方案

分块实现位于 `src/lib/ai/chunking.ts`。默认参数是：

```ts
const maxChars = 1400;
const overlapChars = 180;
```

处理顺序是先按双换行拆分段落。如果某一段仍然超过上限，再使用带重叠的滑动窗口切分。重叠的目的不是提高模型智力，而是减少一句话刚好被切在两个 chunk 边界时的语义损失。

可以把它概括为：

```text
原始文本
  → 按段落拆分
  → 超长段落使用窗口切分
  → 每个 chunk 保存 position
  → 生成 tokenCount 和 embedding
```

这里有一个必须主动说明的限制：当前 `tokenCount` 是基于字符长度的近似值，类似 `ceil(content.length / 2)`，不是真实 tokenizer 的结果。它可以用于粗略观察 chunk 大小，但不能用来严格控制上下文 token 预算。真正上线前，如果需要精确计费、上下文预算或不同模型之间的兼容性，应替换为目标模型对应的 tokenizer，并重新评估中文文本的切分长度。

## Embedding：维度是数据库契约，不是普通配置

当前默认 Embedding 模型是 `text-embedding-3-small`，向量维度固定为 1536。模型创建集中在 `src/lib/ai/embeddings.ts`，索引器使用 `embedMany` 批量生成向量，并限制：

- 并发调用数最多为 4；
- 请求失败最多重试 1 次；
- 每个向量长度必须是 1536；
- 每个元素必须是有限数值，不能包含 `NaN` 或无穷大。

数据库迁移中对应的字段是：

```sql
"embedding" vector(1536)
```

并为文章 chunk 与知识库 chunk 分别建立 HNSW 索引：

```sql
CREATE INDEX "knowledge_chunks_embedding_hnsw_idx"
ON "knowledge_chunks"
USING hnsw ("embedding" vector_cosine_ops)
WHERE "embedding" IS NOT NULL;
```

维度校验和数据库类型校验是同一条契约的两端。以后如果把 Embedding 模型换成别的维度，不能只修改环境变量，必须同步调整向量字段、迁移脚本、校验逻辑和已有数据的重建流程，否则问题会从“模型调用失败”变成“索引数据不一致”。

文章和知识库的重建索引都遵循“先删旧 chunk，再写新 chunk”的事务边界。知识库索引失败时会把文档标记为 `FAILED` 并保存错误信息，因此后台可以区分“还没处理”和“处理失败”，而不是只看到一个空的搜索结果。

## 检索：把两类来源合并成一个候选集

用户问题进入检索前会先去除首尾空白，并截断到 1000 个字符。随后使用同一个 Embedding 模型生成查询向量，在 PostgreSQL 中同时搜索：

1. 状态为 `PUBLISHED` 且已经有向量的社区文章 chunk；
2. 状态为 `INDEXED` 且已经有向量的知识库 chunk。

两类结果通过 `UNION ALL` 合并，再按余弦距离升序排列：

```sql
(c."embedding" <=> CAST($1 AS vector))::double precision AS distance
```

`<=>` 返回的是余弦距离，数值越小代表向量空间中越接近。查询层把 limit 限制在 1 到 8 之间，聊天接口默认取 5 个 chunk；构造上下文时还设置 8000 字符上限，避免把大量检索内容无条件塞进模型上下文。

检索结果会带上来源类型、标题、文章 slug 或原始文件名，最终格式类似：

```text
[Source: knowledge document: 杭州 | 西湖攻略.pdf]
相关资料正文……
```

这让回答可以回指文章或知识库文件，也让调试时能够判断“模型答错”究竟是检索错、上下文组装错，还是生成阶段自行补全了不存在的信息。

不过当前检索还没有 reranker、混合关键词检索或相似度阈值。也就是说，它会返回 Top-K 最近邻，但没有保证这些结果对问题足够相关。对于资料规模较小的演示项目可以接受；如果资料量和问题复杂度上升，应增加关键词召回、重排序和最低相似度门槛，并用评估集验证，而不是只凭几次手工提问判断效果。

## Prompt 和流式接口：限制模型能说什么

`/api/chat` 是 Node.js runtime 的 Route Handler。它只保留最近 12 条消息，并把每条消息文本限制在 4000 个字符以内，然后执行：

```text
最近对话
  → 用户问题向量化
  → 检索 Top 5
  → 组装最多 8000 字符上下文
  → streamText
  → 流式返回浏览器
```

系统提示词要求模型：

- 与用户使用相同语言回答；
- 景点、价格、日期、路线等事实只能来自检索内容；
- 资料不足时明确说“当前知识库没有依据”；
- 通用建议必须明确标注为建议，而不能伪装成资料事实；
- 以 `[Source: ...]` 形式引用相关来源。

生成阶段的 `maxOutputTokens` 是 700，最多重试一次。没有配置 `OPENAI_API_KEY` 时接口返回 503；模型调用失败时返回 502，而不是返回一段看起来正常但实际没有依据的答案。这类错误码不解决模型幻觉，但能让前端、监控和部署检查区分“服务未配置”和“上游调用失败”。

## 当前验证结果和未完成部分

当前代码库已完成 5 个 Vitest 测试文件、23 个单元测试，以及 TypeScript 类型检查和 ESLint。它们主要验证分块、知识库文本规范化、文章和预订领域逻辑，并不等于真实 PostgreSQL 向量检索已经通过。

目前没有在本地完成以下验证：

- 真实 PostgreSQL + pgvector 迁移和 HNSW 查询；
- 真实 Embedding API 的批量成本、延迟和失败重试；
- RAG 的 Recall@K、NDCG 或人工标注集评估；
- 上线后的流式响应、限流和多用户并发。

因此，当前更准确的结论是：宿优圈已经有一条结构清楚、可重复导入、带状态和来源追踪的 RAG 实现，但还不能仅凭代码把它称为“效果经过生产验证的 RAG 系统”。下一步最有价值的工作不是继续堆叠 Agent 工具，而是建立一组真实旅游问题和标准答案，记录检索命中情况、引用完整性、延迟与失败率，再决定是否需要 reranker 或混合搜索。

## 小结

这条管线的核心价值不在于用了多少 AI 库，而在于把边界拆清楚了：

```text
LvBanGPT PDF 数据集
  → 离线提取与 SHA-256 幂等导入
  → PENDING / INDEXED / FAILED 状态
  → 段落分块与近似 token 统计
  → 1536 维 Embedding
  → pgvector HNSW 余弦检索
  → 带来源的上下文
  → AI SDK 流式回答
```

它比 iframe 嵌套一个现成聊天页面更容易测试和解释，但它的不足也同样明确：分块和 token 统计仍是启发式方案，检索没有重排序和评估集，数据库和外部模型的真实运行验证还需要补齐。对一个大学项目来说，这些边界比“支持 RAG”四个字更值得在面试中讲清楚。
