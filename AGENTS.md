# Marketing Agent — Content Generation

## Overview

This document describes the agent design for the **Content Generation** phase of the NovaMind marketing pipeline. The content generation layer uses `claude-opus-4-6` to produce a weekly blog draft and three persona-specific newsletter variants from a single topic input.

---

## Model

```
claude-opus-4-6
```

- Adaptive thinking enabled (`thinking: {type: "adaptive"}`) for all generation calls
- Streaming enabled for blog generation (long output)
- Prompt caching on all system prompts (`cache_control: {type: "ephemeral"}`)

---

## Agents

### 1. BlogAgent

**Responsibility:** Generate a full blog draft from a topic input.

**Input:**
- `topic` — e.g. `"AI in creative automation"`
- `campaign_date` — e.g. `"2026-04-13"`

**Output:**
```json
{
  "blog_title": "How Creative Agencies Can Use AI Without Losing Their Edge",
  "blog_outline": ["Point 1", "Point 2", "Point 3"],
  "blog_markdown": "# Title\n\n..."
}
```

**Claude API pattern:**
- Streaming (`client.messages.stream`) — blog output is long, prevents HTTP timeouts
- Adaptive thinking — reasons through structure and voice before writing
- System prompt cached — stable across all campaigns

**Prompt file:** `app/prompts/blog.py`

---

### 2. NewsletterAgent (× 3, run in parallel)

**Responsibility:** Generate a persona-targeted newsletter email from the blog topic.

One agent instance per persona. All three run concurrently via `ThreadPoolExecutor`.

**Personas:**

| Persona | Focus |
|---|---|
| `agency_owner` | Growth, margins, client delivery efficiency |
| `ops_manager` | Workflow consistency, automation, team capacity |
| `creative_lead` | Protecting creative time, reducing admin overhead |

**Input:**
- `topic` — same topic as BlogAgent
- `persona` — one of the three above
- `blog_title` — output from BlogAgent (passed as context)

**Output:**
```json
{
  "subject_line": "Cut 5 Hours of Admin Work This Week",
  "body": "Hi [Name], ..."
}
```

**Claude API pattern:**
- Non-streaming (`client.messages.create`) — short output
- Adaptive thinking — tailors tone and framing per persona
- System prompt cached — same system prompt reused across all 3 persona calls

**Prompt file:** `app/prompts/newsletter.py`

---

## Sub-Agent Parallelism

Newsletter generation runs three agents concurrently:

```
ContentService.generate_campaign_content(topic)
    │
    ├── BlogAgent.run(topic, date)          ← sequential first
    │       └── returns blog_title, outline, markdown
    │
    └── [parallel]
        ├── NewsletterAgent.run("agency_owner")
        ├── NewsletterAgent.run("ops_manager")
        └── NewsletterAgent.run("creative_lead")
```

Sequential → parallel order is intentional: newsletters reference the blog title, so BlogAgent must complete first.

---

## Tools

Tools are Python functions called by the orchestrator. Claude does not call these directly in the content generation phase — they are post-generation hooks.

| Tool | Trigger | What it does |
|---|---|---|
| `save_campaign(data)` | After BlogAgent completes | Persists blog title, outline, markdown to SQLite `campaigns` table |
| `save_variants(data)` | After all NewsletterAgents complete | Persists 3 newsletter variants to `content_variants` table |
| `save_blog_artifact(markdown)` | After BlogAgent completes | Writes `data/campaigns/{campaign_id}/blog.md` to disk |
| `save_newsletter_artifact(data)` | After all newsletters complete | Writes `data/campaigns/{campaign_id}/newsletters.json` to disk |

---

## Hooks

### Pre-generation
- Validate `topic` is non-empty
- Check if a campaign for this topic + date already exists in the DB
- If duplicate found: skip generation, load existing content

### Post-generation (per agent)
- Log token usage (`input_tokens`, `output_tokens`, `cache_read_input_tokens`) to console
- Validate JSON structure of response — retry once if malformed

### Post-campaign
- Print summary: blog title + 3 newsletter subject lines
- Mark campaign status as `"generated"` in DB

---

## Prompt Caching Strategy

```
Render order: system → messages

[CACHED]   system prompt     ← stable, same for every campaign
[DYNAMIC]  user message      ← changes per topic/persona
```

Both BlogAgent and NewsletterAgent cache their system prompts. On the first request for a new topic, the system prompt is written to cache. On all 3 newsletter calls (same session), it is read from cache.

Expected savings: ~90% on system prompt tokens for the 3 newsletter calls.

---

## File Structure (Content Generation)

```
marketing_agent/
├── app/
│   ├── config.py              # ANTHROPIC_API_KEY, settings
│   └── prompts/
│       ├── blog.py            # BLOG_SYSTEM_PROMPT + build_blog_prompt()
│       └── newsletter.py      # NEWSLETTER_SYSTEM_PROMPT + build_newsletter_prompt()
├── services/
│   └── content_service.py     # BlogAgent + NewsletterAgent + parallelism
└── data/
    └── campaigns/
        └── {campaign_id}/
            ├── blog.md
            └── newsletters.json
```

---

## Environment Variables

```
ANTHROPIC_API_KEY=sk-ant-...
```

---

## Example Run

```bash
python main.py run --topic "AI in creative automation"
```

Expected output:
```
[content] Generating blog...
[content] Streaming blog generation............
[content] Blog: "How Creative Agencies Can Use AI Without Losing Their Edge"

[content] Generating newsletters (3 personas in parallel)...
[content] ✓ agency_owner — "Scale Faster Without Hiring More"
[content] ✓ ops_manager  — "Cut 5 Hours of Admin Work This Week"
[content] ✓ creative_lead — "Protect Your Creative Time With AI"

[content] Artifacts saved → data/campaigns/cmp_2026_04_13_ai_in_creative_automation/
```
