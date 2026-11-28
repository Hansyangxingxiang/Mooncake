# Mooncake 报 REPLICA_IS_NOT_READY，该从哪查起？

**摘要：**推理实例 B 查询 key `K` 收到 `REPLICA_IS_NOT_READY`，说明在这个查询时刻 K 的元数据存在，但没有任何副本通过 `IsReplicaReadable`。副本可能仍在 `PROCESSING`，也可能句柄或传输端点已经失效；错误码本身分不出是哪一种，要沿查询路径逐站查证据。

推理实例 B 查询 key `K`，Master 返回 `REPLICA_IS_NOT_READY`。

这个错误容易被直接理解成“对象不存在”或“再等等就好”。但同一个返回码背后，可能是副本还在 `PROCESSING`，也可能是句柄或传输端点失效。只看一行错误日志，分不出是哪一种。

第 3 篇已经讲过写入可见性：`PutStart` 之后、`PutEnd` 之前，副本停在 `PROCESSING`。这里不再重复系统解释，只跟 B 的一次查询走一遍源码，演示怎样从入口走到具体分支，以及每一站应该拿什么证据。

## 第一站：查询入口只把请求交给 Master

B 的查询入口是 [`Client::Query`](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/src/client_service.cpp#L1211)。它在发出 RPC 之前取一次本地 `steady_clock` 起点，然后调用 `master_client_.GetReplicaList(object_key)`，把返回的副本列表、读租约（lease）截止时间和 `object_checksum` 组装成 `QueryResult`。

这一站要看的证据是 RPC 返回状态。如果 Master 返回 `REPLICA_IS_NOT_READY`，`Client::Query` 不会在本地把它改写成别的错误，也不会在这里替调用方选副本。

因此，看到这个错误时可以先排除一种解释：它不是客户端本地缓存或副本选择阶段产生的，源头在 Master 的查询结果。

## 第二站：Master 先判断 key 是否存在

进入 [`MasterService::GetReplicaList`](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/src/master_service.cpp#L3626)，第一道判断是 `accessor.Exists()`。key 不存在时，Master 直接返回 `OBJECT_NOT_FOUND`。

所以，如果 B 收到的是 `REPLICA_IS_NOT_READY`，在这个查询时刻，K 的 `ObjectMetadata` 是存在的。至少可以排除“这个 key 从未写入过”。

这里要注意范围：`OBJECT_NOT_FOUND` 也不总等于“从未存在”。对象可能曾经存在，但所有副本都已经被删除或移除；这条分支在第四站会再出现一次。

## 第三站：可读副本是过滤出来的

key 存在之后，`GetReplicaList` 用 `VisitReplicas` 遍历副本，只把通过 [`IsReplicaReadable`](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/src/master_service.cpp#L3546) 检查的副本放进结果列表。

`IsReplicaReadable` 至少要求：

- 副本状态是 `COMPLETE`；
- 内存或 NoF 句柄仍然有效；
- 传输端点没有被标记为失效。

如果 R1 是在 `PutStart` 之后、`PutEnd` 之前被查询，它已经分配了地址，也挂在 K 的元数据下面，但状态仍是 `PROCESSING`。第一项检查就不通过。

这一站能排除“对象根本没有副本”：副本结构已经存在，只是不满足可读条件。接下来还要看，过滤后的列表为什么没有变成另一种错误。

## 第四站：空列表最后分成两种错误

遍历结束后，如果没有选出任何可读副本，`GetReplicaList` 会再判断一次：

- 所有副本都是 `REMOVED`：返回 `OBJECT_NOT_FOUND`；
- 否则：记录 `replica_not_ready` 日志，返回 `REPLICA_IS_NOT_READY`。

因此，`REPLICA_IS_NOT_READY` 还排除了“所有副本都已移除”。它表示列表为空，但至少有一个副本不是 `REMOVED`——可能仍在 `PROCESSING`，也可能句柄或传输端点已经失效。

把这条查询路径压成判断顺序，就是：

1. key 不存在 → `OBJECT_NOT_FOUND`；
2. key 存在，且存在通过 `IsReplicaReadable` 的副本 → 返回可读副本列表与读租约；
3. key 存在，没有可读副本，且所有副本都是 `REMOVED` → `OBJECT_NOT_FOUND`；
4. key 存在，没有可读副本，但不是全部 `REMOVED` → `REPLICA_IS_NOT_READY`。

第 1 条对应第二站，第 2 条对应第三站，第 3、4 条对应第四站。

## 用测试确认 `PROCESSING` 这一种情况

基线 `v0.3.13.post1` 的 [`master_service_test.cpp`](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/tests/master_service_test.cpp#L4041) 中，`SingleSliceMultiReplicaFlow` 用例把这条路径固定下来：

| 操作 | 测试实际调用 | 直接观察到的结果 |
|---|---|---|
| 申请 | `PutStart` | 3 个副本，均为 `PROCESSING` |
| 提交前查询 | `GetReplicaList` | 返回 `REPLICA_IS_NOT_READY` |
| 提交 | `PutEnd(MEMORY)` | 返回成功 |
| 提交后查询 | `GetReplicaList` | 3 个副本，均为 `COMPLETE` |

四行里，测试直接断言的是副本状态、`PutStart` 返回的 slice 长度、查询错误码和提交后的副本列表。它没有执行字节传输，也没有验证读回的数据内容。因此，它证明的是“分配完成、`PutEnd` 之前查不到可读副本”这一种 `PROCESSING` 来源；句柄失效、传输端点失效等分支仍要靠第三站的条件和运行证据判断。

## 总结

遇到 `REPLICA_IS_NOT_READY`，按这个顺序查：

1. 先确认 K 的 `ObjectMetadata` 是否存在——存在，才轮得到副本状态；
2. 再看各副本是 `PROCESSING`、`COMPLETE` 还是 `REMOVED`，以及是否存在通过 `IsReplicaReadable` 的副本；
3. 最后核对内存/NoF 句柄与传输端点是否有效。

这三步分别对应第二站、第三站和第四站。测试只覆盖 `PROCESSING` 这一种来源；句柄失效、传输端点失效等分支需要对应的运行证据，不能只凭错误码判断。

**源码延伸：**`Client::Query`、`GetReplicaList`、`IsReplicaReadable`、以及 `master_service_test.cpp` 中的 `PROCESSING` 用例。

- [Client::Query](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/src/client_service.cpp#L1211)
- [MasterService::GetReplicaList](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/src/master_service.cpp#L3626) · [IsReplicaReadable](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/src/master_service.cpp#L3546)
- [PROCESSING 副本的测试用例](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/tests/master_service_test.cpp#L4041)

[系列目录与排版说明](README.md)

[上一篇](04-get-and-lease.md) · [下一篇](06-transfer-lifecycle.md)
