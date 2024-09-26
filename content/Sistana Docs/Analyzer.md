`Analyzer.loopflow` 是 Sistana 工作的首要重点：其承载，实现了 Sistana 中绝大多数的顺序逻辑。

> [!note]
> Analyzer 的逻辑在当前阶段可能随时变动，所以本文档仅会谈到 Analyzer 如何设计，这些设计的目的与效果，还有 Analyzer 如何处理某些特殊情况（如紧凑指令头）。

