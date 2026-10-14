# 篇8 缓存生命周期：何时可读，何时淘汰，谁负责删除？

> 系列：《Mooncake KV Cache 源码解析》
> 版本锚点：`v0.3.13.post1`（commit `7197358`）
> 源码链接基座：`https://github.com/kvcache-ai/Mooncake/blob/v0.3.13.post1/`

## 8.1 具体问题

前几篇看到对象如何写入、读取。这一篇回答生命周期问题：

1. **可读**：对象何时可读，读时如何被保护不被删除？
2. **淘汰**：容量压力下谁被淘汰，淘汰策略如何选？
3. **删除**：谁负责删除，删除与读取/淘汰如何并发协调？

## 8.2 场景与版本

- **场景**：缓存有容量上限。读操作需要保护对象不被并发删除；容量不足时 Master 触发淘汰；客户端可显式删除。
- **版本**：`v0.3.13.post1`。注意：本版本淘汰逻辑在 `src/master_service.cpp`（`include/eviction_strategy.h` 提供 LRU/FIFO 策略），lease 在 `ObjectMetadata` 上（`include/master_service.h`）。
- **主路径**：读保护 `GetReplicaList`（`master_service.cpp:3632`）→ `GrantReadLease`；淘汰 `EvictObject`（`master_service.cpp:9724` 附近）→ `can_evict_replicas` → `try_evict_or_offload`；删除 `RemoveObject`（`master_service.cpp:6428`）。

## 8.3 调用链图

```
读保护：
GetReplicaList (master_service.cpp:3632)
   │  IsReplicaReadable 过滤可读副本
   │  GrantLeaseForGroup / GrantReadLease(default_kv_lease_ttl_)  ← 授予 lease
   ▼
lease 有效期内对象不被 Remove/Eviction

淘汰：
EvictObject (master_service.cpp:9724 附近)
   │  can_evict_replicas  ← 有可淘汰的内存副本
   │  IsLeaseExpired / IsHardPinned / IsSoftPinActive 过滤
   │  try_evict_or_offload  ← 淘汰或下沉 SSD
   ▼
PopReplicasWithCacheTotalAccounting → 释放内存 + 配额

删除：
RemoveObject (master_service.cpp:6428)
   │  !force && !IsLeaseExpired → 拒绝（LEASE_EXPIRED 保护）
   │  force → 强制删除
   ▼
EraseMetadata → 释放副本 + 配额
```

## 8.4 三个关键代码片段

### 片段1：读保护——GrantReadLease（`master_service.cpp:3682`）

`GetReplicaList` 在返回副本列表时授予 lease，注释明确说明目的——**防止客户端读取时对象被删除**：

```cpp
// master_service.cpp:3682  授予 lease 保护读取
// Grant a lease to the object so it will not be removed
// when the client is reading it.
auto* ts = accessor.GetTenantState();
if (ts) {
    GrantLeaseForGroup(*ts, key, metadata);   // ← 组对象：整组授予
} else {
    metadata.GrantReadLease(default_kv_lease_ttl_);   // ← 单对象
}
```

**关键点**：`default_kv_lease_ttl_`（默认 10s）是 lease 有效期。lease 有效期内，`Remove`/`Eviction` 会跳过该对象（见片段2/3 的 `IsLeaseExpired` 检查）。**lease 是读安全的基石**——只要 lease 未过期，读不会读到被删除的数据。组对象用 `GrantLeaseForGroup` 整组授予，保证组内成员一致。

### 片段2：淘汰过滤（`master_service.cpp:9724`）

淘汰前先过滤：只有**有可淘汰内存副本**、**lease 已过期**、**非 hard-pinned**、**非 soft-pin 活跃**的对象才可淘汰：

```cpp
// master_service.cpp:9724  可淘汰判定
auto can_evict_replicas = [&](const ObjectMetadata& metadata) {
    return metadata.HasReplica(is_evictable_memory_replica);
};

// master_service.cpp:10221  淘汰过滤条件
if (member_metadata.IsHardPinned() ||
    !member_metadata.IsLeaseExpired(now) ||
    (!allow_soft_pinned && IsSoftPinActive(member_metadata, now)) ||
    !can_evict_replicas(member_metadata)) {
    continue;   // ← 任一条件不满足，跳过
}
```

**关键点**：淘汰是**多条件过滤**——lease 未过期（正在被读）、hard-pinned（用户强制保留）、soft-pin 活跃（短期保护）、无可淘汰副本，任一命中都跳过。`allow_evict_soft_pinned_objects_` 配置决定 soft-pin 对象是否可被淘汰（`master_service.cpp:9921` 两轮淘汰：先不淘汰 soft-pin，空间仍不足才允许）。

### 片段3：删除保护（`master_service.cpp:6428`）

`RemoveObject` 默认拒绝删除 lease 未过期的对象，`force` 可绕过：

```cpp
// master_service.cpp:6428  删除前检查 lease
if (!force && !metadata.IsLeaseExpired()) {
    return tl::make_unexpected(ErrorCode::LEASE_EXPIRED);  // ← 正在被读，拒绝
}
```

**关键点**：`RemoveObject` 的 `force` 参数是**显式逃生舱**——默认拒绝删除正在被读的对象（返回 `LEASE_EXPIRED`），`force=true` 才强制删除。这保证"读与删除并发"时，读优先，除非调用方明确要求强制删除。

## 8.5 完成语义：lease 是生命周期的核心协调器

回答开篇问题，缓存生命周期由 **lease + pin + 淘汰策略** 三者协调：

1. **可读**：`PutEnd` 把副本置 `COMPLETE`（篇2），`GetReplicaList` 只返回 `IsReplicaReadable` 的副本（`master_service.cpp:3649`），并授予 lease 保护读取。
2. **淘汰**：容量压力触发 `EvictObject`，多条件过滤（lease 过期 + 非 pin + 有可淘汰副本）后淘汰。淘汰策略由 `eviction_strategy_`（LRU/FIFO，`eviction_strategy.h`）决定选哪个 key。
3. **删除**：`RemoveObject` 默认拒绝删除 lease 未过期的对象，`force` 强制删除。

**关键点**：lease 是**读与删除/淘汰的仲裁者**。读授予 lease，删除/淘汰检查 lease——lease 未过期则跳过。这就是"读安全"的机制保证：**只要 lease 有效，对象不会被并发删除或淘汰**。

## 8.6 异常与并发

### 删除与读取并发

`RemoveObject` 检查 `IsLeaseExpired`（`master_service.cpp:6428`）。若读已授予 lease，删除返回 `LEASE_EXPIRED` 拒绝；若删除先发生，`GetReplicaList` 返回 `OBJECT_NOT_FOUND`（`master_service.cpp:3642`）。**两者互斥，不会读到半删数据**。

### 淘汰中读取

淘汰前检查 `IsLeaseExpired`（`master_service.cpp:10222`）。若对象正在被读（lease 有效），淘汰跳过。若淘汰先发生，读的 `GetReplicaList` 返回 `OBJECT_NOT_FOUND` 或 `REPLICA_IS_NOT_READY`（`master_service.cpp:3667`）。

### double-erase

`EraseMetadata` 用 `QuotaEraseMode::kFull`（`master_service.cpp:10246`）清理。`RemoveObject` 对已删除对象返回 `OBJECT_NOT_FOUND`，幂等处理。测试 `master_service_processing_key_double_erase_test.cpp` 覆盖。

### 淘汰与 SSD 下沉

`try_evict_or_offload`（`master_service.cpp:9766`）在 `offload_on_evict_` 开启时，优先把对象**下沉到 SSD** 而非直接淘汰（见篇9）。`offload_force_evict_` 强制淘汰，`offload_cap` 限制下沉队列。

## 8.7 验证与结论

- **读保护**：`mooncake-store/src/master_service.cpp` 的 `GetReplicaList`(L3632)、`GrantReadLease`(L3688)、`GrantLeaseForGroup`(L3686)。
- **淘汰**：`mooncake-store/src/master_service.cpp` 的 `can_evict_replicas`(L9724)、`try_evict_or_offload`(L9766)、`evict_replicas`(L9730)。
- **删除**：`mooncake-store/src/master_service.cpp` 的 `RemoveObject`(L6428)。
- **策略**：`mooncake-store/include/eviction_strategy.h` 的 `LRUEvictionStrategy`(L43)、`FIFOEvictionStrategy`(L79)。
- **测试**：`mooncake-store/tests/master_service_evict_scenario_test.cpp`、`eviction_strategy_test.cpp`、`batch_remove_test.cpp`、`master_service_processing_key_double_erase_test.cpp`。

**结论**：缓存生命周期由 **lease + pin + 淘汰策略** 协调。读授予 lease 保护对象不被删除/淘汰；淘汰多条件过滤（lease 过期 + 非 pin + 有可淘汰副本）后由 LRU/FIFO 策略选 key；删除默认拒绝 lease 未过期的对象，`force` 强制。**lease 是读与删除/淘汰的仲裁者**，保证读安全。淘汰时可下沉 SSD（见篇9）。下一篇深入 SSD 分层。下一篇深入 SSD 分层。

---

*下一篇：篇9 SSD 分层——缓存如何下沉、加载与恢复？*
