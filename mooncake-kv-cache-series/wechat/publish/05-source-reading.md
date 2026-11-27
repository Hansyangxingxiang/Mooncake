# 第5篇｜K 明明存在，B 为什么收到 REPLICA_IS_NOT_READY？

**摘要：**同一个错误码可能来自不同分支。沿着 B 的一次查询走四站，看每一站能排除什么、下一步该查什么。

实例 B 查询 key `K`，Master 返回 `REPLICA_IS_NOT_READY`。

这个错误容易被直接理解成"对象不存在"或"再等等就好"。但同一个返回码背后，可能是副本还在 `PROCESSING`，也可能是句柄或传输端点失效。只看一行错误日志，分不出是哪一种。

本篇不重复第3篇对 Put 可见性的系统解释，只跟 B 的一次查询走一遍源码，演示怎样从入口走到具体分支，以及每一站应该拿什么证据。

## 第一站：Client::Query 只把查询交给 Master

B 的查询入口是 [`Client::Query`](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/src/client_service.cpp#L1211)。它在发出 RPC 之前取一次本地 `steady_clock` 起点，然后调用 `master_client_.GetReplicaList(object_key)`，把返回的副本列表、lease 截止时间和 object checksum 组装成 `QueryResult`。

这一站要看的证据是 RPC 返回状态。如果 Master 返回 `REPLICA_IS_NOT_READY`，`Client::Query` 不会在本地把它改写成别的错误，也不会在这里替调用方选副本。

因此，看到这个错误时可以先排除一种解释：**它不是客户端本地缓存或副本选择阶段产生的，源头在 Master 的查询结果。**

## 第二站：Master 先判断 key 是否存在

进入 [`MasterService::GetReplicaList`](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/src/master_service.cpp#L3626)，第一道判断是 `accessor.Exists()`。key 不存在时，Master 直接返回 `OBJECT_NOT_FOUND`。

所以，如果 B 收到的是 `REPLICA_IS_NOT_READY`，在这个查询时刻，K 的 `ObjectMetadata` 是存在的。至少可以排除"这个 key 从未写入过"。

这里要注意范围：`OBJECT_NOT_FOUND` 也不总等于"从未存在"。对象可能曾经存在，但所有副本都已经被删除或移除；这条分支在第四站会再出现一次。

## 第三站：可读副本是过滤出来的

key 存在之后，`GetReplicaList` 用 `VisitReplicas` 遍历副本，只把通过 [`IsReplicaReadable`](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/src/master_service.cpp#L3546) 检查的副本放进结果列表。

`IsReplicaReadable` 至少要求：

- 副本状态是 `COMPLETE`；
- 内存或 NoF 句柄仍然有效；
- 传输端点没有被标记为失效。

如果 R1 是在 `PutStart` 之后、`PutEnd` 之前被查询，它已经分配了地址，也挂在 K 的元数据下面，但状态仍是 `PROCESSING`。第一项检查就不通过。

这一站能排除"对象根本没有副本"：副本结构已经存在，只是不满足可读条件。接下来还要看，过滤后的列表为什么没有变成另一种错误。

## 第四站：空列表最后分成两种错误

遍历结束后，如果没有选出任何可读副本，`GetReplicaList` 会再判断一次：

- 所有副本都是 `REMOVED`：返回 `OBJECT_NOT_FOUND`；
- 否则：记录 `replica_not_ready` 日志，返回 `REPLICA_IS_NOT_READY`。

因此，`REPLICA_IS_NOT_READY` 还排除了"所有副本都已移除"。它表示列表为空，但至少有一个副本不是 `REMOVED`——可能仍在 `PROCESSING`，也可能句柄或传输端点已经失效。

把这条查询路径压成判断顺序，就是：

1. key 不存在 → `OBJECT_NOT_FOUND`；
2. key 存在，且存在通过 `IsReplicaReadable` 的副本 → 返回可读副本列表与 lease；
3. key 存在，没有可读副本，且所有副本都是 `REMOVED` → `OBJECT_NOT_FOUND`；
4. key 存在，没有可读副本，但不是全部 `REMOVED` → `REPLICA_IS_NOT_READY`。

## 用测试确认 PROCESSING 这一种情况

[`master_service_test.cpp`](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/tests/master_service_test.cpp#L4069) 里有一个用例覆盖了这条分支：`PutStart` 返回的副本状态是 `PROCESSING`；在调用 `PutEnd` 之前，`GetReplicaList` 返回 `REPLICA_IS_NOT_READY`；`PutEnd` 之后，测试继续执行成功的读取。

这个用例把状态固定在"分配完成、提交之前"，断言正好落在第三站和第四站之间。它证明的是这一种 `PROCESSING` 场景，不是所有 `REPLICA_IS_NOT_READY` 的来源。阅读用例时先确认它的 setup 和断言，再判断它能否解释手上的现象。

## 定位时先收集三样证据

遇到 `REPLICA_IS_NOT_READY`，只凭错误码无法继续细分。下一步至少要拿到：

1. K 的元数据是否存在；
2. 各副本的状态是 `PROCESSING`、`COMPLETE` 还是 `REMOVED`；
3. 副本的内存/NoF 句柄和传输端点是否有效。

这三样分别对应第二站、第三站和第四站。只有第一样是"存在"，后两样仍可能让副本不可读。

下一篇进入传输层。提交一个传输任务之后，调用方还要回答另一个问题：什么时候才能安全释放 batch 和 buffer？

**源码延伸：**原系列篇1；`Client::Query`、`GetReplicaList`、`IsReplicaReadable`、以及 `master_service_test.cpp` 中的 `PROCESSING` 用例。版本基线为 `v0.3.13.post1`。

- [Client::Query](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/src/client_service.cpp#L1211)
- [MasterService::GetReplicaList](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/src/master_service.cpp#L3626) · [IsReplicaReadable](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/src/master_service.cpp#L3546)
- [PROCESSING 副本的测试用例](https://github.com/kvcache-ai/Mooncake/blob/719735896c86b56fabec6cf3e825fb2ea640597a/mooncake-store/tests/master_service_test.cpp#L4069)

[源码解析版](../../01-source-navigation.md) · [系列目录与排版说明](index.html)

[上一篇](04-get-and-lease.html) · [下一篇](06-transfer-lifecycle.html)
