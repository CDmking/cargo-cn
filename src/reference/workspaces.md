## 工作空间

*工作空间（workspace）* 是一个或多个包的集合，它们共享相同的依赖解析（拥有同一个 `Cargo.lock` 文件）、输出目录以及诸如配置（profile）等各种设置。作为工作空间一部分的包被称为*工作空间成员（workspace members）*。工作空间有两种形式：作为根包（root package）或作为虚拟清单（virtual manifest）。

### 根包

可以通过在 `Cargo.toml` 中添加 [`[workspace]` 部分](#workspace-部分) 来创建工作空间。这可以添加到一个已经定义了 `[package]` 的 `Cargo.toml` 中，在这种情况下，该包就是工作空间的*根包（root package）*。*工作空间根目录（workspace root）* 是工作空间的 `Cargo.toml` 文件所在的目录。

### 虚拟清单

或者，可以创建一个包含 `[workspace]` 部分但没有 [`[package]` 部分][package] 的 `Cargo.toml` 文件。这被称为*虚拟清单（virtual manifest）*。当没有一个“主”包，或者你想将所有包组织在单独的目录中时，这通常很有用。

### 关键特性

工作空间的要点包括：

* 所有包共享一个位于*工作空间根目录* 的公共 `Cargo.lock` 文件。
* 所有包共享一个公共的[输出目录][output directory]，该目录默认为*工作空间根目录* 中名为 `target` 的目录。
* `Cargo.toml` 中的 [`[patch]`][patch]、[`[replace]`][replace] 和 [`[profile.*]`][profiles] 部分仅在*根*清单中被识别，而在成员 crate 的清单中被忽略。

### `[workspace]` 部分

`Cargo.toml` 中的 `[workspace]` 表定义了哪些包是工作空间的成员：

```toml
[workspace]
members = ["member1", "path/to/member2", "crates/*"]
exclude = ["crates/foo", "path/to/other"]
```

所有位于工作空间目录中的 [`path` 依赖][`path` dependencies] 自动成为成员。可以通过 `members` 键列出额外的成员，它应该是一个字符串数组，包含包含 `Cargo.toml` 文件的目录。

`members` 列表还支持使用 [globs] 来匹配多个路径，使用典型的文件名通配模式，如 `*` 和 `?`。

`exclude` 键可用于防止路径被包含在工作空间中。如果某些路径依赖根本不希望被包含在工作空间中，或者在使用通配模式时你想排除某个目录，这会很有用。

一个空的 `[workspace]` 表可以与 `[package]` 一起使用，以方便地创建一个包含该包及其所有路径依赖的工作空间。

### 工作空间选择

当位于工作空间内的子目录中时，Cargo 会自动搜索父目录中带有 `[workspace]` 定义的 `Cargo.toml` 文件，以确定使用哪个工作空间。成员 crate 的清单中的 [`package.workspace`] 键可用于指向工作空间的根目录，从而覆盖此自动搜索。如果成员不在工作空间根目录的子目录中，手动设置会很有用。

### 包选择

在工作空间中，与包相关的 cargo 命令（如 [`cargo build`]）可以使用 `-p` / `--package` 或 `--workspace` 命令行标志来确定要对哪些包进行操作。如果未指定这些标志，Cargo 将使用当前工作目录中的包。如果当前目录是虚拟工作空间，它将应用于所有成员（就像在命令行中指定了 `--workspace` 一样）。

可以指定可选的 `default-members` 键来设置在处于工作空间根目录且未使用包选择标志时要操作的成员：

```toml
[workspace]
members = ["path/to/member1", "path/to/member2", "path/to/member3/*"]
default-members = ["path/to/member2", "path/to/member3/foo"]
```

当指定时，`default-members` 必须扩展为 `members` 的子集。

### `workspace.metadata` 表

Cargo 会忽略 `workspace.metadata` 表且不会发出警告。这个部分可用于希望将工作空间配置存储在 `Cargo.toml` 中的工具。例如：

```toml
[workspace]
members = ["member1", "member2"]

[workspace.metadata.webcontents]
root = "path/to/webproject"
tool = ["npm", "run", "build"]
# ...
```

在包级别也有一个类似的表集合：[`package.metadata`][package-metadata]。虽然 Cargo 没有为这些表的内容指定格式，但建议外部工具可以以一致的方式使用它们，例如，如果数据在 `package.metadata` 中缺失，并且对于相关工具来说有意义，则可以引用 `workspace.metadata` 中的数据。

[package]: manifest.md#the-package-section
[package-metadata]: manifest.md#the-metadata-table
[output directory]: ../guide/build-cache.md
[patch]: overriding-dependencies.md#the-patch-section
[replace]: overriding-dependencies.md#the-replace-section
[profiles]: profiles.md
[`path` dependencies]: specifying-dependencies.md#specifying-path-dependencies
[`package.workspace`]: manifest.md#the-workspace-field
[globs]: https://docs.rs/glob/0.3.0/glob/struct.Pattern.html
[`cargo build`]: ../commands/cargo-build.md