# MyDay 中文种子

本目录是可整包复制的中文 MyDay 种子，不是运行中的工程。

## 复制到工程目录

例如目标为 `spikes/myday.cn/`：

```text
spikes/myday.cn/
  prompt.md          ← 本目录的 prompt.md
  specs/
    prompt.md        ← 本目录 specs/prompt.md
    req-spec.md      ← req-spec-amap.md 或 req-spec-gmap.md，只复制一份
    tech-spec.md     ← 与需求成对：tech-spec-amap.md 或 tech-spec-gmap.md，只复制一份
    .secrets         ← .secrets.amap 或 .secrets.gmap，只复制一份并改名
    test-img.png     ← 验收时文件框无法操作就用这张
  web/               ← Next.js 工程，由 tech-spec 安装步骤创建
```

代理从 `specs/` 里的 prompt、需求、技术规格和 `.secrets` 开工。安装时把 `.secrets` 写入 `web/.env.local`。`.secrets` 与 `.env.local` 都不提交。

## Pack 内路径

- 中国大陆演示：`req-spec-amap.md` 配 `tech-spec-amap.md`
- 中国大陆以外的中文演示：`req-spec-gmap.md` 配 `tech-spec-gmap.md`
- 一次运行只复制一对，分别命名为 `specs/req-spec.md` 和 `specs/tech-spec.md`
- 共用全文：`pack/tech-spec.cn/tech-spec.md`
