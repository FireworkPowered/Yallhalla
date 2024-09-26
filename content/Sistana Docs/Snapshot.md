Snapshot 存储了 Analyzer 的几乎所有状态与解析结果。`Snapshot` 被设计为可以在任意点从 Analyzer 中导出（根据 Buffer 是否已经为空来退出），并配合任何 Buffer 实例在任意点开始分析。清晰的设计使得 Sistana 具备了无以伦比的灵活性与可扩展性。

## 生成 Snapshot

Sistana 的 Snapshot 模型名为 `AnalyzeSnapshot`，被设定为可变对象。
Snapshot 可以由 `SubcommandPattern` 生成，可以根据 `Subcommand`


