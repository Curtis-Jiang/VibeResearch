<h1 align="center">VibeResearch</h1>

<p align="center"><b>面向机器学习的端到端、SOP 驱动的 AI 科研 agent。</b><br/>
从文献调研到论文发布——一条流水线,每个阶段一个 agent,一份在每次运行中持续学习的记忆。</p>

<p align="center">
  <a href="https://github.com/Curtis-Jiang/VibeResearch/blob/main/LICENSE"><img alt="License: Apache-2.0" src="https://img.shields.io/badge/license-Apache--2.0-blue.svg"></a>
  <img alt="Python 3.11+" src="https://img.shields.io/badge/python-3.11%2B-blue.svg">
  <img alt="Status: early development" src="https://img.shields.io/badge/status-early%20development-orange.svg">
</p>

<p align="center"><b><a href="./README.md">English</a> | 简体中文</b></p>

---

## 为什么做 VibeResearch

现有的"AI 科学家"系统大多是两种形态:要么是带科研味 prompt 的聊天助手,要么是只能跑固定实验的硬编码脚本。两者都不符合 ML 科研的真实过程:一套可重复的**标准作业流程(SOP)**,阶段清晰、阶段之间有明确的门控,并且门控不过时要频繁回退。

VibeResearch 把这套 SOP 做成一等公民。科研项目的每个阶段都是一个独立的 agent,拥有确定的输入、输出和通过/失败标准。流水线在阶段之间、会话之间保持状态,项目可以随时暂停、恢复、重新规划、放大规模,或交给人工审核。目标领域是**机器学习 / 深度学习研究**——模型训练、评估,以及最终产出的论文。

VibeResearch 是 [EvoScientist](https://github.com/EvoScientist/EvoScientist) 的 fork。我们保留它的 agent 运行时(DeepAgents + LangChain + LangGraph)、多 provider LLM 层、middleware 栈、记忆底座、CLI/TUI/WebUI 和聊天渠道集成,在其之上构建科研流水线层。

## 科研 SOP

```
  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │ 1. 文献调研   │──▶│ 2. 想法生成   │──▶│ 3. 数据获取   │──▶│ 4. 实验       │
  │              │   │   与提案      │   │              │   │  (MVP→放大)  │
  └──────────────┘   └──────────────┘   └──────────────┘   └──────┬───────┘
         ▲                  ▲                                     │
         │                  │           门控失败:重新规划          │
         │                  └─────────────────────────────────────┤
         │                                                        ▼
  ┌──────┴───────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │ 8. 发布归档   │◀──│ 7. 审稿       │◀──│ 6. 论文写作   │◀──│ 5. 评估回测   │
  │              │   │  (独立)       │   │              │   │              │
  └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
```

| 阶段 | Agent | 输入 → 输出 | 通过门控 |
|------|-------|-------------|----------|
| 1. 文献调研 | `survey-agent` | 主题 → 结构化文献图谱、bibtex 库、开放问题清单 | 关键会议/论文覆盖度经人工或 rubric 确认 |
| 2. 想法生成与提案 | `idea-agent` | 文献图谱 → 排序后的 idea、新颖性检查、研究提案 | 提案含可证伪假设 + 成功指标 |
| 3. 数据获取 | `data-agent` | 提案 → 定位/下载数据集、data card、EDA、泄漏检查 | data card 完整、划分明确、无泄漏告警 |
| 4. 实验(MVP → 放大) | `experiment-agent` | 提案 + 数据 → 小规模验证通过的可运行管线,再放大 | MVP 在容差内复现 baseline 后才放大 |
| 5. 评估与回测 | `eval-agent` | 运行结果 → 多 seed 结果、消融、置信区间、留出集二次评估 | 主指标优于 baseline 且报告不确定性 |
| 6. 论文写作 | `writing-agent` | 结果 + 文献库 → 完整草稿(LaTeX/Markdown)、图表 | 所有结论可追溯到实验产物 |
| 7. 审稿 | `review-agent` | 草稿 → 模拟同行评审、must-fix 清单 | 无未解决的 must-fix 项 |
| 8. 发布与归档 | `publish-agent` | 终稿 → 投稿包、可复现包、发布说明 | 复现包可端到端重跑 |

门控失败会回退——回到重新规划、换一个 idea,或交给人——而不是假装这一步成功了。项目不一定从阶段 1 开始,流水线会根据你已有的东西匹配起点。

## 核心能力

**流水线状态。** 科研项目是一个持久化对象:当前阶段、各阶段产物、门控结果、重试历史。明天、换一台机器、或从聊天渠道里都能接着做。

**每个阶段一个 agent。** 阶段 agent 用 YAML 声明各自的工具、skill 和 system prompt,以 LangGraph 子图运行——同步、异步,或作为长时训练任务的后台进程。

**MVP → 放大。** 实验先在小切片(数据子集、小模型、少量 step)上验证,再做任何昂贵的运行。资源预检(GPU / 显存 / 磁盘)和预算感知的配置决定放大到什么程度。

**结构化实验跟踪。** 每次运行记录 config、seed、git hash、环境和指标为结构化数据;消融表和多 seed 表自动生成而不是手工拼。回测在新 seed 下用留出数据重新评估最终模型。

**文献与文件管理。** 项目级文献库(论文元数据、PDF、笔记、bibtex)由调研、想法生成和写作阶段共享,并且用固定的项目目录结构让每个 agent 知道该读写哪里:

```
<project>/
  literature/     论文、笔记、library.bib
  ideas/          提案及其评分
  data/           数据集、data card、EDA 报告
  experiments/    每次运行一个目录:config、日志、metrics.jsonl、产物
  paper/          草稿、图、表
  reviews/        模拟审稿与处理记录
  memory/         项目级科研记忆
```

**科研记忆。** 不止是对话记忆:哪些 idea 试过、为什么失败,哪类数据上什么超参有效,数据集有什么坑。每个阶段结束后自动蒸馏,供后续项目复用。

**Human-on-the-loop。** 对破坏性或昂贵操作设审批门控,审稿阶段独立于产出结果的阶段,每个阶段都可以从 CLI、TUI、WebUI 或聊天渠道触达。

## 现状

VibeResearch 处于早期开发阶段。下表区分了今天已继承可用的部分和正在构建的部分。

| 方面 | 状态 |
|------|------|
| Agent 运行时、多 provider LLM、middleware(上下文编辑、摘要、工具选择、模型回退)、HITL 审批 | ✅ 继承自 EvoScientist |
| CLI / TUI / WebUI、headless `stream-json`、定时任务、10 个聊天渠道、MCP 与 skills | ✅ 已继承 |
| Observation 记忆 + autoskills | ✅ 已继承 |
| planner / research / code / debug / analysis / writing 子 agent | ✅ 已继承(将重新切分为 SOP 阶段 agent) |
| 带门控和重新规划的显式流水线状态机 | 🚧 进行中 |
| 固定项目目录 + 阶段 agent(survey、idea、data、experiment、eval、review、publish) | 🚧 进行中 |
| 结构化实验跟踪、多 seed / 消融表、回测 | 🔜 计划中 |
| MVP → 放大策略与资源预检 | 🔜 计划中 |
| 学术检索(arXiv、Semantic Scholar、OpenReview)与 bibtex 库 | 🔜 计划中 |
| 项目级科研记忆 | 🔜 计划中 |
| 独立审稿 agent | 🔜 计划中 |

## 快速开始

需要 Python 3.11+ 和 [uv](https://docs.astral.sh/uv/)。

```bash
git clone https://github.com/Curtis-Jiang/VibeResearch.git
cd VibeResearch
uv sync --dev

# 配置 provider(交互向导)或直接导出 key
uv run EvoSci onboard
# export ANTHROPIC_API_KEY=sk-...   # 或 OPENAI_API_KEY / GOOGLE_API_KEY / ...
# export TAVILY_API_KEY=tvly-...    # 可选,网页搜索

uv run EvoSci                      # 交互式 TUI
uv run EvoSci --ui webui           # 浏览器工作台
uv run EvoSci -p "survey recent work on test-time scaling for small LMs"
```

CLI 名称和包布局目前仍继承自 EvoScientist(`EvoSci`、`EvoScientist/`),待流水线层稳定后统一改名。

## 仓库结构

```
EvoScientist/          核心包(runtime、llm、middleware、memory、subagents、channels、cli……)
  subagents/*.yaml     agent 定义——阶段 agent 放在这里
  prompts.py           主 agent prompt 与 SOP 文本
  memory/              observation store、memory workers、autoskills
tests/                 pytest 测试,不需要 API key
docs/                  recipes 与 guides
AGENTS.md              贡献者与编码 agent 的约定
```

## 参与贡献

```bash
uv run ruff check .    # lint
uv run pytest          # 测试(CI 在 Ubuntu 和 Windows 上跑 3.11–3.13)
```

请先阅读 [AGENTS.md](./AGENTS.md) 了解代码地图、约定和 PR checklist。欢迎 issue 和 PR;较大的改动请先开 issue 讨论设计。

## 致谢

VibeResearch 构建于 [EvoScientist](https://github.com/EvoScientist/EvoScientist)(Apache-2.0)及其生态 [EvoSkills](https://github.com/EvoScientist/EvoSkills),以及 [DeepAgents](https://github.com/langchain-ai/deepagents)、[LangChain](https://python.langchain.com/) 和 [LangGraph](https://langchain-ai.github.io/langgraph/)。

## 许可证

Apache License 2.0——见 [LICENSE](./LICENSE)。
