# go-Cache

基于 Go 实现的分布式缓存库，借鉴了 [groupcache](https://github.com/golang/groupcache) 的设计思想，支持 LRU 淘汰、分布式节点和一致性哈希等特性。

---

## 项目概览

go-Cache 是一个轻量级的分布式内存缓存系统，主要解决高并发场景下的缓存访问问题。当缓存未命中时，支持通过自定义 `Getter` 从数据源加载数据；在分布式部署下，可自动将请求路由到持有该数据的节点，实现节点间数据共享与负载均衡。

**核心能力：**
- 本地 LRU 缓存，支持按字节数限制容量
- 分布式多节点，通过 HTTP 通信
- 一致性哈希选择 peer，降低节点变更带来的数据迁移
- 单飞 (singleflight) 合并并发请求，避免缓存击穿

---

## ✨ 特性

- **LRU 缓存淘汰**：基于内存容量限制，自动淘汰最近最少使用的缓存项
- **Group 命名空间**：支持多个独立的缓存组，按业务维度隔离
- **Getter 接口**：未命中时通过自定义 `Getter` 从数据库、API 等数据源加载
- **单飞模式**：同一 key 的并发请求只执行一次加载，防止缓存击穿
- **HTTP 节点通信**：使用 HTTP 协议实现节点间数据获取，易于部署和调试
- **一致性哈希**：Peer 选择采用一致性哈希，节点变化时减少数据迁移
- **不可变 ByteView**：缓存值以只读视图提供，避免外部修改污染缓存

---

## 🚀 快速开始

### 前提条件

- **运行环境**：Go 1.16+
- **依赖工具**：Go 官方工具链（无需额外第三方依赖，仅使用标准库）

### 安装步骤

1. **克隆仓库**

```bash
git clone https://github.com/park338/go-Cache.git
cd go-Cache
```

2. **安装依赖**

```bash
go mod download
```

3. **构建与测试（可选）**

```bash
go build ./...
go test ./...
```

### 基本使用

**1. 创建缓存组并实现 Getter**

```go
package main

import (
	"fmt"
	"log"
	"net/http"

	"go-Cache/go-cache"
)

// 模拟数据源
var db = map[string]string{
	"Tom":  "630",
	"Jack": "589",
	"Sam":  "567",
}

func main() {
	// 创建缓存组：名称、最大缓存字节数、数据加载函数
	groupName := "scores"
	group := go_cache.NewGroup(groupName, 1<<20, go_cache.GetterFunc(
		func(key string) ([]byte, error) {
			log.Println("[DB] search key", key)
			if v, ok := db[key]; ok {
				return []byte(v), nil
			}
			return nil, fmt.Errorf("%s not exist", key)
		},
	))

	// 启动 HTTP 服务（监听 8001 端口）
	addr := "http://localhost:8001"
	peers := go_cache.NewHTTPPool(addr)
	group.RegisterPeers(peers)

	log.Println("go-cache is running at", addr)
	log.Fatal(http.ListenAndServe(":8001", peers))
}
```

**2. 分布式部署多节点**

分别在多个终端启动不同端口的节点，并注册 peers：

```go
// 节点 1: localhost:8001
peers := go_cache.NewHTTPPool("http://localhost:8001")
peers.Set("http://localhost:8001", "http://localhost:8002", "http://localhost:8003")
group.RegisterPeers(peers)
```

**3. 发起缓存请求**

- 通过 HTTP 访问：`GET http://localhost:8001/_geecache/<groupname>/<key>`
- 在代码中直接使用：`group.Get(key)`

```go
view, err := group.Get("Tom")
if err != nil {
	log.Fatal(err)
}
fmt.Println(view.String()) // 输出: 630
```

---

## 📁 项目结构

```
go-Cache/
├── go.mod
├── go-cache/
│   ├── geecache.go      # 核心 Group、Getter、负载逻辑
│   ├── cache.go         # 缓存封装（LRU + 并发安全）
│   ├── byteview.go      # 不可变字节视图
│   ├── http.go          # HTTPPool、HTTP 通信
│   ├── peers.go         # PeerPicker、PeerGetter 接口
│   ├── lru/
│   │   └── lru.go       # LRU 缓存实现
│   ├── singleflight/
│   │   └── singleflight.go  # 单飞合并请求
│   └── consistenthash/
│       └── consistenthash.go  # 一致性哈希
└── README.md
```

---

## 📄 许可证

MIT License
