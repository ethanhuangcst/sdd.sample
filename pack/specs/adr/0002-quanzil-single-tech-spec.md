# ADR 0002 — 单份 tech-spec（Quanzil）与 Pack / Spikes 布局

日期：2026-10-13 · 状态：已采纳 · 取代 [0001-tech-spec-layering.md](./0001-tech-spec-layering.md)

## 背景

ADR 0001 引入 `tech-delta` 分层，但课程要求技术规格只存在于 `tech-spec.zh.md` / `tech-spec.en.md`，且 LLM 统一为 Quanzil（对话与图像），不再维护 DashScope / OpenAI 分叉。点评能力（Tripadvisor）从课程栈移除。仓库重组为 `pack/`（纯文档）与 `spikes/`（技术探针与真实应用验证）。

## 决策

1. **文档位置**：`pack/specs/tech-spec.zh.md` 与 `pack/specs/tech-spec.en.md` 为唯一技术规格；项目附录（如 MyDay）写在规格内，**废止 tech-delta**。
2. **LLM**：全地区对话与图像均经 Quanzil；环境变量 `QUANZIL_*`；不写厂商特有参数（无官方文档时不臆造）。
3. **地图**：仅 Google Maps 与 AMap，按地区矩阵选择；移除 Google Maps MCP 与 Tripadvisor。
4. **npm**：大陆与香港 npmmirror，海外 npmjs.org，写在 tech-spec。
5. **Pack / Spikes**：`pack/projects` 与 `pack/specs` 仅文档；可运行验证放在 `spikes/`（本 ADR 时不建应用）。

## 后果

- 维护点：两份 tech-spec + 项目 req/brd/prompt；换 LLM 上游由 Quanzil 部署承担，规格只维护 Quanzil 契约。
- MyDay 等技术约束在规格附录中，prompt 引用 `../../specs/tech-spec.*.md`。
- 0001 中 tech-delta 与「按 OpenAI/Qwen 分叉」的决策不再适用。
