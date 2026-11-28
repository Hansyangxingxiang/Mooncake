# Mooncake KV Cache 微信公众号发布稿

本目录由 `wechat/build_publish.py` 生成。系列 11 篇均已生成发布稿 Markdown、内联样式 HTML 与配图 PNG。`index.html` 是本地预览首页。

基线：`v0.3.13.post1` / `7197358`。生成时间：2026-11-28T09:59:50+00:00。

## 系列目录（11 篇）

| 篇次 | 文章 | 状态 | Markdown | HTML | 配图 |
|---|---|---|---|---|---|
| 1 | 同样的 1024 个前缀 token，第二台推理实例还要重算吗？ | 已生成 | [MD](./01-shared-kv-cache.md) | [HTML](./01-shared-kv-cache.html) | 1 |
| 2 | 一个 KV Cache 对象，在 Mooncake 里到底怎么放？ | 已生成 | [MD](./02-objects-and-replicas.md) | [HTML](./02-objects-and-replicas.html) | 2 |
| 3 | KV 已经写到远端，Mooncake 什么时候才让其他实例读？ | 已生成 | [MD](./03-put-visibility.md) | [HTML](./03-put-visibility.html) | 1 |
| 4 | KV 已经读回本地，为什么还会返回 LEASE_EXPIRED？ | 已生成 | [MD](./04-get-and-lease.md) | [HTML](./04-get-and-lease.html) | 1 |
| 5 | 源码阅读番外｜Mooncake 报 REPLICA_IS_NOT_READY，该从哪查起？ | 已生成 | [MD](./05-source-reading.md) | [HTML](./05-source-reading.html) | 0 |
| 6 | Transfer Engine 提交失败后，为什么还不能释放源缓冲区？ | 已生成 | [MD](./06-transfer-lifecycle.md) | [HTML](./06-transfer-lifecycle.html) | 1 |
| 7 | 一次 RDMA WRITE：KV 怎么写到远端，谁来确认传输完成？ | 已生成 | [MD](./07-rdma-data-path.md) | [HTML](./07-rdma-data-path.html) | 2 |
| 8 | 双网卡跑 Mooncake，RDMA 流量到底走了哪张卡？ | 已生成 | [MD](./08-multi-nic-numa.md) | [HTML](./08-multi-nic-numa.html) | 1 |
| 9 | 已经选中要淘汰的 KV，为什么又被跳过了？ | 已生成 | [MD](./09-cache-eviction.md) | [HTML](./09-cache-eviction.html) | 1 |
| 10 | KV Cache 已经写进 SSD，为什么其他节点还可能读不到？ | 已生成 | [MD](./10-ssd-tiering.md) | [HTML](./10-ssd-tiering.html) | 2 |
| 11 | 共享 KV Cache 命中了，首个 token 为什么还可能没变快？ | 已生成 | [MD](./11-kv-reuse-and-inference.md) | [HTML](./11-kv-reuse-and-inference.html) | 2 |

“已生成”表示本次构建写出了对应产物并通过脚本校验；阅读效果以实际浏览器检查为准。

## 构建与校验

```bash
cd mooncake-kv-cache-series/wechat
python3 build_publish.py --articles all \
  --md2wx "$HOME/.claude/skills/producing-tech-blog/scripts/md2wx.py" \
  --chrome /usr/bin/google-chrome
```

脚本只替换 Mermaid 图与相对链接，不改写正文。`manifest.json` 记录每篇的产物、配图尺寸、md2wx 报告与已执行的浏览器视口检查。

已在 375px、390px 的 Chrome 视口打开全部正式 HTML，检查页面与正文横向溢出、图片加载和表格宽度；已人工复核第 1、2、3、4、5、6、7、8、9、10、11 篇的配图与图注。

## 未执行项

- 未在公众号编辑器中粘贴预览。
- 未上传图片到公众号素材库，也未发布。
- GitHub 永久链接按本地 tag 的 commit 构造，未做 HTTP 可达性校验。
