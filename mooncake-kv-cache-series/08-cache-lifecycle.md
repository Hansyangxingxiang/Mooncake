# 篇8 缓存生命周期：何时可读，何时淘汰，谁负责删除？

> 系列：《Mooncake KV Cache 源码解析》
> 版本锚点：`v0.3.13.post1`（commit `7197358`）
> 源码链接基座：`https://github.com/kvcache-ai/Mooncake/blob/v0.3.13.post1/`

## 8.1 具体问题

前几篇看到对象如何写入、读取。这一篇把"生命周期"拆成四个动作，回答它们分别由谁触发：

1. **保护**：客户端拿到副本后，什么机制保证读的过程中对象不被删掉？
2. **淘汰**：容量压力下，Master 按什么顺序挑牺牲者？
3. **删除**：谁有权删除对象，`force` 到底绕过了什么？
4. **等价性**：淘汰一个副本，等于删除一个对象吗？

有两个容易混淆的点，先说清楚。第一，**lease 不是数据有效期，而是"读窗口"**：它只承诺"在这段时间内不主动删"，不承诺数据永远正确。第二，**淘汰和删除不是同一件事**：淘汰移除的是 MEMORY 副本，对象可能因为还有 LOCAL_DISK/DFS 副本而继续存在；删除移除的是元数据。把两者混为一谈，就会误判"内存降了但对象还能读"。

> **版本提示**：本版本没有 `lease.h`，lease 字段和方法都在 `master_service.h` 的 `ObjectMetadata` 里；也没有 `EvictObject` 函数——它只出现在测试用例名里，容量淘汰的入口是 `BatchEvict`。另外 `include/eviction_strategy.h` 虽然定义了 LRU/FIFO 策略类，但 `master_service.cpp` 在本 tag **并未引用 `eviction_strategy_`**（只留了前置声明）。按旧文件名或旧函数名找源码会一无所获。

## 8.2 场景与版本

- **场景**：Store 集群内存有上限。读请求需要一段稳定的读取窗口；写入失败时 Master 需要腾空间；客户端也可能显式删除。三个动作并发发生。
- **版本**：`v0.3.13.post1`。lease 在 `src/master_service.cpp` + `include/master_service.h`；容量淘汰在 `src/master_service.cpp` 的 `BatchEvict` / `NoFBatchEvict`，由 `EvictionThreadFunc` 驱动；删除在 `Remove` / `BatchRemove`。
- **默认参数**（`mooncake-store/conf/master.json:10`–`:15`）：`default_kv_lease_ttl=10000`（10s）、`default_kv_soft_pin_ttl=1800000`（30min）、`max_kv_soft_pin_ttl=86400000`（24h）、`allow_evict_soft_pinned_objects=true`、`eviction_ratio=0.1`、`eviction_high_watermark_ratio=1.0`。
- **主路径**：读保护 `GetReplicaList`（`master_service.cpp:3626`）→ `GrantLeaseForGroup`（`:3686`）/ `GrantReadLease`（`:3688`）；淘汰 `EvictionThreadFunc`（`:9053`）→ `BatchEvict`（`:9954`）→ `try_evict_group_or_object`（`:10179`）→ `EraseMetadata`（`:10550`）；删除 `Remove`（`:6416`）→ lease 检查（`:6428`）→ `accessor.Erase()`（`:6476`）。

## 8.3 调用链图

```
读保护
GetReplicaList (master_service.cpp:3626)
   ├─ VisitReplicas(IsReplicaReadable)               :3648
   ├─ 空列表 → OBJECT_NOT_FOUND :3664 / REPLICA_IS_NOT_READY :3667
   └─ GrantLeaseForGroup :3686 或 GrantReadLease :3688
        └─ ObjectMetadata::GrantReadLease (master_service.h:1322)
             lease_timeout = max(lease_timeout, now + ttl)

淘汰
EvictionThreadFunc (master_service.cpp:9053)   ← 每 10ms 检查
   ├─ used_ratio > high watermark（默认 1.0）        :9061–9062
   ├─ 或 need_mem_eviction_（PutStart 分配失败置位）  :4112 / :4169
   ▼
BatchEvict (master_service.cpp:9954)
   ├─ census：逐 shard 收集可淘汰对象               :10311–10325
   ├─ nth_element 按 lease_timeout 取最老的一批      :10503
   ├─ 第二阶段按 key 重查 + 重新校验                 :10530–10533
   ├─ try_evict_group_or_object                      :10179
   │    ├─ 组对象：任一成员 lease 未过期则整组跳过    :10203–10208
   │    └─ try_evict_or_offload                      :10024
   ├─ evict_replicas：只弹出 MEMORY 且 refcnt==0     :9973
   └─ 元数据已无有效副本 → EraseMetadata             :10550

删除
Remove (master_service.cpp:6416)
   ├─ !force && !IsLeaseExpired → OBJECT_HAS_LEASE   :6428–6431
   ├─ !AllReplicas(completed) → REPLICA_IS_NOT_READY :6439
   └─ PublishKvRemoved → accessor.Erase()            :6475–6476

BatchRemove (master_service.cpp:6767)
   └─ 同样的 lease / 完成态检查                      :6852 / :6858
```

## 8.4 三个关键代码片段

### 片段1：读保护——lease 只延长，不缩短（`master_service.h:1322`）

`GrantReadLease` 用 `std::max` 而不是直接赋值；组对象还有一层"半 TTL 才刷新"的节流：

```cpp
// master_service.h:1322
void GrantReadLease(const uint64_t ttl) const {
    SpinLocker locker(&lock);
    std::chrono::system_clock::time_point now =
        std::chrono::system_clock::now();
    lease_timeout =
        std::max(lease_timeout, now + std::chrono::milliseconds(ttl));
}

// master_service.h:1330
bool NeedsReadLeaseRefresh(const uint64_t ttl) const {
    SpinLocker locker(&lock);
    const auto now = std::chrono::system_clock::now();
    return lease_timeout <= now + std::chrono::milliseconds(ttl / 2);
}
```

```cpp
// master_service.cpp:2456  GrantLeaseForGroup
if (!metadata.IsGrouped()) {
    metadata.GrantReadLease(default_kv_lease_ttl_);
    return;
}
bool needs_refresh = metadata.NeedsReadLeaseRefresh(default_kv_lease_ttl_);
if (!needs_refresh) {
    // 再查 groups_needing_lease_refresh_，仍然不需要就原样返回
    ...
    return;
}
for (const auto& member_key : group_it->second) {
    auto mit = tenant_state.metadata.find(member_key);
    if (mit != tenant_state.metadata.end())
        mit->second.GrantReadLease(default_kv_lease_ttl_);
}
```

**关键点**：`max` 保证并发读不会把别人的 lease 缩短——两个线程同时读，先到的把 deadline 推到 `now+ttl`，后到的若时钟略早也不会写回更早的值。组对象每次读都刷新**整组**，但只有剩余 lease 不足半个 TTL 时才做，避免一次读触发组成员的全量 lease 更新。**分析（推断）**：这是把"组一致性"从"每次读都同步"降级为"到期前批量续租"，用最多半个 TTL 的窗口换取读路径的 O(1)。

### 片段2：淘汰候选——lease 过期、没人用、不是 pin（`master_service.cpp:10311`）

淘汰不是遍历所有对象随便挑，而是先做一次 census，只把"lease 已过期 + 有可淘汰副本 + 非 hard pin"的对象放进候选池；soft pin 对象进另一个桶：

```cpp
// master_service.cpp:10311  census 过滤
if (it->second.IsHardPinned()) continue;
bool has_evictable = can_evict_replicas(it->second);
if (has_evictable) shard_evictable_count++;
if (!it->second.IsLeaseExpired(now) || !has_evictable)
    continue;
if (!IsSoftPinActive(it->second, now)) {
    local_no_pin[t].push_back(it->second.lease_timeout);
} else if (allow_evict_soft_pinned_objects_) {
    local_soft_pin[t].push_back(it->second.lease_timeout);
}
```

```cpp
// master_service.cpp:10503  按 lease_timeout 取最老的一批
std::nth_element(candidates.begin(),
                 candidates.begin() + (evict_num - 1), candidates.end(),
                 [](const Candidate& a, const Candidate& b) {
                     return a.lease_timeout < b.lease_timeout;
                 });
```

**关键点**：可淘汰副本由 `IsEvictableMemoryReplica`（`:3570`）定义——**MEMORY 类型 + 可读 + `refcnt==0`**。`refcnt` 非零表示该副本正被后台任务（offload/copy/promotion）当作源副本钉住，不能动；**读并不增加 `refcnt`**，读的保护完全靠 lease。排序键是 `lease_timeout`：读会延长它，所以它同时充当"最近一次读的时间 + TTL"，按它升序就是先淘汰最久没被读的对象；从未被读过的对象其 `lease_timeout` 是 epoch（`master_service.h:1074` 的默认构造），天然排在最前。**因此本版本不需要 `eviction_strategy_`——LRU 语义由 lease 时间戳自然产生**。soft pin 是另一条时间线（写入时承诺的保护期），第一轮不碰；只有第一轮没达到下界，第二轮才在 `allow_evict_soft_pinned_objects_` 允许时处理（`:10660`–`:10663`）。

### 片段3：删除保护——拒绝码是 `OBJECT_HAS_LEASE`（`master_service.cpp:6428`）

删除的入口检查与淘汰共用同一个 lease 判据，但返回码和语义不同：

```cpp
// master_service.cpp:6428
if (!force && !metadata.IsLeaseExpired()) {
    VLOG(1) << "key=" << key << ", error=object_has_lease";
    return tl::make_unexpected(ErrorCode::OBJECT_HAS_LEASE);
}

// :6439  force 也不绕过"副本必须全部完成"的检查
if (!metadata.AllReplicas(&Replica::fn_is_completed)) {
    LOG(ERROR) << "key=" << key << ", error=replica_not_ready";
    return tl::make_unexpected(ErrorCode::REPLICA_IS_NOT_READY);
}
```

**关键点**：`force` 只绕过 lease，不绕过完成态与复制任务检查。源码注释解释了原因（`:6433`–`:6437`）——Put/Copy/Move 与 Remove 可能并发，直接删除半写对象极其危险。`BatchRemove` 在 `:6852` 做同样的 lease 判断，单个 key 失败只影响该 key 的结果项。

## 8.5 lease 不是锁：它只是一段被承诺的读窗口

`GetReplicaList` 返回副本描述符时授予 lease（`:3682`–`:3689`），客户端拿到的 `lease_ttl_ms` 由 `QueryResult` 换算成本地 `steady_clock` 的绝对时间（篇3）。之后：

- **删除/淘汰**在 Master 侧检查 `IsLeaseExpired()`，未过期就跳过；
- **客户端**在数据传输完成后才检查 lease 是否过期（篇3 的 `LEASE_EXPIRED`）。

所以 lease 不是互斥锁，而是"时间窗内的不删除承诺"。窗口默认 10 秒；超过 10 秒的慢读可能传输成功但被判定过期，调用方应把它当可重试错误，而不是数据损坏。**观点**：对 KV Cache 这种"读失败可以重算"的负载，这个取舍是合理的——用 10 秒窗口覆盖正常读延迟，避免为极慢读引入引用计数式的强同步。

`hard_pinned` 是另一层保护，创建时固定、不可变（`master_service.h:1113`），淘汰 census 第一步就跳过（`:10311`）。soft pin 则是一个可过期的写入承诺（`master_service.h:1106`）：默认动作是 `PRESERVE`（不设置，`replica.h:108`），只有写入方显式 `ENABLE` 才生效，未指定 TTL 时取默认 30 分钟（`master_service.cpp:2355`）。它与读 lease **相互独立**——读不会延长 soft pin（`master_service.h:1319` 注释），soft pin 也不等于不可淘汰，只是让对象进入第二轮候选。

## 8.6 淘汰：触发、目标与两阶段执行

`EvictionThreadFunc` 每 10ms 醒一次（`kEvictionThreadSleepMs`，`master_service.h:2129`），触发条件是：

```cpp
// master_service.cpp:9061
if (used_ratio > eviction_high_watermark_ratio_ ||
    (need_mem_eviction_ && eviction_ratio_ > 0.0)) { ... }
```

这里有个反直觉的事实：默认 `eviction_high_watermark_ratio=1.0`，而正常情况下 `used_ratio = used_bytes / capacity_bytes ≤ 1.0`（`include/storage_usage.h:14`），所以**默认配置下水位线触发几乎不会发生**；真正让淘汰跑起来的是 `need_mem_eviction_`——`PutStart` 分配副本失败且判断总容量足够时置位（`:4112`、`:4169`）。**分析（推断）**：这是"按需淘汰"而非"后台提前淘汰"，避免缓存还没真正吃满就主动丢数据；代价是写入路径先吃一次 `NO_AVAILABLE_HANDLE`，靠客户端重试等到空间释放。

一次 `BatchEvict` 的流程可以概括为：

1. **目标量**：`evict_ratio_target = max(ratio, used - high + ratio)`，下界 `max(target/2, used - high)`（`:9068`–`:9073`）；`ideal_evict_num = ceil(eviction_base * target)`（`:10388`），单位是**对象数**，不是字节。
2. **并行 census**：按 shard 多线程扫描，只收集时间戳/计数，不持有全局锁（`:10295`–`:10325`）。
3. **取前沿**：用 `nth_element` 按 `lease_timeout` 找第 `evict_num` 老的时间戳作为 cutoff，只物化 cutoff 以内的候选；候选带 `shard_idx + tenant_id + key` 而非迭代器，因为两阶段之间迭代器会失效（`:10252`–`:10258`）。
4. **串行驱逐**：逐个 key 重新查表、重新校验 lease/soft pin/可淘汰性（`:10530`–`:10533`），通过后调 `try_evict_group_or_object`。census 与驱逐之间状态会变，重校验是必须的。
5. **第二轮**：若第一轮没达到下界，先 `ReleaseExpiredDiscardedReplicas`（`:10586`），再按 `evict_ratio_lowerbound` 补一轮，必要时才动 soft pin 对象（`:10660`–`:10663`）。

`NoFBatchEvict`（`:10821`）是 NOF/SSD 段对应的同类路径；`EvictTenantMemoryForQuota`（`:9710`）是租户配额超限时的旁路，复用 `can_evict_replicas`（`:9724`）和 `try_evict_or_offload`（`:9766`），但触发条件不是全局水位，而是租户配额。

## 8.7 淘汰 ≠ 删除：副本、对象与 EraseMetadata

淘汰只动 MEMORY 副本。`evict_replicas`（`:9973`）通过 `PopReplicasWithCacheTotalAccounting` 弹出 `refcnt==0` 的 MEMORY 副本并释放内存，但**只要对象还有一个 LOCAL_DISK/DFS 副本，`metadata.IsValid()` 仍为真**，元数据不会被删。只有最后一个有效副本也没了，才在 `:10549`–`:10552` 调 `EraseMetadata`。

`try_evict_or_offload`（`:10024`）在此之上加了 SSD 下沉：开启 `offload_on_evict_` 且对象还没有 LOCAL_DISK 副本时，先从可淘汰的 MEMORY 副本里挑一个推到卸载队列并 `inc_refcnt` 钉住，再淘汰其余冗余内存副本。**如果入队失败且没有 `offload_force_evict_`，它返回 0、本轮跳过**（`:10080`–`:10088`）——这是"宁可少腾空间，也不静默丢数据"的默认语义。下沉细节见篇9。

`EraseMetadata`（`:2111`）是所有删除路径的共同出口，做的远不止 erase map：

- 取消未完成的 offload 任务并 `dec_refcnt`，同时清掉 SSD 侧邮箱项，避免孤儿任务；
- 清 `processing_keys`、`replication_tasks`（含引用计数与配额回滚）、promotion 任务；
- 归还 LOCAL_DISK 用量、释放 DFS 副本、更新 cache total 计数；
- 清 soft pin 截止时间索引；
- 按 `QuotaEraseMode` 归还租户配额。

**关键点**：删除的语义责任集中在 `EraseMetadata`，而不是分散在各调用点。`accessor.Erase()` 内部也走它（`:3417`、`:3529` 的注释），所以 `Remove` 只需要做前置校验。

那么"谁负责删除"？有三方：

1. **客户端**：`Remove` / `BatchRemove`（`real_client.h` 的 `remove` / `batch_remove`）；
2. **Master 淘汰线程**：`BatchEvict` 在最后一份副本消失后删元数据；
3. **Master 的僵尸清理**：`DiscardExpiredProcessingReplicas`（`:9127`）在 `put_start_time + put_start_release_timeout_sec_` 超时后（`:9167`）丢弃未完成的 PROCESSING 副本。

三方都汇入 `EraseMetadata`，这是"一个出口"的设计。

## 8.8 异常与并发

### 删除与读取并发

读先：lease 有效，`Remove` 返回 `OBJECT_HAS_LEASE`（`:6430`），删除方需等或显式 `force`。删除先：`GetReplicaList` 找不到对象，返回 `OBJECT_NOT_FOUND`（`:3664`）或副本未就绪（`:3667`）。由于 `GetReplicaList` 授 lease 与返回描述符在同一个 RO 访问器内完成，不存在"读完才授 lease"的窗口。**但 lease 到期后传输仍在进行**的情况存在，这正是客户端在传输后检查 lease 的原因。

### 淘汰与读取并发

淘汰 census 与驱逐是两阶段，期间对象可能被读、被写、被删。驱逐阶段按 key 重查并重新校验（`:10530`–`:10533`），校验不过就跳过，不会误删刚被读的对象。`refcnt==0` 则保证不会把正被 offload/复制/提升任务当作源副本的对象抽走。

### double-erase

`processing_keys` 与 metadata 的清理是两条路径：僵尸清理和显式删除都可能撞上同一个 key。`DiscardExpiredProcessingReplicas` 在 key 已不在 metadata 时先 `erase` 并打日志（`:9140`–`:9143`），`EraseMetadata` 自身对 `offloading_tasks`、`replication_tasks` 做存在性检查后再清理（`:2123`–`:2148`）。测试 `master_service_processing_key_double_erase_test.cpp` 覆盖这条边界。

### 组的 lease 原子性

组对象淘汰前先遍历全组成员：**任一成员 lease 未过期，整组直接返回空**（`:10203`–`:10208`）。通过这一关后，再逐成员过滤 hard pin/soft pin/可淘汰性（`:10221`–`:10227`）。所以"组"保证的是 lease 层面的一致性，不是"每个成员必须一起消失"——被 pin 住的成员可以留下。

### census 与驱逐之间的 churn

候选只记录 `lease_timeout` 和 key，不持迭代器。这既避免了跨阶段悬垂，也让 census 可以并行。代价是驱逐阶段可能白跑一些已失效的候选；代码用 `reserve_slack` 预留候选余量，并在不足时从 cutoff 之外补扫（`:10570`–`:10582`）。

### offload 与删除的竞态

`EraseMetadata` 会主动取消该 key 的 `offloading_tasks` 并 `dec_refcnt`，否则 Store Worker 的在途卸载会变成 600 秒才过期的孤儿（`:2123`–`:2138` 注释）。这是"元数据删除"必须先于"后台任务完成"的典型处理。

### HA / oplog

`enable_oplog_` 打开时，`evict_replicas` 不直接弹副本，而是按 `REMOVED` 状态计数（`:9975`–`:9980`）；驱逐前先 `persist_evict_oplog_or_skip`（`:10091`），持久化失败就跳过本轮且不删元数据，保证本地状态与已发布状态一致。

## 8.9 验证与结论

- **读保护**：`mooncake-store/src/master_service.cpp` 的 `GetReplicaList`(L3626)、`GrantLeaseForGroup`(L3686)、`GrantReadLease`(L3688)；`include/master_service.h` 的 `GrantReadLease`(L1322)、`NeedsReadLeaseRefresh`(L1330)、`IsLeaseExpired`(L1337)、lease 字段(L1104)、soft pin 字段(L1106)、`hard_pinned`(L1113)。
- **淘汰**：`src/master_service.cpp` 的 `EvictionThreadFunc`(L9053)、触发条件(L9061)、`BatchEvict`(L9954)、`IsEvictableMemoryReplica`(L3570)、census(L10311)、`ideal_evict_num`(L10388)、`nth_element`(L10503)、重校验(L10530)、`try_evict_group_or_object`(L10179)、组 lease 检查(L10203)、`try_evict_or_offload`(L10024)、`evict_replicas`(L9973)、`EraseMetadata` 调用(L10550)、`NoFBatchEvict`(L10821)。
- **配额淘汰**：`EvictTenantMemoryForQuota`(L9710)、`can_evict_replicas`(L9724)、`try_evict_or_offload`(L9766)、`try_evict_group_or_object`(L9818)。
- **删除**：`Remove`(L6416)、lease 检查(L6428)、完成态检查(L6439)、`BatchRemove`(L6767)、`EraseMetadata` 实现(L2111)、僵尸清理(L9127)。
- **配置**：`conf/master.json:10`–`:15`；`kEvictionThreadSleepMs`(`include/master_service.h:2129`)。
- **测试**：`mooncake-store/tests/master_service_evict_scenario_test.cpp`、`batch_remove_test.cpp`、`master_service_processing_key_double_erase_test.cpp`、`offload_on_evict_test.cpp`、`promotion_on_hit_test.cpp`。注意 `eviction_strategy_test.cpp` 测的是独立策略类，与 `master_service.cpp` 的容量淘汰路径不是同一实现。

**结论**：生命周期由 **lease + pin + 时间戳排序 + 单一删除出口** 协调。读路径用 `GrantReadLease` 把 `lease_timeout` 推到"当前时间 + 10s"，且只增不减；组对象在半 TTL 时整组续租。淘汰线程按水位或分配失败触发 `BatchEvict`，先并行 census 出"lease 过期、非 hard pin、有 `refcnt==0` 的 MEMORY 副本"的对象，再用 `nth_element` 按 `lease_timeout` 取最老的一批，串行重校验后驱逐；`lease_timeout` 因此同时是保护截止时间和 LRU 时间戳，本版本并不使用 `eviction_strategy_`。**淘汰只移除 MEMORY 副本，删除才移除元数据**；只有最后一份有效副本消失才 `EraseMetadata`。删除有三方触发（客户端、淘汰线程、僵尸清理），但都汇入 `EraseMetadata` 这一个出口，由它统一清理 offload/replication/配额/soft pin 状态。下一篇看 SSD 分层：内存被淘汰后，数据如何下沉、如何加载回来。

---

*下一篇：篇9 SSD 分层——缓存如何下沉、加载与恢复？*
