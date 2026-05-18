---
name: OSS Radar — Pathology AI
key: oss-radar
description: |
  Scan the open-source pathology AI ecosystem (HuggingFace, arXiv, GitHub, PapersWithCode,
  high-impact journals, Microsoft Research blog, Azure Foundry Labs, company blogs) for new
  foundation models, datasets, and approaches relevant to a configurable set of active research
  hypotheses (read from a user-provided Notion DB). Score each finding 0-19 (relevance +
  openness + technical), write rich-analysis cards to a findings Notion DB, push a Telegram
  digest for hot findings (raw score ≥ 8 OR open code+weights). Use this skill when triggered
  for scheduled OSS monitoring or manually via Assign Task.
version: 1.0.0
metadata:
  recommended_cadence: twice weekly (e.g. Mon + Thu)
  config_sources:
    hypotheses_db: Notion DB with active research hypotheses (id in env NOTION_HYPOTHESES_DS_ID)
    findings_db: Notion DB for findings — read for dedup/calibration, write new entries (id in env NOTION_FINDINGS_DS_ID)
    local_context_endpoint: optional HTTP endpoint returning your local domain inventory (url in env LOCAL_CONTEXT_ENDPOINT_URL)
---

# OSS Radar — Open Source Pathology AI Monitor

## Mission

A scheduled digest of new open-source models, papers, datasets in computational pathology. Each finding is scored 0-19 against your active research hypotheses (configured in Notion) and — optionally — against your local domain inventory (e.g. what datasets you have annotated, what models you already use). Hot findings (score ≥ 8 OR full open stack) go to Telegram; everything ≥ 4 goes to the findings Notion DB with rich analysis (openness, training population, clinical limits, local-context match).

**Not a news aggregator** — a filter producing 0-5 hot findings per run, plus 5-15 medium-relevance entries for later review.

## Configuration sources (runtime)

Three things vary between runs and must be **fetched at runtime**, not embedded in the skill:

1. **`active_hypotheses`** — from Notion DB `$NOTION_HYPOTHESES_DS_ID`
   - Filter rows where status indicates active work (configure via `$HYPOTHESES_ACTIVE_STATUSES`, comma-separated, e.g. `Dev,In progress,To do,Review`)
   - Extract multi-select tags from a configurable column (`$HYPOTHESES_TAGS_COLUMN`, default `Models`) — these are your "what we care about" signals (e.g. specific tumor types, stains, biomarkers)
   - These tags drive scoring boosts

2. **`local_context_snapshot`** — from `$LOCAL_CONTEXT_ENDPOINT_URL` (optional)
   - HTTP GET, returns JSON describing your local domain inventory in any shape
   - The LLM uses this as context when filling the `Local Context Match` field for each finding
   - If env var not set or endpoint fails → skip this enrichment, write `n/a — local context unavailable` in that field

3. **`recent_findings`** — from Notion DB `$NOTION_FINDINGS_DS_ID`
   - Query 1: rows where date-found ≥ 90 days ago → build seen-set (URLs + names) for dedup
   - Query 2: last 20 rows with `Status` in (e.g. `Not Relevant`, `Relevant`, `In Use`) → calibration signals (Step 0)

**Output:** write new findings to the SAME `$NOTION_FINDINGS_DS_ID` DB.

Halt conditions:
- Hypotheses DB unreachable → halt + Telegram alert ("can't determine what to score against")
- Findings DB unreachable → halt + Telegram alert ("can't read seen-set, refuse to risk duplicates")
- Local context endpoint unreachable → continue, mark `Local Context Match` as `n/a`

## Notion API access (critical — read before any Notion call)

The Paperclip-managed runtime has **Node.js** but **no `curl` / `jq`**. Use Node's built-in `fetch`.

**All Notion API calls must use the data-source API (2025-09-03) — not the legacy databases API.**

The env vars `NOTION_HYPOTHESES_DS_ID` and `NOTION_FINDINGS_DS_ID` contain **data source IDs**. They go in the `data_sources` URL path.

### Reading rows from a DB

```
POST https://api.notion.com/v1/data_sources/{id}/query
Headers:
  Authorization: Bearer $NOTION_API_KEY
  Notion-Version: 2025-09-03
  Content-Type: application/json
Body: {"page_size": 100, "filter": {...optional...}}
```

### Creating a row (new finding)

```
POST https://api.notion.com/v1/pages
Headers: (same as above)
Body: {
  "parent": {"data_source_id": "$NOTION_FINDINGS_DS_ID"},
  "properties": {...see Recommended schema below...}
}
```

### Updating an existing row (finding has new info — e.g. weights released after preprint)

```
PATCH https://api.notion.com/v1/pages/{page_id}
Headers: (same as above)
Body: {"properties": {...changed fields only...}}
```

### Do NOT use these endpoints — they return 404/400

- ❌ `GET /v1/databases/{id}` (legacy)
- ❌ `POST /v1/databases/{id}/query` (legacy)

If Notion returns "Make sure the relevant pages and databases are shared with your integration" on a legacy endpoint — **don't trust that text**. Verify with `/v1/data_sources/{id}/query` first.

## Pipeline

```
1. Preflight (≤60s)
   - Fetch active_hypotheses from Notion → cache tags (e.g. cancer types, stains)
   - Fetch local_context_snapshot from $LOCAL_CONTEXT_ENDPOINT_URL (optional)
     → on failure: continue with empty snapshot, flag in run summary
   - Fetch recent_findings (last 90 days) from Notion findings DB → build seen-set:
     - seen_urls = lowercased URLs from URL / huggingface / weights / dataset fields
     - seen_names = lowercased Name (model name match when URLs differ)
   - Fetch calibration set: last 20 rows with terminal Status (Not Relevant / Relevant / In Use)
     → derive per-source / per-organization boost or penalty (see scoring-rubric.md)
   - Send Telegram "🟡 OSS Radar started — N active hypotheses, M models in seen-set"

2. Scan (sequential sub-agents — MANDATORY, not parallel)

   Spawn sub-agents via Task tool, one per source category. Total: 8 sub-agents.

   **CRITICAL — spawn sub-agents SEQUENTIALLY, one at a time.**
   Do NOT spawn multiple Task tool calls in parallel (single message with multiple tool_use blocks).
   Anthropic API has per-minute token rate limits; parallel sub-agents hit HTTP 429.

   Sub-agent batches (see source-process.md for full details):
     A. HuggingFace orgs (MahmoodLab, bioptimus, paige-ai, owkin, google, histai, kaiko-ai, etc.)
        + general HF search
     B. arXiv (cs.CV, eess.IV, cs.LG with histopathology / "computational pathology" / "foundation model")
     C. GitHub (topic:computational-pathology, "digital pathology" recently updated)
     D. PapersWithCode + MICCAI/CVPR accepted lists
     E. Company blogs (bioptimus, owkin, paige, aignostics, modella, kaiko)
     F. Aggregators (grand-challenge, awesome-lists, PathBench leaderboard)
     G. High-impact journals + news (Cell/Nature via Scholar+news-medical proxies — direct WebFetch fails 403)
     H. Microsoft Research blog + Azure Foundry Labs + Google Health blog

   Each sub-agent:
     - Receives: scan window (default last 4 days), active_hypotheses tags, seen-set
     - Scans its category per source-process.md
     - Filters out URLs and names already in seen-set BEFORE returning
     - Returns compact JSON [{name, source, organization, url, hf_url, github_url, weights_url,
       dataset_url, license, summary_raw, parameters, datasets_mentioned, openness_signals}, ...]
     - Target: ≤800 tokens per sub-agent response

   This takes ~10-15 minutes total instead of ~3-4 parallel, but avoids 429 errors.
   If a sub-agent hits 429 internally, wait 60s and retry once.

3. Aggregate + score
   - Cross-category dedup (same model from HuggingFace + GitHub → merge into one finding)
   - Apply calibration boost/penalty per source/organization (from Step 1 calibration set)
   - For each unique candidate: compute score per scoring-rubric.md (0-19)
   - Drop anything score < 4 (noise)
   - Apply hard rules from scoring-rubric.md:
     - Open code + open weights → hard min Relevance=High
     - Code-only or paper-with-promised-code → Status=`To Research`, Notes marker [CODE_PROMISED]
     - Closed model from major lab → Relevance ≤ Medium, "closed" flag in Summary
   - Determine Telegram trigger: raw score ≥ 8 OR (open code AND open weights)

4. Local context matching (if local_context_snapshot was loaded)
   For each finding:
   - Pass finding metadata + local_context_snapshot to the LLM as context
   - LLM generates one of: ✅ Applicable / ⚠️ Partial / ❌ Not applicable (with reason)
   - Written to `Local Context Match` field
   - If snapshot was unavailable → write `n/a — local context unavailable this run`

5. Write to Notion findings DB
   - For each finding (score ≥ 4): create new page OR update existing page if matched in seen-set with new info
   - All fields populated per "Recommended schema" below — empty fields are a failure mode

6. Telegram digest (only if Telegram-trigger count > 0)
   - Build compact digest with top 3 hot findings expanded + rest as one-liners
   - Send via direct Telegram API (Node.js fetch — no shell scripts in Paperclip runtime)

7. On any failure
   - Send Telegram "⚠️ partial: <what worked, what didn't>"
   - Update Paperclip issue accordingly
```

**Critical sub-agent rule**: do NOT process source categories inline in the main session. Without sub-agents the context balloons past 200k tokens (each WebFetch is 5-30KB raw). The Task tool is the only way to keep context under control.

## Token economy — critical for cost control

This pipeline can spend $2-3 per run at Sonnet 4.5 prices if done naively (8 source categories × 30+ web fetches each). With the rules below it stays at $1.0-1.5. **Follow these or runs become expensive.**

### 1. Sub-agents per source category (mandatory)
8 source categories → 8 sub-agents, sequential. Each gets fresh context, returns compact JSON.

### 2. Compact extractor pattern
Within each sub-agent: when fetching a source page, **don't keep raw HTML in context**. Extract immediately to compact JSON (~200 tokens per candidate). The classifier in the main session sees only this.

### 3. Dedup before LLM
`seen_urls` and `seen_names` from preflight (Notion query, no separate DB). Each sub-agent receives these and filters BEFORE returning. Saves the entire scoring + Notion-write cost for already-known findings.

### 4. Prompt caching
The skill body + scoring-rubric.md + source-process.md are **stable across runs**. The agent runtime should mark them as `cache_control: {type: "ephemeral"}`. Dynamic content (hypotheses snapshot, candidate list) goes AFTER cached blocks.

### 5. Tiered Cell/Nature handling
Don't WebFetch cell.com / nature.com directly — they return 403 and waste a turn. Use Google Scholar + news-medical proxies (see source-process.md category G).

### 6. Don't reload supplementary skill files in sub-agents
Pass relevant pieces of source-process.md into each sub-agent's input prompt instead of having the sub-agent re-fetch the file.

## Supplementary files (lazy-loaded when needed)

- **`scoring-rubric.md`** — 0-19 scoring formula + hard rules + thresholds + anchor examples. Load in Step 3 (classify).
- **`source-process.md`** — 8 categories of sources with URLs, scan rules, extractor patterns, anti-patterns. Load when spawning sub-agents in Step 2.

## Recommended Notion schema for the findings DB

Create the findings Notion DB with the following properties. **Property names are case-sensitive in the Notion API.** If you rename any of them, update the keys in the skill's output-write step accordingly.

| Property | Type | Notes |
|---|---|---|
| `Name` | title | Model/paper name (required title property) |
| `Type` | select | Options: `Model`, `Paper`, `Approach`, `Dataset`, `Benchmark` |
| `Domain` | multi_select | Options (suggested): `Histology`, `Cytology`, `CDI`, `Oncology`, `General PathAI`, `Artifacts` |
| `Source` | select | Options: `HuggingFace`, `GitHub`, `ArXiv`, `Company Blog`, `Conference`, `PapersWithCode` |
| `Organization` | select | Options (extend as needed): `MahmoodLab`, `Bioptimus`, `PAIGE`, `Owkin`, `Google`, `Microsoft`, `Lunit`, `KatherLab`, `Other` |
| `Status` | select | Options: `New`, `Review`, `Relevant`, `Not Relevant`, `To Research`, `In Use` |
| `Relevance` | select | Options: `High`, `Medium`, `Low` |
| `Date Found` | date | ISO date (today) |
| `Parameters` | text | N parameters, training compute, architecture brief |
| `Datasets` | text | Training datasets with sizes, OR `n/a (private)` |
| `URL` | url | Canonical source URL (paper or repo) |
| `HuggingFace` | url | HF URL (empty if N/A) |
| `Weights` | url | Direct link to weights (empty if not open) |
| `Dataset URL` | url | Dataset URL (empty if not open) |
| `Summary` | text | 2-3 sentences: what it is, what's different, key numbers |
| `Notes` | text | **Rich analysis — see template below. MANDATORY to fill fully.** |
| `Local Context Match` | text | Match against your local inventory — see template. MANDATORY. |

### `Notes` field — structured rich analysis (MANDATORY)

Empty Notes is a failure. Use this template — write `n/a — <reason>` for sub-sections where data is genuinely unknown, but never leave them empty:

```
[score=X/19] hypothesis_match: <tag from active hypotheses or "—">
match: annotation=<task type>, localization=<organ>, stain=<H&E/IHC/etc>
rationale: <1-2 lines — why this score>

📦 Openness:
- code: ✅ <github URL> | ❌ closed | ⏳ promised <ETA>
- weights: ✅ <HF URL> | ❌ closed | ⏳ promised <ETA>
- dataset: ✅ <URL> | ❌ private | ⏳ promised
- license: <name, commercial use Y/N>

📊 Population / training data:
- N patients / N slides / N tiles
- geo: <countries>, age: <range>, sex: <if stated>
- staining: <H&E / IHC marker / multiplex>
- bias risk: <single-center / cohort skew / etc>

⚠️ Clinical limitations / gaps:
- <what's NOT covered>
- <missing classes>
- <missing validation — no external / no prospective>

🎯 Relevance to your work:
- hypothesis: <specific hypothesis name or "—">
- value: <how it could help — fine-tune backbone / eval set / replace existing approach>

[CODE_PROMISED: <ETA or "unknown">]    ← add ONLY if paper promises code but hasn't released
```

### `Local Context Match` field — comparison to your local inventory (MANDATORY)

Templates depend on whether the local context endpoint is configured + responding:

**Endpoint configured and applicable:**
```
✅ Applicable: <what overlaps from your inventory>
   Use case: <fine-tune classification head / use as eval set / compare against your current SOTA>
```

**Endpoint configured, no overlap:**
```
❌ Not applicable: <reason — e.g. "model is on molecular data, your inventory is H&E/IHC slides only">
```

**Endpoint configured, partial match:**
```
⚠️ Partial: <what works — e.g. "tumor data good for backbone fine-tune, but IHC HER2 doesn't cover their multiplex">
```

**Endpoint not configured or failed:**
```
n/a — local context unavailable this run
```

## Telegram digest (only if Telegram-trigger count K > 0)

**API call** (Paperclip env has no `curl` — use Node.js `fetch`):

```
POST https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage
Content-Type: application/json
Body: {
  "chat_id": "{TELEGRAM_CHAT_ID}",
  "text": "<message body — see template below>",
  "disable_web_page_preview": true
}
```

Do NOT set `parse_mode` — send as plain text. Telegram's markdown parser breaks on `[`, `_`, `(`, etc. that appear in paper titles and URLs.

If `TELEGRAM_BOT_TOKEN` or `TELEGRAM_CHAT_ID` env vars are missing → skip Telegram silently (Notion is primary sink). Log `telegram_skipped: not configured` in run summary.

### Telegram trigger (UNAMBIGUOUS)

A finding goes to Telegram if EITHER:
- `raw score ≥ 8` (before hard-rule modifiers to Status/Relevance), OR
- `open code AND open weights` (full open stack — always notify regardless of score)

Status/Relevance modifiers from hard rules **do not block** Telegram. If a finding gets Relevance=Medium due to being closed but raw score=14 — it STILL goes to Telegram (with 🔒 closed marker).

### Telegram message body template (plain text)

```
🔬 OSS Radar — <K> hot findings

1. <Model Name> (<Source>) score=X/19
   match: <hypothesis tag>, <annotation>, <localization>
   📦 code: ✅/❌  weights: ✅/❌  data: ✅/❌  license: <short>
   <one-line summary with key number>
   🎯 Value: <fine-tune / eval / replace X>
   📄 paper/repo: <URL>
   📋 Notion: <Notion page URL>

2. <Model Name 2> (<Source>) score=Y/19
   ...

(up to top 3 expanded)

Plus N more in Notion: <URL of findings DB filtered to today>
Run: <YYYY-MM-DD HH:MM TZ>
```

Remaining hot findings get one-line entries:
```
4. <Name> score=X — see Notion: <URL>
```

**Mandatory fields in each expanded block**: name, source, score, code/weights/data flags, value, Notion URL.

### Run status (Telegram)

Always sent at run start:
```
🟡 OSS Radar started
Active hypotheses: <list of tags>
Seen-set: <N> models in last 90 days
Local context: ✅ loaded | ⚠️ unavailable | — not configured
```

Always sent at end:
```
✅ OSS Radar done — <K> hot, <M> medium-relevance (total <T> findings)
Telegram digest: <yes if K>0 else "no hot findings this run">
Notion: <URL filtered to today>
```

Or on partial failure:
```
⚠️ OSS Radar partial — <what worked, what didn't>
```

## Constraints

- **Never invent.** If a source returned nothing, log it. Don't guess parameters/datasets/etc.
- **Never duplicate.** Dedup via Notion seen-set happens in every sub-agent BEFORE classification.
- **Fill every field.** Empty `Notes` or `Local Context Match` is a failure mode — write `n/a — <reason>` if data genuinely unknown.
- **Output language.** Match the dominant language of your configured Notion DBs. If hypotheses are in Russian, write Notes/Summary in Russian. If English, English. URLs and proper nouns stay in their original form.
- **Per-agent monthly budget** is a hard cap set in Paperclip — respect it. Expect ~$15-25/month at twice-weekly cadence.

## Failure modes

| Failure | Response |
|---|---|
| Notion API auth fails (401/500) | Halt. Telegram "🔴 OSS Radar: Notion auth failed". |
| Hypotheses DB returns 0 active rows | Halt. Telegram "🔴 OSS Radar: no active hypotheses configured". |
| Local context endpoint fails (timeout/500) | Continue. Write "n/a — local context unavailable" in `Local Context Match`. Note in run summary. |
| Findings DB unreachable | Halt. Telegram "🔴 OSS Radar: can't read seen-set, refuse to risk duplicates". |
| Single source category fails (timeout/404) | Skip category, continue with others. Log "X unavailable" in run digest. |
| Anthropic rate-limit (429) | Wait 60s, retry once. Second fail → halt. |
| Telegram notify fails | Continue. Notion is primary sink. Log to run summary. |
| All sub-agents return 0 findings | Send Telegram "✅ Quiet run — 0 new findings". |
| Sub-agent crash (>2 retries) | Skip that source category, continue. Telegram "⚠️ category X failed". |

## Environment variables

| Variable | Required | Description |
|---|---|---|
| `NOTION_API_KEY` | yes | Notion Personal Access Token or Internal Integration secret |
| `NOTION_HYPOTHESES_DS_ID` | yes | Data source ID of your active-hypotheses DB |
| `NOTION_FINDINGS_DS_ID` | yes | Data source ID of your findings DB (read for dedup, write new entries) |
| `HYPOTHESES_ACTIVE_STATUSES` | no | Comma-separated status values that count as active (default `Dev,In progress,To do,Review`) |
| `HYPOTHESES_TAGS_COLUMN` | no | Column name in hypotheses DB holding the topic tags (default `Models`) |
| `LOCAL_CONTEXT_ENDPOINT_URL` | no | HTTP GET URL returning your local domain inventory JSON. If unset, `Local Context Match` becomes `n/a` |
| `TELEGRAM_BOT_TOKEN` | no | Telegram bot token. If unset, Telegram digest is skipped (Notion-only mode) |
| `TELEGRAM_CHAT_ID` | no | Telegram chat to push digest to |
| `SCAN_WINDOW_DAYS` | no | How many days back each source scans (default `4`) |

## See also

- Methodology: `scoring-rubric.md` (this directory)
- Scan procedure: `source-process.md` (this directory)
- Hypothesis configuration: lives in your Notion DB (set by env var, not by this skill)
- Local context provider: optional HTTP service you build separately (returns your domain inventory as JSON)
