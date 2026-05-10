# GLM + NVIDIA 模型综合评测报告

> 测试日期：2026-05-08 ~ 2026-05-09 | 6维度评分 | 16个模型 | 质量75% + 速度25%

**在线文档：**
- 飞书综合评测报告（含全部8个章节）：https://www.feishu.cn/docx/CE5Xdc9rOolJO5xCwyectHRRnMd
- 飞书NVIDIA评测结果：https://www.feishu.cn/docx/WANLdXWtAodLROxC7ejcBY6RnyY

---

## 一、评测概述

### 评测目标
对智谱GLM系列（8个）和NVIDIA免费API模型（8个）进行中文能力综合评估，为OpenClaw智能体系统选择最优模型配置。

### 评测维度（各0-10分，满分60分）

| 维度 | 测试方法 | 评分重点 |
|------|----------|----------|
| **中文写作** | 300字介绍量子计算基本原理 | 字数(250+/150+/80+)、三个关键词覆盖、原理表述 |
| **逻辑推理** | 三开关三灯问题 | 触摸热度法、等待策略、开关编号、开/关操作 |
| **代码生成** | Python二叉树层序遍历BFS | def定义、队列使用、BFS/层序关键词、循环逻辑 |
| **数学** | 因式分解 x²+5x+6=0 | 正确解(-2,-3)、因式分解格式、验证步骤 |
| **指令遵循** | 列出5种语言(严格格式约束) | 恰好5项、无编号、字母排序、无多余文字 |
| **幻觉检测** | 1真1假1真三问 | Python/Guido正确、虚构人物拒绝回答、诺贝尔奖正确 |

### 综合评分公式
```
综合分 = 质量(总分/60×10) × 75% + 速度分 × 25%
速度分 = (TTFT归一化 + TPS归一化) / 2
```

---

## 二、测试模型

### GLM系列（智谱开放平台）

| 模型 | 特点 | 定价(元/千token) |
|------|------|------------------|
| glm-5.1 | 最新旗舰，主模型 | 输入1.2/输出4.0 |
| glm-5 | 高端模型 | 输入1.0/输出3.2 |
| glm-5-turbo | 旗舰加速版 | 输入1.2/输出4.0 |
| glm-4.7 | 标准模型 | 输入0.6/输出2.2 |
| glm-4.6v-flash | 免费多模态视觉 | 免费 |
| glm-4.6 | 旧版模型 | 输入0.6/输出2.2 |
| glm-4.7-flash | 便宜快速版 | 输入0.07/输出0.4 |
| glm-4.7-flashx | 最便宜版 | 输入0.06/输出0.4 |

### NVIDIA系列（免费API）

| 模型 | 参数量 | 特点 |
|------|--------|------|
| qwen/qwen3.5-122b-a10b | 122B | 通义千问，综合最强 |
| minimaxai/minimax-m2.5 | 128B | MiniMax，速度最快 |
| mistralai/mistral-medium-3.5-128b | 128B | Mistral中端 |
| mistralai/mistral-small-4-119b-2603 | 119B | Mistral轻量 |
| openai/gpt-oss-120b | 120B | OpenAI开源 |
| stepfun-ai/step-3.5-flash | Flash | 阶跃星辰 |
| nvidia/nemotron-3-nano | 30B | 小模型 |
| nvidia/nemotron-3-super | 120B | 大模型 |

---

## 三、评测结果

### 综合排名

| # | 提供商 | 模型 | 总分/60 | 中文 | 推理 | 代码 | 数学 | 指令 | 幻觉 | 速度分 | 综合 |
|---|--------|------|---------|------|------|------|------|------|------|--------|------|
| 1 | NVIDIA | qwen3.5-122b | 57 | 10 | 8 | 10 | 9 | 10 | 10 | 6.0 | **8.6** |
| 2 | NVIDIA | minimax-m2.5 | 47 | 10 | 10 | 10 | 9 | 4 | 4 | 10.0 | **8.4** |
| 3 | NVIDIA | mistral-medium-3.5 | 55 | 10 | 10 | 10 | 9 | 10 | 6 | 5.2 | **8.2** |
| 4 | NVIDIA | mistral-small-4 | 53 | 10 | 8 | 10 | 9 | 10 | 6 | 6.1 | **8.2** |
| 5 | GLM | **glm-5.1** | **59** | **10** | **10** | **10** | **9** | **10** | **10** | 2.6 | **8.0** |
| 6 | NVIDIA | step-3.5-flash | 55 | 10 | 10 | 10 | 9 | 10 | 6 | 3.8 | **7.8** |
| 7 | NVIDIA | gpt-oss-120b | 46 | 9 | 8 | 10 | 9 | 10 | 0 | 6.2 | **7.3** |
| 8 | GLM | glm-4.6v-flash | 50 | 10 | 8 | 10 | 9 | 10 | 3 | 2.9 | **7.0** |
| 9 | GLM | glm-5-turbo | 46 | 10 | 5 | 10 | 1 | 10 | 10 | 3.9 | **6.7** |
| 10 | NVIDIA | nemotron-nano | 37 | 0 | 8 | 10 | 9 | 10 | 0 | 6.0 | **6.1** |
| 11 | GLM | glm-5 | 38 | 10 | 3 | 10 | 2 | 10 | 3 | 2.2 | **5.3** |
| 12 | NVIDIA | nemotron-super | 32 | 0 | 0 | 10 | 9 | 10 | 3 | 4.4 | **5.1** |
| 13 | GLM | glm-4.7 | 30 | 10 | 5 | 3 | 2 | 10 | 0 | 3.2 | **4.5** |
| 14 | GLM | glm-4.6 | 30 | 10 | 0 | 10 | 0 | 10 | 0 | 2.9 | **4.5** |
| 15 | GLM | glm-4.7-flash | 10 | 0 | 0 | 0 | 0 | 10 | 0 | 2.7 | **1.9** |
| 16 | GLM | glm-4.7-flashx | 14 | 0 | 0 | 4 | 0 | 10 | 0 | 0.0 | **1.8** |

### 速度指标

| 提供商 | 模型 | TTFT | TPS | 速度分 |
|--------|------|------|-----|--------|
| NVIDIA | minimax-m2.5 | 765ms | 428.3 | 10.0 |
| NVIDIA | mistral-small-4 | 676ms | 96.6 | 6.1 |
| NVIDIA | gpt-oss-120b | 2394ms | 131.4 | 6.2 |
| NVIDIA | qwen3.5-122b | 1973ms | 107.8 | 6.0 |
| NVIDIA | nemotron-nano | 3086ms | 116.8 | 6.0 |
| NVIDIA | mistral-medium-3.5 | 2669ms | 45.9 | 5.2 |
| NVIDIA | nemotron-super | 8682ms | 67.2 | 4.4 |
| NVIDIA | step-3.5-flash | 10026ms | 31.0 | 3.8 |
| GLM | glm-5-turbo | 9329ms | 33.1 | 3.9 |
| GLM | glm-4.6v-flash | 14909ms | 28.5 | 2.9 |
| GLM | glm-4.6 | 14328ms | 17.4 | 2.9 |
| GLM | glm-4.7 | 12477ms | 13.3 | 3.2 |
| GLM | glm-5.1 | 16000ms | 15.0 | 2.6 |
| GLM | glm-5 | 18526ms | 14.8 | 2.2 |
| GLM | glm-4.7-flash | 14607ms | 1.5 | 2.7 |
| GLM | glm-4.7-flashx | 30646ms | 0.9 | 0.0 |

---

## 四、关键发现

1. **GLM-5.1 质量第一（59/60）但速度最慢**（TTFT 16s，TPS 15）
2. **NVIDIA 模型速度碾压 GLM**（minimax-m2.5 TPS 428.3）
3. **GLM 系列数学能力普遍偏弱**（仅 glm-5.1 和 glm-4.6v-flash 9分合格）
4. **GLM 幻觉检测突出**（glm-5.1、glm-5-turbo 满分10分）
5. **glm-4.7-flash/flashx 不可用于生产**（超时率>20%）
6. **glm-4.6v-flash（免费）性价比极高**（综合7.0，排名第8）

---

## 五、脚本说明

| 文件 | 用途 |
|------|------|
| `bench-glm.sh` | GLM 评测主脚本（v2，含历史回退+快速跳过） |
| `bench-rank.sh` | NVIDIA 评测脚本 |
| `bench-model.sh` | 统一评测脚本（GLM+NVIDIA） |
| `bench-rank-final-*.json` | 最终合并排名数据 |
| `bench-rank-20260507-*.json` | NVIDIA 历史评测数据 |
| `bench-rank-glm-*.json` | GLM 评测数据 |

### 使用方法

```bash
# GLM 评测（需设置 API key）
export GLM_API_KEY="your_key_here"
bash bench-glm.sh

# NVIDIA 评测（需设置环境变量）
export NVIDIA_API_KEY="your_key_here"
bash bench-rank.sh
```
