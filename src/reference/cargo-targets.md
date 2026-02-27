## Cargo Targets

Cargo 包由*targets*组成，这些targets对应着可以编译成 crate 的源文件。包可以包含[库](#库)、[二进制程序](#二进制程序)、[示例](#examples)、[测试](#tests)和[基准测试](#基准测试benchmarks)targets。targets列表可以在 `Cargo.toml` 清单中配置，通常由源文件的[目录布局][包布局]自动推断。

有关配置targets设置的详细信息，请参阅下文 [配置target](#配置target)。

### 库

库targets定义了一个可以被其他库和可执行程序使用和链接的“库”。源文件默认路径为 `src/lib.rs`，库的名称默认为包的名称。一个包只能有一个库。库的设置可以在 `Cargo.toml` 的 `[lib]` 表中进行[自定义][customized]。

```toml
# 在 Cargo.toml 中自定义库的示例。
[lib]
crate-type = ["cdylib"]
bench = false
```

### 二进制程序

二进制程序targets是编译后可运行的可执行程序。默认的二进制程序源文件是 `src/main.rs`，其名称默认为包的名称。额外的二进制程序存储在 [`src/bin/` 目录][package layout]中。每个二进制程序的设置可以在 `Cargo.toml` 的 `[[bin]]` 表中进行[自定义][customized]。

二进制程序可以使用包的库的公共 API。它们也会与 `Cargo.toml` 中定义的 [`[dependencies]`][dependencies] 链接。

你可以使用 [`cargo run`] 命令配合 `--bin <bin-name>` 选项来运行单个二进制程序。[`cargo install`] 可以用于将可执行文件复制到一个公共位置。

```toml
# 在 Cargo.toml 中自定义二进制程序的示例。
[[bin]]
name = "cool-tool"
test = false
bench = false

[[bin]]
name = "frobnicator"
required-features = ["frobnicate"]
```

### Examples

位于 [`examples` 目录][package layout] 下的文件是展示库所提供的功能用法的示例。编译后，它们会被放置在 [`target/debug/examples` 目录][build cache] 中。

Examples可以使用包的库的公共 API。它们也会与 `Cargo.toml` 中定义的 [`[dependencies]`][dependencies] 和 [`[dev-dependencies]`][dev-dependencies] 链接。

默认情况下，examples是可执行的二进制程序（具有 `main()` 函数）。你可以指定 [`crate-type` 字段](#crate-type-字段) 来让示例编译为库：

```toml
[[example]]
name = "foo"
crate-type = ["staticlib"]
```

你可以使用 [`cargo run`] 命令配合 `--example <example-name>` 选项来运行单个可执行示例。库示例可以使用 [`cargo build`] 并指定 `--example <example-name>` 选项来构建。[`cargo install`] 配合 `--example <example-name>` 选项可以用于将可执行二进制程序复制到一个公共位置。示例在默认情况下会由 [`cargo test`] 编译，以防止其因长期未使用而无法编译。如果示例中包含你想要用 [`cargo test`] 运行的 `#[test]` 函数，请将 [`test` 字段](#test-字段) 设置为 `true`。

### Tests

Cargo 项目中有两种test：

*   *单元测试（Unit tests）*：这些是位于库或二进制程序（或任何启用了 [`test` 字段](#test-字段) 的targets）中、标记有 [`#[test]` 属性][test-attribute] 的函数。这些测试可以访问它们所在targets中的私有 API。
*   *集成测试（Integration tests）*：这是一个独立的可执行二进制程序，同样包含 `#[test]` 函数，它链接到项目的库并可以访问其*公共* API。

Tests通过 [`cargo test`] 命令运行。默认情况下，Cargo 和 `rustc` 使用 libtest 测试框架，它负责收集标有 [`#[test]` 属性][test-attribute] 的函数并并行执行它们，报告每个测试的成功与失败。如果你想使用不同的测试框架或测试策略，请参阅 [`harness` 字段](#harness-字段)。

#### 集成测试（Integration tests）

位于 [`tests` 目录][package layout] 下的文件是集成测试。当你运行 [`cargo test`] 时，Cargo 会将每个这样的文件编译为单独的 crate 并执行它们。

集成测试可以使用包的库的公共 API。它们也会与 `Cargo.toml` 中定义的 [`[dependencies]`][dependencies] 和 [`[dev-dependencies]`][dev-dependencies] 链接。

如果你想在多个集成测试之间共享代码，可以将其放在单独的模块中，例如 `tests/common/mod.rs`，然后在每个测试中通过 `mod common;` 导入它。

每个集成测试都会生成一个单独的可执行二进制程序，[`cargo test`] 会串行运行它们。在某些情况下，这可能效率不高，因为编译时间会更长，并且在运行测试时可能无法充分利用多核 CPU。如果你有很多集成测试，可以考虑创建一个单一的集成测试，并将测试分割成多个模块。libtest 测试框架会自动找到所有标有 `#[test]` 的函数并并行运行它们。你可以将模块名称传递给 [`cargo test`] 来仅运行该模块内的测试。

如果存在集成测试，二进制targets会自动被构建。这使得集成测试可以执行二进制程序以测试其行为。集成测试构建时，会设置 `CARGO_BIN_EXE_<name>` [环境变量]，以便使用 [`env` 宏] 来定位可执行文件。

[环境变量]: environment-variables.md#environment-variables-cargo-sets-for-crates
[`env` 宏]: ../../std/macro.env.html

### 基准测试（Benchmarks）

基准测试（Benchmarks）提供了一种使用 [`cargo bench`] 命令测试代码性能的方法。它们的结构与[测试](#tests)类似，每个基准测试函数都标有 `#[bench]` 属性。与测试类似：

*   基准测试位于 [`benches` 目录][package layout]。
*   在库和二进制程序中定义的基准测试函数可以访问它们所在targets中的*私有* API。`benches` 目录中的基准测试可以使用*公共* API。
*   [`bench` 字段](#bench-字段) 可用于定义默认情况下哪些targets被基准测试。
*   [`harness` 字段](#harness-字段) 可用于禁用内置的测试框架。

> **注意**：[`#[bench]` 属性](../../unstable-book/library-features/test.html) 目前还不稳定，仅在 [nightly 频道][nightly channel] 中可用。在 [crates.io](https://crates.io/keywords/benchmark) 上有一些包可以帮助在稳定频道上运行基准测试，例如 [Criterion](https://crates.io/crates/criterion)。

### 配置target

`Cargo.toml` 中的所有 `[lib]`、`[[bin]]`、`[[example]]`、`[[test]]` 和 `[[bench]]` 部分都支持类似的配置，用于指定应如何构建targets。像 `[[bin]]` 这样的双括号部分是 [TOML 的数组表格](https://toml.io/en/v1.0.0-rc.3#array-of-tables)，这意味着你可以编写多个 `[[bin]]` 部分来在你的 crate 中创建多个可执行程序。你只能指定一个库，因此 `[lib]` 是一个普通的 TOML 表。

以下是每个targets的 TOML 设置概述，每个字段的详细说明如下。

```toml
[lib]
name = "foo"           # target的名称。
path = "src/lib.rs"    # target的源文件。
test = true            # 默认情况下是否进行测试。
doctest = true         # 默认情况下是否测试文档示例。
bench = true           # 默认情况下是否进行基准测试。
doc = true             # 默认情况下是否包含在文档中。
plugin = false         # 用作编译器插件（已弃用）。
proc-macro = false     # 对于过程宏库，设置为 `true`。
harness = true         # 使用 libtest 测试框架。
edition = "2015"       # targets使用的 Rust 版本。
crate-type = ["lib"]   # 要生成的 crate 类型。
required-features = [] # 构建此targets所需的功能（不适用于 lib）。
```

#### `name` 字段

`name` 字段指定target的名称，对应于将生成的产物的文件名。对于库，这是依赖项将用来引用它的 crate 名称。

对于 `[lib]` 和默认的二进制程序（`src/main.rs`），此名称默认为包的名称，其中的任何连字符（-）都将替换为下划线（_）。对于其他[自动发现](#targets自动发现)的targets，它默认为目录或文件名。

除了 `[lib]` 之外，所有targets都需要此字段。

#### `path` 字段

`path` 字段指定 crate 源文件的位置，相对于 `Cargo.toml` 文件。

如果未指定，则使用基于target名称的[推断路径](#targets自动发现)。

#### `test` 字段

`test` 字段指示target是否默认由 [`cargo test`] 进行测试。对于库、二进制程序和测试，默认值为 `true`。

> **注意**：Examples默认由 [`cargo test`] 构建以确保它们能继续编译，但默认情况下不会*测试*它们。将示例的 `test` 设置为 `true` 也会将其作为测试构建，并运行示例中定义的任何 [`#[test]`][test-attribute] 函数。

#### `doctest` 字段

`doctest` 字段指示[文档示例]是否默认由 [`cargo test`] 进行测试。这仅与库相关，对其他部分没有影响。对于库，默认值为 `true`。

#### `bench` 字段

`bench` 字段指示targets是否默认由 [`cargo bench`] 进行基准测试。对于库、二进制程序和基准测试，默认值为 `true`。

#### `doc` 字段

`doc` 字段指示targets是否默认包含在 [`cargo doc`] 生成的文档中。对于库和二进制程序，默认值为 `true`。

> **注意**：如果二进制程序的名称与库targets相同，则会被跳过。

#### `plugin` 字段

此字段用于 `rustc` 插件，该功能已弃用。

#### `proc-macro` 字段

`proc-macro` 字段指示该库是一个[过程宏][procedural macro]（[参考][proc-macro-reference]）。这仅对 `[lib]` targets有效。

#### `harness` 字段

`harness` 字段指示 [`--test` 标志][`--test` flag] 将传递给 `rustc`，这将自动包含 libtest 库，该库是收集和运行标有 [`#[test]` 属性][test-attribute] 的测试或标有 `#[bench]` 属性的基准测试的驱动。对于所有targets，默认值为 `true`。

如果设置为 `false`，则需要你自己定义 `main()` 函数来运行测试和基准测试。

无论是否启用测试框架，测试都会启用 [`cfg(test)` 条件表达式][cfg-test]。

#### `edition` 字段

`edition` 字段定义了targets将使用的 [Rust 版本][Rust Edition]。如果未指定，则默认为 `[package]` 的 [`edition` 字段][package-edition]。通常不应设置此字段，仅用于高级场景，例如逐步将大型包迁移到新版本。

#### `crate-type` 字段

`crate-type` 字段定义了targets将生成的 [crate 类型][crate types]。它是一个字符串数组，允许你为单个targets指定多种 crate 类型。这只能为库和示例指定。二进制程序、测试和基准测试始终是 "bin" crate 类型。默认值如下：

targets | Crate 类型
-------|-----------
普通库 | `"lib"`
过程宏库 | `"proc-macro"`
示例 | `"bin"`

可用选项包括 `bin`、`lib`、`rlib`、`dylib`、`cdylib`、`staticlib` 和 `proc-macro`。你可以在 [Rust 参考手册][crate types] 中阅读更多关于不同 crate 类型的信息。

#### `required-features` 字段

`required-features` 字段指定了构建targets所需的[功能][features]。如果所需功能中有任何一项未启用，则跳过该targets。这仅与 `[[bin]]`、`[[bench]]`、`[[test]]` 和 `[[example]]` 部分相关，对 `[lib]` 没有影响。

```toml
[features]
# ...
postgres = []
sqlite = []
tools = []

[[bin]]
name = "my-pg-tool"
required-features = ["postgres", "tools"]
```

### targets自动发现

默认情况下，Cargo 根据文件系统上的[文件布局][package layout]自动确定要构建的targets。targets配置表，如 `[lib]`、`[[bin]]`、`[[test]]`、`[[bench]]` 或 `[[example]]`，可用于添加不遵循标准目录布局的额外targets。

可以禁用自动targets发现，以便仅构建手动配置的targets。在 `[package]` 部分将 `autobins`、`autoexamples`、`autotests` 或 `autobenches` 键设置为 `false` 将禁用相应targets类型的自动发现。

```toml
[package]
# ...
autobins = false
autoexamples = false
autotests = false
autobenches = false
```

禁用自动发现通常仅在特殊情况下需要。例如，如果你有一个库，并且你想要一个名为 `bin` 的*模块*，这将带来问题，因为 Cargo 通常会将 `bin` 目录中的任何文件作为可执行程序编译。以下是此场景的示例布局：

```text
├── Cargo.toml
└── src
    ├── lib.rs
    └── bin
        └── mod.rs
```

为了防止 Cargo 将 `src/bin/mod.rs` 推断为可执行程序，请在 `Cargo.toml` 中设置 `autobins = false` 以禁用自动发现：

```toml
[package]
# …
autobins = false
```

> **注意**：对于 2015 版本的包，如果在 `Cargo.toml` 中至少手动定义了一个targets，则自动发现的默认值为 `false`。从 2018 版本开始，默认值始终为 `true`。

[Build cache]: ../guide/build-cache.md
[Rust Edition]: ../../edition-guide/index.html
[`--test` flag]: ../../rustc/command-line-arguments.html#option-test
[`cargo bench`]: ../commands/cargo-bench.md
[`cargo build`]: ../commands/cargo-build.md
[`cargo doc`]: ../commands/cargo-doc.md
[`cargo install`]: ../commands/cargo-install.md
[`cargo run`]: ../commands/cargo-run.md
[`cargo test`]: ../commands/cargo-test.md
[cfg-test]: ../../reference/conditional-compilation.html#test
[crate types]: ../../reference/linkage.html
[crates.io]: https://crates.io/
[customized]: #配置target
[dependencies]: specifying-dependencies.md
[dev-dependencies]: specifying-dependencies.md#development-dependencies
[documentation examples]: ../../rustdoc/documentation-tests.html
[features]: features.md
[nightly channel]: ../../book/appendix-07-nightly-rust.html
[package layout]: ../guide/project-layout.md
[package-edition]: manifest.md#the-edition-field
[proc-macro-reference]: ../../reference/procedural-macros.html
[procedural macro]: ../../book/ch19-06-macros.html
[test-attribute]: ../../reference/attributes/testing.html#the-test-attribute