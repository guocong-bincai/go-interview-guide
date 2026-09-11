# 服务注册与发现（Service Discovery）

> 考察频率：★★★★★  难度：★★★★☆  关键词：注册中心、心跳续约、客户端发现 vs 服务端发现、CP vs AP

## 一、为什么需要服务注册与发现

微服务拆分后，服务实例的 IP 是**动态**的（扩缩容、发布、漂移、K8s 重建 Pod）。调用方不可能靠静态配置文件写死地址，于是需要一个"电话簿"：实例启动时登记地址，下线时注销，调用方按服务名查询可用实例列表。

一条完整链路：**注册 → 健康检查 → 查询 → 负载均衡分发 → 注销**。

| 阶段 | 做什么 | 失败后果 |
|------|--------|----------|
| 注册 | 实例启动上报 `{service, ip, port, weight, metadata}` | 调用方查不到，流量为 0 |
| 健康检查 | 心跳续约 / 主动探活（TCP/HTTP/gRPC Health） | 流量打到已死实例 |
| 查询 | 调用方拉取或订阅实例列表 | 地址过期（stale） |
| 注销 | 优雅下线时删除实例 | 发布期间 5xx |

---

## 二、客户端发现 vs 服务端发现

| 维度 | 客户端发现（Client-Side） | 服务端发现（Server-Side） |
|------|--------------------------|--------------------------|
| 谁做负载均衡 | SDK（调用方自己选实例） | 独立 LB / 网关 / Sidecar |
| 代表 | gRPC + etcd、Eureka + Ribbon、Kratos/go-zero 内置 | K8s Service + kube-proxy、Nginx、Istio |
| 优点 | 少一跳、延迟低、可自定义 LB 策略（P2C/一致性哈希） | 语言无关、SDK 轻、策略集中管控 |
| 缺点 | 多语言各写一套 SDK、升级困难 | 多一跳、LB 成为潜在瓶颈 |
| 适合 | Go/Java 单一技术栈、追求极致延迟 | 多语言、异构、K8s 环境 |

> 面试点：**K8s 本身就是服务发现**（Service → Endpoints/EndpointSlice + CoreDNS + kube-proxy）。所以"上了 K8s 还要不要注册中心"的答案是：南北向/集群内可只用 K8s，跨集群、多环境、需要权重/元数据路由时仍要 Nacos/Consul。

---

## 三、注册中心选型：CP vs AP

| 组件 | 一致性模型 | 健康检查 | 特点 |
|------|-----------|---------|------|
| **etcd** | CP（Raft） | 租约 Lease + KeepAlive | 强一致，K8s 底层；租约到期自动删除，天然防脏数据 |
| **Consul** | CP（Raft） | 主动探活 + 反熵 | 多数据中心、内置 DNS/健康检查 |
| **Nacos** | 可切换 AP/CP | 心跳 + 主动探测 | 注册与配置一体，国内生态好 |
| **Eureka** | AP | 心跳续约 | 已停更，去中心化，允许脏数据 |

**核心权衡**：注册中心本质上"宁可返回一个已死的实例，也不能返回空列表"——空列表意味着全部流量失败。所以很多注册中心默认选择 **AP + 客户端容错**（本地缓存 + 缓存不过期兜底），而不是 CP。

```
CP 崩溃场景：网络分区时 Raft 少数派拒绝写入 → 新实例注册不上 → 但老实例列表仍可读，业务大多可用
AP 崩溃场景：分区时各节点返回各自视图 → 可能返回已下线实例 → 靠客户端重试 + 熔断兜底
```

---

## 四、Go 实现：基于 etcd 租约的注册与发现

```go
package discovery

import (
	"context"
	"time"

	"go.etcd.io/etcd/client/v3"
)

type Registry struct {
	cli    *clientv3.Client
	lease  clientv3.LeaseID
	key    string
	cancel context.CancelFunc
}

// Register 注册并持续续约（KeepAlive 内部自动续租）
func Register(cli *clientv3.Client, service, addr string, ttl int64) (*Registry, error) {
	ctx, cancel := context.WithCancel(context.Background())

	grant, err := cli.Grant(ctx, ttl) // 申请租约，ttl 秒
	if err != nil {
		cancel()
		return nil, err
	}

	key := "/services/" + service + "/" + addr
	if _, err = cli.Put(ctx, key, addr, clientv3.WithLease(grant.ID)); err != nil {
		cancel()
		return nil, err
	}

	// 进程存活期间持续续约；进程挂掉 → 租约到期 → key 自动删除
	ch, err := cli.KeepAlive(ctx, grant.ID)
	if err != nil {
		cancel()
		return nil, err
	}
	go func() {
		for range ch { // 消费续约响应，避免 channel 阻塞
		}
	}()

	return &Registry{cli: cli, lease: grant.ID, key: key, cancel: cancel}, nil
}

// Close 优雅注销
func (r *Registry) Close() {
	r.cancel()
	_, _ = r.cli.Revoke(context.Background(), r.lease) // 主动撤销租约 → 立即删除 key
}
```

```go
// Watch 订阅实例变化，维护本地缓存 + 负载均衡
func Watch(ctx context.Context, cli *clientv3.Client, service string, onChange func([]string)) {
	key := "/services/" + service + "/"

	resp, _ := cli.Get(ctx, key, clientv3.WithPrefix())
	instances := decode(resp.Kvs)
	onChange(instances)

	// 从当前 revision 之后开始监听，避免漏事件
	wch := cli.Watch(ctx, key, clientv3.WithPrefix(), clientv3.WithRev(resp.Header.Revision+1))
	for wresp := range wch {
		for _, ev := range wresp.Events {
			switch ev.Type {
			case clientv3.EventTypePut:
				instances = addInstance(instances, string(ev.Kv.Value))
			case clientv3.EventTypeDelete:
				instances = removeInstance(instances, string(ev.Kv.Value))
			}
		}
		onChange(instances)
	}
}
```

**生产必做的三件事**：

1. **本地缓存兜底**：注册中心全挂时，用最后一次的实例列表继续跑（stale read 好过不可用）。
2. **优雅下线**：先 `Revoke` 注销、再从 LB 摘流、最后停进程（配合 `preStop` + SIGTERM）。
3. **订阅式而非轮询**：Watch 长连接推变更，不要每秒拉一次（会把注册中心打爆）。

---

## 五、注册中心的经典故障与排查

| 现象 | 根因 | 处理 |
|------|------|------|
| 流量打到已下线实例 | 注销延迟 / 缓存未过期 | 缩短 TTL + 优雅下线 + 客户端熔断快速失败 |
| 发布期间 5xx 抖动 | 摘流与进程退出顺序错误 | JVM/Go 先摘流再 shutdown，留 `sleep` 窗口 |
| 注册中心被打爆 | 大量实例轮询 + 无租约 | 改 Watch 订阅、调大 TTL、实例数分片 |
| 集群脑裂后数据错乱 | CP 组件选错/分区 | 用 Raft 系（etcd/Consul），Quorum 保证写入唯一 |

---

## 面试话术

> **"注册发现解决的是'实例地址动态化'问题。选型上我通常分两步：K8s 环境优先复用 Service + CoreDNS，跨集群或需要权重/元数据路由才引入 Nacos 或 etcd。客户端发现省一跳、延迟低但要多语言 SDK；服务端发现语言无关、策略集中，代价是多一跳。可用性上我倾向 AP + 客户端本地缓存兜底——返回过期地址好过返回空列表，因为空列表等于全量失败。etcd 的租约机制是关键：进程崩溃后 TTL 到期自动删除，天然避免脏数据。"**
