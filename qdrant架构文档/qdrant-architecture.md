# Qdrant 架构文档

> **项目**: [qdrant/qdrant](https://github.com/qdrant/qdrant)
> **版本**: 1.18.3
> **许可**: Apache-2.0
> **作者**: Andrey Vasnetsov & Qdrant Team
> **语言**: Rust (edition 2024)
> **代码规模**: 1,391 个 .rs 文件，约 360,000 行 Rust 代码

---

## 1. 项目概述

Qdrant（读作 "quadrant"）是一个用 Rust 编写的**向量相似度搜索引擎和向量数据库**。它提供生产级的服务，通过便捷的 API 来存储、搜索和管理带有附加 payload 的点——即向量及其元数据。Qdrant 专为扩展过滤支持而设计，使其适用于各种神经网络或语义匹配、分面搜索和其他应用。

### 1.1 核心特性

- **Dense, Sparse 和 Multi Vector 搜索**：支持稠密向量（语义相似度）、稀疏向量（全文搜索）和多向量搜索（ColBERT 等 late interaction 模型）
- **Payload 过滤**：附加任意 JSON payload 到向量上，支持 should/must/must_not 条件组合，包括关键词匹配、全文搜索、数值范围、地理位置等过滤
- **Hybrid Search**：在单个查询中组合多个向量，通过 RRF（Reciprocal Rank Fusion）或 DBSF（Distribution-Based Score Fusion）策略融合结果
- **向量量化**：内置量化（Scalar/PRODUCT/BNARY），可减少 97% RAM 使用
- **分布式部署**：通过分片和副本实现水平扩展，支持零停机更新和扩缩容
- **双 API**：REST API（OpenAPI 3.0）+ gRPC API
- **快照与恢复**：支持本地和 S3 存储快照
- **RBAC**：JWT 角色访问控制
- **Qdrant Edge**：轻量级嵌入式版本，运行在应用进程内

### 1.2 技术栈

| 层级 | 技术 |
|------|------|
| 语言 | Rust 2024 edition, MSRV 1.94 |
| HTTP 框架 | actix-web 4.x |
| gRPC 框架 | tonic 0.14 (自定义 fork) |
| 异步运行时 | tokio 1.x |
| 分布式共识 | raft (tikv/raft-rs) |
| 内存分配器 | tikv-jemallocator |
| 序列化 | serde / serde_json / prost (Protocol Buffers) |
| Schema | schemars (JSON Schema generation → OpenAPI) |
| TLS | rustls 0.23 (ring crypto provider) |
| 对象存储 | object_store (AWS/GCP/Azure) |
| 指标 | prometheus |
| 内存映射 | memmap2 |
| SIMD | bytemuck + 手写 AVX/SSE/NEON |

### 1.3 项目规模

| 模块 | 文件数 | 代码行数 | 职责 |
|------|--------|----------|------|
| `lib/segment` | 570 | 132,045 | 核心索引/搜索/存储引擎 |
| `lib/collection` | 204 | 66,679 | Collection 管理、分片、操作 |
| `src/` | 114 | 31,633 | 服务入口、API、共识 |
| `lib/api` | 20 | 26,937 | REST/gRPC 数据模型 |
| `lib/common` | 158 | 23,178 | 公共工具库 |
| `lib/quantization` | 51 | 19,724 | 向量量化 |
| `lib/shard` | 63 | 17,695 | 分片实现 |
| `lib/edge` | 82 | 14,082 | 边缘设备版本 |
| `lib/storage` | 47 | 13,201 | 存储管理器 |
| `lib/gridstore` | 20 | 6,275 | Grid 存储 |
| `lib/sparse` | 24 | 5,525 | 稀疏向量 |
| `lib/wal` | 11 | 3,589 | 写前日志 |
| `lib/gpu` | 12 | 2,659 | GPU 加速 |
| `lib/posting_list` | 8 | 1,121 | 倒排索引表 |
| `lib/bm25` | 2 | 371 | BM25 评分 |
| `lib/trififo` | 3 | 883 | 三路 FIFO 缓冲 |
| **合计** | **~1,391** | **~360,000** | |

---

## 2. 架构总览

### 2.1 分层架构

```
┌─────────────────────────────────────────────────────────────┐
│                     客户端 (Python/JS/Go/Rust/Java/.NET)      │
└──────────┬───────────────────────────────┬──────────────────┘
           │ REST (6333)                    │ gRPC (6334)
┌──────────▼───────────┐          ┌────────▼──────────────────┐
│    actix-web REST    │          │    tonic gRPC              │
│  (actix/api/*)       │          │  (tonic/api/*)             │
│  auth/metrics/CORS   │          │  auth/logging              │
└──────────┬───────────┘          └────────┬──────────────────┘
           │           ┌─────────────┐      │
           └──────────►│  Dispatcher  │◄─────┘
                       │  (路由决策)   │
                       └──────┬───────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
    ┌─────────▼──────┐ ┌─────▼──────┐ ┌──────▼──────┐
    │ TableOfContent │ │ Consensus  │ │  RBAC       │
    │ (ToC - 存储目录) │ │ (Raft)     │ │  (权限控制)  │
    └────────┬───────┘ └────────────┘ └─────────────┘
             │
    ┌────────▼────────┐
    │  Collection     │   每个 Collection = 多个 Shard
    │  Manager        │
    └────────┬────────┘
             │
    ┌────────▼────────┐
    │  Shard          │   Local / Remote / Proxy
    │  (分片)          │
    └────────┬────────┘
             │
    ┌────────▼────────┐
    │  Segment        │   不可变存储单元
    │  (段)            │
    ├─────────────────┤
    │ • Vector Storage│   稠密/稀疏/多向量/量化
    │ • HNSW Index    │   近邻图索引
    │ • Payload Index │   字段索引 (keyword/integer/float/geo/text/bool/datetime/uuid)
    │ • ID Tracker    │   内部 ID ↔ 外部 ID 映射
    │ • Payload Store │   payload 存储 (in-memory / mmap)
    └────────┬────────┘
             │
    ┌────────▼────────┐
    │  WAL            │   写前日志
    └─────────────────┘
```

### 2.2 Workspace 成员（Cargo workspace）

```
lib/api          → REST/gRPC 数据模型和 Schema
lib/segment      → 核心引擎：索引、向量存储、评分、payload
lib/collection   → Collection 生命周期、分片、操作处理
lib/shard        → 分片实现（Local/Remote/Proxy Segment）
lib/storage      → 存储管理器、TableOfContent、Dispatcher
lib/wal          → 写前日志
lib/quantization → 标量/乘积量化
lib/sparse       → 稀疏向量支持
lib/bm25         → BM25 相关性评分
lib/common       → 公共工具（cancel/issues/io_bridge/dataset）
lib/gridstore    → Grid 存储引擎
lib/posting_list → 倒排索引
lib/gpu          → GPU 加速索引构建
lib/edge         → Qdrant Edge（嵌入式版本）
lib/trififo      → 三路 FIFO 缓冲区
lib/macros       → 过程宏
```

---

## 3. 核心概念与数据模型

### 3.1 点（Point）

Qdrant 的基本数据单元。一个 Point 包含：

- **ID**：`u64` 或 `UUID`（外部标识）
- **Vector(s)**：一个或多个命名向量（Dense / Sparse / Multi-vector）
- **Payload**：任意 JSON 元数据

### 3.2 Collection（集合）

顶层容器，包含一组 Points。每个 Collection 配置：

- 向量参数（维度、距离度量、量化配置）
- 分片数量和副本因子
- HNSW 索引参数
- 优化器参数

### 3.3 Shard（分片）

Collection 的水平分割单元。每个 Shard 持有 Collection 数据的一个子集。

- **Local Shard**：本地存储和查询
- **Remote Shard**：在另一个节点上，通过 gRPC 访问
- **Proxy Shard**：转发到远端节点
- **Replica Set**：同一 Shard 的多个副本，通过 Raft 共识保持一致

### 3.4 Segment（段）

Shard 内部的**不可变存储单元**。这是 Qdrant 性能设计的核心：

- 每个 Segment 独立持有向量存储、HNSW 索引、payload 索引、ID 追踪器
- 写入时创建新 Segment，后台优化器合并 Segment
- 读取时并行查询多个 Segment，结果合并
- Segment 大小由 `max_segment_size_kb` 控制

### 3.5 距离度量

`lib/segment/src/spaces/` 实现了四种度量空间：

- **Cosine**：余弦相似度（归一化点积）
- **Dot Product**：点积（内积）
- **Euclid**：欧几里得距离（L2）
- **Manhattan**：曼哈顿距离（L1）

每种度量都有 SIMD 优化版本：`simple_avx.rs`（x86-64）、`simple_neon.rs`（ARM）、`simple_sse.rs`（x86 老架构），以及 `metric_f16.rs`（半精度浮点）。

---

## 4. 核心模块详解

### 4.1 Segment 引擎 (`lib/segment`, 570 文件 / 132K 行)

这是 Qdrant 的心脏，负责向量索引、搜索和过滤。

#### 4.1.1 目录结构

```
lib/segment/src/
├── index/
│   ├── hnsw_index/          # HNSW 图索引（核心）
│   ├── field_index/         # Payload 字段索引
│   │   ├── ...              # keyword/integer/float/geo/text/bool/datetime/uuid
│   ├── sparse_index/        # 稀疏向量索引
│   ├── query_optimization/  # 查询优化器
│   ├── read_only/           # 只读索引
│   └── struct_payload_index/# 结构化 payload 索引
├── vector_storage/
│   ├── dense/               # 稠密向量存储
│   ├── multi_dense/         # 多稠密向量（ColBERT 等）
│   ├── sparse/              # 稀疏向量存储
│   ├── quantized/           # 量化向量存储
│   ├── query_scorer/        # 查询评分器
│   ├── chunked_vectors/     # 分块向量管理
│   └── read_only/           # 只读存储
├── payload_storage/
│   ├── in_memory_payload_storage.rs  # 内存 payload
│   ├── mmap_payload_storage.rs       # mmap payload
│   └── condition_checker.rs          # 过滤条件检查器
├── id_tracker/              # 内部 ID ↔ 外部 ID 映射
├── spaces/                  # 距离度量空间
│   ├── simple.rs            # 基础实现
│   ├── simple_avx.rs        # AVX SIMD
│   ├── simple_neon.rs       # NEON SIMD
│   └── metric_f16.rs        # 半精度浮点
├── data_types/              # 数据类型定义
├── segment_constructor/     # Segment 构建器
├── entry/                   # 入口点
├── json_path/               # JSON 路径解析
├── common/                  # Segment 内部公共工具
├── types.rs (5,405 行)      # 核心类型定义
└── lib.rs                   # 模块入口
```

#### 4.1.2 HNSW 索引实现

HNSW（Hierarchical Navigable Small World）是 Qdrant 的核心 ANN（近似最近邻）索引算法。

关键参数（来自 config.yaml）：

- `m: 16` — 每个节点的边数
- `ef_construct: 100` — 构建时考虑的邻居数
- `full_scan_threshold_kb: 10000` — 小于此大小时使用全扫描替代 HNSW
- `max_indexing_threads: 0` — 并行构建线程数（0 = 自动）
- `on_disk: false` — 是否存储在磁盘上

#### 4.1.3 Payload 过滤

Qdrant 的过滤系统支持丰富的条件类型：

- **Keyword 匹配**：精确关键词匹配
- **全文搜索**：基于分词的文本搜索
- **数值范围**：Integer/Float 范围过滤
- **地理位置**：基于 geo bounding box / radius
- **Datetime 范围**：RFC3339 时间过滤
- **UUID 匹配**：UUID 精确匹配
- **Boolean**：布尔值过滤
- **条件组合**：should（OR）/ must（AND）/ must_not（NOT）

过滤在搜索时通过 `ConditionChecker` 实现，支持预过滤（先过滤再搜索）和后过滤（先搜索再过滤）。

#### 4.1.4 量化

`lib/quantization/`（51 文件 / 19,724 行）实现了三种量化方式：

- **Scalar Quantization**：将 float32 压缩到 int8，减少 75% 内存
- **Product Quantization (PQ)**：将向量分割成子向量后量化，减少 ~97% 内存
- **Binary Quantization**：将向量压缩为二进制，极致压缩

量化后使用原始向量进行 rescoring 保证精度。

### 4.2 Collection 管理 (`lib/collection`, 204 文件 / 66K 行)

```
lib/collection/src/
├── collection/           # Collection 生命周期管理
├── collection_manager/   # Collection 操作调度
├── operations/           # 各种操作定义（搜索/更新/推荐等）
├── shards/              # 分片管理
│   ├── channel_service.rs    # 节点间 gRPC 通道池
│   ├── replica_set.rs        # 副本集管理
│   ├── local_shard.rs        # 本地分片
│   ├── remote_shard.rs       # 远程分片
│   └── proxy_shard.rs        # 代理分片
├── update_workers/       # 更新工作线程池
├── grouping/             # 分组搜索
├── lookup/               # 查找器
├── profiling/            # 性能分析
└── problems/             # 问题检测
```

Collection 管理的核心流程：

1. **创建**：通过 REST/gRPC API 发起 → Dispatcher → TableOfContent → 分配 Shard → 创建 Segment
2. **更新**：写入 WAL → 分发到对应 Shard → 应用到 Segment
3. **搜索**：并行查询所有 Shard → 合并结果 → 返回 Top-K
4. **优化**：后台优化器持续合并 Segment、清理已删除向量

### 4.3 分片 (`lib/shard`, 63 文件 / 17K 行)

```
lib/shard/src/
├── operations/          # 操作处理
├── optimizers/          # Segment 优化器
├── segment_holder/      # Segment 持有器
├── proxy_segment/       # 代理 Segment
├── snapshots/           # 分片快照
├── retrieve/            # 点检索
├── query/               # 查询处理
├── files/               # 文件管理
├── update.rs            # 更新操作
├── search.rs            # 搜索操作
├── scroll.rs            # 滚动遍历
├── count.rs             # 点计数
├── facet.rs             # 分面统计
├── optimize.rs          # 优化触发
└── wal.rs               # WAL 集成
```

### 4.4 WAL (`lib/wal`, 11 文件 / 3,589 行)

Qdrant 的写前日志是一个**自定义实现**（非通用 WAL 库），特点：

- **分段存储**：WAL 分为多个 Segment（默认每段 32MB），满后关闭并创建新段
- **追加写入**：只追加，不修改，保证写入性能
- **异步 flush**：关闭的段异步 flush 到磁盘
- **前缀截断**：支持 `prefix_truncate` 回收旧日志空间
- **段保留**：`retain_closed` 控制保留多少已关闭段（默认 1）
- **文件锁**：通过 `flock(2)` 防止多进程同时操作同一 WAL

核心结构：
```rust
pub struct Wal {
    open_segment: OpenSegment,        // 当前写入段
    closed_segments: Vec<ClosedSegment>, // 已关闭段
    creator: SegmentCreatorV2,        // 段创建器
    retain_closed: NonZeroUsize,      // 保留段数
    dir: File,                        // 目录文件锁
    path: PathBuf,                    // 路径
    flush: Option<JoinHandle<Result<()>>>, // 异步 flush 句柄
}
```

### 4.5 存储管理器 (`lib/storage`, 47 文件 / 13K 行)

```
lib/storage/src/
├── content_manager/
│   ├── toc/                 # TableOfContent - 存储目录
│   │   └── dispatcher.rs    # ToC 分发器
│   ├── consensus/           # 共识操作
│   │   ├── operation_sender.rs  # 操作发送器
│   │   └── persistent.rs    # 持久化共识状态
│   ├── consensus_manager.rs # 共识管理器
│   ├── collection_meta_ops.rs # Collection 元数据操作
│   ├── shard_distribution.rs # 分片分布
│   ├── snapshots/           # 存储快照
│   └── staging.rs           # 暂存操作
├── dispatcher.rs            # 全局 Dispatcher
└── rbac/                    # 基于角色的访问控制
```

#### TableOfContent (ToC)

ToC 是存储层的中央协调器：

- 管理所有 Collection 的生命周期
- 协调分片在节点间的分布
- 处理共识操作（分布式模式下）
- 管理更新和搜索请求的路由

#### Dispatcher

Dispatcher 是请求路由的入口点：

- **单节点模式**：直接转发到 ToC
- **分布式模式**：决定操作走 ToC 还是走 Consensus（需要共识的操作）

### 4.6 API 层 (`lib/api` + `src/actix` + `src/tonic`)

#### 4.6.1 REST API (`src/actix/`)

基于 actix-web 4.x，提供 16 个 API 模块：

| 模块 | 功能 |
|------|------|
| `collections_api` | Collection CRUD、配置管理 |
| `search_api` | 向量搜索（含过滤） |
| `recommend_api` | 推荐搜索 |
| `discover_api` | 发现搜索 |
| `query_api` | 统一查询接口 |
| `facet_api` | 分面统计 |
| `update_api` | 点的增删改 |
| `retrieve_api` | 点检索、滚动遍历 |
| `count_api` | 点计数 |
| `snapshot_api` | 快照管理 |
| `cluster_api` | 集群管理 |
| `shards_api` | 分片管理 |
| `service_api` | 服务信息、指标 |
| `debug_api` | 调试工具 |
| `profiler_api` | 性能分析 |
| `local_shard_api` | 本地分片操作 |
| `audit_api` | 审计日志 |
| `issues_api` | 问题报告 |
| `vector_name_api` | 向量命名管理 |

认证方式：
- **API Key**：通过 `api-key` HTTP 头
- **JWT RBAC**：通过 JWT token 实现细粒度权限控制
- 白名单路径：`/`, `/healthz`, `/readyz`, `/livez`

#### 4.6.2 gRPC API (`src/tonic/`)

使用 tonic 0.14（Qdrant 自定义 fork），对应 REST API 的全部功能，性能更高。Protobuf 定义在 `lib/api/src/grpc/`。

#### 4.6.3 数据模型 (`lib/api`)

- `lib/api/src/rest/` — REST 模型，通过 `schemars` 自动生成 OpenAPI 3.0 规范
- `lib/api/src/grpc/` — Protobuf 定义和 tonic 生成代码
- `lib/api/src/conversions/` — REST ↔ gRPC 类型转换

---

## 5. 分布式架构

### 5.1 Raft 共识

Qdrant 使用 TiKV 的 raft-rs 实现分布式共识。共识线程在独立操作系统中运行。

启动流程（`src/main.rs`）：

1. 解析 `--bootstrap` URI（从已有集群引导）
2. 加载持久化共识状态 `Persistent::load_or_init()`
3. 创建 `ConsensusManager` 管理共识状态
4. `Consensus::run()` 在独立线程中运行 Raft 状态机
5. 集群间通过 gRPC (端口 6335) 通信

共识操作通过 `OperationSender` 发送到 propose channel，由 Raft 状态机处理后应用到 ToC。

### 5.2 分片分布

- **自动分片**：创建 Collection 时自动分配 Shard 到各节点
- **手动分片**：通过 API 指定 Shard 分布
- **分片转移**：支持 `stream_records`、`snapshot`、`wal_delta` 三种方式
- **重分片**（resharding）：动态调整分片数量，零停机

### 5.3 副本与一致性

- **replication_factor**：每个 Shard 的副本数（默认 1）
- **write_consistency_factor**：写入成功的最小副本数（默认 1）
- 节点类型：Normal（读写）vs Listener（只写不读，用于备份）

---

## 6. 性能优化设计

### 6.1 内存管理

- **jemalloc**：默认内存分配器（非 msvc 平台），支持 stats/profiling/background_threads
- **mmap**：大向量数据通过 mmap 映射，减少内存占用
- **on_disk_payload**：payload 默认存储在磁盘（索引部分在内存）
- **低内存模式**：`low_memory_mode` 配置，启动时降低内存使用

### 6.2 并行计算

- **rayon**：数据并行（HNSW 构建、量化等）
- **tokio**：异步 I/O（网络请求、磁盘操作）
- **max_search_threads**：搜索线程数控制
- **optimizer_cpu_budget**：优化器 CPU 预算
- **max_workers**：actix-web 工作线程数

### 6.3 优化器

后台优化器持续运行，优化策略包括：

- **Segment 合并**：小 Segment 合并为大 Segment
- **垃圾回收**：删除已标记删除的向量（`deleted_threshold: 0.2`）
- **索引重建**：为达到阈值的 Segment 构建 HNSW 索引
- **量化应用**：对 Segment 应用量化配置

优化参数：
- `indexing_threshold_kb: 10000` — 超过此大小时构建索引
- `default_segment_number: 0` — 目标 Segment 数（0 = 自动）
- `flush_interval_sec: 5` — 强制 flush 间隔
- `max_optimization_threads: null` — 优化线程数（null = 无限制）

### 6.4 SIMD 加速

距离度量计算有针对不同 CPU 架构的 SIMD 优化：

- **AVX**（x86-64）：256-bit 向量运算
- **SSE**（x86 老架构）：128-bit 向量运算
- **NEON**（ARM）：128-bit 向量运算

运行时自动选择最优实现。

### 6.5 io_uring 支持

Linux 平台支持 `async_scorer`，利用 io_uring 进行异步 rescoring，进一步提升搜索吞吐。

### 6.6 GPU 加速

`lib/gpu/` 提供 HNSW 索引构建的 GPU 加速（feature flag `gpu`）。

---

## 7. 安全设计

### 7.1 传输安全

- **TLS**：REST 和 gRPC 均支持 TLS（rustls + ring）
- **P2P TLS**：节点间通信可加密
- **证书轮换**：`cert_ttl` 支持 HTTPS 证书热更新

### 7.2 认证与授权

- **API Key**：全量或只读 API Key
- **JWT RBAC**：细粒度角色访问控制
- **内部认证**：`enforce_internal_auth` 强制 P2P 通信认证
- **mTLS**：可选客户端证书验证

### 7.3 审计日志

- 记录所有访问检查的 API 请求
- 结构化 JSON 格式
- 支持日志轮转（daily）
- 可配置信任 X-Forwarded-For（反向代理场景）

### 7.4 SSRF 防护

- `enable_snapshot_url_recovery` 默认开启，可禁用远程 URL 快照恢复
- 防止用户通过快照恢复访问内部资源

---

## 8. 启动流程

`src/main.rs` 的 `main()` 函数执行以下步骤：

```
1. 安装 ring crypto provider
2. 解析 CLI 参数 (--bootstrap, --uri, --config-path 等)
3. 加载配置文件 (config/config.yaml)
4. 初始化 feature flags 和 low_memory_mode
5. 设置 tracing 日志和 panic hook
6. 初始化审计日志
7. [可选] 初始化 GPU 设备
8. 创建存储目录并检查文件系统兼容性
9. 加载/初始化 Raft 共识状态
10. [可选] 从快照恢复 Collection
11. 初始化 CPU/IO 资源预算
12. 创建 ChannelService (节点间通信)
13. 创建 TableOfContent (核心存储)
14. 加载所有已存储的 Collection
15. 创建 Dispatcher
16. [分布式模式] 启动 Raft 共识线程
17. 启动 Telemetry 上报
18. 启动 REST API 服务器 (actix-web, 端口 6333)
19. [可选] 启动 /metrics 服务器 (单独端口)
20. 启动 gRPC API 服务器 (tonic, 端口 6334)
21. [debug] 启动死锁检测器
22. 等待所有服务线程完成
```

---

## 9. CLI 工具

除了主服务 `qdrant`，还有 4 个调试工具（需要 `service_debug` feature）：

| 工具 | 功能 |
|------|------|
| `schema_generator` | 生成 JSON Schema |
| `wal_inspector` | WAL 内容检查 |
| `wal_pop` | WAL 弹出操作 |
| `segment_inspector` | Segment 内容检查 |

---

## 10. 编译与部署

### 10.1 编译 Profiles

| Profile | LTO | codegen-units | 用途 |
|---------|-----|---------------|------|
| `release` | fat | 1 | 生产发布（最大优化） |
| `ci` | false | default | CI（保留 debug-assertions） |
| `bench` | false | 256 | 基准测试（带 debug 信息） |
| `perf` | false | 256 | 性能测试（比 release 编译快） |
| `dev` | — | — | 开发（sha2 优化为 O3） |

### 10.2 部署方式

- **Docker**：`docker run -p 6333:6333 qdrant/qdrant`
- **DEB 包**：`cargo deb` 生成 .deb 包
- **二进制**：直接编译运行
- **Qdrant Cloud**：全托管云服务

### 10.3 Feature Flags

| Feature | 说明 |
|---------|------|
| `gpu` | GPU 加速索引构建 |
| `tracing` | 分布式追踪 |
| `console-subscriber` | tokio console 追踪 |
| `tracy` | Tracy profiler 集成 |
| `stacktrace` | rstack 死锁诊断 |
| `chaos-testing` | 混沌测试 |
| `data-consistency-check` | 数据一致性检查 |
| `staging` | 实验性功能 |
| `service_debug` | 调试工具和服务 |

---

## 11. 架构亮点与启示

### 11.1 设计优势

- **Rust 零成本抽象**：高性能的同时保持内存安全，无需 GC
- **Segment 不可变设计**：写入创建新段，读取并行查询，无锁竞争
- **分层量化**：三级压缩（Scalar/PQ/Binary），灵活权衡速度与精度
- **SIMD 自适应**：运行时自动选择 AVX/SSE/NEON
- **模块化 Workspace**：17 个独立 crate，编译可并行，职责清晰
- **双协议 API**：REST 易用 + gRPC 高性能
- **Raft 共识**：强一致性的分布式方案

### 11.2 技术挑战

- **编译时间**：360K 行 Rust + LTO fat，release 编译可能很慢
- **内存管理**：mmap + jemalloc 组合需要仔细调优
- **分布式复杂性**：Raft + 分片转移 + 副本一致性
- **HNSW 图一致性**：并发写入时图的正确性保证

### 11.3 开发者启示

- **types.rs 5,405 行**：Qdrant 的类型定义极其丰富，是理解数据模型的入口
- **配置驱动**：几乎所有行为可通过 `config.yaml` 调整
- **实验性功能分离**：staging feature flag 隔离不成熟功能
- **Clippy 严格**：18 条 Clippy lint 规则（workspace 级别），代码质量高

---

## 12. 关键文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/main.rs` | 816 | 服务入口、启动流程 |
| `src/actix/mod.rs` | 293 | REST API 路由配置 |
| `src/startup.rs` | 61 | 启动辅助函数 |
| `lib/segment/src/types.rs` | 5,405 | 核心类型定义 |
| `lib/segment/src/lib.rs` | 18 | Segment 模块入口 |
| `lib/wal/src/lib.rs` | 1,334 | WAL 实现 |
| `lib/storage/src/dispatcher.rs` | — | 请求路由 |
| `config/config.yaml` | 408 | 默认配置 |
| `Cargo.toml` | 438 | 项目元数据和依赖 |

---

*本文档基于 Qdrant v1.18.3 源码深度分析生成，涵盖 1,391 个 Rust 源文件、约 360,000 行代码。*

---

☕️ 制作不易，请我喝咖啡☕️关注我➕

![promotion](../res/promotion.jpg)