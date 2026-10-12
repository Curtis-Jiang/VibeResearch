<h1 align="center">VibeResearch</h1>

<p align="center"><b>An end-to-end, SOP-driven AI research agent for machine learning.</b><br/>
From literature survey to published paper — one pipeline, one agent per stage, a memory that learns from every run.</p>

<p align="center">
  <a href="https://github.com/Curtis-Jiang/VibeResearch/blob/main/LICENSE"><img alt="License: Apache-2.0" src="https://img.shields.io/badge/license-Apache--2.0-blue.svg"></a>
  <img alt="Python 3.11+" src="https://img.shields.io/badge/python-3.11%2B-blue.svg">
  <img alt="Status: early development" src="https://img.shields.io/badge/status-early%20development-orange.svg">
</p>

<p align="center"><b>English | <a href="./README.zh-CN.md">简体中文</a></b></p>

---

## Why VibeResearch

Most "AI scientist" systems are either a chat assistant with research-flavoured prompts, or a rigid script that runs one fixed experiment. Neither matches how ML research actually happens: a repeatable **standard operating procedure** (SOP) with clear stages, explicit gates between them, and a lot of going back when a gate fails.

VibeResearch makes that SOP a first-class object. Every stage of a research project is its own agent with a defined input, output and pass/fail criterion. The pipeline keeps state across stages and across sessions, so a project can be paused, resumed, re-planned, scaled up or handed to a human reviewer at any point. The target domain is **machine learning / deep learning research** — model training, evaluation and the paper that comes out of it.

VibeResearch is a fork of [EvoScientist](https://github.com/EvoScientist/EvoScientist). We keep its agent runtime (DeepAgents + LangChain + LangGraph), multi-provider LLM layer, middleware stack, memory base, CLI/TUI/WebUI and channel integrations, and build the research-pipeline layer on top of it.

## The research SOP

```
  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │ 1. Literature │──▶│ 2. Ideation  │──▶│ 3. Data      │──▶│ 4. Experiment│
  │    survey     │   │  & proposal  │   │  acquisition │   │  (MVP→scale) │
  └──────────────┘   └──────────────┘   └──────────────┘   └──────┬───────┘
         ▲                  ▲                                     │
         │                  │          gate failed: re-plan        │
         │                  └─────────────────────────────────────┤
         │                                                        ▼
  ┌──────┴───────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
  │ 8. Publish   │◀──│ 7. Review    │◀──│ 6. Paper     │◀──│ 5. Evaluation│
  │  & archive   │   │  (independent)│   │  writing     │   │  & backtest  │
  └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
```

| Stage | Agent | Input → Output | Gate to pass |
|------|-------|----------------|--------------|
| 1. Literature survey | `survey-agent` | topic → structured literature map, bibtex library, open-problem list | coverage of the key venues/papers confirmed by human or rubric |
| 2. Ideation & proposal | `idea-agent` | literature map → ranked ideas, novelty check, research proposal | proposal has a falsifiable hypothesis + success metric |
| 3. Data acquisition | `data-agent` | proposal → datasets located/downloaded, data card, EDA, leakage checks | data card complete, splits defined, no leakage flagged |
| 4. Experiment (MVP → scale) | `experiment-agent` | proposal + data → runnable pipeline validated at small scale, then scaled | MVP reproduces baseline within tolerance before scale-up |
| 5. Evaluation & backtest | `eval-agent` | runs → multi-seed results, ablations, confidence intervals, held-out re-evaluation | primary metric beats baseline with uncertainty reported |
| 6. Paper writing | `writing-agent` | results + library → full draft (LaTeX/Markdown), figures, tables | all claims trace to an experiment artifact |
| 7. Review | `review-agent` | draft → simulated peer review, must-fix list | no unresolved must-fix items |
| 8. Publish & archive | `publish-agent` | final draft → submission package, reproducibility bundle, release notes | bundle re-runs end to end |

A gate that fails routes back — to re-planning, to a new idea, or to a human — instead of pretending the step succeeded. Not every project starts at stage 1; the pipeline matches the starting point to what you already have.

## Core capabilities

**Pipeline state.** A research project is a persistent object: current stage, artifacts per stage, gate results, retry history. Resume it tomorrow, from another machine, or from a chat channel.

**One agent per stage.** Stage agents are declared in YAML with their own tools, skills and system prompt, and run as LangGraph sub-graphs — synchronous, async or in the background for long training jobs.

**MVP → scale.** Experiments are validated on a small slice (data subset, tiny model, few steps) before any expensive run. Resource preflight (GPU / VRAM / disk) and budget-aware configs decide how far to scale.

**Structured experiment tracking.** Every run records config, seed, git hash, environment and metrics as structured data; ablation and multi-seed tables are generated, not hand-assembled. Backtesting re-evaluates the final model on held-out data under a fresh seed.

**Literature & file management.** A project-level library (paper metadata, PDFs, notes, bibtex) shared by the survey, ideation and writing stages, plus a fixed project layout so each agent knows where to read and write:

```
<project>/
  literature/     papers, notes, library.bib
  ideas/          proposals and their scores
  data/           datasets, data cards, EDA reports
  experiments/    one folder per run: config, logs, metrics.jsonl, artifacts
  paper/          draft, figures, tables
  reviews/        simulated reviews and resolution log
  memory/         project-level research memory
```

**Research memory.** Beyond conversational memory: which ideas were tried and why they failed, which hyper-parameters worked on which kind of data, dataset pitfalls. Distilled automatically after each stage and reused in later projects.

**Human-on-the-loop.** Approval gates on destructive or expensive actions, a review stage that is independent from the stage that produced the results, and every stage reachable from CLI, TUI, WebUI or a chat channel.

## Status

VibeResearch is in early development. The table shows what is inherited and working today versus what is being built.

| Area | Status |
|------|--------|
| Agent runtime, multi-provider LLMs, middleware (context editing, summarisation, tool selection, model fallback), HITL approvals | ✅ inherited from EvoScientist |
| CLI / TUI / WebUI, headless `stream-json`, scheduled tasks, 10 chat channels, MCP & skills | ✅ inherited |
| Observation memory + autoskills | ✅ inherited |
| Planner / research / code / debug / analysis / writing sub-agents | ✅ inherited (to be re-cut into SOP stage agents) |
| Explicit pipeline state machine with gates and re-planning | 🚧 in progress |
| Fixed project layout + stage agents (survey, idea, data, experiment, eval, review, publish) | 🚧 in progress |
| Structured experiment tracking, multi-seed / ablation tables, backtest | 🔜 planned |
| MVP → scale strategy with resource preflight | 🔜 planned |
| Academic search (arXiv, Semantic Scholar, OpenReview) and bibtex library | 🔜 planned |
| Project-level research memory | 🔜 planned |
| Independent review agent | 🔜 planned |

## Quick start

Requires Python 3.11+ and [uv](https://docs.astral.sh/uv/).

```bash
git clone https://github.com/Curtis-Jiang/VibeResearch.git
cd VibeResearch
uv sync --dev

# configure a provider (interactive wizard) or export a key
uv run EvoSci onboard
# export ANTHROPIC_API_KEY=sk-...   # or OPENAI_API_KEY / GOOGLE_API_KEY / ...
# export TAVILY_API_KEY=tvly-...    # optional, web search

uv run EvoSci                      # interactive TUI
uv run EvoSci --ui webui           # browser workspace
uv run EvoSci -p "survey recent work on test-time scaling for small LMs"
```

The CLI name and package layout are still inherited from EvoScientist (`EvoSci`, `EvoScientist/`); they will be renamed once the pipeline layer stabilises.

## Repository layout

```
EvoScientist/          core package (runtime, llm, middleware, memory, subagents, channels, cli, …)
  subagents/*.yaml     agent definitions — stage agents live here
  prompts.py           main-agent prompt and the SOP text
  memory/              observation store, memory workers, autoskills
tests/                 pytest suite, no API keys needed
docs/                  recipes and guides
AGENTS.md              conventions for contributors and coding agents
```

## Contributing

```bash
uv run ruff check .    # lint
uv run pytest          # tests (CI runs 3.11–3.13 on Ubuntu and Windows)
```

Read [AGENTS.md](./AGENTS.md) for the code map, conventions and PR checklist. Issues and PRs are welcome; for larger changes open an issue first so we can agree on the design.

## Acknowledgments

VibeResearch builds on [EvoScientist](https://github.com/EvoScientist/EvoScientist) (Apache-2.0) and its ecosystem of [EvoSkills](https://github.com/EvoScientist/EvoSkills), and on [DeepAgents](https://github.com/langchain-ai/deepagents), [LangChain](https://python.langchain.com/) and [LangGraph](https://langchain-ai.github.io/langgraph/).

## License

Apache License 2.0 — see [LICENSE](./LICENSE).
