# Spikes

本目录用于**技术探针**与**真实应用验证**：依据 `pack/tech-spec.cn/`、`pack/tech-spec.en/` 与 [`pack/projects/`](../pack/projects/) 中的 req-spec / brd / prompt 搭建可运行的 Next.js 应用，验证集成、性能契约与规格是否可实现。

- **`pack/`**：仅文档，不包含应用代码。
- **`spikes/`**：每个子目录对应一次探针或一门课程项目的参考实现（例如 `spikes/myday.cn`）。

种子来源：`pack/projects/myday.cn/` 或 `pack/projects/myday.en/`。把 `req-spec.md` 和对应的 `pack/tech-spec.cn/tech-spec.md` 或 `pack/tech-spec.en/tech-spec.md` 复制到工程内的 `specs/`，文件名保持 `req-spec.md` 与 `tech-spec.md`。见各种子目录下的 `README.md`。

`spikes/myday.cn/` 可只保留占位 spec；完整工程由 prompt 驱动代理生成。
