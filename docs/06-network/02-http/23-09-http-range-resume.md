# HTTP Range 请求与断点续传：206、分片上传与大文件传输

> 考察频率：★★★☆☆  优先级：P2
> 关键词：Range、206 Partial Content、Accept-Ranges、Content-Range、断点续传、分片上传、秒传、If-Range、多段响应

---

## 🎯 面试官考察意图

这道题通常从**"怎么实现大文件上传/下载"**切入。面试官想看的是：

1. 你知不知道 **HTTP 原生就支持 Range**（不用自己发明协议）；
2. 能不能**把"断点续传"拆成下载侧和上传侧两套方案**；
3. 有没有想过**分片上传的工程细节**——分片大小多少？并发几路？怎么保证顺序？怎么清理垃圾分片？

**这是"系统设计 + 协议细节"的结合点**，在网盘、视频、云存储类公司的面试中非常高频。

---

## ⚡ 核心答案（30秒）

**HTTP Range 是 HTTP/1.1 原生的部分内容请求机制。**

```http
GET /video.mp4 HTTP/1.1
Range: bytes=1024-2047        ← 请求第 1024~2047 字节（共 1024 字节）
```

```http
HTTP/1.1 206 Partial Content
Accept-Ranges: bytes
Content-Range: bytes 1024-2047/10485760   ← 本次范围 / 总大小
Content-Length: 1024
```

**要点：**

| 要素 | 值 |
|------|-----|
| 成功状态码 | **206 Partial Content**（不是 200！） |
| 服务端声明支持 | `Accept-Ranges: bytes` |
| 越界请求 | 返回 **416 Range Not Satisfiable** |
| 不支持 Range 的服务端 | 返回 200 + 完整内容（客户端应降级处理） |
| 条件 Range | `If-Range: <etag 或 date>` —— 资源变了就返回完整 200 |

**断点续传三要素：**
1. **下载侧**：客户端记录已下载字节数，续传时带 `Range: bytes=N-`；
2. **上传侧**：HTTP 没有原生分片上传，需要自己设计（切片 + 编号 + 合并）；
3. **幂等校验**：用 ETag / MD5 保证分片和最终文件的一致性。

---

## 🔬 深度展开

### 1. Range 请求的几种形式

```http
Range: bytes=0-499        # 前 500 字节
Range: bytes=500-999      # 第 501~1000 字节
Range: bytes=-500         # 最后 500 字节（注意：这里是"后缀范围"）
Range: bytes=500-         # 从 500 字节到结尾
Range: bytes=0-0,-1       # 多段请求（首字节 + 末字节）
Range: bytes=0-9, 20-29   # 多段请求
```

**多段响应用 `multipart/byteranges`：**

```http
HTTP/1.1 206 Partial Content
Content-Type: multipart/byteranges; boundary=3d6b6a416f9b5

--3d6b6a416f9b5
Content-Type: text/plain
Content-Range: bytes 0-9/100

0123456789
--3d6b6a416f9b5
Content-Type: text/plain
Content-Range: bytes 20-29/100

abcdefghij
--3d6b6a416f9b5--
```

> ⚠️ **实践建议**：多段响应解析复杂、收益低，**实际生产几乎不用**。要并行下载就发多个独立的单段 Range 请求。

**`If-Range`：防止续传拿到脏数据**

```http
GET /video.mp4 HTTP/1.1
Range: bytes=1024-
If-Range: "abc123"        ← ETag 或 Date
```

- 资源**未变** → 返回 `206` + 相应范围；
- 资源**已变** → 返回 `200` + **完整内容**（客户端必须丢弃已下载的部分，重新开始）。

**为什么需要它？** 如果没有 `If-Range`，文件在两次请求之间被更新了，客户端把"旧文件的前 1KB"和"新文件的剩余部分"拼接，得到的就是一个损坏的文件。

### 2. Go 服务端实现下载：优先用标准库

```go
func downloadHandler(w http.ResponseWriter, r *http.Request) {
	f, err := os.Open("/data/video.mp4")
	if err != nil {
		http.Error(w, "not found", http.StatusNotFound)
		return
	}
	defer f.Close()

	// ServeContent 自动处理：
	//   ✅ Accept-Ranges / Range / 206 / 416
	//   ✅ If-Range / If-None-Match / If-Modified-Since（协商缓存）
	//   ✅ Content-Type 嗅探（也可显式传 Content-Type 覆盖）
	//   ✅ Range 与 ETag 的联动
	http.ServeContent(w, r, "video.mp4", time.Now(), f)
}
```

**`http.ServeContent` 的签名：**

```go
func ServeContent(w ResponseWriter, req *Request, name string, modtime time.Time, content io.ReadSeeker) 
```

> **关键要求：`content` 必须是 `io.ReadSeeker`**（能 Seek 才能定位到 Range 起点）。
> 如果你手上只有 `io.Reader`（比如从 S3 流式读），`ServeContent` 会退化成一次全量读进内存，非常危险。

**从 S3/OSS 流式转发 + 保留 Range（常见需求）：**

```go
// 完整保留 Range 语义的代理实现（划重点：透传 Range 和状态码）
func s3Proxy(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()
	key := r.URL.Query().Get("key")

	out, err := s3Client.GetObject(ctx, &s3.GetObjectInput{
		Bucket: aws.String("my-bucket"),
		Key:    aws.String(key),
		Range:  aws.String(r.Header.Get("Range")), // ← 关键：透传 Range 给 S3
	})
	if err != nil {
		// 416 必须原样返回，不能吞掉变成 500
		var apiErr smithy.APIError
		if errors.As(err, &apiErr) && apiErr.ErrorCode() == "InvalidRange" {
			w.WriteHeader(http.StatusRequestedRangeNotSatisfiable)
			return
		}
		http.Error(w, "upstream error", http.StatusBadGateway)
		return
	}
	defer out.Body.Close()

	// 透传关键头
	if out.ContentRange != nil {
		w.Header().Set("Content-Range", *out.ContentRange)
	}
	if out.AcceptRanges != nil {
		w.Header().Set("Accept-Ranges", *out.AcceptRanges)
	}
	if out.ETag != nil {
		w.Header().Set("ETag", *out.ETag)
	}
	if out.ContentLength != nil {
		w.Header().Set("Content-Length", strconv.FormatInt(*out.ContentLength, 10))
	}
	if out.ContentType != nil {
		w.Header().Set("Content-Type", *out.ContentType)
	}

	// 关键：S3 返回 206 时，我们也必须返回 206
	status := http.StatusOK
	if r.Header.Get("Range") != "" && out.ContentRange != nil {
		status = http.StatusPartialContent
	}
	w.WriteHeader(status)

	// 流式拷贝，不要 io.ReadAll（大文件会 OOM）
	if _, err := io.Copy(w, out.Body); err != nil {
		log.Printf("proxy copy: %v", err) // 客户端断开是常态，不要 panic
	}
}
```

> ⚠️ **大文件传输最重要的三个字：不要 `io.ReadAll`。** 10GB 的文件读进内存会直接 OOM。用 `io.Copy`（底层走 `sendfile` 或 `splice`）。

### 3. 断点续传（下载）：客户端实现

```go
// 支持断点续传的下载器
type Downloader struct {
	url      string
	filePath string
	client   *http.Client
}

func (d *Downloader) Download(ctx context.Context) error {
	// 1) 看本地已下载多少
	var offset int64
	if fi, err := os.Stat(d.filePath); err == nil {
		offset = fi.Size()
	}

	// 2) 构造请求（带 Range）
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, d.url, nil)
	if err != nil {
		return err
	}
	if offset > 0 {
		req.Header.Set("Range", fmt.Sprintf("bytes=%d-", offset))
	}

	resp, err := d.client.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()

	// 3) 判断服务端是否支持续传
	var flag int
	switch resp.StatusCode {
	case http.StatusPartialContent:
		flag = os.O_WRONLY | os.O_APPEND // 206：支持，追加写
	case http.StatusOK:
		flag = os.O_WRONLY | os.O_CREATE | os.O_TRUNC // 200：不支持，从头开始（必须清空！）
		offset = 0
	case http.StatusRequestedRangeNotSatisfiable:
		// 416：本地文件比服务端还大 → 文件已损坏或已完整下载
		return fmt.Errorf("range not satisfiable, local file may be corrupt")
	default:
		return fmt.Errorf("unexpected status: %d", resp.StatusCode)
	}

	dst, err := os.OpenFile(d.filePath, flag, 0644)
	if err != nil {
		return err
	}
	defer dst.Close()

	n, err := io.Copy(dst, resp.Body)
	if err != nil {
		return fmt.Errorf("copy after %d bytes: %w", n, err)
	}
	log.Printf("downloaded %d more bytes (total offset was %d)", n, offset)
	return nil
}
```

**关键细节（面试可讲）：**

| 细节 | 为什么 |
|------|-------|
| 收到 `200` 时必须 **`O_TRUNC` 清空文件** | 否则会得到"新文件头 + 旧文件尾"的损坏文件 |
| 收到 `416` 要处理 | 说明本地比服务端大，通常是上次续传出错了 |
| 用 `If-Range` 而非裸 `Range` | 避免文件变了还续传 |
| 断点信息落地（如 `.download.meta`） | 记录 ETag、总大小、已下载字节，重启后可校验 |
| 慢速下载要加超时 | 用 `context` 而不是 `client.Timeout`（大文件总时长不可控） |

### 4. 分片上传（上传侧，HTTP 没有原生支持）

**HTTP 协议没有"续传上传"的标准机制**，必须自己设计。业界标准方案：

```
┌─────────────────────────────────────────────────────────┐
│  Step 1: 初始化上传                                       │
│  POST /api/upload/init                                   │
│    { "filename": "a.zip", "size": 1073741824,            │
│      "sha256": "<whole-file-hash>", "chunkSize": 5242880 }│
│  ← { "uploadId": "u_xxx", "chunks": [0,1,2,...],         │
│      "uploaded": [0,3] }    ← 返回已完成的分片（断点续传） │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│  Step 2: 并发上传分片                                     │
│  PUT /api/upload/u_xxx/chunk/5                           │
│    Body: <binary 5MB>                                    │
│  ← { "etag": "chunk-md5" }                               │
│  （并发 3~5 路，失败重试，每片独立校验）                    │
└─────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────┐
│  Step 3: 合并                                            │
│  POST /api/upload/u_xxx/complete                         │
│    { "parts": [{"n":0,"etag":"..."}, ...],               │
│      "sha256": "<whole-file-hash>" }                     │
│  ← { "fileId": "f_yyy", "url": "https://..." }           │
│  （服务端校验完整性 → 合并分片 → 删临时文件 → 返回可访问URL）│
└─────────────────────────────────────────────────────────┘
```

**Go 服务端：分片合并的坑**

```go
// ❌ 错误：把所有分片读进内存再写
func mergeBad(uploadID string, n int) error {
	var buf bytes.Buffer
	for i := 0; i < n; i++ {
		data, _ := os.ReadFile(chunkPath(uploadID, i))
		buf.Write(data)     // 1GB 文件 → 1GB 内存 → OOM
	}
	return os.WriteFile(finalPath, buf.Bytes(), 0644)
}

// ✅ 正确：逐片流式拷贝到最终文件
func mergeGood(uploadID string, n int, finalPath string) error {
	out, err := os.Create(finalPath)
	if err != nil {
		return err
	}
	defer out.Close()

	for i := 0; i < n; i++ {
		in, err := os.Open(chunkPath(uploadID, i))
		if err != nil {
			return fmt.Errorf("open chunk %d: %w", i, err)
		}
		// 顺序合并，必须严格按分片序号
		if _, err := io.Copy(out, in); err != nil {
			in.Close()
			return fmt.Errorf("copy chunk %d: %w", i, err)
		}
		in.Close()
	}

	// 关键：落盘后再返回，避免返回 URL 后文件还没写完
	if err := out.Sync(); err != nil {
		return err
	}
	return nil
}
```

**分片大小怎么选？（高频追问）**

| 文件大小 | 建议分片 |
|---------|---------|
| < 10MB | 不用分片，直接传 |
| 10MB ~ 100MB | 1~5MB |
| 100MB ~ 1GB | 5~10MB |
| > 1GB | 10~50MB |

**权衡：**
- 分片太小 → 分片数爆炸（S3 上限 10000 片），请求开销大；
- 分片太大 → 单片失败重传成本高，内存占用大；
- **经验：以"单片重传成本 / 请求数"平衡，5MB 是很好的默认值**（S3 官方推荐）。

**并发路数：** 3~5 路。太少慢，太多会打满带宽导致所有分片都慢，而且服务端压力大。

### 5. 秒传与去重

**秒传原理：**

```
客户端计算文件 sha256（大文件要分片算再合并哈希）
   │
   ▼
POST /api/upload/check  { "sha256": "..." }
   │
   ├─ 服务端命中已有文件（引用计数 +1）→ 直接返回 fileId 【秒传】
   └─ 未命中 → 返回 uploadId，走正常分片上传流程
```

**工程细节：**

| 问题 | 方案 |
|------|------|
| 计算 sha256 本身很慢（1GB 要几秒） | 用分片哈希并行计算；或先用「大小 + 前 64KB MB 哈希」做快速预判（样抽样哈希） |
| 秒传被恶意利用（探测别人文件） | 命中后**不直接返回可访问 URL**，改为"复制到当前用户的命名空间"，并做权限校验 |
| 存储去重 | 内容寻址存储（CAS，如 OSS 的 `content-md5` 校验、Git 的对象存储） |
| 引用计数 | 只有引用数为 0 才真删物理文件 |

> ⚠️ **安全警告**：早期网盘因为"秒传 + 未鉴权"被大量用来"收藏"违规/他人文件（俗称"秒传链接"）。**正确做法：秒传命中后做所有权归属复制，而不是共享同一个文件引用。**

### 6. Nginx 配置要点

```nginx
location /download/ {
    # 开启 Range 支持（默认开启）
    # 显式声明，避免误解
    add_header Accept-Ranges bytes always;

    # 大文件传输优化
    sendfile on;
    tcp_nopush on;        # 配合 sendfile，减少网络包
    tcp_nodelay on;       # 对下载影响不大，但对小响应有用

    # 输出缓冲：大文件建议关闭缓冲，直接流式发送
    proxy_buffering off;
    proxy_max_temp_file_size 0;

    # 超时：大文件下载要放宽
    proxy_read_timeout 300s;
    proxy_send_timeout 300s;

    client_max_body_size 0;   # 上传不限制（0 = 不限制，慎用）
}
```

**缓存与 Range 的冲突（容易踩的坑）：**

如果开启了 `proxy_cache`，默认情况下 Nginx 会在**回源完整文件后**才能响应 Range 请求（因为它要先缓存全量）。对大视频会导致"首播卡很久"。

```nginx
# 方案 A：对视频类路径关闭缓存，只做透传
location ~* \.(mp4|mkv|flv)$ {
    proxy_pass http://backend;
    proxy_cache off;
    # 让后端直接处理 Range，流式返回
}

# 方案 B：Nginx 支持 slice 模块，切片缓存
location /video/ {
    slice 1m;                    # 每个 slice 1MB
    proxy_cache video_cache;
    proxy_cache_key $uri$is_args$args$slice_range;
    proxy_set_header Range $slice_range;   # 关键：把 slice 范围转成 Range 请求
    proxy_cache_valid 200 206 1h;
    proxy_pass http://backend;
}
```

> `slice` 模块是 Nginx 处理大文件缓存的**最优解**：它把文件切成 1MB 的块，每块独立缓存和回源，客户端请求 Range 时只回源缺失的块。

---

## ❓ 高频追问

### Q1：为什么 206 响应必须带 `Content-Range`？

`Content-Range: bytes 1024-2047/10485760` 告诉客户端：
- 本次返回的是哪个字节范围；
- 文件总大小是多少（用于显示进度、计算剩余部分）。

**缺了它，客户端无法知道数据在文件中的位置**，只能按顺序追加，无法做并发分片下载。

### Q2：什么是"多段 Range"？为什么不推荐用？

多段 Range 是一次请求拿多个不连续区间（`bytes=0-9,20-29`），服务端返回 `multipart/byteranges`。

不推荐的原因：
1. 客户端解析成本高（要解析 MIME 边界）；
2. 服务端实现复杂，容易出错；
3. **现代做法是并发发多个单段请求**（浏览器、下载器、`aria2` 都这么做），更简单也更快（可以并行）。

### Q3：什么情况下服务端会忽略 Range 请求返回 200？

| 情况 | 说明 |
|------|------|
| 服务端未声明 `Accept-Ranges: bytes` | 不支持部分内容 |
| `If-Range` 校验失败（资源已变） | 按规范返回完整内容 |
| Range 语法非法 | 规范允许忽略而非报错 |
| 资源是动态生成的（无已知总长度） | 没有 Seek 能力 |

**客户端必须处理 200 的情况**：清空本地已下载内容，从头开始。

### Q4：视频网站的"拖动进度条秒开"是怎么实现的？

1. **服务端支持 Range**（声明 `Accept-Ranges: bytes`）；
2. **播放器解析 moov box 在文件头**（`faststart` 优化）——这样播放器不用下载整个文件就知道总时长和索引；
3. 拖动时**发 `Range` 请求**获取目标时间点附近的分片；
4. **CDN 边缘节点支持 Range 缓存**——命中则本地切片返回，不命中才回源；
5. **HLS/DASH 场景**：视频被切成 2~10 秒的 TS/fMP4 小文件，拖动就是切换分片 URL，天然支持"任意位置起播"。

> 面试加分：**HLS/DASH 是现代流媒体的主流方案，它放弃了"单文件 Range"路线，改用"小分片 + m3u8 索引"**，因为这样可以做多码率自适应（ABR）、分片级缓存、分片级 DRM。

### Q5：断点续传怎么保证"断在半路"的数据没被写坏？

1. **分片上传**：每片独立校验（`Content-MD5` 或 `sha256`），坏片重传；
2. **下载**：定期 `fsync`，并维护 `.meta` 文件记录 `(已下载字节数, ETag, 总大小)`；重启时校验 ETag 是否一致；
3. **最终校验**：合并完成后计算整体哈希与服务端对比（如 HTTP `Digest` 头或业务字段）；
4. **原子替换**：先在临时文件下载，校验通过后再 `os.Rename` 到最终路径（同分区内的 rename 是原子的）。

```go
// 原子替换：避免用户读到"半成品"文件
if err := verify(tmpPath, expectedSHA256); err != nil {
    return err
}
if err := os.Rename(tmpPath, finalPath); err != nil { // 同目录下 rename 是原子操作
    return err
}
```

---

## 📋 总结 checklist

- [ ] 知道 Range 请求的 4 种形式，206/416/200 的语义
- [ ] 知道 `If-Range` 的作用和为什么需要它
- [ ] 知道 Go 里用 `http.ServeContent` 而不是手写 Range 解析
- [ ] 知道 `ServeContent` 要求 `io.ReadSeeker`
- [ ] 知道下载续传收到 200 时必须清空本地文件
- [ ] 能画出分片上传的三阶段流程（init / chunk / complete）
- [ ] 知道分片合并必须流式拷贝，不能 `ReadAll`
- [ ] 知道 5MB 是分片大小的良好默认值，并发 3~5 路
- [ ] 知道秒传的安全风险（要复制归属而非共享引用）
- [ ] 知道 Nginx `slice` 模块解决大文件 Range 缓存问题

---

## 📚 延伸阅读

- [RFC 9110: Range Requests](https://www.rfc-editor.org/rfc/rfc9110.html#name-range-requests)
- [MDN: HTTP Range requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Range_requests)
- [AWS S3: Multipart Upload 概览](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [Nginx: ngx_http_slice_module](https://nginx.org/en/docs/http/ngx_http_slice_module.html)
