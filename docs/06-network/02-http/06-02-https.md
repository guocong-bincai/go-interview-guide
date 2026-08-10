# HTTPS 原理、TLS 握手流程、证书链与性能优化

> 考察频率：★★★★☆  难度：★★★☆☆
> 关键词：TLS 1.2/1.3、RSA/ECDHE 握手、证书链、HSTS、0-RTT

---

## 核心答案（30 秒版）

HTTPS = HTTP + TLS。TLS 握手的核心是**密钥交换**：客户端和服务器协商出一个对称密钥，后续用这个密钥加密 HTTP 流量。

**TLS 1.2 完整握手（RSA）：**
```
TCP 三次握手
    ↓
Client Hello（支持的加密套件、客户端随机数）
    →
←  Server Hello（选中的加密套件、服务器随机数）
←  Certificate（服务器证书 + 证书链）
←  ServerHelloDone
    ↓
客户端用 CA 公钥验证证书，提取服务器公钥
    ↓
ClientKeyExchange（用服务器公钥加密的 PreMasterSecret）
    →
    ↓
双方用 随机数 + PreMasterSecret → MasterSecret → 对称密钥
    ↓
ChangeCipherSpec + Finished（用对称密钥加密的握手摘要）
    →
←  ChangeCipherSpec + Finished
    ↓
对称加密的 HTTP 通信开始
```

**TLS 1.3（2020 年后主流）：** 把握手从 2-RTT 降到 1-RTT，废弃 RSA（前向保密），引入 0-RTT。

---

## 1. 为什么 HTTPS 重要

| 风险 | 无 HTTPS 的后果 |
|------|---------------|
| 窃听 | HTTP 明文，路由器/中间人可直接读内容 |
| 篡改 | 注入广告、JS 劫持 |
| 伪装 | 假基站、钓鱼网站 |

HTTPS 防的就是中间人攻击（MITM）。

---

## 2. TLS 1.2 完整握手（RSA 密钥交换）

### 2.1 抓包分析

```
No.  Time       Source          Destination   Protocol  Length  Info
1    0.000000   192.168.1.100   93.184.216.34  TCP       74      443→52420 [SYN]
2    0.032103   93.184.216.34   192.168.1.100  TCP       74      52420→443 [SYN, ACK]
3    0.032234   192.168.1.100   93.184.216.34  TCP       66      443→52420 [ACK]
                                           ← TCP 三次握手完成
4    0.032456   192.168.1.100   93.184.216.34  TLSv1.2   517     Client Hello
5    0.063456   93.184.216.34   192.168.1.100  TLSv1.2   1484    Server Hello, Certificate, Server Hello Done
6    0.063789   192.168.1.100   93.184.216.34  TLSv1.2   333     Client Key Exchange, Change Cipher Spec, Finished
7    0.094123   93.184.216.34   192.168.1.100  TLSv1.2   131     New Session Ticket, Change Cipher Spec, Finished
```

### 2.2 各阶段详解

#### Client Hello

客户端发送：
- **TLS 版本**（如 TLS 1.2）
- **客户端随机数**（Client Random，32 字节）
- **Session ID**（复用会话时用）
- **加密套件列表**（按优先级排序）

```go
// Wireshark 中的 Client Hello 截图可看到：
Cipher Suites (18 suites)
    0xc02c  TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
    0xc02b  TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
    0x009c  TLS_RSA_WITH_AES_128_GCM_SHA256
    ...
```

#### Server Hello

服务器选择并响应：
- 选中的加密套件
- **服务器随机数**（Server Random）
- Session ID

#### Certificate（证书 + 证书链）

服务器发送证书链（从服务器证书到根 CA）：

```
Certificate chain:
0  s:www.example.com
   i:C=US, O=Let's Encrypt, CN=R3
1  s:C=US, O=Let's Encrypt, CN=R3
   i:O=Digital Signature Trust Co., CN=DST Root CA X3
2  s:O=Digital Signature Trust Co., CN=DST Root CA X3
   i:O=Digital Signature Trust Co., CN=DST Root CA X3
```

#### 密钥计算

```
MasterSecret = PRF(PreMasterSecret, "master secret", ClientRandom + ServerRandom)
SessionKey = PRF(MasterSecret, "key expansion", ClientRandom + ServerRandom)

// 后续加密用的对称密钥从这个 SessionKey 派生
```

### 2.3 RSA vs ECDHE

| 特性 | RSA 密钥交换 | ECDHE 密钥交换 |
|------|-------------|---------------|
| 握手轮数 | 2-RTT | 1-RTT（TLS 1.2）|
| 前向保密 | ❌ 不支持 | ✅ 支持 |
| 私钥泄露 | 可以解密历史记录 | 不能解密历史记录 |
| 计算成本 | RSA 私钥解密较慢 | ECDHE 更高效 |

> **前向保密（PFS）**：即使服务器的私钥未来被泄露，攻击者也无法解密之前的通信。ECDHE 每次会话用临时密钥对实现。

---

## 3. TLS 1.3（2018 年标准化）

TLS 1.3 的核心改进：

### 3.1 握手从 2-RTT 降到 1-RTT

```
TLS 1.2 ECDHE:
Client Hello (RSA 加密套件列表)
    →                              ← 1-RTT
Server Hello (选中的 ECDHE 套件)
...（后面还有密钥交换）

TLS 1.3:
Client Hello (含客户端 Key Share)
    →                              ← 1-RTT
Server Hello + Key Share + Finished
...直接进入加密HTTP
```

### 3.2 废弃的算法

TLS 1.3 **废弃**：
- RSA 密钥交换（没有前向保密）
- CBC 模式（AES-CBC 易被 BEAST 攻击）
- SHA-1
- 3DES
- RC4

TLS 1.3 **只支持**：
- AES-GCM、ChaCha20（AEAD）
- ECDHE（P-256、P-384、X25519）
- SHA-256/SHA-384

### 3.3 0-RTT

TLS 1.3 支持 0-RTT（早发数据），在 TCP 连接建立后立即发送加密数据：

```
TCP 三次握手
    ↓
Client Hello + Early Data（用 PSK 加密的应用数据）
    →
←  Server Hello + Early Data
```

**风险**：0-RTT 有重放攻击风险，只适合幂等请求。CDN 常用。

---

## 4. 证书与证书链

### 4.1 证书内容（X.509）

```
Certificate:
    Data:
        Version: v3
        Serial Number: 04:AB:CD:...
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, O=Let's Encrypt, CN=R3
        Validity:
            Not Before: 2024-01-01
            Not After:  2024-04-01
        Subject: CN=www.example.com
        Subject Alternative Name: DNS:example.com, DNS:www.example.com
        X509v3 extensions:
            X509v3 Key Usage: Digital Signature, Key Encipherment
            X509v3 Extended Key Usage: TLS Web Server Authentication
```

### 4.2 证书验证流程

```go
func verifyCert(cert *x509.Certificate, chain []*x509.Certificate, pool *x509.CertPool) error {
    // 1. 检查证书有效期
    now := time.Now()
    if now.Before(cert.NotBefore) || now.After(cert.NotAfter) {
        return errors.New("证书已过期或未生效")
    }

    // 2. 检查域名匹配
    err := cert.VerifyHostname("www.example.com")
    if err != nil {
        return errors.New("域名不匹配: " + err.Error())
    }

    // 3. 检查证书链（构建 CA 链）
    opts := x509.VerifyOptions{
        DNSName: "www.example.com",
        Intermediates: x509.NewCertPool(),
        Roots: pool, // 系统的根 CA 证书
    }

    // 4. 验证签名链
    for _, cert := range chain {
        opts.Intermediates.AddCert(cert)
    }

    _, err = cert.Verify(opts)
    return err
}
```

### 4.3 自签名证书与私有 CA

开发环境用自签名证书时，客户端需要手动导入根证书：

```bash
# 生成自签名 CA
openssl genrsa -out ca.key 2048
openssl req -new -x509 -days 365 -key ca.key -out ca.crt

# 用 CA 签发服务器证书
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt

# Go 客户端忽略证书验证（仅开发环境！）
// ⚠️ 生产环境绝对不要这样写
http.DefaultTransport.(*http.Transport).TLSClientConfig = &tls.Config{
    InsecureSkipVerify: true, // 禁止生产环境使用
}
```

---

## 5. HTTPS 性能优化

### 5.1 TLS 计算成本

HTTPS 的性能开销主要来自：
1. **TCP 三次握手**（1.5-RTT）
2. **TLS 握手**（1-2-RTT）
3. **加密计算**（AES/ChaCha20 CPU 开销）
4. **证书验证**（链式验证）

### 5.2 优化手段

#### ① TLS Session Resumption（会话恢复）

避免重复完整握手，复用之前协商的密钥：

```
// Session Ticket（服务器存密钥，客户端存票）
// 第二次连接：
Client Hello + Session Ticket
    →
←  Server Hello (认出Session) → 直接用之前密钥
```

#### ② OCSP Stapling

传统方式：浏览器下载证书后，还要在线查询 CA 的 OCSP 服务器验证证书是否吊销（额外网络开销）。

OCSP Stapling：服务器把 OCSP 响应**附在证书里**一起发给客户端，客户端无需再查 CA。

```nginx
server {
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8;
}
```

#### ③ HSTS（HTTP Strict Transport Security）

告诉浏览器强制使用 HTTPS，避免 HTTP 重定向：

```bash
# 响应头
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

#### ④ HTTP/2 多路复用

TLS 1.2 + HTTP/2 可以复用多个请求，复用 TCP 连接：

```
TCP 连接 1
    ↓
TLS 握手（1-RTT）
    ↓
HTTP/2 多路复用（多个请求共用一个连接）
```

#### ⑤ 证书链优化

证书链越短越好，通常 2-3 层最理想。Chrome 会缓存中间证书，减少重复验证。

#### ⑥ TLS 1.3 强制降级保护

```go
// Go 启用 TLS 1.3
tlsConfig := &tls.Config{
    MinVersion: tls.VersionTLS13,
    CurvePreferences: []tls.CurveID{
        tls.X25519,
        tls.CurveP256,
    },
}
```

### 5.3 性能对比

| 方案 | RTT | CPU 开销 |
|------|-----|---------|
| HTTP（无 TLS） | 1（TCP）| 0 |
| HTTPS (TLS 1.2) | 3（TCP + TLS）| AES-NI 硬件加速后很低 |
| HTTPS (TLS 1.3) | 2（TCP + TLS）| 略低于 1.2 |
| HTTPS + 0-RTT | 1（TCP）| 同 TLS 1.3 |

---

## 6. Go 中 HTTPS 实战

### 6.1 启动 HTTPS 服务

```go
package main

import (
    "crypto/tls"
    "crypto/x509"
    "fmt"
    "net/http"
    "io/ioutil"
)

func main() {
    // 方式一：直接用证书
    server := &http.Server{
        Addr: ":8443",
        TLSConfig: &tls.Config{
            MinVersion: tls.VersionTLS12,
            Curves: []tls.CurveID{tls.X25519, tls.CurveP256},
            CipherSuites: []uint16{
                tls.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
                tls.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
                tls.TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305,
            },
        },
    }

    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, HTTPS!")
    })

    // certFile, keyFile 为 PEM 格式的证书和私钥
    err := server.ListenAndServeTLS("server.crt", "server.key")
    if err != nil {
        panic(err)
    }
}
```

### 6.2 客户端验证自定义 CA

```go
func createClientWithCA(caCertPath string) *http.Client {
    caCert, err := ioutil.ReadFile(caCertPath)
    if err != nil {
        panic(err)
    }

    caCertPool := x509.NewCertPool()
    caCertPool.AppendCertsFromPEM(caCert)

    return &http.Client{
        Transport: &http.Transport{
            TLSClientConfig: &tls.Config{
                RootCAs: caCertPool,
                // 也可验证服务器证书的域名
                ServerName: "www.example.com",
            },
        },
    }
}
```

### 6.3 用 pprof 分析 TLS 开销

```go
import _ "net/http/pprof"

func main() {
    go func() {
        http.ListenAndServe(":6060", nil)
    }()

    // 启动 HTTPS 服务
    server := &http.Server{Addr: ":8443", Handler: mux}
    server.ListenAndServeTLS("cert.pem", "key.pem")
}
```

```bash
# 查看 TLS 握手时间
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
```

---

## 面试话术

**Q：HTTPS 握手需要几次网络往返？**

> TLS 1.2 RSA 握手：TCP 三次握手（1.5-RTT）+ TLS 握手（2-RTT）= 总共 3.5-RTT。TLS 1.2 ECDHE：TCP（1.5）+ TLS（1-RTT）= 2.5-RTT。TLS 1.3：TCP（1.5）+ TLS（1-RTT）= 2.5-RTT，但 1-RTT 内完成加密协商。如果算上 DNS 解析（1-RTT），一次完整的 HTTPS 请求需要 4-5 轮网络往返。

**Q：数字证书的签名是什么原理？**

> 证书是 CA 用自己的**私钥**对服务器的公钥 + 域名等信息做签名生成的。客户端用 CA 的**公钥**（预装在系统或浏览器中）验证这个签名，确认证书是 CA 签发的、且内容没被篡改。证书链就是层层签名：根 CA → 中间 CA → 服务器证书。

**Q：为什么 TLS 1.3 废弃了 RSA 密钥交换？**

> RSA 密钥交换的问题是：服务器用私钥加密的 PreMasterSecret，如果私钥泄露，攻击者可以解出这个 Secret，进而算出对称密钥，就能解密历史通信（没有前向保密）。ECDHE 每次用临时密钥对，私钥泄露也只能解密当前会话，无法解密历史。

**Q：证书过期了怎么办？**

> 证书过期后浏览器会阻止访问（NET::ERR_CERT_EXPIRED）。处理方式：1）续期证书（Let's Encrypt 自动续期）；2）如果是内部系统，用心跳监控提前预警；3）浏览器缓存了旧证书时，清缓存 + 重启浏览器；4）紧急情况可以用 `chrome://flags/#allow-insecure-localhost` 临时绕过（仅开发环境）。

**Q：HTTP Public Key Pinning (HPKP) 为什么被废弃了？**

> HPKP 允许网站告诉浏览器"只信任这个证书的公钥"。但一旦网站误配或私钥泄露，用户会被永久封在网站外（不可逆）。2018 年被 Chrome 废弃，改用 CT（Certificate Transparency）+ CSP。

---

## 延伸阅读

- [RFC 8446 - TLS 1.3](https://tools.ietf.org/html/rfc8446)
- [RFC 5246 - TLS 1.2](https://tools.ietf.org/html/rfc5246)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
- [Let's Encrypt 免费证书](https://letsencrypt.org/)
