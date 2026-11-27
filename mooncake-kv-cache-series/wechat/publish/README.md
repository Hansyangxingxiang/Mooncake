# Mooncake KV Cache 微信公众号发布稿

本目录由 `wechat/build_publish.py` 生成。系列 11 篇均已生成发布稿 Markdown、内联样式 HTML 与配图 PNG。`index.html` 是本地预览首页。

基线：`v0.3.13.post1` / `7197358`。生成时间：2026-11-27T07:54:26+00:00。

## 系列目录（11 篇）

| 篇次 | 文章 | 状态 | Markdown | HTML | 配图 |
|---|---|---|---|---|---|
| 1 | 第1篇｜同样的 1,024 个前缀 token，第二台推理实例还要再算一遍吗？ | 已生成 | [MD](./01-shared-kv-cache.md) | [HTML](./01-shared-kv-cache.html) | 1 |
| 2 | 第2篇｜一个 KV Cache 对象，在 Mooncake 里到底怎么放？ | 已生成 | [MD](./02-objects-and-replicas.md) | [HTML](./02-objects-and-replicas.html) | 2 |
| 3 | 第3篇｜字节已经写完，为什么另一个 Client 还读不到？ | 已生成 | [MD](./03-put-visibility.md) | [HTML](./03-put-visibility.html) | 1 |
| 4 | 第4篇｜字节读回来了，这次 Get 为什么还可能失败？ | 已生成 | [MD](./04-get-and-lease.md) | [HTML](./04-get-and-lease.html) | 1 |
| 5 | 第5篇｜Mooncake 里 key 明明存在，为什么还是读不到副本？ | 已生成 | [MD](./05-source-reading.md) | [HTML](./05-source-reading.html) | 0 |
| 6 | 第6篇｜Transfer Engine 返回成功，数据就传完了吗？ | 已生成 | [MD](./06-transfer-lifecycle.md) | [HTML](./06-transfer-lifecycle.html) | 1 |
| 7 | 第7篇｜一次 RDMA 传输，究竟经历了什么？ | 已生成 | [MD](./07-rdma-data-path.md) | [HTML](./07-rdma-data-path.html) | 2 |
| 8 | 第8篇｜双网卡跑 Mooncake，RDMA 流量到底走了哪张卡？ | 已生成 | [MD](./08-multi-nic-numa.md) | [HTML](./08-multi-nic-numa.html) | 1 |
| 9 | 第9篇｜缓存满了，K 刚被读到，淘汰线程为什么删不掉它？ | 已生成 | [MD](./09-cache-eviction.md) | [HTML](./09-cache-eviction.html) | 1 |
| 10 | 第10篇｜KV Cache 已经写进 SSD，为什么其他节点还可能读不到？ | 已生成 | [MD](./10-ssd-tiering.md) | [HTML](./10-ssd-tiering.html) | 2 |
| 11 | 第11篇｜共享了 KV Cache，为什么推理不一定更快？ | 已生成 | [MD](./11-kv-reuse-and-inference.md) | [HTML](./11-kv-reuse-and-inference.html) | 2 |

“已生成”表示本次构建写出了对应产物并通过脚本校验；阅读效果以实际浏览器检查为准。

## 构建与校验

```bash
cd mooncake-kv-cache-series/wechat
python3 build_publish.py --articles all \
  --md2wx "$HOME/.claude/skills/producing-tech-blog/scripts/md2wx.py" \
  --chrome /usr/bin/google-chrome
```

脚本只替换 Mermaid 图与相对链接，不改写正文。`manifest.json` 记录每篇的产物、配图尺寸、md2wx 报告与已执行的浏览器视口检查。

已在 360px、390px 的 Chrome 视口打开全部正式 HTML，检查页面与正文横向溢出、图片加载和表格宽度；已人工复核第 1、3、4、6、7、10 篇的配图、图注与正文排版。

## 未执行项

- 未在公众号编辑器中粘贴预览。
- 未上传图片到公众号素材库，也未发布。
- GitHub 永久链接按本地 tag 的 commit 构造，未做 HTTP 可达性校验。
