# Thesis Review Notes — Gaps Between the .docx and the Code

Read against `Διπλωματική Μούλιαρας Θωμάς.docx` (476 paragraphs) and the current `rossmann/` codebase. These are observations and suggested fixes only — nothing has been changed in the .docx itself. Paragraph indices below refer to the document's internal paragraph order (0-based, as extracted via `python-docx`), for easy relocation.

## 1. Content gap: code implements 5 methods, thesis text covers 3

Chapter 4.3 (§4.3.1–4.3.3) writes up Moving Average, SES, and Linear Regression only. The codebase additionally has a fully working **SARIMAX** implementation (`forecasting.forecast_sarimax`) and a fully working **XGBoost** pipeline (`ml.py`, with GPU support, model persistence, a global all-stores model, and a dedicated 30-day backtest with an Excel dashboard). `readme.md` already advertises these ("SARIMAX", "XGBoost", "Backtesting" are listed under "What it does").

This is the single biggest opportunity to strengthen the thesis without writing new code — the XGBoost/backtest module in particular is the most developed part of the codebase (own file, GPU detection, persisted model, dedicated evaluation) and currently gets zero mentions in the analysis chapter. Suggested treatment:
- Add §4.3.4 (SARIMAX) and §4.3.5 (XGBoost), mirroring the structure already used for MA/SES/Regression (theory → application procedure → worked example → interpretation).
- Extend §4.4's comparison to all five methods with one shared numeric table (see `03_methods_comparison.md`).
- The existing "Μελλοντικές επεκτάσεις" paragraph (index ~[435]) explicitly lists "μηχανικής μάθησης" and tree-boosting algorithms as *future work* — but XGBoost is already implemented and runs. That paragraph should be corrected so it doesn't describe already-completed work as a future extension.

## 2. Duplicate/mislabeled heading: §1.4 and §1.5 share a title

- Paragraph [53] — Heading 2 — `"1.4. Σημαντικότητα μελέτης"` — content is about why the study matters (correct for that title).
- Paragraph [57] — Heading 2 — `"1.5. Σημαντικότητα μελέτης"` (**same title**) — but paragraph [58] under it is actually the thesis's chapter-by-chapter outline ("Η εργασία οργανώνεται σε έξι κεφάλαια...") — i.e. this is the **"Διάρθρωση της εργασίας" (structure of the thesis)** section, mislabeled with a copy-pasted heading.

**Fix:** rename the heading at [57] to something like `"1.5. Διάρθρωση της εργασίας"`.

## 3. Duplicate paragraph in §4.2

Paragraphs [176] and [181] are word-for-word identical: *"Τα αποτελέσματα ανέδειξαν έντονη διαφοροποίηση στις πωλήσεις μεταξύ καθημερινών και Σαββατοκύριακων ημερών, με τις υψηλότερες τιμές να καταγράφονται συνήθως στις αρχές της εβδομάδας."* One of the two (likely surrounding a missing chart/table that didn't extract as text) should be removed.

## 4. Citation year mismatch: Μόδης

In-text citation at paragraph [39]: `(Μόδης Θ, 2012)`. Bibliography entry at paragraph [456]: `Μόδης, Θ. (2002). Προβλέψεις.` — the year doesn't match (2012 vs 2002). One of the two is a typo; needs reconciling against the actual publication date of the source.

## 5. Bibliography contains stray in-text-citation fragments, not full references

Paragraphs [445]–[447], sitting inside the "Βιβλιογραφία" section, read as leftover fragments rather than formatted references:
- `[445]` `"Beamon 1998 Lambert et al. 1998· Mentzer et al. 2001)."`
- `[446]` `"Διάγραμμα supply chain (https://www.edrawmax.com/article/what-is-supply-chain-diagram.html)"`
- `[447]` `"Cooper & Ellram 1993· Cooper et al. 1997· Mentzer et al. 2001)."`

These look like an in-text parenthetical citation (and a figure-source URL) that got pasted into the bibliography list instead of staying inline in the body text near a supply-chain figure/diagram. Each author cited here (Beamon, Lambert, Cooper, Ellram, Mentzer et al.) needs either a proper full reference entry, or — if they were only meant as an inline citation — removal from the bibliography and reinsertion in the body where the claim they support actually appears. The edrawmax URL is a figure-source citation and belongs in a figure caption, not the bibliography.

## 6. "Decision Support System (DSS)" framing vs. actual deliverable

The abstract, introduction, and conclusion all describe the deliverable as a **"Decision Support System (DSS) which brings analysis and forecasting into a unified environment"** (e.g. paragraphs [11], [18], [26], [41], [434]). The actual code is a command-line pipeline: `run.py --steps sarimax ols_regression ...`, printing to stdout and writing files to `outputs/`. There is no unified interface — a user runs the CLI, or imports functions individually in a script/REPL.

This isn't wrong exactly (a CLI pipeline is a legitimate lightweight DSS), but as written it may read as promising more than what's delivered (dashboards, an interactive UI, a single "environment"). Two ways to resolve it, either is defensible:
- **Option A (cheaper):** soften the DSS language to describe it precisely — "a Python-based command-line analysis and forecasting pipeline" — and keep "DSS" only as a functional description of what the outputs (Excel workbooks with embedded charts) are *used for* by a decision-maker, not as a claim about the software's UI.
- **Option B (more work, stronger claim):** build a minimal actual front end (even a single argparse-driven interactive menu, or a small Streamlit app) that ties `run.py`'s steps together in one session — this would make the existing DSS language literally accurate.

## 7. `readme.md` promises CLI steps that `run.py` doesn't register

`readme.md`'s "Run" section shows `python run.py --steps sarimax ols_regression xgboost_train backtest` and separately documents `xgboost_train`, `xgboost_single_store`, `xgboost_all_stores`, and `backtest` as available steps. In the current `run.py`, the `STEPS` dict only registers **`moving_average`, `ses`, `sales_by_day`, `sarimax`** — none of `regression.py`'s or `ml.py`'s functions are wired in. Running the readme's example command as written would fail with "Unknown step(s)".

This is a documentation/code sync issue independent of the thesis itself, but worth fixing before the thesis is finalized/submitted with the repo, since a grader or reader following the readme's own instructions would hit an error immediately. Either:
- register `ols_regression`, `xgboost_train`, `xgboost_single_store`, `xgboost_all_stores`, `backtest` as `@step(...)`-decorated wrappers in `run.py` (straightforward — same pattern as the existing four), or
- trim `readme.md`'s examples down to the steps that actually exist today.

## 8. Placeholder tables/figures still in the document

Several spots reference tables/figures by generic placeholder names rather than actual content: `"Πίνακας Χ"` (multiple occurrences, e.g. [163], [190], [229], [271]), `"Σχήμα 4.x"` ([264]), `"Πίνακας 4.x"` ([277], [313]). These need to be replaced with real numbered tables/figures (actual screenshots or generated tables from the code's Excel/CSV outputs) before submission — currently a reader can't see the referenced Store 1 moving-average table, the feature table, or the comparison table.

## 9. `preprocess_data` output isn't consistently reused

Not a thesis-text issue, but relevant if the thesis's described pipeline (§4.2: "Date converted to datetime format, then to dd/mm/yyyy for readability") is meant to describe what every downstream step does: `data.preprocess_data` turns `Date` back into a *string* (`dd/mm/yyyy`) at the end, so `forecasting.moving_average` and other functions that receive its output re-parse `Date` with their own `pd.to_datetime` calls rather than continuing from a real datetime column. Functions like `ses_forecast_store_stats`, `forecast_sarimax`, and everything in `ml.py`/`analytics.py` don't call `preprocess_data` at all — they re-read `train.csv` from disk independently. If the thesis's methodology section is meant to describe one linear pipeline all methods pass through, that's not quite what the code does; each method effectively does its own lightweight load-and-clean. Worth a one-line clarification in §4.2 that preprocessing is applied per-method rather than globally upstream of all five.
