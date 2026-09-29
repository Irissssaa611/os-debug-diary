# async-os 复现文档

note：记录遇到的问题以及成功复现步骤。

#### 1. 环境

- Ubuntu 24.04.3 LTS
- Rust nightly-2026-05-26
- QEMU (riscv64)

#### 2. 目录结构

四个仓库需要放在同级目录下：

```
~/data/
├── async-os/              # rosy233333/async-os, vdso-test 分支
├── vsched2/               # rosy233333/vsched2
├── vdso_crate_template/   # rosy233333/vdso_crate_template
├── page_table_multiarch/  # arceos-org/page_table_multiarch（本地补丁用）
└── prebuild_vdso/         # 临时项目，用于先编译 vsched2 生成 vdso_output
```

#### 2. 复现步骤

##### 1. Clone 仓库

```bash
cd ~/data
git clone https://github.com/rosy233333/async-os.git
cd async-os && git checkout vdso-test && cd ..
git clone https://github.com/rosy233333/vsched2.git
git clone https://github.com/rosy233333/vdso_crate_template.git
git clone https://github.com/arceos-org/page_table_multiarch.git
cd page_table_multiarch && git checkout v0.6.1 -b fix-for-async-os && cd ..
```

##### 2. 配 Rust 镜像

直连 Rust 服务器太慢，用字节的镜像：

```bash
# rustup 镜像
export RUSTUP_DIST_SERVER=https://rsproxy.cn
export RUSTUP_UPDATE_ROOT=https://rsproxy.cn/rustup

# 写入 bashrc
cat >> ~/.bashrc << 'EOF'
export RUSTUP_DIST_SERVER=https://rsproxy.cn
export RUSTUP_UPDATE_ROOT=https://rsproxy.cn/rustup
EOF
source ~/.bashrc

# cargo 镜像
mkdir -p ~/.cargo
cat > ~/.cargo/config.toml << 'EOF'
[source.crates-io]
replace-with = 'rsproxy'

[source.rsproxy]
registry = "sparse+https://rsproxy.cn/index/"

[net]
git-fetch-with-cli = true
EOF
```

##### 3. 安装 Rust 工具链

```bash
rustup install nightly-2026-05-26
rustup target add riscv64gc-unknown-none-elf x86_64-unknown-none \
    aarch64-unknown-none aarch64-unknown-none-softfloat \
    --toolchain nightly-2026-05-26
rustup component add rust-src llvm-tools rustfmt clippy \
    --toolchain nightly-2026-05-26

# 安装 cargo-binutils（提供 rust-objcopy 等命令）
cargo +nightly-2026-05-26 install cargo-binutils
```

> **如果遇到文件冲突**（`detected conflict`），直接：
>
> ```bash
> rustup toolchain remove nightly-2026-05-26
> rustup install nightly-2026-05-26
> # 然后重新 add target 和 component
> ```

##### 4. 修补源码（核心步骤）

以下四个修补是由于 async-os 原本的缺陷导致编译失败所采取的解决办法。

###### a. 修补 `x86_64` crate 以兼容 nightly-2026-05-26

`x86_64` v0.15.5 实现了 `Step` trait 的 `forward_overflowing` / `backward_overflowing` 方法，这两个方法在 nightly-2026-05-26 中已从 trait 中移除，导致编译失败。

用编辑器打开以下三个文件，找到残留的 `cfg(not(kani))` 块（每个文件各 2 处，共 6 处），整块删除：

```bash
X86_CRATE="/root/.cargo/registry/src/rsproxy.cn-e3de039b2554c837/x86_64-0.15.5"

# 三个需要修补的文件：
# - $X86_CRATE/src/addr.rs
# - $X86_CRATE/src/structures/paging/page.rs
# - $X86_CRATE/src/structures/paging/page_table.rs
```

每个文件中搜索 `Kani`，找到并删除如下形式的代码块：

```rust
    // Kani's bundled toolchain predates these methods being added to `Step`.
    // Exclude them there so the crate still compiles under `cargo kani`.
    // This can be removed once Kani upgrades its bundled toolchain to nightly-2026-07-10 or later.
    #[cfg(not(kani))]
    #[inline]
	……
    }
```

###### b. 修补 `gen_wrapper.rs`：添加 `page_table_entry` patch 和 `raw_trap_handle`

文件：`~/data/vdso_crate_template/build_vdso/src/gen_wrapper.rs`

**修改一**：`page_table_entry` v0.5.7 无条件依赖 `x86_64`，需要 patch 到本地修改版（v0.6.1 已将 x86_64 设为 cfg 条件依赖，但版本号不兼容 0.5.x）。在 `[features]` 块之后添加 `[patch.crates-io]`：

找到 `format!` 宏中的：
```rust
default = [{}]
"#,
```

改为：
```rust
default = [{}]

[patch.crates-io]
page_table_entry = {{ path = "/home/iristseng/data/page_table_multiarch/page_table_entry" }}	//路径根据自己本地情况进行修改
"#,
```

**修改二**：`vsched2/src/arch/riscv.rs` 的汇编代码中有 `beq a0, a1, raw_trap_handle` 指令，但 `raw_trap_handle` 的实现在 完整内核中提供，单独编译成的共享库中没有。需要提供一个桩函数。

在 `lib_rs_content` 函数的 `panic_loop()` 定义之后添加：

```rust
/// 为共享库提供 raw_trap_handle 符号，避免链接时出现 R_RISCV_JAL 错误。
/// 在 vDSO 中不应触发，触发时直接 panic。
#[no_mangle]
pub extern "C" fn raw_trap_handle() -> ! {{
    panic_loop();
}}
```

###### c. 修补 `lib.rs`：添加 `-Bsymbolic` 链接选项

文件：`~/data/vdso_crate_template/build_vdso/src/lib.rs`

vsched2/src/arch/riscv.rs 文件中使用了一个跳转的汇编代码，并且它跳转的对象是一个向外部公开的符号，这类符号通常意味着能够被外部程序提供的同名符号替换掉，链接器会选择运行时确定跳转的地址，但是使用的这个跳转汇编代码不允许运行时确定，所以编译就报错了，解决办法就是添加 `-Bsymbolic` 让链接器优先绑定库内符号。

找到约第 204 行的 `linker_cmd.args([`，在 `"-shared",` 后添加 `"-Bsymbolic",`：

```rust
    linker_cmd.args([
        "-shared",
        "-Bsymbolic",
        "-soname",
```

###### d. 修补本地 `page_table_multiarch` 版本号（和b的修改一相关）

我们从 GitHub 拉的是 v0.6.1 的代码，但 `vsched_hal` 依赖的是 `"0.5.7"`，按 Rust 语义化版本规则，`0.6.x` 与 `0.5.x` 不兼容。cargo 拒绝用不兼容的版本来 `[patch]`。

解决办法是修改版本号为 0.5.8，骗过 cargo 的版本兼容性检查。

```bash
cd ~/data/page_table_multiarch	# 改成自己那对应的地址
sed -i 's/version = "0.6.1"/version = "0.5.8"/' page_table_entry/Cargo.toml
```

##### 5. 创建 prebuild_vdso 项目

async-os 的整个复现过程是先编译 vsched2，再编译 async-os。我在编译过程中，前者编译好后，在编译后者的过程中遇到了下面这个问题：

async-os的编译：cargo build涉及两个步骤。

1. 解析 Cargo.toml 中的所有依赖：vdso/Cargo.toml 中有 libvsched2 = { path = "../vdso_output/libvsched2" }，可是此时还没有这个目录，为什么呢，因为这个目录是在第二个步骤才生成的，cargo 直接报错退出。
2. 执行 vdso/build.rs，它会调用 build_vdso，生成 vdso_output/libvsched2/，可是这个步骤因为第一个问题，导致无法执行。

解决方案：创建一个独立的 Rust 项目，单独调用 `build_vdso` 生成 `vdso_output/`。

```bash
cd ~/data
mkdir -p prebuild_vdso/src

cat > prebuild_vdso/Cargo.toml << 'EOF'
[package]
name = "prebuild_vdso"
version = "0.1.0"
edition = "2021"

[dependencies]
build_vdso = { path = "../vdso_crate_template/build_vdso" }
EOF

cat > prebuild_vdso/src/main.rs << 'EOF'
use build_vdso::*;

fn main() {
    let mut config = BuildConfig::new("../vsched2", "vsched2");
    config.out_dir = String::from("../async-os/vdso_output");
    config.toolchain = String::from("nightly-2026-05-26");
    config.features = vec![String::from("vdso_only")];
    config.log = true;
    config.mode = String::from("release");
    build_vdso(&config);
    println!("Done! vdso_output generated.");
}
EOF
```

##### 6. 预编译 vsched2（生成 vdso_output）

```bash
rm -rf ~/data/async-os/vdso_output/
cd ~/data/prebuild_vdso
cargo +nightly-2026-05-26 run
```

##### 7. 编译 async-os

```bash
cd ~/data/async-os	# 根据自己的路径进行修改
make run ARCH=riscv64 LOG=info BLK=n	# 协程测试不用磁盘，BLK=n 可以禁掉
```
