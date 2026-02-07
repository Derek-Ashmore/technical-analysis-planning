# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

This is a **research and planning** repository for a technical analysis application that will analyze historical daily spot price data. There is **no application code yet** -- only research deliverables and planning documents. The project is currently blocked on user answers to critical questions in `notes/Research_Questions_v1.md` (asset class, data source, volume availability, long/short).

## Repository Structure

- `requirements/Main.md` -- Application requirements (what the app should do)
- `instructions/` -- Instructions given to agent teams and command templates
- `research/Research_v1.md` -- Comprehensive research report (indicators, backtesting, metrics, combinations, tooling)
- `notes/Research_Thinking_v1.md` -- Reasoning behind research decisions
- `notes/Research_Questions_v1.md` -- 16 prioritized questions for the user (4 critical, 4 important, 6 clarification, 2 data-dependent)

## Branch Strategy

- `main` -- Base branch
- `claude-agent-teams` -- Claude Code agent teams work
- `claude-flow` -- Reserved for Claude-Flow swarm work

## Key Research Conclusions

Before designing or building the application, understand these core architectural decisions from the research:

1. **ADX-based regime detection** switches between trend-following indicators (MACD, MA crossovers) and mean-reversion indicators (RSI, Stochastic, Bollinger Bands) based on whether ADX > 25 (trending) or < 20 (ranging)
2. **Hierarchical filtering**: Trend filter (Layer 1) -> Confirmation (Layer 2) -> Entry signal (Layer 3). This outperforms single indicators by ~23%
3. **Walk-forward analysis** is the recommended backtesting method (not simple backtesting which overfits)
4. **Recommended stack**: TA-Lib (indicators) + vectorbt (backtesting) + yfinance (data). Avoid pandas-ta (at risk of archival by July 2026)
5. **Composite scoring** for ranking indicators: Sharpe 30%, MaxDD 25%, Profit Factor 20%, Expectancy 15%, Win Rate 10%

## Agent Teams

This repository uses Claude Code agent teams (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`). When creating agent teams for research tasks:

- Use `bypassPermissions` mode for research-only agents doing web searches
- 3 parallel agents works well for this scope
- Compile deliverables from the team-lead rather than delegating to another agent
- Agents may not deliver messages on first attempt -- follow up if needed
