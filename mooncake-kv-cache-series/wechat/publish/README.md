# Mooncake KV Cache 微信公众号发布稿

本目录的发布稿 Markdown 由 `wechat/build_publish.py` 转换生成。系列 11 篇 Markdown 与配图 PNG 保留；HTML、`report.json` 与 `manifest.json` 已移除。

基线：`v0.3.13.post1` / `7197358`。Markdown 同步时间：2026-11-28。

## 系列目录（11 篇）

| 篇次 | 文章 | Markdown | 配图 |
|---|---|---|---|
| 1 | 同样的 1024 个前缀 token，第二台推理实例还要重算吗？ | [MD](./01-shared-kv-cache.md) | 1 |
| 2 | 一个 KV Cache 对象，在 Mooncake 里到底怎么放？ | [MD](./02-objects-and-replicas.md) | 2 |
| 3 | KV 已经写到远端，Mooncake 什么时候才让其他实例读？ | [MD](./03-put-visibility.md) | 1 |
| 4 | KV 已经读回本地，为什么还会返回 LEASE_EXPIRED？ | [MD](./04-get-and-lease.md) | 1 |
| 5 | Mooncake 报 REPLICA_IS_NOT_READY，该从哪查起？ | [MD](./05-source-reading.md) | 0 |
| 6 | Transfer Engine 返回成功，数据就传完了吗？ | [MD](./06-transfer-lifecycle.md) | 1 |
| 7 | 一次 RDMA WRITE：KV 怎么写到远端，谁来确认传输完成？ | [MD](./07-rdma-data-path.md) | 2 |
| 8 | 双网卡跑 Mooncake，RDMA 流量到底走了哪张卡？ | [MD](./08-multi-nic-numa.md) | 1 |
| 9 | 已经选中要淘汰的 KV，为什么又被跳过了？ | [MD](./09-cache-eviction.md) | 1 |
| 10 | KV Cache 已经写进 SSD，为什么其他节点还可能读不到？ | [MD](./10-ssd-tiering.md) | 2 |
| 11 | 共享 KV Cache 命中了，首个 token 为什么还可能没变快？ | [MD](./11-kv-reuse-and-inference.md) | 2 |

11 篇发布稿 Markdown 与配图均已保留；配图数量与源稿一致。发布稿与 `wechat/` 源稿正文尚未逐段同步，源稿保留 Mermaid 图源。第 1 篇配图 1、第 7 篇配图 2 已改为 Graphviz DOT 图源（`../figures/*.dot`），PNG 由 `dot` 手动渲染；源稿中对应 Mermaid 块仅保留作语义参考。第 4 篇配图 1 仍由 Mermaid 渲染。HTML 已删除，当前目录没有可打开的本地预览页。

## 构建与校验

```bash
cd mooncake-kv-cache-series/wechat
python3 build_publish.py --articles all \
  --md2wx "$HOME/.claude/skills/producing-tech-blog/scripts/md2wx.py" \
  --chrome /usr/bin/google-chrome
```

脚本只替换 Mermaid 图与相对链接，不改写正文。完整构建会重新生成 HTML、`report.json`、`manifest.json` 与配图；当前仓库只保留发布稿 Markdown 和已导出的 PNG。

注意：构建从 `wechat/NN-*.md` 读取正文，而 `publish/` 与源稿正文尚未逐段同步。直接重跑完整构建会用源稿正文覆盖当前发布稿；若需重建，先核对两边正文。

注意：`build_publish.py` 目前只渲染 Mermaid，不认识 `figures/*.dot`。直接重跑完整构建会用 Mermaid 版覆盖第 1 篇配图 1 和第 7 篇配图 2 的 PNG。这两张图需要先运行：

```bash
dot -Tpng -Gdpi=288 figures/01-shared-kv-cache-fig1.dot -o publish/images/01-shared-kv-cache-fig1.png
dot -Tpng -Gdpi=288 figures/07-rdma-data-path-fig2.dot -o publish/images/07-rdma-data-path-fig2.png
```

本轮没有重建 HTML，因此没有新的 md2wx 报告或浏览器视口检查。HTML 恢复后，仍需在 360px、390px 视口检查横向溢出、图片加载和表格宽度。

## 未执行项

- 未在公众号编辑器中粘贴预览。
- 未上传图片到公众号素材库，也未发布。
- GitHub 永久链接按本地 tag 的 commit 构造，未做 HTTP 可达性校验。
