# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install (development)
pip install .

# Run CLI
tradingagents
# or
python -m cli.main

# Run tests
pytest tests/

# Run a single test file
pytest tests/test_memory_log.py -v

# Run by marker
pytest tests/ -m unit
pytest tests/ -m integration

# Docker
docker compose run --rm tradingagents
```

Test markers: `unit`, `integration`, `smoke`.

## Architecture

TradingAgents is a **LangGraph multi-agent framework** that analyzes stocks through a pipeline of specialized LLM agents. The main entry point is `TradingAgentsGraph` in `tradingagents/graph/trading_graph.py`.

### Pipeline topology

```
Analyst Team (parallel) → Researcher Debate → Trader → Risk Debate → Portfolio Manager
```

1. **Analyst Team** — 4 agents run in parallel (Market, Social, Sentiment, Fundamentals). Each drives a tool-call loop fetching stock data, then writes a markdown report into shared `AgentState`.
2. **Researcher Debate** — Bull and Bear Researchers debate for `max_debate_rounds` rounds, then the **Research Manager** synthesizes a structured `ResearchPlan` (Pydantic schema).
3. **Trader** — Converts the investment plan into a structured `TraderAction`.
4. **Risk Debate** — Aggressive, Neutral, and Conservative debaters argue for `max_risk_discuss_rounds`, then the **Portfolio Manager** issues a final structured `PortfolioAction`.

The graph topology is defined in `tradingagents/graph/setup.py`; conditional routing logic lives in `tradingagents/graph/conditional_logic.py`.

### Key abstractions

**`AgentState`** (`tradingagents/agents/utils/agent_states.py`) — LangGraph state dict shared across all nodes. Contains analyst reports, debate history, investment plan, and final decision.

**Structured outputs** (`tradingagents/agents/schemas.py`) — `ResearchPlan`, `TraderAction`, and `PortfolioAction` are Pydantic models. The three synthesis agents (Research Manager, Trader, Portfolio Manager) use provider-native JSON modes to produce these. This is the only place structured output is used; analysts and debaters produce free-form markdown.

**Dual-LLM strategy** — `TradingAgentsGraph` accepts `deep_think_llm` (for synthesis agents) and `quick_think_llm` (for analysts and debaters). Both are configured independently via `tradingagents/llm_clients/factory.py`.

**Data tool abstraction** (`tradingagents/dataflows/interface.py`) — Tools are vendor-agnostic; routing to yfinance, Alpha Vantage, etc. is set per tool category in `tool_vendors` config. Add new data sources by implementing the vendor module and registering it here.

**Memory & reflection** (`tradingagents/agents/utils/memory.py`) — An append-only markdown decision log at `~/.tradingagents/memory/trading_memory.md` stores past decisions. On re-analysis of the same ticker, past decisions and actual returns are injected as context for reflection.

**Checkpointing** (`tradingagents/graph/checkpointer.py`) — Optional per-ticker SQLite databases under `~/.tradingagents/cache/checkpoints/`. When enabled, each LangGraph node's output is saved; a crashed run resumes from the last successful node.

### LLM provider system

`tradingagents/llm_clients/factory.py` creates provider clients via `create_llm_client(provider, model, ...)`. All clients implement `.get_llm()` returning a LangChain-compatible chat model.

- OpenAI-compatible pool: `openai`, `xai`, `deepseek`, `qwen`, `glm`, `ollama`, `openrouter`
- Specialized clients: `anthropic`, `google`, `azure`
- DeepSeek thinking mode uses a `DeepSeekChatOpenAI` subclass (`tradingagents/llm_clients/deepseek_client.py`)
- Provider-specific kwargs (e.g., `reasoning_effort`, `thinking_level`, `effort`) are passed through `_get_provider_kwargs()` in `trading_graph.py`

### Configuration

All runtime knobs are in `tradingagents/default_config.py`. Key fields: `llm_provider`, `deep_think_model`, `quick_think_model`, `max_debate_rounds`, `max_risk_discuss_rounds`, `tool_vendors`, `output_language`, `online_tools`. Pass overrides as a dict to `TradingAgentsGraph`.

### Tests

Tests live in `tests/`. Notable test files:
- `test_memory_log.py` — Decision log append/parse/reflect behavior
- `test_structured_agents.py` — Pydantic schema validation for all three synthesis agents
- `test_checkpoint_resume.py` — Crash recovery via SQLite checkpointer
- `test_signal_processing.py` — Extracting Buy/Hold/Sell signal from final state
- `test_ticker_validation.py` — Security: ticker sanitization before use as path component
