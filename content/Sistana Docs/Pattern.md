Sistana 依赖于两种 Pattern 数据模型来描述实际运行时 Sistana 解析输入的依据，我们称这种依据为「模式」，在后文中，我们也会使用该术语来指代这两种数据模型、其实例化对象及其背后的设计本身。

## `SubcommandPattern`

`SubcommandPattern` 是组成模式树的主干。


> [!tip] 
> 
> 在由 [[Analyzer]] 执行的状态流转中，状态总是从 Subcommand 流转到其他 Subcommand，或是流转到分支上并最终返回到原 Subcommand。

### `SubcommandPattern.build`

你可以通过简单的调用 `SubcommandPattern.build` 来构造一个 `SubcommandPattern` 实例。

```python
lp = SubcommandPattern.build(
    header="lp"
)
```

通过传入 [[Fragment]] 以构造指令参数。


> [!seealso]
> [[Analyzer]] 谈到了从 `SubcommandPattern` 构造 `AnalyzeSnapshot` 的多种方式，并详细解释了不同 entrypoint 在 Analyzer 中的处置方式。
> [[Fragment]] 谈到了更多与 Fragment 相关的设置。


```python
cp = SubcommandPattern.build("cp", Fragment("source"), Fragment("dest"))
```

使用 `prefixes` 控制指令前缀；仅在获取 [[Snapshot]] 的 `prefix_entrypoint` 时起作用；传入的序列会被迭代并复制其元素引用。
当匹配时，Sistana 将尝试匹配可能的最长（最接近）前缀，这可能导致指令头匹配失败。

```python
pop = SubcommandPattern.build("pop", prefixes=["/"])
```

使用 `compact_header` 控制是否将指令头以紧凑形式匹配。

> [!seealso] 
> [[Analyzer]] 解释了紧凑样式设置在实际运行时起到的作用。


```python
dnd = SubcommandPattern.build("d", compact_header=True)
```

指定 `separators` 以控制分割输入时使用的分割字符集。

> [!danger]
> 指定 separators 为空字符串（`""`）时，极有可能发生未定义行为，但这是被允许的，只是现阶段没有很好的外围设施来帮助用户使用该特性。


指定 `header_fragment` 将允许传入该参数上的 `Fragment` 接收触发该 Subcommand 时使用的触发词（此处仅为指令头）。

> [!seealso]
> 在 [[Capture, Validator, Transform, Receiver]] 中对 `header_fragment` 有更详尽的解释。


### `SubcommandPattern.subcommand`

该方法用于为 `SubcommandPattern` 添加一个子指令，并返回该子指令对应的 `SubcommandPattern` 实例以供后继操作。  
和 `SubcommandPattern.build` 一样，可以可选的传入多个 [[Fragment]] 以构造指令参数。

```python
user_command = lp.subcommand("user", Fragment("user"))
```

通过指定 `aliases` 来指定指令别名。

```python
lp.subcommand("update", aliases=["u"])
```

通过指定 `soft_keyword=True` ，指令关键字可以在前驱指令或选项的参数未满足条件时充作参数值，而不是开始匹配该子指令。

> [!tip]
> 指令关键字是包括指令的 `header` 与其别名的集合。

```python
user_command.subcommand("permission", Fragment("node"), soft_keyword=True)
```

指定 `compact_header=True` 可以指定将指令头用于匹配可能的紧凑形式的情况（如 mysql 中 `-uusername -ppassword`，或是跑团指令）。
指定 `compact_aliases=True` 可以将别名也用于参与紧凑匹配。

> [!warning]
> 此处语义与 `SubcommandPattern.build` 不同，由前者的 `header_entrypoint` 介导不会匹配别名 —— 从内部结构角度上来说，`SubcommandPattern.compact_keywords` 不会用法。

指定 `satisfy_previous=False` 将禁用前驱匹配满足检查，这通常用于 `help` 等内置指令的设计。

指定 `header_fragment` 将允许传入该参数上的 `Fragment` 接收触发该 Subcommand 时使用的触发词（指令头或别名）。

## `OptionPattern`

Sistana 将 Option 作为 Subcommand 的附属，当匹配 Option 时，总是从 Subcommand 流转状态到 Option，并最终总会流转回对应 Subcommand。

> [!seealso]
> [[Analyzer]] 详细讲述了 Analyzer 如何处理状态流转。

建议总是使用 `SubcommandPattern.option` 为子指令添加选项；该方法返回 `SubcommandPattern` 自身，所以你可以使用链式调用为单个子指令添加多个选项。 

```python
cp.option("--recursion")
```

传入 `aliases` 指定选项别名。

```python
cp.option("--recursion", aliases=["-r"])
```

可选的传入一个或多个 `Fragment` 以构造 Option 的指令参数序列。

```python
cp.option("--user", Fragment("user"), aliases=["-u"])
```

设定 `soft_keyword=True` 使选项关键字可以在前驱指令或选项的参数未满足条件时充作参数值，而不是开始匹配该子指令。相比 Subcommand，对于 Option 来说这个似乎没有太大用途。

```python
query_cmd.option("name", soft_keyword=True)
```

指定 `separators` 以指定 option 使用的分割字符，但如果想要实现 `name=...`，你应该使用 `header_separators`。

```python
query_cmd.option("name", Fragment("name"), soft_keyword=True, header_separators="=")
```

这将匹配 `name=...`，并将 `...` 赋作为 Fragment `name` 的值。

`hybrid_separators` 影响 `separators` 的行为：如果设作 `True`，则会在创建时继承当时 Subcommand 的 `separators`。

`allow_duplicate` 使得选项可以多次出现并参与匹配，对应 Fragment 上的 `Rx`，`rx_prev` 将可以获取到之前的赋值。这通常配合 `CountRx` 或 `AccumRx` 使用，`header_fragment` 也会被触发多次（根据触发的关键字或别名）。

`SubcommandPattern.option` 方法中的 `compact_header`、`compact_aliases`、`header_fragment` 与 `SubcommandPattern.subcommand` 中的同名参数作用类似，行为基本一致，在此略。
