# LBS 地理位置系统设计（附近的人/店）

> 考察频率：★★★★☆  优先级：P1  字节/美团/滴滴必考
> 关键词：GeoHash、Redis GEO、附近的人、Geofence、空间索引

## 1. GeoHash 编码原理

### 核心思想：二维坐标 → 一维字符串

```
GeoHash = 经度 + 纬度 分别二分编码，交替拼接

地球坐标：经度 [-180, 180]，纬度 [-90, 90]

编码步骤（以 116.397, 39.908 北京为例）：
1. 经度 116.397：
   [116.397 < 0?] 否 → 右半边(0) → 右半边(0) → ... → 01101...
2. 纬度 39.908：
   [39.908 < 0?] 否 → 右半边(1) → 左半边(0) → ... → 11001...
3. 交替拼接：01101 11001... → 最终 Base32 编码为 "wx4g0e"
```

### 精度等级表

| 位数 | 矩形边长（长×宽）| 适用场景 |
|------|----------------|---------|
| 1 位 | ~5000km | 大洲级别 |
| 2 位 | ~1250km | 国家级别 |
| 3 位 | ~156km | 省份级别 |
| 4 位 | ~39km | 城市级别 |
| 5 位 | ~4.9km | 城市内部 |
| **6 位** | **~1.2km** | **街道/商圈（附近的人推荐）** |
| 7 位 | ~153m | 精确位置 |
| **8 位** | **~38m** | **精确到建筑门口** |

### 相邻格子共享前缀

```
GeoHash 格子邻居关系：
┌─────┬─────┬─────┐
│ nw  │  n  │ ne  │
├─────┼─────┼─────┤
│  w  │  C  │  e  │   C = 当前位置
├─────┼─────┼─────┤
│ sw  │  s  │ se  │
└─────┴─────┴─────┘

任意格子都有 8 个邻居（n/s/e/w/nw/ne/sw/se）
相邻格子 GeoHash 前缀长度 ≈ 相同
```

### 边界问题（Z 曲线折叠）

```
GeoHash 的边界问题：
某些相邻位置可能落在不同 GeoHash 格子，且不共享长前缀

例如：一个商家恰好在格子边界上
用户搜"附近 1km"，搜的是 A 格子
但商家在相邻的 B 格子
如果只查 A 格子，会漏掉商家

解决方案：
1. GEORADIUS（Redis 自动处理）→ 自动搜索周围 8 个格子
2. 手动实现：搜中心格子 + 8 个邻居格子 → 合并 → 去重
```

---

## 2. Redis GEO 命令

### Go 实现

```go
import "github.com/redis/go-redis/v9"

type LBSService struct {
    redis *redis.Client
}

const driverLocationKey = "geo:drivers"

// GEOADD：添加用户/司机位置
// key = 地图 key，longitude = 经度，latitude = 纬度，member = ID
func (s *LBSService) AddDriverLocation(ctx context.Context, driverID string, lng, lat float64) error {
    return s.redis.GeoAdd(ctx, driverLocationKey, &redis.GeoLocation{
        Name:      driverID,
        Longitude: lng,
        Latitude:  lat,
    }).Err()
}

// GEORADIUS：查询附近 N km 内所有成员
func (s *LBSService) FindNearbyDrivers(ctx context.Context, lng, lat, radiusKm float64, limit int64) ([]DriverInfo, error) {
    results, err := s.redis.GeoRadius(ctx, driverLocationKey, lng, lat, &redis.GeoRadiusQuery{
        Radius:    radiusKm,
        Unit:      "km",
        WithDist:  true,  // 返回距离
        WithCoord: true,  // 返回坐标
        Count:     limit,
        Sort:      "ASC",  // 按距离升序（最近的排前面）
    }).Result()
    if err != nil {
        return nil, err
    }

    drivers := make([]DriverInfo, 0, len(results))
    for _, r := range results {
        drivers = append(drivers, DriverInfo{
            DriverID: r.Name,
            Distance:  r.Dist,
            Longitude: r.Longitude,
            Latitude:  r.Latitude,
        })
    }
    return drivers, nil
}

// GEOSEARCH（Redis 6.2+）：更灵活的附近查询
func (s *LBSService) FindNearbyByBox(ctx context.Context, lng, lat, widthKm, heightKm float64) ([]DriverInfo, error) {
    results, err := s.redis.GeoSearchLocation(ctx, driverLocationKey, &redis.GeoSearchLocationQuery{
        GeoSearchQuery: redis.GeoSearchQuery{
            Longitude:  lng,
            Latitude:   lat,
            Radius:     widthKm,
            Unit:       "km",
            WithDist:   true,
            WithCoord:  true,
            Sort:       "ASC",
        },
        BoxWidth:  widthKm,
        BoxHeight: heightKm,
    }).Result()
    if err != nil {
        return nil, err
    }
    return results, nil
}

// 计算两个成员之间的距离
func (s *LBSService) DistanceBetween(ctx context.Context, driverID1, driverID2 string) (float64, error) {
    dist, err := s.redis.GeoDist(ctx, driverLocationKey, driverID1, driverID2, "km").Result()
    if err != nil {
        return 0, err
    }
    return dist, nil
}
```

### Redis GEO 底层实现

```
Redis GEO 底层 = 有序集合（ZSet）

GeoHash → ZSet Score 转换：
1. 计算经纬度的 GeoHash 值（52 位整数）
2. 将 52 位 GeoHash 转为浮点数作为 ZSet 的 score
3. ZSet 按 score 排序 = 按 GeoHash 排序 = 按地理位置排序
4. GEORADIUS = ZSet 范围查询 + Haversine 距离过滤

为什么 ZSet？
- ZSet 有序 → 按 GeoHash 分数范围查询 = 按地理位置查询
- ZSet 支持 ZRANGEBYSCORE → 按分数范围查 → 按地理范围查
- ZSet 支持 ZINTERSTORE → 可以做地理交集（附近商家 + 商户分类）
```

---

## 3. 边界问题处理

### 手动实现周围 9 个格子搜索

```go
// GeoHash 邻居格子方向
var neighborDirections = [][2]int{
    {-1, -1}, {-1, 0}, {-1, 1},  // nw, n, ne
    {0, -1},           {0, 1},    // w, e
    {1, -1},  {1, 0},  {1, 1},    // sw, s, se
}

// 计算某个方向上的邻居 GeoHash
func getNeighborGeohash(baseHash string, direction [2]int) string {
    // 实现：根据方向修正经纬度，重新计算 GeoHash
    lat, lng := decodeGeohash(baseHash)
    // 根据 direction 偏移（不同精度对应不同距离）
    lat += float64(direction[0]) * geohashPrecision(baseHash)
    lng += float64(direction[1]) * geohashPrecision(baseHash)
    return encodeGeohash(lat, lng, len(baseHash))
}

// 搜索附近：中心格子 + 8 个邻居格子
func (s *LBSService) FindNearby9Grids(ctx context.Context, lng, lat, radiusKm float64) ([]DriverInfo, error) {
    // 1. 计算中心点 GeoHash
    precision := s.radiusToPrecision(radiusKm)
    centerHash := encodeGeohash(lat, lng, precision)

    // 2. 收集 9 个格子 key
    gridKeys := []string{driverLocationKey + ":" + centerHash}
    for _, dir := range neighborDirections {
        neighbor := getNeighborGeohash(centerHash, dir)
        gridKeys = append(gridKeys, driverLocationKey+":"+neighbor)
    }

    // 3. 批量查询所有格子（用 Pipeline）
    results := make([]DriverInfo, 0)
    for _, key := range gridKeys {
        nearby, err := s.redis.GeoRadius(ctx, key, lng, lat, &redis.GeoRadiusQuery{
            Radius:    radiusKm,
            Unit:      "km",
            WithDist:  true,
            WithCoord: true,
        }).Result()
        if err != nil {
            continue
        }
        for _, r := range nearby {
            results = append(results, DriverInfo{
                DriverID: r.Name,
                Distance: r.Dist,
            })
        }
    }

    // 4. 去重（同一司机可能出现在多个格子边界）
    return dedupeByDriverID(results), nil
}

func (s *LBSService) radiusToPrecision(radiusKm float64) int {
    switch {
    case radiusKm >= 100: return 3  // 156km
    case radiusKm >= 20:  return 4  // 39km
    case radiusKm >= 5:   return 5  // 4.9km
    case radiusKm >= 1:   return 6  // 1.2km
    case radiusKm >= 0.2: return 7  // 153m
    default:              return 8  // 38m
    }
}
```

---

## 4. 大规模 LBS 系统架构

### 分层架构

```
┌─────────────────────────────────────────────┐
│           用户层（乘客 APP / 司机 APP）       │
└──────────────────┬──────────────────────────┘
                   │ 位置上报（1s/次）
┌──────────────────▼──────────────────────────┐
│            接入层（API Gateway）              │
│   - 频率限制（每个司机最多 1 次/秒）          │
│   - 坐标校验（经纬度合法性检查）              │
│   - 数据脱敏（内部坐标 vs 对外展示坐标）       │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│         写入路径（高并发）                    │
│  Kafka Producer → Kafka → Flink Consumer    │
│   位置数据写入 Redis（实时查询）              │
│   位置数据写入 HBase（历史轨迹存储）           │
└─────────────────────────────────────────────┘
```

### 冷热分离

```go
// 热点数据：当前位置 → Redis GEO（只保留最近 5 分钟）
// 历史数据：轨迹 → HBase / TiDB（按时间分表）
//
// Redis 只存最新的 N 条位置：
// DRIVER:LOC:{driver_id} → 最新 (lat, lng, ts)
// Hash 结构，TTL = 5 分钟（过期自动清理）

func updateDriverLocation(driverID string, lat, lng float64) {
    key := fmt.Sprintf("DRIVER:LOC:%s", driverID)
    redis.HSet(ctx, key, map[string]interface{}{
        "lat": lat,
        "lng": lng,
        "ts":  time.Now().Unix(),
    })
    redis.Expire(key, 5*time.Minute)  // 5 分钟后自动过期

    // 异步写入历史轨迹
    go func() {
        hbase.Put(driverID, &HBaseRow{
            ColumnFamily: "loc",
            Columns: map[string]string{
                "ts":  time.Now().Format("20060102150405"),
                "lat": strconv.FormatFloat(lat, 'f', 6, 64),
                "lng": strconv.FormatFloat(lng, 'f', 6, 64),
            },
        })
    }()
}
```

### 地理围栏（Geofence）

```go
// Geofence：检测司机是否进入/离开某个区域
// 场景：司机进入某商圈 → 推送优惠通知

type Geofence struct {
    ID       string
    Name     string
    Polygon  []Point  // 矩形/多边形区域
    Type     string   // "circle" | "polygon"
    Center   Point    // 圆心（用于 circle 类型）
    RadiusKm float64  // 半径（用于 circle 类型）
}

// 检测点是否在多边形内（射线法）
func pointInPolygon(p Point, polygon []Point) bool {
    inside := false
    j := len(polygon) - 1
    for i := 0; i < len(polygon); i++ {
        if ((polygon[i].Lat > p.Lat) != (polygon[j].Lat > p.Lat)) &&
            (p.Lng < (polygon[j].Lng-polygon[i].Lng)*(p.Lat-polygon[i].Lat)/(polygon[j].Lat-polygon[i].Lat)+polygon[i].Lng) {
            inside = !inside
        }
        j = i
    }
    return inside
}

// 实时检测：Flink 流处理
// 每收到一个位置上报：遍历所有活跃 Geofence，判断是否触发进入/离开事件
```

---

## 5. 滴滴/美团外卖典型场景

### 场景 1：司机位置上报（高并发写入）

```
滴滴场景：
- 10 万在线司机，每秒 1 次上报 → 10 万 QPS 写入
- 乘客查询：每秒可能有 100 万次附近司机查询

优化：
1. 客户端本地缓存：每 200ms 上报一次（而非 1s）
2. 服务端批量写 Kafka：Kafka → Flink → Redis（批量减少网络次数）
3. 乘客查询：读 Redis GEORADIUS（10 万 QPS 完全没问题）
```

### 场景 2：附近 3km 内空闲司机

```go
// 乘客下单时，查询附近 3km 所有空闲司机
func findAvailableDrivers(ctx context.Context, lat, lng float64) ([]DriverInfo, error) {
    // 1. Redis GEO 查询附近 3km
    nearby, err := lbs.GeoRadius(ctx, lng, lat, 3.0, 100)
    if err != nil {
        return nil, err
    }

    // 2. 过滤状态（只返回空闲司机）
    driverIDs := make([]string, 0, len(nearby))
    for _, d := range nearby {
        driverIDs = append(driverIDs, d.DriverID)
    }

    // 3. MGET 批量查司机状态（IN/Sleeping/Offline）
    statusMap, _ := redis.MGet(ctx, driverIDs...).Result()
    available := make([]DriverInfo, 0)
    for i, status := range statusMap {
        if status == "available" {
            available = append(available, nearby[i])
        }
    }
    return available, nil
}
```

### 场景 3：商家配送范围判断

```go
// 判断用户位置是否在商家配送范围内
// 商家设置配送半径 3km，用户坐标 (lat, lng)

func isInDeliveryRange(ctx context.Context, merchantID string, userLat, userLng float64) (bool, error) {
    // 商家配送半径
    radiusKm, err := redis.Get(ctx, "merchant:"+merchantID+":delivery_radius").Float64()
    if err != nil {
        return false, err
    }

    // 商家坐标
    merchantGeo, err := redis.GeoPos(ctx, "merchant:geo", merchantID).Result()
    if err != nil || len(merchantGeo) == 0 {
        return false, errors.New("商家位置不存在")
    }

    // 计算距离
    dist, err := redis.GeoDist(ctx, "merchant:geo",
        merchantID, "user:"+merchantID, "km").Result()
    if err != nil {
        return false, err
    }

    return dist <= radiusKm, nil
}
```

### 场景 4：点是否在多边形内（外卖配送范围）

```go
// 商家配送范围：多边形区域（而非圆形）
// 判断用户坐标是否在商家多边形配送范围内

func pointInPolygon(p Point, polygon []Point) bool {
    inside := false
    n := len(polygon)
    for i, j := 0, n-1; i < n; j = i {
        if ((polygon[i].Lat > p.Lat) != (polygon[j].Lat > p.Lat)) &&
            (p.Lng < (polygon[j].Lng-polygon[i].Lng)*(p.Lat-polygon[i].Lat)/(polygon[j].Lat-polygon[i].Lat)+polygon[i].Lng) {
            inside = !inside
        }
    }
    return inside
}
```

---

## 6. 距离计算：Haversine 公式

```go
import "math"

// Haversine 计算两个经纬度之间的球面距离（km）
func haversine(lat1, lng1, lat2, lng2 float64) float64 {
    const earthRadius = 6371.0 // 地球半径 km

    dLat := degreesToRadians(lat2 - lat1)
    dLng := degreesToRadians(lng2 - lng1)

    a := math.Sin(dLat/2)*math.Sin(dLat/2) +
        math.Cos(degreesToRadians(lat1))*math.Cos(degreesToRadians(lat2))*
            math.Sin(dLng/2)*math.Sin(dLng/2)

    c := 2 * math.Atan2(math.Sqrt(a), math.Sqrt(1-a))
    return earthRadius * c
}

func degreesToRadians(deg float64) float64 {
    return deg * math.Pi / 180
}
```

### Redis GEODIST 底层

Redis 的 `GEODIST` 命令底层使用 Haversine 公式计算球面距离（当距离足够大时），但对近距离做了优化（平面近似）。实际误差 < 0.5%。

---

## 高频追问

### Q1：GeoHash 精度从 6 位升到 8 位，存储量增加多少？

**不是存储量增加，而是精度变高：**

| 精度 | 格子大小 | 格子数量（全球）|
|------|---------|--------------|
| 6 位 | ~1.2km × 0.6km | 约 5369 万 |
| 8 位 | ~38m × 19m | 约 1.38 亿 |

**Redis ZSet 角度：**
- 每个 member 只存一次，和精度无关
- 精度影响的是 GeoHash 编码长度（不影响存储数量）
- 精度提升 → 查询时格子更小 → 搜索格子数相同但每个格子内数据更少 → 查询更快

### Q2：Redis GEO 能支持多少个点？性能瓶颈在哪？

**Redis GEO 本质是 ZSet，性能瓶颈：**
- 单个 ZSet 建议 < 1000 万条（~640MB 内存）
- ZREVRANGE 查询时间 = O(log N + M)，M = 返回数量

**10 万司机场景：**
- 10 万条数据 × 64 字节 ≈ 6.4MB
- 查询延迟：O(log 10万) ≈ 17 步，完全没问题
- 可以支撑 10 万 QPS

**亿级场景（滴滴级别）：**
- 不适合单 Redis ZSet
- 需要按城市/区域分 Shard
- 每个城市一个 ZSet，查询时先路由到城市 ZSet

### Q3：如何实现"附近的人"同时保护用户隐私？

**隐私保护策略：**

```go
// 方案 1：位置模糊化（对外展示粗粒度位置）
func fuzzLocation(lat, lng float64, precisionKm float64) (float64, float64) {
    // 精度 500m → 只保留到 6 位 GeoHash
    geohash := encodeGeohash(lat, lng, 6)
    return decodeGeohashToPoint(geohash)  // 返回格子中心点
}

// 方案 2：用户主动开启"可见性"
func canSeeMe(myID, viewerID string) bool {
    // 双向好友关系才可见
    return redis.SIsMember(ctx, "friends:"+myID, viewerID)
}

// 方案 3：位置偏移（对非好友添加随机偏移）
func getDisplayedLocation(lat, lng float64, canSeePrecise bool) (float64, float64) {
    if canSeePrecise {
        return lat, lng
    }
    // 添加 ±500m 随机偏移
    return lat + (rand.Float64()-0.5)*0.01,
           lng + (rand.Float64()-0.5)*0.01
}
```
