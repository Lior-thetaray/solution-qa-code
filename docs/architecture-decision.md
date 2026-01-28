# Solution QA System - Architecture Decision

**Date:** January 28, 2026  
**Status:** Proposed  
**Author:** Solution QA Team

---

## Overview

This document compares different approaches for building an LLM-based Solution QA system and provides a recommendation.

---

## Requirements

| Requirement | Description |
|-------------|-------------|
| **Code Understanding** | Analyze and assess solution implementation code |
| **PostgreSQL Queries** | Read and validate data from databases |
| **Performance Testing** | Measure load times and UI responsiveness |
| **Quality Scoring** | Score and report solution quality |
| **LLM-Based** | Use LLM for reasoning and decision-making |

---

## Options Evaluated

### Option 1: Codex CLI + MCP

OpenAI's agentic CLI tool that runs locally, connects to MCP servers for tools, and executes tasks autonomously.

**Pros:**
- Agentic by design — built for autonomous multi-step tasks with tool calling
- First-class MCP server integration
- Simple orchestration — just shell out to `codex exec --full-auto`
- Local execution — code runs locally, can access files/DBs directly
- Native code understanding

**Cons:**
- Limited control — black-box execution, hard to customize agent behavior
- Debugging — difficult to debug or trace reasoning
- Coupling — tight coupling to OpenAI's tool/model

---

### Option 2: OpenAI SDK Framework

Building custom agents using OpenAI's Python SDK (including the Agents SDK/Swarm patterns).

**Pros:**
- Full control over agent loop, retry logic, scoring
- Transparency — can log every decision, tool call, and response
- Custom scoring — build your own quality scoring rubrics
- Multi-model — can mix models (cheap for routing, powerful for analysis)
- Tool flexibility — define tools as Python functions directly

**Cons:**
- More code — need to implement agent loop, context management, error handling
- No native code understanding — needs file reading tools or code indexing solution
- Vendor lock-in — tied to OpenAI (though can abstract)

---

### Option 3: Copilot SDK / GitHub Copilot Extensions

Building on GitHub Copilot's infrastructure via extensions or the emerging Copilot SDK.

**Pros:**
- Native code understanding — deep IDE/codebase integration
- VS Code integration — can leverage editor context
- GitHub ecosystem — works with PRs, issues, Actions

**Cons:**
- Limited SDK availability — Copilot SDK is still emerging/restricted
- IDE dependency — designed for interactive use, not batch QA
- Less control — harder to run headless/CI pipelines
- Tool limitations — not designed for DB queries, perf testing
- No Azure OpenAI support

---

### Option 4: LangChain/LangGraph

Popular framework for building LLM applications with agent orchestration.

**Pros:**
- Model agnostic — easy to swap OpenAI ↔ Azure ↔ Anthropic ↔ local
- Rich tooling — built-in tools for SQL, web, code execution
- State management — LangGraph provides stateful agent graphs
- Community — large ecosystem of integrations

**Cons:**
- Abstraction overhead — can be over-engineered for simpler tasks
- Learning curve — many concepts to learn
- Debugging — chains can be hard to trace

---

## Comparison Matrix

| Criteria | Codex CLI + MCP | OpenAI SDK | Copilot SDK | LangChain |
|----------|-----------------|------------|-------------|-----------|
| **Code understanding** | ⭐⭐⭐⭐⭐ Native | ⭐⭐ Needs tools | ⭐⭐⭐⭐ Native | ⭐⭐⭐ Needs tools |
| **PostgreSQL tools** | ⭐⭐⭐⭐ Via MCP | ⭐⭐⭐⭐⭐ Direct | ⭐⭐ Limited | ⭐⭐⭐⭐ Built-in |
| **Playwright/UI tools** | ⭐⭐⭐⭐ Via MCP | ⭐⭐⭐⭐⭐ Direct | ⭐⭐ Limited | ⭐⭐⭐⭐ Built-in |
| **Quality scoring** | ⭐⭐⭐ Parse output | ⭐⭐⭐⭐⭐ Structured | ⭐⭐⭐ Parse output | ⭐⭐⭐⭐ Structured |
| **Autonomous execution** | ⭐⭐⭐⭐⭐ Full auto | ⭐⭐⭐ Manual loop | ⭐⭐ Interactive | ⭐⭐⭐⭐ Agent mode |
| **Control & debugging** | ⭐⭐ Black box | ⭐⭐⭐⭐⭐ Full control | ⭐⭐⭐ Moderate | ⭐⭐⭐ Moderate |
| **CI/CD friendly** | ⭐⭐⭐⭐ Yes | ⭐⭐⭐⭐⭐ Yes | ⭐⭐ IDE-focused | ⭐⭐⭐⭐ Yes |
| **Implementation effort** | ⭐⭐⭐⭐⭐ Low (existing) | ⭐⭐⭐ Medium | ⭐⭐ High | ⭐⭐⭐ Medium |
| **Azure OpenAI support** | ⭐⭐⭐⭐⭐ Yes | ⭐⭐⭐⭐⭐ Yes | ⭐⭐ No | ⭐⭐⭐⭐⭐ Yes |

---

## Recommendation: Codex CLI + MCP (Unified Approach)

### Why This Approach?

| Reason | Explanation |
|--------|-------------|
| ✅ **Already built** | We have working MCP infrastructure and orchestrator |
| ✅ **Best code understanding** | Codex natively understands codebases, no extra tooling needed |
| ✅ **Extensible via MCP** | Add PG, Playwright, scoring tools to existing server |
| ✅ **Autonomous** | `--full-auto` handles multi-step reasoning without custom agent loop |
| ✅ **Single architecture** | One system to maintain, not two |
| ✅ **Azure compatible** | Already configured for Azure OpenAI |

### Why Not the Alternatives?

| Alternative | Why Not |
|-------------|---------|
| **OpenAI SDK** | Loses native code understanding, requires more code |
| **Copilot SDK** | Not designed for batch/CI QA, no Azure support |
| **LangChain** | Abstraction overhead, Codex already provides agent loop |
| **Hybrid** | Adds complexity without significant benefit |

---

## Recommended Architecture

```
┌────────────────────────────────────────────────────────────┐
│              Solution QA Orchestrator (Python)             │
│                   agents/solution_qa.py                    │
└────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌────────────────────────────────────────────────────────────┐
│                  Codex CLI (--full-auto)                   │
│                    codex/config.toml                       │
└────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌────────────────────────────────────────────────────────────┐
│                     MCP Server (FastMCP)                   │
│                   mcp_server/mcp_server.py                 │
├────────────────────────────────────────────────────────────┤
│  📊 Risk Model Tools        (existing)                     │
│  🗄️ PostgreSQL Tools        (to add)                       │
│  🎭 Playwright/Perf Tools   (to add)                       │
│  📝 Scoring & Report Tools  (to add)                       │
└────────────────────────────────────────────────────────────┘
```

---

## Implementation Plan

| Phase | Tools to Add | Purpose |
|-------|--------------|---------|
| **Phase 1** ✅ | `read_risk_model_raw`, `validate_risk_model`, etc. | Risk model QA |
| **Phase 2** | `query_postgres`, `list_tables`, `validate_data` | Data validation |
| **Phase 3** | `measure_load_time`, `check_ui_element`, `capture_screenshot` | Performance QA |
| **Phase 4** | `score_feature`, `generate_report`, `compare_with_baseline` | Scoring & reporting |

---

## Decision Summary

| Decision | Choice |
|----------|--------|
| **Architecture** | Codex CLI + MCP (unified) |
| **Why not hybrid?** | Adds complexity without significant benefit |
| **Why not OpenAI SDK?** | Loses native code understanding, more code to write |
| **Why not Copilot SDK?** | Not designed for batch/CI QA, Azure not supported |
| **Why not LangChain?** | Abstraction overhead, Codex already provides agent loop |
| **Next step** | Add PostgreSQL and Playwright MCP tools |

---

## Open Questions

1. **CI/CD Integration** - Should this run in GitHub Actions or a dedicated QA environment?
2. **Scoring Rubrics** - What specific quality metrics should we score?
3. **Baseline Comparison** - Do we need to compare against previous QA runs?
4. **Alerting** - Should failures trigger Slack/email notifications?

---

## References

- [Codex CLI Documentation](https://github.com/openai/codex)
- [MCP Specification](https://modelcontextprotocol.io/)
- [FastMCP Library](https://github.com/jlowin/fastmcp)
- Internal: `solution-qa-code` repository
