# 篇1 源码导航：一次 KV Cache 读写经过哪些组件？

> 系列：《Mooncake KV Cache 源码解析》
> 版本锚点：`v0.3.13.post1`（commit `7197358`）
> 源码链接基座：`https://github.com/kvcache-ai/Mooncake/blob/v0.3.13.post1/`

## 1.1 具体问题

一个推理实例生成了 KV Cache，另一个实例怎样找到并读取它？这一篇沿最小 Put/Get 路径，把篇0 的全局地图跑通一遍，建立三个认识：

1. **对象标识**：key 是什么，如何构造。
2. **元数据与数据分路**：哪些调用走 Master RPC，哪些走 Transfer Engine。
3. **完成判断**：Put/Get 如何判断成功。

## 1.2 场景与版本

- **场景**：两个进程（或两个推理实例）通过 Mooncake Store 共享 KV Cache。进程 A Put 一个对象，进程 B Get 它。
- **版本**：`v0.3.13.post1`。注意：本版本中 Python 绑定位于 `mooncake-integration/store/store_py.cpp`，`python/mooncake/` 下只有 `__init__.py`（Python 包由 `mooncake-integration` 构建），计划中提到的 `cli_client.py`/`cli.py` 在本版本已不存在。
- **最小示例**：`mooncake-store/rust/examples/basic_usage.rs` 提供了最干净的 Put/Get 全流程。

## 1.3 调用链图

```
Python 客户端 (store_py.cpp)
   │  put_tensor_from / get_tensor_into
   ▼
RealClient (real_client.h / real_client.cpp)
   │  put_from / get_into / batch_query
   ├───────────────┬──────────────────────────────┐
   ▼               ▼                              ▼
Master RPC       Transfer Engine              本地 buffer
(PutStart/       (submitTransfer)             (split_into_slices)
 PutEnd/          │
 GetReplicaList)  ▼
   │          远端 Client 内存 (RDMA/TCP)
   ▼
Master Service (master_service.cpp)
```

## 1.4 三个关键代码片段

### 片段1：最小 Put/Get 全流程（`basic_usage.rs`）

Rust 示例展示了最干净的调用序列：`setup` → `put` → `is_exist` → `get_size` → `get` → `remove`。

```rust
// basic_usage.rs:75  setup：连接 metadata server + 传输层
store.setup("localhost", &metadata_server,
            512 << 20,  // global_segment_size = 512 MiB
            128 << 20,  // local_buffer_size   = 128 MiB
            "tcp", "", "127.0.0.1:50051");

// basic_usage.rs:97  put：把数据拷入 store
store.put(key, value, None);

// basic_usage.rs:120  get：读回数据
match store.get(key) {
    Ok(data) => assert_eq!(data, value, "round-trip mismatch!"),
    ...
}
```

`setup` 的参数与 `RealClient::setup_real`（`real_client.h:85`）一一对应：`global_segment_size` 决定贡献给集群的内存，`local_buffer_size` 决定本地暂存 buffer，`protocol`/`device_name` 决定传输后端，`master_server_addr` 指向 Master。

### 片段2：Python 绑定 → RealClient（`store_py.cpp`）

Python 的 `get_tensor_into` 释放 GIL 后直接调用 `store_->get_into`：

```cpp
// store_py.cpp:662  get_tensor_into
int64_t total_length;
{
    py::gil_scoped_release release_gil;   // 释放 GIL，避免阻塞 Python 线程
    total_length = store_->get_into(key, buffer, size);
}
return buffer_to_tensor(NULL, buffer, total_length);
```

`put_tensor_from` 同样释放 GIL 后调用 `store_->put_from`（`store_py.cpp:1226`）。**释放 GIL 是关键**：KV Cache 传输是长耗时操作，不能占住 Python 解释器。

### 片段3：RealClient 的 put/get 实现（`real_client.cpp`）

`put_from` 把用户 buffer 切成 slice，然后调用 `client_->Put`（底层 PyClient 的 Put，内部走 Master RPC + TE）：

```cpp
// real_client.cpp:4189  put_from_internal
std::vector<mooncake::Slice> slices = split_into_slices(buffer, size);
auto put_result = client_->Put(key, slices, config);   // 内部：PutStart → TE 传输 → PutEnd
if (!put_result) return tl::unexpected(put_result.error());
return {};
```

`get_into` 走 `get_into_range_internal` → `execute_ranged_read`，先 `batch_query` 拿元数据，再选副本、TE 读取：

```cpp
// real_client.cpp:3758  get_into
return get_into_range_internal(key, buffer, 0, 0, size, true, true);

// real_client.cpp:4033  batch_query → 底层 BatchQuery（Master RPC）
return client_->BatchQuery(keys);
```

## 1.5 三个认识

### 认识1：对象标识——key 是不透明字符串

Mooncake Store 把 key 当作**不透明字符串**（`docs/source/design/store/mooncake-store.md`）。key 由推理框架构造，因此框架决定了哪些请求/实例能复用同一份 KV Cache。vLLM 场景下，key 由模型名 + `cache_prefix` + KV block hash 拼成（详见篇10）。隔离边界（模型/发布/请求组/tenant）见 `docs/source/deployment/kv-cache-sharing-and-isolation.md`。

### 认识2：元数据与数据分路

- **元数据路径**：`client_->Put`/`BatchQuery` 内部走 Master RPC（`PutStart`/`PutEnd`/`GetReplicaList`）。Master 只返回"数据在哪"（segment + offset + size），不碰数据。
- **数据路径**：`client_->Put` 拿到副本位置后，通过 Transfer Engine 把数据从本机 buffer 传到远端 Client 内存。Master 完全不参与。

### 认识3：完成判断

- **Put**：`client_->Put` 返回 `tl::expected<void, ErrorCode>`，`put_from` 用 `to_py_ret` 转成 Python 整数（0 成功，负值错误）。Put 成功 = 数据已写入所有要求的副本 + Master `PutEnd` 已提交（对象可见）。
- **Get**：`get_into` 返回 `int64_t`，成功为读取的字节数，负值为错误码。Get 成功 = 数据已完整读入本地 buffer。

## 1.6 异常与并发

- **对象不存在**：`get_size("nonexistent")` 返回 `OperationFailed(code)`（`basic_usage.rs:154`）。
- **副本未就绪**：`build_ranged_read_metadata_from_query_result`（`real_client.cpp:4043`）对 `OBJECT_NOT_FOUND`/`REPLICA_IS_NOT_READY` 直接透传错误；`SelectBestReplica` 找不到可用副本时返回 `INVALID_REPLICA`。
- **GIL 与并发**：Python 绑定释放 GIL 后调用 C++，多个 Python 线程可并发发起传输。

## 1.7 验证与结论

- **最小路径**：`mooncake-store/rust/examples/basic_usage.rs` 的 setup/put/get/remove 全流程。
- **Python 绑定**：`mooncake-integration/store/store_py.cpp` 的 `get_tensor_into`(L662)、`put_tensor_from`(L1226)、`batch_get_tensor`(L641)。
- **RealClient**：`mooncake-store/src/real_client.cpp` 的 `put_from`(L4219)、`get_into`(L3758)、`batch_query`(L4033)。
- **设计文档**：`docs/source/design/store/mooncake-store.md` 的 Master/Client 双角色说明。

**结论**：一次 KV Cache 读写 = **Python 绑定（释放 GIL）→ RealClient → 元数据走 Master RPC + 数据走 Transfer Engine → 远端 Client 内存**。key 是不透明字符串，由框架构造；元数据与数据严格分路；Put 成功即对象可见，Get 成功即数据完整读回。下一篇深入 Put 的可见性语义。

---

*下一篇：篇2 一次 Put——缓存对象如何从客户端写入分布式内存？*
