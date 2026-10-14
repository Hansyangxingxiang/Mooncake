# 篇2 一次 Put：缓存对象如何从客户端写入分布式内存？

> 系列：《Mooncake KV Cache 源码解析》
> 版本锚点：`v0.3.13.post1`（commit `7197358`）
> 源码链接基座：`https://github.com/kvcache-ai/Mooncake/blob/v0.3.13.post1/`

## 2.1 具体问题

客户端 Put 返回成功时，另一个客户端是否已经可以读取？**可见性**是这一篇的核心。Mooncake 用两阶段 Put（`PutStart` → 写数据 → `PutEnd`）保证：`PutEnd` 之前对象不可读，防止脏读。

## 2.2 场景与版本

- **场景**：客户端 A 把一个 KV Cache 对象写入分布式内存，客户端 B 随后读取。Put 返回成功的那一刻，B 能否读到完整数据？
- **版本**：`v0.3.13.post1`。
- **主路径**：`Client::Put`（`client_service.cpp:1853`）→ `MasterService::PutStart`（`master_service.cpp:4292`）→ `TransferWrite`（`client_service.cpp:4479`）→ `DetermineFinalizeDecision`（`client_service.cpp:332`）→ `MasterService::PutEnd`（`master_service.cpp:4509`）。

## 2.3 调用链图

```
Client::Put (client_service.cpp:1853)
   │
   ├─ 1. master_client_.PutStart(key, slice_lengths, config)   ← 空间申请
   │      └─ MasterService::PutStart (master_service.cpp:4292)
   │            └─ AllocateAndInsertMetadata (master_service.cpp:4021)
   │                  └─ allocation_strategy_->Allocate(...)   ← 副本位置选择
   │                        └─ offset_allocator 分配 segment+offset
   │
   ├─ 2. TransferWrite(replica, slices)  (client_service.cpp:4479)  ← 数据路径
   │      └─ TransferData → TransferEngine::submitTransfer
   │            └─ 远端 Client 内存 (RDMA/TCP)
   │
   └─ 3. DetermineFinalizeDecision(config, transfer_summary)  ← 失败处理
         ├─ 成功 → master_client_.PutEnd(...)   ← 提交元数据，对象可见
         └─ 失败 → master_client_.PutRevoke(...) ← 撤销，对象不可见
```

## 2.4 三个关键代码片段

### 片段1：Client::Put 的主流程（`client_service.cpp:1853`）

Put 的核心是"先申请空间，再写数据，最后提交"。注意 `PutStart` 返回的 `replica_list` 就是 Master 分配好的副本位置：

```cpp
// client_service.cpp:1882  ① 空间申请
auto start_result = master_client_.PutStart(key, slice_lengths, client_cfg);
if (!start_result) {
    if (err == ErrorCode::OBJECT_ALREADY_EXISTS) return {};  // 已存在视为成功
    return tl::unexpected(err);
}

// client_service.cpp:1922  ② 数据路径：对每个内存/NoF 副本做传输
for (const auto& replica : start_result.value()) {
    if (replica.is_memory_replica() || replica.is_nof_replica()) {
        ErrorCode transfer_err = TransferWrite(replica, slices);
        if (transfer_err != ErrorCode::OK) {
            transfer_summary.RecordFailure(replica_type, transfer_err);
            continue;   // 部分副本失败，继续尝试其他副本
        }
        transfer_summary.RecordSuccess(replica_type);
    }
}

// client_service.cpp:1967  ③ 失败处理 + 提交
const auto finalize_decision = DetermineFinalizeDecision(config, transfer_summary);
if (finalize_decision.end_type.has_value()) {
    auto end_result = master_client_.PutEnd(ObjectMeta{key, object_checksum},
                                            *finalize_decision.end_type);
    ...
}
if (finalize_decision.revoke_type.has_value()) {
    auto revoke_result = master_client_.PutRevoke(key, *finalize_decision.revoke_type);
    ...
}
if (!finalize_decision.success) return tl::unexpected(finalize_decision.error);
return {};
```

### 片段2：Master 的 PutEnd 让对象可见（`master_service.cpp:4509`）

`PutEnd` 把 `PROCESSING` 状态的副本标记为 `COMPLETE`，这一步就是**可见性**的开关：

```cpp
// master_service.cpp:4574  把 processing 副本标记为 complete
metadata.VisitReplicas(
    [&is_target_replica](const Replica& replica) {
        return replica.is_processing() && is_target_replica(replica);
    },
    [this, &key, &metadata, &completed_pending_replica](Replica& replica) {
        if (replica.is_processing() && metadata.PendingSoftPinOwnsReplica(replica.id())) {
            completed_pending_replica = true;
        }
        replica.mark_complete();   // ← 副本变为 COMPLETE，对象可读
        ...
    });
```

`PutEnd` 还校验：只有发起 `PutStart` 的 client 才能 `PutEnd`（`client_id != metadata.client_id` 返回 `ILLEGAL_CLIENT`，`master_service.cpp:4522`），防止别的客户端篡改。

### 片段3：副本位置选择（`master_service.cpp:4098`）

Master 通过 `allocation_strategy_->Allocate` 决定副本放在哪些 segment。分配策略保证**同一对象的每个 slice 放在不同 segment**（best-effort）：

```cpp
// master_service.cpp:4098  内存副本分配
auto allocation_result = allocation_strategy_->Allocate(
    allocator_access, value_length, config.replica_num,
    preferred_segments, std::set<std::string>(), ReplicaType::MEMORY);
if (!allocation_result.has_value()) {
    if (allocation_result.error() == ErrorCode::INVALID_PARAMS) {
        return tl::make_unexpected(ErrorCode::INVALID_PARAMS);
    }
    need_mem_eviction_ = true;   // 空间不足，触发淘汰
    return tl::make_unexpected(ErrorCode::NO_AVAILABLE_HANDLE);
}
```

## 2.5 可见性语义：Put 返回成功 = 对象可读

回答开篇问题：**Put 返回成功时，另一个客户端已经可以读取**。原因：

1. `Client::Put` 只有在 `DetermineFinalizeDecision` 判定 `success=true` 且 `PutEnd` 成功后才返回 `{}`（成功）。
2. `PutEnd` 把副本标记为 `COMPLETE`，此后 `GetReplicaList` 才会返回该副本，其他客户端才能读到。
3. 若 `PutEnd` 失败，`Client::Put` 返回错误，调用方知道 Put 未完成。

**关键点**：网络传输完成 ≠ 对象写入成功。`TransferWrite` 只是把数据搬到远端内存，真正让对象"可见"的是 Master 的 `PutEnd`。这就是两阶段 Put 的意义——防止其他客户端读到半写状态。

## 2.6 异常与失败路径

### 部分副本失败（`DetermineFinalizeDecision`，`client_service.cpp:332`）

Put 是 best-effort 的：不是所有副本都必须成功。`DetermineFinalizeDecision` 根据 `transfer_summary` 决定最终动作：

- **默认模式**：所有分配的副本都传输成功 → `PutEnd(ALL)`，成功。否则 → `PutRevoke(ALL)`，失败。
- **FLEXIBLE_DUAL_REPLICA 模式**（内存 + NoF 双副本）：
  - 内存和 NoF 都成功 → `PutEnd(ALL)`，成功。
  - 只有内存成功 → `PutEnd(MEMORY)` + `PutRevoke(NOF_SSD)`，**仍算成功**。
  - 只有 NoF 成功 → `PutEnd(NOF_SSD)` + `PutRevoke(MEMORY)`，仍算成功。
  - 都失败 → `PutRevoke(ALL)`，失败。

**重试是否安全？** `PutEnd` 对重试做了幂等处理：若所有目标副本已 `COMPLETE`，重复 `PutEnd` 视为 no-op（`master_service.cpp:4551`）。但 `PutStart` 对已存在对象返回 `OBJECT_ALREADY_EXISTS`，`Client::Put` 把它当作成功返回（`client_service.cpp:1885`）——这意味着**重复 Put 同一 key 不会覆盖**，而是静默成功。这是 KV cache 的语义：对象不可变，重复写同一 key 是幂等的。

### 空间不足

`PutStart` 分配失败返回 `NO_AVAILABLE_HANDLE`，Master 置 `need_mem_eviction_ = true` 触发淘汰（`master_service.cpp:4112`）。`Client::Put` 收到该错误后返回失败，不重试（重试由上层决定）。

### 僵尸对象

若 client 在 `PutStart` 后崩溃、未发 `PutEnd`/`PutRevoke`，对象进入"zombie"状态。Master 用 `put_start_discard_timeout`（默认 30s）允许后续 `PutStart` 抢占，用 `put_start_release_timeout`（默认 10min）回收空间（见 `mooncake-store.md` 的 Zombie Object Cleanup 章节）。

## 2.7 验证与结论

- **Put 主流程**：`mooncake-store/src/client_service.cpp` 的 `Client::Put`(L1853)、`TransferWrite`(L4479)、`DetermineFinalizeDecision`(L332)。
- **Master 侧**：`mooncake-store/src/master_service.cpp` 的 `PutStart`(L4292)、`PutEnd`(L4509)、`AllocateAndInsertMetadata`(L4021)。
- **测试**：`mooncake-store/tests/client_integration_test.cpp`（Put/Get 集成）、`master_service_placement_scenario_test.cpp`（副本放置）、`task_integration_test.cpp`（传输任务）。
- **设计文档**：`docs/source/design/store/mooncake-store.md` 的 PutStart/PutEnd 说明、Zombie Object Cleanup、AllocationStrategy。

**结论**：Put 是两阶段操作。`PutStart` 从 Master 申请空间并选副本，`TransferWrite` 把数据搬到远端内存，`PutEnd` 把副本标记为 `COMPLETE` 使对象可见。**Put 返回成功 = 数据已落位 + Master 已提交 = 其他客户端可读**。部分副本失败时 best-effort 处理，`PutEnd` 幂等，重复 Put 同一 key 静默成功。下一篇看 Get 如何找到并读回。

---

*下一篇：篇3 一次 Get——如何找到远端缓存并读回数据？*
