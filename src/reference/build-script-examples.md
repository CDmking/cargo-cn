## 构建脚本示例

以下章节展示了编写构建脚本的一些示例。

一些常见的构建脚本功能可以通过 [crates.io] 上的 crate 找到。查看 [`build-dependencies` 关键字](https://crates.io/keywords/build-dependencies) 以了解有哪些可用的 crate。以下是一些流行 crate 的示例[^†]：

* [`bindgen`](https://crates.io/crates/bindgen) — 自动为 C 库生成 Rust FFI 绑定。
* [`cc`](https://crates.io/crates/cc) — 编译 C/C++/汇编语言代码。
* [`pkg-config`](https://crates.io/crates/pkg-config) — 使用 `pkg-config` 工具检测系统库。
* [`cmake`](https://crates.io/crates/cmake) — 运行 `cmake` 构建工具来构建本地库。
* [`autocfg`](https://crates.io/crates/autocfg)、[`rustc_version`](https://crates.io/crates/rustc_version)、[`version_check`](https://crates.io/crates/version_check) — 这些 crate 提供了基于当前 `rustc`（例如编译器的版本）实现条件编译的方法。

[^†]: 此列表并非背书。请评估你的依赖项，以确定哪个适合你的项目。

### 代码生成

由于各种原因，一些 Cargo 包需要在编译前生成代码。这里我们将通过一个简单的示例来演示，该示例在构建脚本中生成一个库调用。

首先，让我们看一下这个包的目录结构：

```text
.
├── Cargo.toml
├── build.rs
└── src
    └── main.rs

1 directory, 3 files
```

在这里，我们可以看到有一个 `build.rs` 构建脚本和一个位于 `main.rs` 中的二进制文件。这个包有一个基本的清单文件：

```toml
# Cargo.toml

[package]
name = "hello-from-generated-code"
version = "0.1.0"
```

让我们看看构建脚本里面有什么：

```rust,no_run
// build.rs

use std::env;
use std::fs;
use std::path::Path;

fn main() {
    let out_dir = env::var_os("OUT_DIR").unwrap();
    let dest_path = Path::new(&out_dir).join("hello.rs");
    fs::write(
        &dest_path,
        "pub fn message() -> &'static str {
            \"Hello, World!\"
        }
        "
    ).unwrap();
    println!("cargo:rerun-if-changed=build.rs");
}
```

这里有几点需要注意：

* 脚本使用 `OUT_DIR` 环境变量来发现输出文件应该放在哪里。它可以使用进程的当前工作目录来找到输入文件的位置，但在这个例子中我们没有输入文件。
* 一般来说，构建脚本不应该修改 `OUT_DIR` 之外的任何文件。乍一看可能没什么问题，但当你将这样的 crate 作为依赖项使用时，会导致问题，因为存在一个*隐含*的不变性：`.cargo/registry` 中的源文件应该是不可变的。打包时，`cargo` 不允许这样的脚本。
* 这个脚本相对简单，因为它只是写出了一个小的生成文件。可以想象，可能会发生其他更复杂的操作，例如根据 C 头文件或其他语言定义生成 Rust 模块。
* [`rerun-if-changed` 指令](build-scripts.md#rerun-if-changed)告诉 Cargo，只有在构建脚本本身发生变化时才需要重新运行构建脚本。没有这一行，Cargo 会在包中的任何文件更改时自动运行构建脚本。如果你的代码生成使用了一些输入文件，你应该在这里打印出这些文件的列表。

接下来，让我们看看库本身：

```rust,ignore
// src/main.rs

include!(concat!(env!("OUT_DIR"), "/hello.rs"));

fn main() {
    println!("{}", message());
}
```

这就是真正的魔法发生的地方。库使用了 rustc 定义的 [`include!` 宏][include-macro]，结合 [`concat!`][concat-macro] 和 [`env!`][env-macro] 宏，将生成的文件（`hello.rs`）包含到 crate 的编译中。

使用这里展示的结构，crate 可以从构建脚本本身包含任意数量的生成文件。

[include-macro]: ../../std/macro.include.html
[concat-macro]: ../../std/macro.concat.html
[env-macro]: ../../std/macro.env.html

### 构建本地库

有时，作为包的一部分，需要构建一些本地的 C 或 C++ 代码。这是利用构建脚本在 Rust crate 本身之前构建本地库的另一个绝佳用例。作为一个例子，我们将创建一个调用 C 来打印 "Hello, World!" 的 Rust 库。

像上面一样，让我们先看一下包的布局：

```text
.
├── Cargo.toml
├── build.rs
└── src
    ├── hello.c
    └── main.rs

1 directory, 4 files
```

与之前非常相似！接下来是清单文件：

```toml
# Cargo.toml

[package]
name = "hello-world-from-c"
version = "0.1.0"
edition = "2018"
```

目前我们不打算使用任何构建依赖项，所以现在让我们看看构建脚本：

```rust,no_run
// build.rs

use std::process::Command;
use std::env;
use std::path::Path;

fn main() {
    let out_dir = env::var("OUT_DIR").unwrap();

    // 请注意，这种方法有许多缺点，下面的注释详细说明了如何提高这些命令的可移植性。
    Command::new("gcc").args(&["src/hello.c", "-c", "-fPIC", "-o"])
                       .arg(&format!("{}/hello.o", out_dir))
                       .status().unwrap();
    Command::new("ar").args(&["crus", "libhello.a", "hello.o"])
                      .current_dir(&Path::new(&out_dir))
                      .status().unwrap();

    println!("cargo:rustc-link-search=native={}", out_dir);
    println!("cargo:rustc-link-lib=static=hello");
    println!("cargo:rerun-if-changed=src/hello.c");
}
```

这个构建脚本首先通过调用 `gcc` 将 C 文件编译成目标文件，然后通过调用 `ar` 将这个目标文件转换为静态库。最后一步是反馈给 Cargo 本身，说明我们的输出在 `out_dir` 目录中，并且编译器应该通过 `-l static=hello` 标志静态地链接到 `libhello.a`。

请注意，这种硬编码方法存在许多缺点：

* `gcc` 命令本身不能跨平台移植。例如，Windows 平台可能没有 `gcc`，甚至并非所有 Unix 平台都有 `gcc`。`ar` 命令也有类似的情况。
* 这些命令没有考虑交叉编译。如果我们为 Android 这样的平台进行交叉编译，`gcc` 不太可能产生 ARM 可执行文件。

不过别担心，这正是 `build-dependencies` 条目可以发挥作用的地方！Cargo 生态系统中有许多包可以使这类任务变得更加容易、可移植和标准化。让我们尝试一下来自 [crates.io] 的 [`cc` crate](https://crates.io/crates/cc)。首先，将它添加到 `Cargo.toml` 的 `build-dependencies` 中：

```toml
[build-dependencies]
cc = "1.0"
```

并重写构建脚本来使用这个 crate：

```rust,ignore
// build.rs

fn main() {
    cc::Build::new()
        .file("src/hello.c")
        .compile("hello");
    println!("cargo:rerun-if-changed=src/hello.c");
}
```

[`cc` crate] 抽象了 C 代码的一系列构建脚本需求：

* 它调用适当的编译器（Windows 上用 MSVC，MinGW 上用 `gcc`，Unix 平台用 `cc` 等）。
* 它考虑了 `TARGET` 变量，向使用的编译器传递适当的标志。
* 其他环境变量，如 `OPT_LEVEL`、`DEBUG` 等，都会自动处理。
* 标准输出和 `OUT_DIR` 位置也由 `cc` 库处理。

在这里，我们开始看到将尽可能多的功能转移到通用的构建依赖项中，而不是在所有构建脚本中重复逻辑的一些主要好处！

不过回到案例研究，让我们快速查看一下 `src` 目录的内容：

```c
// src/hello.c

#include <stdio.h>

void hello() {
    printf("Hello, World!\n");
}
```

```rust,ignore
// src/main.rs

// 注意没有 `#[link]` 属性。我们将选择链接什么的责任委托给构建脚本，而不是在源文件中硬编码。
extern { fn hello(); }

fn main() {
    unsafe { hello(); }
}
```

就是这样！这个例子展示了如何使用构建脚本本身从 Cargo 包构建 C 代码。这也说明了为什么在许多情况下使用构建依赖项至关重要，甚至更加简洁！

我们还简要地看到了构建脚本如何纯粹为了构建过程而使用一个 crate 作为依赖项，而不是在运行时为 crate 本身使用。

[`cc` crate]: https://crates.io/crates/cc

### 链接到系统库

这个示例演示了如何链接系统库，以及如何利用构建脚本来支持这种用例。

Rust crate 经常希望链接到系统上提供的本地库，以便绑定其功能，或者仅将其作为实现细节的一部分。要在与平台无关的方式下执行此操作，这是一个非常微妙的问题。如果可能的话，最好将尽可能多的此类工作外包出去，以便让使用者尽可能容易。

在这个例子中，我们将创建一个绑定到系统的 zlib 库的绑定。这是一个在大多数类 Unix 系统上常见的库，提供数据压缩功能。这已经被封装在 [`libz-sys` crate] 中，但在这个例子中，我们将做一个极其简化的版本。查看完整示例的[源代码][libz-source]。

为了便于找到库的位置，我们将使用 [`pkg-config` crate]。这个 crate 使用系统的 `pkg-config` 工具来发现库的信息。它会自动告诉 Cargo 链接库需要什么。这可能只在安装了 `pkg-config` 的类 Unix 系统上有效。让我们从设置清单开始：

```toml
# Cargo.toml

[package]
name = "libz-sys"
version = "0.1.0"
edition = "2018"
links = "z"

[build-dependencies]
pkg-config = "0.3.16"
```

请注意，我们在 `package` 表中包含了 `links` 键。这告诉 Cargo 我们正在链接到 `libz` 库。有关将利用这一点的示例，请参阅 ["使用另一个 sys crate"](#使用另一个-sys-crate)。

构建脚本相当简单：

```rust,ignore
// build.rs

fn main() {
    pkg_config::Config::new().probe("zlib").unwrap();
    println!("cargo:rerun-if-changed=build.rs");
}
```

让我们用一个基本的 FFI 绑定来完善这个例子：

```rust,ignore
// src/lib.rs

use std::os::raw::{c_uint, c_ulong};

extern "C" {
    pub fn crc32(crc: c_ulong, buf: *const u8, len: c_uint) -> c_ulong;
}

#[test]
fn test_crc32() {
    let s = "hello";
    unsafe {
        assert_eq!(crc32(0, s.as_ptr(), s.len() as c_uint), 0x3610a686);
    }
}
```

运行 `cargo build -vv` 以查看构建脚本的输出。在已安装 `libz` 的系统上，输出可能类似于：

```text
[libz-sys 0.1.0] cargo:rustc-link-search=native=/usr/lib
[libz-sys 0.1.0] cargo:rustc-link-lib=z
[libz-sys 0.1.0] cargo:rerun-if-changed=build.rs
```

很好！`pkg-config` 完成了所有查找库并告诉 Cargo 其位置的工作。

包中包含库的源代码，如果系统上找不到库，或者设置了某个功能或环境变量，则静态构建它，这并不少见。例如，真正的 [`libz-sys` crate] 会检查环境变量 `LIBZ_SYS_STATIC` 或 `static` 功能，以便从源代码构建它，而不是使用系统库。查看更完整示例的[源代码][libz-source]。

[`libz-sys` crate]: https://crates.io/crates/libz-sys
[`pkg-config` crate]: https://crates.io/crates/pkg-config
[libz-source]: https://github.com/rust-lang/libz-sys

### 使用另一个 `sys` crate

当使用 `links` 键时，crate 可以设置元数据，这些元数据可以被依赖它的其他 crate 读取。这提供了一种在 crate 之间传递信息的机制。在这个示例中，我们将创建一个使用来自真正的 [`libz-sys` crate] 的 zlib 的 C 库。

如果你有一个依赖 zlib 的 C 库，你可以利用 [`libz-sys` crate] 来自动找到或构建它。这对于跨平台支持非常有用，例如在通常不安装 zlib 的 Windows 上。`libz-sys` [设置了 `include` 元数据](https://github.com/rust-lang/libz-sys/blob/3c594e677c79584500da673f918c4d2101ac97a1/build.rs#L156)，以告诉其他包在哪里可以找到 zlib 的头文件。我们的构建脚本可以使用 `DEP_Z_INCLUDE` 环境变量读取该元数据。下面是一个示例：

```toml
# Cargo.toml

[package]
name = "zuser"
version = "0.1.0"
edition = "2018"

[dependencies]
libz-sys = "1.0.25"

[build-dependencies]
cc = "1.0.46"
```

这里我们包含了 `libz-sys`，它将确保最终库中只有一个 `libz` 被使用，并允许我们从构建脚本中访问它：

```rust,ignore
// build.rs

fn main() {
    let mut cfg = cc::Build::new();
    cfg.file("src/zuser.c");
    if let Some(include) = std::env::var_os("DEP_Z_INCLUDE") {
        cfg.include(include);
    }
    cfg.compile("zuser");
    println!("cargo:rerun-if-changed=src/zuser.c");
}
```

通过 `libz-sys` 完成所有繁重的工作，C 源代码现在可以包含 zlib 头文件，并且应该能找到该头文件，即使在未安装它的系统上。

```c
// src/zuser.c

#include "zlib.h"

// … 其余使用 zlib 的代码。
```

### 条件编译

构建脚本可以发出 [`rustc-cfg` 指令][`rustc-cfg` instructions]，这些指令可以启用编译时检查的条件。在这个示例中，我们将看看 [`openssl` crate] 如何使用它来支持多个版本的 OpenSSL 库。

[`openssl-sys` crate] 实现了构建和链接 OpenSSL 库。它支持多种不同的实现（如 LibreSSL）和多个版本。它使用了 `links` 键，以便可以将信息传递给其他构建脚本。它传递的内容之一是 `version_number` 键，即检测到的 OpenSSL 版本。构建脚本中的代码看起来[像这样](https://github.com/sfackler/rust-openssl/blob/dc72a8e2c429e46c275e528b61a733a66e7877fc/openssl-sys/build/main.rs#L216)：

```rust,ignore
println!("cargo:version_number={:x}", openssl_version);
```

这个指令导致 `DEP_OPENSSL_VERSION_NUMBER` 环境变量在任何直接依赖于 `openssl-sys` 的 crate 中被设置。

提供高级接口的 `openssl` crate 将 `openssl-sys` 指定为依赖项。`openssl` 构建脚本可以使用 `DEP_OPENSSL_VERSION_NUMBER` 环境变量读取 `openssl-sys` 构建脚本生成的版本信息。它使用这个来生成一些 [`cfg` 值](https://github.com/sfackler/rust-openssl/blob/dc72a8e2c429e46c275e528b61a733a66e7877fc/openssl/build.rs#L18-L36)：

```rust,ignore
// (构建脚本的一部分)

if let Ok(version) = env::var("DEP_OPENSSL_VERSION_NUMBER") {
    let version = u64::from_str_radix(&version, 16).unwrap();

    if version >= 0x1_00_01_00_0 {
        println!("cargo:rustc-cfg=ossl101");
    }
    if version >= 0x1_00_02_00_0 {
        println!("cargo:rustc-cfg=ossl102");
    }
    if version >= 0x1_01_00_00_0 {
        println!("cargo:rustc-cfg=ossl110");
    }
    if version >= 0x1_01_00_07_0 {
        println!("cargo:rustc-cfg=ossl110g");
    }
    if version >= 0x1_01_01_00_0 {
        println!("cargo:rustc-cfg=ossl111");
    }
}
```

然后可以使用 [`cfg` 属性][`cfg` attribute] 或 [`cfg` 宏][`cfg` macro] 来有条件地包含代码。例如，SHA3 支持是在 OpenSSL 1.1.1 中添加的，因此对于旧版本，它被[有条件地排除](https://github.com/sfackler/rust-openssl/blob/dc72a8e2c429e46c275e528b61a733a66e7877fc/openssl/src/hash.rs#L67-L85)：

```rust,ignore
// (openssl crate 的一部分)

#[cfg(ossl111)]
pub fn sha3_224() -> MessageDigest {
    unsafe { MessageDigest(ffi::EVP_sha3_224()) }
}
```

当然，在使用此功能时应小心，因为它会使生成的二进制文件更加依赖于构建环境。在这个例子中，如果二进制文件被分发到另一个系统，它可能没有完全相同版本的共享库，这可能会导致问题。

[`cfg` attribute]: ../../reference/conditional-compilation.md#the-cfg-attribute
[`cfg` macro]: ../../std/macro.cfg.html
[`rustc-cfg` instructions]: build-scripts.md#rustc-cfg
[`openssl` crate]: https://crates.io/crates/openssl
[`openssl-sys` crate]: https://crates.io/crates/openssl-sys

[crates.io]: https://crates.io/