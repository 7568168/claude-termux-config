# Claude Code on Termux

Termux/Android 环境下 Claude Code 完整配置与优化，包含五个子项目：

- **[Claude Code + OMC 配置与优化指南](CLAUDE-TERMUX-SETUP.md)** — Termux 安装最新版 Claude Code、OMC 集成、claude-mem 自动记忆系统融合、更新持久化
- **[原生启动器修正（v2.1.180+ 必读）](CLAUDE-TERMUX-NATIVE-LAUNCHER.md)** — 修复 glibc-runner wrapper 在新 Bun + 新内核下 `--dangerously-skip-permissions` segfault；用 ld.so 直调 + uname-spoof.so 注入，零性能损耗
- **[OpenCode 极致性能安装指南](OPENCODE-TERMUX-SETUP.md)** — Termux 本机 glibc 兼容层运行 OpenCode，零容器、零 proot、ld.so 直调，性能 ≈ Linux 裸跑
- **[OpenClaw 完整优化指南](OPENCLAW-TERMUX-SETUP.md)** — Workspace 深度优化（条件注入+紧凑标记+token 节省）、子 Agent 配置、问题排查、模型评测与对比分析
- **[GLM + NVIDIA 模型综合评测](benchmark/README.md)** — 16 个模型 × 6 维度基准测试，含评测脚本与排名数据
- **[GitHub Hosts 本地加速](github-hosts/README.md)** — 本地获取最优 GitHub hosts 的脚本，5 路并行 DNS + TCP 测速选最优 IP，支持完整/快速/远程三种模式

---

**在线文档：**
- 飞书综合评测报告：https://www.feishu.cn/docx/CE5Xdc9rOolJO5xCwyectHRRnMd
- 飞书 NVIDIA 评测结果：https://www.feishu.cn/docx/WANLdXWtAodLROxC7ejcBY6RnyY
- 飞书 OpenClaw 优化文档：https://www.feishu.cn/docx/FKWddJw2uopZ38xOFr8cE5rUn0f
