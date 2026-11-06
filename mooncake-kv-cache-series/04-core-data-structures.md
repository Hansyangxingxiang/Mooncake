# 篇4 缓存对象如何组织？读懂核心数据结构

> 系列：《Mooncake KV Cache 源码解析》
> 版本锚点：`v0.3.13.post1`（commit `7197358`）
> 源码链接基座：`https://github.com/kvcache-ai/Mooncake/blob/v0.3.13.post1/`

## 4.1 具体问题

一个 KV Cache 对象的 key 变成一个可被 RDMA 读取的内存地址，中间经过了几层结构？每一层各自负责什么，谁持有谁，谁负责释放？

搞清楚这条映射链，后面讨论可见性、淘汰、SSD 下沉时就不会迷路——因为那些机制操作的正是这几层结构。本篇回答四个具体问题：

1. **对象**在 Master 里以什么结构存在？
2. **副本**如何表达"同一份数据的多种介质"？
3. **内存区域**如何携带 TE 需要的地址信息？
4. 这些结构的**所有权与生命周期**分别是什么？

> **版本提示**：计划里提到的 `object_entry.h`、`object_metadata.h`、`object_index.h`、`object_runtime_state.h` 在 `v0.3.13.post1` 中**都不存在**。对象元数据是 `master_service.h` 里的 `ObjectMetadata`，副本在 `replica.h`，内存区域在 `allocator.h`。按旧文件名录源码会一无所获。

## 4.2 场景与版本

- **场景**：一个对象，单副本，写入后被读取、可能被淘汰。
- **版本**：`v0.3.13.post1`。
- **核心文件**：`include/types.h`、`include/allocator.h`、`include/replica.h`、`include/master_service.h`。

## 4.3 数据结构关系图

```
key (ObjectKey = std::string)                      types.h:181
 │
 ▼
ObjectMetadata                                     master_service.h:1031
 │  client_id / put_start_time / size / data_type
 │  lease_timeout / soft_pin_timeout / hard_pinned
 │  replicas_ : ReplicaList                         types.h:187
 ▼
ReplicaList = unordered_map<uint32_t, Replica>
 │  一个对象多个副本，key 是 ReplicaID
 ▼
Replica                                            replica.h:259
 │  id_ / status_ / refcnt_
 │  data_ : variant<MemoryReplicaData, NoFReplicaData,
 │                  DiskReplicaData, LocalDiskReplicaData,
 │                  DfsReplicaData>                 replica.h:700
 ▼
MemoryReplicaData { unique_ptr<AllocatedBuffer> }   replica.h:203
 │
 ▼
AllocatedBuffer                                    allocator.h:39
 │  buffer_ptr_ / size_ / segment_name_
 │  weak_ptr<BufferAllocatorBase> allocator_        allocator.h:96
 ▼
AllocatedBuffer::Descriptor                        allocator.h:83
 │  size_ / buffer_address_ / protocol_ / transport_endpoint_
 ▼
Segment（存储节点的一段连续内存）                    types.h:433
    base / size / te_endpoint / host_id
```

注意这条链上出现了**两种 Descriptor**：`Replica::Descriptor`（副本级，含状态）和 `AllocatedBuffer::Descriptor`（内存级，含地址）。前者是后者加上类型和状态的外壳。

## 4.4 三个关键代码片段

### 片段1：`ObjectMetadata`——对象状态集中在一个不可移动的结构里

```cpp
// mooncake-store/include/master_service.h:1031（节选）
struct ObjectMetadata {
    ~ObjectMetadata() {
        MasterMetricManager::instance().dec_key_count(1);
        if (soft_pin_timeout) {
            MasterMetricManager::instance().dec_soft_pin_key_count(1);
        }
    }
    ObjectMetadata(const ObjectMetadata&) = delete;
    ObjectMetadata& operator=(const ObjectMetadata&) = delete;
    ObjectMetadata(ObjectMetadata&&) = delete;
    ObjectMetadata& operator=(ObjectMetadata&&) = delete;

    UUID client_id;                                     // 谁发起的 PutStart
    std::chrono::system_clock::time_point put_start_time; // 僵尸清理用
    const size_t size;                                  // 对象大小
    std::optional<uint64_t> object_checksum;            // 可选校验和
    const ObjectDataType data_type;                     // KVCACHE / TENSOR / ...
    const std::string group_id;
    const TenantId tenant_id;
    const std::string user_key;

    mutable SpinLock lock;
    mutable std::chrono::system_clock::time_point lease_timeout GUARDED_BY(lock);
    mutable std::optional<std::chrono::system_clock::time_point>
        soft_pin_timeout GUARDED_BY(lock);
    const bool hard_pinned{false};
    TenantQuotaLedger quota_ledger;
    // replicas_ : ReplicaList
};
```

**为什么拷贝/移动全部 delete**：结构里有 `SpinLock`，还有一组靠析构维持的指标（`key_count`）。允许移动会导致锁语义和指标计数错乱。因此它被原地构造进 `unordered_map` 的节点（`emplace(std::piecewise_construct, ...)`，见 `master_service.cpp:4256`），之后地址不再变化。

**输入/输出**：`ObjectMetadata` 不对外暴露，只作为 Master 内存态存在。客户端看到的是它的副本列表序列化结果。

> **分析（推断）**：`lease_timeout` 用 `SpinLock` 保护而不是 `shared_mutex`，是因为 lease 刷新非常频繁（每次 Get），临界区只有几行，自旋比睡眠更划算。

### 片段2：`Replica`——介质用 variant，状态用枚举，引用计数只表示 busy

`Replica` 有两个 variant：**数据**（物理介质）和**描述符**（发给客户端的可序列化形式）。

```cpp
// mooncake-store/include/replica.h:700
std::variant<MemoryReplicaData, NoFReplicaData, DiskReplicaData,
             LocalDiskReplicaData, DfsReplicaData>
    data_;

// mooncake-store/include/replica.h:566
struct Descriptor {
    ReplicaID id;
    std::variant<MemoryDescriptor, NoFDescriptor, DiskDescriptor,
                 LocalDiskDescriptor, DistributedFSDescriptor>
        descriptor_variant;
    ReplicaStatus status;
    YLT_REFL(Descriptor, id, descriptor_variant, status);
};

// mooncake-store/include/replica.h:203
struct MemoryReplicaData { std::unique_ptr<AllocatedBuffer> buffer; };
// :211
struct DiskReplicaData { std::string file_path; uint64_t object_size = 0; };
// :216
struct LocalDiskReplicaData {
    UUID client_id; uint64_t object_size = 0; std::string transport_endpoint;
};
```

`Descriptor` 用 `YLT_REFL` 做反射序列化，这就是 Master 返回给客户端、再交给 TE 的那份"位置信息"。

状态机是受限的，非法转换只记日志不改状态：

```cpp
// mooncake-store/include/replica.h:511
void mark_complete() {
    if (status_ == ReplicaStatus::PROCESSING) status_ = ReplicaStatus::COMPLETE;
    else if (status_ == ReplicaStatus::COMPLETE)
        LOG(WARNING) << "Replica already marked as complete";
    else LOG(ERROR) << "Invalid replica status: " << status_;
}
// :521  mark_processing 只允许 COMPLETE → PROCESSING
// :529  mark_removed 允许 COMPLETE/PROCESSING → REMOVED
```

**关键点**：`refcnt_` **不是生命周期引用计数**。副本的生命周期由 `ObjectMetadata` 里的 `unique_ptr` / 容器持有；`refcnt_` 只表示"当前是否被读写占用"，用于淘汰判断：

```cpp
// mooncake-store/include/replica.h:386
[[nodiscard]] bool is_busy() const { return refcnt_.load() > 0; }
```

> **观点**：把"忙"和"活"分成两个概念是正确的。淘汰要看的是"有没有人在用"，而不是"有几个持有者"；后者由所有权模型静态保证。

### 片段3：`AllocatedBuffer`——地址信息的载体，析构即归还

```cpp
// mooncake-store/include/allocator.h:83
struct Descriptor {
    uint64_t size_;
    uintptr_t buffer_address_;
    std::string protocol_;
    std::string transport_endpoint_;
    YLT_REFL(Descriptor, size_, buffer_address_, protocol_,
             transport_endpoint_);
};

// mooncake-store/include/allocator.h:96
std::weak_ptr<BufferAllocatorBase> allocator_;   // 所属分配器
std::string segment_name_;                       // 所属 segment
void* buffer_ptr_{nullptr};
std::size_t size_{0};
std::optional<offset_allocator::OffsetAllocationHandle> offset_handle_;
```

析构时若无条件归还：

```cpp
// mooncake-store/src/allocator.cpp:45
AllocatedBuffer::~AllocatedBuffer() {
    auto alloc = allocator_.lock();
    if (alloc) {
        alloc->deallocate(this);          // 归还 offset / 内存
        VLOG(1) << "buf_handle_deallocated size=" << size_;
    }
}
```

**所有权链**：`ObjectMetadata` → `Replica` → `unique_ptr<AllocatedBuffer>` → `weak_ptr<BufferAllocatorBase>`。前三级是强持有，最后一级是弱引用：**allocator 可以比 buffer 先死**（例如 segment 被卸载），此时析构不再归还，因为分配器自身已经没了。

这个设计有一个直接后果，体现在可读性判断里：

```cpp
// mooncake-store/include/replica.h:444
[[nodiscard]] bool has_invalid_mem_handle() const {
    if (is_memory_replica()) {
        const auto& mem_data = std::get<MemoryReplicaData>(data_);
        return !mem_data.buffer->isAllocatorValid();   // weak_ptr 是否还有效
    }
    return false;
}
```

`IsReplicaReadable`（`master_service.cpp:3546`）正是靠它把"segment 已卸载"的副本排除掉。**所以副本记录还在，不代表副本还能读**——这是故障恢复时必须理解的一点。

## 4.5 一次 slice 的旅程

把这些结构按一次写入串起来：

1. 调用方给 `(buffer, size)`，`split_into_slices` 把它切成 `Slice{ptr, size}` 列表（`types.h:421`）。
2. Master 的 `AllocateAndInsertMetadata` 为每个副本分配 `AllocatedBuffer`，`Replica` 以 `PROCESSING` 状态插入 `ObjectMetadata.replicas_`。
3. `Replica::Descriptor` 序列化后返回客户端，其中 `AllocatedBuffer::Descriptor` 携带 `transport_endpoint_` 与 `buffer_address_`。
4. 客户端把 slice 和这份 descriptor 交给 TE；TE 用 endpoint 找远端 segment，用 address + size 定位写入范围。
5. `PutEnd` 只改 `ReplicaStatus`，**不移动任何地址信息**。
6. 对象被删除/淘汰时，`ObjectMetadata` 连同其 `Replica` 一起析构，`AllocatedBuffer` 析构触发 allocator 归还。

第 5 步值得强调：**可见性切换只改状态位，不改地址**。所以"读完再失效"不会导致地址漂移，这也是 lease 机制能够奏效的前提。

## 4.6 异常与并发

- **状态非法转换**：`mark_complete` 从 `UNDEFINED` 调用会记 `ERROR` 但不改变状态。**分析**：这是"防御性记录而非崩溃"的风格——Master 不会因为一次非法调用退出，但日志会留下证据。
- **segment 卸载**：`AllocatedBuffer` 的 `weak_ptr` 失效，`has_invalid_mem_handle()` 为真，副本从可读集合中消失。数据可能还物理存在，但系统不再把它当作可读副本——**这与"数据丢失"不同，属于"寻址能力丢失"**。
- **并发读**：`refcnt_` 只做 `fetch_add/sub`，`is_busy()` 是瞬时判断。淘汰方需要先检查再操作，但检查与操作之间不是原子的——真正的保护靠 lease（篇8），`is_busy` 只是辅助信号。
- **指标一致性**：`ObjectMetadata` 析构扣 `key_count`，`Replica` 析构对磁盘副本扣 `allocated_file_size`（`replica.h:311`）。**分析**：把指标维护绑定在 RAII 上，避免"忘记上报"这类泄漏。
- **锁与不可移动**：`ObjectMetadata` 禁止移动，保证 `GUARDED_BY(lock)` 的字段始终与锁同址。移动语义一旦放开，静态分析就失效。

## 4.7 验证与结论

- **类型定义**：`mooncake-store/include/types.h` 的 `ObjectKey`(`:181`)、`ReplicaList`(`:187`)、`Slice`(`:421`)、`Segment`(`:433`)。
- **内存区域**：`mooncake-store/include/allocator.h` 的 `AllocatedBuffer`(`:39`)、`Descriptor`(`:83`)、`allocator_`(`:96`)；析构在 `src/allocator.cpp:45`。
- **副本**：`mooncake-store/include/replica.h` 的 `ReplicaStatus`(`:52`)、各 `*ReplicaData`(`:203`–`:232`)、`Replica`(`:259`)、`Descriptor`(`:566`)、`data_`(`:700`)、`is_busy`(`:386`)、`mark_complete`(`:511`)、`has_invalid_mem_handle`(`:444`)。
- **对象元数据**：`mooncake-store/include/master_service.h` 的 `ObjectMetadata`(`:1031`)。
- **测试**：`mooncake-store/tests/replica_selection_test.cpp`、`segment_test.cpp`、`master_service_replication_scenario_test.cpp`。注意：`object_entry_test.cpp` / `object_index_test.cpp` 在该 tag 不存在。

**结论**：Mooncake 的对象组织是**五层映射**——key → `ObjectMetadata`（不可移动，含副本表与生命周期字段）→ `Replica`（variant 区分介质 + 状态机 + busy 计数）→ `AllocatedBuffer`（地址 + 弱引用分配器）→ `Segment`（物理内存段）。`Replica::Descriptor` 是元数据与数据分路的落点：它携带 `transport_endpoint_` 与 `buffer_address_`，让 TE 在不经过 Master 的情况下直连数据。

**边界**：本篇讨论的是单副本、内存介质的最短链。NoF/DFS/LocalDisk 副本在 `Replica` 层替换了 variant，但上层映射不变；SSD 相关的额外字段（文件路径、offset、shard）见篇9。

下一篇进入 Transfer Engine：一份 `Descriptor` 是如何变成一个异步传输任务的。

---

*下一篇：篇5 Transfer Engine——传输任务如何提交和完成？*
