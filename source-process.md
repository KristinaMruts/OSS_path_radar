# OSS Radar Source Process

8 source categories. Each gets its own sub-agent in Step 2 of the pipeline.

**Default scan window:** last 4 days (matches twice-weekly cadence — Mon scans Thu–Sun, Thu scans Mon–Thu). Configurable via `$SCAN_WINDOW_DAYS`.

**Backfill mode:** when prompt explicitly says "backfill <N> months", widen the window to last N months. Use for first-time runs or recovering from extended downtime.

For each sub-agent, pass: `active_hypotheses tags`, `seen_urls`, `seen_names`, and the relevant section of this file below.

---

## ⚠️ DOMAIN FILTER — apply BEFORE returning any candidate

**Every extractor sub-agent MUST apply the domain filter from SKILL.md before returning candidates.**

Quick reference (full spec in SKILL.md "Domain filter" section):

**IN SCOPE** (score normally): histology, cytology, IHC, special stains on tissue, FNA/Pap smears, spatial omics + histology (when histology is primary input), WSI analysis, tile classification, cell/mitosis detection, tumor segmentation, stain normalization.

**OUT OF SCOPE** (DROP, do not return): radiology (X-ray/CT/MRI/mammography imaging/ultrasound/PET), dermatology clinical photos, ophthalmology (retinal/OCT), cardiology (ECG/echo), endoscopy video, general medical NLP, genomics-only, surgical/robotic video.

**Ambiguous** → DROP with rationale `out-of-domain: ambiguous`.

Each sub-agent reports counts at the end of its return:
- `candidates_found`: total before filtering
- `filtered_out_of_scope`: dropped by domain filter
- `filtered_already_seen`: dropped by dedup

---

## Category A — HuggingFace

### Watched organizations

**The list of organizations to scan is NOT in this file.** It lives in the Notion DB `$NOTION_WATCHED_SOURCES_DS_ID`, in rows where `Type = hf_org` and `Status = active`.

The main session fetches these rows in preflight and passes them to this sub-agent as `hf_orgs = [{name, url}, ...]`. To add or remove a watched organization, edit the Notion DB — no code or skill change needed.

WebFetch each `url` from the passed list; check "Models" tab + "Datasets" tab for recent uploads.

If the passed list is empty → skip the org-scan part of this category, only run the general HuggingFace search below.

### General HuggingFace search

```
WebSearch: "site:huggingface.co/models pathology histology" published in last week
WebSearch: "site:huggingface.co foundation model histopathology" recent
```

### Attention filters
- `pipeline_tag` ∈ {`image-segmentation`, `image-classification`, `image-feature-extraction`, `image-to-text`}
- Tags include: `medical`, `pathology`, `histopathology`, `wsi`, `cytology`
- Model card has a real "Model description" section (not auto-generated)

### Extractor output per model
```json
{
  "name": "UNI-2",
  "organization": "MahmoodLab",
  "source": "HuggingFace",
  "url": "https://huggingface.co/MahmoodLab/UNI-2",
  "hf_url": "https://huggingface.co/MahmoodLab/UNI-2",
  "github_url": "https://github.com/mahmoodlab/UNI",
  "weights_url": "https://huggingface.co/MahmoodLab/UNI-2/resolve/main/pytorch_model.bin",
  "dataset_url": null,
  "license": "CC-BY-NC-4.0",
  "summary_raw": "Vision transformer pretrained on 100k WSI from 20 organs...",
  "parameters": "600M params, ViT-Large architecture",
  "datasets_mentioned": "MGH-private (100k WSI)",
  "openness_signals": {"code": "✅", "weights": "✅", "dataset": "❌"}
}
```

## Category B — arXiv

### Searches
```
WebSearch: site:arxiv.org cs.CV histopathology OR "computational pathology" recent
WebSearch: site:arxiv.org "foundation model" pathology last week
WebSearch: site:arxiv.org "vision-language" pathology recent
```

### Filters
- Categories: cs.CV, eess.IV, cs.LG
- Submitted within scan window
- Ignore re-submissions / v2/v3 of papers already in seen-set (check by paper title, not URL)

### Anti-patterns
- Don't fetch arXiv PDFs (huge — 5MB+). Read the abstract page only.

### Look for `code coming soon` signals in abstract
- Phrases: "code will be released", "weights to be made available", "upon publication"
- If found → `openness_signals.code = "⏳ promised"`, capture ETA if mentioned (else `"unknown"`)

## Category C — GitHub

### Searches
```
WebSearch: github.com topic:computational-pathology updated:>YYYY-MM-DD
WebSearch: github.com "digital pathology" OR "histology AI" stars:>10 created:>YYYY-MM-DD
WebSearch: github.com topic:histopathology pushed:>7d
```

(Compute the `YYYY-MM-DD` cutoffs from scan window.)

### Filters
- Repo has README (not empty)
- License is not GPL-only (preferably MIT / Apache 2.0 / BSD / CC-BY)
- Real code (not just paper-link or `awesome-list`)
- ≥10 stars OR comes from a known lab

### Extractor: read repo's README first 500 lines + LICENSE file

## Category D — Papers with Code + Conference proceedings

### Searches
```
WebSearch: paperswithcode.com pathology benchmark new SOTA recent
WebSearch: "MICCAI" OR "CVPR" pathology accepted (current year)
WebSearch: "ICLR" OR "NeurIPS" pathology (current year)
```

### Look for
- SOTA submissions on Patho-Bench, PathBench, EVA, HEST benchmarks
- Conference accepted-papers lists (often have links to code)

## Category E — Company blogs

### Watched URLs

**The list of company blogs to scan is NOT in this file.** It lives in the Notion DB `$NOTION_WATCHED_SOURCES_DS_ID`, in rows where `Type = company_blog` and `Status = active`.

The main session fetches these rows in preflight and passes them to this sub-agent as `company_blogs = [{name, url}, ...]`. To add or remove a watched blog, edit the Notion DB — no code or skill change needed.

WebFetch each `url` from the passed list; look at posts published within the scan window.

If the passed list is empty → skip this category entirely.

### Extractor: post title + date + summary

If a post mentions a GitHub/HF link, follow it for openness signals (don't classify based on blog text alone — verify open code/weights).

## Category F — Aggregators + Benchmarks

### Watched URLs

- https://grand-challenge.org/challenges/ — new competitions (filter by recent)
- https://github.com/DearCaat/Awesome-Computational-Pathology-Papers — curated paper list (check recent commits)
- https://birkhoffkiki.github.io/PathBench/ — FM leaderboard updates

### Filter for findings in scan window

New leaderboard entries = high signal.

## Category G — High-impact journals + news (proxy approach)

### Direct WebFetch FAILS on these (403 / paywall)
- ❌ cell.com
- ❌ nature.com (mostly)
- ❌ nejm.org

### Use proxies instead
```
WebSearch: site:cell.com pathology OR histology OR "tumor microenvironment" foundation model recent
WebSearch: site:nature.com pathology AI model recent
WebSearch: nejm.org OR "lancet digital health" pathology AI recent
WebSearch: site:news-medical.net pathology AI breakthrough recent
WebSearch: site:geekwire.com pathology AI Microsoft OR Google OR Owkin recent
WebSearch: scholar.google.com "pathology" "foundation model" recent
```

From news/PR articles extract: model name, organization, links to GitHub/HF (often in an "Open Source Availability" block). Then follow up via HF/GitHub directly to verify openness signals.

## Category H — Microsoft Research Blog + Azure Foundry Labs + Google Health

### Watched URLs
- https://www.microsoft.com/en-us/research/blog/?topic=ai-machine-learning
- https://labs.ai.azure.com (Microsoft Foundry Labs — GigaTIME, Path Foundation, etc.)
- https://blog.research.google/search/label/health

### Filter posts in scan window by keywords
pathology, histology, tumor, digital pathology, WSI.

## Common extractor anti-patterns

- ❌ Don't keep raw HTML in sub-agent context after parsing — extract to JSON, drop HTML
- ❌ Don't guess openness signals — verify by following links (HF page exists? GitHub repo has code?)
- ❌ Don't include findings already in seen-set (check `seen_urls` AND `seen_names`)
- ❌ Don't return long descriptions — `summary_raw` is one paragraph max
- ❌ Don't fetch paper PDFs — abstracts only (PDFs are 5–30 MB each)
- ❌ Don't generate `summary_raw` from URL alone — actually fetch and read the page

## Output contract for each sub-agent

Each sub-agent returns a single JSON array. Empty array `[]` is valid (nothing new in this category).

```json
[
  {
    "name": "...",
    "source": "HuggingFace|GitHub|ArXiv|Company Blog|Conference|PapersWithCode",
    "organization": "MahmoodLab|...|Other",
    "url": "https://...",
    "hf_url": "https://huggingface.co/... or null",
    "github_url": "https://github.com/... or null",
    "weights_url": "direct link or null",
    "dataset_url": "https://... or null",
    "license": "MIT|Apache 2.0|CC-BY-NC|...|unknown",
    "summary_raw": "one paragraph from source",
    "parameters": "extracted or 'unknown'",
    "datasets_mentioned": "extracted or 'unknown'",
    "openness_signals": {
      "code": "✅|❌|⏳",
      "weights": "✅|❌|⏳",
      "dataset": "✅|❌|⏳"
    },
    "date_published": "ISO YYYY-MM-DD or null"
  }
]
```

Target ≤800 tokens per sub-agent response. If a category has 10+ new findings, return only the top 5 by openness-signal richness — better to lose 5 weak findings than blow the context budget.
