# OSS Radar Scoring Rubric

Score range: **0–19** (raw, before hard-rule modifiers).

Score determines:
1. Notion `Status` (default — see Thresholds)
2. Notion `Relevance` (default — see Thresholds, can be raised/lowered by hard rules)
3. Telegram trigger (raw score ≥ 8 OR open code+weights)

## Relevance (clinical / business fit) — max +11

| Criterion | Points |
|---|---|
| **Direct hypothesis match** — finding directly matches a tag from your configured active hypotheses (e.g. specific biomarker, tumor type, artefact category) | +5 |
| **Annotation type match** — finding's task aligns with a domain you care about (e.g. segmentation, detection, scoring) | +3 |
| **Localization match** — finding's organ/tissue matches your area (e.g. mammary, reproductive, GI, lung, prostate, lymph) | +2 |
| **Stain match** — finding uses a stain you work with (e.g. H&E, specific IHC marker) | +1 |

A direct match to one of your hypothesis tags is the strongest signal. If a finding matches multiple criteria (e.g. exact biomarker + matching organ + H&E), sum them all (+5+2+1=+8 base).

## Openness — max +7 (critical — открытый стек = реально можно использовать)

| Criterion | Points |
|---|---|
| **Open code** — public GitHub repo with real code (not paper-only) | +2 |
| **Open weights** — weights downloadable (HuggingFace, direct link) | +2 |
| **Open dataset** — training/eval data downloadable | +2 |
| **Permissive license** — MIT / Apache 2.0 / CC-BY on top of open code/weights | +1 |

Openness is heavily weighted because closed models (private weights, commercial-only) are unactionable for experimentation. You can read the paper but can't fine-tune or evaluate.

## Technical — max +3

| Criterion | Points |
|---|---|
| **SOTA on a pathology benchmark** (Patho-Bench / PathBench / EVA / HEST / similar) | +2 |
| **Foundation model or novel architecture** (not yet-another-UNet) | +1 |
| **Language compatibility note** — if model has language-specific gating (e.g. report generation in English only), flag in Notes | +0 |

## Calibration boost/penalty (applied BEFORE thresholds)

From Step 0 preflight (last 20 rows in findings DB with terminal statuses):

- If finding's `Source` appeared 5+ times as `Not Relevant` recently → **−1**
- If finding's `Organization` appeared 5+ times as `Not Relevant` recently → **−1**
- If finding's `Source` + `Domain` combo appeared 3+ times as `Relevant`/`In Use` recently → **+1**
- If finding looks identical to an `In Use` entry (same architecture family) → **+1**

Calibration adjustments are capped at ±2 total. Track which adjustments fired in the `rationale:` line of Notes.

## Thresholds → Notion classification

| Raw score | Notion Status | Notion Relevance |
|---|---|---|
| ≥ 8 | `Review` | `High` |
| 4–7 | `New` | `Medium` |
| < 4 | (drop — don't write to Notion) | — |

## Hard rules (OVERRIDE thresholds for Notion Status/Relevance)

These modify Notion classification but **DO NOT affect Telegram trigger** (Telegram uses raw score, unmodified).

### Rule 1: Full open stack → minimum Relevance=High

If finding has **open code AND open weights** (both ✅):
- Notion `Relevance` = `High` (even if raw score < 8)
- Notion `Status` = `Review`
- Telegram: ALWAYS send (overrides raw-score threshold)
- Rationale: full open stack is the rarest, most actionable signal

### Rule 2: Closed model from major lab → Relevance ≤ Medium

If finding is closed (no public weights, no public code) but comes from a major lab (PAIGE, Owkin, Bioptimus, major pharma R&D):
- Notion `Relevance` ≤ `Medium` (downgrade from High even if score ≥ 8)
- Notion `Status` = `Review`
- Note in `Summary`: "🔒 closed — paper available, weights/code not released"
- Telegram: send if raw score ≥ 8 (competitive intel — important to know about)

### Rule 3: Code-promised paper → Status=To Research, marker

If paper says "code coming soon" / "code will be released" / "weights upon publication":
- Notion `Status` = `To Research`
- Notion `Relevance` per threshold
- Add to `Notes`: `[CODE_PROMISED: <ETA or "unknown">]` exactly (case-sensitive, brackets included)
- Telegram: send if raw score ≥ 8
- Rationale: a separate code-promise-tracker skill (if you build one) can watch for these markers and re-check weekly

### Rule 4: Dataset-only release → Type=Dataset

If finding is purely a new dataset release (no model):
- Notion `Type` = `Dataset`
- Score still applies per rubric (openness criteria become: open data = +2, license = +1)
- Local context matching becomes critical — applicable datasets should bubble up

## Telegram trigger (UNAMBIGUOUS — restate)

A finding goes to Telegram digest if EITHER:
- raw score ≥ 8 (before any hard-rule modifiers to Status/Relevance), OR
- open code AND open weights (full open stack)

A finding with raw score 5 and full open stack → Telegram ✅
A finding with raw score 14 but closed → Telegram ✅ (still hot competitive intel)
A finding with raw score 7 and code-only (no weights) → Telegram ❌ (close-but-not-quite, lives in Notion only)

## Anchor examples (calibration sanity checks)

### Example 1: Strong foundation model release with full open stack

Hypothetical: a major lab releases a new ViT-based pathology FM on HuggingFace with code + weights.

- Hypothesis tag match: yes (e.g. tumor segmentation) (+5)
- Annotation type match (tumor): (+3)
- Localization: any organ (+2)
- Stain: H&E (+1)
- Open code: ✅ (+2), open weights: ✅ (+2), open dataset: partial (+1)
- License: CC-BY-NC (no commercial → +0)
- Foundation model (+1), SOTA on EVA benchmark (+2)
- **Raw score: 19/19**
- Calibration: same lab seen 3× as `In Use` → +1 (cap at +2, no change)
- Notion: Status=Review, Relevance=High
- Telegram: YES (both raw≥8 AND open stack)

### Example 2: Random arXiv preprint on an irrelevant cancer type, no code

- Hypothesis tag match: none (—)
- Annotation type (tumor): (+3)
- Localization: rare cancer not in your scope (—)
- Stain: H&E (+1)
- Open code: paper-only (+0), weights: ❌ (+0)
- Foundation model: no, UNet variant (+0)
- **Raw score: 4/19**
- Calibration: arXiv seen 8× as `Not Relevant` → −1
- Adjusted score: 3 → DROP (don't write to Notion)

### Example 3: Closed model from major lab

Hypothetical: PAIGE releases a closed model paper, no public weights.

- Hypothesis tag match (tumor): (+5)
- Annotation type (tumor): (+3), localization: any (+2), stain: H&E (+1)
- Open code: ❌ (+0), weights: ❌ (+0), dataset: ❌ (+0)
- Foundation model (+1), SOTA (+2)
- **Raw score: 14/19**
- Hard rule 2: closed from major lab → Relevance=Medium (not High despite score)
- Notion: Status=Review, Relevance=Medium, Summary notes "🔒 closed"
- Telegram: YES (raw ≥ 8)

### Example 4: Niche tool with full open stack but obscure benchmark

Hypothetical: a small lab releases an artefact-detection model on HuggingFace with code + Apache 2.0.

- Hypothesis tag match (artefacts): (+5)
- Annotation type (artefacts): (+3), localization: any (+2), stain: any (+1)
- Open code: ✅ (+2), weights: ✅ (+2), dataset: ⏳ promised (+0)
- License: Apache 2.0 (+1)
- Not SOTA but novel approach (+1)
- **Raw score: 17/19**
- Calibration: same lab seen 2× as `Relevant` → +1 (cap at +2, no change)
- Notion: Status=Review, Relevance=High
- Telegram: YES (both criteria)
