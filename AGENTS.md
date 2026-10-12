# AGENTS.md — EvoScientist

Guidance for AI coding agents (and new contributors) working in this repository.
Read this before touching code. `CONTRIBUTING.md` is the human-facing version;
this file is the operational one.

## What this project is

EvoScientist is a self-evolving AI-scientist agent: a main agent (built on
`deepagents` + LangChain + LangGraph) that delegates to specialised sub-agents
(plan, research, code, debug, analyze, write), keeps a cross-session memory,
distils recurring patterns into skills, and is reachable through a CLI/TUI, a
desktop WebUI, a headless LangGraph server and ten messaging channels.

- Language: Python 3.11+ (CI matrix: 3.11 / 3.12 / 3.13 on Ubuntu + Windows)
- Package manager: `uv` (`uv.lock` is the source of truth; `constraints.txt` is generated)
- License: Apache-2.0 · Version in `pyproject.toml` (`[project].version`)
- CLI entry points: `EvoSci`, `evosci`, `EvoScientist`, `evoscientist` → `EvoScientist.cli:main`
- User config: `~/.config/evoscientist/config.yaml` (env prefix `EVOSCIENTIST_`), see `EvoScientist/config/settings.py`

## Commands you will actually run

```bash
uv sync --dev                       # install (add --extra all-channels to mirror CI)
uv run pytest                       # full suite, no API keys needed (~180 files)
uv run pytest tests/test_x.py -k name   # one test
uv run pytest -v --timeout=30       # what CI runs
uv run ruff check .                 # lint (CI gate)
uv run ruff format .                # format (pre-commit runs check --fix + format)
uv run EvoSci -h                    # CLI help
uv run EvoSci -p "question" --output-format stream-json --auto-mode   # headless JSONL run
```

`uv run ruff check .` and `uv run pytest` must both pass before a PR is opened;
they are the CI gates (`.github/workflows/lint.yml`, `test.yml`).

## Repository map

```
EvoScientist/                 the package
  EvoScientist.py             agent graph factory (create_cli_agent, lazy init, MCP cache, HITL config)
  prompts.py                  main-agent system prompt
  paths.py                    Workspace dataclass, run dirs, virtual-path resolution
  config/                     EvoScientistConfig dataclass, load/save, onboarding wizard (config/onboard/)
  llm/                        multi-provider model registry (Anthropic, OpenAI, Google, DeepSeek, NVIDIA, Ollama, OpenRouter, MiniMax, Opper…)
  subagents/                  *.yaml sub-agent specs + _factory.py + expert containers (sync/async)
  middleware/                 LangChain AgentMiddleware stack: memory, context_editing, context_overflow,
                              summarization, tool_selector, skill_pin, skills_reload, model_fallback,
                              ask_user, background, scheduler, code_interpreter, error_normalization, events…
  memory/                     observation store/index/relations, memory workers, autoskills (memory → skill proposals)
  skills/                     built-in skills shipped with the package (find-skills, skill-creator)
  tools/                      search (Tavily), skill managers
  mcp/                        MCP client + registry (tools cached by config signature)
  backends.py                 CompositeBackend: / sandbox, /skills/ read-only merge, /memory/ persistent FS
  stream/                     event emitter → tracker → state → Rich/TUI display; json_sink for stream-json
  cli/                        Typer CLI, classic Rich interactive loop, Textual TUI (tui_*.py, widgets/)
  commands/                   slash commands (/model, /mcp, /skills, /schedule, /autoskills, /experts…)
  channels/                   10 chat platforms (telegram, discord, slack, feishu, wechat, dingtalk, qq, signal, email, imessage)
                              + bus, middleware (dedup/allowlist/pairing/mention), HITL budget. See channels/README.md
  gateway/                    local / composite graph gateways, background runs
  langgraph_dev/              `EvoSci deploy` server: langgraph.json graphs, HTTP app, SDK, stream transformers
  deploy/                     launcher, busy-state, WebUI (@evoscientist/webui via Node)
  cron/ , sessions.py         scheduled tasks; SQLite checkpointer + thread/session store
  setup/                      `EvoSci setup`: git, node, webui, research env installers
tests/                        pytest, asyncio_mode=auto; fakes in tests/fakes.py, tests/stream_v3_fakes.py
docs/                         recipes (macOS 24/7, subscription OAuth) and guides (stream-json protocol)
.github/workflows/            lint, test, build, docker, publish, release-check
```

## Architecture in one picture

```
User (CLI / TUI / WebUI / channels / langgraph SDK)
  └─ Main agent (EvoScientist.py: create_deep_agent)
       ├─ system prompt (prompts.py) — rewritten per turn by context_editing middleware
       ├─ chat model (llm/) + optional auxiliary model for background workers
       ├─ middleware stack (middleware/) — order matters, see _get_default_middleware()
       ├─ CompositeBackend (backends.py): /  → sandbox, /skills/ → merged RO, /memory/ → persistent
       ├─ MCP tools (mcp/) — optional, cached
       └─ task tool → sub-agents from subagents/*.yaml (+ expert containers, async variants)
  Stream events → stream/emitter → tracker/state → Rich display | TUI | JSON sink
  Memory workers (memory/agents) run as separate LangGraph graphs (see langgraph_dev/langgraph.json)
```

Key design rules baked into the code:

- **Lazy everything.** Importing `EvoScientist` must stay cheap; heavy init (deepagents,
  backends, LLM, MCP) happens on first use so `EvoSci config` / `EvoSci onboard` never pay for it.
- **Caches keyed by config.** Chat model and MCP tools are cached by `(model, provider)` /
  config signature; invalidate correctly when adding config fields that affect them.
- **HITL gates.** `execute`, `run_in_background`, `schedule_task`, `delete` require approval on
  attended agents (`HITL_INTERRUPT_ON`). Don't add tools that bypass this.
- **Workspace is the unit.** `paths.Workspace` owns skills, runs, media and memory. Build one at
  the process edge (CLI startup, `/resume`, `serve`, server graph build) and pass it down.
- **Sub-agents are data.** Add/modify sub-agents in `subagents/*.yaml`, not in Python, unless
  the factory itself needs a change.

## Conventions

- Ruff is the only style authority (`pyproject.toml [tool.ruff]`): line length 88, double
  quotes, isort via ruff, rules E/W/F/I/B/C4/UP/PT/PLE/RUF/FURB, target py311. Use modern
  syntax (`X | None`, `match`, `StrEnum`, dataclasses).
- Type hints everywhere; use `from __future__ import annotations` and `TYPE_CHECKING` imports
  to keep import time low.
- Docstrings explain *why* and reference issues/PRs where behaviour is non-obvious
  (the codebase does this a lot — keep it up, e.g. `# see issue #179`).
- Logging via `logging.getLogger(__name__)`; never `print` outside CLI display code.
- Async-first runtime: anything touching the agent/stream/channels should be `async` and
  cancellation-safe (see `cancellation.py`, `runtime.py`).
- Windows is a supported platform (CI runs it). Mind path separators, `_winloop.py`, Git Bash
  for `execute` (#519).
- Commit messages follow Conventional Commits with a scope:
  `feat(cli): …`, `fix(gateway): …`, `ci: …`, `release: vX.Y.Z`. Branch names like
  `fix/session-crash`, `feat/export-csv`.

## Testing rules

- Every behavioural change ships with tests in `tests/`. Tests must run **without API keys or
  network**; mock providers with the fakes in `tests/fakes.py` / `tests/stream_v3_fakes.py`.
- File naming mirrors the module: `test_<module>.py`, with `-k` friendly test names.
- `asyncio_mode = "auto"` — async tests need no decorator.
- Channel code has per-channel test files (`test_telegram_channel.py`, …) plus
  `test_channel_comprehensive.py` / `test_additional_channel_smoke.py`; extend those when
  touching `channels/`.
- Middleware tests follow `test_<name>_middleware.py`; a new middleware needs one.
- CI uses `--timeout=30` per test — avoid real sleeps and real subprocesses.

## Things to NOT do

- Don't edit `constraints.txt` by hand (generated from `uv.lock` on release PRs only).
- Don't bump `pyproject.toml` version outside a `release:` PR.
- Don't commit `.env`, keys, or `~/.config/evoscientist` contents.
- Don't widen dependency pins without a reason comment — several are deliberately capped
  (`openrouter<0.11`, `textual<8.2.7`, `cryptography<49` on Intel macOS); read the comments first.
- Don't add niche research workflows here — they belong in
  [EvoSkills](https://github.com/EvoScientist/EvoSkills) as installable skills.
- Don't `playwright install` / download browsers in CI steps; don't add network calls to tests.

## Where to look when…

| Task | Start at |
|------|----------|
| Add an LLM provider | `llm/registry.py`, `llm/models.py`, `config/settings.py`, `tests/test_llm.py` |
| Add a slash command | `commands/implementation/`, register in `commands/manager.py`, `tests/test_*_command.py` |
| Add a middleware | `middleware/`, wire in `EvoScientist.py:_get_default_middleware`, `tests/test_<name>_middleware.py` |
| Add/modify a sub-agent | `subagents/*.yaml`, `subagents/_factory.py`, `tests/test_load_subagents.py` |
| Add a channel | `channels/<name>/`, `channels/channel_manager.py`, `channels/README.md`, `pyproject.toml` extras |
| Change stream events / stream-json | `stream/events.py`, `stream/json_sink.py`, `docs/guides/stream-json.md` |
| Change config fields | `config/settings.py` (`EvoScientistConfig`), onboarding `config/onboard/steps.py`, `tests/test_config.py` |
| Memory / autoskills | `memory/`, `middleware/memory*.py`, `tests/test_observation_memory.py`, `tests/test_autoskills.py` |
| Headless server / WebUI | `langgraph_dev/`, `deploy/`, `gateway/` |

## PR checklist

1. `uv run ruff check .` and `uv run ruff format --check .` are clean.
2. `uv run pytest` passes locally (ideally also with `--extra all-channels`).
3. New behaviour has tests; changed behaviour has updated tests.
4. User-facing changes update `README.md` (and `README.zh-CN.md` when the section exists there),
   `channels/README.md`, or `docs/` as appropriate.
5. Commit message uses `type(scope): summary`; PR fills `.github/PULL_REQUEST_TEMPLATE.md`.
