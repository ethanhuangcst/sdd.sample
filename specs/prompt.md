# 限制

- 本次只执行下面的 prompt。不创建 Makefile，不使用 i18n。完成前不必再问是否可用。
- 我正在处理其他任务，请勿打扰。
- 自动运行；仅在受阻时停止。
- 在当前工程根目录工作。不要参考其他项目。不要把代码或文档写到本工程以外。

# 开发

1. 确认当前目录已有 `specs/req-spec.md`、`specs/tech-spec.md` 和 `specs/.secrets`。缺任一文件则视为受阻。
2. 阅读这三个文件。
3. 若还没有可运行的 Next.js 工程，按 `specs/req-spec.md` 非功能需求中的安装清单，用非交互命令建最小工程。不要停在创建工具的提问上。不要因为目录非空而改到别的目录。
4. 按 `specs/req-spec.md` 和 `specs/tech-spec.md` 完成开发。

# 验收

完成 `specs/req-spec.md` 里的「验收测试」。
