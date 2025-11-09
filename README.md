README — LlamaIndex + Gemini PDF Challenge Extraction (BMI5300_Project)
======================================================================

Purpose
-------
This repo/notebook extracts **challenge phrases** about *semantic interoperability* from a set of PDFs,
then aggregates and visualizes them. It serves as an appendix / reproducibility artifact for the poster.

High-Level Flow
---------------
1) **Read PDFs** from `./BMI5300_Project/` with **PyMuPDF** and wrap pages as LlamaIndex `Document`s.
2) **Chunk** pages using LlamaIndex `SentenceSplitter` (default: `chunk_size=1200`, `chunk_overlap=150`).
3) **Score & select** the top-N “challenge-like” chunks **per paper** with regex hints (`TOP_N = 10`).
4) **Extract challenge phrases** per chunk via **Gemini 2.5-Flash** (JSON-only schema).
5) **De-duplicate within paper**, consolidate to per-paper lists → `per_file_challenges.csv`.
6) **Canonicalize** phrases into buckets via regex → `challenge_counts_canonical.csv` & pivot table.
7) **Visualize** (word cloud over phrases; bar chart over canonical buckets, excluding “other”).

Environment / Setup
-------------------
Python 3.12.x (tested)
Install core libs:
```
!pip -q install "llama-index-core>=0.11.0" "pymupdf>=1.23.8" "pypdf>=4.0.0" pandas
!pip -q install google-generativeai>=0.7.0 pandas tqdm tenacity
!pip -q install wordcloud matplotlib
```
Set the Gemini API key:
- Preferred: `export GEMINI_API_KEY="..."` (or set in Colab secrets)
- The notebook also prompts if the env var is missing.

Input Data
----------
Place **6 PDFs** in `./BMI5300_Project/`, e.g.:
- `Data Interoperability in Context- The Importance of Open-Source Implementations When Choosing Open Standards.pdf`
- `Semantic Interoperability of Electronic Health Records- Systematic Review of Alternative Approaches for Enhancing Patient Information Availability.pdf`
- `Standards in sync five principles to achieve semantic interoperability for TRUE research for healthcare.pdf`
- `Verifying the Feasibility of Implementing Semantic Interoperability in Different Countries Based on the OpenEHR Approach- Comparative Study of Acute Coronary Syndrome Registries.pdf`
- `Why digital medicine depends on interoperability.pdf`
- `gavrilov-et-al-2019-healthcare-data-warehouse-system-supporting-cross-border-interoperability.pdf`

Key Notebook Steps (Code Outline)
---------------------------------
**1) Load PDFs & build LlamaIndex nodes**
- PyMuPDF extracts text (null chars removed).
- Store page text + metadata (`file_name`, `page_label`) into `Document`s.
- Split to nodes with `SentenceSplitter`.

**2) Persist chunks for audit/reuse**
- Write to `BMI5300_Project/itc_chunks.jsonl` (one JSON per chunk).
- Preview CSV (first 50): `BMI5300_Project/itc_chunks_preview.csv`.

**3) Chunk scoring & selection**
- Regex hints (`CHALLENGE_HINTS`) score paragraphs that likely mention limitations/challenges.
- Keep **top `TOP_N=10` chunks per paper** → `scored_selection`.

**4) Gemini extraction (JSON only)**
- Prompt enforces strict JSON: `{"challenges": ["...", ...]}`.
- Conservative rule: *no guessing; empty list if not present*.
- Retry/backoff with `tenacity`.
- Chunk-level outputs written to `BMI5300_Project/challenge_extractions_raw.jsonl`.

**5) Per-paper de-duplication**
- Within each paper, **exact same phrase** is kept once (`seen_local`).
- Output: `BMI5300_Project/per_file_challenges.csv` with columns:
  - `file_name`, `challenge_phrase`

**6) Frequency tables**
- **Phrase frequency** (across papers): `BMI5300_Project/challenge_counts.csv`
- **Canonical buckets** (regex map → bucket names) on each `challenge_phrase`:
  - Result counts: `BMI5300_Project/challenge_counts_canonical.csv`
  - Paper × bucket pivot: `BMI5300_Project/per_file_canonical_pivot.csv`

**7) Visualization**
- Word cloud over raw phrases (we often **exclude the 'other / uncategorized'** bucket).
- Horizontal bar chart for top canonical buckets (with larger y-axis font).

Outputs (Files)
---------------
- `BMI5300_Project/itc_chunks.jsonl` — all chunks in JSONL (audit)
- `BMI5300_Project/itc_chunks_preview.csv` — first 50 chunks preview
- `BMI5300_Project/challenge_extractions_raw.jsonl` — Gemini per-chunk extractions
- `BMI5300_Project/per_file_challenges.csv` — per-paper unique phrases
- `BMI5300_Project/challenge_counts.csv` — phrase-level global frequency
- `BMI5300_Project/challenge_counts_canonical.csv` — counts by bucket
- `BMI5300_Project/per_file_canonical_pivot.csv` — paper × bucket matrix
- Figures: generated inline in the notebook (word cloud + bar chart)

Important Concepts (How Counts Work)
------------------------------------
- **Within-paper de-dup only**: If the same phrase appears multiple times in a single paper, we count it once for that paper.
- **Across papers**: The same phrase mentioned in different papers contributes **multiple counts** (phrase-frequency).
- **Buckets can exceed paper count**: A single paper can contribute **multiple different phrases** that map to the **same bucket**, so bucket counts can be greater than the number of papers.
- If you prefer **paper-frequency** (each paper counts at most once per bucket), drop duplicates on `(file_name, canonical_challenge)` before counting.

Re-running with Different Settings
----------------------------------
- **Change number of chunks per paper**: set `TOP_N = 5/10/15`.
- **Exclude 'other' in plots**:
  ```python
  NO_OTHER = "other / uncategorized"
  pf_no_other = per_file_df[per_file_df["canonical_challenge"] != NO_OTHER]
  ```
- **Make y-axis labels larger** in the bar chart:
  ```python
  ax.tick_params(axis='y', labelsize=14)
  ```
- **Paper-frequency mode** (each paper contributes at most 1 per bucket):
  ```python
  unique_fb = per_file_df[per_file_df["canonical_challenge"] != "other / uncategorized"]\
                .drop_duplicates(["file_name", "canonical_challenge"])
  canon_counts_paper = (unique_fb["canonical_challenge"].value_counts().reset_index())
  ```

Canonical Buckets (Regex Map)
-----------------------------
We map phrases to these buckets (plus a fallback):
1) terminology mapping inconsistencies
2) data heterogeneity
3) data quality & provenance
4) profile version drift / upgrades
5) standards misalignment & governance
6) privacy, security & consent regulations
7) cross-border regulatory differences
8) multilingual terminology & localization
9) tooling, infrastructure & resource gaps
10) licensing & IP barriers
11) other / uncategorized (fallback when no pattern matches)

Troubleshooting
---------------
- **429 warnings** during Gemini calls: transient rate limiting; the notebook retries automatically. If it persists, lower `TOP_N` or add small `time.sleep(...)` between calls.
- **Empty/garbled PDF text**: some PDFs have non-extractable text; consider OCR or alternative parsers.
- **Large 'other / uncategorized'**: expand `CANON_MAP` with more patterns or normalize phrases more aggressively.

Reproducibility Notes
---------------------
- All intermediate artifacts are saved in `./BMI5300_Project/` for audit.
- Prompting is **JSON-only** to minimize parsing ambiguity.
- LlamaIndex chunking parameters and selection heuristics are documented above.

Credits
-------
- Text parsing: **PyMuPDF**
- Chunking: **LlamaIndex**
- LLM extraction: **Google Gemini 2.5-Flash**
- Viz: **matplotlib**, **wordcloud**
- Orchestration & retries: **tqdm**, **tenacity**

License / Data
--------------
- Ensure you have the right to process and share the PDFs. Outputs are derived metadata, not full-text redistribution.
