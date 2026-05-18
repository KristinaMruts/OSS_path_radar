# OSS Radar — Pathology AI

A Paperclip-ready skill for scheduled monitoring of the open-source computational pathology ecosystem. Scans HuggingFace, arXiv, GitHub, PapersWithCode, high-impact journals, and Microsoft/Google research blogs for new foundation models, datasets, and approaches. Scores each finding 0–19 against a user-configured set of research hypotheses (read at runtime from Notion). Writes rich-analysis cards to a Notion findings DB and pushes a Telegram digest for hot findings.

## What this skill does

1. **Preflight** — Reads your active research hypotheses from Notion. Reads recent findings for deduplication and calibration. Optionally loads a snapshot of your local domain inventory.
2. **Scans 8 source categories** (HuggingFace orgs, arXiv, GitHub, conference proceedings, company blogs, aggregators, journals, research blogs) — sequentially via Task sub-agents to stay within Anthropic rate limits.
3. **Scores each finding** on a 0–19 rubric (relevance + openness + technical).
4. **Writes findings** to a Notion DB with rich analysis (openness flags, training population, clinical limitations, local-context match).
5. **Pushes Telegram digest** for hot findings (raw score ≥ 8 OR full open stack).

## Repository contents

| File | Purpose |
|---|---|
| `SKILL.md` | Main skill definition (Paperclip frontmatter, pipeline, output format, env vars) |
| `scoring-rubric.md` | 0–19 scoring formula, hard rules, anchor examples |
| `source-process.md` | 8 source categories with URLs, scan rules, extractor patterns |
| `README.md` | This file |
| `LICENSE` | MIT |

## How to use

### 1. Install in Paperclip

Add this repository as a Company Skill in Paperclip (Settings → Skills → Add from GitHub). Pin to a specific commit for reproducibility.

### 2. Set up Notion DBs

Create two Notion DBs in your workspace:

**Hypotheses DB** — your active research hypotheses. Required columns:
- A `title` property (any name)
- A `status` select column (any name) — values denoting "active work"
- A `multi-select` column holding topic tags (e.g. specific biomarkers, tumor types) — this is what findings are matched against

**Findings DB** — where the skill writes results. Recommended schema (column names ARE case-sensitive in the API; if you rename, update the skill accordingly):

| Property | Type | Notes |
|---|---|---|
| `Name` | title | Model/paper name |
| `Type` | select | `Model` / `Paper` / `Approach` / `Dataset` / `Benchmark` |
| `Domain` | multi-select | `Histology` / `Cytology` / `CDI` / etc. |
| `Source` | select | `HuggingFace` / `GitHub` / `ArXiv` / `Company Blog` / `Conference` / `PapersWithCode` |
| `Organization` | select | `MahmoodLab` / `Bioptimus` / etc. |
| `Status` | select | `New` / `Review` / `Relevant` / `Not Relevant` / `To Research` / `In Use` |
| `Relevance` | select | `High` / `Medium` / `Low` |
| `Date Found` | date |  |
| `Parameters` | text |  |
| `Datasets` | text |  |
| `URL` | url |  |
| `HuggingFace` | url |  |
| `Weights` | url |  |
| `Dataset URL` | url |  |
| `Summary` | text |  |
| `Notes` | text | Rich analysis — see `SKILL.md` for template |
| `Local Context Match` | text | Comparison to your local domain inventory — see `SKILL.md` |

### 3. Generate a Notion PAT

Create a Personal Access Token at https://notion.so/profile/integrations (workspace must match where your DBs live). Set as the `NOTION_API_KEY` env var.

### 4. Set env vars in Paperclip

See `SKILL.md` for the full list. Minimum required:

```
NOTION_API_KEY          (secret)
NOTION_HYPOTHESES_DS_ID (data source ID, not database ID)
NOTION_FINDINGS_DS_ID   (data source ID)
```

Optional:

```
TELEGRAM_BOT_TOKEN           (secret)
TELEGRAM_CHAT_ID
LOCAL_CONTEXT_ENDPOINT_URL   (HTTP endpoint returning your domain inventory JSON)
HYPOTHESES_ACTIVE_STATUSES   (comma-separated, default: Dev,In progress,To do,Review)
HYPOTHESES_TAGS_COLUMN       (default: Models)
SCAN_WINDOW_DAYS             (default: 4)
```

### 5. Create the agent + Routine

In Paperclip:
- New Agent: Claude Code adapter, Primary model `claude-sonnet-4-5-20250929`, Skip permissions ON, Heartbeat OFF
- Tick this skill under Company Skills
- New Routine with your cron expression (e.g. `7 9 * * 1,4` for Mon+Thu 09:07 local)

## Local context endpoint (optional)

If you want findings annotated with "how does this compare to what we already have", expose a small HTTP service that returns your domain inventory as JSON. The shape is up to you — the LLM uses it as free-form context. Examples of what to include:

- A list of annotation types you've already labeled (with rough sizes)
- A list of organs/localizations your existing datasets cover
- A list of models you already have in production / experimentation

Point `LOCAL_CONTEXT_ENDPOINT_URL` at this service. The skill will populate the `Local Context Match` field with one of `✅ Applicable` / `⚠️ Partial` / `❌ Not applicable` per finding.

## Cost expectation

At Sonnet 4.5 prices, with sequential sub-agents and prompt caching, expect **~$1.0–1.5 per run** (~10–15 minutes wall time). At twice-weekly cadence: **~$15–25 per month**.

If cost too high, options in priority order:
- Reduce frequency (weekly instead of twice-weekly)
- Trim source list (drop categories with low signal in your domain)
- Use a cheaper model (Haiku) for extractor sub-agents, Sonnet only for final classification
- Run Notion-only (skip Telegram)

## License

MIT — see `LICENSE`.
