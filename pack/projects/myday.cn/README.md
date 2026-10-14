# MyDay 中文种子

本目录是可整包复制的中文 MyDay 种子，不是运行中的工程。

## 复制到工程目录

例如目标为 `spikes/myday.cn/`：

```text
spikes/myday.cn/
  prompt.md          ← 本目录的 prompt.md
  specs/
    req-spec.md      ← 本目录的 req-spec.md
    tech-spec.md     ← 从 pack/tech-spec.cn/tech-spec.md 复制
    .secrets         ← 由本目录的 .secrets.example 复制，只填写 Key
```

代理从 `specs/` 里的需求、技术规格和 `.secrets` 开工。`.secrets` 与生成的 `.env.local` 都不提交。`.secrets.example` 可以提交。

## Pack 内路径

- 共用技术规格：`pack/tech-spec.cn/tech-spec.md`
- 本种子需求：`pack/projects/myday.cn/req-spec.md`
