# Mooncake KV Cache 微信公众号发布稿

本目录的发布稿 Markdown 由 `wechat/build_publish.py` 转换生成。系列 11 篇 Markdown 与配图 PNG 保留；HTML、`report.json` 与 `manifest.json` 已移除。

基线：`v0.3.13.post1` / `7197358`。Markdown 同步时间：2026-11-28。

## 系列目录（11 篇）

| 篇次 | 文章 | Markdown | 配图 |
|---|---|---|---|
| 1 | 第1篇｜同样的 1,024 个前缀 token，第二台推理实例还要再算一遍吗？ | [MD](./01-shared-kv-cache.md) | 1 |
| 2 | 第2篇｜一个 KV Cache 对象，在 Mooncake 里到底怎么放？ | [MD](./02-objects-and-replicas.md) | 2 |
| 3 | 第3篇｜Mooncake 里 KV Cache 字节已经写完，为什么另一个 Client 还读不到？ | [MD](./03-put-visibility.md) | 1 |
| 4 | 第4篇｜Mooncake 里 KV Cache 字节读回来了，这次 Get 为什么还可能失败？ | [MD](./04-get-and-lease.md) | 1 |
| 5 | 第5篇｜Mooncake 里 key 明明存在，为什么还是读不到副本？ | [MD](./05-source-reading.md) | 0 |
| 6 | 第6篇｜Transfer Engine 返回成功，数据就传完了吗？ | [MD](./06-transfer-lifecycle.md) | 1 |
| 7 | 第7篇｜一次 RDMA 传输，究竟经历了什么？ | [MD](./07-rdma-data-path.md) | 2 |
| 8 | 第8篇｜双网卡跑 Mooncake，RDMA 流量到底走了哪张卡？ | [MD](./08-multi-nic-numa.md) | 1 |
| 9 | 第9篇｜缓存满了，刚被读到的 KV Cache，淘汰线程为什么删不掉它？ | [MD](./09-cache-eviction.md) | 1 |
| 10 | 第10篇｜KV Cache 已经写进 SSD，为什么其他节点还可能读不到？ | [MD](./10-ssd-tiering.md) | 2 |
| 11 | 第11篇｜共享了 KV Cache，为什么推理不一定更快？ | [MD](./11-kv-reuse-and-inference.md) | 2 |

11 篇 Markdown 均已与源稿同步；配图数量为源稿中的 Mermaid 图数量。HTML 已删除，当前目录没有可打开的本地预览页。

## 构建与校验

```bash
cd mooncake-kv-cache-series/wechat
python3 build_publish.py --articles all \
  --md2wx "$HOME/.claude/skills/producing-tech-blog/scripts/md2wx.py" \
  --chrome /usr/bin/google-chrome
```

脚本只替换 Mermaid 图与相对链接，不改写正文。完整构建会重新生成 HTML、`report.json`、`manifest.json` 与配图；当前仓库只保留发布稿 Markdown 和已导出的 PNG。

本轮没有重建 HTML，因此没有新的 md2wx 报告或浏览器视口检查。HTML 恢复后，仍需在 360px、390px 视口检查横向溢出、图片加载和表格宽度。

## 未执行项

- 未在公众号编辑器中粘贴预览。
- 未上传图片到公众号素材库，也未发布。
- GitHub 永久链接按本地 tag 的 commit 构造，未做 HTTP 可达性校验。
