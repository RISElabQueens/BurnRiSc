# BurnRiSc Replication Package — Single-Developer Burnout Scoring

`Burnout_Replication_Package.ipynb` runs the core BurnRiSc pipeline end to
end for one developer: it extracts the behavioral and linguistic signals
described in the paper (mapped onto the Oldenburg Burnout Inventory's
Exhaustion and Disengagement dimensions), combines them into a monthly
Burnout Risk Score (BRS), and reports every sustained high-risk episode
along with the departure / activity-decline / role-withdrawal outcomes that
followed it. It merges two earlier working notebooks — `AllMetrics_3-4`
(signal extraction) and `Monthly_Evaluation2` (BRS scoring + outcome
evaluation) — into one notebook, scoped to run on a single developer at a
time rather than a full repository sweep.

This README covers that one notebook. It is not a README for the whole
BurnRiSc codebase (`DevRanking.py`, `DataMarking.py`, `Organization.py`, the
RQ3 activity-profiler script, etc. are separate, related scripts — see
[Scope of this notebook](#scope-of-this-notebook) below).


## What it does

Run top to bottom, the notebook:

1. **Setup** — installs the Python packages the rest of the notebook needs.
2. **Helpers** — mounts Google Drive and defines the shared utilities
   (loading a developer's raw activity into one text/timestamp table,
   buffering per-signal results, writing them to an Excel workbook).
3. **One cell per signal** — Arousal, Commits, Pronouns, Lexical Diversity,
   Lexical Features (exhaustion/disengagement semantic similarity),
   Off-Hours Activity, PR Throughput, Task Abandonment, and Sentiment. Each
   cell computes its signal at monthly granularity and appends it as a
   sheet in `{developer}_metrics.xlsx`.
4. **Burnout Calculation** — reads every signal sheet back out, combines
   them into monthly Exhaustion, Disengagement, and Combined Risk (BRS)
   scores using one of four weight profiles, detects every sustained
   high-risk onset, and measures what happened to the developer's activity
   in the months after each onset.
5. **Main Loop** — the only cell you interact with. It prompts for a
   developer's folder name, then runs steps 3–4 for that developer alone.

## Requirements

- Google Colab is the expected environment (the notebook mounts Google
  Drive directly); it will also run in a local Jupyter environment if you
  either mount an equivalent Drive path or edit `DEVPATH` near the top of
  the Helpers cell to point at your data.
- A GPU is recommended but not required — sentiment scoring and the
  sentence-embedding similarity step both fall back to CPU automatically
  (`DEVICE = torch.device("cuda" if torch.cuda.is_available() else "cpu")`),
  just slower.
- Python packages, installed by the Setup cell: `pandas`, `numpy`,
  `matplotlib`, `openpyxl`, `torch`, `transformers`, `sentence-transformers`.

## Data you need to provide

The raw per-developer activity data nor the two resources below are
bundled with this notebook — they are the project's own scraped GitHub data
and a fine-tuned model checkpoint, and are outside the scope of what a
notebook file can carry. 

The best fold for the sentiment analysis portion was also not able to be included. 
Best fold was created using methods provided by Biswas et al. (2020) in the paper "Achieving Reliable Sentiment Analysis in the Software Engineering Domain using BERT", with the BERT model being replaced with RoBERTa

NRC vad lexicon has been provided (Saif M. Mohammad,  https://saifmohammad.com/WebPages/nrc-vad.html)

### Per-developer activity files

The notebook expects one folder per developer under `DEVPATH`
(`.../CAPSTONE/data/DevProfiles/BurnoutData/{developer}/` by default),
containing:

| File | Required column(s) |
|---|---|
| `{developer}_commits.xlsx` | `date`, `message` |
| `{developer}_prs.xlsx` | `created_at`, `merged_at`, `state`, `title` |
| `{developer}_reviews.xlsx` | `submitted_at`, `pr_title` |
| `{developer}_issue_comments.xlsx` | `created_at`, `comment_body` |
| `{developer}_review_comments.xlsx` | `created_at`, `comment_body` |

This is exactly the folder layout `Organization.py` produces when it sorts
the six raw scrape folders (`devComs`, `devDiscussions`, `devIssueComments`,
`devPRs`, `devReviewComments`, `devReviews`) into one folder per developer —
run that first if you're starting from the raw scrape output.

### External resources

- **NRC-VAD Lexicon v2.1** (used by the Arousal signal) — download from
  [saifmohammad.com/WebPages/nrc-vad.html](https://saifmohammad.com/WebPages/nrc-vad.html)
  and place it at the `VAD_PATH` set in the Arousal cell
  (`.../CAPSTONE/data/graphs/Arousal/NRC-VAD/NRC-VAD-Lexicon-v2.1.txt`).
- **Fine-tuned RoBERTa sentiment checkpoint** — the Sentiment cell loads a
  checkpoint the authors fine-tuned for this project
  (`CHECKPOINT_DIR` in that cell). This is not a public/off-the-shelf
  checkpoint; without it, substitute your own sentiment classifier or skip
  that cell (see [Running a subset](#running-a-subset) below).

## How to run

1. Open the notebook in Google Colab.
2. Run the **Setup**, **Helpers**, every **signal cell**, and **Burnout
   Calculation** cells once, top to bottom, in order — they only define
   functions and load shared resources (the VAD lexicon, the sentiment
   model), they don't process a developer yet.
3. Run **Main Loop** and enter the developer's folder name (the same name
   as their subfolder under `DEVPATH`) at the prompt.
4. Repeat step 3 for additional developers — everything above it only
   needs to run once per session.

### Running a subset

Every signal cell buffers into the same in-memory cache and is independent
of the others, so if a resource above is unavailable (e.g. no sentiment
checkpoint), you can comment out that signal's call in the Main Loop cell
and run the rest — the Burnout Calculation cell simply skips any signal
whose sheet isn't present when it combines scores.

## What it produces

For the developer you enter:

- `{developer}_metrics.xlsx`, in their folder — one sheet per signal, plus
  `burnout_risk` (monthly BRS and an `at_risk` flag) and `burnout_episodes`
  (one row per detected high-risk onset, with the departure / activity
  decline / role withdrawal outcome measured for each).
- `graphs/{developer}_burnout_timeline.png` — the risk-and-activity
  timeline chart for that developer.

## Configuration

Set in the **Burnout Calculation** cell:

- `EVAL_PROFILE` — which of `WEIGHT_PROFILES` (`"default"`,
  `"exhaustion_heavy"`, `"disengagement_heavy"`, `"equal"`) to score with.
  Defaults to `"disengagement_heavy"`.
- `RISK_THRESHOLD` (default `0.305`) — the BRS level a month must reach to
  count toward a risk onset.
- `ONSET_COOLDOWN_MONTHS`, `DEPARTURE_SILENCE_MONTHS`, `DECLINE_THRESHOLD`,
  `LOOKBACK_MONTHS` / `LOOKAHEAD_MONTHS`, `MIN_BASELINE_ACTIVITY` — control
  onset detection and how the three outcomes are measured; see the
  docstrings on `find_risk_onsets` and `measure_outcomes` for exactly how
  each is used.

## Scope of this notebook

This notebook deliberately does **one developer at a time**. It does not
reproduce:

- The cross-repository, cross-developer ranking/comparison used to
  validate BRS against the ten confirmed cases in the paper (that lives in
  `DevRanking.py` and the fuller `Monthly_Evaluation2`), or the RQ3
  activity-confound analysis (a separate script) — both need a population
  of developers to compare against.
- `DataMarking.py`'s search for additional "soft" burnout candidates
  (developers with an unexplained activity collapse but no public
  disclosure).

`CONFIRMED_BURNOUT_DEVS` / `GROUND_TRUTH_EVENTS` in the Burnout Calculation
cell are carried over from the paper's ten-developer validation set purely
as an optional label on the summary output — they don't affect scoring, and
running this notebook on a developer not in that list works identically.

## Citation

If you use this pipeline, please cite the BurnRiSc paper (details to be
added on publication).
