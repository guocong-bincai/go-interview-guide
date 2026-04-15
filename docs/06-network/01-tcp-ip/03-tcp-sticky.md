# TCP 粘包与拆包

> 考察频率：★★★★★  难度：★★★☆☆
> 关键词：粘包、拆包、半包、消息边界、协议设计、LengthFieldBasedFrameDecoder

## 🎯 面试官考察意图

- 是否真正理解 TCP 是流协议而非消息协议
- 能否说清粘包和拆包产生的原因（应用层缓冲区、TCP 自身机制）
- 是否掌握常见的解决方案（固定长度、特殊分隔符、长度字段）
- 能否针对业务场景设计合理的协议

---

## ⚡ 核心答案（30秒）

> TCP 是**流协议**，不是消息协议。数据像水流一样在字节流中传输，没有消息边界。粘包是指多个业务消息粘在一起，拆包是指一个业务消息被分散到多个 TCP 报文中。
>
> **产生原因**：发送方每条消息大小不确定；接收方缓冲区一次可能包含多条消息或多条消息的一部分。
>
> **解决方案**：固定长度（如 100 字节，不够补 0）；特殊分隔符（如 \n，但内容中不能出现）；长度字段（最常用，如前 4 字节表示消息长度，后跟实际数据）。

---

## 🔬 深度展开

### 一、为什么 TCP 会粘包/拆包？

#### 1. TCP 自身机制

```
发送方进程 A                     接收方进程 B
write("hello")  ──>  [TCP发送缓冲区]  ──>  [TCP接收缓冲区]  ──>  read()
write("world")  ──>                                    可能一次性读到 "helloworld"
```

**Nagle 算法**：把小包合并成大包再发（减少网络小包数量）
**TCP_NODELAY**：禁用 Nagle，小包立即发

#### 2. 应用层缓冲区

接收方的 `read()` 是从操作系统缓冲区读取，不保证一次读完一条业务消息：
- 可能只读到半条消息（半包）
- 可能一次读到多条消息（粘包）

```go
// 接收方视角
conn, _ := net.Dial("tcp", "server:8080")

buf := make([]byte, 1024)
// 场景1：服务端发了两条消息 "hello" + "world"
//        read() 可能读到 "helloworld"（粘包）
// 场景2：服务端发了一条很长的消息 "hello..."
//        read() 可能只读到 "hel"（半包）
n, _ := conn.Read(buf)
```

#### 3. 实际栗子

```
发送：["hello", "world"]
TCP层：
  情况1（粘包）：收到 "helloworld" 一次性 read() 读到
  情况2（拆包+粘包）：分两次 read()："hel" + "loworld"

发送：[100KB 业务数据]
TCP层：
  情况1（正常）：一次 read() 读到 100KB
  情况2（半包）：第一次 read() 只读到 50KB
                第二次 read() 读到剩余 50KB
```

---

### 二、四大解决方案

#### 方案1：固定长度（Fixed Length）

**原理**：每个消息都是固定长度，不够补 0 或空格。

```go
// 服务端：每次读取固定 100 字节
func handleFixedLength(conn net.Conn) {
    buf := make([]byte, 100)
    for {
        n, err := conn.Read(buf)
        if err != nil {
            return
        }
        // buf[0:n] 就是一条消息
        processMessage(buf[:n])
    }
}

// 客户端：消息补齐到固定长度
func sendFixedLength(conn net.Conn, msg string) {
    const MaxLen = 100
    data := make([]byte, MaxLen)
    copy(data, msg)
    conn.Write(data)
}
```

**缺点**：空间浪费；消息长度受限；不适合变长消息

#### 方案2：特殊分隔符（Delimiter）

**原理**：用特殊字符（如 `\n`、`\r\n`、自定义 `\x00`）标记消息边界。

```go
// 使用bufio.Scanner（内置分隔符支持）
func handleDelimiter(conn net.Conn) {
    scanner := bufio.NewScanner(conn)
    // 设置分隔符为换行符
    scanner.Split(bufio.ScanLines)
    for scanner.Scan() {
        line := scanner.Text()
        processMessage(line)
    }
    if err := scanner.Err(); err != nil {
        // handle error
    }
}

// 自定义分隔符：比如HTTP的\r\n\r\n
func handleCustomDelimiter(conn net.Conn) {
    scanner := bufio.NewScanner(conn)
    splitFunc := func(data []byte, atEOF bool) (advance int, token []byte, err error) {
        if atEOF && len(data) == 0 {
            return 0, nil, nil
        }
        // 查找特殊分隔符 \x00
        if i := bytes.Index(data, []byte{0}); i >= 0 {
            return i + 1, data[0:i], nil
        }
        if atEOF {
            return len(data), data, nil
        }
        return 0, nil, nil
    }
    scanner.Split(splitFunc)
    for scanner.Scan() {
        processMessage(scanner.Bytes())
    }
}
```

**缺点**：内容中不能包含分隔符（需转义）；性能略差

#### 方案3：长度字段（Length Field）—— 最常用

**原理**：消息头部包含长度字段，先读长度，再读对应字节。

```go
// 自定义协议：
// [4字节长度][实际数据]
// 长度字段 = 4字节big-endian整数，表示后续数据长度

import "encoding/binary"

// Encode 编码：长度 + 数据
func Encode(msg string) []byte {
    length := uint32(len(msg))
    result := make([]byte, 4+length)
    binary.BigEndian.PutUint32(result[0:4], length)
    copy(result[4:], msg)
    return result
}

// Decode 解码（处理半包/粘包）
type Decoder struct {
    buf *bytes.Buffer
}

func NewDecoder() *Decoder {
    return &Decoder{buf: bytes.NewBuffer(nil)}
}

func (d *Decoder) Decode(conn net.Conn) ([]string, error) {
    // 先把所有数据读入缓冲区
    io.Copy(d.buf, conn)

    var messages []string
    for {
        // 剩余字节数 < 4，无法读长度
        if d.buf.Len() < 4 {
            break
        }

        // 偷看前4字节，获取消息长度
        lengthBytes := d.buf.Next(4)
        length := binary.BigEndian.Uint32(lengthBytes)

        // 剩余字节数 < length，无法完整读消息
        if d.buf.Len() < int(length) {
            // 把偷看的4字节放回去（d.buf.Next已消费）
            d.buf.Unreadn(lengthBytes)
            break
        }

        // 读取完整消息
        msg := d.buf.Next(int(length))
        messages = append(messages, string(msg))
    }
    return messages, nil
}
```

**更完善的 LengthFieldBased 解码器**（Netty 风格）：

```go
// LengthFieldBasedFrameDecoder 参数说明：
// lengthFieldOffset:   长度字段的偏移量（跳过前多少字节）
// lengthFieldLength:   长度字段本身占多少字节（1/2/3/4）
// lengthAdjustment:    长度修正值（总长度 = 长度字段值 + adjustment）
// initialBytesToStrip: 解码后跳过前多少字节

// 示例： "\x00\x00\x00\x0Ahello"
// lengthFieldOffset = 0, lengthFieldLength = 4
// lengthAdjustment = 0, initialBytesToStrip = 0
// 解析结果：length=10, data="hello"

// 示例："\x00\x00\x00\x0Ahello world" (带header)
// 如果 header 占 2 字节，如 "\xAB\xCD\x00\x00\x00\x0Ahello"
// lengthFieldOffset = 2
```

#### 方案4：混合方案 + 协议设计

实际生产用得最多的是**长度字段**，但会组合使用：

```go
// 生产级协议设计
// [魔数2字节][版本1字节][消息类型1字节][长度4字节][数据][校验4字节]
// 魔数：固定 0xAB 0xCD，防错
// 版本：协议版本，兼容升级
// 消息类型：区分请求/响应/心跳等
// 长度：数据部分长度
// 校验：CRC32，防数据损坏

type Message struct {
    Magic    uint16
    Version  uint8
    Type     uint8
    Length   uint32
    Data     []byte
    Checksum uint32
}

func DecodeMessage(conn net.Conn) (*Message, error) {
    header := make([]byte, 8) // 2+1+1+4
    if _, err := io.ReadFull(conn, header); err != nil {
        return nil, err
    }

    magic := binary.BigEndian.Uint16(header[0:2])
    if magic != 0xABCD {
        return nil, errors.New("invalid magic")
    }

    version := header[2]
    msgType := header[3]
    length := binary.BigEndian.Uint32(header[4:8])

    data := make([]byte, length)
    if _, err := io.ReadFull(conn, data); err != nil {
        return nil, err
    }

    // 读校验和
    checksumBuf := make([]byte, 4)
    io.ReadFull(conn, checksumBuf)
    checksum := binary.BigEndian.Uint32(checksumBuf)

    // 校验
    if crc32.ChecksumIEEE(data) != checksum {
        return nil, errors.New("checksum mismatch")
    }

    return &Message{
        Magic:    magic,
        Version:  version,
        Type:     msgType,
        Length:   length,
        Data:     data,
        Checksum: checksum,
    }, nil
}
```

---

### 三、Go 标准库中的粘包处理

#### bufio.Reader 的行为

```go
// bufio.Reader.ReadString('\n') 内部会处理半包
// 但如果一条消息超过 bufio.DefaultBufSize (4096)，需调整
reader := bufio.NewReaderSize(conn, 8192) // 增大缓冲区
```

#### binary 包的多字节读写

```go
// 大端序编码/解码
binary.BigEndian.PutUint32(buf, value)
binary.LittleEndian.PutUint32(buf, value)

// Varint 编码（可变长度，整数小的时候占1字节，省空间）
// Google Protocol Buffers 用的是 Varint
```

---

### 四、常见协议对比

| 协议 | 消息边界 | 特点 |
|------|---------|------|
| HTTP | 分隔符（\r\n\r\n） | 文本， header + body |
| WebSocket | 长度字段 | 二进制帧，支持掩码 |
| gRPC | Length-Prefixed-Message | 基于 HTTP/2，Length-Prefixed |
| Redis RESP | 分隔符（\r\n） | 简单文本协议 |
| Thrift | 长度字段 | 二进制，跨语言 |

---

## 💻 面试手写题

### 题目：实现一个粘包处理解码器

实现一个带长度字段的 TCP 消息解码器，能正确处理粘包和半包。

```go
package main

import (
	"bytes"
	"encoding/binary"
	"errors"
	"fmt"
	"io"
)

// Packet 消息包
type Packet struct {
	Type    uint8  // 消息类型
	Payload []byte // 消息内容
}

// Decoder 基于长度字段的解码器
type Decoder struct {
	buf *bytes.Buffer
}

func NewDecoder() *Decoder {
	return &Decoder{buf: bytes.NewBuffer(nil)}
}

// Feed 将数据放入解码器缓冲区
func (d *Decoder) Feed(data []byte) (int, error) {
	return d.buf.Write(data)
}

// Extract 从缓冲区提取一条完整消息，返回剩余未处理的数据
// 可能返回 nil, nil（半包）或者 Packet, nil（完整消息）
func (d *Decoder) Extract() (*Packet, error) {
	// 协议格式：[Type(1字节)][Length(4字节 bigendian)][Payload(n字节)]
	// 至少需要 5 字节才能解析一条消息
	if d.buf.Len() < 5 {
		return nil, nil
	}

	// 保存当前位置，以便半包时回退
	originalLen := d.buf.Len()
	originalBytes := d.buf.Bytes()

	// 读取 Type
	msgType, _ := d.buf.ReadByte()

	// 读取 Length
	lengthBuf := make([]byte, 4)
	if _, err := io.ReadFull(d.buf, lengthBuf); err != nil {
		// 回退
		d.buf = bytes.NewBuffer(originalBytes)
		return nil, nil
	}
	length := binary.BigEndian.Uint32(lengthBuf)

	// 检查数据是否足够
	if uint32(d.buf.Len()) < length {
		// 半包，回退
		d.buf = bytes.NewBuffer(originalBytes)
		return nil, nil
	}

	// 读取 Payload
	payload := make([]byte, length)
	if _, err := io.ReadFull(d.buf, payload); err != nil {
		return nil, err
	}

	return &Packet{
		Type:    msgType,
		Payload: payload,
	}, nil
}

// ExtractAll 提取所有完整消息
func (d *Decoder) ExtractAll() ([]*Packet, error) {
	var packets []*Packet
	for {
		pkt, err := d.Extract()
		if err != nil {
			return packets, err
		}
		if pkt == nil {
			break
		}
		packets = append(packets, pkt)
	}
	return packets, nil
}

// Encode 编码一条消息
func Encode(pkt *Packet) []byte {
	// [Type(1)][Length(4)][Payload(n)]
	result := make([]byte, 5+len(pkt.Payload))
	result[0] = pkt.Type
	binary.BigEndian.PutUint32(result[1:5], uint32(len(pkt.Payload)))
	copy(result[5:], pkt.Payload)
	return result
}

// 模拟粘包/半包场景
func main() {
	decoder := NewDecoder()

	// 发送两条消息
	msg1 := Encode(&Packet{Type: 1, Payload: []byte("hello")})
	msg2 := Encode(&Packet{Type: 2, Payload: []byte("world")})

	// 场景1：正常收到两条消息
	fmt.Println("=== 正常两条消息 ===")
	decoder.Feed(msg1)
	decoder.Feed(msg2)
	packets, _ := decoder.ExtractAll()
	for _, p := range packets {
		fmt.Printf("收到: type=%d, payload=%s\n", p.Type, string(p.Payload))
	}

	// 场景2：粘包（两条消息粘在一起）
	fmt.Println("\n=== 粘包 ===")
	decoder = NewDecoder()
	stuck := append(msg1, msg2...)
	decoder.Feed(stuck)
	packets, _ = decoder.ExtractAll()
	for _, p := range packets {
		fmt.Printf("收到: type=%d, payload=%s\n", p.Type, string(p.Payload))
	}

	// 场景3：半包（只收到部分数据）
	fmt.Println("\n=== 半包 ===")
	decoder = NewDecoder()
	decoder.Feed(msg1[:6]) // 只收到前6字节（type + length，不够）
	fmt.Printf("缓冲区大小: %d\n", decoder.buf.Len())
	pkt, _ := decoder.Extract()
	fmt.Printf("提取结果: %v\n", pkt) // nil

	decoder.Feed(msg1[6:]) // 收到剩余部分
	pkt, _ = decoder.Extract()
	fmt.Printf("收到: type=%d, payload=%s\n", pkt.Type, string(pkt.Payload))
}
```

---

## 📋 总结 checklist

- [ ] 能解释 TCP 是流协议，不是消息协议
- [ ] 能列举粘包和拆包的 3 个产生原因
- [ ] 能说清 4 种解决方案及其适用场景
- [ ] 能手写 LengthFieldBased 解码器，处理半包和粘包
- [ ] 了解生产级协议设计的要素（魔数、版本、校验和）
- [ ] 知道 HTTP/1.1、gRPC、WebSocket 分别怎么解决消息边界
