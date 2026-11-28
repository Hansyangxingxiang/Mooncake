# Mooncake KV Cache 微信公众号发布稿

**10 篇主线 + 1 篇源码阅读番外**。打开 HTML 可复制公众号正文。标题与摘要见 Markdown，分别填写到公众号对应栏位。配图需上传到公众号素材库。

1. 同样的 1024 个前缀 token，第二台推理实例还要重算吗？ · [Markdown](01-shared-kv-cache.md) · [HTML](.work/gzh-all/01-shared-kv-cache.html) · [排版报告](.work/gzh-all/01-shared-kv-cache.report.json)
2. 一个 KV Cache 对象，在 Mooncake 里到底怎么放？ · [Markdown](02-objects-and-replicas.md) · [HTML](.work/gzh-all/02-objects-and-replicas.html) · [排版报告](.work/gzh-all/02-objects-and-replicas.report.json)
3. KV 已经写到远端，Mooncake 什么时候才让其他实例读？ · [Markdown](03-put-visibility.md) · [HTML](.work/gzh-all/03-put-visibility.html) · [排版报告](.work/gzh-all/03-put-visibility.report.json)
4. KV 已经读回本地，为什么还会返回 LEASE_EXPIRED？ · [Markdown](04-get-and-lease.md) · [HTML](.work/gzh-all/04-get-and-lease.html) · [排版报告](.work/gzh-all/04-get-and-lease.report.json)
5. Mooncake 提交失败后，为什么还不能释放源缓冲区？ · [Markdown](06-transfer-lifecycle.md) · [HTML](.work/gzh-all/06-transfer-lifecycle.html) · [排版报告](.work/gzh-all/06-transfer-lifecycle.report.json)
6. 一次 RDMA WRITE：KV 怎么写到远端，谁来确认传输完成？ · [Markdown](07-rdma-data-path.md) · [HTML](.work/gzh-all/07-rdma-data-path.html) · [排版报告](.work/gzh-all/07-rdma-data-path.report.json)
7. 双网卡跑 Mooncake，RDMA 流量到底走了哪张卡？ · [Markdown](08-multi-nic-numa.md) · [HTML](.work/gzh-all/08-multi-nic-numa.html) · [排版报告](.work/gzh-all/08-multi-nic-numa.report.json)
8. 已经选中要淘汰的 KV，为什么又被跳过了？ · [Markdown](09-cache-eviction.md) · [HTML](.work/gzh-all/09-cache-eviction.html) · [排版报告](.work/gzh-all/09-cache-eviction.report.json)
9. KV Cache 已经写进 SSD，为什么其他节点还可能读不到？ · [Markdown](10-ssd-tiering.md) · [HTML](.work/gzh-all/10-ssd-tiering.html) · [排版报告](.work/gzh-all/10-ssd-tiering.report.json)
10. 共享 KV Cache 命中了，首个 token 为什么还可能没变快？ · [Markdown](11-kv-reuse-and-inference.md) · [HTML](.work/gzh-all/11-kv-reuse-and-inference.html) · [排版报告](.work/gzh-all/11-kv-reuse-and-inference.report.json)

## 源码阅读番外

源码阅读番外｜Mooncake 报 REPLICA_IS_NOT_READY，该从哪查起？ · [Markdown](05-source-reading.md) · [HTML](.work/gzh-all/05-source-reading.html) · [排版报告](.work/gzh-all/05-source-reading.report.json)

基线为 `v0.3.13.post1`。14 张 PNG 来自正文 Mermaid 与两份 DOT 图源。
