# AI 时代 Go 后端：向量数据库与 RAG 应用

> 考察频率：★★★☆☆  优先级：P1（AI 加分项）
> 关键词：向量数据库、RAG、Embedding、Milvus、Pinecone、pgvector、Go 接入、AI 应用

---

## 面试官考察意图

考察候选人对 AI + Go 实际落地场景的理解。
AI 时代后端工程师不仅要会 CRUD，还要能接入 Embedding 服务、存储向量、构建 RAG 链路。高级工程师要能讲清楚**向量数据库选型（专用 vs  pgvector）、RAG 三件套（Embedding/VectorDB/Recall）的工作原理、Go 中的集成方式**，以及实际项目中的踩坑经验（如 Embedding 延迟、向量检索精度调优）。

---

## 核心答案（30 秒版）

RAG（检索增强生成）是 LLM 落地的主流架构，Go 后端的角色是**向量数据库管理员 + RAG 链路编排者**：

```
文档 → Embedding 服务 → 向量存储 → 检索 → LLM 上下文 → 回复
```

向量数据库分两类：
- **专用**：Milvus、Qdrant、Pinecone（高性能，亿级向量）
- **集成型**：pgvector（PostgreSQL 内置，100万向量内够用）

Go 后端不跑 LLM（调用 OpenAI/Claude），而是负责**Embedding + 存储 + 检索 + 结果后处理**。

---

## 深度展开

### 1. RAG 架构与 Go 后端职责

```
┌──────────┐    ┌──────────────┐    ┌────────────┐    ┌──────────┐    ┌──────────┐
│  文档    │ → │ Go 后端       │ → │ Embedding   │ → │ 向量数据库│ → │   LLM    │
│ (PDF/HTML)│    │ (chunk/parse)│    │  服务       │    │          │    │ (GPT-4)  │
└──────────┘    └──────────────┘    └────────────┘    └──────────┘    └──────────┘
                      │                                                         ↑
                      │              ┌──────────────┐                          │
                      └─────────────→│ 上下文拼接   │──────────────────────────┘
                                     └──────────────┘
```

**Go 后端的具体工作：**

1. **文档解析 + 分块（Chunking）**：PDF 解析、HTML 清洗、按固定长度/语义分块
2. **Embedding 调用**：调用 OpenAI Embedding API（text-embedding-3-small）或本地模型
3. **向量存储**：写入 Milvus/pgvector
4. **检索编排**：根据用户 query 检索最相关 chunks
5. **上下文拼接**：将 chunks 组装成 prompt，返回给 LLM

### 2. 向量数据库对比与选型

#### 2.1 专用向量数据库

| 数据库 | 适用规模 | Go 支持 | 特点 |
|--------|---------|---------|------|
| **Milvus** | 亿级向量 | ✅ milvus-sdk-go | Apache 协议，成熟度高，国产云支持好 |
| **Qdrant** | 千万级 | ✅ qdrant-go | Rust 实现，延迟最低，支持过滤检索 |
| **Pinecone** | 亿级 | ✅ pinecone-client | 云服务，按量付费，无需运维 |
| **Weaviate** | 千万级 | ✅ weaviate-go | 支持混合搜索（向量+关键词） |

#### 2.2 PostgreSQL pgvector（推荐轻量场景）

```sql
-- 开启 pgvector 扩展
CREATE EXTENSION IF NOT EXISTS vector;

-- 创建向量表（1536 维 = OpenAI embedding 维度）
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT NOT NULL,
    metadata JSONB,
    embedding vector(1536)  -- 存储 OpenAI text-embedding-3-small
);

-- 创建 HNSW 索引（高召回，低延迟）
CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 200);

-- 近似最近邻检索
SELECT content, 1 - (embedding <=> $1) AS similarity
FROM documents
WHERE metadata->>'category' = 'tech'
ORDER BY embedding <=> $1
LIMIT 5;
```

#### 2.3 Go 接入 pgvector

```go
package main

import (
    "context"
    "fmt"
    "strings"

    "github.com/pgvector/pgvector-go"
    "github.com/jackc/pgx/v5"
    "github.com/jackc/pgx/v5/pgxpool"
)

// Document 向量记录
type DocEmbedding struct {
    ID        int64
    Content   string
    Metadata  map[string]any
    Embedding pgvector.Vector
}

// 插入向量（需要先调用 Embedding 服务）
func insertEmbedding(ctx context.Context, pool *pgxpool.Pool, content string, embedding []float32) error {
    _, err := pool.Exec(ctx, `
        INSERT INTO documents (content, embedding)
        VALUES ($1, $2)
    `, content, pgvector.NewVector(embedding))
    return err
}

// 检索相似文档
func searchSimilar(ctx context.Context, pool *pgxpool.Pool, queryEmbedding []float32, topK int) ([]DocEmbedding, error) {
    rows, err := pool.Query(ctx, `
        SELECT id, content, metadata, embedding
        FROM documents
        ORDER BY embedding <=> $1
        LIMIT $2
    `, pgvector.NewVector(queryEmbedding), topK)
    if err != nil {
        return nil, fmt.Errorf("vector search: %w", err)
    }
    defer rows.Close()

    var docs []DocEmbedding
    for rows.Next() {
        var d DocEmbedding
        if err := rows.Scan(&d.ID, &d.Content, &d.Metadata, &d.Embedding); err != nil {
            return nil, err
        }
        docs = append(docs, d)
    }
    return docs, rows.Err()
}
```

#### 2.4 Go 接入 Milvus

```go
package main

import (
    "context"
    "fmt"

    "github.com/milvus-io/milvus-sdk-go/v2/client"
    "github.com/milvus-io/milvus-sdk-go/v2/entity"
)

// Milvus 配置
const (
    milvusAddr = "localhost:19530"
    collection = "my_docs"
    dim        = 1536 // OpenAI embedding dimension
)

func searchMilvus(ctx context.Context, queryEmbedding []float32) ([]string, error) {
    cli, err := client.NewClient(ctx, client.Config{Address: milvusAddr})
    if err != nil {
        return nil, err
    }
    defer cli.Close()

    // 加载 collection
    coll, err := cli.GetCollectionByName(ctx, collection)
    if err != nil {
        return nil, fmt.Errorf("get collection: %w", err)
    }

    // 搜索向量
    searchReq := entity.NewSearchRequest(collection,
        entity.NewFloatVector(queryEmbedding),
        "embedding",
        entity.L2,
        5, // topK
    )
    results, err := cli.Search(ctx, client.NewSearchOption(coll.Name(), 5, []string{"content"}).WithConsistencyLevel(entity.CL_ReadMySession))
    if err != nil {
        return nil, fmt.Errorf("search: %w", err)
    }

    var contents []string
    for _, result := range results.GetRow().([]entity.Row) {
        if content, ok := result["content"]; ok {
            contents = append(contents, content.(string))
        }
    }
    return contents, nil
}
```

### 3. Embedding 服务接入

#### 3.1 OpenAI Embedding（最常用）

```go
package embedding

import (
    "context"
    "encoding/json"
    "fmt"
    "net/http"
    "time"
)

// OpenAIEmbeddingResponse OpenAI 响应结构
type OpenAIEmbeddingResponse struct {
    Data []struct {
        Embedding []float32 `json:"embedding"`
    } `json:"data"`
}

// GetEmbedding 调用 OpenAI API 获取向量
func GetEmbedding(ctx context.Context, text string) ([]float32, error) {
    payload := map[string]any{
        "model": "text-embedding-3-small", // 或 text-embedding-3-large (1536/3072 维)
        "input": text,
    }
    body, _ := json.Marshal(payload)

    req, _ := http.NewRequestWithContext(ctx, "POST",
        "https://api.openai.com/v1/embeddings",
        strings.NewReader(string(body)))
    req.Header.Set("Authorization", "Bearer "+openaiAPIKey)
    req.Header.Set("Content-Type", "application/json")

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, fmt.Errorf("openai request: %w", err)
    }
    defer resp.Body.Close()

    var result OpenAIEmbeddingResponse
    if err := json.NewDecoder(resp.Body).Decode(&result); err != nil {
        return nil, fmt.Errorf("decode response: %w", err)
    }
    return result.Data[0].Embedding, nil
}
```

#### 3.2 本地 Embedding 模型（Nomic/often）

对于数据隐私敏感场景，使用本地模型（Ollama + nomic-embed-text）：

```go
// 使用 Ollama 本地模型（无需 API key，数据不出境）
func GetEmbeddingLocal(ctx context.Context, text string) ([]float32, error) {
    payload := map[string]string{
        "model": "nomic-embed-text",
        "prompt": text,
    }
    body, _ := json.Marshal(payload)

    req, _ := http.NewRequestWithContext(ctx, "POST",
        "http://localhost:11434/api/embeddings",  // Ollama 默认端口
        strings.NewReader(string(body)))
    req.Header.Set("Content-Type", "application/json")

    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()

    var result struct {
        Embedding []float32 `json:"embedding"`
    }
    json.NewDecoder(resp.Body).Decode(&result)
    return result.Embedding, nil
}
```

### 4. 文档分块策略（Chunking）

分块策略直接影响 RAG 召回质量：

```go
// 固定长度分块（简单但可能切断语义）
func chunkByFixedSize(content string, chunkSize int, overlap int) []string {
    var chunks []string
    runes := []rune(content)
    start := 0

    for start < len(runes) {
        end := start + chunkSize
        if end > len(runes) {
            end = len(runes)
        }
        chunks = append(chunks, string(runes[start:end]))

        // overlap 防止语义截断
        start = end - overlap
    }
    return chunks
}

// 语义分块（按段落/句子，适合结构化文档）
func chunkByParagraph(content string) []string {
    paragraphs := strings.Split(content, "\n\n")
    var chunks []string
    current := ""

    for _, p := range paragraphs {
        if len(current)+len(p) > 1000 { // 动态阈值
            chunks = append(chunks, strings.TrimSpace(current))
            current = p
        } else {
            current += "\n" + p
        }
    }
    if current != "" {
        chunks = append(chunks, strings.TrimSpace(current))
    }
    return chunks
}
```

**分块策略对比：**

| 策略 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| 固定长度 | 简单可控 | 可能切断语义 | 代码类、格式化文本 |
| 段落分块 | 保留语义 | 大段落需要合并 | 技术文档、报告 |
| 语义分块 | 质量最高 | 实现复杂 | 问答系统、聊天机器人 |

### 5. 完整 RAG 链路实现

```go
package rag

import (
    "context"
    "fmt"
    "strings"
)

type RAGService struct {
    embedder Embedder
    store    VectorStore
    llm      LLMClient
}

// Embedder 接口
type Embedder interface {
    Embed(ctx context.Context, text string) ([]float32, error)
    EmbedBatch(ctx context.Context, texts []string) ([][]float32, error)
}

// VectorStore 接口
type VectorStore interface {
    Insert(ctx context.Context, docs []Document) error
    Search(ctx context.Context, query []float32, topK int) ([]Document, error)
}

// Document 向量文档
type Document struct {
    ID       string
    Content  string
    Metadata map[string]any
}

// 索引文档（离线）
func (r *RAGService) IndexDocuments(ctx context.Context, docs []Document) error {
    // 1. 分块
    var chunks []Document
    for _, doc := range docs {
        subChunks := chunkByParagraph(doc.Content)
        for i, chunk := range subChunks {
            chunks = append(chunks, Document{
                ID:       fmt.Sprintf("%s-%d", doc.ID, i),
                Content:  chunk,
                Metadata: doc.Metadata,
            })
        }
    }

    // 2. Embedding
    contents := make([]string, len(chunks))
    for i, c := range chunks {
        contents[i] = c.Content
    }
    embeddings, err := r.embedder.EmbedBatch(ctx, contents)
    if err != nil {
        return fmt.Errorf("embedding: %w", err)
    }

    // 3. 存储
    for i := range chunks {
        chunks[i].Vector = embeddings[i]
    }
    return r.store.Insert(ctx, chunks)
}

// 检索 + 生成（在线）
func (r *RAGService) Query(ctx context.Context, question string) (string, error) {
    // 1. 查询向量
    queryEmbedding, err := r.embedder.Embed(ctx, question)
    if err != nil {
        return "", fmt.Errorf("query embedding: %w", err)
    }

    // 2. 检索 top-K 相关 chunks
    docs, err := r.store.Search(ctx, queryEmbedding, 5)
    if err != nil {
        return "", fmt.Errorf("vector search: %w", err)
    }

    // 3. 拼接上下文
    var contextBuilder strings.Builder
    for _, doc := range docs {
        contextBuilder.WriteString(doc.Content)
        contextBuilder.WriteString("\n\n")
    }
    contextStr := contextBuilder.String()

    // 4. 组装 prompt 给 LLM
    prompt := fmt.Sprintf(`
根据以下上下文回答问题。如果上下文中没有相关信息，说明不知道。

上下文：
%s

问题：%s
`, contextStr, question)

    // 5. 调用 LLM
    answer, err := r.llm.Complete(ctx, prompt)
    if err != nil {
        return "", fmt.Errorf("llm complete: %w", err)
    }
    return answer, nil
}
```

### 6. 召回质量调优

```go
// 1. 混合检索：向量 + 关键词（提高召回）
func hybridSearch(ctx context.Context, store VectorStore, queryEmbedding []float32, queryText string) []Document {
    // 向量检索
    vectorResults := store.Search(ctx, queryEmbedding, 10)

    // BM25 关键词检索（可以用 elasticsearch 或 pg 内置）
    keywordResults := bm25Search(queryText, 10)

    // RRF 融合（Reciprocal Rank Fusion）
    return rrfFusion(vectorResults, keywordResults, k=60)
}

// 2. 重排序（ReRank）：用更精准的模型排序 top-K
type Reranker interface {
    Rerank(query string, documents []Document) []Document
}

// 3. 元数据过滤：减少无关召回
func searchWithFilter(ctx context.Context, store VectorStore, embedding []float32, category string) []Document {
    docs, _ := store.Search(ctx, embedding, 20)
    var filtered []Document
    for _, d := range docs {
        if d.Metadata["category"] == category {
            filtered = append(filtered, d)
        }
    }
    return filtered
}
```

### 7. 生产踩坑经验

#### 坑 1：Embedding 延迟影响索引速度

```go
// 问题：逐条 embedding 太慢（1000 条文档 × 1536 维 × API 延迟）
// 解决：并发 embedding + 批量 API

func embedBatchFast(ctx context.Context, embedder Embedder, texts []string, parallelism int) ([][]float32, error) {
    sem := make(chan struct{}, parallelism)
    var wg sync.WaitGroup
    results := make([][]float32, len(texts))
    errCh := make(chan error, 1)

    for i, text := range texts {
        sem <- struct{}{}
        wg.Add(1)
        go func(idx int, t string) {
            defer wg.Done()
            vec, err := embedder.Embed(ctx, t)
            if err != nil {
                select {
                case errCh <- err:
                default:
                }
                return
            }
            results[idx] = vec
            <-sem
        }(i, text)
    }
    wg.Wait()
    return results, nil
}
```

#### 坑 2：向量维度不匹配

```go
// 问题：OpenAI text-embedding-3-small = 1536 维，但数据库建了 1536 却没考虑模型差异
// 解决：统一 embedding 模型，在写入前校验维度

func validateDimension(embedding []float32, expectedDim int) error {
    if len(embedding) != expectedDim {
        return fmt.Errorf("embedding dimension mismatch: got %d, expected %d",
            len(embedding), expectedDim)
    }
    return nil
}
```

#### 坑 3：pgvector HNSW 索引创建慢

```go
-- 生产环境：m=16, ef_construction=200 是精度/速度的平衡点
-- 不要用默认值 m=2（精度差）或 m=64（太慢）

CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 200);

-- 查询时指定 ef_search（越大越精准，越慢）
SET local hnsw.ef_search = 100;  -- 比默认值 40 更精准
SELECT * FROM documents ORDER BY embedding <=> $1 LIMIT 5;
```

---

## 高频追问

**Q：向量数据库和普通数据库的本质区别是什么？**

> 普通数据库存储精确值（如 id=123），查询是等值或范围查询。向量数据库存储高维浮点向量（如 1536 维），查询是**近似最近邻**（ANN），衡量的是语义相似度而非精确匹配。这是本质区别，决定了索引结构和查询算法完全不同（倒排索引 vs HNSW/IVF）。

**Q：什么时候用 pgvector 而不是 Milvus？**

> 数据量 < 1000 万条、团队有 PostgreSQL 运维能力、不需要分布式向量检索时，用 pgvector 足够。Milvus 适合**亿级向量**、需要**混合检索**（向量+标量过滤）、对**可用性要求高**的生产环境。

**Q：Go 能不能跑 LLM 推理？**

> 能，但 Go 不适合做 LLM 推理。LLM 推理是 GPU 密集计算，Go 的优势在于并发调度，而 LLM 需要的是矩阵乘法优化。目前主流方案是 Go 作为调度层，LLM 推理层用 Python/C++（如 vLLM、llama.cpp）通过 gRPC/API 调用。

**Q：RAG 的三个核心环节中，哪个最容易成为瓶颈？**

> Embedding 延迟（批量索引时）和向量检索（亿级数据时）是常见瓶颈。对于 10 万条文档级规模，pgvector + HNSW 足够；如果需要秒级索引万条文档，用 Qdrant（Rust 实现延迟最低）。

**Q：Go 接入 AI 有哪些成熟的 SDK？**

> - **OpenAI Go**：`github.com/sashabaranov/go-openai`（最完整）
> - **LangChain 适配**：`github.com/tmc/langchaingo`（Go 版 LangChain，RAG/Agent 支持）
> - **Ollama**：`github.com/ollama/ollama-go`

---

## 延伸阅读

- [pgvector GitHub](https://github.com/pgvector/pgvector) - PostgreSQL 向量扩展
- [Milvus SDK Go](https://github.com/milvus-io/milvus-sdk-go) - 官方 Go SDK
- [Qdrant Go Client](https://github.com/qdrant/qdrant-go) - 官方 Go SDK
- [sashabaranov/go-openai](https://github.com/sashabaranov/go-openai) - OpenAI Go 客户端
- [LangChain Go](https://github.com/tmc/langchaingo) - Go 版 LangChain
- [RAG 技术白皮书](https://arxiv.org/abs/2312.10997) - retrieval-augmented generation 原始论文