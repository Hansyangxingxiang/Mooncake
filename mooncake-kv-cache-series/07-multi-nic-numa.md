# 篇7 多网卡与 NUMA：数据传输路径如何选择？

> 系列：《Mooncake KV Cache 源码解析》
> 版本锚点：`v0.3.13.post1`（commit `7197358`）
> 源码链接基座：`https://github.com/kvcache-ai/Mooncake/blob/v0.3.13.post1/`

## 7.1 具体问题

一台机器往往有多张网卡（HCA），内存分布在多个 NUMA 节点。篇6 的 `selectDevice` 选 HCA，这一篇回答三个问题：

1. **发现**：拓扑矩阵如何生成——每个存储位置（cpu:N / gpu:N）该用哪张 HCA？
2. **选择**：传输时如何按内存地址选 preferred/secondary NIC？
3. **降级**：首选 NIC 失败后如何处理？

## 7.2 场景与版本

- **场景**：多网卡 + 多 NUMA 节点的高性能节点上，TE 需要为每个内存地址选最优 HCA，避免跨 NUMA 访问。
- **版本**：`v0.3.13.post1`。注意：本版本拓扑逻辑在 `src/topology.cpp`（`include/topology.h`），`multi_transport_locality.h` 提供主机/段名解析辅助。
- **主路径**：`Topology::discover`（`topology.cpp:625`）→ `Topology::resolve`（`topology.cpp:794`）→ `Topology::selectDevice`（`topology.cpp:761`）→ `RdmaTransport::selectDevice`（`rdma_transport.cpp:1155`）。

## 7.3 调用链图

```
Topology::discover (topology.cpp:625)
   │  ├─ listInfiniBandDevices  ← 枚举 HCA，读 PCI bus id + numa_node
   │  ├─ discoverCpuTopology    ← 每个 cpu:N 节点，同 NUMA 的 HCA 为 preferred
   │  └─ discoverCudaTopology   ← 每个 gpu:N，按 PCI 距离选 preferred HCA
   ▼
Topology::resolve (topology.cpp:794)
   │  把 HCA 名映射成稳定索引，构建 resolved_matrix_
   │  + precomputeResolvedHcaPeerAffinity (HCA 对端亲和)
   ▼
Topology::selectDevice (topology.cpp:761)
   │  ├─ retry_count==0 → 随机/轮询选 preferred_hca
   │  └─ retry_count>0  → 按 (retry-1) 索引遍历 preferred + avail
   ▼
RdmaTransport::selectDevice (rdma_transport.cpp:1155)
   │  按内存地址 + 目标段选 buffer_id + device_id
   ▼
submitTransferTask → 用选中的 device 下发 WR
```

## 7.4 三个关键代码片段

### 片段1：拓扑发现（`topology.cpp:625`）

`discover` 枚举 HCA，为每个存储位置（cpu:N / gpu:N）计算 preferred/avail HCA。CPU 拓扑按 NUMA 节点归类，GPU 拓扑按 PCI 距离：

```cpp
// topology.cpp:473  CPU 拓扑：同 NUMA 节点的 HCA 是 preferred
for (const auto &hca : all_hca) {
    if (hca.numa_node == node_id) {
        preferred_hca.push_back(hca.name);   // ← 同 NUMA，优先
    } else {
        avail_hca.push_back(hca.name);
    }
}
topology.push_back(TopologyEntry{
    .name = "cpu:" + std::to_string(node_id),
    .preferred_hca = std::move(preferred_hca),
    .avail_hca = std::move(avail_hca)});

// topology.cpp:566  GPU 拓扑：按 PCI 距离选 preferred HCA
for (const auto &hca : candidate_preferred_hca) {
    int distance = getPciDistance(hca.pci_bus_id.c_str(), pci_bus_id);
    if (distance < min_distance) {
        min_distance = distance;
        min_distance_hcas.clear();
        min_distance_hcas.push_back(hca.name);
    } else if (distance == min_distance) {
        min_distance_hcas.push_back(hca.name);
    }
}
```

**关键点**：`getPciDistance`（`topology.cpp:494`）比较两个 PCI bus id 的公共前缀长度，公共前缀越长距离越近。GPU 拓扑优先选**同 NUMA 且 PCI 距离最近**的 HCA——这是避免跨 NUMA 访问的核心。`MC_TE_FILTERS` 环境变量可做 HCA 白名单（`topology.cpp:161`）。

### 片段2：HCA 索引解析（`topology.cpp:794`）

`resolve` 把 HCA 名映射成**稳定索引**，构建 `resolved_matrix_`。索引稳定性至关重要——RDMA transport 按索引存 lkey/rkey 和 context 数组：

```cpp
// topology.cpp:800  为每个 HCA 分配稳定索引
for (auto &hca : entry.second.preferred_hca) {
    if (!hca_id_map.count(hca)) {
        hca_list_.push_back(hca);
        hca_id_map[hca] = next_hca_map_index++;   // ← 稳定索引
        resolved_matrix_[kWildcardLocation].preferred_hca.push_back(
            hca_id_map[hca]);
    }
    resolved_matrix_[entry.first].preferred_hca.push_back(hca_id_map[hca]);
    ...
}
```

**关键点**：`disableDevice`（`topology.cpp:836`）注释明确说明——**必须保持 HCA 索引稳定**，因为 RDMA transport 的 lkey/rkey 和 context 数组按 resolved HCA 索引存储。若重新 `resolve()` 压缩索引，这些数组会指向错误的 RNIC。所以 `disableDevice` 只从各 entry 里删索引，不重排。

### 片段3：设备选择（`topology.cpp:761`）

`selectDevice` 按 `retry_count` 区分：首次随机/轮询选 preferred，重试时按索引遍历 preferred + avail：

```cpp
// topology.cpp:769  首次：随机或轮询选 preferred
if (retry_count == 0) {
    int rand_value;
    if (use_round_robin_) {   // MC_PATH_ROUNDROBIN=1 时轮询
        thread_local int tl_counter = 0;
        rand_value = tl_counter;
        tl_counter = (tl_counter + 1) % 10000;
    } else
        rand_value = SimpleRandom::Get().next();
    if (!entry.preferred_hca.empty())
        return entry.preferred_hca[rand_value % entry.preferred_hca.size()];
    else
        return entry.avail_hca[rand_value % entry.avail_hca.size()];
} else {
    // 重试：按 (retry-1) 索引遍历 preferred + avail
    size_t index = (retry_count - 1) %
                   (entry.preferred_hca.size() + entry.avail_hca.size());
    if (index < entry.preferred_hca.size())
        return entry.preferred_hca[index];
    else {
        index -= entry.preferred_hca.size();
        return entry.avail_hca[index];
    }
}
```

**关键点**：`selectDeviceByLocalHca`（`topology.cpp:731`）支持 HCA 对端亲和——本地 HCA 与远端 GPU 的 preferred HCA 有映射时，优先选对端可达的 HCA（`precomputeResolvedHcaPeerAffinity`，`topology.cpp:304`）。`MC_ENABLE_HCA_PEER_AFFINITY` + `MC_NIC_PEER_AFFINITY` 配置。

## 7.5 完成语义：路径选择是"尽力而为"

路径选择是**尽力而为**的：`selectDevice` 返回一个 HCA 索引，但**不保证该 HCA 一定可用**。`RdmaTransport::selectDevice`（`rdma_transport.cpp:1155`）在 `submitTransferTask` 里被调用，若选中的 context 不 active，会带 `retry_cnt` 重试其他设备（见篇6 片段2 的 `while (retry_cnt < kMaxRetryCount && !found_device)` 循环）。

**关键点**：拓扑矩阵是**静态发现**的（启动时 `discover`），但设备可用性是**动态**的。`selectDevice` 只负责"选最优"，`submitTransferTask` 负责"验证可用 + 重试降级"。两者配合实现"首选失败 → 降级到 secondary"。

## 7.6 异常与并发

### 设备不可用

`isIbDeviceAvailable`（`topology.cpp:102`）检查设备可访问、有 active port。`discover` 跳过不可用设备。但运行中设备可能失效——`submitTransferTask` 里 `context->active()` 检查 + `retry_cnt` 重试（见篇6）。

### 无可用设备

`selectDevice` 找不到任何 HCA 返回 `ERR_DEVICE_NOT_FOUND`（`topology.cpp:762`）。`submitTransferTask` 对未注册地址返回 `AddressNotRegistered`（见篇6）。

### 索引稳定性

`disableDevice` 保持 HCA 索引稳定（`topology.cpp:858` 注释）。若破坏稳定性，RDMA transport 的 lkey/rkey 数组会指向错误 RNIC，导致数据写到错误网卡。

### 跨主机判定

`multi_transport_locality.h` 的 `isGpuIpcReachableTarget`（L72）判断目标是否同主机——GPU IPC transport（HIP/MUSA）只在本机进程间工作，跨主机必须回退 RDMA/TCP。`segmentHost`（L32）正确处理 IPv4/IPv6/带端口段名。

## 7.7 验证与结论

- **头文件**：`mooncake-transfer-engine/include/topology.h` 的 `Topology`(L64)、`TopologyEntry`(L38)、`selectDevice`(L92)、`selectDeviceByLocalHca`(L95)。
- **实现**：`mooncake-transfer-engine/src/topology.cpp` 的 `discover`(L625)、`resolve`(L794)、`selectDevice`(L761)、`selectDeviceByLocalHca`(L731)、`disableDevice`(L836)、`getPciDistance`(L494)。
- **locality**：`mooncake-transfer-engine/include/multi_transport_locality.h` 的 `segmentHost`(L32)、`isGpuIpcReachableTarget`(L72)。
- **测试**：`mooncake-transfer-engine/tests/topology_test.cpp`、`multi_transport_locality_test.cpp`、`show_links_test.cpp`。

**结论**：多网卡与 NUMA 的路径选择是**发现 → 解析 → 选择 → 降级**四步。`discover` 按 NUMA 节点和 PCI 距离为每个存储位置计算 preferred/avail HCA，`resolve` 把 HCA 名映射成稳定索引（RDMA transport 依赖索引稳定性），`selectDevice` 首次随机/轮询选 preferred、重试时遍历降级，`submitTransferTask` 验证可用性并重试。**路径选择是尽力而为**，静态拓扑 + 动态可用性检查配合实现首选失败降级。下一篇回到 Store 侧，看缓存生命周期。下一篇回到 Store 侧。

---

*下一篇：篇8 缓存生命周期——何时可读，何时淘汰，谁负责删除？*
