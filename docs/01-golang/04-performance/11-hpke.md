# 混合公钥加密（HPKE）：Go 1.26 `crypto/hpke` 实战与面试解析

## 面试官考察意图

考察候选人对混合加密体系（Hybrid Encryption）和后量子密码学（Post-Quantum Cryptography, PQC）的理解深度。
HPKE（Hybrid Public Key Encryption，RFC 9180）是 Go 1.26 引入的重要密码学标准，它**结合传统 ECDH 与后量子 ML-KEM**，解决"今日加密、明日解密"（Harvest Now, Decrypt Later）威胁。高级工程师在涉及 TLS 1.3、Messaging Layer Security（MLS）、VPN、数字信封等场景时必须掌握 HPKE 的设计动机和 API 使用。

---

## 核心答案（30 秒版）

HPKE 是一种"混合"加密方案：发送方用接收方公钥封装（Encapsulate）一个对称密钥，再用该对称密钥加密消息。接收方用私钥解封装（Decapsulate）得到对称密钥，从而解密。

Go 1.26 的 `crypto/hpke` 支持两种密钥封装机制（KEM）：
- **传统 KEM**：ECDH（X25519、P256 等），与 TLS 1.3 的 ECDHE 同源
- **后量子 KEM**：ML-KEM-768（Go 1.24 引入的 Kyber），NIST PQC 标准

```go
kem, kdf, aead := MLKEM768X25519(), HKDFSHA256(), AES256GCM()

// 接收方生成公钥
recipientPrivateKey := kem.GenerateKey()
publicKeyBytes := recipientPrivateKey.PublicKey().Bytes()

// 发送方加密
publicKey, _ := kem.NewPublicKey(publicKeyBytes)
ciphertext, _ := Seal(publicKey, kdf, aead, []byte("app-info"), []byte("secret message"))

// 接收方解密
plaintext, _ := Open(recipientPrivateKey, kdf, aead, []byte("app-info"), ciphertext)
```

**为何重要**：传统 ECDH 被量子计算机攻击后会失效，HPKE 通过 ML-KEM 提供后量子安全，ML-KEM768X25519 混合方案兼具"今天安全 + 量子安全"。

---

## 深度展开

### 1. 为什么需要 HPKE？

**问题 1：TLS 1.3 已经解决了前向保密，HPKE 有什么不同？**

TLS 1.3 使用 ECDHE 进行密钥交换，提供了前向保密（Forward Secrecy）。但 TLS 1.3 是**传输层协议**，需要 TCP 握手，不适合 UDP 场景（QUIC 除外）或嵌入式消息系统。

HPKE 是**应用层原语**，可以独立使用，适用于：
- 消息加密（Signal Protocol、MLS）
- 数字信封（Enveloped Data，PKCS#7/CMS 的现代替代）
- 存储加密（At-rest encryption）
- 轻量级 IoT 协议

**问题 2：Harvest Now, Decrypt Later 威胁**

量子计算机尚未实用，但攻击者已经开始**今天截获加密流量，明天用量子计算机解密**。这对长期敏感数据（医疗记录、知识产权、政府文档）危害极大。

HPKE 通过 ML-KEM 提供**量子安全**的密钥封装，即使量子计算机出现，已加密数据仍受保护。

### 2. HPKE 架构：三个密码学组件

HPKE 由三部分组成（ciphersuite）：

| 组件 | 作用 | Go 中对应 |
|------|------|---------|
| **KEM**（Key Encapsulation Mechanism） | 用公钥加密封装对称密钥 | MLKEM768X25519, DHKEM(P256/X25519) |
| **KDF**（Key Derivation Function） | 从原始材料派生出加密密钥 | HKDFSHA256, HKDFSHA384, HKDFSHA512 |
| **AEAD**（Authenticated Encryption with Associated Data） | 对称加密 + 防篡改 | AES256GCM, ChaCha20Poly1305 |

**HPKE 模式（Mode）**：

| 模式 | API | 说明 |
|------|-----|------|
| **Base** | `Seal` / `Open` | 只使用接收方公钥，最常用 |
| **PSK**（Pre-Shared Key） | `NewSender` + `NewRecipient` | 额外使用预共享密钥 |
| **Auth** | DHKEM + 签名 | 发送方也拥有密钥对（相互认证） |
| **AuthPSK** | 组合 | 既认证又用 PSK |

### 3. 代码示例：数字信封场景

数字信封 = 用接收方公钥封装内容加密密钥（CEK），再用 CEK 加密实际数据：

```go
package main

import (
    "crypto/hpke"
    "crypto/rand"
    "fmt"
)

// HPKE_KEM_X25519_HKDFSHA256_AES256GCM
func encryptForRecipient(recipientPublicKey []byte, plaintext []byte) ([]byte, error) {
    kem := hpke.KEM{X: hpke.MLKEM768X25519}
    kdf := hpke.KDF{H: hpke.HKDFSHA256}
    aead := hpke.AEAD{A: hpke.AES256GCM}

    pk, err := kem.NewPublicKey(recipientPublicKey)
    if err != nil {
        return nil, fmt.Errorf("invalid public key: %w", err)
    }

    // info 防止在不同应用场景中混用密钥
    info := []byte("my-app/v1/envelope")

    // 封装密钥 + 加密消息（一次 API 调用）
    return Seal(pk, kdf, aead, info, plaintext)
}

func decryptForRecipient(recipientPrivateKey hpke.PrivateKey, ciphertext []byte) ([]byte, error) {
    kem := hpke.KEM{X: hpke.MLKEM768X25519}
    kdf := hpke.KDF{H: hpke.HKDFSHA256}
    aead := hpke.AEAD{A: hpke.AES256GCM}
    info := []byte("my-app/v1/envelope")

    return Open(recipientPrivateKey, kdf, aead, info, ciphertext)
}
```

### 4. HPKE vs 传统 hybrid encryption（ECIES）

| 维度 | HPKE（RFC 9180）| ECIES |
|------|----------------|-------|
| 标准性 | RFC 9180，通用标准 | 各实现不兼容 |
| KEM 灵活性 | KEM/KDF/AEAD 自由组合 | 固定组合 |
| 后量子支持 | ✅ ML-KEM | ❌ 通常不支持 |
| 使用场景 | MLS、Signal、文档加密 | 传统 CMS/PKCS#7 |

### 5. HPKE 与 TLS 1.3 的关系

TLS 1.3 使用了类似 HPKE 的思想（ECDHE + HKDF + AEAD），但 HPKE 不是 TLS 的替代品，而是**构建块**：

- **MLS（Messaging Layer Security）** 使用 HPKE 作为密钥封装协议
- **混合存储加密**（文件系统、云存储）可用 HPKE
- **QUIC** 的 0-RTT 模式也受 HPKE 思想影响

### 6. Go 中 HPKE 的 KEM 类型速查

```go
// 纯传统（量子计算机可破解）
DHKEM(P256), DHKEM(X25519), DHKEM(P384)

// 纯后量子（传统计算机不可破解，但量子计算机理论上可）
MLKEM768, MLKEM1024

// 混合方案（今天 + 量子都安全）⭐ 推荐
MLKEM768X25519, MLKEM768P256, MLKEM1024P384
```

---

## 高频追问

**Q：HPKE 和 ECDH + AES-GCM 相比有什么优势？**

> 优势有三点：① 标准统一，不同实现可互操作；② 天然支持后量子 KEM；③ 组件可拆卸，可根据安全等级选择 KEM+KDF+AEAD 组合。传统 ECDH+AES-GCM 是定制方案，HPKE 是通用原语。

**Q：ML-KEM 安全吗？有 NIST 认证吗？**

> ML-KEM（原 Kyber）已通过 NIST 后量子密码标准化流程，于 2024 年正式成为 FIPS 203。Go 1.24 在 `crypto/mlkem` 中实现，Go 1.26 在 `crypto/hpke` 中集成。

**Q：HPKE 的 `info` 参数有什么用？**

> `info` 是场景标识（application info），类似于 HKDF 的 context。相同公钥+不同 info 会派生出不同密钥，防止同一密钥在不同协议中混用。例如：用不同 info 区分"加密"和"签名"用途。

**Q：HPKE 的性能如何？**

> ML-KEM768 的 Encapsulate 约 10~20 万次/秒（单核），比传统 ECDH 慢但差距不大。混合方案（MLKEM768X25519）需要运行两次 KEM，稍慢于纯 ECDH。接收方 Decapsulate 通常更快。

---

## 延伸阅读

- [RFC 9180: HPKE](https://www.rfc-editor.org/rfc/rfc9180.html)
- [crypto/hpke package docs](https://pkg.go.dev/crypto/hpke)
- [NIST Post-Quantum Cryptography](https://csrc.nist.gov/projects/post-quantum-cryptography)
- [FIPS 203: ML-KEM](https://doi.org/10.6028/NIST.FIPS.203)
