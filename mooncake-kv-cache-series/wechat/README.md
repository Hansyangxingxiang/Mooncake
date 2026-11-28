# Mooncake KV Cache 微信公众号系列

系列由 **10 篇主线 + 1 篇源码阅读番外** 组成。读者可以从跨实例复用开始，依次追踪对象布局、读写、传输、回收和推理接入。遇到 `REPLICA_IS_NOT_READY` 时，可直接读番外。

## 主线阅读顺序

1. [相同前缀能否跨实例复用？](01-shared-kv-cache.md)
2. [一个对象的两份副本占用哪段内存？](02-objects-and-replicas.md)
3. [Put 写完什么时候可见？](03-put-visibility.md)
4. [Get 读回字节为什么仍可能过期？](04-get-and-lease.md)
5. [Transfer Engine 提交失败后，为什么还不能释放源缓冲区？](06-transfer-lifecycle.md)
6. [一次 RDMA WRITE 由谁确认完成？](07-rdma-data-path.md)
7. [双网卡传输会走哪条路径？](08-multi-nic-numa.md)
8. [为什么淘汰候选会被跳过？](09-cache-eviction.md)
9. [SSD 上有数据，为什么还查不到？](10-ssd-tiering.md)
10. [共享 KV Cache 命中为何未必缩短 TTFT？](11-kv-reuse-and-inference.md)

**源码阅读番外**　[Mooncake 报 `REPLICA_IS_NOT_READY`，该从哪查起？](05-source-reading.md)

篇次以上面的目录为准。文件名前缀是原有索引，沿用原路径，便于已引用的链接继续定位。更完整的调用链、类型与行号见[源码解析版目录](../README.md)。

## 贯穿示例

推理实例 A 把一个前缀 KV 分块写入 Store，key 记为 `K`，有效负载为 **4 MiB**。B 在另一次请求中尝试复用它。R1、R2 是 K 的两份内存副本，R3 是 SSD 篇单独讨论的磁盘副本。具体场景可能只用其中一份，另外两份是否存在取决于当篇的配置和阶段。

4 MiB 是示意对象的大小。首篇的 1024 个 token 描述请求前缀，实际的字节数、对象数和传输分片由模型、框架布局和实际复用范围决定。

## 排版与预览

- 一级标题填入公众号标题栏，摘要填入摘要栏，正文从摘要后的开场段开始。正文建议 16 px、行距 1.75，图注建议 14 px。
- 正文里的 Mermaid 是发布图源，`figures/` 下的 Graphviz 文件留作备用，不参与 `build_publish.py` 导出。
- 导出后以手机正文宽度检查字号、箭头和图注，再将图片插入公众号编辑器。把上一篇、下一篇等相对链接换成对应的已发布文章链接。
- 在[发布稿目录](publish/README.md)打开当前版本的 HTML、Markdown 或排版报告。`build_publish.py` 的参数仍按文件索引选篇，整理发布顺序时以本页主线目录为准。

## 版本与证据边界

Mooncake 源码以 [`v0.3.13.post1`](https://github.com/kvcache-ai/Mooncake/tree/719735896c86b56fabec6cf3e825fb2ea640597a) 为基线，文末链接定位到具体路径。示意值、配置分支和静态源码核对分别标注。RDMA 硬件行为、SSD 掉电恢复和端到端性能需要在相应环境实测。

推理集成篇给出 Store 复用与 PD 交接的机制示意，以及 TTFT 的实验设计。vLLM 的 connector、scheduler 和失败恢复行为需要按部署版本核对。
