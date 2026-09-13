# MPLADS AI Risk Intelligence & Decision-Support Layer

**SIH Problem Statement:** Build an AI-powered risk-detection and
decision-support system that examines MPLADS project, expenditure and
progress data and tells officials which works look suspicious/risky, why,
and what should be checked first.

This prototype is deliberately split into **two independent halves**, the
same pattern you used for your last SIH project:

```
┌─────────────────────────────┐        ┌──────────────────────────────┐
│   PART 1 — Google Colab      │  JSON  │   PART 2 — Streamlit App      │
│   colab/                     │ ─────► │   streamlit_app/               │
│   The "AI engine":           │ files  │   The decision-support UI:    │
│   loads data, engineers risk │        │   reads the JSON, shows       │
│   features, scores with a    │        │   officials a priority queue, │
│   rule-based + IsolationForest│       │   explanations, and a         │
│   hybrid model, explains why │        │   verify-first checklist      │
└─────────────────────────────┘        └──────────────────────────────┘
```

Nothing about this split is arbitrary: Colab is where you'll retrain /
re-tune the model; Streamlit is what officials actually click through. They
never need to touch each other's code — they're connected only by the JSON
files in `streamlit_app/data/`.

## Why this isn't "just another MPLADS dashboard"

The official eSAKSHI/MPLADS dashboard already tells you *what* happened
(allocations, expenditures, completion %). This prototype answers three
questions that dashboard doesn't:

1. **Which** of the ~130,000+ works should an official look at first?
2. **Why** does each one look unusual?
3. **What** exactly should be verified before drawing any conclusion?

Every score is a *"this pattern is unusual, a human should check it"*
signal — never a fraud verdict. That framing is enforced throughout the
code and the UI copy.

## Folder structure

```
project/
├── README.md                     <- you are here
├── colab/
│   └── MPLADS_AI_Risk_Engine_Colab.ipynb
└── streamlit_app/
    ├── app.py
    ├── requirements.txt
    └── data/                     <- JSON outputs (pre-generated so the app runs immediately)
        ├── national_overview.json
        ├── risk_scores_works.json
        ├── risk_scores_mps.json
        ├── aggregate_by_state.json
        ├── aggregate_by_worktype.json
        └── meta_summary.json
```

The `data/` folder already contains outputs generated from your real
uploaded extracts, so **the Streamlit app runs immediately** without you
needing to run Colab first. Re-run Colab whenever you want to refresh the
scores (new data, new weights, new model).

## Step-by-step: running Part 1 (Colab)

1. Open [Google Colab](https://colab.research.google.com), upload
   `colab/MPLADS_AI_Risk_Engine_Colab.ipynb`.
2. Run cells top to bottom.
3. Section 1 will prompt you to upload the 4 MPLADS CSV extracts (completed
   works, recommended works, expenditures, MP summary).
4. Section 10 exports a `mplads_ai_outputs.zip` and downloads it
   automatically.
5. Unzip it into `streamlit_app/data/`, replacing the demo files.

## Step-by-step: running Part 2 (Streamlit) locally

```bash
cd streamlit_app
pip install -r requirements.txt
streamlit run app.py
```

Open the URL Streamlit prints (usually `http://localhost:8501`).

## Connecting the real API later

You mentioned you'll provide an API to interconnect. Both halves already
have a clearly marked, single-function integration point so you only edit
one place in each file:

- **Colab:** Section 11, function `fetch_live_mplads_data()` — replace the
  `NotImplementedError` with real `requests.get(...)` calls, flip
  `USE_LIVE_API = True`.
- **Streamlit:** top of `app.py`, `API_BASE_URL` / `API_KEY` constants and
  the three `load_*` functions right below them — set `API_BASE_URL` and
  the functions will call the live API instead of reading local JSON.

Nothing else in either file needs to change — every other section works
purely off the DataFrames / JSON these functions return.

## How the risk model works (summary)

**Work-level risk** (individual projects):
| Signal | What it catches |
|---|---|
| Cost outlier | Cost far outside the normal range for similar work types nationwide |
| Round amount | Suspiciously exact round figures |
| Missing evidence | Completed but no site photo on record |
| Fragmentation | Many near-identical works by the same MP at an identical amount (possible splitting — or a legitimate bulk scheme; either way it's flagged for a one-line check) |

**MP / fund-level risk** (fund management patterns):
| Signal | What it catches |
|---|---|
| Utilisation–completion mismatch | Money recorded as spent, little physically completed |
| Vendor concentration | One vendor receiving a disproportionate share of an MP's spend |
| High pending balance | Large sums sanctioned but never paid out |

Each signal contributes to a transparent **rule-based score**; an
**Isolation Forest** (unsupervised anomaly detection) adds a second,
ML-driven layer on the same features. The two are blended
(60% rules / 40% ML) into one `final_risk_score` (0–100), banded into
Low / Medium / High. Every flagged item comes with plain-English reasons
and a verification checklist — see Section 8 of the notebook for the exact
templates.

## Honest limitations (worth saying out loud in your SIH pitch)

- This is an **unsupervised** anomaly detector — there is no labelled
  "confirmed fraud" dataset to validate against, so scores indicate
  *statistical unusualness*, not proven wrongdoing.
- The expenditure file doesn't carry a `Work ID`, so vendor-payment data
  can only be linked at the **MP level**, not to a specific work — this is
  a data-availability constraint, not a modelling choice.
- Work "category" in the source data is almost always `Normal/Others`, so
  work-type grouping uses a keyword classifier — good enough to compare
  costs sensibly, but not a substitute for the department's own
  classification.
