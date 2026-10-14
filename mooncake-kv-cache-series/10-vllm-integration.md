# 篇10 接入 vLLM：一次前缀命中如何减少 Prefill？

> 系列：《Mooncake KV Cache 源码解析》
> 版本锚点：`v0.3.13.post1`（commit `7197358`）
> 源码链接基座：`https://github.com/kvcache-ai/Mooncake/blob/v0.3.13.post1/`

## 10.1 具体问题

前九篇都在 Mooncake 内部。这一篇从框架入口追到 Store/TE，再回到请求执行，回答三个问题：

1. **接入**：vLLM 如何通过 connector 接入 Mooncake？
2. **命中**：一次前缀命中如何从 Store 读回 KV Cache，跳过 Prefill？
3. **端到端**：从请求到响应，KV Cache 走完哪些阶段？

## 10.2 场景与版本

- **场景**：vLLM 用 `MooncakeConnector`（PD 分离，RDMA 直传）或 `MooncakeStoreConnector`（共享 KV 池，CPU/SSD 卸载 + 前缀缓存）接入 Mooncake。
- **版本**：`v0.3.13.post1`。注意：本版本 vLLM 集成文档已合并到 `docs/source/deployment/integrations/vllm/disagg-prefill-decode.md`（`vllm-integration-v1.0.md` 已归档为 orphan）。Python 绑定在 `mooncake-integration/store/store_py.cpp`。
- **主路径**：vLLM connector → `store_py.cpp`（`get_tensor`/`put_tensor`）→ `real_client` → Master + TE → 前缀命中 → 跳过 Prefill。

## 10.3 调用链图

```
vLLM 请求 (proxy → prefill/decode 实例)
   │  kv_connector: MooncakeConnector / MooncakeStoreConnector
   ▼
MooncakeStoreConnector (vLLM 侧 KV connector)
   │  get_tensor / put_tensor
   ▼
store_py.cpp (mooncake-integration/store/store_py.cpp)
   │  get_tensor_into (L662) / put_tensor_from (L1226)   ← 释放 GIL
   ▼
RealClient (real_client.cpp)
   │  get_into / put_from
   ├───────────────┬──────────────────────────────┐
   ▼               ▼                              ▼
Master RPC       Transfer Engine              本地 buffer
(PutStart/       (submitTransfer)             (split_into_slices)
 PutEnd/
 GetReplicaList)
   ▼
远端 Client 内存 (RDMA/TCP)
   ▼
前缀命中 → 跳过 Prefill → 直接 Decode
```

## 10.4 三个关键代码片段

### 片段1：PD 分离部署（`disagg-prefill-decode.md:51`）

Prefill 节点生成 KV Cache（`kv_producer`），Decode 节点消费（`kv_consumer`），通过 RDMA 直传：

```bash
# disagg-prefill-decode.md:54  Prefiller 节点
vllm serve Qwen/Qwen2.5-7B-Instruct \
  --port 8010 \
  --kv-transfer-config '{"kv_connector":"MooncakeConnector","kv_role":"kv_producer"}'

# disagg-prefill-decode.md:62  Decoder 节点
vllm serve Qwen/Qwen2.5-7B-Instruct \
  --port 8020 \
  --kv-transfer-config '{"kv_connector":"MooncakeConnector","kv_role":"kv_consumer"}'
```

**关键点**：`kv_role` 决定实例角色——`kv_producer`（Prefill 生成 KV）、`kv_consumer`（Decode 消费 KV）、`kv_both`（对称，实验性）。`MooncakeConnector` 用 RDMA 直传，`MooncakeStoreConnector` 用共享 KV 池（支持 CPU/SSD 卸载 + 前缀缓存）。`VLLM_MOONCAKE_BOOTSTRAP_PORT`（默认 8998）为每个 worker 提供唯一端口（`disagg-prefill-decode.md:129`）。

### 片段2：Python 绑定释放 GIL（`store_py.cpp:662`）

`get_tensor_into` 释放 GIL 后调用 `store_->get_into`，避免长耗时传输阻塞 Python 线程：

```cpp
// store_py.cpp:662  get_tensor_into
int64_t total_length;
{
    py::gil_scoped_release release_gil;   // ← 释放 GIL
    total_length = store_->get_into(key, buffer, size);
}
return buffer_to_tensor(NULL, buffer, total_length);
```

**关键点**：KV Cache 传输是长耗时操作，释放 GIL 让多个 Python 线程可并发发起传输。`put_tensor_from`（`store_py.cpp:1226`）同样释放 GIL。这是 vLLM 高并发推理的关键——不占住解释器。

### 片段3：前缀缓存（`kv-cache-storage.md:34`）

`MooncakeStoreConnector` 用**块哈希去重**实现跨实例前缀缓存：

```markdown
# kv-cache-storage.md:34  前缀缓存能力
- **CPU/Disk offloading**: Extend effective KV cache capacity by offloading
  to CPU memory or SSD via Mooncake's transfer engine.
- **Hash-based prefix caching across instances**: Multiple vLLM instances
  share cached KV blocks through the store using block-hash deduplication.
```

**关键点**：`kv-cache-storage.md:143` 强调——DP 部署时需设固定 `PYTHONHASHSEED`，否则相同 prompt 在不同 DP rank 产生不同块哈希，无法跨实例前缀命中。**块哈希是前缀命中的 key**（见篇1 的 key 构造：模型名 + `cache_prefix` + KV block hash）。

## 10.5 完成语义：前缀命中 = 跳过 Prefill

回答开篇问题，一次前缀命中的端到端流程：

1. **请求到达**：proxy 把请求路由到 Prefill 实例（`max_tokens=1` 只做 Prefill）或 Decode 实例。
2. **KV 生成**：Prefill 实例生成 KV Cache，`put_tensor_from` 写入 Store（`PutStart` → TE 传输 → `PutEnd`，见篇2）。
3. **前缀命中**：Decode 实例收到请求，计算块哈希，`get_tensor_into` 从 Store 读回已缓存的 KV 块（`batch_query` 拿副本 → `SelectBestReplica` → TE 读取，见篇3）。
4. **跳过 Prefill**：命中的前缀 KV 块直接复用，Decode 只对未命中的后缀做增量计算。

**关键点**：**前缀命中 = 跳过 Prefill**。命中的 KV 块从 Store 读回，无需重新计算 attention 的 K/V。`MooncakeStoreConnector` 的块哈希去重让多个 vLLM 实例共享缓存，`MooncakeConnector` 的 RDMA 直传让 PD 分离下 KV 传输带宽达 142.25 GB/s（`disagg-prefill-decode.md:5`）。

## 10.6 异常与并发

### 块哈希不一致

DP 部署未设固定 `PYTHONHASHSEED`，相同 prompt 产生不同块哈希，前缀无法命中（`kv-cache-storage.md:143`）。修复：设 `PYTHONHASHSEED=0`。

### 实例崩溃

`kv-cache-storage.md:25` 强调 v1 的容错——单实例崩溃可容忍，实例间连接已移除，每个实例作为独立 vLLM 实例可处理请求。但 V0 下实例异常退出会污染连接元数据，需重启 `mooncake_master`（`kv-cache-storage.md:312`）。

### 端口冲突

同节点跑 prefill 和 decode，`decode_url` 端口需与 `prefill_url` 至少差 50（`disagg-prefill-decode.md:263`）。`VLLM_MOONCAKE_BOOTSTRAP_PORT` 每个 worker 唯一。

### 请求中止

`VLLM_MOONCAKE_ABORT_REQUEST_TIMEOUT`（默认 480s）在请求中止时自动释放 KV Cache，防止资源无限占用（`disagg-prefill-decode.md:133`）。

## 10.7 验证与结论

- **PD 分离**：`docs/source/deployment/integrations/vllm/disagg-prefill-decode.md`（`vllm-integration-v1.0.md` 已归档）。
- **KV 存储**：`docs/source/deployment/integrations/vllm/kv-cache-storage.md` 的 `MooncakeStoreConnector`、块哈希前缀缓存、XpYd 动态调整。
- **Python 绑定**：`mooncake-integration/store/store_py.cpp` 的 `get_tensor_into`(L662)、`put_tensor_from`(L1226)、`batch_get_tensor`(L641)。
- **测试**：`mooncake-integration/tests/`、`python/tests/integration/`。

**结论**：vLLM 通过 connector 接入 Mooncake——`MooncakeConnector` 做 PD 分离 RDMA 直传，`MooncakeStoreConnector` 做共享 KV 池 + 块哈希前缀缓存。Python 绑定释放 GIL 后调用 `real_client`，元数据走 Master RPC、数据走 TE。**前缀命中 = 跳过 Prefill**：命中的 KV 块从 Store 读回直接复用，块哈希是命中的 key。至此，从 Store 到 TE 到推理请求的完整链路已串通。

---

*系列完。从篇0 架构总览到篇10 接入 vLLM，11 篇覆盖 Mooncake KV Cache 的完整路径。*
