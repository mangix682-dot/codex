# Codex 本地编译与运行 (Windows 11)

## 前置条件

| 工具 | 版本要求 | 安装方式 |
|------|---------|---------|
| **Rust** | 工具链 1.93.0 (由 `rust-toolchain.toml` 指定) | [rustup-init.exe](https://rustup.rs/) |
| **Node.js** | >= 22 | [nodejs.org](https://nodejs.org/) |
| **pnpm** | >= 10.33.0 | `npm i -g pnpm@10.33.0` |
| **Git** | >= 2.23 | [git-scm.com](https://git-scm.com/) |
| **just** (可选) | 最新 | `cargo install --locked just` |
| **cargo-nextest** (可选) | 最新 | `cargo install --locked cargo-nextest` |
| **Visual Studio Build Tools** | 2019+ | 安装 "使用 C++ 的桌面开发" 工作负载 |

> RAM 建议 8 GB 以上。

**必须开启 Windows 开发人员模式：** v8 crate 编译时需要创建符号链接，否则会报错 `symlink_dir failed (code 1314)`。
开启路径：**设置 → 系统 → 开发者选项 → 开发人员模式 → 开**。开启后重启终端。

## 编译 Rust 核心 (codex-rs)

```powershell
cd codex\codex-rs

# 首次拉取依赖
cargo fetch

# 编译 (debug)
cargo build

# 编译 (release)
cargo build --release
```

生成的二进制文件位于 `codex-rs/target/debug/codex.exe` 或 `codex-rs/target/release/codex.exe`。

## 直接运行（跳过预编译，方便调试）

无需先 `cargo build`，`cargo run` 会自动增量编译并立即运行。

**关键：** `cargo run` 必须在 `codex-rs/` 目录执行，但 codex 支持 `-C`（`--cd`）参数指定实际工作目录，这样 agent 会在你的目标项目中操作：

```powershell
cd codex\codex-rs

# 首次运行前需要拉取依赖（只需执行一次）
cargo fetch

# 指定工作目录，直接进入交互式 TUI（推荐）
cargo run --bin codex -- -C D:\codextester

cargo run --release --bin codex -- -C D:\codextester

# 也可以附带初始 prompt，进入 TUI 后会自动执行第一条消息
cargo run --bin codex -- -C D:\my-project "explain this codebase to me"

# 非交互模式
cargo run --bin codex -- exec -C D:\my-project "fix the login bug"

# 如果不加 -C，工作目录默认为当前目录 (codex-rs)
cargo run --bin codex
```

如果需要在调试器中运行（VS Code + CodeLLDB / MSVC 调试器）：

```powershell
# 编译后获取二进制路径
cargo build --bin codex --message-format=short
# 然后在 VS Code 中对 target\debug\codex.exe 启动调试 (F5)
# launch.json 的 args 中加入: ["-C", "D:\\my-project", "your prompt"]
```

## 运行已编译的二进制

```powershell
.\target\debug\codex.exe "your prompt here"
.\target\release\codex.exe "your prompt here"
```

## 常用开发命令

```powershell
# 格式化
cargo fmt

# Lint
cargo clippy --tests

# 运行单个 crate 的测试
cargo test -p codex-tui

# 运行全量测试 (需安装 cargo-nextest)
cargo nextest run --no-fail-fast
```

## 日志

TUI 的日志默认写入 `~/.codex/log/codex-tui.log`，可通过环境变量控制级别：

```powershell
$env:RUST_LOG="codex_core=debug,codex_tui=debug"
cargo run --bin codex -- "your prompt"
```

## codex-cli (Node.js 包装层)

`codex-cli/` 是一个 Node.js 的薄包装，一般开发时只需关注 `codex-rs`。如果需要测试 npm 包：

```powershell
cd codex\codex-cli
pnpm install
node bin/codex.js "your prompt"
```
