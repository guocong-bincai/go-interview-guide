# 搜索系统设计

> 考察频率：★★★☆☆  难度：★★★★☆

## 搜索系统架构

```
用户请求
    ↓
Query 解析（分词 / 纠错 / 改写）
    ↓
Query 理解（意图识别 / 实体识别 / 搜索建议）
    ↓
检索引擎（倒排索引 / BM25 排序）
    ↓
排序层（相关性 / 质量分 / 商业权重）
    ↓
返回结果
```

---

## 倒排索引

### 原理

```
正排索引：文档ID → 文档内容
  doc1 → "Go 语言 高并发"
  doc2 → "Go 语言 性能"

倒排索引：词 → 包含该词的文档列表
  Go    → [doc1, doc2]
  语言  → [doc1, doc2]
  高并发 → [doc1]
  性能  → [doc2]
```

### 倒排表结构

```go
// 倒排表
type InvertedIndex struct {
    mu    sync.RWMutex
    terms map[string][]Posting // 词 → 文档列表
}

//  Posting：文档中该词的位置信息
type Posting struct {
    DocID    uint64
    Positions []int // 词在文档中出现的位置
    TF       float64 // Term Frequency
}

// 添加文档到索引
func (idx *InvertedIndex) AddDoc(docID uint64, text string) {
    tokens := tokenize(text) // 分词
    tf := make(map[string]float64)

    for pos, token := range tokens {
        idx.mu.Lock()
        idx.terms[token] = append(idx.terms[token], Posting{
            DocID:    docID,
            Positions: []int{pos},
            TF:       1.0,
        })
        idx.mu.Unlock()
        tf[token]++
    }

    // 归一化 TF
    for token, count := range tf {
        tf[token] /= float64(len(tokens))
    }
}

// 搜索：返回包含所有查询词的文档
func (idx *InvertedIndex) Search(query string) []uint64 {
    tokens := tokenize(query)
    var result []uint64
    docCount := make(map[uint64]int)

    for _, token := range tokens {
        postings := idx.terms[token]
        for _, p := range postings {
            docCount[p.DocID]++
        }
    }

    for docID, count := range docCount {
        if count == len(tokens) { // 必须包含所有词
            result = append(result, docID)
        }
    }
    return result
}
```

---

## 分词

### 中文分词方案

```go
// 1. 正向最大匹配（MM）
func mmSegment(text string, dict map[string]bool) []string {
    var result []string
    i := 0
    for i < len(text) {
        matched := false
        for j := min(i+maxWordLen, len(text)); j > i; j-- {
            word := text[i:j]
            if dict[word] {
                result = append(result, word)
                i = j
                matched = true
                break
            }
        }
        if !matched {
            result = append(result, text[i:i+1])
            i++
        }
    }
    return result
}

// 2. 双向最大匹配
// 同时用正向和反向最大匹配，取词数更少或单字更少的结果

// 3. 基于 HMM 的分词（Viterbi 算法）
// 处理未登录词（人名、地名等）
```

### 分词器选型

| 分词器 | 特点 |
|--------|------|
| jieba | 简单高效，支持自定义词典 |
| go-jieba | Go 版本 |
| sego | 纯 Go 实现 |
| analyzer (bluge) | Go 全文搜索库 |

---

## BM25 排序算法

BM25 是搜索引擎常用的相关性评分算法：

```go
import "math"

// BM25 评分
func bm25Score(docLen int, avgLen float64, tf float64, idf float64, k1, b float64) float64 {
    // TF 部分：饱和函数，防止 TF 过高时评分继续线性增长
    tfComponent := (tf * (k1 + 1)) / (tf + k1*(1 - b + b*float64(docLen)/avgLen))
    return idf * tfComponent
}

// IDF（逆文档频率）
func idf(df, N int) float64 {
    return math.Log((float64(N) - float64(df) + 0.5) / (float64(df) + 0.5))
}

// 完整 BM25 搜索
func bm25Search(query string, docs map[uint64]string, idx *InvertedIndex) []Result {
    tokens := tokenize(query)
    N := len(docs)
    avgLen := calculateAverageDocLen(docs)

    scores := make(map[uint64]float64)
    docLens := make(map[uint64]int)

    for _, token := range tokens {
        postings := idx.terms[token]
        df := len(postings)
        idf := idf(df, N)

        for _, p := range postings {
            docLens[p.DocID] += len(tokens) // 估算文档长度
            scores[p.DocID] += bm25Score(docLens[p.DocID], avgLen, p.TF, idf, 1.2, 0.75)
        }
    }

    // 按分数排序
    var results []Result
    for docID, score := range scores {
        results = append(results, Result{DocID: docID, Score: score})
    }
    sort.Slice(results, func(i, j int) bool {
        return results[i].Score > results[j].Score
    })
    return results
}
```

---

## 搜索建议（AutoComplete）

### Trie 树实现

```go
type TrieNode struct {
    children map[rune]*TrieNode
    isEnd    bool
    freq     int // 词频，用于排序
}

type Trie struct {
    root *TrieNode
}

func (t *Trie) Insert(word string, freq int) {
    node := t.root
    for _, ch := range word {
        if node.children[ch] == nil {
            node.children[ch] = &TrieNode{children: make(map[rune]*TrieNode)}
        }
        node = node.children[ch]
    }
    node.isEnd = true
    node.freq = freq
}

func (t *Trie) SearchPrefix(prefix string) []string {
    node := t.root
    for _, ch := range prefix {
        if node.children[ch] == nil {
            return nil
        }
        node = node.children[ch]
    }
    // BFS 收集所有以 prefix 开头的词
    var results []string
    t.collect(node, prefix, &results)
    sort.Slice(results, func(i, j int) bool {
        return results[i] > results[j] // 按 freq 排序
    })
    return results[:10]
}
```

---

## Elasticsearch vs 自己实现

| 维度 | 自研 | Elasticsearch |
|------|------|--------------|
| 适用规模 | 亿级以下 | 任意规模 |
| 分布式 | 需自己实现 | 开箱即用 |
| 功能 | 简单场景 | 复杂查询、聚合 |
| 维护成本 | 低 | 高 |

### Go + Elasticsearch 集成

```go
import "github.com/elastic/go-elasticsearch/v8"

func search(es *elasticsearch.Client, query string) ([]string, error) {
    res, err := es.Search(
        es.Search.WithIndex("posts"),
        es.Search.WithBody(strings.NewReader(`{
            "query": {
                "bool": {
                    "must": [
                        {"match": {"content": "` + query + `"}}
                    ]
                }
            },
            "highlight": {
                "fields": {"content": {}}
            }
        }`)),
    )
    if err != nil {
        return nil, err
    }
    defer res.Body.Close()

    var result map[string]interface{}
    json.NewDecoder(res.Body).Decode(&result)
    // 解析返回的 doc ID
    ...
}
```

---

## 总结

| 模块 | 技术方案 |
|------|---------|
| 倒排索引 | 内存 Trie（小型）或 ES（大型） |
| 分词 | jieba / go-seg |
| 排序 | BM25 + 质量分 |
| 搜索建议 | Trie 树 + 热度排序 |

### 面试话术

**Q：倒排索引和正排索引的区别？**
> 正排索引是"文档 → 词"，知道文档后找词；倒排索引是"词 → 文档"，知道词后找文档。搜索场景用倒排，因为先有关键词，再找相关文档。

**Q：BM25 解决了什么问题？**
> 解决了词频饱和问题。简单 TF*IDF 会因为词频越高评分越高，导致长文档永远排名靠前。BM25 通过引入饱和函数和文档长度归一化，让评分更合理。
