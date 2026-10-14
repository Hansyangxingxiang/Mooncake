# 篇3 一次 Get：如何找到远端缓存并读回数据？

> 系列：《Mooncake KV Cache 源码解析》
> 版本锚点：`v0.3.13.post1`（commit `7197358`）
> 源码链接基座：`https://github.com/kvcache-ai/Mooncake/blob/v0.3.13.post1/`

## 3.1 具体问题

Get 如何定位副本、选择读取路径、判断完成？这一篇回答三个问题：

1. **定位**：如何从 key 找到对象的所有副本位置？
2. **选择**：多个副本（可能在不同节点）时，选哪个读？
3. **完成**：如何判断读取完成，错误如何返回？

## 3.2 场景与版本

- **场景**：客户端 B 读取客户端 A 写入的 KV Cache 对象。对象可能有多个副本，分布在不同的存储节点。
- **版本**：`v0.3.13.post1`。注意：本版本中副本选择逻辑在 `include/replica_selection.h`（`src/replica_selection.cpp` 已不存在）。
- **主路径**：`get_into`（`real_client.cpp:3758`）→ `get_into_range_internal` → `execute_ranged_read`（`real_client.cpp:3475`）→ `batch_query`（`real_client.cpp:4033`）→ `Client::BatchQuery`（`client_service.cpp:1230`）→ `SelectBestReplica`（`replica_selection.h:123`）。

## 3.3 调用链图

```
get_into (real_client.cpp:3758)
   │
   └─ get_into_range_internal
        │
        ├─ 1. batch_query (real_client.cpp:4033)   ← 元数据 + lease
        │      └─ Client::BatchQuery (client_service.cpp:1230)
        │            └─ master_client_.BatchGetReplicaList  ← Master RPC
        │                  └─ 返回 replicas + lease_ttl_ms
        │
        ├─ 2. build_ranged_read_metadata_from_query_result (real_client.cpp:4043)
        │      └─ SelectBestReplica (replica_selection.h:123)  ← 副本选择
        │
        └─ 3. execute_ranged_read (real_client.cpp:3475)   ← 数据路径
               └─ client_->Get(key, filtered_qr, slices)  ← TE 读取
                     └─ 远端 Client 内存 (RDMA/TCP)
```

## 3.4 三个关键代码片段

### 片段1：batch_query 拿元数据 + lease（`client_service.cpp:1230`）

`BatchQuery` 调用 Master 的 `BatchGetReplicaList`，返回副本列表，并**附带 lease**。lease 是 Get 安全读的关键——lease 有效期内对象不会被 Remove/Eviction：

```cpp
// client_service.cpp:1234  调用 Master RPC
auto response = master_client_.BatchGetReplicaList(object_keys, tenant_id);

// client_service.cpp:1251  构造 QueryResult，携带 lease 过期时间
for (size_t i = 0; i < response.size(); ++i) {
    if (response[i]) {
        results.emplace_back(QueryResult(
            std::move(response[i].value().replicas),
            start_time + std::chrono::milliseconds(response[i].value().lease_ttl_ms),
            response[i].value().object_checksum));
    } else {
        results.emplace_back(tl::unexpected(response[i].error()));
    }
}
```

`QueryResult` 里的 `lease_ttl_ms` 就是 lease 的剩余时间。**lease 过期前，对象受保护**；过期后，对象可能被删除/淘汰，此时读会失败（见篇8）。

### 片段2：副本选择（`replica_selection.h:123`）

`SelectBestReplica` 按优先级选择副本：**本地 MEMORY → 本地 NOF_SSD → 远端 MEMORY（最优评分）→ 远端 NOF_SSD → LOCAL_DISK → DFS → DISK**。只考虑 `COMPLETE` 状态的副本：

```cpp
// replica_selection.h:128  只选 COMPLETE 副本
for (const auto &r : replicas) {
    if (r.status != ReplicaStatus::COMPLETE) continue;
    if (r.is_memory_replica()) {
        if (local_endpoints.count(r.get_memory_descriptor()
                                      .buffer_descriptor.transport_endpoint_)) {
            return &r;  // 本地 MEMORY — 最优，零网络开销
        }
        if (!first_memory) first_memory = &r;
    } else if (r.is_nof_replica()) {
        if (local_endpoints.count(r.get_nof_descriptor()
                                      .buffer_descriptor.transport_endpoint_)) {
            return &r;  // 本地 NOF_SSD — 也很好
        }
        if (!first_nof) first_nof = &r;
    }
}
// 无本地副本：远端 MEMORY 中选最优评分（issue #2516）
if (first_memory && RemoteReplicaScoringEnabled()) {
    if (const auto *scored = PickBestRemoteMemory(replicas, local_endpoints))
        return scored;
}
if (first_memory) return first_memory;
if (first_nof) return first_nof;
// 兜底：LOCAL_DISK > DFS > DISK
```

**本地优先**是核心优化：如果对象副本就在本机内存，直接 memcpy，零网络开销。

### 片段3：execute_ranged_read 读数据（`real_client.cpp:3475`）

选好副本后，`execute_ranged_read` 校验 buffer 大小，然后走 `client_->Get` 通过 TE 读回数据：

```cpp
// real_client.cpp:3483  校验 buffer 容量
if (size_is_buffer_capacity) {
    if (size < total_size) {
        return tl::unexpected(ErrorCode::INVALID_PARAMS);  // buffer 太小
    }
    size = total_size;
}

// real_client.cpp:3539  内存副本：走 client_->Get（TE 读取）
auto get_result = client_->Get(key, filtered_qr, tmp_slices);
if (!get_result) {
    return tl::unexpected(get_result.error());
}
```

## 3.5 完成判断

- **成功**：`get_into` 返回读取的字节数（`int64_t`，正数）。数据已完整读入本地 buffer。
- **失败**：返回负值错误码。`execute_ranged_read` 对 buffer 太小返回 `INVALID_PARAMS`，对 `client_->Get` 失败透传错误。

## 3.6 异常与失败路径

### LEASE_EXPIRED

`build_ranged_read_metadata_from_query_result`（`real_client.cpp:4043`）对 `OBJECT_NOT_FOUND`/`REPLICA_IS_NOT_READY` 直接透传。若 lease 在读取完成前过期，对象可能被删除，读会失败——这是**防止读到被删除数据**的保护（见 `mooncake-store.md` 的 Lease 章节）。

### 副本不可达

`SelectBestReplica` 找不到任何 `COMPLETE` 副本时返回 `nullptr`，`build_ranged_read_metadata_from_query_result` 返回 `INVALID_REPLICA`（`real_client.cpp:4066`）。Get 失败，不重试（重试由上层决定）。

### 读取与删除并发

lease 机制协调：`GetReplicaList` 成功授予 lease，lease 有效期内对象不被 `Remove`/`Eviction`。因此**只要 lease 未过期，读是安全的**。若删除先于读发生，`BatchGetReplicaList` 返回 `OBJECT_NOT_FOUND`，读失败。

## 3.7 验证与结论

- **Get 主流程**：`mooncake-store/src/real_client.cpp` 的 `get_into`(L3758)、`execute_ranged_read`(L3475)、`batch_query`(L4033)、`build_ranged_read_metadata_from_query_result`(L4043)。
- **副本选择**：`mooncake-store/include/replica_selection.h` 的 `SelectBestReplica`(L123)。
- **Master RPC**：`mooncake-store/src/client_service.cpp` 的 `Client::BatchQuery`(L1230)。
- **测试**：`mooncake-store/tests/client_integration_test.cpp`、`master_service_scenario_test.cpp`、`replica_selection_test.cpp`（覆盖本地优先、评分选择、无可用副本返回 nullptr）。

**结论**：Get 分三步——`batch_query` 从 Master 拿副本列表 + lease，`SelectBestReplica` 按"本地优先"选副本，`execute_ranged_read` 通过 TE 读回数据。**本地 MEMORY 副本零网络开销**；lease 保证读安全；`COMPLETE` 状态过滤保证只读完整数据。下一篇深入核心数据结构。

---

*下一篇：篇4 缓存对象如何组织？读懂核心数据结构*
