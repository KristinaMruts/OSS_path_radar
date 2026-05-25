---
name: OSS Radar — Pathology AI
key: oss-radar
description: |
  Scan the open-source pathology AI ecosystem (HuggingFace, arXiv, GitHub, PapersWithCode,
  high-impact journals, Microsoft Research blog, Azure Foundry Labs, company blogs) for new
  foundation models, datasets, and approaches relevant to a configurable set of active research
  hypotheses (read from a user-provided Notion DB). Apply a HARD domain filter (pathology only).
  Score each finding 0-21 (relevance + openness + technical) using EXACT rubric components.
  Apply applicability filter (drop findings outside user's local stack). Write rich-analysis
  cards to a findings Notion DB, push digests to Telegram (plain text) and/or Mattermost
  (markdown) for ALL new findings (HIGH + MED), grouped by relevance bucket. Use this skill
  when triggered for scheduled OSS monitoring or manually via Assign Task.
version: 1.4.0
metadata:
  recommended_cadence: twice weekly (e.g. Mon + Thu)
  config_sources:
    hypotheses_db: Notion DB with active research hypotheses (id in env NOTION_HYPOTHESES_DS_ID)
    watched_sources_db: Notion DB with HF orgs + company blogs to scan (id in env NOTION_WATCHED_SOURCES_DS_ID)
    findings_db: Notion DB for findings — read for dedup/calibration, write new entries (id in env NOTION_FINDINGS_DS_ID)
    local_context_endpoint: optional HTTP endpoint returning your local domain inventory (url in env LOCAL_CONTEXT_ENDPOINT_URL)
  changelog:
    "1.4.0": Digest includes BOTH HIGH and MED findings (previously HIGH only). Visual bucket separation (━━━ 🔴 HIGH ━━━ / ━━━ 🟡 MED ━━━). Trigger simplified to "any new finding this run".
    "1.3.1": Digest format — show ALL hot findings expanded (no top-3 cap), explicit Russian labels (Оценка/Источник/Код/Dataset/Описание), splitting rules for Telegram 4096 / Mattermost 16383 char limits
    "1.3.0": Add Mattermost output channel (parallel to Telegram, markdown-rich format via REST API + PAT)
    "1.2.0": Inline scoring rubric, hard domain filter, applicability filter, enforce Notes/Telegram templates, fix max score 19→21
    "1.1.0": Externalize HF orgs + company blogs to Notion config DB
    "1.0.0": Initial public version
---

# OSS Radar — Open Source Pathology AI Monitor

## Mission

A scheduled digest of new open-source models, papers, datasets in computational pathology. Each finding is filtered (hard domain check), scored 0-21 against your active hypotheses, and assessed for applicability against your local stack. Hot findings (raw score ≥ 8 OR full open stack) go to Telegram; everything ≥ 4 AND applicable to your stack goes to the findings Notion DB with full rich analysis.

**Not a news aggregator** — a filter producing 0-15 findings per run (typically 0-3 HIGH + 3-10 MED), all delivered in the digest with HIGH/MED visually separated. Findings that aren't usable in your stack (e.g. radiology when you do pathology, spatial transcriptomics when you do H&E only) are dropped or marked Not Relevant — they don't pollute the digest or findings DB.

## Configuration sources (runtime)

Four things vary between runs and must be **fetched at runtime**, not embedded in the skill:

1. **`active_hypotheses`** — from Notion DB `$NOTION_HYPOTHESES_DS_ID`
   - Filter rows where status indicates active work (configure via `$HYPOTHESES_ACTIVE_STATUSES`, comma-separated, e.g. `Dev,In progress,To do,Review`)
   - Extract multi-select tags from a configurable column (`$HYPOTHESES_TAGS_COLUMN`, default `Models`) — these are your "what we care about" signals
   - These tags drive scoring boosts

2. **`watched_sources`** — from Notion DB `$NOTION_WATCHED_SOURCES_DS_ID`
   - Required columns: `Name` (title), `Type` (select: `hf_org` / `company_blog`), `URL` (url), `Status` (select: `active` / `inactive`), `Notes` (text)
   - Filter rows where `Status = active`
   - Rows with `Type = hf_org` → scan targets for Category A; rows with `Type = company_blog` → Category E
   - The skill does NOT hardcode any organization or blog URLs

3. **`local_context_snapshot`** — from `$LOCAL_CONTEXT_ENDPOINT_URL` (optional)
   - HTTP GET, returns JSON describing your local domain inventory in any shape
   - Used by the **applicability filter** (Step 4) and `Local Context Match` field
   - If env var not set or endpoint fails → applicability filter skipped, write `n/a — local context unavailable` in field

4. **`recent_findings`** — from Notion DB `$NOTION_FINDINGS_DS_ID`
   - Query 1: rows where date-found ≥ 90 days ago → seen-set for dedup
   - Query 2: last 20 rows with `Status` in (`Not Relevant`, `Relevant`, `In Use`) → calibration

**Output:** write new findings to the SAME `$NOTION_FINDINGS_DS_ID` DB.

Halt conditions:
- Hypotheses DB unreachable → halt + Telegram alert
- Watched sources DB unreachable → halt + Telegram alert
- Findings DB unreachable → halt + Telegram alert
- Local context endpoint unreachable → continue, write `n/a` in field

## Domain filter — HARD (applies BEFORE scoring)

**This is the first filter applied to every candidate, BOTH inside extractor sub-agents AND in the main session classifier (defense in depth).**

### IN SCOPE — score normally

- Histology (H&E, IHC, special stains on tissue sections)
- Cytology (FNA, Pap smears, urine, body fluid smears)
- Spatial omics + histology (when histology is the primary modality the model operates on directly)
- Immunohistochemistry (any marker)
- Molecular pathology coupled with histology
- WSI analysis, tile classification, cell detection, mitosis counting, tumor segmentation, stain normalization, gland detection

### OUT OF SCOPE — DROP without scoring, do NOT write to Notion

- **Radiology**: X-ray, chest X-ray, CT, MRI, mammography imaging (not pathology slides), ultrasound, PET, fluoroscopy
- **Dermatology imaging**: clinical skin photos, dermoscopy
- **Ophthalmology**: retinal scans, OCT, fundus
- **Cardiology**: ECG, echocardiogram, cardiac MRI
- **Endoscopy**: colonoscopy video, gastroscopy, capsule endoscopy
- **General medical NLP**: clinical note generation, EHR-only models, medical Q&A without imaging
- **Genomics-only**: pure DNA/RNA sequence models without paired imaging
- **Surgical / robotic surgery video**

### Ambiguous cases

If you cannot confidently determine the domain from the source page → DROP with rationale `out-of-domain: ambiguous, source did not clarify modality`. Better to skip a real pathology finding than pollute the findings DB with non-pathology noise.

### How the filter is applied

- **Extractor sub-agents** (Step 2): apply filter BEFORE returning candidates. Out-of-scope items never reach the main session. Each sub-agent logs a count of `filtered_out_of_scope` in its return summary.
- **Main session classifier** (Step 3): apply filter again as defense in depth, before scoring. If a candidate has slipped through, drop it now with rationale.
- **Hard rule**: no out-of-scope item should ever be written to Notion. If you see one, regenerate.

## Notion API access (critical — read before any Notion call)

The Paperclip-managed runtime has **Node.js** but **no `curl` / `jq`**. Use Node's built-in `fetch`.

**All Notion API calls must use the data-source API (2025-09-03) — not the legacy databases API.**

The env vars `NOTION_HYPOTHESES_DS_ID` / `NOTION_FINDINGS_DS_ID` / `NOTION_WATCHED_SOURCES_DS_ID` contain **data source IDs**. They go in the `data_sources` URL path.

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

### Updating an existing row
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
   - Fetch active_hypotheses from Notion → cache tags
   - Fetch watched_sources from Notion → build hf_orgs + company_blogs scan-target lists
   - Fetch local_context_snapshot from $LOCAL_CONTEXT_ENDPOINT_URL (optional)
   - Fetch recent_findings (last 90 days) from Notion findings DB → build seen-set:
     - seen_urls (lowercased URL/huggingface/weights/dataset fields)
     - seen_names (lowercased Name)
   - Fetch calibration set: last 20 rows with terminal Status → derive boosts
   - Send Telegram "🟡 OSS Radar started — N hypotheses, M scan targets, K seen"

2. Scan (sequential sub-agents — MANDATORY, not parallel)

   8 sub-agents, one per source category. CRITICAL: spawn SEQUENTIALLY.

   Each sub-agent:
     - Receives: scan window, active_hypotheses tags, seen-set, (A/E only) watched_sources list
     - Apply DOMAIN FILTER (see above) — out-of-scope items dropped immediately, never returned
     - Filter out URLs and names already in seen-set BEFORE returning
     - Returns compact JSON [{name, source, organization, url, hf_url, github_url, weights_url,
       dataset_url, license, summary_raw, parameters, datasets_mentioned, openness_signals, domain_check}, ...]
     - Returns count: candidates_found, filtered_out_of_scope, filtered_already_seen
     - Target: ≤800 tokens per sub-agent response

3. Aggregate + score (classifier in main session)
   - Cross-category dedup
   - Apply DOMAIN FILTER again (defense in depth) — drop any out-of-scope item that slipped through
   - For each remaining candidate: compute score per RUBRIC (inline below)
     - Use EXACT component names listed in rubric. Do NOT invent components like "base", "recency".
   - Apply calibration boost/penalty (from preflight)
   - Drop anything score < 4 (noise)
   - Apply hard rules (inline below)
   - All findings ≥ 4 that survive applicability filter go into the digest (HIGH + MED).
     No separate "trigger" — everything written to Notion in this run is also in the digest.

4. Local context matching + applicability filter (CRITICAL)
   For each remaining candidate:
   - Pass finding metadata + local_context_snapshot to the LLM
   - Assess Local Context Match: ✅ Applicable / ⚠️ Partial / ❌ Not applicable (with reason)

   APPLICABILITY POLICY:
     - Local Context = ✅ Applicable → write normally
     - Local Context = ⚠️ Partial → write normally
     - Local Context = ❌ Not applicable AND raw_score < 12 → DROP, do NOT write to Notion
     - Local Context = ❌ Not applicable AND raw_score ≥ 12 → write with Status=Not Relevant
       (high-score out-of-stack findings preserved as research signal but explicitly marked)
     - Local context unavailable (endpoint failed) → skip this filter, write all candidates normally

5. Write to Notion findings DB
   - For each surviving finding: create new page OR update existing if matched in seen-set with new info
   - VALIDATE Notes field BEFORE writing (see "Notes validation" below)
   - VALIDATE all required schema fields filled (Domain, Type, Source, Organization, Status, Relevance, Date Found, URL, Summary, Notes, Local Context Match)

6. Digests (Telegram + Mattermost — both independent, neither blocks the other)
   - Build digest with ALL findings expanded, grouped by bucket:
     ━━━ 🔴 HIGH (N) ━━━ first, sorted by score desc
     ━━━ 🟡 MED  (M) ━━━ second, sorted by score desc
   - Continuous numbering across buckets (1, 2, ... N+M)
   - VALIDATE every expanded block has all mandatory fields
   - If a block can't be validated → DROP from digest, don't send abbreviated
   - Send Telegram digest (only if total findings > 0 AND Telegram env vars set)
   - Send Mattermost digest (only if total findings > 0 AND Mattermost env vars set)
   - If one channel fails, the other still proceeds — log which succeeded in run summary
   - If 0 findings → send "✅ Quiet run" to configured channels

7. On any failure
   - Send to whichever channel is configured: "⚠️ partial: <what failed>"
```

## Token economy — critical for cost control

### 1. Sub-agents per source category (mandatory)
8 sequential sub-agents. Each gets fresh context, returns compact JSON.

### 2. Compact extractor pattern
Extract immediately to compact JSON (~200 tokens per candidate). Don't keep raw HTML in context.

### 3. Dedup before LLM
seen_urls + seen_names passed to each sub-agent. Filter BEFORE returning.

### 4. Domain filter saves money
Domain filter at extractor stage drops out-of-scope items before they consume scoring tokens. Radiology / cardiology / ECG papers should never reach the classifier.

### 5. Applicability filter saves Notion writes
Step 4's applicability filter prevents low-score out-of-stack findings from polluting the findings DB.

### 6. Prompt caching
Skill body marked `cache_control: {type: "ephemeral"}`.

### 7. Tiered Cell/Nature handling
Don't WebFetch cell.com / nature.com directly (403). Use Scholar + news-medical proxies (see source-process.md G).

---

# SCORING RUBRIC (INLINE — canonical version, agent MUST follow exactly)

> **Why inline:** This rubric is the most important part of the skill. To prevent the agent from improvising or inventing scoring components, it lives here in SKILL.md (always loaded), NOT in a lazy-loaded supplementary file.

## Score range: 0–21 (raw, before hard-rule modifiers)

Components break down as:
- Relevance components: max +11 (5 + 3 + 2 + 1)
- Openness components: max +7 (2 + 2 + 2 + 1)
- Technical components: max +3 (2 + 1)
- Calibration: ±2 (signed adjustment)

## Relevance — max +11

| Criterion | Points |
|---|---|
| **hypothesis_match** — finding directly matches a tag from your active hypotheses (e.g. specific biomarker, tumor type, artefact category) | +5 |
| **annotation_match** — finding's task aligns with an annotation type you care about (e.g. segmentation, mitosis count, scoring) | +3 |
| **localization_match** — finding's organ/tissue matches your area (e.g. mammary, GI, lung, prostate, lymph) | +2 |
| **stain_match** — finding uses a stain you work with (e.g. H&E, specific IHC marker) | +1 |

If hypothesis_match = 0 (no direct tag match), points from annotation/localization/stain can still accumulate, but see Hard Rule 0 below.

## Openness — max +7

| Criterion | Points |
|---|---|
| **openness.code** — public GitHub repo with real code (not paper-only) | +2 |
| **openness.weights** — weights downloadable (HuggingFace, direct link) | +2 |
| **openness.dataset** — training/eval data downloadable | +2 |
| **openness.license** — permissive (MIT / Apache 2.0 / CC-BY) on top of open code/weights | +1 |

## Technical — max +3

| Criterion | Points |
|---|---|
| **technical.sota** — SOTA on a pathology benchmark (Patho-Bench, PathBench, EVA, HEST) | +2 |
| **technical.novel_arch** — foundation model or novel architecture (not yet-another-UNet) | +1 |

## Calibration — ±2 (signed)

Derived from last 20 findings with terminal Status:
- Source appeared 5+ times as `Not Relevant` → −1
- Organization appeared 5+ times as `Not Relevant` → −1
- Source + Domain combo appeared 3+ times as `Relevant`/`In Use` → +1
- Architecture family identical to an `In Use` entry → +1

Sum is capped at ±2 total.

## MANDATORY scoring header in Notes (verbatim format)

Every Notes field MUST start with the following block. Use these EXACT component names. Do NOT invent components like "base", "recency", "freshness", "popularity". Each component must appear; write 0 if N/A.

```
[score=X/21]
- hypothesis_match: N (of +5)
- annotation_match: N (of +3)
- localization_match: N (of +2)
- stain_match: N (of +1)
- openness.code: N (of +2)
- openness.weights: N (of +2)
- openness.dataset: N (of +2)
- openness.license: N (of +1)
- technical.sota: N (of +2)
- technical.novel_arch: N (of +1)
- calibration: ±N

hypothesis_tag_matched: <tag name from active hypotheses, or "—">
match: annotation=<type>, localization=<organ>, stain=<stain>
rationale: <1-2 sentences explaining the score>
```

If you find yourself writing "base=N" or "recency=N" — STOP. Those aren't in the rubric. Use the components above only.

## Thresholds → Notion classification

| Raw score | Notion Status | Notion Relevance |
|---|---|---|
| ≥ 8 | `Review` | `High` |
| 4–7 | `New` | `Medium` |
| < 4 | (drop — don't write) | — |

## Hard rules (override default classification)

### Hard Rule 0 — Hypothesis-zero floor
If `hypothesis_match = 0` AND total raw score < 8 AND not a "full open stack" → DROP, do not write. Pure openness without hypothesis match is interesting only at very high score.

### Hard Rule 1 — Full open stack → minimum Relevance=High
If `openness.code = +2` AND `openness.weights = +2` (both open):
- Notion `Relevance` = `High` (even if raw score < 8)
- Notion `Status` = `Review`
- Telegram: ALWAYS send (overrides raw-score threshold)
- BUT: still subject to Hard Filter (domain) and Applicability Filter (Step 4)

### Hard Rule 2 — Closed model from major lab → Relevance ≤ Medium
If finding is closed (no public weights, no public code) but from a major lab (PAIGE, Owkin, Bioptimus, major pharma R&D):
- Notion `Relevance` ≤ `Medium` (downgrade even if score ≥ 8)
- Notion `Status` = `Review`
- Summary notes: "🔒 closed — paper available, weights/code not released"
- Telegram: send if raw score ≥ 8

### Hard Rule 3 — Code-promised paper
If paper says "code coming soon" / "weights upon publication":
- Notion `Status` = `To Research`
- Notion `Relevance` per threshold
- Add to Notes: `[CODE_PROMISED: <ETA or "unknown">]` exactly
- Telegram: send if raw score ≥ 8

### Hard Rule 4 — Dataset-only release
If finding is purely a dataset (no model):
- Notion `Type` = `Dataset`
- Openness criteria: open data = +2, license = +1

## Digest trigger (UNAMBIGUOUS)

A finding goes to the digest (Telegram + Mattermost) if it is **written to Notion in this run** — i.e. it survived:
- Domain filter (Step 2/3 — pathology only)
- Score threshold (Step 3 — raw score ≥ 4)
- Applicability filter (Step 4 — not "Not applicable" with score < 12)

In other words: everything in the day's Notion findings page is in the digest, grouped by HIGH (score ≥ 8 or full open stack per Hard Rule 1) and MED (4-7).

There is NO separate "send to Telegram only the hot ones" filter anymore (was removed in v1.4.0).

## Anchor examples

### Example: Strong full-open release in hypothesis area
A foundation model from a tracked lab, code+weights, matches Tumor hypothesis.
- hypothesis_match=+5, annotation_match=+3, localization_match=+2 (any), stain_match=+1 (H&E)
- openness.code=+2, openness.weights=+2, openness.dataset=+1 (partial), openness.license=+0 (CC-BY-NC)
- technical.sota=+2, technical.novel_arch=+1
- calibration=+1 (lab seen as `In Use` before)
- **Raw: 19/21** → Status=Review, Relevance=High, Telegram=YES

### Example: Radiology paper that mentions "tumor"
Chest X-ray model for lung tumor detection.
- **DOMAIN FILTER applies → DROP at extractor stage. Never reaches scoring.**

### Example: Pathology foundation model but requires ST input
Spatial transcriptomics + histology FM (e.g. STORM).
- Passes domain filter (uses histology)
- hypothesis_match=0 (no ST hypothesis), annotation_match=+3 (tumor), localization_match=+2, stain_match=+1
- openness: paper-only +0
- technical.novel_arch=+1
- **Raw: 7/21**
- Local Context Match: ❌ Not applicable (we don't have ST input pipeline)
- raw_score (7) < 12 → APPLICABILITY FILTER drops it. Not written to Notion.

### Example: Closed model from major lab, matches hypothesis
Hypothetical PAIGE closed release, matches Tumor hypothesis.
- hypothesis_match=+5, annotation_match=+3, localization_match=+2, stain_match=+1
- openness: all closed +0
- technical.sota=+2, technical.novel_arch=+1
- **Raw: 14/21**
- Hard Rule 2: closed from major lab → Relevance=Medium (not High despite score)
- Telegram: YES (raw ≥ 8)

---

# RECOMMENDED NOTION SCHEMA — findings DB

Create the findings Notion DB with the following properties. **Property names are case-sensitive in the Notion API.**

| Property | Type | Notes |
|---|---|---|
| `Name` | title | Model/paper name |
| `Type` | select | `Model` / `Paper` / `Approach` / `Dataset` / `Benchmark` |
| `Domain` | multi_select | `Histology` / `Cytology` / `CDI` / `Oncology` / `General PathAI` / `Artifacts` |
| `Source` | select | `HuggingFace` / `GitHub` / `ArXiv` / `Company Blog` / `Conference` / `PapersWithCode` |
| `Organization` | select | Extend as needed |
| `Status` | select | `New` / `Review` / `Relevant` / `Not Relevant` / `To Research` / `In Use` |
| `Relevance` | select | `High` / `Medium` / `Low` |
| `Date Found` | date | ISO date (today) |
| `Parameters` | text | N params, training compute, architecture brief |
| `Datasets` | text | Training datasets with sizes, OR `n/a (private)` |
| `URL` | url | Canonical source URL |
| `HuggingFace` | url |  |
| `Weights` | url |  |
| `Dataset URL` | url |  |
| `Summary` | text | 2-3 sentences |
| `Notes` | text | **Rich analysis — see validation below. MANDATORY full template.** |
| `Local Context Match` | text | **Match against your inventory — see validation below. MANDATORY.** |

## `Notes` field — MANDATORY validation (run before write)

The Notes field MUST contain ALL of these sections in order:

```
[score breakdown header — exact format from rubric above]

📦 Openness:
- code: ✅ <URL> | ❌ closed | ⏳ promised <ETA>
- weights: ✅ <URL> | ❌ closed | ⏳ promised <ETA>
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
- <missing validation>

🎯 Relevance to your work:
- hypothesis: <specific hypothesis name or "—">
- value: <how it could help — fine-tune backbone / eval set / replace existing approach>

[CODE_PROMISED: <ETA or "unknown">]    ← add ONLY if paper promises code but hasn't released
```

### Validation checklist (run before EVERY Notion write)

- [ ] score breakdown header present with all 11 components (no invented ones)?
- [ ] 📦 Openness section present with all 4 sub-items?
- [ ] 📊 Population section present with at least 3 sub-items?
- [ ] ⚠️ Limitations section present with at least 1 bullet?
- [ ] 🎯 Relevance section present with hypothesis + value?

If ANY check fails → regenerate Notes before writing. Empty or abbreviated Notes is a HARD failure mode — Kristina has flagged this explicitly as unacceptable. Do not write to Notion until all checks pass.

For sub-sections where data is genuinely unknown, write `n/a — <one-sentence reason>` but NEVER omit the section.

## `Local Context Match` field — MANDATORY validation

Templates depend on whether local context endpoint was loaded:

**Endpoint loaded, applicable:**
```
✅ Applicable: <what overlaps from your inventory>
   Use case: <fine-tune classification head / use as eval set / compare against SOTA>
```

**Endpoint loaded, no overlap:**
```
❌ Not applicable: <reason — e.g. "model requires spatial transcriptomics on input, your inventory is H&E/IHC slides only">
```

**Endpoint loaded, partial:**
```
⚠️ Partial: <what works / what doesn't>
```

**Endpoint not configured or failed:**
```
n/a — local context unavailable this run
```

## Telegram digest — MANDATORY validation

**API call** (Paperclip env has no `curl` — use Node.js `fetch`):

```
POST https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage
Content-Type: application/json
Body: {
  "chat_id": "{TELEGRAM_CHAT_ID}",
  "text": "<message body>",
  "disable_web_page_preview": true
}
```

Do NOT set `parse_mode` — plain text only.

### Telegram message body — mandatory template

**Include ALL findings (HIGH + MED) expanded, grouped by bucket. No top-N truncation.**

```
🔬 OSS Radar — <T> новых findings (<H> HIGH, <M> MED) · <YYYY-MM-DD>

━━━ 🔴 HIGH (<H>) ━━━

1. <Model Name>
   Оценка: X/21
   Источник: <HuggingFace | ArXiv | GitHub | ...>
   Код: ✅/❌/⏳  Веса: ✅/❌/⏳  Dataset: ✅/❌/⏳  Лицензия: <short>
   Match: <hypothesis tag>, <annotation>, <localization>
   Описание: <one-line summary with key number — what is it, what's special>
   🎯 Ценность: <fine-tune / eval / replace / research signal>
   📄 <paper/repo URL>
   📋 Notion: <Notion page URL>

2. <Model Name 2>
   Оценка: Y/21
   ...

━━━ 🟡 MED (<M>) ━━━

<H+1>. <Model Name 3>
   Оценка: Z/21
   ...

(continue numbering across buckets, expand ALL findings — do not truncate)

Run: <YYYY-MM-DD HH:MM TZ>
```

If a bucket is empty, omit its header (e.g. "0 HIGH, 5 MED" → only show MED block).

### Splitting long messages

Telegram has a **4096 character limit** per message. If the assembled digest exceeds this:
- Split into multiple sequential messages: header `🔬 OSS Radar (часть P/N) — продолжение`
- Each part is a complete message (no mid-block split) — break at finding boundaries
- Validation still applies per-block

### Telegram validation checklist (run BEFORE sending each block)

- [ ] Name present?
- [ ] `Оценка: X/21` present?
- [ ] `Источник:` present?
- [ ] `Код:` / `Веса:` / `Dataset:` / `Лицензия:` all 4 present?
- [ ] `Match:` line present (with hypothesis tag OR "—", annotation, localization)?
- [ ] `Описание:` present (NOT empty)?
- [ ] `🎯 Ценность:` present (NOT empty)?
- [ ] paper/repo URL present?
- [ ] Notion page URL present?

If ANY check fails for a block → DROP this finding from Telegram digest (it still goes to Notion). Do NOT send abbreviated/incomplete messages — better to skip one entry than confuse with broken cards.

### Run status (Telegram) — always sent (if Telegram configured)

Start:
```
🟡 OSS Radar started
Active hypotheses: <list>
Watched sources: <N hf_orgs + M blogs>
Seen-set: <K> models in last 90 days
Local context: ✅ loaded | ⚠️ unavailable | — not configured
```

End:
```
✅ OSS Radar done — <T> findings (<H> HIGH, <M> MED)
Filtered: <X> out-of-domain, <Y> not-applicable
Digest: Telegram <yes/no>, Mattermost <yes/no>
Notion: <URL filtered to today>
```

Partial failure:
```
⚠️ OSS Radar partial — <what worked, what didn't>
```

## Mattermost digest (parallel to Telegram, independent)

Mattermost supports full markdown (unlike Telegram), so the format is richer: bold names, hyperlinked URLs, blockquotes for summaries. Both channels receive the same content, just formatted differently.

If `MATTERMOST_URL`, `MATTERMOST_TOKEN`, AND `MATTERMOST_CHANNEL_ID` are NOT all set → skip Mattermost silently (Notion + Telegram still work). Log `mattermost_skipped: not configured` in run summary.

### API call (Personal Access Token auth, REST API)

```
POST {MATTERMOST_URL}/api/v4/posts
Headers:
  Authorization: Bearer {MATTERMOST_TOKEN}
  Content-Type: application/json
Body:
{
  "channel_id": "{MATTERMOST_CHANNEL_ID}",
  "message": "<markdown body — see template below>",
  "props": {
    "override_username": "{MATTERMOST_USERNAME}"   // optional, requires server-side EnablePostUsernameOverride
  }
}
```

Use Node.js `fetch` (no `curl` in Paperclip runtime). The token is a Personal Access Token — generated by the user in Profile → Security → Personal Access Tokens. The user posts as the token's owner (the override_username field only works if the Mattermost admin enabled it).

If the API returns 401 → token invalid/expired. If 403 → user lacks permission to post in channel. If 404 → wrong URL or channel_id. If 429 → rate-limited (rare).

### Mattermost message body — mandatory template (markdown)

**Include ALL findings (HIGH + MED) expanded, grouped by bucket. No top-N truncation.**

```markdown
## 🔬 OSS Radar — <T> новых findings · <YYYY-MM-DD>
**<H> HIGH · <M> MED**

---

## 🔴 HIGH (<H>)

### 1. [<Model Name>](<canonical URL>) — **<X>/21**

**Оценка:** X/21  
**Источник:** <HuggingFace | ArXiv | GitHub | ...>  
**Код** <✅/❌/⏳> · **Веса** <✅/❌/⏳> · **Dataset** <✅/❌/⏳> · **Лицензия:** <short>  
**Match:** <hypothesis tag> / <annotation> / <localization> (<stain>)

> <Описание: one-line summary with key number>

🎯 **Ценность:** <fine-tune / eval / replace / research signal>  
📋 [Notion entry](<Notion page URL>)

---

### 2. [<Model Name 2>](<URL>) — **<Y>/21** ...

...

---

## 🟡 MED (<M>)

### <H+1>. [<Model Name>](<URL>) — **<Z>/21** ...

...
```

(Continue numbering across buckets, expand ALL findings with the structure above, separated by `---`. No truncation. If a bucket is empty, omit its header.)

### Splitting long messages

Mattermost default max post size is **16383 characters** (server-configurable). If the assembled digest exceeds this:
- Split into multiple sequential posts: header `## 🔬 OSS Radar (часть P/N) — продолжение`
- Break at finding boundaries (never mid-block)
- 16K is rarely hit (≈ 20+ findings); if hit, calibration probably needs review

### Mattermost validation checklist (run BEFORE sending each block)

Same content rules as Telegram:

- [ ] Name + URL present (clickable hyperlink)?
- [ ] **Оценка** X/21 present?
- [ ] **Источник** present?
- [ ] **Код / Веса / Dataset / Лицензия** all 4 present?
- [ ] **Match** line present?
- [ ] Описание (blockquote) present (NOT empty)?
- [ ] 🎯 **Ценность** present (NOT empty)?
- [ ] Notion URL present?

If ANY check fails for a block → DROP this finding from Mattermost digest. Better to skip than send broken card.

### Run status (Mattermost) — always sent if Mattermost configured

Start:
```markdown
🟡 **OSS Radar started** — <N> hypotheses, <M> scan targets, <K> seen
```

End:
```markdown
✅ **OSS Radar done** — <T> findings (<H> HIGH, <M> MED) · [today's findings](<Notion URL>)
```

Partial failure:
```markdown
⚠️ **OSS Radar partial** — <what worked, what didn't>
```

Quiet run:
```markdown
✅ **OSS Radar quiet run** — 0 new findings
```

## Constraints

- **Never invent.** If a source returned nothing, log it. Don't guess parameters/datasets/etc.
- **Never invent scoring components.** Use ONLY the 11 components from the rubric. No "base", "recency", etc.
- **Never duplicate.** Dedup via Notion seen-set in every sub-agent BEFORE classification.
- **Fill every Notes section.** Empty/abbreviated Notes = HARD failure. Write `n/a — <reason>` if truly unknown, never omit.
- **Domain filter is HARD.** Out-of-pathology findings never reach Notion. No exceptions.
- **Applicability filter is GRADED.** Low-score not-applicable = drop. High-score not-applicable = write with Not Relevant.
- **Output language.** Match dominant language of your Notion DBs (Russian if hypotheses are in Russian, English otherwise). URLs and proper nouns stay in original form.
- **Per-agent monthly budget** — respect Paperclip cap. Expect ~$15-25/month at twice-weekly cadence.

## Failure modes

| Failure | Response |
|---|---|
| Notion API auth fails (401/500) | Halt. Telegram "🔴 OSS Radar: Notion auth failed". |
| Hypotheses DB returns 0 active rows | Halt. Telegram "🔴 OSS Radar: no active hypotheses configured". |
| Watched sources DB unreachable | Halt. Telegram "🔴 OSS Radar: can't read watched sources DB". |
| Watched sources DB has 0 active rows | Continue. Skip categories A and E. Note in run summary. |
| Local context endpoint fails | Continue. Write "n/a — local context unavailable" in field. Note in summary. |
| Findings DB unreachable | Halt. Telegram "🔴 OSS Radar: can't read seen-set, refuse to risk duplicates". |
| Single source category fails | Skip category, continue. Log in run digest. |
| Anthropic rate-limit (429) | Wait 60s, retry once. Second fail → halt. |
| Telegram notify fails | Continue. Notion + Mattermost still work. Log to run summary. |
| Mattermost notify fails (401/403/404/429) | Continue. Notion + Telegram still work. Log which error in run summary. |
| Neither Telegram nor Mattermost configured | Continue (Notion-only mode). Log `digests_skipped: not configured`. |
| All sub-agents return 0 findings | Send "✅ Quiet run — 0 new findings" to whichever channels are configured. |
| Notes validation fails after 2 regenerations | Skip this finding, log warning in summary. |
| Domain filter catches >50% of candidates | Continue but log. May indicate scan targets need tuning. |

## Environment variables

| Variable | Required | Description |
|---|---|---|
| `NOTION_API_KEY` | yes | Notion PAT or Internal Integration secret |
| `NOTION_HYPOTHESES_DS_ID` | yes | Data source ID of hypotheses DB |
| `NOTION_FINDINGS_DS_ID` | yes | Data source ID of findings DB (read + write) |
| `NOTION_WATCHED_SOURCES_DS_ID` | yes | Data source ID of watched-sources DB |
| `HYPOTHESES_ACTIVE_STATUSES` | no | CSV active statuses (default `Dev,In progress,To do,Review`) |
| `HYPOTHESES_TAGS_COLUMN` | no | Tags column name (default `Models`) |
| `LOCAL_CONTEXT_ENDPOINT_URL` | no | HTTP GET URL for local domain inventory JSON |
| `TELEGRAM_BOT_TOKEN` | no | Telegram bot token (skip Telegram if unset) |
| `TELEGRAM_CHAT_ID` | no | Telegram chat ID |
| `MATTERMOST_URL` | no | Mattermost base URL (no trailing slash, e.g. `https://mattermost.example.com`). Skip Mattermost if any of the 3 Mattermost vars missing. |
| `MATTERMOST_TOKEN` | no | Personal Access Token (from Mattermost Profile → Security → Personal Access Tokens) |
| `MATTERMOST_CHANNEL_ID` | no | 26-char Channel ID where to post (from channel ⋮ → View Info) |
| `MATTERMOST_USERNAME` | no | Override author username (requires server-side `EnablePostUsernameOverride=true`). Default: posts as PAT owner. |
| `SCAN_WINDOW_DAYS` | no | Default `4` |

## See also

- Scan procedure: `source-process.md` (this directory)
- Scoring rubric reference (with examples): `scoring-rubric.md` (this directory) — canonical version is inline above, this file is reference only
- Hypothesis config: lives in your Notion DB
- Local context provider: optional HTTP service you build separately
