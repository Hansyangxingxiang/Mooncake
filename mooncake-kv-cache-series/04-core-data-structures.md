# 篇4 缓存对象如何组织？读懂核心数据结构

> 系列：《Mooncake KV Cache 源码解析》
> 版本锚点：`v0.3.13.post1`（commit `7197358`）
> 源码链接基座：`https://github.com/kvcache-ai/Mooncake/blob/v0.3.13.post1/`

## 4.1 具体问题

key、对象、分片、内存区域、副本及其关系是什么？这一篇把 Mooncake 的核心数据结构串起来，回答"一个 KV Cache 对象在内存里到底长什么样"。

> 注意：本版本中，计划提到的 `object_entry.h`/`object_metadata.h`/`object_index.h` 已不存在。对象元数据在 `include/master_service.h` 的 `ObjectMetadata`，副本在 `include/replica.h`，内存区域在 `include/allocator.h`。

## 4.2 场景与版本

- **场景**：理解一个对象从 key 到内存地址的完整映射链。
- **版本**：`v0.3.13.post1`。
- **核心文件**：`include/types.h`、`include/allocator.h`、`include/replica.h`、`include/replica_selection.h`、`include/master_service.h`。

## 4.3 数据结构关系图

```
ObjectKey (std::string)  ← 不透明字符串，框架构造
   │
   ▼
ObjectMetadata (master_service.h:1031)   ← Master 内存中的对象元数据
   │  client_id, put_start_time, size, lease_timeout, soft_pin_timeout, hard_pinned
   │  replicas_ : ReplicaList = unordered_map<uint32_t, Replica>
   ▼
Replica (replica.h:259)   ← 一个副本
   │  id_, status_, refcnt_
   │  data_ : variant<MemoryReplicaData, NoFReplicaData, DiskReplicaData,
   │                  LocalDiskReplicaData, DfsReplicaData>
   ▼
MemoryReplicaData → AllocatedBuffer (allocator.h:39)
   │  buffer_ptr_, size_, segment_name_, offset_handle_
   ▼
AllocatedBuffer::Descriptor (allocator.h:83)   ← 序列化后发给 TE 的"地址"
   │  size_, buffer_address_, protocol_, transport_endpoint_
   ▼
Segment (types.h:433)   ← 存储节点的一段连续内存
   │  id, name, base, size, te_endpoint, protocol, host_id
```

## 4.4 三个关键代码片段

### 片段1：对象元数据（`master_service.h:1031`）

`ObjectMetadata` 是 Master 内存中一个对象的全部状态。它持有副本列表、lease、soft/hard pin 等生命周期信息：

```cpp
// master_service.h:1091  对象元数据核心字段
UUID client_id;                              // 发起 PutStart 的客户端
std::chrono::system_clock::time_point put_start_time;  // 用于僵尸对象清理
const size_t size;                           // 对象大小
std::optional<uint64_t> object_checksum;     // 校验和
const ObjectDataType data_type;              // KVCACHE/TENSOR/WEIGHT...
const std::string group_id;                  // 可选对象组
const TenantId tenant_id;                    // 租户
const std::string user_key;                  // 用户 key
// ... lease_timeout, soft_pin_timeout, hard_pinned ...
// replicas_ : ReplicaList = unordered_map<uint32_t, Replica>
```

**关键点**：`replicas_` 是 `unordered_map<uint32_t, Replica>`（`types.h:187`），一个对象可以有多个副本。`client_id` 记录谁写的，`PutEnd` 时校验（见篇2）。

### 片段2：副本（`replica.h:259`）

`Replica` 用 `std::variant` 区分五种副本类型（内存/NoF SSD/共享磁盘/本地磁盘/DFS），每种持有不同的数据：

```cpp
// replica.h:700  副本数据是 variant
std::variant<MemoryReplicaData, NoFReplicaData, DiskReplicaData,
             LocalDiskReplicaData, DfsReplicaData> data_;
ReplicaStatus status_{ReplicaStatus::UNDEFINED};   // 状态机
std::atomic<uint32_t> refcnt_{0};                  // 引用计数（busy 判断）

// replica.h:52  副本状态机
enum class ReplicaStatus {
    UNDEFINED = 0, INITIALIZED, PROCESSING, COMPLETE, REMOVED, FAILED,
};
```

`Replica::Descriptor`（`replica.h:566`）是序列化后发给客户端的"副本位置"，包含 `id`、`descriptor_variant`（内存/NoF/Disk/LocalDisk/DFS 描述符）、`status`。`SelectBestReplica` 只选 `COMPLETE` 的副本（见篇3）。

### 片段3：内存区域（`allocator.h:39`）

`AllocatedBuffer` 是内存副本的核心，它的 `Descriptor` 就是 TE 传输所需的"地址信息"：

```cpp
// allocator.h:83  序列化后的内存描述符
struct Descriptor {
    uint64_t size_;                 // 大小
    uintptr_t buffer_address_;      // 内存地址
    std::string protocol_;          // tcp/rdma...
    std::string transport_endpoint_; // ip:port，TE 寻址用
};

// allocator.h:96  内部状态
std::weak_ptr<BufferAllocatorBase> allocator_;  // 所属分配器
std::string segment_name_;                      // 所属 segment
void* buffer_ptr_{nullptr};                     // 内存指针
std::size_t size_{0};
std::optional<offset_allocator::OffsetAllocationHandle> offset_handle_;  // offset 分配句柄
```

**关键点**：`transport_endpoint_`（ip:port）是 TE 找到远端内存的关键。`buffer_address_` + `size_` 告诉 TE 数据在哪。这就是"元数据与数据分路"的落点——Master 返回 `Replica::Descriptor`，客户端拿 `transport_endpoint_` 去 TE 找远端，拿 `buffer_address_` 定位数据。

## 4.5 对象 → 内存的完整映射

把三个片段串起来，一个 KV Cache 对象的完整映射是：

1. **key**（`ObjectKey` = `std::string`）→ 框架构造的不透明字符串。
2. **ObjectMetadata** → Master 内存中按 key 索引（`tenant_state.metadata` 是 `unordered_map<string, ObjectMetadata>`）。
3. **Replica** → 对象的一个副本，`variant` 区分类型，`status` 状态机，`refcnt` 判断 busy。
4. **AllocatedBuffer** → 内存副本的实际内存区域，`Descriptor` 序列化出 `transport_endpoint_` + `buffer_address_` + `size_`。
5. **Segment** → 存储节点的一段连续内存，多个 buffer 属于同一 segment（同一故障域）。

## 4.6 异常与并发

- **refcnt 与 busy**：`Replica::is_busy()` 返回 `refcnt_ > 0`（`replica.h:386`）。副本被读取/写入时 refcnt 增加，busy 的副本不能被删除/淘汰（见篇8）。
- **状态机约束**：`mark_complete` 只能从 `PROCESSING` 转 `COMPLETE`（`replica.h:511`），`mark_processing` 只能从 `COMPLETE` 转 `PROCESSING`（`replica.h:521`）。非法转换会 LOG(ERROR)，防止状态错乱。
- **RAII 指标**：`ObjectMetadata` 析构时 `dec_key_count`（`master_service.h:1048`），`Replica` 析构时对 disk/local_disk 副本 `dec_allocated_file_size`（`replica.h:311`），保证指标一致。

## 4.7 验证与结论

- **类型定义**：`mooncake-store/include/types.h`（`ObjectKey`、`Slice`、`Segment`、`ErrorCode`、`ObjectDataType`）。
- **内存区域**：`mooncake-store/include/allocator.h`（`AllocatedBuffer`、`Descriptor`、`OffsetBufferAllocator`、`CachelibBufferAllocator`）。
- **副本**：`mooncake-store/include/replica.h`（`Replica`、`ReplicaStatus`、`ReplicateConfig`、`Replica::Descriptor`）。
- **对象元数据**：`mooncake-store/include/master_service.h` 的 `ObjectMetadata`(L1031)。
- **测试**：`mooncake-store/tests/replica_selection_test.cpp`、`segment_test.cpp`、`object_entry_test.cpp`（若存在）。

**结论**：Mooncake 的对象组织是**分层映射**——key → ObjectMetadata（含副本列表）→ Replica（variant 区分类型 + 状态机）→ AllocatedBuffer（内存区域）→ Segment（存储节点）。`Replica::Descriptor` 的 `transport_endpoint_` + `buffer_address_` 是元数据与数据分路的落点。下一篇进入 Transfer Engine，看传输任务如何提交和完成。

---

*下一篇：篇5 Transfer Engine——传输任务如何提交和完成？*
