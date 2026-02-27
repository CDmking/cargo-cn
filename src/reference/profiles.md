## Profile

Profile 提供了一种改变编译器设置的方式，从而影响优化和调试符号等事项。

Cargo 有 4 个内置的 profile：`dev`、`release`、`test` 和 `bench`。它会根据运行的命令、正在构建的包和目标，以及像 `--release` 这样的命令行标志，自动选择合适的 profile。选择过程[如下所述](#profile-选择)。

可以在 [`Cargo.toml`](manifest.md) 中使用 `[profile]` 表修改 profile 设置。在每个命名的 profile 内部，可以通过键/值对来更改单独的设置，如下所示：

```toml
[profile.dev]
opt-level = 1               # 使用稍好一些的优化。
overflow-checks = false     # 禁用整数溢出检查。
```

Cargo 仅查看工作空间根目录的 `Cargo.toml` 清单中的 profile 设置。依赖项中定义的 profile 设置将被忽略。

此外，可以通过 [config] 定义覆盖 profile 设置。在配置文件或环境变量中指定 profile 将覆盖来自 `Cargo.toml` 的设置。

[config]: config.md

### Profile 设置

以下是可以由 profile 控制的设置列表。

#### opt-level

`opt-level` 设置控制 [`-C opt-level` 标志]，该标志控制优化级别。更高的优化级别可能会产生运行更快的代码，但代价是更长的编译时间。更高的级别也可能改变和重新排列编译后的代码，这可能使其更难与调试器一起使用。

有效选项包括：

* `0`：无优化
* `1`：基本优化
* `2`：部分优化
* `3`：完全优化
* `"s"`：优化二进制大小
* `"z"`：优化二进制大小，同时关闭循环向量化。

建议尝试不同的级别，以为你的项目找到合适的平衡点。可能会有意想不到的结果，例如级别 `3` 可能比 `2` 更慢，或者 `"s"` 和 `"z"` 级别生成的不一定更小。随着 `rustc` 新版本改变优化行为，你可能也需要重新评估你的设置。

另请参阅 [Profile Guided Optimization] 以了解更高级的优化技术。

[`-C opt-level` 标志]: ../../rustc/codegen-options/index.html#opt-level
[Profile Guided Optimization]: ../../rustc/profile-guided-optimization.html

#### debug

`debug` 设置控制 [`-C debuginfo` 标志]，该标志控制编译后的二进制文件中包含的调试信息量。

有效选项包括：

* `0` 或 `false`：完全没有调试信息
* `1`：仅行号表
* `2` 或 `true`：完整的调试信息

根据你的需求，你可能还需要配置 [`split-debuginfo`](#split-debuginfo) 选项。

[`-C debuginfo` 标志]: ../../rustc/codegen-options/index.html#debuginfo

#### split-debuginfo

`split-debuginfo` 设置控制 [`-C split-debuginfo` 标志]，该标志控制调试信息（如果生成）是放在可执行文件本身还是其旁边。

此选项是一个字符串，可接受的值与[编译器接受的][`-C split-debuginfo` 标志]相同。对于启用了调试信息的 profile，此选项在 macOS 上的默认值是 `unpacked`。否则，此选项的默认值[记录在 rustc 的文档中][`-C split-debuginfo` 标志]，且与平台相关。某些选项仅在 [nightly 频道] 上可用。一旦进行了更多测试，并且对 DWARF 的支持稳定下来，Cargo 的默认值将来可能会更改。

[nightly 频道]: ../../book/appendix-07-nightly-rust.html
[`-C split-debuginfo` 标志]: ../../rustc/codegen-options/index.html#split-debuginfo

#### debug-assertions

`debug-assertions` 设置控制 [`-C debug-assertions` 标志]，该标志打开或关闭 `cfg(debug_assertions)` [条件编译]。Debug 断言旨在包含仅在 debug/development 构建中可用的运行时验证。这些验证可能是在 release 构建中过于昂贵或不希望存在的。Debug 断言启用了标准库中的 [`debug_assert!` 宏]。

有效选项包括：

* `true`：启用
* `false`：禁用

[`-C debug-assertions` 标志]: ../../rustc/codegen-options/index.html#debug-assertions
[条件编译]: ../../reference/conditional-compilation.md#debug_assertions
[`debug_assert!` 宏]: ../../std/macro.debug_assert.html

#### overflow-checks

`overflow-checks` 设置控制 [`-C overflow-checks` 标志]，该标志控制[运行时整数溢出]的行为。当溢出检查启用时，溢出会发生 panic。

有效选项包括：

* `true`：启用
* `false`：禁用

[`-C overflow-checks` 标志]: ../../rustc/codegen-options/index.html#overflow-checks
[运行时整数溢出]: ../../reference/expressions/operator-expr.md#overflow

#### lto

`lto` 设置控制 [`-C lto` 标志]，该标志控制 LLVM 的[链接时优化]。LTO 可以通过全程序分析产生更好的优化代码，但代价是更长的链接时间。

有效选项包括：

* `false`：执行"精简局部 LTO（thin local LTO）"，仅对本地 crate 在其[代码生成单元](#codegen-units)范围内执行"精简（thin）" LTO。如果代码生成单元是 1 或 [opt-level](#opt-level) 为 0，则不执行 LTO。
* `true` 或 `"fat"`：执行"完整（fat）" LTO，尝试对依赖图中的所有 crate 执行优化。
* `"thin"`：执行["精简（thin）" LTO]。这与"完整（fat）"类似，但运行时间大幅减少，同时仍能实现与"完整（fat）"相似的性能提升。
* `"off"`：禁用 LTO。

另请参阅用于跨语言 LTO 的 [`-C linker-plugin-lto`] `rustc` 标志。

[`-C lto` 标志]: ../../rustc/codegen-options/index.html#lto
[链接时优化]: https://llvm.org/docs/LinkTimeOptimization.html
[`-C linker-plugin-lto`]: ../../rustc/codegen-options/index.html#linker-plugin-lto
["精简（thin）" LTO]: http://blog.llvm.org/2016/06/thinlto-scalable-and-incremental-lto.html

#### panic

`panic` 设置控制 [`-C panic` 标志]，该标志控制使用哪种 panic 策略。

有效选项包括：

* `"unwind"`：在 panic 时展开堆栈。
* `"abort"`：在 panic 时终止进程。

当设置为 `"unwind"` 时，实际值取决于目标平台的默认值。例如，NVPTX 平台不支持展开，因此它总是使用 `"abort"`。

测试、基准测试、构建脚本和过程宏会忽略 `panic` 设置。`rustc` 测试框架目前需要 `unwind` 行为。请参阅启用 `abort` 行为的 [`panic-abort-tests`] 不稳定标志。

此外，当使用 `abort` 策略并构建测试时，所有依赖项也将被迫使用 `unwind` 策略构建。

[`-C panic` 标志]: ../../rustc/codegen-options/index.html#panic
[`panic-abort-tests`]: unstable.md#panic-abort-tests

#### incremental

`incremental` 设置控制 [`-C incremental` 标志]，该标志控制是否启用增量编译。增量编译导致 `rustc` 将额外信息保存到磁盘，这些信息将在重新编译 crate 时重用，从而提高重新编译速度。附加信息存储在 `target` 目录中。

有效选项包括：

* `true`：启用
* `false`：禁用

增量编译仅用于工作空间成员和"路径（path）"依赖项。

增量值可以通过 `CARGO_INCREMENTAL` [环境变量] 或 [`build.incremental`] 配置变量全局覆盖。

[`-C incremental` 标志]: ../../rustc/codegen-options/index.html#incremental
[环境变量]: environment-variables.md
[`build.incremental`]: config.md#buildincremental

#### codegen-units

`codegen-units` 设置控制 [`-C codegen-units` 标志]，该标志控制一个 crate 将被分割成多少个"代码生成单元"。更多的代码生成单元允许并行处理更多 crate 内容，可能减少编译时间，但可能产生更慢的代码。

此选项接受一个大于 0 的整数。

对于[增量](#incremental)构建，默认值为 256，对于非增量构建，默认值为 16。

[`-C codegen-units` 标志]: ../../rustc/codegen-options/index.html#codegen-units

#### rpath

`rpath` 设置控制 [`-C rpath` 标志]，该标志控制是否启用 [`rpath`]。

[`-C rpath` 标志]: ../../rustc/codegen-options/index.html#rpath
[`rpath`]: https://en.wikipedia.org/wiki/Rpath

### 默认 profile

#### dev

`dev` profile 用于常规开发和调试。它是诸如 [`cargo build`] 之类的构建命令的默认 profile。

`dev` profile 的默认设置为：

```toml
[profile.dev]
opt-level = 0
debug = true
split-debuginfo = '...'  # 平台特定。
debug-assertions = true
overflow-checks = true
lto = false
panic = 'unwind'
incremental = true
codegen-units = 256
rpath = false
```

#### release

`release` profile 用于发布和生产环境中使用的优化产物。当使用 `--release` 标志时，将使用此 profile，并且它是 [`cargo install`] 的默认 profile。

`release` profile 的默认设置为：

```toml
[profile.release]
opt-level = 3
debug = false
split-debuginfo = '...'  # 平台特定。
debug-assertions = false
overflow-checks = false
lto = false
panic = 'unwind'
incremental = false
codegen-units = 16
rpath = false
```

#### test

`test` profile 用于构建测试，或者当使用 `cargo build` 以 debug 模式构建基准测试时。

`test` profile 的默认设置为：

```toml
[profile.test]
opt-level = 0
debug = 2
split-debuginfo = '...'  # 平台特定。
debug-assertions = true
overflow-checks = true
lto = false
panic = 'unwind'    # 此设置总是被忽略。
incremental = true
codegen-units = 256
rpath = false
```

#### bench

`bench` profile 用于构建基准测试，或者当使用 `--release` 标志构建测试时。

`bench` profile 的默认设置为：

```toml
[profile.bench]
opt-level = 3
debug = false
split-debuginfo = '...'  # 平台特定。
debug-assertions = false
overflow-checks = false
lto = false
panic = 'unwind'    # 此设置总是被忽略。
incremental = false
codegen-units = 16
rpath = false
```

#### 构建依赖项

默认情况下，所有 profile 都不优化构建依赖项（构建脚本、过程宏及其依赖项）。构建覆盖的默认设置为：

```toml
[profile.dev.build-override]
opt-level = 0
codegen-units = 256

[profile.release.build-override]
opt-level = 0
codegen-units = 256
```

构建依赖项在其他方面会继承正在使用的活跃 profile 的设置，如下所述。

### Profile 选择

使用的 profile 取决于命令、包、Cargo target 以及像 `--release` 这样的命令行标志。

构建命令如 [`cargo build`]、[`cargo rustc`]、[`cargo check`] 和 [`cargo run`] 默认使用 `dev` profile。`--release` 标志可用于切换到 `release` profile。

[`cargo install`] 命令默认使用 `release` profile，并且可以使用 `--debug` 标志切换到 `dev` profile。

测试目标默认使用 `test` profile 构建。`--release` 标志将测试切换到 `bench` profile。

基准测试目标默认使用 `bench` profile 构建。[`cargo build`] 命令可用于以 `test` profile 构建基准测试目标以启用调试。

请注意，当使用 [`cargo test`] 和 [`cargo bench`] 命令时，`test`/`bench` profile 仅适用于最终的测试可执行文件。依赖项将继续使用 `dev`/`release` profile。另请注意，当为单元测试构建库时，该库是使用 `test` profile 构建的。然而，当构建集成测试目标时，库目标是使用 `dev` profile 构建并链接到集成测试可执行文件中的。

![cargo test 的 profile 选择](../images/profile-selection.svg)

[`cargo bench`]: ../commands/cargo-bench.md
[`cargo build`]: ../commands/cargo-build.md
[`cargo check`]: ../commands/cargo-check.md
[`cargo install`]: ../commands/cargo-install.md
[`cargo run`]: ../commands/cargo-run.md
[`cargo rustc`]: ../commands/cargo-rustc.md
[`cargo test`]: ../commands/cargo-test.md

### 覆盖

Profile 设置可以针对特定的包和构建时的 crate 进行覆盖。要覆盖特定包的设置，请使用 `package` 表来更改命名包的设置：

```toml
# `foo` 包将使用 -Copt-level=3 标志。
[profile.dev.package.foo]
opt-level = 3
```

包名实际上是一个 [Package ID Spec](pkgid-spec.md)，因此你可以使用诸如 `[profile.dev.package."foo:2.1.0"]` 的语法来定位包的特定版本。

要覆盖所有依赖项（但不包括任何工作空间成员）的设置，请使用 `"*"` 包名：

```toml
# 设置依赖项的默认值。
[profile.dev.package."*"]
opt-level = 2
```

要覆盖构建脚本、过程宏及其依赖项的设置，请使用 `build-override` 表：

```toml
# 设置构建脚本和过程宏的设置。
[profile.dev.build-override]
opt-level = 3
```

> 注意：当一个依赖项同时是普通依赖项和构建依赖项时，Cargo 在未指定 `--target` 的情况下会尝试只构建它一次。当使用 `build-override` 时，依赖项可能需要构建两次，一次作为普通依赖项，一次使用覆盖后的构建设置。这可能会增加初始构建时间。

使用的值的优先级按以下顺序确定（首次匹配胜出）：

1. `[profile.dev.package.name]` — 一个指定名称的包。
2. `[profile.dev.package."*"]` — 针对任何非工作空间成员。
3. `[profile.dev.build-override]` — 仅针对构建脚本、过程宏及其依赖项。
4. `[profile.dev]` — `Cargo.toml` 中的设置。
5. Cargo 内置的默认值。

覆盖不能指定 `panic`、`lto` 或 `rpath` 设置。

#### 覆盖与泛型

泛型代码被实例化的位置会影响该泛型代码使用的优化设置。当使用 profile 覆盖来更改特定 crate 的优化级别时，这可能导致微妙的交互。如果你尝试提高定义了泛型函数的依赖项的优化级别，这些泛型函数在你的本地 crate 中使用时可能不会被优化。这是因为代码可能在它被实例化的 crate 中生成，因此可能使用该 crate 的优化设置。

例如，[nalgebra] 是一个大量使用泛型参数定义向量和矩阵的库。如果你的本地代码定义了具体的 nalgebra 类型，例如 `Vector4<f64>` 并使用了它们的方法，相应的 nalgebra 代码将在你的 crate 内实例化和构建。因此，如果你尝试使用 profile 覆盖来提高 `nalgebra` 的优化级别，它可能不会产生更快的性能。

更复杂的是，`rustc` 有一些优化，它会尝试在 crate 之间共享单态化的泛型。如果 opt-level 是 2 或 3，那么一个 crate 将不会使用来自其他 crate 的单态化泛型，也不会导出来本地定义的、要与其他 crate 共享的单态化项。当尝试为开发优化依赖项时，可以考虑尝试 opt-level 1，它将应用一些优化，同时仍然允许单态化项被共享。

[nalgebra]: https://crates.io/crates/nalgebra