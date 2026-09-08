# OpenCode on Termux 完整安装与优化指南

> 在 Termux/aarch64 上通过本机 glibc 兼容层运行 OpenCode，零容器、零 proot、极致性能

---

## 一、核心原理

| 组件 | 说明 |
|------|------|
| **运行方式** | 原生 aarch64 ELF 二进制（官方 linux-arm64 构建） |
| **加载器** | glibc `ld-linux-aarch64.so.1` 直调 |
| **兼容层** | Termux glibc 仓库（`glibc`、`glibc-runner` 包） |
| **Wrapper** | 绕过 `glibc-runner`，直接 `exec ld.so --library-path` |
| **性能损耗** | ≈ 0%（与 Linux 裸跑等价；启动多一次 ld.so 显式解析，运行时零开销） |
| **对比 proot+qemu** | 快约 10–50 倍（无系统调用翻译，无指令解码） |

---

## 二、前置依赖（一次性安装）

```bash
# 启用 glibc 仓库并安装基础包
pkg install glibc-repo
pkg update
pkg install glibc glibc-runner patchelf-glibc

# 验证 glibc 可用
/data/data/com.termux/files/usr/glibc/bin/patchelf --version
# 应输出: patchelf 0.19.1 (或更新版本)
```

---

## 三、部署 OpenCode 原生二进制

### 3.1 从官方 npm registry 下载（绕过 GitHub Release 可达性问题）

```bash
# 创建临时目录
TMPD=$(mktemp -d -p /data/data/com.termux/files/usr/tmp)
cd "$TMPD"

# 下载最新版 OpenCode linux-arm64 包
# 版本号可从 https://www.npmjs.com/package/opencode 查询
VERSION=1.18.29  # 替换为最新版本
curl -sL -o opencode.tgz \
  "https://registry.npmjs.org/opencode/-/opencode-${VERSION}.tgz"

# 解包
tar xzf opencode.tgz

# 目标安装目录（复用 glibc 包的 opt 结构）
OPT=/data/data/com.termux/files/usr/opt/opencode
mkdir -p "$OPT"

# 直接复制二进制，不做 patchelf（保持 ELF interpreter 原值）
cp package/bin/opencode "$OPT/opencode-${VERSION}"
chmod +x "$OPT/opencode-${VERSION}"
ln -sfn opencode-${VERSION} "$OPT/current"

# 清理
cd - >/dev/null && rm -rf "$TMPD"
```

### 3.2 验证二进制完整性

```bash
# 检查 ELF interpreter 必须为原值
/data/data/com.termux/files/usr/glibc/bin/patchelf --print-interpreter \
  /data/data/com.termux/files/usr/opt/opencode/current
# 必须输出: /lib/ld-linux-aarch64.so.1

# 检查架构
readelf -h /data/data/com.termux/files/usr/opt/opencode/current | grep Machine
# 必须输出: Machine: AArch64
```

---

## 四、高性能 Launcher（关键：绕过 glibc-runner）

### 4.1 问题背景

`glibc-runner` 在启动时会 `unset LD_PRELOAD`（见 `glibc-runner.sh:18`），导致无法注入自定义 `.so`。虽然 OpenCode 不像 Claude Code（Bun）那样有 `epoll_pwait2` segfault 问题，但绕过 glibc-runner 仍能：
- 减少一层 bash 启动开销（~几十 ms）
- 完全自主控制 `LD_PRELOAD` 与环境变量
- 避免未来 glibc-runner 更新引入的兼容性风险

### 4.2 部署 Wrapper

```bash
cat > ~/.npm-global/bin/opencode << 'WRAPPER'
#!/data/data/com.termux/files/usr/bin/bash
# OpenCode launcher for Termux/aarch64
# 直调 glibc ld.so，自己控制 LD_PRELOAD，避免 glibc-runner unset LD_PRELOAD

GLIBC=/data/data/com.termux/files/usr/glibc
export LD_PRELOAD=""  # 按需添加，如需 uname spoof 等
export TMPDIR=/data/data/com.termux/files/usr/tmp
export PATH="$GLIBC/bin:$PATH"

exec "$GLIBC/lib/ld-linux-aarch64.so.1" --library-path "$GLIBC/lib" \
  /data/data/com.termux/files/usr/opt/opencode/current "$@"
WRAPPER

chmod +x ~/.npm-global/bin/opencode
```

### 4.3 确保 PATH 优先级

```bash
# 在 ~/.bashrc 或 ~/.zshrc 中确保：
export PATH="$HOME/.npm-global/bin:$PATH"
# 然后重新加载
source ~/.bashrc
```

---

## 五、验证与基准测试

```bash
# 版本检查
opencode --version
# 期望: 1.18.29 (或你安装的版本)

# 简单功能测试
opencode --print "echo hello"

# 启动冷延迟测试
time opencode --version
# 期望: ~100-200 ms（Bun/Node 自身开销，与 Linux 一致）

# 内存占用
/usr/bin/time -v opencode --version 2>&1 | grep "Maximum resident"
# 期望: ~30-50 MB RSS
```

### 性能对比表

| 维度 | glibc-runner wrapper | ld.so 直调 wrapper |
|------|---------------------|-------------------|
| 启动开销 | 多一层 bash (~250 行) | 直接 exec ld.so |
| LD_PRELOAD 受控性 | ❌ 被 unset | ✅ 完全自主 |
| 兼容性风险 | 依赖 glibc-runner 维护 | 仅依赖 glibc loader |
| 冷启动延迟 | ~150-250 ms | ~100-200 ms |
| 运行时性能 | 等价 | 等价 |

---

## 六、关键约束（必须遵守）

1. **必须保留 `/lib/ld-linux-aarch64.so.1` 作为 ELF interpreter**，永远不要 `patchelf --set-interpreter` 改它
2. **必须用 ld.so 间接加载**：`ld-linux-aarch64.so.1 --library-path $GLIBC/lib /path/to/opencode`
3. **TMPDIR 必须指向 Termux prefix**：Android 无写入权限的 `/tmp`
4. **不要把 LD_PRELOAD 设为 `libtermux-exec-ld-preload.so`**：那是 bionic 预加载库，与 glibc loader 不兼容

---

## 七、升级 OpenCode 版本

```bash
# 1. 更新 VERSION 变量为新版本号
# 2. 重跑 §3.1 下载/解包/复制步骤
# 3. 无需修改 wrapper（§4.2 指向 current 符号链接）
# 4. 验证：opencode --version
```

---

## 八、故障排除

### Q1: `opencode` 报 `No such file or directory`

```bash
# 检查 interpreter
/data/data/com.termux/files/usr/glibc/bin/patchelf --print-interpreter \
  /data/data/com.termux/files/usr/opt/opencode/current
# 输出 /lib/ld-linux-aarch64.so.1 → wrapper 问题，检查 §4.2
# 输出其他路径 → 二进制被 patchelf 改过，重新下载
```

### Q2: 立刻 `Segmentation fault` / `CANNOT LINK EXECUTABLE`

```bash
# 预检：设备是否支持直接 exec glibc 二进制
LD_PRELOAD='' /data/data/com.termux/files/usr/glibc/bin/patchelf --version
# 输出版本号 → 支持，检查 wrapper 写法
# 报错 → 设备内核不支持，无法运行原生 glibc 二进制
```

### Q3: 模块加载失败 / `Cannot find module`

```bash
# 确认 TMPDIR 设置正确
grep TMPDIR ~/.npm-global/bin/opencode
# 必须包含:
# export TMPDIR=/data/data/com.termux/files/usr/tmp
```

### Q4: 权限错误 `EACCES`

```bash
# 修复二进制权限
chmod +x /data/data/com.termux/files/usr/opt/opencode/current
chmod +x ~/.npm-global/bin/opencode
```

---

## 九、环境信息（编写时验证环境）

| 项 | 值 |
|----|-----|
| Termux | aarch64 |
| Android 内核 | Linux 6.6.89-android15 (OnePlus 13) |
| glibc | 2.44 |
| OpenCode | 1.18.29 |
| 验证命令 | `opencode --version`, `opencode --print "test"` |
| 验证结果 | 双双正常，exit 0 |

---

## 十、参考链接

- Termux glibc 仓库: https://github.com/termux-pacman/glibc-packages
- OpenCode 官网: https://opencode.ai
- OpenCode npm 包: https://www.npmjs.com/package/opencode
- glibc-runner 源码: `/data/data/com.termux/files/usr/opt/glibc-runner/glibc-runner.sh`
- Claude Code 同类方案参考: [CLAUDE-TERMUX-NATIVE-LAUNCHER.md](CLAUDE-TERMUX-NATIVE-LAUNCHER.md)

---

## 十一、致谢

本方案参考了：
- [gtbuchanan/claude-code-termux](https://github.com/gtbuchanan/claude-code-termux) 的 `uname-spoof.so` 思路
- Termux 社区 glibc 兼容层工作
- OpenCode 官方 linux-arm64 构建