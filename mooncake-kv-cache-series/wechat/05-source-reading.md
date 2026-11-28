# 源码阅读番外｜Mooncake 报 REPLICA_IS_NOT_READY，该从哪查起？

**摘要：** 推理实例 B 查询系列里那个 4 MiB 的示意对象 `K`，收到 `REPLICA_IS_NOT_READY`。在 `v0.3.13.post1` 的单 key 查询路径上，这表示对象元数据存在、可读副本为空，而且还有副本尚未标记移除。要找出具体原因，得看副本状态、句柄和端点。

实例 A 为 `K` 申请了内存副本 R1。实例 B 随后查询 `K`，Master 返回 `REPLICA_IS_NOT_READY`。该继续等 A 调用 `PutEnd`，还是去查失效的副本？

主线第 3 篇讲的是 `PutStart` 之后、`PutEnd` 之前的写入窗口。这个番外沿 B 的单 key 查询追到实际分支，看看哪些证据能把写入中和副本失效区分开。 [主线第 3 篇 · Put 可见性](03-put-visibility.md) · [主线第 4 篇 · Get 和读租约](04-get-and-lease.md)

## 第一站：查询入口只把请求交给 Master

B 的查询入口是 `Client::Query`。它在调用 `master_client_.GetReplicaList(object_key)` 前记录本地 `steady_clock` 起点。RPC 成功时，客户端以起点加上返回的租约时长得到截止时间，再把它与副本列表、`object_checksum` 一起放进 `QueryResult`。

这一站记录 RPC 返回状态。Master 返回 `REPLICA_IS_NOT_READY` 时，`Client::Query` 原样返回错误，调用方还没有拿到可供选择的副本。

## 第二站：Master 先判断 key 是否存在

进入 `MasterService::GetReplicaList`，Master 按请求的 key 和租户定位对象。第一道判断是 `accessor.Exists()`。对象不存在时，Master 直接返回 `OBJECT_NOT_FOUND`。

B 收到 `REPLICA_IS_NOT_READY`，说明查询当时 Master 已有 `K` 的 `ObjectMetadata`。要判断 A 是否调用了 `PutEnd`，还要看副本状态和 `PutEnd` 的结果。如果稍后重查时返回 `OBJECT_NOT_FOUND`，还要核对两次请求的租户和时间点，以及这期间是否有副本移除。

## 第三站：可读副本是过滤出来的

找到对象后，`GetReplicaList` 用 `VisitReplicas` 遍历副本，只把通过 `IsReplicaReadable` 检查的副本放进结果列表。

`IsReplicaReadable` 至少要求：

- 副本状态是 `COMPLETE`；
- 内存或 NoF 副本的相应句柄有效；
- 对于读取端点字段的内存、NoF 或本地磁盘副本，其端点未进入失效集合。

如果 R1 是在 `PutStart` 之后、`PutEnd` 之前被查询，它已经分配了地址，也挂在 K 的元数据下面，但状态仍是 `PROCESSING`。第一项检查就不通过。

如果已经看到 R1 的副本状态为 `COMPLETE`，下一步就核对 Master 当时记录的句柄和端点状态。

## 第四站：空列表最后分成两种错误

遍历结束后，如果没有选出任何可读副本，`GetReplicaList` 会再判断一次：

- 所有副本都是 `REMOVED`：返回 `OBJECT_NOT_FOUND`；
- 否则：记录 `replica_not_ready` 日志，返回 `REPLICA_IS_NOT_READY`。

因此，`REPLICA_IS_NOT_READY` 还排除了“所有副本都已移除”。它表示列表为空，但至少有一个副本不是 `REMOVED`——可能仍在 `PROCESSING`，也可能句柄或传输端点已经失效。

一旦列表里有可读副本，这次查询便能返回副本描述和读租约。后续的字节搬运是主线 Get 和 Transfer Engine 要处理的下一步。

## 测试用例如何覆盖 `PROCESSING`

基线 `v0.3.13.post1` 的 `SingleSliceMultiReplicaFlow` 使用一个 5 MiB 对象，配置 3 个内存副本。它用下面四步断言副本状态从 `PROCESSING` 到 `COMPLETE` 的查询结果。

1. `PutStart` 返回 3 个副本状态为 `PROCESSING` 的副本，缓冲区长度与申请值一致。
2. `GetReplicaList` 返回 `REPLICA_IS_NOT_READY`。
3. `PutEnd(MEMORY)` 返回成功。
4. 再查 `GetReplicaList`，返回 3 个副本状态为 `COMPLETE` 的副本，长度仍与申请值一致。

上面是基线用例的静态阅读，本次没有执行该用例。它覆盖的是 Master 的元数据和可见性。运行中遇到相同错误码，仍要用日志和副本状态查明是 `PutEnd` 未完成、句柄失效，还是端点失效。

## 手上只有一条报错，下一条找什么？

先用 key、租户和时间把 B 的查询与 A 的写入对齐，查 Master 当时有没有接受 `PutEnd`。确认 R1 在查询前的副本状态已经是 `COMPLETE`，再检查那个时刻的句柄和端点记录。晚一些看到的副本状态只能说明重查时的情况。

**源码延伸**：`Client::Query`、`GetReplicaList`、`IsReplicaReadable`、以及 `master_service_test.cpp` 中的 `PROCESSING` 用例。

- [Client::Query](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/src/client_service.cpp#L1211)
- [MasterService::GetReplicaList](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/src/master_service.cpp#L3626) · [IsReplicaReadable](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/src/master_service.cpp#L3546)
- [PROCESSING 副本的测试用例](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/tests/master_service_test.cpp#L4041)

[系列目录与排版说明](README.md)

[返回主线 · Transfer Engine 的完成与回收](06-transfer-lifecycle.md)
