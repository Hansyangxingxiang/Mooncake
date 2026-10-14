# 篇9 SSD 分层：缓存如何下沉、加载与恢复？

> 系列：《Mooncake KV Cache 源码解析》
> 版本锚点：`v0.3.13.post1`（commit `7197358`）
> 源码链接基座：`https://github.com/kvcache-ai/Mooncake/blob/v0.3.13.post1/`

## 9.1 具体问题

内存容量有限，SSD 容量大。这一篇回答三个问题：

1. **下沉**：内存压力下，对象如何被卸载到 SSD？
2. **加载**：SSD 上的对象被命中时，如何经内存中转加载？
3. **恢复**：Master 重启后，SSD 对象元数据如何恢复？

## 9.2 场景与版本

- **场景**：缓存分两层——L1 内存（快、小）+ L2 本地 SSD（慢、大）。内存不足时对象下沉到 SSD，命中时提升回内存。
- **版本**：`v0.3.13.post1`。注意：本版本 SSD 逻辑在 `src/file_storage.cpp`（`include/file_storage.h`）、`src/local_ssd/manager.cpp`、`src/storage/`，`LocalHotCache` 在 `include/local_hot_cache.h`。
- **主路径**：下沉 `FileStorage::Heartbeat`（`file_storage.h:154`）→ `OffloadObjects`（`file_storage.h:119`）；加载 `ProcessPromotionTasks`（`file_storage.h:166`）→ `BatchLoad`（`file_storage.h:175`）；恢复 `ReRegisterOffloadedObjects`（`file_storage.h:199`）。

## 9.3 调用链图

```
下沉 (L1 → L2)：
FileStorage::Heartbeat (file_storage.h:154)
   │  1. 上报对象状态给 Master
   │  2. 接收 Master 反馈：哪些对象该 offload
   │  3. 触发异步 offload
   ▼
OffloadObjects (file_storage.h:119)
   │  读源数据 → 写入 SSD bucket → 通知 Master
   ▼
Master 侧：PushOffloadingQueue (master_service.cpp:9793)
   │  replica.inc_refcnt() 保护源副本
   ▼
SSD 副本就绪，内存副本可释放

加载 (L2 → L1)：
ProcessPromotionTasks (file_storage.h:166)
   │  从 Master 拉 promotion 任务
   │  为每个 key 暂存 MEMORY replica
   │  从本地 SSD 拷字节到 replica
   ▼
BatchLoad (file_storage.h:175) → 通知 Master

恢复：
ReRegisterOffloadedObjects (file_storage.h:199)
   │  Master 重启后重新注册 SSD 对象元数据
   ▼
ScanMeta → 同步 SSD 对象元数据
```

## 9.4 三个关键代码片段

### 片段1：下沉——Heartbeat 驱动（`file_storage.h:154`）

`Heartbeat` 是 SSD 分层的**驱动引擎**，注释列出完整流程——上报状态、接收 offload 反馈、触发异步 offload、处理 promotion、水位淘汰：

```cpp
// file_storage.h:144  Heartbeat 的职责
// 1. Sends object status (e.g., access frequency, size) to the master
// 2. Receives feedback on which objects should be offloaded
// 3. Triggers asynchronous offloading of pending objects
// 4. If offload work was returned, pulls and processes any pending L2->L1
//    promotion tasks queued by the master
// 5. Runs proactive local-disk watermark eviction
tl::expected<void, ErrorCode> Heartbeat();
```

**关键点**：下沉是**Master 决策、FileStorage 执行**。FileStorage 心跳上报对象状态，Master 决定哪些对象该 offload，FileStorage 异步执行。这是"元数据与数据分路"在 SSD 层的体现——Master 只做决策，数据搬运在 FileStorage。

### 片段2：下沉保护——inc_refcnt（`master_service.cpp:9793`）

Master 侧 `PushOffloadingQueue` 把对象加入 offload 队列时，对源副本 `inc_refcnt()` 保护，防止 offload 期间被删除：

```cpp
// master_service.cpp:9793  入队 offload + 保护源副本
auto result = PushOffloadingQueue(
    MakeObjectIdentity(key, normalized_tenant), replica, &mirror_clients);
if (result) {
    replica.inc_refcnt();   // ← 保护源副本，offload 期间不可删
    tenant_state.offloading_tasks.emplace(
        key, OffloadingTask{replica.id(), now, std::move(mirror_clients)});
    queued = true;
}
```

**关键点**：`inc_refcnt` 让源副本在 offload 期间 busy（见篇4 的 `is_busy`），`Remove`/`Eviction` 会跳过。offload 完成后 Master 侧 reaper 递减 refcnt 并擦除任务。`offload_force_evict_` 强制淘汰，`offload_cap` 限制每轮 offload 队列长度（`master_service.cpp:9761`）。

### 片段3：加载——ProcessPromotionTasks（`file_storage.h:166`）

`ProcessPromotionTasks` 驱动 L2→L1 提升：从 Master 拉任务、暂存 MEMORY replica、从 SSD 拷字节、通知 Master：

```cpp
// file_storage.h:158  L2->L1 promotion 流程
// Pulls promotion work from the master, stages a MEMORY replica for each
// key, copies the bytes from local SSD into that replica, and notifies the
// master on success. A failure on any single key is logged and skipped;
// the master-side reaper decrements the source replica's refcnt and
// erases the task entry on TTL expiry, and any orphaned PROCESSING
// MEMORY replica is reaped via the standard discarded-replicas path.
tl::expected<void, ErrorCode> ProcessPromotionTasks();
```

**关键点**：提升是**经内存中转**的——先暂存 MEMORY replica，再从 SSD 拷字节进去，最后通知 Master。失败单 key 跳过，Master 侧 reaper 处理超时任务和孤儿 PROCESSING replica。`promotion_on_hit_`（`master_service.cpp:3695`）在命中时触发提升：无 MEMORY 副本但有 LOCAL_DISK 副本时，读命中后提升回内存。

## 9.5 完成语义：下沉是异步的、可失败的

回答开篇问题，SSD 分层的完成语义分三层：

1. **下沉**：`Heartbeat` 触发异步 offload，`OffloadObjects` 读源数据写 SSD。**下沉是异步的**——`PushOffloadingQueue` 入队后立即返回，实际搬运在 FileStorage 后台。`inc_refcnt` 保护源副本直到 offload 完成。
2. **加载**：`ProcessPromotionTasks` 经内存中转加载。**失败单 key 跳过**，不阻塞整批。
3. **恢复**：`ReRegisterOffloadedObjects` 在 Master 重启后重新注册 SSD 对象元数据（`file_storage.h:199` 注释：与 `Init()` 的 `ScanMeta` 逻辑相同）。

**关键点**：下沉是**Master 决策 + FileStorage 执行**的异步流程。`inc_refcnt` 保证 offload 期间源副本不被删除；`offload_force_evict_` 提供强制淘汰逃生舱；`offload_cap` 限制队列长度防止积压。

## 9.6 异常与并发

### 卸载与读取并发

`inc_refcnt`（`master_service.cpp:9797`）让源副本 busy，读取/删除会跳过。offload 完成后 reaper 递减 refcnt。若读取先于 offload，读走内存副本；若 offload 先完成，读走 SSD 副本（`GetReplicaList` 返回 LOCAL_DISK 副本）。

### 每 bucket 软错误

`IsPerBucketSoftOffloadError`（`file_storage.h:140`）区分软错误——`INVALID_READ`（源数据读不到）、`OBJECT_ALREADY_EXISTS`（已 offload 或并发 offload）。软错误只影响当前 bucket，`OffloadObjects` 报告失败后继续其余 bucket，**不中止整个 offload 周期**（issue #2827 修复）。

### SSD 故障 / 元数据丢失

`DrainLocalDiskSegment`（`file_storage.h:44`）在关闭时注销磁盘层并等待宽限期，让 in-flight offload 读完成。`ReRegisterOffloadedObjects` 在 Master 重启后重新发现 SSD 对象元数据。

### 水位淘汰

`RunDiskWatermarkEviction`（`file_storage.h:170`）在 SSD 水位过高时主动淘汰磁盘副本，`NotifyEvictedDiskReplicas`（`file_storage.h:172`）通知 Master。

## 9.7 验证与结论

- **FileStorage**：`mooncake-store/include/file_storage.h` 的 `Heartbeat`(L154)、`OffloadObjects`(L119)、`ProcessPromotionTasks`(L166)、`BatchLoad`(L175)、`ReRegisterOffloadedObjects`(L199)、`DrainLocalDiskSegment`(L44)。
- **Master 侧**：`mooncake-store/src/master_service.cpp` 的 `PushOffloadingQueue`(L9793)、`try_evict_or_offload`(L9766)、`promotion_on_hit_`(L3695)。
- **热缓存**：`mooncake-store/include/local_hot_cache.h` 的 `LocalHotCache`(L47)、`LocalHotCacheHandler`(L284)。
- **测试**：`mooncake-store/tests/local_ssd/`、`file_storage_test.cpp`、`file_storage_promotion_test.cpp`、`offload_on_evict_test.cpp`、`promotion_on_hit_test.cpp`、`master_service_ssd_test.cpp`。

**结论**：SSD 分层是**下沉 → 加载 → 恢复**三阶段。下沉由 `Heartbeat` 驱动，Master 决策、FileStorage 异步执行，`inc_refcnt` 保护源副本；加载经内存中转，`ProcessPromotionTasks` 从 SSD 拷字节到 MEMORY replica，失败单 key 跳过；恢复由 `ReRegisterOffloadedObjects` 在 Master 重启后重新注册元数据。**下沉是异步的、可失败的**，软错误只影响当前 bucket。最后一篇接入 vLLM，看端到端前缀命中。最后一篇接入 vLLM。

---

*下一篇：篇10 接入 vLLM——一次前缀命中如何减少 Prefill？*
