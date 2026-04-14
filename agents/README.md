# Marketing Agent — Agent Index

This folder documents every agent in the NovaMind marketing pipeline. Each file covers one phase: its agents, tools, hooks, and Claude API patterns.

## Pipeline Overview

```
Topic Input
    │
    ▼
01. content_generator.md  → ResearchAgent + BlogAgent + NewsletterAgents (×3)
    │
    ▼
02. crm_agent.md          → ContactSyncAgent + SegmentationAgent + DispatchAgent
    │
    ▼
03. analytics_agent.md    → MetricsCollectorAgent
    │
    ▼
04. optimization_agent.md → OptimizationAgent (AI performance summary + recommendations)
```

## Files

| File | Phase | Agents |
|---|---|---|
| [content_generator.md](./content_generator.md) | 1 — Content | ResearchAgent, BlogAgent, NewsletterAgent ×3 |
| [crm_agent.md](./crm_agent.md) | 2 — CRM / Leads | ContactSyncAgent, SegmentationAgent, DispatchAgent |
| [analytics_agent.md](./analytics_agent.md) | 3 — Analytics | MetricsCollectorAgent |
| [optimization_agent.md](./optimization_agent.md) | 4 — Optimization | OptimizationAgent |

## Model

All agents use `claude-opus-4-6` with adaptive thinking unless noted otherwise.

## Shared Environment Variables

```
ANTHROPIC_API_KEY=sk-ant-...
HUBSPOT_ACCESS_TOKEN=...       # leave empty to run in simulation mode
```
