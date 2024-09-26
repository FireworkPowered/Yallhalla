> [!note] 
> 该篇文档与 `elaina-segment` 相关，其使用 Cython 工具链编写，需要使用 `cibuildwheel` 针对不同平台编译 whl，故从 Alconna v2 仓库中独立。
> 
> 以类似方法处理的还有 `elaina-triehard`，但这篇文档不会谈到。其被用于实现高效的前缀树与最长前缀匹配，我们将其用于 Subcommand 的前缀匹配与紧凑关键字匹配。你可以在 [[Analyzer]] 了解更多信息。

`Buffer` 及 `Token` 机制是 [[Analyzer]] 进行输入解析的重要组件，在这篇文档中，我们将讨论其基本原理，遵循的数个规则与其对 Sistana 的作用。
## Runes

Buffer 处理形如 `list[str | list[T]]` 的输入，我们称之为 Runes。Runes 通过将字符串视作字符串的序列（`Sequence[str]`），与 `list[T]` 混合，并利用二者皆可视作 `Sequence[...]` 这一同构来简化处理。

Runes 通常不会直接构造，而是通过 `elaina-segment` 提供的 `build_runes` 函数从 `Iterable[str | T]` 中生成。通过这种方法生成的 Runes 会具备这些良好性质：

1) 不会存在空字符串或空序列；
2) 输入中的连续字符串会被合并；
3) 非字符串序列不会连续出现。

> [!note]
> 对于 Alconna 面向 Sistana 的前端实现 Stargazing 中，由于 Alconna 使用了更一般的 `DataCollection` （等价于 Sequence）的类型作为输入，我们通过 `elaina-flywheel` / Flywheel 中的 Anycast 特性为其提供了可重载的 `build_runes` 实现。
## `segment_once`

`elaina-segment` 的核心函数 `segment_once` 提供了基于 `Runes` 生成 `(head: Segment[T], tail: () => Runes[T])` 二元组的实现。

`Segment[T]` 是一个 TypeAlias，其等价于 `str | T | Quoted[T] | UnmatchedQuote[T]`。`Quoted[T]` 用于表示被引号包裹的 `str | T` 序列，`UnmatchedQuote[T]` 用于表示被未匹配的引号介导进行引号匹配，在这一尝试中失败并*退而求全*的分段结果（半包裹的 `str | T`）。

通过调用 `segment_once(runes, separators)`，我们可以得到一个 `head` 与一个 `tail`。`head` 为当前分段的结果，`tail` 为返回剩余的 Runes 的闭包。`separators` 用于指定分割符，其类型为 `Sequence[str]`。

> [!note]
> 因为生成 tail 在通常情况下是个费时过程，将其用闭包包裹，类似某种惰性求值，可以避免不必要的计算，并实现 Buffer 的 failsafe。
>
> failsafe 是 [[Analyzer]] 的重要特性，使之能做到在不必要的情况下不消费输入，从而使得外界能够根据比 Analyzer 所知道更多的信息来实现更好的反馈。

### `Quoted` 与 `UnmatchedQuoted`

`Quoted` 与 `UnmatchedQuoted` 用于表示引号包裹的字符串或未匹配的引号。

```python
T = TypeVar("T")

@dataclass
class Quoted(Generic[T]):
    ref: list[str | T] | str
    trigger: str
    target: str

@dataclass
class UnmatchedQuoted(Generic[T]):
    ref: list[str | T] | str
    trigger: str
```

可以通过向 `Buffer.next` 或 `segment_once` 传递 `quote_pairs: dict[str, str]` 参数来指定括号集合。
当无法找到匹配的括号时，会返回被最近的分隔符字符隔断的 `UnmatchedQuoted` 段落实例。

## `Token`

`Buffer` 将 `segment_once` 封装为 `Buffer.next` 方法，并透过此生成 `Token` 对象。`Token` 是 `Buffer` 的输出，其持有对于从 `segment_once` 得到的 `Segment[T]`、`tail` 闭包与 `Buffer` 的引用。

调用 `Buffer.next` 不会立刻对 `Buffer` 产生副作用，而是由 Token 托管这一操作。

`Token.val` 即为获取到的 `Segment[T]`，`tail` 的闭包则是私有属性，当调用 `Token.apply` 时，`tail` 的闭包被更新到 Buffer 中，从而实现对 `Token.val` 的消费。

## Ahead

Buffer 持有一特殊的栈 `ahead`，其用于外界存储已经预订被特定逻辑以 `segment_once` 外的方法处理好了的 `Segment[T]`。当 `Buffer.ahead` 不为空时，`Buffer.next` 将返回将消费该栈中栈顶元素的 Token 对象。

> [!warning]
> 尤其注意，`ahead` 设计为仅存放 **被特殊逻辑分好的 Segment**，而通常情况下，我们对一个序列性质的 `Segment[T]` 一刀两半后，会得到两段 `Segment[T]`。此时，我们可能只需要前半段，而后半段可能会有后续的逻辑来处理。这种情况下，我们应仅将前半段推入 `ahead`，而将后半段重新提交到 `Buffer` 的待分割输入中。

## 实际使用

### 构造 Buffer

我们可以通过以下方式构造一个 Buffer 对象：

```python
Buffer(["lp user", 123, "permission node"])
```

如果我们已经通过外部的 `build_runes` 方法生成了 Runes 对象，应当使用：

```python
Buffer(runes, runes=False)
```

这将禁用 Buffer 对输入的自动 `build_runes` 处理。

### 获取 Token

我们可以通过以下方式获取 Token 对象：

```python
buffer = Buffer(["lp user", 123, "permission node"])
token = buffer.next(" ")
```

这将返回一个 Token 对象，其 `val` 属性为 `"lp"`，并持有余下部分 Runes 的闭包。

通过 `token.apply()`，我们可以更新 Buffer，从而消费这一 Token。

```python
token.apply()
```

我们可以通过传入 `separators: str` 来指定分割符，当 `ahead` 栈不为空时，这一参数将被忽略。

```python
token = buffer.next(" |")  # 以空格或管道符分割
```

### Ahead 使用

使用 `Buffer.add_to_ahead` 将一个 `Segment[T]` 推入 `ahead` 栈中。

```python
buffer.add_to_ahead("lp")
```

这将使得下一次调用 `Buffer.next` 时，返回的 Token 对象的 `val` 为 `"lp"`；当调用 `Token.apply` 时，`"lp"` 将被消费。

通过调用 `Buffer.pushleft` 将待处理的 `Runes[T]` 推入 Buffer 的待处理区中。这个函数会处理 `Quoted` 与 `UnmatchedQuoted`，并将传入的 Segment 重新 `build_runes`。
你可以一次性推入多个 `Segment[T]`，就像使用 `build_runes` 一样。

```python
buffer.pushleft("lp", 123)
```

这将使得下次调用 `Buffer.next` 时，如果 ahead 栈为空，则这些参数将被重新通过 `segment_once` 分割。
