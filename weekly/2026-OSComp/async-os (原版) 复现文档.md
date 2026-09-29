# async-os 原始版复现文档

## 1. 环境

- Ubuntu 24.04.3 LTS
- Rust nightly-2026-05-26（当时复现的时候误打误撞用了这个版本，可以尝试使用async-os指定的版本，不知道问题会不会少一些）
- QEMU (riscv64)
- riscv64-linux-musl 交叉编译器（用户程序编译 + vDSO 链接需要）
- gdb-multiarch

## 2. 复现步骤

### 2.1 Clone 仓库

```bash
cd ~/data
git clone git@github.com:AsyncModules/async-os.git async-os-original
cd async-os-original
# 保持在 main 分支即可
```

### 2.2 安装 Rust 工具链

```bash
rustup install nightly-2026-05-26
rustup target add riscv64gc-unknown-none-elf riscv64gc-unknown-linux-musl \
    x86_64-unknown-none aarch64-unknown-none aarch64-unknown-none-softfloat \
    --toolchain nightly-2026-05-26
rustup component add rust-src llvm-tools rustfmt clippy \
    --toolchain nightly-2026-05-26

# 安装 cargo-binutils（提供 rust-objcopy）
cargo +nightly-2026-05-26 install cargo-binutils
```

### 2.3 更新 rust-toolchain.toml

将 toolchain 改为 `nightly-2026-05-26`：

```toml
[toolchain]
profile = "minimal"
channel = "nightly-2026-05-26"
components = ["rust-src", "llvm-tools", "rustfmt", "clippy"]
targets = ["x86_64-unknown-none", "riscv64gc-unknown-none-elf", "aarch64-unknown-none", "aarch64-unknown-none-softfloat"]
```

### 2.4 修补源码

以下修补是为了解决 async-os 在两年前的 nightly（nightly-2025-03-15）与当前 nightly-2026-05-26 之间的 Rust 不兼容变更。

---

#### a. `doc_auto_cfg` → `doc_cfg`

`doc_auto_cfg` feature 在 Rust 1.92.0 被合并到 `doc_cfg`。需要修复 cargo git 缓存和项目源码两处。

```bash
cd ~/data/async-os-original

# 修复所有 cargo git 依赖中的 doc_auto_cfg
find /root/.cargo/git/checkouts/ -name "lib.rs" -exec grep -l "doc_auto_cfg" {} \; | while read f; do
    sed -i 's/feature(doc_auto_cfg)/feature(doc_cfg)/' "$f"
done

# 修复项目自身源码
find . -name "lib.rs" -exec grep -l "doc_auto_cfg" {} \; | while read f; do
    sed -i 's/feature(doc_auto_cfg)/feature(doc_cfg)/' "$f"
done
```

#### b. `aos_api` 重复 `doc_cfg`

```bash
sed -i '7d' modules/aos_api/src/lib.rs
```

#### c. `taic-driver` API 变更：`register_sender` 等增加了 `irq` 参数

```bash
# scheduler
sed -i 's/self.inner.register_sender(recv_os, recv_proc)/self.inner.register_sender(recv_os, recv_proc, 0)/' crates/scheduler/src/taic.rs
sed -i 's/self.inner.cancel_sender(recv_os, recv_proc)/self.inner.cancel_sender(recv_os, recv_proc, 0)/' crates/scheduler/src/taic.rs
sed -i 's/self.inner.register_receiver(send_os, send_proc, handler)/self.inner.register_receiver(send_os, send_proc, handler, 0)/' crates/scheduler/src/taic.rs
sed -i 's/self.inner.send_intr(recv_os, recv_proc)/self.inner.send_intr(recv_os, recv_proc, 0)/' crates/scheduler/src/taic.rs

# uruntime
sed -i 's/self.inner.as_ref().unwrap().send_intr(recv_os, recv_proc)/self.inner.as_ref().unwrap().send_intr(recv_os, recv_proc, 0)/' uruntime/src/scheduler.rs
```

#### d. `#[naked]` → `#[unsafe(naked)]`

新版 Rust 要求 `#[naked]` 必须包裹在 `unsafe()` 中。

```bash
find modules/taskctx modules/trampoline -name "*.rs" -exec sed -i 's/#\[naked\]/#[unsafe(naked)]/g' {} \;
```

#### e. `taskctx`：内联汇编宏 `LDR`/`STR`/`POP_GENERAL_REGS` 失效

**根因**：`global_asm!` 定义的汇编宏在新版编译器中不会跨 crate 传播，也不会在同 crate 的不同模块间传播。`taskctx` 的 `naked_asm!` 代码中使用了 `LDR`/`STR`/`POP_GENERAL_REGS` 宏，这些宏定义在 `axhal` 中，`taskctx` 无法访问。

**解决**：将 `modules/taskctx/src/arch/riscv/mod.rs` 中的宏引用全部展开为真正的 RISC-V 指令。

使用 Python 脚本做替换：

```python
# 将 LDR rd, rs, off → ld rd, off*8(rs)
# 将 STR rs2, rs1, off → sd rs2, off*8(rs1)
# 将 POP_GENERAL_REGS → 28 条 ld 指令
```

> 具体脚本见附录 A。

#### f. `trampoline`：`#[repr(align(4))]` + 汇编宏全部失效

三个问题叠加：
1. `#[repr(align(4))]` 不再支持放在函数上
2. `SAVE_REGS`/`RESTORE_REGS` 宏定义在 `global_asm!` 中，`naked_asm!` 无法使用
3. `LDR`/`STR`/`PUSH_GENERAL_REGS`/`POP_GENERAL_REGS` 同上

**解决**：删除 `#[repr(align(4))]`，改为在汇编中加 `.balign 4`；将 `modules/trampoline/src/arch/riscv/mod.rs` 中**所有**宏引用全部内联展开为真实 RISC-V 指令。这意味着原本 ~192 行的文件会展开为较长的指令序列（`SAVE_REGS` 本身就有 40+ 条指令，在 `trap_vector_base` 中被调用两次）。

> 完整替换后的文件见附录 B。

#### g. `AxError::Timeout` → `TimedOut`、`AxError::Again` → `WouldBlock`

```bash
# 批量替换
find modules/ -name "*.rs" -exec sed -i 's/AxError::Timeout/AxError::TimedOut/g' {} \;
sed -i 's/AxError::Again/AxError::WouldBlock/g' modules/syscall/src/syscall_net/imp.rs
```

#### h. `trampoline` 的 `axsignal` 仓库地址变更

```bash
sed -i 's|git = "https://github.com/Starry-OS/axsignal.git"|git = "https://github.com/Starry-Old/axsignal-old.git"|' \
    modules/trampoline/Cargo.toml
```

#### i. vDSO cops：`#[no_mangle]` 与 lang item 冲突

新版 nightly 中 `#[lang = "eh_personality"]` 不允许加 `#[no_mangle]`：

```bash
sed -i '/#\[lang = "eh_personality"\]/,/fn rust_eh_personality/{s/#\[no_mangle\]//;}' vdso/cops/src/lib.rs
```

#### j. 修改 `user_boot` 加载的用户程序

原始版的 `batch_syscall` 依赖 `std::pipe`，在 riscv64 musl target 上不可用。改用 `hello_world`（最简单的用户程序，只需 `fn main() { println!("Hello, world!"); }`）。

修改 `apps/user_boot/src/main.rs`，将 `TESTCASES` 改为：

```rust
const TESTCASES: &[&str] = &[
    "hello_world",
    // "batch_syscall",
    // ...
];
```

### 3.5 编译 vDSO（cops）

vDSO 编译需要 `riscv64-linux-musl-ld` 在 PATH 中。cops 的 `.cargo/config.toml` 已配置 `target-dir = "../target"`，因此产物会输出到 `vdso/target/`，与 `vdso.S` 中的 `.incbin` 路径一致。

```bash
cd ~/data/async-os-original/vdso/cops
PATH="/home/iristseng/data/riscv64-linux-musl-cross/bin:$PATH" \
    cargo +nightly-2026-05-26 build --release

# 验证产物
ls -la ../target/riscv64gc-unknown-linux-musl/release/libcops.so
```

### 3.6 编译内核 + 用户程序

```bash
cd ~/data/async-os-original

# 完整编译（内核 + user_boot + 用户程序）
make build ARCH=riscv64 BLK=y NET=y A=apps/user_boot
```

> 注意：部分用户程序（`batch_syscall`、`pipetest`、`std_thread_test`）会在编译时失败，但不影响 `hello_world` 和内核本身的编译。只需确保磁盘中至少有 `hello_world` 即可。

### 3.7 运行

```bash
cd ~/data/async-os-original
make run ARCH=riscv64 LOG=info BLK=y NET=y \
    QEMU=/usr/local/bin/qemu-system-riscv64 \
    A=apps/user_boot
```

> **QEMU 路径说明**：项目默认使用 `../taic-qemu/build/qemu-system-riscv64`，如果该路径不存在，需要通过 `QEMU=...` 覆盖为系统自带的 QEMU。
>
> **NET=y 说明**：`async_net` 模块在 `user_boot` 的依赖链中被间接引入，即使不需要网络功能也需要启用（否则会在初始化时 panic）。

---

## 附录 A：taskctx 宏展开脚本

```python
import re

path = 'modules/taskctx/src/arch/riscv/mod.rs'
with open(path) as f:
    content = f.read()

POP_GENERAL_REGS_EXPANDED = """\
                ld      ra, 0*8(sp)
                ld      t0, 4*8(sp)
                ld      t1, 5*8(sp)
                ld      t2, 6*8(sp)
                ld      s0, 7*8(sp)
                ld      s1, 8*8(sp)
                ld      a0, 9*8(sp)
                ld      a1, 10*8(sp)
                ld      a2, 11*8(sp)
                ld      a3, 12*8(sp)
                ld      a4, 13*8(sp)
                ld      a5, 14*8(sp)
                ld      a6, 15*8(sp)
                ld      a7, 16*8(sp)
                ld      s2, 17*8(sp)
                ld      s3, 18*8(sp)
                ld      s4, 19*8(sp)
                ld      s5, 20*8(sp)
                ld      s6, 21*8(sp)
                ld      s7, 22*8(sp)
                ld      s8, 23*8(sp)
                ld      s9, 24*8(sp)
                ld      s10, 25*8(sp)
                ld      s11, 26*8(sp)
                ld      t3, 27*8(sp)
                ld      t4, 28*8(sp)
                ld      t5, 29*8(sp)
                ld      t6, 30*8(sp)"""

content = re.sub(r'LDR\s+(\w+),\s*(\w+),\s*(\d+)',
    lambda m: f"ld      {m.group(1)}, {m.group(3)}*8({m.group(2)})", content)
content = re.sub(r'STR\s+(\w+),\s*(\w+),\s*(\d+)',
    lambda m: f"sd      {m.group(1)}, {m.group(3)}*8({m.group(2)})", content)
content = content.replace('POP_GENERAL_REGS', POP_GENERAL_REGS_EXPANDED)

with open(path, 'w') as f:
    f.write(content)
```

## 附录 B：trampoline 完整替换文件

`modules/trampoline/src/arch/riscv/mod.rs` 的最终版本（宏全部内联展开、不再依赖任何 `global_asm!` 宏）

```rust
use crate::trampoline;
use riscv::register::stvec;
use taskctx::TrapFrame;

/// Writes Supervisor Trap Vector Base Address Register (`stvec`).
#[inline]
pub fn set_trap_vector_base(stvec: usize) {
    unsafe { stvec::write(stvec, stvec::TrapMode::Direct) }
}

/// To initialize the trap vector base address.
pub fn init_interrupt() {
    set_trap_vector_base(trap_vector_base as usize);
}

#[unsafe(naked)]
#[link_section = ".text"]
pub unsafe extern "C" fn trap_vector_base() {
    core::arch::naked_asm!(
        "
        .balign 4
        csrrw   sp, sscratch, sp

        bnez    sp, 1f

        // --- 内核态 Trap ---
        csrr    sp, sscratch
        addi    sp, sp, -{trapframe_size}

        // SAVE_REGS: 保存所有通用寄存器
        sd      ra, 0*8(sp)
        sd      t0, 4*8(sp)
        sd      t1, 5*8(sp)
        sd      t2, 6*8(sp)
        sd      s0, 7*8(sp)
        sd      s1, 8*8(sp)
        sd      a0, 9*8(sp)
        sd      a1, 10*8(sp)
        sd      a2, 11*8(sp)
        sd      a3, 12*8(sp)
        sd      a4, 13*8(sp)
        sd      a5, 14*8(sp)
        sd      a6, 15*8(sp)
        sd      a7, 16*8(sp)
        sd      s2, 17*8(sp)
        sd      s3, 18*8(sp)
        sd      s4, 19*8(sp)
        sd      s5, 20*8(sp)
        sd      s6, 21*8(sp)
        sd      s7, 22*8(sp)
        sd      s8, 23*8(sp)
        sd      s9, 24*8(sp)
        sd      s10, 25*8(sp)
        sd      s11, 26*8(sp)
        sd      t3, 27*8(sp)
        sd      t4, 28*8(sp)
        sd      t5, 29*8(sp)
        sd      t6, 30*8(sp)
        // sepc, sstatus, sp
        csrr    t0, sepc
        csrr    t1, sstatus
        csrrw   t2, sscratch, zero
        sd      t0, 31*8(sp)
        sd      t1, 32*8(sp)
        sd      t2, 1*8(sp)
        // fs0, fs1
        .short  0xa622
        .short  0xaa26
        // scause, stval
        csrr    t0, scause
        csrr    t1, stval
        sd      t0, 35*8(sp)
        sd      t1, 36*8(sp)
        // trap_status = 1
        li      t0, 1
        sd      t0, 37*8(sp)

        mv      a0, sp
        li      a1, 1
        li      a2, 0
        call    {trampoline}

        // RESTORE_REGS
        ld      t0, 31*8(sp)
        ld      t1, 32*8(sp)
        csrw    sepc, t0
        csrw    sstatus, t1
        .short  0x2432
        .short  0x24d2
        // POP_GENERAL_REGS
        ld      ra, 0*8(sp)
        ld      t0, 4*8(sp)
        ld      t1, 5*8(sp)
        ld      t2, 6*8(sp)
        ld      s0, 7*8(sp)
        ld      s1, 8*8(sp)
        ld      a0, 9*8(sp)
        ld      a1, 10*8(sp)
        ld      a2, 11*8(sp)
        ld      a3, 12*8(sp)
        ld      a4, 13*8(sp)
        ld      a5, 14*8(sp)
        ld      a6, 15*8(sp)
        ld      a7, 16*8(sp)
        ld      s2, 17*8(sp)
        ld      s3, 18*8(sp)
        ld      s4, 19*8(sp)
        ld      s5, 20*8(sp)
        ld      s6, 21*8(sp)
        ld      s7, 22*8(sp)
        ld      s8, 23*8(sp)
        ld      s9, 24*8(sp)
        ld      s10, 25*8(sp)
        ld      s11, 26*8(sp)
        ld      t3, 27*8(sp)
        ld      t4, 28*8(sp)
        ld      t5, 29*8(sp)
        ld      t6, 30*8(sp)
        ld      sp, 1*8(sp)

        sret

        // --- 用户态 Trap ---
        1:

        // SAVE_REGS
        sd      ra, 0*8(sp)
        sd      t0, 4*8(sp)
        sd      t1, 5*8(sp)
        sd      t2, 6*8(sp)
        sd      s0, 7*8(sp)
        sd      s1, 8*8(sp)
        sd      a0, 9*8(sp)
        sd      a1, 10*8(sp)
        sd      a2, 11*8(sp)
        sd      a3, 12*8(sp)
        sd      a4, 13*8(sp)
        sd      a5, 14*8(sp)
        sd      a6, 15*8(sp)
        sd      a7, 16*8(sp)
        sd      s2, 17*8(sp)
        sd      s3, 18*8(sp)
        sd      s4, 19*8(sp)
        sd      s5, 20*8(sp)
        sd      s6, 21*8(sp)
        sd      s7, 22*8(sp)
        sd      s8, 23*8(sp)
        sd      s9, 24*8(sp)
        sd      s10, 25*8(sp)
        sd      s11, 26*8(sp)
        sd      t3, 27*8(sp)
        sd      t4, 28*8(sp)
        sd      t5, 29*8(sp)
        sd      t6, 30*8(sp)
        // sepc, sstatus, sp
        csrr    t0, sepc
        csrr    t1, sstatus
        csrrw   t2, sscratch, zero
        sd      t0, 31*8(sp)
        sd      t1, 32*8(sp)
        sd      t2, 1*8(sp)
        // fs0, fs1
        .short  0xa622
        .short  0xaa26
        // scause, stval
        csrr    t0, scause
        csrr    t1, stval
        sd      t0, 35*8(sp)
        sd      t1, 36*8(sp)
        // trap_status = 1
        li      t0, 1
        sd      t0, 37*8(sp)

        // gp/tp 保存和加载
        ld      t1, 2*8(sp)
        ld      t0, 3*8(sp)
        sd      gp, 2*8(sp)
        sd      tp, 3*8(sp)
        mv      gp, t1
        mv      tp, t0

        li      a0, 1
        sd      a0, 37*8(sp)
        mv      a0, sp
        li      a1, 1
        li      a2, 1
        ld      sp, 38*8(sp)
        call    {trampoline}
        ",
        trapframe_size = const core::mem::size_of::<TrapFrame>(),
        trampoline = sym trampoline,
    )
}

```