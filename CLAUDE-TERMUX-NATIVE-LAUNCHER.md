# Claude Code 原生启动器修正（Termux/aarch64）

> **本文档是对 [CLAUDE-TERMUX-SETUP.md](CLAUDE-TERMUX-SETUP.md) §3.3 启动脚本的关键修正。** 适用于 Claude Code **v2.1.180+**（含最新 v2.1.187+），在 Android 内核 ≥ 5.11 的设备上修复 `glibc-runner` wrapper 启动后 `--dangerously-skip-permissions` 等命令触发 Bun segfault 的问题。

---

## 一、问题现象

`glibc-runner /usr/.../claude.exe` 这种 wrapper 在新版 Claude Code（Bun v1.4.0 内嵌）+ 新内核（≥ 6.6，Android 15/16）组合下，运行携带 IO 多路复用的子命令时会崩溃：

```
panic: Segmentation fault at address 0x0
oh no: Bun has crashed.
...
https://bun.report/1.4.0/...
```

崩溃栈对应 Bun issue [oven-sh/bun#32489](https://github.com/oven-sh/bun/issues/32489)：Bun 在内核版本字符串看起来 ≥ 5.11 时启用 `epoll_pwait2` 系统调用路径，该路径在某些 Android 内核实现下立刻段错误。

典型可复现命令（v2.1.187 + Linux 6.6.89）：

```bash
claude --version                          # ✅ 正常
claude --dangerously-skip-permissions ... # ❌ Bun panic / SIGSEGV (139/133)
```

---

## 二、根因分析

社区方案 [`claude-code-termux`](https://github.com/gtbuchanan/claude-code-termux)（apt 包）已提供 `uname-spoof.so` 共享库：劫持 `uname()` 把 `release` 改成 `5.10.x`，让 Bun 跳过 `epoll_pwait2` 路径。其 launcher 通过 `LD_PRELOAD=…/uname-spoof.so` 注入。

**陷阱**：直接复用 `glibc-runner` 来启动 patched 二进制时，spoof 失效。原因在 `glibc-runner.sh`（v2.0）启动逻辑的第一行：

```bash
_glibc-runner_set_up_shell() {
    unset LD_PRELOAD                       # ← 这里清空了用户预设的 LD_PRELOAD
    export PATH_LIBTERMUX_EXEC_GLIBC="..."
    ...
}
```

即便外层 `export LD_PRELOAD=uname-spoof.so` 也会被 glibc-runner 在执行 `ld.so` 前清掉，Bun 仍读到真实的 6.6.89，仍走 `epoll_pwait2`，仍 segfault。

---

## 三、修正方案：绕过 glibc-runner，直调 ld.so

**核心原则不变**（与现有文档一致）：
- ❌ 不要 `patchelf --set-interpreter` 修改二进制（会破坏 Bun 的 TLS 布局）
- ✅ 保持二进制 ELF interpreter 原值 `/lib/ld-linux-aarch64.so.1`
- ✅ 通过 glibc loader 间接加载

**新 wrapper 直接调用 `ld-linux-aarch64.so.1 --library-path`** 而不是经过 `glibc-runner`，自己控制 `LD_PRELOAD`，确保 `uname-spoof.so` 被加载。

### 3.1 安装前置（一次）

```bash
# 安装含 uname-spoof.so 的社区包（只用其库文件，不用其 launcher）
curl -fsSL https://raw.githubusercontent.com/gtbuchanan/claude-code-termux/main/install.sh | bash
```

> 该 install.sh 会启用 glibc apt 源、安装 `glibc-runner`、`patchelf-glibc` 等依赖，并在 `/data/data/com.termux/files/usr/lib/claude-code-termux/uname-spoof.so` 放下 spoof 库。`postinst` 阶段会尝试从 `downloads.claude.ai` 下载 Claude 二进制；该域名在中国大陆常被超时，**postinstall 失败不影响本方案**（我们用 npm registry 拿二进制）。

### 3.2 部署原生二进制（不 patchelf）

```bash
# 从 npm registry 下载平台包（绕过 downloads.claude.ai 的可达性问题）
TMPD=$(mktemp -d -p /data/data/com.termux/files/usr/tmp)
cd "$TMPD"
curl -sL -o c.tgz \
  "https://registry.npmjs.org/@anthropic-ai/claude-code-linux-arm64/-/claude-code-linux-arm64-2.1.187.tgz"
tar xzf c.tgz

# 直接复制到 claude-code-termux 包的 opt 目录（不做 patchelf）
OPT=/data/data/com.termux/files/usr/opt/claude-code-termux
mkdir -p "$OPT"
cp package/claude "$OPT/claude-2.1.187"
chmod +x "$OPT/claude-2.1.187"
ln -sfn claude-2.1.187 "$OPT/current"

# 应用 execPath patch（让子进程重入走 wrapper，不直跑 bare 二进制）
python3 /data/data/com.termux/files/usr/libexec/claude-code-termux/patch-execpath.py \
  "$OPT/claude-2.1.187"

cd - >/dev/null && rm -rf "$TMPD"
```

> `patch-execpath.py` 只改字节级的 `process.execPath` 赋值字符串（保持长度），**不修改 ELF header**。验证：
> ```bash
> glibc-runner /usr/glibc/bin/patchelf --print-interpreter \
>   /data/data/com.termux/files/usr/opt/claude-code-termux/current
> # 必须输出: /lib/ld-linux-aarch64.so.1   （原值，未被 patchelf 改）
> ```

### 3.3 新 wrapper 脚本

```bash
cat > ~/.npm-global/bin/claude << 'WRAPPER'
#!/data/data/com.termux/files/usr/bin/bash
# Claude Code launcher for Termux/aarch64
# 直调 glibc ld.so，自己控制 LD_PRELOAD，避免 glibc-runner unset LD_PRELOAD

GLIBC=/data/data/com.termux/files/usr/glibc
export LD_PRELOAD=/data/data/com.termux/files/usr/lib/claude-code-termux/uname-spoof.so
export TMPDIR=/data/data/com.termux/files/usr/tmp
export CLAUDE_CODE_TMPDIR=/data/data/com.termux/files/usr/tmp
export DISABLE_AUTOUPDATER=1
export PATH="$GLIBC/bin:$PATH"

exec "$GLIBC/lib/ld-linux-aarch64.so.1" --library-path "$GLIBC/lib" \
  /data/data/com.termux/files/usr/opt/claude-code-termux/current "$@"
WRAPPER

chmod +x ~/.npm-global/bin/claude
```

### 3.4 验证

```bash
claude --version
# 期望: 2.1.187 (Claude Code)

claude --dangerously-skip-permissions --print "ping"
# 期望: 模型回复，无 Bun panic、无 SIGSEGV
```

如仍崩，按 §六故障排除逐项核对。

---

## 四、性能分析

新 wrapper 是**原生执行 + 零翻译**，与裸 Linux glibc 等价。

| 项 | 详情 |
|---|---|
| 运行方式 | 原生 aarch64 ELF（Anthropic 官方 linux-arm64 二进制） |
| 二进制路径 | `/usr/opt/claude-code-termux/current` → `claude-2.1.187`（≈ 222 MB） |
| 启动器 | `~/.npm-global/bin/claude` → glibc `ld-linux-aarch64.so.1` |
| 加载链 | bash → ld.so `--library-path $GLIBC/lib` → claude (Bun 1.4.0) |
| 是否 patchelf | ❌ 否（二进制 ELF interpreter 保持原值，避免 TLS 损坏） |
| 是否 glibc-runner | ❌ 否（绕过，避免它 `unset LD_PRELOAD`） |
| 是否 proot/qemu | ❌ 否（无指令模拟，无沙箱） |
| LD_PRELOAD 注入 | `uname-spoof.so`（1.8 KB，仅劫持 `uname()` 让 Bun 跳过 epoll_pwait2） |
| CPU 架构 | aarch64 原生（CPU 直接执行 ARM64 指令） |
| 指令翻译 / 虚拟化 | 无 |
| 性能损耗 | ≈ 0%（与 Linux 上裸跑等价；启动多一次 ld.so 显式解析 + uname 劫持，运行时为零开销） |
| 启动冷延迟 | ~200 ms（Bun 自身开销，与 Linux 一致） |
| 内存 RSS | ≈ 37 MB |
| 比 proot+qemu 方案 | 快约 10–50 倍（无 syscall 翻译，无指令解码） |

**与旧 glibc-runner wrapper 对比**：

| 维度 | 旧 `glibc-runner` wrapper | 新 ld.so 直调 wrapper |
|---|---|---|
| `claude --version` | ✅ 正常 | ✅ 正常 |
| `--dangerously-skip-permissions` | ❌ Bun segfault（v2.1.180+） | ✅ 正常 |
| LD_PRELOAD 受控性 | ❌ 被 `unset` | ✅ 自己管 |
| 启动开销 | 多一层 bash 源码层（glibc-runner.sh ~250 行） | 直接 exec ld.so |
| 兼容性 | 与新 Bun + 新内核组合冲突 | 当前已验证最新版可用 |

---

## 五、关键约束（必须遵守）

1. **必须保留 `/lib/ld-linux-aarch64.so.1` 作为二进制的 ELF interpreter**，永远不要 `patchelf --set-interpreter` 改它。Bun 的 TLS 区段大小依赖原始 loader 的偏移规则，被 patchelf 改过后会以 `Segmentation fault at address 0x0` 立刻挂掉（这是 [CLAUDE-TERMUX-SETUP.md §3.5](CLAUDE-TERMUX-SETUP.md) 已经强调过的；本方案完全遵守）。

2. **必须用 ld.so 间接加载**：`ld-linux-aarch64.so.1 --library-path $GLIBC/lib /path/to/claude`。
   不要直接 `exec /path/to/claude`（内核拒绝；interpreter `/lib/ld-linux-aarch64.so.1` 在 Termux 文件系统不存在）。

3. **必须保留 LD_PRELOAD=uname-spoof.so**：内核版本 ≥ 5.11 时 Bun 会启用 `epoll_pwait2` → segfault。spoof 让 `uname()` 返回 `5.10.x`，Bun 走回 `epoll_pwait` 旧路径。
   反向检验：删掉 `export LD_PRELOAD=...` 那一行后再跑 `claude --dangerously-skip-permissions`，应再现 panic。

4. **TMPDIR 必须指向 Termux prefix**：Android 没有写入权限的 `/tmp`。`TMPDIR` 和 `CLAUDE_CODE_TMPDIR` 都要设。

5. **DISABLE_AUTOUPDATER=1**：在席自动更新器会下载并安装 stock glibc 二进制覆盖 `~/.local/share/claude/versions/`，并把 `~/.local/bin/claude` 重指过去 — 那个二进制不能在 Termux 启动。同时 `~/.claude/settings.json` 里加 `"autoUpdates": false` 作为第二道防线。

6. **不要把 LD_PRELOAD 设为 `libtermux-exec-ld-preload.so`**：那是 bionic 的预加载库，与 glibc loader 不兼容。

---

## 六、故障排除

### Q1: `claude --version` 报 `No such file or directory`

启动二进制时内核找不到 ELF interpreter。检查：

```bash
glibc-runner /usr/glibc/bin/patchelf --print-interpreter \
  /data/data/com.termux/files/usr/opt/claude-code-termux/current
```

- 输出 `/lib/ld-linux-aarch64.so.1` → 正确，问题在 wrapper（应通过 ld.so 启动，见 3.3）
- 输出别的（例如 `/data/.../glibc/lib/ld-linux-aarch64.so.1`） → 二进制被 patchelf 改过，**重新下载未修改的版本**（见 3.2）

### Q2: `claude` 立刻 `Segmentation fault`（exit 139/133）+ Bun panic

依次检查：

1. `~/.npm-global/bin/claude` 是 bash 脚本，不是符号链接：
   ```bash
   file ~/.npm-global/bin/claude   # 应输出: Bourne-Again shell script
   ```
   如果是符号链接（被 `npm update` 覆盖了），重新执行 §3.3。

2. wrapper 里 `LD_PRELOAD` 指向真实存在的 spoof 库：
   ```bash
   ls -la /data/data/com.termux/files/usr/lib/claude-code-termux/uname-spoof.so
   ```
   不存在 → `pkg install claude-code-termux` 或执行 §3.1 install.sh。

3. wrapper 用的是 `ld.so` 直调，不是 `glibc-runner`：
   ```bash
   grep -E '(glibc-runner|ld-linux)' ~/.npm-global/bin/claude
   ```
   若出现 `glibc-runner` → 重新写 §3.3 的 wrapper。

4. 二进制未被 patchelf：见 Q1。

### Q3: `claude --version` 正常，但 `--dangerously-skip-permissions` 仍崩

LD_PRELOAD 没生效（被某层清掉了）。验证：

```bash
LD_PRELOAD=/data/data/com.termux/files/usr/lib/claude-code-termux/uname-spoof.so \
TMPDIR=/data/data/com.termux/files/usr/tmp \
/data/data/com.termux/files/usr/glibc/lib/ld-linux-aarch64.so.1 \
  --library-path /data/data/com.termux/files/usr/glibc/lib \
  /data/data/com.termux/files/usr/opt/claude-code-termux/current \
  --dangerously-skip-permissions --print "ping"
```

直接这条命令成功 → 是 wrapper 写法有问题，比对 §3.3。
直接这条仍 panic → 检查 spoof 库是否真的拦截 `uname`：
```bash
strace -e uname -- \
  ~/.npm-global/bin/claude --version 2>&1 | grep uname | head -3
# 应看到 uname() 返回包含 release="5.10..." 的结果
```

### Q4: 模型回复成功，但出现 `Cannot find module '/tmp/...'` 类错误

`TMPDIR` 没设到 Termux prefix。确认 wrapper 里有：
```bash
export TMPDIR=/data/data/com.termux/files/usr/tmp
export CLAUDE_CODE_TMPDIR=/data/data/com.termux/files/usr/tmp
```

### Q5: `npm update -g @anthropic-ai/claude-code` 后崩了

npm 覆盖了 `~/.npm-global/bin/claude`（把 bash wrapper 换成 symlink）和 `~/.npm-global/lib/.../bin/claude.exe`（占位脚本）。本方案的二进制存放在 `/usr/opt/claude-code-termux/current`，不受 npm 影响。**只需重新写 §3.3 的 wrapper**。

> 想升级 Claude 版本时：重跑 §3.2（把 `2.1.187` 改成目标版本号），然后跑一遍 `patch-execpath.py`，wrapper 不用动。

### Q6: 我设备能不能跑？

先做预检：
```bash
LD_PRELOAD='' /data/data/com.termux/files/usr/glibc/bin/patchelf --version
```
- 输出 `patchelf 0.18.0` → 设备能直接 exec patched-interpreter glibc 二进制 → 本方案可行
- 报 `CANNOT LINK EXECUTABLE` / `Could not find a PHDR` → 设备内核不支持，回退到 JS 版：
  ```bash
  npm install -g @anthropic-ai/claude-code@2.1.112
  ```

---

## 七、对现有文档的修正点

相对 [CLAUDE-TERMUX-SETUP.md](CLAUDE-TERMUX-SETUP.md)：

| 章节 | 旧描述 | 修正 |
|---|---|---|
| §3.3 启动脚本 | `exec glibc-runner /.../claude.exe "$@"` | 改为 ld.so 直调 + LD_PRELOAD uname-spoof.so（见本文 §3.3） |
| §3.5 注意事项 | "不要设置 LD_PRELOAD：libtermux-exec-ld-preload.so 与 glibc 冲突" | **保留** — 但要补一条：**必须** 设 `LD_PRELOAD=uname-spoof.so`（这是 glibc 端的 spoof 库，不是 bionic 的 termux-exec） |
| §十 Q: Segmentation fault | 列了 2 个原因 | 新增第 3 原因：wrapper 用 glibc-runner 导致 LD_PRELOAD 被 unset（v2.1.180+ + 内核 ≥ 5.11 触发 Bun #32489） |
| §九 性能分析 | "glibc-runner 启动对比" | 新 wrapper 少了一层 bash（glibc-runner.sh），冷启动更快几十 ms |

---

## 八、参考链接

- 上游 launcher 包（提供 uname-spoof.so）: https://github.com/gtbuchanan/claude-code-termux
- Bun segfault on epoll_pwait2: https://github.com/oven-sh/bun/issues/32489
- Termux glibc 包: https://github.com/termux-pacman/glibc-packages
- Claude Code npm 包: https://www.npmjs.com/package/@anthropic-ai/claude-code
- glibc-runner.sh 源码（看 `_glibc-runner_set_up_shell` 的 unset）: `/data/data/com.termux/files/usr/opt/glibc-runner/glibc-runner.sh`

---

## 九、附：验证环境（撰写本文档时）

| 项 | 值 |
|---|---|
| Termux | aarch64 |
| Android 内核 | Linux 6.6.89-android15 |
| glibc | 2.42 |
| Claude Code | 2.1.187 |
| Bun（内嵌） | 1.4.0 |
| 验证命令 | `claude --version`, `claude --dangerously-skip-permissions --print "ping"` |
| 验证结果 | 双双正常，exit 0 |
