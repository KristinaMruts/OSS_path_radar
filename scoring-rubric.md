# OSS Radar Scoring Rubric — Reference

> **⚠️ The canonical rubric is INLINE in `SKILL.md`** (sections "SCORING RUBRIC" and "Hard rules"). That version is what the agent loads and follows. This file exists as a longer reference with worked examples — use it for human review, calibration discussions, and onboarding.

## Score range: 0–21 (raw, before hard-rule modifiers)

Total = sum of 11 components + calibration (±2).

| Bucket | Max | Components |
|---|---|---|
| Relevance | +11 | hypothesis_match (+5), annotation_match (+3), localization_match (+2), stain_match (+1) |
| Openness | +7 | openness.code (+2), openness.weights (+2), openness.dataset (+2), openness.license (+1) |
| Technical | +3 | technical.sota (+2), technical.novel_arch (+1) |
| Calibration | ±2 | signed adjustment from findings history |

**These component names are CANON.** The agent MUST use these exact names in the Notes scoring header. No inventing "base", "recency", "freshness", etc.

## Thresholds

| Raw score | Notion Status | Notion Relevance |
|---|---|---|
| ≥ 8 | `Review` | `High` |
| 4–7 | `New` | `Medium` |
| < 4 | (drop) | — |

## Hard rules

- **Rule 0 — Hypothesis-zero floor**: if `hypothesis_match=0` AND raw < 8 AND not full open stack → DROP
- **Rule 1 — Full open stack**: code+weights both open → minimum `Relevance=High`, Telegram=always
- **Rule 2 — Closed major lab**: closed code+weights from PAIGE/Owkin/Bioptimus/etc. → `Relevance` ≤ `Medium`, Telegram if raw≥8
- **Rule 3 — Code-promised**: paper says "code coming soon" → `Status=To Research`, marker `[CODE_PROMISED: <ETA>]`
- **Rule 4 — Dataset-only**: pure dataset → `Type=Dataset`, openness criteria become (open data +2, license +1)

## Applicability filter (post-scoring, applied at pipeline Step 4)

If Local Context Match = ❌ Not applicable:
- raw_score < 12 → DROP
- raw_score ≥ 12 → write with Status=Not Relevant

## Telegram trigger

Goes to digest if EITHER:
- raw score ≥ 8, OR
- open code AND open weights (full open stack)

---

## Anchor examples (calibration sanity checks)

These examples ground the rubric for human readers. The skill applies the rubric programmatically; these are NOT loaded by the agent.

### Example 1: Strong full-open release in hypothesis area

A foundation model from a tracked lab, code+weights, matches Tumor hypothesis.

- hypothesis_match=+5, annotation_match=+3, localization_match=+2, stain_match=+1
- openness.code=+2, openness.weights=+2, openness.dataset=+1 (partial), openness.license=+0 (CC-BY-NC)
- technical.sota=+2, technical.novel_arch=+1
- calibration=+1 (lab seen as `In Use` before)
- **Raw: 19/21** → Status=Review, Relevance=High, Telegram=YES (both raw≥8 AND full open stack)

### Example 2: Radiology paper that mentions "tumor"

Chest X-ray model for lung tumor detection.

- **DOMAIN FILTER applies → DROP at extractor stage.** Never reaches scoring.
- Rationale: out-of-domain (radiology, not pathology)

### Example 3: Pathology FM but requires ST input (STORM-like)

Spatial transcriptomics + histology FM.

- Passes domain filter (uses histology as primary modality)
- hypothesis_match=0 (no ST hypothesis configured), annotation_match=+3 (tumor), localization_match=+2, stain_match=+1
- openness: paper-only +0
- technical.novel_arch=+1
- **Raw: 7/21**
- Local Context Match: ❌ Not applicable (no ST pipeline in local stack)
- raw_score (7) < 12 → **APPLICABILITY FILTER drops it.** Not written to Notion.

### Example 4: Closed major-lab model, hypothesis match

Hypothetical PAIGE closed release, matches Tumor hypothesis.

- hypothesis_match=+5, annotation_match=+3, localization_match=+2, stain_match=+1
- openness: all closed +0
- technical.sota=+2, technical.novel_arch=+1
- **Raw: 14/21**
- Hard Rule 2: closed from major lab → Notion `Relevance=Medium` (downgrade)
- Local Context Match: ⚠️ Partial (methodology applicable, weights not) → write normally
- Telegram: YES (raw ≥ 8)

### Example 5: Niche full-open artefact detector

Small lab releases artefact-detection model on HuggingFace with code + Apache 2.0.

- hypothesis_match=+5 (matches Artefacts hypothesis)
- annotation_match=+3 (artefacts), localization_match=+2 (any), stain_match=+1 (any)
- openness.code=+2, openness.weights=+2, openness.dataset=+0 (promised), openness.license=+1 (Apache)
- technical.novel_arch=+1
- calibration=+1 (lab seen as `Relevant` before)
- **Raw: 18/21** → Status=Review, Relevance=High, Telegram=YES (both criteria)

### Example 6: Random arXiv preprint, no code, off-topic

- hypothesis_match=0, annotation_match=+3 (tumor), localization_match=0 (rare cancer not in scope), stain_match=+1 (H&E)
- openness: paper-only +0
- technical: no novelty +0
- **Raw: 4/21**
- calibration: arXiv seen 8× as `Not Relevant` → −1
- Adjusted: 3 → DROP (don't write)

---

## Calibration sources

The signed ±2 calibration is derived from preflight Step 1, querying the last 20 findings with terminal Status:

- Source × `Not Relevant` count ≥ 5 → source penalty (−1)
- Organization × `Not Relevant` count ≥ 5 → org penalty (−1)
- (Source, Domain) × `Relevant`/`In Use` count ≥ 3 → boost (+1)
- Architecture family matches an `In Use` entry → boost (+1)

The total adjustment is **capped at ±2**. Track which adjustments fired in the Notes `rationale:` line.

## Anti-patterns

- ❌ Don't invent scoring components (no "base", "recency", "freshness", "popularity")
- ❌ Don't merge components (no "openness=5" — list each as `openness.code: +2`, etc.)
- ❌ Don't skip a component you don't think applies — write `0` explicitly
- ❌ Don't downgrade out-of-domain to Low — DROP entirely (radiology never belongs in pathology findings DB)
- ❌ Don't ignore Local Context Match = Not applicable — apply the applicability filter
