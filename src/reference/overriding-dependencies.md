## 覆盖依赖项

覆盖依赖项的需求可能出现在多种场景中。不过，大多数情况下，这种需求都归结为能够在某个 crate 发布到 [crates.io] 之前就使用它。例如：

* 你正在开发的某个 crate 同时被用于另一个你正在开发的大型应用程序，而你希望在这个大型应用程序内部测试对该库的 bug 修复。
* 一个上游 crate（并非你直接维护）在其 git 仓库的主分支上有一个新功能或 bug 修复，你想测试一下。
* 你即将发布你的 crate 的一个新的主要版本，但你想在整个包中进行集成测试，以确保新的主要版本能够正常工作。
* 你为你发现的某个 bug 向上游 crate 提交了修复，但你希望你的应用程序能立即开始依赖该 crate 的修复版本，以避免因为等待 bug 修复被合并而受阻。

这些场景可以通过 [`[patch]` 清单部分] 来解决。

本章将介绍几种不同的用例，并包含有关覆盖依赖项的不同方式的详细信息。

* 示例用例
    * [测试bug修复](#测试bug修复)
    * [使用未发布的次要版本](#使用未发布的次要版本)
        * [覆盖仓库URL](#覆盖仓库url)
    * [预发布破坏性变更](#预发布破坏性变更)
    * [对多个版本使用`[patch]`](#对多个版本使用patch)
* 参考
    * [`[patch]`部分](#patch部分)
    * [`[replace]`部分](#replace部分)
    * [`paths`覆盖](#paths覆盖)

> **注意**：请参阅 [多个位置] 指定依赖项，它可以用于覆盖本地包中单个依赖项声明的源。

### 测试bug修复

假设你正在使用 [`uuid` crate]，但在开发过程中你发现了一个 bug。不过，你非常有进取心，所以决定也要尝试修复这个 bug！最初你的清单（manifest）会是这样的：

[`uuid` crate]: https://crates.io/crates/uuid

```toml
[package]
name = "my-library"
version = "0.1.0"

[dependencies]
uuid = "1.0"
```

我们要做的第一件事是通过以下命令在本地克隆 [`uuid` 仓库][uuid-repository]：

```console
$ git clone https://github.com/uuid-rs/uuid
```

接下来，我们将编辑 `my-library` 的清单，使其包含：

```toml
[patch.crates-io]
uuid = { path = "../path/to/uuid" }
```

这里我们声明要用一个新的依赖项来*打补丁（patch）* `crates-io` 源。这将有效地把本地签出（checkout）的 `uuid` 版本添加到我们本地包的 crates.io 注册表中。

接下来，我们需要确保锁定文件（lock file）已更新为使用这个新版本的 `uuid`，以便我们的包使用本地签出的副本，而不是 crates.io 上的一个。`[patch]` 的工作方式是：它会从 `../path/to/uuid` 加载依赖项，然后每当 crates.io 被查询 `uuid` 的版本时，它*也*会返回本地版本。

这意味着本地签出版本的版本号很重要，并且会影响补丁是否被使用。我们的清单声明了 `uuid = "1.0"`，这意味着我们只会解析到 `>= 1.0.0, < 2.0.0` 的范围，而 Cargo 的贪婪解析算法也意味着我们将解析到该范围内的最大版本。通常情况下这并不重要，因为 git 仓库的版本通常已经大于或等于 crates.io 上发布的最大版本，但请务必记住这一点！

无论如何，通常你现在需要做的只是：

```console
$ cargo build
   Compiling uuid v1.0.0 (.../uuid)
   Compiling my-library v0.1.0 (.../my-library)
    Finished dev [unoptimized + debuginfo] target(s) in 0.32 secs
```

就是这样！你现在正在使用本地版本的 `uuid` 进行构建（注意构建输出中括号内的路径）。如果你没有看到本地路径版本被构建，那么你可能需要运行 `cargo update -p uuid --precise $version`，其中 `$version` 是本地签出的 `uuid` 副本的版本。

一旦你修复了最初发现的 bug，下一步你可能想做的就是将其作为拉取请求提交给 `uuid` crate 本身。完成后，你也可以更新 `[patch]` 部分。`[patch]` 内部的列表就像 `[dependencies]` 部分一样，所以一旦你的拉取请求被合并，你就可以将你的 `path` 依赖更改为：

```toml
[patch.crates-io]
uuid = { git = 'https://github.com/uuid-rs/uuid' }
```

[uuid-repository]: https://github.com/uuid-rs/uuid

### 使用未发布的次要版本

现在让我们把重点从 bug 修复转移到添加新功能上。在开发 `my-library` 时，你发现 `uuid` crate 需要一个全新的功能。你已经实现了这个功能，在上面使用 `[patch]` 在本地进行了测试，并提交了拉取请求。让我们来看看在它实际发布之前，你如何继续使用和测试它。

同时，假设 crates.io 上 `uuid` 的当前版本是 `1.0.0`，但 git 仓库的主分支已经更新到 `1.0.1`。这个分支包含了你之前提交的新功能。为了使用这个仓库，我们将把 `Cargo.toml` 编辑成如下样子：

```toml
[package]
name = "my-library"
version = "0.1.0"

[dependencies]
uuid = "1.0.1"

[patch.crates-io]
uuid = { git = 'https://github.com/uuid-rs/uuid' }
```

请注意，我们对 `uuid` 的本地依赖已更新到 `1.0.1`，因为一旦该 crate 发布，我们实际上就需要这个版本。然而，这个版本在 crates.io 上并不存在，所以我们通过清单的 `[patch]` 部分来提供它。

现在，当我们的库被构建时，它将从 git 仓库获取 `uuid`，并解析到仓库内的 1.0.1 版本，而不是试图从 crates.io 下载一个版本。一旦 1.0.1 在 crates.io 上发布，`[patch]` 部分就可以被删除。

还需要注意的是，`[patch]` 具有*传递性*。假设你在一个更大的包中使用 `my-library`，例如：

```toml
[package]
name = "my-binary"
version = "0.1.0"

[dependencies]
my-library = { git = 'https://example.com/git/my-library' }
uuid = "1.0"

[patch.crates-io]
uuid = { git = 'https://github.com/uuid-rs/uuid' }
```

请记住，`[patch]` 具有*传递性*，但只能在*顶层*定义，所以作为 `my-library` 的使用者，我们必要时必须重复 `[patch]` 部分。不过在这里，新的 `uuid` crate 适用于我们对 `uuid` 的依赖*以及* `my-library -> uuid` 的依赖。对于整个 crate 图，`uuid` crate 将被解析为一个版本，即 1.0.1，并且它将从 git 仓库拉取。

#### 覆盖仓库URL

如果你想要覆盖的依赖项不是从 `crates.io` 加载的，那么你使用 `[patch]` 的方式需要稍作改变。例如，如果依赖项是一个 git 依赖项，你可以像下面这样将其覆盖为本地路径：

```toml
[patch."https://github.com/your/repository"]
my-library = { path = "../my-library/path" }
```

就是这样！

### 预发布破坏性变更

让我们来看一下如何与一个 crate 的新主要版本（通常伴随着破坏性变更）一起工作。沿用我们之前的 crate，这意味着我们将要创建 `uuid` crate 的 2.0.0 版本。在我们向上游提交了所有更改之后，我们可以将 `my-library` 的清单更新为如下内容：

```toml
[dependencies]
uuid = "2.0"

[patch.crates-io]
uuid = { git = "https://github.com/uuid-rs/uuid", branch = "2.0.0" }
```

就是这样！和前面的例子一样，2.0.0 版本实际上并不存在于 crates.io 上，但我们仍然可以通过使用 `[patch]` 部分，将其作为 git 依赖项引入。作为思考练习，让我们再看看上面的 `my-binary` 清单：

```toml
[package]
name = "my-binary"
version = "0.1.0"

[dependencies]
my-library = { git = 'https://example.com/git/my-library' }
uuid = "1.0"

[patch.crates-io]
uuid = { git = 'https://github.com/uuid-rs/uuid', branch = '2.0.0' }
```

请注意，这实际上会解析出两个版本的 `uuid` crate。`my-binary` crate 将继续使用 `uuid` crate 的 1.x.y 系列，但 `my-library` crate 将使用 `uuid` 的 `2.0.0` 版本。这将允许你通过依赖图逐步推出一个 crate 的破坏性变更，而无需强制一次性更新所有内容。

### 对多个版本使用`[patch]`

你可以通过使用用于重命名依赖项的 `package` 键，来为同一个 crate 的多个版本打补丁。例如，假设 `serde` crate 有一个我们想要用到其 `1.*` 系列的 bug 修复，但同时我们也想使用我们 git 仓库中的 `2.0.0` 版本的 serde 进行原型设计。为此，我们可以这样配置：

```toml
[patch.crates-io]
serde = { git = 'https://github.com/serde-rs/serde' }
serde2 = { git = 'https://github.com/example/serde', package = 'serde', branch = 'v2' }
```

第一个 `serde = ...` 指令表示 `1.*` 版本的 serde 应该从 git 仓库获取（拉入我们需要的 bug 修复），而第二个 `serde2 = ...` 指令表示 `serde` 包也应该从 `https://github.com/example/serde` 的 `v2` 分支拉取。我们在此假设该分支上的 `Cargo.toml` 声明了版本 `2.0.0`。

请注意，当使用 `package` 键时，这里的 `serde2` 标识符实际上是被忽略的。我们只需要一个不与其他已打补丁的 crate 冲突的唯一名称。

### `[patch]`部分

`Cargo.toml` 的 `[patch]` 部分可以用来用其他副本覆盖依赖项。其语法类似于 [`[dependencies]`][dependencies] 部分：

```toml
[patch.crates-io]
foo = { git = 'https://github.com/example/foo' }
bar = { path = 'my/local/bar' }

[dependencies.baz]
git = 'https://github.com/example/baz'

[patch.'https://github.com/example/baz']
baz = { git = 'https://github.com/example/patched-baz', branch = 'my-branch' }
```

`[patch]` 表由类似依赖项的子表组成。`[patch]` 之后的每个键都是正在被打补丁的源的 URL 或注册表的名称。名称 `crates-io` 可用于覆盖默认的注册表 [crates.io]。上面示例中的第一个 `[patch]` 演示了如何覆盖 [crates.io]，第二个 `[patch]` 演示了如何覆盖一个 git 源。

这些表中的每个条目都是一个普通的依赖项规范，与清单的 `[dependencies]` 部分中的相同。列在 `[patch]` 部分中的依赖项会被解析并用于修补指定 URL 的源。上面的清单片段用 `foo` crate 和 `bar` crate 修补了 `crates-io` 源（例如 crates.io 本身）。它还用来自其他地方的 `my-branch` 修补了 `https://github.com/example/baz` 源。

源（source）可以用不存在的 crate 版本来修补，也可以用已经存在的 crate 版本来修补。如果一个源被一个已经存在于该源中的 crate 版本修补，那么源的原始 crate 将被替换。

### `[replace]`部分

> **注意**：`[replace]` 已废弃。你应该使用 [`[patch]`](#patch部分) 表来代替。

Cargo.toml 的这一部分可以用来用其他副本覆盖依赖项。其语法类似于 `[dependencies]` 部分：

```toml
[replace]
"foo:0.1.0" = { git = 'https://github.com/example/foo' }
"bar:1.0.2" = { path = 'my/local/bar' }
```

`[replace]` 表中的每个键都是一个 [包 ID 规格](pkgid-spec.md)，它允许任意选择依赖图中的一个节点进行覆盖（需要完整的 3 段式版本号）。每个键的值与指定依赖项的 `[dependencies]` 语法相同，只是不能指定特性（features）。请注意，当一个 crate 被覆盖时，用于覆盖它的副本必须具有相同的名称和版本，但可以来自不同的源（例如，git 或本地路径）。

### `paths`覆盖

有时你只是临时处理一个 crate，并且你不想像上面的 `[patch]` 部分那样必须修改 `Cargo.toml`。对于这种用例，Cargo 提供了一个限制性更强的覆盖版本，称为**路径覆盖（path overrides）**。

路径覆盖通过 [`.cargo/config.toml`](config.md) 指定，而不是 `Cargo.toml`。在 `.cargo/config.toml` 中，你需要指定一个名为 `paths` 的键：

```toml
paths = ["/path/to/uuid"]
```

这个数组应该填入包含 `Cargo.toml` 的目录。在这个例子中，我们只添加了 `uuid`，所以它将是唯一被覆盖的。这个路径可以是绝对路径，也可以是相对于包含 `.cargo` 文件夹的目录的相对路径。

然而，路径覆盖比 `[patch]` 部分的限制更多，因为它们不能改变依赖图的结构。当使用路径替换时，先前的依赖集合必须与新的 `Cargo.toml` 规范完全匹配。例如，这意味着路径覆盖不能用于测试向一个 crate 添加依赖项，在这种情况下必须使用 `[patch]`。因此，使用路径覆盖通常仅限于快速的 bug 修复，而不是较大的变更。

注意：使用本地配置来覆盖路径仅适用于已发布到 [crates.io] 的 crate。你不能使用此功能来告诉 Cargo 如何查找本地未发布的 crate。

[crates.io]: https://crates.io/
[multiple locations]: specifying-dependencies.md#multiple-locations
[dependencies]: specifying-dependencies.md