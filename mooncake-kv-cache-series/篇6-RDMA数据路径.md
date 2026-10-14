# 篇6 RDMA 数据路径：从内存注册到完成事件

> 系列：《Mooncake KV Cache 源码解析》
> 版本锚点：`v0.3.13.post1`（commit `7197358`）
> 源码链接基座：`https://github.com/kvcache-ai/Mooncake/blob/v0.3.13.post1/`

## 6.1 具体问题

篇5 看到 `submitTransferTask` 把任务交给 transport。这一篇深入 RDMA 后端，回答三个问题：

1. **注册**：本地内存如何变成远端可访问的 MR（Memory Region）？
2. **下发**：一个传输请求如何被拆成 slice、绑定 lkey/rkey、提交 WR？
3. **完成**：CQ 完成事件如何驱动 slice 状态更新？

## 6.2 场景与版本

- **场景**：Store 的 `TransferWrite`/`Get` 通过 TE 在节点间搬数据。RDMA 是默认高性能后端。
- **版本**：`v0.3.13.post1`。注意：本版本 RDMA 后端拆成多文件，位于 `src/transport/rdma_transport/`（`rdma_transport.cpp`、`rdma_context.cpp`、`rdma_endpoint.cpp`、`endpoint_store.cpp`、`worker_pool.cpp`），头文件在 `include/transport/rdma_transport/rdma_transport.h`。
- **主路径**：`registerLocalMemory`（`rdma_transport.cpp:276`）→ `submitTransferTask`（`rdma_transport.cpp:822`）→ `RdmaContext::submitPostSend` → CQ 完成 → `getTransferStatus`（`rdma_transport.cpp:998`）。

## 6.3 调用链图

```
registerLocalMemory (rdma_transport.cpp:276)
   │  registerLocalMemoryInternal (L284)
   │    ├─ 1. 计算 access_rights (IBV_ACCESS_REMOTE_WRITE/READ)
   │    ├─ 2. 按 max_mr_size 拆分大 buffer 成 chunks (Mooncake#2017)
   │    ├─ 3. getMemoryLocation 解析 location (cpu:N / gpu:N)
   │    └─ 4. 每个 chunk 在每个 context 上 ibv_reg_mr → 得到 lkey/rkey
   ▼
submitTransferTask (rdma_transport.cpp:822)
   │  ├─ 1. selectDevice 选本地 HCA + buffer
   │  ├─ 2. SliceLengthCalculator 把 request 拆成 slice
   │  ├─ 3. 每个 slice 绑定 source_lkey + dest_addr
   │  └─ 4. 按 context 聚合 → RdmaContext::submitPostSend (WR 下发)
   ▼
异步 RDMA 传输 (ibv_post_send)
   ▼
CQ 完成事件 → slice->markSuccess()/markFailed()
   ▼
getTransferStatus (rdma_transport.cpp:998)   ← 轮询聚合
   │  success + failed == slice_count → COMPLETED/FAILED
   ▼
is_finished = true → freeBatchID 可释放
```

## 6.4 三个关键代码片段

### 片段1：内存注册（`rdma_transport.cpp:284`）

`registerLocalMemoryInternal` 是 RDMA 数据路径的起点。它决定访问权限、按 `max_mr_size` 拆分大 buffer、解析 location，然后逐 chunk 注册 MR：

```cpp
// rdma_transport.cpp:289  访问权限
int access_rights = IBV_ACCESS_LOCAL_WRITE;
if (remote_accessible)
    access_rights |= IBV_ACCESS_REMOTE_WRITE | IBV_ACCESS_REMOTE_READ;

// rdma_transport.cpp:303  按 max_mr_size 拆分大 buffer
size_t chunk_limit = (size_t)globalConfig().max_mr_size;
if (chunk_limit > 0 && length > chunk_limit) {
    for (size_t offset = 0; offset < length;) {
        size_t chunk_len = std::min(chunk_limit, length - offset);
        chunks.emplace_back((char*)addr + offset, chunk_len);
        offset += chunk_len;
    }
    // Mooncake#2017: ibv_reg_mr 会静默截断到 max_mr_size，
    // 但元数据仍广播完整长度，导致远端 op 越界失败
}

// rdma_transport.cpp:321  解析 location
if (name == kWildcardLocation) {
    const std::vector<MemoryLocationEntry> entries =
        getMemoryLocation(addr, length, only_first_page);
    resolved_name = entries[0].location;   // ← cpu:N / gpu:N
}
```

**关键点**：`remote_accessible` 决定是否加 `IBV_ACCESS_REMOTE_WRITE/READ`——只有远端可访问的 buffer 才能被对端 RDMA 读写。**大 buffer 拆分**（Mooncake#2017）是重要 bugfix：`ibv_reg_mr` 会静默截断到设备 `max_mr_size`，但元数据仍广播完整长度，导致远端 op 越界报 `IBV_WC_REM_ACCESS_ERR`。拆成 chunk 后每个 chunk 一个 MR、一个 BufferDesc，地址区间查找能拿到正确的 key。

### 片段2：请求拆 slice + 绑定 key（`rdma_transport.cpp:822`）

`submitTransferTask` 把每个 request 拆成 slice，为每个 slice 选设备、绑定 `source_lkey` 和 `dest_addr`，按 context 聚合后下发：

```cpp
// rdma_transport.cpp:901  拆 slice
for (uint64_t offset = 0; offset < request.length;) {
    size_t slice_length = slice_calc.calculate(offset);   // ← 按 slice_size 切分
    Slice *slice = getSliceCache().allocate();
    slice->source_addr = (char*)request.source + offset;
    slice->length = slice_length;
    slice->opcode = request.opcode;
    slice->rdma.dest_addr = request.target_offset + offset;  // ← 远端地址
    slice->rdma.retry_cnt = request.advise_retry_cnt;
    slice->status = Slice::PENDING;
    task.slice_list.push_back(slice);
    ...
    // rdma_transport.cpp:970  绑定本地 lkey
    slice->rdma.source_lkey =
        local_segment_desc->buffers[buffer_id].lkey[device_id];
    slices_to_post[context].push_back(slice);   // ← 按 context 聚合
    __sync_fetch_and_add(&task.slice_count, 1);
    ...
    // rdma_transport.cpp:982  达到水位就下发，避免 WR 堆积
    if (nr_slices >= kSubmitWatermark) {
        for (auto &entry : slices_to_post)
            entry.first->submitPostSend(entry.second);
        slices_to_post.clear();
        nr_slices = 0;
    }
}
```

**关键点**：`source_lkey` 是本地 MR 的 key，`dest_addr` 是远端地址（由篇4 的 `Replica::Descriptor` 的 `buffer_address_` + `transport_endpoint_` 提供）。`kSubmitWatermark = max_wr * num_qp_per_ep` 控制 WR 下发节奏，防止一次提交过多 WR 撑爆 QP。`slice_count` 用原子自增，供 `getTransferStatus` 判断完成。

### 片段3：完成判断（`rdma_transport.cpp:998`）

`getTransferStatus` 读原子的 `success_slice_count`/`failed_slice_count`，与 `slice_count` 比较判断完成：

```cpp
// rdma_transport.cpp:1006  读原子计数
uint64_t success_slice_count =
    __atomic_load_n(&task.success_slice_count, __ATOMIC_ACQUIRE);
uint64_t failed_slice_count =
    __atomic_load_n(&task.failed_slice_count, __ATOMIC_ACQUIRE);
if (success_slice_count + failed_slice_count == task.slice_count) {
    if (failed_slice_count)
        status[task_id].s = TransferStatusEnum::FAILED;
    else
        status[task_id].s = TransferStatusEnum::COMPLETED;
    task.is_finished = true;   // ← 完成标记，freeBatchID 依赖
} else {
    status[task_id].s = TransferStatusEnum::WAITING;
}
```

**关键点**：`success/failed_slice_count` 由 CQ 完成事件回调（`slice->markSuccess()`/`markFailed()`）原子更新。`getTransferStatus` 只是**读聚合**——真正的完成驱动在 CQ 轮询线程。`is_finished` 置位后，篇5 的 `freeBatchID` 才允许释放。

## 6.5 完成语义：CQ 事件驱动 slice 状态

RDMA 的完成是**事件驱动**的：`submitPostSend` 下发 WR 后，硬件完成时在 CQ 产生完成事件，worker 线程轮询 CQ 并调用 `slice->markSuccess()`/`markFailed()`，原子更新 `success/failed_slice_count`。`getTransferStatus` 只是读这些计数判断任务是否收尾。

**关键点**：这与篇5 的 legacy 路径一致——`success + failed == slice_count` 是统一的终态判据。RDMA 路径的 slice 状态由 CQ 回调驱动，`getTransferStatus` 只做聚合，不主动触发完成。

## 6.6 异常与并发

### 大 buffer 越界（Mooncake#2017）

`ibv_reg_mr` 静默截断到 `max_mr_size`，元数据却广播完整长度，远端 op 越界报 `IBV_WC_REM_ACCESS_ERR`。修复：按 `max_mr_size` 拆 chunk，每 chunk 一个 MR + 一个 BufferDesc（`rdma_transport.cpp:303`）。`unregisterLocalMemory` 只收到 base addr，靠 `chunk_map_` 记住每个 base buffer 的 chunk 起始地址用于清理（`rdma_transport.h:159`）。

### 内存未注册

`selectDevice` 找不到任何 active 设备注册该地址时，`submitTransferTask` 返回 `AddressNotRegistered`（`rdma_transport.cpp:958`），并 `fail_task_and_cleanup` 把未启动的 task 也标记失败（`fail_unstarted_tasks`，`rdma_transport.cpp:845`）——保证 `success + failed == slice_count` 终态成立，任务能正常收尾。

### 设备不 active

选中的 context 不 active，返回 `InvalidArgument`（`rdma_transport.cpp:966`）。`selectDevice` 带 `retry_cnt` 重试，尝试其他设备。

### 部分 slice 失败

CQ 回调把失败 slice 的 `failed_slice_count` 加一。`getTransferStatus` 只要 `failed_slice_count > 0` 就置 `FAILED`（`rdma_transport.cpp:1011`），即使其余 slice 成功。

## 6.7 验证与结论

- **头文件**：`mooncake-transfer-engine/include/transport/rdma_transport/rdma_transport.h` 的 `registerLocalMemory`(L64)、`submitTransferTask`(L92)、`getTransferStatus`(L100)、`selectDevice`(L130)。
- **实现**：`mooncake-transfer-engine/src/transport/rdma_transport/rdma_transport.cpp` 的 `registerLocalMemoryInternal`(L284)、`submitTransferTask`(L822)、`getTransferStatus`(L998)、`onSetupRdmaConnections`(L1055)。
- **location 解析**：`mooncake-transfer-engine/include/memory_location.h` 的 `getMemoryLocation`(L37)、`MemoryLocationEntry`(L28)。
- **测试**：`mooncake-transfer-engine/tests/rdma_transport_test.cpp`、`rdma_loopback_test.cpp`、`rdma_large_mr_test.cpp`（大 MR 拆分）、`rdma_transport_submit_task_test.cpp`。

**结论**：RDMA 数据路径是**注册 → 拆 slice → 下发 WR → CQ 完成 → 聚合**五步。`registerLocalMemory` 决定访问权限、按 `max_mr_size` 拆大 buffer、解析 location 后逐 chunk 注册 MR；`submitTransferTask` 把 request 拆成 slice、绑定 `source_lkey`/`dest_addr`、按 context 聚合下发；CQ 完成事件驱动 slice 状态，`getTransferStatus` 读原子计数判断终态。**大 buffer 拆分**（Mooncake#2017）和**未注册地址的失败收尾**是两个关键异常处理。下一篇看多网卡与 NUMA 如何选路径。下一篇看多网卡与 NUMA。

---

*下一篇：篇7 多网卡与 NUMA——数据传输路径如何选择？*
