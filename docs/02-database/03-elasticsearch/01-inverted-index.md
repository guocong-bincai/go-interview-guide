# 倒排索引原理与搜索引擎实战

> 面试频率：★★★☆☆  考察角度：倒排索引原理、分词算法、相关性评分、Elasticsearch 原理

---

## 1. 正排索引 vs 倒排索引

### 1.1 正排索引（Forward Index）

```
文档ID → 文档内容
doc1 → "Go 语言高性能编程"
doc2 → "Go 并发编程实战"
doc3 → "高性能 Go 应用"
```

### 1.2 倒排索引（Inverted Index）

```
词 → 文档列表
Go        → [doc1, doc2, doc3]
语言      → [doc1]
高性能    → [doc1, doc3]
并发      → [doc2]
编程      → [doc1, doc2]
```

倒排索引是**搜索引擎的核心数据结构**，能够实现 O(1) 时间复杂度的关键词查找。

---

## 2. 倒排索引构建过程

### 2.1 分词（Tokenization）

```go
package analyzer

import (
    "github.com/blevesearch/segment"
    "regexp"
)

// 简单中英文分词器
type SimpleTokenizer struct{}

func (t *SimpleTokenizer) Tokenize(text string) []string {
    // 英文：按空格 + 正则提取单词
    enPattern := regexp.MustCompile(`[a-zA-Z]+`)
    enTokens := enPattern.FindAllString(text, -1)

    // 中文：简单按 runes 滑动窗口（2-gram，实际生产用 jieba）
    var cnTokens []string
    runes := []rune(text)
    for i := 0; i < len(runes)-1; i++ {
        cnTokens = append(cnTokens, string(runes[i:i+2]))
    }

    // 合并去重
    tokenSet := make(map[string]bool)
    for _, t := range enTokens {
        tokenSet[t] = true
    }
    for _, t := range cnTokens {
        tokenSet[t] = true
    }

    var result []string
    for t := range tokenSet {
        result = append(result, t)
    }
    return result
}
```

### 2.2 构建 Posting List

```go
type PostingList struct {
    DocumentIDs []int64   // 包含该词的文档 ID 列表
    Frequencies []int     // 每个文档中出现的频率
    Positions   [][]int   // 每个文档中的位置信息
}

// 倒排索引结构
type InvertedIndex struct {
    mu      sync.RWMutex
    index   map[string]*PostingList  // 词 → Posting List
    docs    map[int64]*Document      // docID → 文档内容
}

func (idx *InvertedIndex) AddDocument(docID int64, text string) {
    tokens := Tokenize(text)

    for pos, token := range tokens {
        pl := idx.index[token]
        if pl == nil {
            pl = &PostingList{}
            idx.index[token] = pl
        }

        pl.DocumentIDs = append(pl.DocumentIDs, docID)
        pl.Frequencies = append(pl.Frequencies, 1)
        pl.Positions = append(pl.Positions, []int{pos})
    }

    idx.docs[docID] = &Document{ID: docID, Content: text}
}

func (idx *InvertedIndex) Search(query string) []int64 {
    tokens := Tokenize(query)
    if len(tokens) == 0 {
        return nil
    }

    // 取第一个词的文档列表作为初始结果
    result := make(map[int64]int) // docID → 命中词数

    for _, token := range tokens {
        pl := idx.index[token]
        if pl == nil {
            continue
        }
        for _, docID := range pl.DocumentIDs {
            result[docID]++
        }
    }

    // 返回命中所有查询词的文档
    var docs []int64
    for docID, count := range result {
        if count == len(tokens) {
            docs = append(docs, docID)
        }
    }
    return docs
}
```

---

## 3. 相关性评分：TF-IDF 与 BM25

### 3.1 TF-IDF

```
Score = TF × IDF
TF = 词在文档中出现次数 / 文档总词数
IDF = log(总文档数 / 包含该词的文档数)
```

```go
func (idx *InvertedIndex) TFIDFScore(docID int64, token string) float64 {
    pl := idx.index[token]
    if pl == nil {
        return 0
    }

    // 找 token 在 doc 中的频率
    var tf float64
    for i, d := range pl.DocumentIDs {
        if d == docID {
            tf = float64(pl.Frequencies[i])
            break
        }
    }

    // IDF
    totalDocs := float64(len(idx.docs))
    docsWithToken := float64(len(pl.DocumentIDs))
    idf := math.Log((totalDocs + 1) / (docsWithToken + 1))

    return tf * idf
}
```

### 3.2 BM25（现代搜索引擎标准）

Elasticsearch 默认使用 BM25，解决了 TF-IDF 的饱和问题：

```
Score = IDF × (TF × (k1 + 1)) / (TF + k1 × (1 - b + b × |d|/avgdl))

k1 = 词频饱和参数（默认 1.2）
b = 文档长度归一化参数（默认 0.75）
|d| = 文档长度
avgdl = 平均文档长度
```

```go
func BM25Score(tf int, docLen, avgDL int, docsWithToken, totalDocs int, k1, b float64) float64 {
    idf := math.Log((float64(totalDocs)-float64(docsWithToken)+0.5)/(float64(docsWithToken)+0.5) + 1)

    tfNorm := (float64(tf) * (k1 + 1)) / (float64(tf) + k1*(1-b+b*float64(docLen)/float64(avgDL)))

    return idf * tfNorm
}
```

---

## 4. 倒排索引在 Go 项目中的实际应用

### 4.1 搜索建议（Autocomplete）

```go
// 前缀树 + 倒排索引结合
type SearchSuggestion struct {
    mu    sync.RWMutex
    // term → [docID...] 倒排索引
    postingList map[string][]int64
    // docID → doc
    docs        map[int64]*DocInfo
}

func (s *SearchSuggestion) Suggest(prefix string) []string {
    s.mu.RLock()
    defer s.mu.RUnlock()

    var results []string
    for term := range s.postingList {
        if strings.HasPrefix(term, prefix) {
            results = append(results, term)
        }
    }

    sort.Strings(results)
    if len(results) > 10 {
        results = results[:10]
    }
    return results
}
```

### 4.2 Elasticsearch 倒排索引原理（面试加分项）

```
Elasticsearch 使用 Lucene 倒排索引：
1. 写入时：分词 → 构建倒排列表（FST 压缩）
2. 查询时：O(1) 定位 term → 获取 posting list → 合并

FST（Finite State Transducer）：
- 将 term 列表编码为有限状态自动机
- 前缀共享存储，空间节省 50-80%
- 支持前缀查询和范围查询
```

---

## 5. 搜索系统的工程问题

### 5.1 冷启动问题

新文档没有足够的搜索/点击数据。

**解决方案**：内容相似度推荐（基于倒排索引找相似文档），用内容特征弥补行为特征缺失。

### 5.2 分词质量决定搜索效果

| 分词方式 | 优点 | 缺点 | 适用场景 |
|---------|------|------|---------|
| 2-gram | 简单，不需要词典 | 召回高、精确低 | 通用搜索 |
| 正向最大匹配 | 精确 | 召回低 | 固定领域 |
| jieba/hanLP | 效果好 | 依赖词典 | 中文通用 |

### 5.3 搜索延迟优化

```go
// 并行查询多个 term 的 posting list，最后合并
func (idx *InvertedIndex) ParallelSearch(query string, n int) ([]int64, error) {
    tokens := Tokenize(query)
    
    ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
    defer cancel()

    var mu sync.Mutex
    results := make([][]int64, len(tokens))

    var wg sync.WaitGroup
    for i, token := range tokens {
        wg.Add(1)
        go func(i int, token string) {
            defer wg.Done()
            docs := idx.SearchTerm(token)
            mu.Lock()
            results[i] = docs
            mu.Unlock()
        }(i, token)
    }
    wg.Wait()

    return MergePostingLists(results), nil
}
```

---

## 6. 面试高频追问

**Q：倒排索引和正排索引的区别？**
> 正排索引是文档→词，用于文档展示；倒排索引是词→文档，用于快速检索。搜索引擎先用倒排索引定位文档，再用正排索引获取文档摘要。

**Q：倒排索引的 posting list 如何存储？**
> 顺序存储文档 ID（整数压缩：Frame of Reference + Roaring Bitmap）；FST 压缩 term 前缀节省空间；位置信息单独存储用于短语查询。

**Q：搜索结果如何做相关性排序？**
> 早期用 TF-IDF，现在主流用 BM25（解决了词频饱和问题）；Elasticsearch/Lucene 使用 BM25 作为默认评分算法。

**Q：Go 中如何实现一个简单的搜索功能？**
> 用第三方库如 blevesearch（纯 Go 实现，支持倒排索引）；或者对接 Elasticsearch/OpenSearch。
