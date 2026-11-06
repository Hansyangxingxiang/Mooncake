# 篇9 SSD 分层：缓存如何下沉、加载与恢复？

> 系列：《Mooncake KV Cache 源码解析》
> 版本锚点：`v0.3.13.post1`（commit `7197358`）
> 源码链接基座：`https://github.com/kvcache-ai/Mooncake/blob/v0.3.13.post1/`

## 9.1 具体问题

篇8 结束在"淘汰只移除 MEMORY 副本，对象可能还有 LOCAL_DISK 副本"。这一篇把这层 SSD 存储拆成四个动作：

1. **下沉**：内存副本如何写到本机 SSD？谁决定、谁执行？
2. **落盘**：数据以什么形态存在于磁盘？崩溃时靠什么判断哪些能恢复？
3. **加载**：命中一个只有 SSD 副本的对象，数据如何回到内存？
4. **恢复**：Master 重启或段丢失后，已下沉的对象如何重新变得可读？

先澄清三个容易混的点。第一，**SSD 副本是 `ReplicaType::LOCAL_DISK` 副本，不是一台独立存储节点**——它由拥有该 segment 的 client 本地管理，Master 只保留"哪个 endpoint 上有这份数据"的元数据。第二，**下沉成功不删除 MEMORY 副本**，它是"多了一份副本"，不是"把内存搬走"；只有 `offload_on_evict_` 打开时，下沉才和淘汰绑定。第三，`LocalHotCache` 是 client 侧为**远端读**做的 DRAM 热缓存，与本文的 offload/promotion 不是一条路径，别按名字对号入座。

## 9.2 场景与版本

- **场景**：Store 集群的内存副本被淘汰后，希望数据仍在本地 SSD 上，下次读命中时能重新加载进内存，而不是回源重算。
- **版本**：`v0.3.13.post1`。Master 侧在 `src/master_service.cpp`（入队、注册、promotion 状态机、TTL 回收）；执行侧在 `src/file_storage.cpp`（Heartbeat、D2H 拷贝、promotion 执行）；落盘在 `src/storage_backend.cpp` 的 `BucketStorageBackend`；后台邮箱在 `src/local_ssd/manager.cpp`。
- **默认参数**：`heartbeat_interval_seconds=10`（`include/storage_backend.h:354`）；`bucket_size_limit=256MB`、`bucket_keys_limit=500`（`:205`、`:208`）；`total_size_limit=2TB`（`:350`）；磁盘水位淘汰默认开启，high `0.90` / low `0.80`（`:367`–`:369`）；`offloading_queue_limit=50000`（`include/master_config.h:129`）；`promotion_on_hit=false`、`promotion_admission_threshold=2`、`promotion_queue_limit=50000`、`promotion_max_per_heartbeat=1`（`:134`–`:142`）。
- **主路径**：下沉 `PutEnd` 的下沉块（`master_service.cpp:4610`）→ `PushOffloadingQueue`（`:7541`）→ `Heartbeat`（`file_storage.cpp:649`）→ `OffloadObjects`（`:309`）→ `BatchOffload`（`storage_backend.cpp:1735`）→ `NotifyOffloadSuccess`（`master_service.cpp:7365`）；加载 `GetReplicaList`（`:3691`）→ `TryPushPromotionQueue`（`:8546`）→ `ProcessPromotionTasks`（`file_storage.cpp:855`）→ `NotifyPromotionSuccess`（`master_service.cpp:8837`）。

## 9.3 调用链图

```
下沉（PutEnd 后异步）
PutEnd (master_service.cpp:4610)
   ├─ 条件：非 DFS + enable_offload_ + !offload_on_evict_
   ├─ VisitReplicas(completed && memory)              :4619
   │    └─ PushOffloadingQueue (master_service.cpp:7541)
   │         └─ local_ssd_manager_.EnqueueOffload(...)  ← 只入队
   ├─ 首个成功入队的副本 inc_refcnt()                  :4626
   └─ offloading_tasks.emplace(key, OffloadingTask{source_id, now, mirrors}) :4631
   ▼
FileStorage::Heartbeat (file_storage.cpp:649)   ← 每 10s
   ├─ OffloadObjectHeartbeat (master_service.cpp:7292)  ← 取 pending
   ├─ OffloadObjects (file_storage.cpp:309)
   │    ├─ 按 tenant 拼 scoped key                     :319
   │    ├─ AllocateOffloadingBuckets (storage_backend.cpp:2432)
   │    ├─ GPU 源 → pinned_buffer_pool_ D2H staging    :439
   │    └─ BatchOffload (storage_backend.cpp:1735)
   │         ├─ BuildBucket                            :2553
   │         ├─ PrepareEviction → notify Master → Commit → 删旧文件 :1775–1790
   │         ├─ WriteBucket (+datasync)                :2585
   │         ├─ 本地 object_bucket_map_ 提交           :1809
   │         └─ complete_handler → NotifyOffloadSuccess :7365
   └─ 失败 key 汇总成 NACK 哨兵                        :546–552
   ▼
NotifyOffloadSuccess (master_service.cpp:7365)
   ├─ data_size < 0 → dec_refcnt + 删任务（NACK）      :7390
   ├─ 已有对象 → source->dec_refcnt()                  :7451
   └─ 新增 / 更新 LOCAL_DISK COMPLETE 副本              :7460–7506

加载（promotion，默认关闭）
GetReplicaList (master_service.cpp:3691)
   ├─ 无 MEMORY 副本 + 有 LOCAL_DISK 副本 → 可提升
   └─ TryPushPromotionQueue (master_service.cpp:8546)
        ├─ 频次门槛 promotion_admission_threshold_
        ├─ 水位门槛 used_ratio >= eviction_high_watermark_ratio_
        └─ source->inc_refcnt() + PushPromotionQueue :7610
   ▼
PromotionObjectHeartbeat (master_service.cpp:8704)  ← 按 seq 取 top-N
   ▼
ProcessPromotionTasks (file_storage.cpp:855)
   ├─ PromotionAllocStart (master_service.cpp:8718)  ← 追加 PROCESSING MEMORY 副本
   ├─ AllocateBatch (file_storage.cpp:1080)          ← staging buffer
   ├─ BatchLoad (file_storage.cpp:1005)              ← SSD 读入 staging
   ├─ PromotionWrite (file_storage.cpp:974)          ← TE 写入新副本
   └─ NotifyPromotionSuccess (master_service.cpp:8837) ← mark_complete
        失败 → NotifyPromotionFailure (master_service.cpp:8958)

恢复
FileStorage::Init (file_storage.cpp:131)
   ├─ MountLocalDiskSegment + ReportSsdCapacity
   └─ ScanMeta (storage_backend.cpp:2371)
        └─ NotifyOffloadSuccess（元数据来自 .meta 文件）  :186
Heartbeat → SEGMENT_NOT_FOUND → 重新挂载 + ReRegisterOffloadedObjects :1195
```

## 9.4 三个关键代码片段

### 片段1：下沉入队——只登记，不搬数据（`master_service.cpp:4610`）

`PutEnd` 在对象完成时把可下沉的 MEMORY 副本登记进每个 client 的邮箱，同时给**一个**源副本加 `refcnt`，并把这次任务记进 `offloading_tasks`：

```cpp
// master_service.cpp:4610
if (replica_type != ReplicaType::DFS && enable_offload_ &&
    !offload_on_evict_ && !metadata.HasReplica(...dfs processing...)) {
    std::optional<ReplicaID> source_id;
    std::vector<UUID> mirror_clients;
    metadata.VisitReplicas(
        [](const Replica& replica) {
            return replica.is_completed() && replica.is_memory_replica();
        },
        [this, &object_id, &source_id, &mirror_clients](Replica& replica) {
            auto result = PushOffloadingQueue(object_id, replica, &mirror_clients);
            if (result && !source_id.has_value()) {
                replica.inc_refcnt();          // :4626 固定源副本
                source_id = replica.id();
            }
        });
    if (source_id.has_value()) {
        tenant_state.offloading_tasks.emplace(
            object_id.user_key,
            OffloadingTask{*source_id, std::chrono::system_clock::now(),
                           std::move(mirror_clients)});   // :4631
    }
}
```

**关键点**：`PushOffloadingQueue`（`:7541`）按 segment 名解析出 owner client，再 `local_ssd_manager_.EnqueueOffload`，邮箱满时返回 `KEYS_ULTRA_LIMIT`、重复 key 返回 `OBJECT_ALREADY_EXISTS`（`local_ssd/manager.cpp:13` 起）。它**不读数据、不写盘**，真正执行在 client 的 Heartbeat。`inc_refcnt` 只加在第一个成功入队的副本上，`OffloadingTask` 也是每个 key 一条，用 `mirror_clients` 记录真实入队的所有 client。这样即使多个 client 挂着同一 segment，也只保护一份源副本。**分析（推断）**：单 source + mirror 列表是"一次下沉、多份等价副本"的简化——反正 bucket 内容相同，保护一份就够；真正的去重责任被推给了邮箱的 `OBJECT_ALREADY_EXISTS`。

这里还有一个开关差异：`!offload_on_evict_` 是条件的一部分。打开 `offload_on_evict_` 后，`PutEnd` 不做主动下沉，改由淘汰路径在 `try_evict_or_offload`（`:10024`）里下沉并淘汰其余副本（篇8）。**所以"下沉"有两个入口，默认入口在写完成时。**

### 片段2：执行下沉——D2H 中转、bucket 落盘、失败 NACK（`file_storage.cpp:309`）

client 侧先按 tenant 拼 scoped key，再把每个对象的数据整理成 host 可写形态。GPU 源要经过 pinned buffer 的 D2H 拷贝，CPU 源直接复用原 slice：

```cpp
// file_storage.cpp:433
for (auto& [obj_key, slices] : batch_object) {
    std::vector<Slice> host_slices;
    bool obj_success = true;
    for (const auto& slice : slices) {
        auto* device = runtime_accelerator.FindDeviceForPointer(slice.ptr, &info);
        if (device) {
            device->SetContext(info.device_id);
            auto buf = pinned_buffer_pool_->Acquire(slice.size);
            if (!device->Copy(buf.data, slice.ptr, slice.size,
                              device::CopyDirection::kDeviceToHost)) {
                failed_tasks.push_back(task_by_storage_key.at(obj_key));
                obj_success = false;
                break;
            }
            host_slices.emplace_back(Slice{buf.data, slice.size});
            staging_bufs.push_back(std::move(buf));
        } else {
            host_slices.push_back(slice);   // CPU 源零拷贝
        }
    }
    ...
}
```

随后 `BatchOffload(host_batch_object, complete_handler, eviction_handler)`（`:498`）同步完成"淘汰腾空间 → 写 bucket → 通知 Master"。任何没写成功的 key 都必须在本次 Heartbeat 内被释放，否则源副本的 `refcnt` 要一直挂到 TTL 回收：

```cpp
// file_storage.cpp:546
for (const auto& [storage_key, task] : task_by_storage_key) {
    if (all_bucket_keys.find(storage_key) == all_bucket_keys.end())
        failed_tasks.push_back(task);   // 被 GroupOffloadingKeysByBucket 跳过的 key
}
if (!failed_tasks.empty()) {
    std::vector<StorageObjectMetadata> failed_metadatas;
    for (size_t i = 0; i < failed_tasks.size(); ++i)
        failed_metadatas.push_back(StorageObjectMetadata{-1, 0, 0, -1, ""});
    client_->NotifyOffloadSuccess(failed_tasks, failed_metadatas);  // :552
}
```

**关键点**：NACK 不是一个错误码，而是**一条哨兵元数据**：`data_size = -1`。Master 在 `NotifyOffloadSuccess` 里看到 `metadata.data_size < 0`（`master_service.cpp:7390`），就知道"这个 key 下沉失败"，于是 `dec_refcnt`、删除 `offloading_tasks`，绝不 `AddReplica`。这个设计把"失败"塞进和"成功"完全相同的 RPC 通道，省掉一条反向 API，代价是接收方必须永远先判符号。**分析（推断）**：这是典型的"用数据本身编码控制语义"——只有 5 个字段的结构没有错误位可用，`data_size` 是唯一允许为负的字段。

错误分两级：`IsPerBucketSoftOffloadError` 为真的错误只放弃当前 bucket，继续下一个；`KEYS_ULTRA_LIMIT` 或其它硬错误则置 `abort_error` 并 `enable_offloading_ = false`（`:520`–`:530`），本轮不再下沉。

### 片段3：加载——三次 RPC 加一次 TE 写（`file_storage.cpp:855`）

promotion 是异步状态机，client 每轮从 Master 取少量任务，逐个执行"申请副本 → 读 SSD → 写内存 → 提交"：

```cpp
// file_storage.cpp:902
auto alloc_result = client_->PromotionAllocStart(
    key, tenant_id, size, ..., &alloc_id, ...);
if (!alloc_result) {
    auto release = client_->NotifyPromotionFailure(key, tenant_id);
    continue;
}
// :948  按对象大小申请 staging buffer
auto allocated_batch = client_->AllocateBatch(key, size, ...);
// :956  从 SSD 读进 staging
auto load_res = BatchLoad(allocated_batch->slices);
// :974  用 TE 把 staging 写进刚申请的 MEMORY 副本
ErrorCode write_err = client_->PromotionWrite(
    key, tenant_id, alloc_id, slices, ...);
if (write_err != ErrorCode::OK) { ... NotifyPromotionFailure ... continue; }
// :985  告诉 Master "这个副本可以读了"
auto notify_res = client_->NotifyPromotionSuccess(key, tenant_id, alloc_id);
```

**关键点**：`PromotionAllocStart`（`master_service.cpp:8718`）不搬数据：它先扣租户配额、再用 allocation strategy 选 segment、往 `replicas_` 追加一个 **PROCESSING** 状态的 MEMORY 副本并记下 `alloc_id`。数据经 staging buffer 走两跳：SSD → host 内存，host 内存 → 远端（或本地）segment。`NotifyPromotionSuccess`（`:8837`）才是可见性开关：它校验 `alloc_id`、holder、副本仍是 memory+processing，然后 `staged->mark_complete()`（`:8878`），副本从 PROCESSING 变 COMPLETE，从此可读。**所以"加载完成"不是数据落地那一刻，而是 `mark_complete` 那一刻**。失败路径统一走 `NotifyPromotionFailure`（`:8958`），它 `dec_refcnt` 源副本、删掉 `alloc_id` 副本、释放配额、删任务、减 in-flight；任务不存在时直接返回 OK（`:8972`–`:8979`），因此重复失败通知是幂等的。

## 9.5 落盘：一个 bucket 就是两个文件

下沉的物理单位是 bucket，不是单个对象。`BucketStorageBackend` 为每个 bucket 生成两个文件：`<bucket_id>.bucket`（数据）和 `<bucket_id>.meta`（索引），后缀定义在 `include/storage_backend.h:1191`–`:1192`。

数据文件里 key 和 value 是**交错顺序排列**的，不是分开存放：

```cpp
// storage_backend.cpp:2553  BuildBucket
iovs.emplace_back(iovec{const_cast<char*>(object.first.data()),
                        object.first.size()});        // 先放 key 字节
for (const auto& slice : object.second) {
    object_total_size += slice.size;
    iovs.emplace_back(...);                           // 再放各 slice
}
bucket->data_size += object_total_size + object.first.size();
bucket->metadatas.emplace_back(BucketObjectMetadata{
    storage_offset, static_cast<int64_t>(object.first.size()), object_total_size});
storage_offset += object_total_size + object.first.size();
```

`.meta` 是 `struct_pb` 序列化的 `BucketMetadata`：`meta_size`、`data_size`、`keys`，以及每个对象的 `{offset, key_size, data_size}`（`storage_backend.cpp:3479`；结构见 `storage_backend.h:30`–`:37`）。读取时 `LoadBucketMetadata`（`:3511`）整文件反序列化，按 offset 直接定位，不需要扫描数据文件。

写入顺序是**数据先、元数据后**。开了 `USE_URING` 时，数据还要 `datasync` 之后才写元数据；非 uring 回退路径只做 `vector_write`，没有显式 sync（`:2678`–`:2699`）：

```cpp
// storage_backend.cpp:2665
// Flush bucket data to stable storage before writing metadata.
auto sync_result = uring_file->datasync();
...
StoreBucketMetadata(bucket_id, bucket_metadata);      // :2704，失败则删数据文件
```

**关键点**：这是崩溃一致性的核心约定——只有 `.meta` 存在且合法，bucket 才被认为有效；元数据永远指向已写完的数据。但要注意它**不保证数据一定已落盘**：非 uring 路径没有显式 `fsync`，掉电后可能出现 `.meta` 在、数据块损坏的情况。**分析（推断）**：这是把持久化强度交给后端配置——`USE_URING`/O_DIRECT 路径买 durability，回退路径买可用性；对可重算的 KV Cache 通常可接受，但对"重启后必须可读"的场景需要显式选前者。

崩溃后的判定在 `Init`：先遍历所有 `.meta`（`:2119`），逐个 `LoadBucketMetadata` 并校验 `data_size`/`meta_size` 非零、`keys`/`metadatas` 非空，非法就删文件（`:2194`）；再扫一遍 `.bucket`，删掉没有 `.meta` 的孤儿（`:2243`–`:2298`）。**分析（推断）**：这等于用"元数据存在性"当提交记录——不需要额外的 WAL 或 journal，代价是每个 bucket 的创建是两阶段且必须由 `Init` 兜底。

bucket 的容量边界是 `256MB` / `500` keys（`storage_backend.h:205`、`:208`）。`GroupOffloadingKeysByBucket`（`storage_backend.cpp:2448`）按这两个上限把本轮 key 切桶。**观点**：256MB 是"写放大 vs 元数据开销"的折中——桶越大，元数据越省、GC 越粗；桶越小，单次失败影响面越小，但 `.meta` 数量和 Init 扫描成本上升。对 KV Cache 这种对象大小差异极大的负载，按对象数（500）而非字节数限桶，能避免一个大对象把整桶的 key 数压到个位数。

## 9.6 完成语义：三个"完成"不是一回事

这条链上有三个不同的完成点，混淆它们会误判数据是否可读：

1. **入队完成**：`PushOffloadingQueue` 返回成功，只是 client 邮箱里多了一条记录。数据没动，副本仍是 MEMORY。
2. **落盘完成**：`BatchOffload` 写完 bucket + `.meta`，本地 `object_bucket_map_` 已提交，随后 `complete_handler` 通知 Master。`NotifyOffloadSuccess` 成功后才新增 LOCAL_DISK COMPLETE 副本（`master_service.cpp:7460`–`:7506`）。**默认路径下 MEMORY 副本仍在**，对象有两份副本；`offload_on_evict_` 路径才会在注册后淘汰其余内存副本。
3. **加载完成**：`NotifyPromotionSuccess` → `mark_complete()`，新的 MEMORY 副本才可读。

`BatchOffload` 内部还有一个顺序值得单独说：**先提交本地索引，再通知 Master**（`storage_backend.cpp:1809`–`:1865`）。注释解释了原因——Master 一旦把读请求重定向到本节点，`BatchLoad` 必须能在 `object_bucket_map_` 里查到 key。反过来，如果通知失败，本地就是"client 能读但 Master 不知道"的幽灵副本，所以 `RollbackCommittedBucket`（`:1876`）会删索引、等 in-flight 读结束、删磁盘文件。**关键点**：数据路径的可见性由 Master 元数据决定，本地索引只是执行层；宁可回滚已落盘的文件，也不留一个 Master 看不见的副本。

### 保护：refcnt 与 TTL

下沉期间源 MEMORY 副本被 `inc_refcnt` 固定（`master_service.cpp:4626`），淘汰 census 的 `refcnt==0` 条件（篇8）因此会跳过它。但 `refcnt` 只表示"有后台任务在用"，不表示"任务一定会完成"——client 可能崩溃、网络可能断。所以还有一层 TTL：`offloading_tasks` 和 promotion 任务都在 `start_time + put_start_release_timeout_sec_` 之后被回收，默认 **600 秒**（`types.h:120`–`:121`）。回收时 `dec_refcnt`、删任务；promotion 还要删 `alloc_id` 副本、释放配额、减 in-flight（`master_service.cpp:9376`–`:9423`）。

**分析（推断）**：600 秒远大于一次 10 秒 Heartbeat 周期，这是"给慢任务留余量、但不能无限等"的取舍。promotion 在 `PromotionAllocStart` 时重置 `start_time`（`master_service.cpp:8833`），让 TTL 覆盖真正的传输阶段而不是排队等待——否则队列一长，任务还没开始就被判死。源码注释补充了另一半：排队阶段的 `alloc_id == 0`，回收线程的 `EraseReplicaByID` 分支是 no-op，所以重置前的那段 TTL 只会删任务、不会误删副本。

## 9.7 异常与并发

### 下沉与读、删、upsert 并发

读不增加 `refcnt`（篇8），所以下沉期间对象仍可被读；bucket 写的是内存里的源副本，与读路径不冲突。删除对象时由 `EraseMetadata` 清理 `offloading_tasks` 并 `dec_refcnt`（篇8）；upsert 则走 `CancelQueuedOffloadTask`（`master_service.cpp:7580`），只在**所有 mirror 的邮箱项都可取消**时才成功，然后 `dec_refcnt` 并删任务。已经在执行的下沉无法取消，upsert 会拿到 `OBJECT_HAS_REPLICATION_TASK`。

### 软错误与整轮中止

`OffloadObjects` 对每个 bucket 独立判断：软错误只把该 bucket 的 key 塞进 `failed_tasks`，继续下一个；硬错误则 `abort_error` 并停止本轮。但注意最后那段 NACK 清扫（`file_storage.cpp:546`–`:552`）是**无条件**的——所有被 `GroupOffloadingKeysByBucket` 跳过、或因为提前 break 没走到的 key，都会在这里被 NACK。**关键点**：这保证"本轮 drain 出来的每个 key 都有结论"，不会让 `refcnt` 悬空到 TTL。`KEYS_ULTRA_LIMIT` 还会把 `enable_offloading_` 置 false，之后的 bucket 直接失败。

### promote 失败与 holder 校验

promotion 的每个阶段失败都走 `NotifyPromotionFailure`，且失败通知可重复（`:8972`–`:8979` 任务不存在直接 OK）。`NotifyPromotionSuccess` 和 `NotifyPromotionFailure` 都校验 **holder**：任务里记的 `holder_id` 必须等于当前 client。**分析（推断）**：这是为了防止 segment 迁移后，旧 holder 的迟到 RPC 改坏新 holder 的状态。`alloc_id` 也参与校验——只有 `GetReplicaByID(alloc_id)` 仍是 memory+processing 才允许 `mark_complete`，避免把已经被淘汰/删除的副本"复活"。

### 段丢失与重注册

Heartbeat 调 `OffloadObjectHeartbeat` 时若 Master 返回 `SEGMENT_NOT_FOUND`（`file_storage.cpp:709` 起），client 会重新 `MountLocalDiskSegment` + `ReportSsdCapacity`，重试本轮，再异步 `ReRegisterOffloadedObjects`（`:1195`）：`ResetScanIterator` 后重新 `ScanMeta`，逐批 `NotifyOffloadSuccess`。**所以恢复不是"Master 主动扫盘"，而是"client 发现自己被忘了，主动重报"。** `ReRegisterOffloadedObjects` 在 `draining_` 时直接中止，避免和卸载赛跑。

`DrainLocalDiskSegment`（`:816`）在 `offloading_mutex_` 下置 `draining_`、卸载段、再 sleep 一个宽限期。注释说明了防的是什么：Heartbeat 可能正好在卸载后重新挂载，让 Master 又看见一个正在退出的段。Master 侧还有第二道闸：`NotifyOffloadSuccess` 在登记副本前检查 `HasMountedLocalDiskSegment(client_id)`（`master_service.cpp:7429`–`:7430`），段已卸载就只清理任务、不 `AddReplicas`，避免"给已离开的 owner 打广告"。**关键点**：`draining_` 是一个 latch，不是锁——一旦置位，后续所有重注册都放弃，直到进程重启。

### 磁盘水位淘汰与物理上限

本地 SSD 也会满。`RunDiskWatermarkEviction`（`file_storage.cpp:615`）在 `enable_disk_watermark_eviction` 打开时，按 high/low 水位（默认 0.90/0.80）调 `EvictAboveDiskWatermark`，把选中的 key 交给 `NotifyEvictedDiskReplicas`（`:566`）→ `BatchEvictDiskReplica(..., LOCAL_DISK)`。Master 侧只删 LOCAL_DISK 副本；`OBJECT_NOT_FOUND` 视为良性。**注意这与篇8 的内存淘汰是两条独立通道**：一条由 Master 的 `BatchEvict` 驱动删 MEMORY 副本，一条由 client 的磁盘水位驱动删 LOCAL_DISK 副本。

物理占用不按逻辑 `data_size` 累加，而按 `stat st_blocks` 实测（`storage_backend.h:231`，注释引用 #3220/#3221）。原因是 bucket GC 的实际释放量不可靠，逻辑计数会漂移。`disk_scan_cache_ms = 500`（`:239`）缓存扫描结果，避免每次判断都走一次 `stat`。**分析（推断）**：这是"相信文件系统而不是自己的账本"——代价是判断有最多 500ms 的滞后，但换来不会因为账本错误而写爆磁盘。

## 9.8 验证与结论

- **下沉入队**：`mooncake-store/src/master_service.cpp` 的 `PutEnd` 下沉块与入队(`L4610`–`L4635`)、`PushOffloadingQueue`(`L7541`)、`CancelQueuedOffloadTask`(`L7580`)；`src/local_ssd/manager.cpp` 的 `EnqueueOffload`(`L13`)。
- **执行与落盘**：`src/file_storage.cpp` 的 `OffloadObjects`(`L309`)、D2H staging(`L433`–`L453`)、`BatchOffload` 调用(`L498`)、NACK 汇总(`L546`–`L552`)、`Heartbeat`(`L649`)、`OffloadObjects` 调用(`L774`)、`ProcessPromotionTasks` 调用(`L788`)；`src/storage_backend.cpp` 的 `BatchOffload`(`L1735`)、`PrepareEviction`/通知/提交顺序(`L1775`–`L1790`)、本地索引提交(`L1809`)、`complete_handler` 与回滚(`L1865`/`L1876`)、`BuildBucket`(`L2553`)、`WriteBucket`(`L2585`)、`datasync`(`L2665`)、`StoreBucketMetadata`(`L2704`/`L3479`)、`LoadBucketMetadata`(`L3511`)、`Init` 扫描(`L2119`)、孤儿清理(`L2243`)、`GroupOffloadingKeysByBucket`(`L2448`)。
- **完成注册**：`src/master_service.cpp` 的 `NotifyOffloadSuccess`(`L7365`)、NACK 判定(`L7390`)、`dec_refcnt`(`L7451`)、LOCAL_DISK 副本写入(`L7460`–`L7506`)。
- **加载**：`src/master_service.cpp` 的 `GetReplicaList` 提升判定(`L3691`)、`TryPushPromotionQueue`(`L8546`)、`PushPromotionQueue`(`L7610`)、`PromotionObjectHeartbeat`(`L8704`)、`PromotionAllocStart`(`L8718`)、`NotifyPromotionSuccess`(`L8837`)、`NotifyPromotionFailure`(`L8958`)、TTL 回收(`L9376`)；`src/file_storage.cpp` 的 `ProcessPromotionTasks`(`L855`)、`AllocateBatch`(`L1080`)、`BatchLoad`(`L1005`)、`PromotionWrite`(`L974`)。
- **恢复与磁盘淘汰**：`src/file_storage.cpp` 的 `Init`(`L131`)、`ScanMeta` 回调(`L186`)、`SEGMENT_NOT_FOUND` 处理(`L709`–`L750`)、`ReRegisterOffloadedObjects`(`L1195`)、`DrainLocalDiskSegment`(`L816`)、`NotifyEvictedDiskReplicas`(`L566`)、`RunDiskWatermarkEviction`(`L615`)。
- **配置**：`include/storage_backend.h` 的 bucket 上限(`L205`/`L208`)、`max_physical_bytes`(`L231`)、`disk_scan_cache_ms`(`L239`)、`total_size_limit`(`L350`)、`heartbeat_interval_seconds`(`L354`)、磁盘水位(`L367`–`L369`)、文件后缀(`L1191`/`L1192`)；`include/master_config.h`(`L129`–`L142`)；`include/types.h`(`L120`–`L121`)。
- **设计文档**：`docs/source/design/store/ssd-offload.md` 的 Heartbeat 职责(`L54`)、`PrepareEviction`/通知顺序(`L75`–`:76`)、`NotifyOffloadSuccess` 加 LOCAL_DISK 副本(`L82`/`L92`)、bucket 写入与淘汰两阶段(`L206`–`:233`)。
- **测试**：`mooncake-store/tests/local_ssd/`、`file_storage_test.cpp`、`file_storage_promotion_test.cpp`、`offload_on_evict_test.cpp`、`promotion_on_hit_test.cpp`、`master_service_ssd_test.cpp`。

**结论**：SSD 分层是**下沉 → 加载 → 恢复**三阶段，且"Master 决策、FileStorage 执行"贯穿始终。下沉在 `PutEnd` 入队，`inc_refcnt` 固定一份源副本，由 client 每 10 秒的 `Heartbeat` 取出并执行；数据经 D2H staging 后按 256MB/500 keys 切成 bucket，写成 `<id>.bucket` + `<id>.meta` 两个文件，数据先写、元数据后写（uring 路径要求 `datasync`），失败 key 用 `data_size=-1` 的 NACK 哨兵回传。落盘成功只是新增一份 LOCAL_DISK 副本，**不删 MEMORY 副本**，所以 offload 不是 eviction。加载默认关闭（`promotion_on_hit=false`），开启后由 `TryPushPromotionQueue` 过频次/水位/队列三道门槛，`PromotionAllocStart` 申请一个 PROCESSING MEMORY 副本，`BatchLoad` + `PromotionWrite` 把数据搬进去，`NotifyPromotionSuccess` 才 `mark_complete` 使其可读；失败路径幂等，TTL 600 秒兜底。恢复靠 client 在 `SEGMENT_NOT_FOUND` 后主动 `ReRegisterOffloadedObjects`，从 `.meta` 重建 Master 侧元数据。**这套设计把"数据在哪"交给 Master、"数据怎么写"交给 FileStorage，用 bucket 文件当提交记录、用 refcnt + TTL 当在途保护**——代价是两阶段提交和一个需要 Init 兜底的孤儿清理。

下一篇回到推理请求侧，看 vLLM 如何通过一次前缀命中跳过 Prefill。

---

*下一篇：篇10 接入 vLLM——一次前缀命中如何减少 Prefill？*
