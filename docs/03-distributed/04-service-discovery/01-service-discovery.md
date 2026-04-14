# 服务发现

> 考察频率：★★★☆☆  难度：★★★☆☆
> 关键词：Consul、etcd、Nacos、注册中心、健康检查

## 服务发现核心概念

```
┌─────────────────────────────────────────────────┐
│                   服务发现架构                     │
├─────────────────────────────────────────────────┤
│                                                 │
│   Service A ──注册──▶ ┌──────────────┐          │
│   Service B ──注册──▶ │  注册中心     │          │
│   Service C ──注册──▶ │  (Registry)  │          │
│                        └──────┬───────┘          │
│                               │                  │
│   Service A ◀──查询──┬────────┘                  │
│   Service B ◀─订阅──┘                          │
│                                                 │
└─────────────────────────────────────────────────┘

注册中心职责：
1. 服务注册（Register）：Provider 上线时向注册中心注册
2. 服务注销（Unregister）：Provider 下线时主动注销
3. 心跳续约（Heartbeat）：Provider 定期发送心跳，证明自己活着
4. 服务订阅（Subscribe）：Consumer 订阅服务列表变化
```

---

## 三种服务发现模式

| 模式 | 描述 | 代表 |
|------|------|------|
| **客户端发现** | Client 直接查询注册中心 | Consul + Consul Template |
| **服务端发现** | 通过 LB/网关转发到注册中心 | Kubernetes Service、nginx + Consul |
| **DNS 发现** | 通过 DNS 解析服务名 | K8s DNS (CoreDNS) |

---

## Consul 详解

### 核心特性

- **服务注册/注销**：HTTP API 或 Agent
- **健康检查**：支持 HTTP/TCP/TTL/脚本
- **KV 存储**：可用于配置中心
- **多数据中心**：支持 WAN Gossip
- **DNS 接口**：支持通过 DNS 查询服务

### Go + Consul 示例

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/hashicorp/consul/api"
)

func main() {
	// 创建 Consul 客户端
	config := api.DefaultConfig()
	config.Address = "127.0.0.1:8500"
	client, err := api.NewClient(config)
	if err != nil {
		log.Fatal(err)
	}

	// 注册服务
	err = registerService(client)
	if err != nil {
		log.Fatal(err)
	}

	// 发现服务
	err = discoverService(client)
	if err != nil {
		log.Fatal(err)
	}
}

func registerService(client *api.Client) error {
	// 定义服务注册信息
	registration := &api.AgentServiceRegistration{
		ID:   "order-service-1",         // 实例唯一 ID
		Name: "order-service",           // 服务名
		Port: 8080,                       // 服务端口
		Address: "192.168.1.100",        // 服务地址
		Tags: []string{"v1", "prod"},    // 标签
		Meta: map[string]string{
			"version": "1.0.0",
		},
		// 健康检查
		Check: &api.AgentServiceCheck{
			HTTP:                           "http://192.168.1.100:8080/health",
			Interval:                       "10s",
			Timeout:                        "5s",
			DeregisterCriticalServiceAfter: "30s", // 30s后未通过则注销
		},
	}

	return client.Agent().ServiceRegister(registration)
}

func discoverService(client *api.Client) error {
	// 方式1：通过 DNS 查询（负载均衡默认是 round-robin）
	// curl http://localhost:8500/ui/dc1/services/order-service/instances
	
	// 方式2：通过 API 查询
	services, _, err := client.Health().Service("order-service", "", true, nil)
	if err != nil {
		return err
	}

	fmt.Println("Discovered order-service instances:")
	for _, s := range services {
		fmt.Printf("  - %s:%d (status=%s, version=%s)\n",
			s.Service.Address,
			s.Service.Port,
			s.Checks[0].Status,
			s.Service.Meta["version"],
		)
	}

	// 方式3：过滤特定标签
	healthy, _, err := client.Health().ServiceWithTags("order-service", []string{"v1"}, true, nil)
	if err != nil {
		return err
	}
	fmt.Printf("V1 instances: %d\n", len(healthy))

	return nil
}

// 监听服务变化
func watchServiceChanges(client *api.Client) {
	// 监控 order-service 的变化
	index := uint64(0)
	for {
		// 带阻塞查询的服务列表
		services, meta, err := client.Health().Service("order-service", "", true,
			&api.QueryOptions{WaitIndex: index})
		if err != nil {
			log.Printf("Watch error: %v", err)
			time.Sleep(time.Second)
			continue
		}
		index = meta.LastIndex

		fmt.Printf("Service changed! %d instances:\n", len(services))
		for _, s := range services {
			fmt.Printf("  - %s:%d\n", s.Service.Address, s.Service.Port)
		}
	}
}
```

---

## etcd 服务发现

### Raft 存储 + 租约机制

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"go.etcd.io/etcd/client/v3"
)

func main() {
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"localhost:2379"},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		log.Fatal(err)
	}
	defer cli.Close()

	ctx := context.Background()

	// 注册服务：使用租约（Lease）实现自动注销
	// 1. 创建租约（TTL = 10秒）
	leaseResp, err := cli.Grant(ctx, 10)
	if err != nil {
		log.Fatal(err)
	}

	// 2. 注册服务（key = /services/order-service/{instance_id}）
	serviceKey := fmt.Sprintf("/services/order-service/%s", "instance-001")
	_, err = cli.Put(ctx, serviceKey, "192.168.1.100:8080", clientv3.WithLease(leaseResp.ID))
	if err != nil {
		log.Fatal(err)
	}

	// 3. 维持心跳（自动续约）
	ch, err := cli.KeepAlive(ctx, leaseResp.ID)
	if err != nil {
		log.Fatal(err)
	}
	go func() {
		for {
			select {
			case resp := <-ch:
				if resp == nil {
					fmt.Println("Lease expired")
					return
				}
				fmt.Printf("KeepAlive: TTL=%d\n", resp.TTL)
			}
		}
	}()

	// 发现服务：前缀查询
	resp, err := cli.Get(ctx, "/services/order-service/", clientv3.WithPrefix())
	if err != nil {
		log.Fatal(err)
	}

	fmt.Printf("Found %d service instances:\n", len(resp.Kvs))
	for _, kv := range resp.Kvs {
		fmt.Printf("  - %s: %s\n", string(kv.Key), string(kv.Value))
	}

	// 监听变化（Watch）
	go func() {
		rch := cli.Watch(ctx, "/services/order-service/", clientv3.WithPrefix())
		for wresp := range rch {
			for _, ev := range wresp.Events {
				fmt.Printf("Event: %s %s:%s\n", ev.Type, string(ev.Kv.Key), string(ev.Kv.Value))
			}
		}
	}()

	time.Sleep(30 * time.Second)
}
```

---

## Nacos（阿里）

### 特性

- **两种模式**：临时实例（ephemeral=true）和持久实例
- **命名空间隔离**：dev/test/prod
- **分组管理**：不同机房/集群
- **保护阈值**：避免全部宕机时雪崩

### Go + Nacos

```go
package main

import (
	"fmt"
	"log"

	"github.com/nacos-group/nacos-sdk-go/v2/clients"
	"github.com/nacos-group/nacos-sdk-go/v2/clients/naming_client"
	"github.com/nacos-group/nacos-sdk-go/v2/common/constant"
	"github.com/nacos-group/nacos-sdk-go/v2/vo"
)

func nacosExample() {
	// 服务端配置
	sc := []constant.ServerConfig{
		{
			IpAddr:      "127.0.0.1",
			ContextPath: "/nacos",
			Port:        8848,
			Scheme:      "http",
		},
	}

	// 客户端配置
	cc := constant.ClientConfig{
		NamespaceId:         "public", // 默认 public
		TimeoutMs:           5000,
		NotLoadCacheAtStart: true,
	}

	// 创建命名服务客户端
	namingClient, err := clients.NewNamingClient(vo.NacosClientParam{
		ServerConfigs: sc,
		ClientConfig:  &cc,
	})
	if err != nil {
		log.Fatal(err)
	}

	// 注册服务
	err = namingClient.RegisterInstance(vo.RegisterInstanceParam{
		Ip:          "192.168.1.100",
		Port:        8080,
		ServiceName: "order-service",
		Weight:      1.0,
		ClusterName: "DEFAULT",
		GroupName:   "DEFAULT_GROUP",
		Ephemeral:   true, // 临时实例（心跳保活）
	})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Println("Service registered")

	// 查询服务
	instances, err := namingClient.SelectInstances(vo.SelectInstancesParam{
		ServiceName: "order-service",
		Clusters:    []string{"DEFAULT"},
		HealthyOnly: true,
	})
	if err != nil {
		log.Fatal(err)
	}
	fmt.Printf("Found %d healthy instances\n", len(instances))
}
```

---

## 三者对比

| 对比项 | Consul | etcd | Nacos |
|--------|--------|------|-------|
| 一致性 | Raft | Raft | Raft（配置）/ Distro（注册）|
| 健康检查 | ✅ 多种 | ⚠️ TTL | ✅ 多种 |
| 多数据中心 | ✅ WAN Gossip | ⚠️ 需配置 | ✅ |
| K8s 集成 | ✅ | ✅ | ⚠️ |
| 服务网格 | ✅ Consul Connect | ❌ | ❌ |
| 成熟度 | 高 | 高 | 高（阿里大规模验证）|
| 适用场景 | 通用微服务 | K8s 原生 | 阿里技术栈 |

---

## 面试话术

**Q：服务发现是怎么发现服务下线的？**

> 两种方式：**主动探测**（注册中心定期 ping/tcp 检查）和**心跳续约**（服务主动报告自己活着）。etcd/Consul 用租约（Lease）+ 心跳续约，Nacos 用 TCP/HTTP 健康检查。服务连续几次不续约或探测失败，注册中心就把它从列表里摘除。消费者下次查询就不会拿到已下线的实例了。

**Q：注册中心挂了怎么办？**

> 1）注册中心本身高可用（etcd/Consul 都是集群）；2）客户端本地缓存（注册中心挂了也能从缓存拿到实例列表）；3）部分注册中心支持临时降级（Consul 的 Local Agent 缓存）。服务发现本身是 AP 的，CAP 取舍上优先保证可用性。生产环境一定不要单点部署注册中心。

**Q：如何实现灰度发布配合服务发现？**

> 通过标签分组：给实例打标签（version=v1/v2），消费者按标签筛选。比如 Nacos/Consul 都支持带 Tag 查询，`selectInstances(serviceName, tags=["v2"])` 就能只拿到 v2 实例，实现流量切分。也可以用 Istio/Envoy 做更细粒度的流量控制。
