# 大文件上传/下载系统设计：分片/断点续传/秒传

> 考察频率：★★★★☆  优先级：P1  网盘/直播/视频类公司必考
> 关键词：分片上传、MD5秒传、断点续传、预签名URL、CDN回源

---

## 核心答案（30 秒版）

| 场景 | 方案 | 关键点 |
|------|------|--------|
| **大文件上传** | 客户端分片 → 服务端合并 | md5 去重 → 分片上传 → 并发 → 合并校验 |
| **断点续传** | 已完成的分片跳过 | uploadId + chunk index 状态记录 |
| **秒传** | MD5 指纹匹配已有文件 | 上传前先算 MD5，命中则直接完成 |
| **大文件下载** | Range Header + CDN | HTTP Range 支持断点下载，静态化走 CDN |

---

## 深度展开

### 1. 完整上传流程

```
用户上传文件（假设 2GB 视频文件）

Step 1: 初始化上传（获取 uploadId）
POST /upload/init
{
    "fileName": "video.mp4",
    "fileSize": 2147483648,
    "contentType": "video/mp4"
}
→ { "uploadId": "abc-123-xyz" }

Step 2: 计算文件 MD5（用于秒传检测）
client.md5("video.mp4") = "d41d8cd98f00b204e9800998ecf8427e"
→ 查询存储层是否有该 MD5 的文件？
   YES → 返回 URL（秒传成功 ✨）
   NO  → 继续分片上传

Step 3: 分片上传（客户端切 10MB 一个分片）
PUT /upload/chunk?uploadId=abc-123&partNumber=1
Content-MD5: ...
ETag: "chunk-1-hash"

PUT /upload/chunk?uploadId=abc-123&partNumber=2
...

Step 4: 全部上传完成后确认
POST /upload/complete?uploadId=abc-123
[{"PartNumber":1,"ETag":"chunk-1-hash"},{"PartNumber":2,"ETag":"chunk-2-hash"},...]
→ 服务端按 partNumber 排序，逐个追加到目标文件

Step 5: 验证与转码（异步）
触发 MD5 校验 → 触发 FFmpeg 转码 → 结果推送到 MQ
```

### 2. Go 实现分片上传服务

```go
package upload

import (
    "context"
    "crypto/md5"
    "encoding/hex"
    "fmt"
    "io"
    "net/http"
    "os"
    "path/filepath"
    "sync"
)

type UploadManager struct {
    tmpDir     string      // 临时目录存储分片
    storeDir   string      // 最终存储路径
    chunksPath map[string]string // uploadId -> 各分片的 ETag
    mu         sync.RWMutex
}

// InitUpload 初始化上传任务
func (um *UploadManager) InitUpload(fileName string, fileSize int64) (string, error) {
    uploadID := fmt.Sprintf("%x", randBytes(16))
    chunkDir := filepath.Join(um.tmpDir, uploadID)
    if err := os.MkdirAll(chunkDir, 0755); err != nil {
        return "", err
    }
    // 保存元数据（实际应该存 DB）
    meta := UploadMeta{
        UploadID: uploadID,
        FileName: fileName,
        FileSize: fileSize,
        Status:   "init",
    }
    saveMetadata(uploadID, meta)
    return uploadID, nil
}

// CheckMDCopy 检查是否需要秒传
func (um *UploadManager) CheckMD5(ctx context.Context, fileMD5 string) (*FileRef, error) {
    // 从数据库/对象存储中查找相同 MD5 的文件
    ref, err := um.store.FindByMD5(fileMD5)
    if err == nil && ref != nil {
        return ref, nil // 秒传命中！
    }
    return nil, ErrNotMatched
}

// UploadChunk 接收一个分片
func (um *UploadManager) UploadChunk(r *http.Request) error {
    uploadID := r.URL.Query().Get("uploadId")
    partNumber, _ := strconv.Atoi(r.URL.Query().Get("partNumber"))

    if uploadID == "" || partNumber <= 0 {
        http.Error(w, "missing params", http.StatusBadRequest)
        return fmt.Errorf("invalid params")
    }

    // 读取分片数据（限制大小）
    maxChunkSize := int64(100 << 20) // 100MB max
    data := io.LimitReader(r.Body, maxChunkSize+1)
    content, err := io.ReadAll(data)
    if len(content) > int(maxChunkSize) {
        http.Error(w, "chunk too large", http.StatusRequestEntityTooLarge)
        return fmt.Errorf("chunk too large")
    }

    // 计算分片 ETag
    hash := md5.Sum(content)
    etag := fmt.Sprintf("\"%s\"", hex.EncodeToString(hash[:]))

    // 保存到临时文件
    chunkFile := filepath.Join(um.tmpDir, uploadID, fmt.Sprintf("%d", partNumber))
    if err := os.WriteFile(chunkFile, content, 0644); err != nil {
        return err
    }

    // 记录 ETag
    um.mu.Lock()
    if um.chunksPath[uploadID] == nil {
        um.chunksPath[uploadID] = make(map[int]string)
    }
    um.chunksPath[uploadID][partNumber] = etag
    um.mu.Unlock()

    w.Header().Set("ETag", etag)
    w.WriteHeader(http.StatusOK)
    return nil
}

// CompleteUpload 合并所有分片为完整文件
func (um *UploadManager) CompleteUpload(uploadID string, parts []PartInfo) error {
    // 1. 按 partNumber 排序
    sort.Slice(parts, func(i, j int) bool { return parts[i].PartNumber < parts[j].PartNumber })

    // 2. 逐分片追加写入
    targetFile := filepath.Join(um.storeDir, filename)
    fh, err := os.OpenFile(targetFile, os.O_CREATE|os.O_WRONLY, 0644)
    if err != nil {
        return err
    }
    defer fh.Close()

    for _, part := range parts {
        chunkPath := filepath.Join(um.tmpDir, uploadID, fmt.Sprintf("%d", part.PartNumber))
        chunkData, err := os.ReadFile(chunkPath)
        if err != nil {
            return fmt.Errorf("read chunk %d failed: %v", part.PartNumber, err)
        }
        if _, err := fh.Write(chunkData); err != nil {
            return err
        }
    }

    // 3. 校验完整文件的 MD5
    fh.Seek(0, io.SeekStart)
    fullHash := md5.New()
    io.Copy(fullHash, fh)
    finalMD5 := hex.EncodeToString(fullHash.Sum(nil))

    // 4. 更新元数据，标记完成
    updateMetadata(uploadID, UploadMeta{
        Status: "completed",
        MD5:    finalMD5,
    })

    // 5. 清理临时分片
    cleanupTempDir(filepath.Join(um.tmpDir, uploadID))

    return nil
}
```

### 3. 秒传原理详解

```
秒传的核心是「以文件内容特征代替文件名」。

传统做法：不同用户各自上传同一个文件，服务器存多份副本 → 浪费空间
秒传做法：先算文件的 MD5，如果已经有用户上传过相同内容的文件
         → 直接返回现有文件的下载链接 → 零等待、零传输 ✨
```

**面试追问点**：

```
Q：MD5 碰撞怎么办？安全性够吗？

A：MD5 碰撞概率极低（2^128 分之一），对普通文件上传来说完全可接受。
如果是金融级安全要求，可以用 SHA-256 替代 MD5，或者用双指纹：
  fingerprint = SHA256(MD5(file_content) + file_size)

Q：秒传和 CDN 缓存的区别？

A：完全不同。秒传是"服务端存储去重"——多个用户的同一份文件只存一份。
CDN 缓存是"网络层加速"——把热门内容缓存在离用户最近的边缘节点。
两者可以组合使用：秒传减少存储冗余，CDN 加速分发。
```

### 4. 断点续传协议设计

```
HTTP Range 头实现断点续传：

客户端：GET /files/video.mp4 HTTP/1.1
Range: bytes=1048576-       # 从第 1MB 位置开始下载

服务端：HTTP/1.1 206 Partial Content
Content-Range: bytes 1048576-2147483647/2147483648
Content-Length: 2146435072

// Go 标准库处理 Range
func HandleDownload(w http.ResponseWriter, r *http.Request) {
    fileInfo, _ := os.Stat("/storage/video.mp4")
    size := fileInfo.Size()
    
    rangeHeader := r.Header.Get("Range")
    var start, end int64
    
    if rangeHeader != "" {
        // 解析 Range: bytes=1048576-
        parts := strings.Split(strings.TrimPrefix(rangeHeader, "bytes="), "-")
        start, _ = strconv.ParseInt(parts[0], 10, 64)
        if len(parts) > 1 && parts[1] != "" {
            end, _ = strconv.ParseInt(parts[1], 10, 64)
        } else {
            end = size - 1
        }
    } else {
        start, end = 0, size - 1
    }
    
    w.Header().Set("Content-Range", 
        fmt.Sprintf("bytes %d-%d/%d", start, end, size))
    w.Header().Set("Accept-Ranges", "bytes")
    w.Header().Set("Content-Length", fmt.Sprintf("%d", end-start+1))
    w.WriteHeader(http.StatusPartialContent)
    
    // 从指定位置开始写
    fh, _ := os.Open("/storage/video.mp4")
    fh.Seek(start, io.SeekStart)
    io.CopyN(w, fh, end-start+1)
}
```

### 5. 生产环境架构

```
                  ┌─────────┐
                  │ Client  │
                  └────┬────┘
                       │
              分片上传 (multipart/form-data)
                       │
                  ┌────▼────┐
             ┌────│ Gateway │── 鉴权 + 限流
             │    └────┬────┘
             │         │
             │    ┌────▼────┐    ┌──────────┐
             │    │ Upload  │───→│ MinIO/S3 │  ← 分片存储
             │    │ Service │    │ (对象存储) │
             │    └────┬────┘    └────┬─────┘
             │         │              │
             │         ▼              │
             │    ┌──────────┐        │
             │    │ Merge    │───→ 合并后存入 OSS
             │    │ Service  │        │
             │    └──────────┘        │
             │                        │
             │    生成 Pre-Signed URL  │
             │         ▼              │
             │    ┌──────────┐        │
             │    │ Download │←───────┘
             │    │ Service  │
             │    └──────────┘
             │
          CDN 边缘缓存 ←── 优化下载带宽
```

**关键决策点**：

| 问题 | 方案 | 理由 |
|------|------|------|
| 分片大小 | 5~10MB | 太小增加请求数，太大容错差 |
| 存储方式 | MinIO/S3 对象存储 | 自带分片上传 API，成本低 |
| 合并策略 | 全量上传完再合并 vs 边传边合并 | 前者简单可靠，后者省空间 |
| 重试机制 | 每个分片独立 IDempotency Key | 单分片失败不影响整体 |

### 6. 面试话术

**Q：如何防止上传过程中恶意提交超大文件？**

> 三层防护：1）网关层限制单次请求大小（如 Nginx client_max_body_size）；2）UploadService 对每个分片设置上限（如 100MB），拒绝超大分片；3）初始化阶段传入预期文件大小，合并时做长度校验。三层配合基本杜绝了恶意大文件攻击。

**Q：分片数量和分片大小的选取依据是什么？**

> S3 的分片数量在 10000 以内，建议每片 5MB 以上。对于 2GB 的文件，5MB 切片约 400 个分片。太多分片会增加管理复杂度，太少会导致单个分片过大，一旦某个分片失败就需要重传大量数据。我们内部实践是动态调整：小文件固定 1 个分片，大文件按 min(10MB, fileSize/400) 来切。
