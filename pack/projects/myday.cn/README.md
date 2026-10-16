# MyDay 中文种子

本目录是可整包复制的中文 MyDay 种子，不是运行中的工程。

## 复制到工程目录

例如目标为 `spikes/myday.cn/`：

```text
spikes/myday.cn/
  prompt.md          ← 本目录的 prompt.md
  web/               ← Next.js 工程，由 prompt 在空目录里创建
    .env.local       ← 人事先放入。代理不读 .secrets，也不写这个文件
  specs/
    req-spec.md      ← 本目录的 req-spec-amap.md 或 req-spec-gmap.md，只复制一份
    tech-spec.md     ← 与需求成对：tech-spec-amap.md 或 tech-spec-gmap.md，只复制一份
    test-img.png     ← 本目录 specs/test-img.png，验收时文件框无法操作就用这张
```

代理从 `specs/` 里的需求和技术规格开工。密钥只读 `web/.env.local`。`.env.local` 不提交。

## Pack 内路径

- 中国大陆演示：`req-spec-amap.md` 配 `tech-spec-amap.md`
- 中国大陆以外的中文演示：`req-spec-gmap.md` 配 `tech-spec-gmap.md`
- 一次运行只复制一对，分别命名为 `specs/req-spec.md` 和 `specs/tech-spec.md`
- 共用全文：`pack/tech-spec.cn/tech-spec.md`
