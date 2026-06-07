# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

UOC course **Mineria de dades** — *PRA2: Projecte de mineria de dades* (Catalan).
Continuation of PRA1: the cleaned dataset from that practice is reused here to fit unsupervised and supervised models. The deliverable is a single rendered R Markdown report. Deadline noted in the Rmd: **10/06/2026**.

## Build / Render

The repo is a single-file R Markdown project managed with `renv` (R 4.3.3 pinned in `renv.lock`).

- Restore packages (first time / after pulling): `R -e 'renv::restore()'`
- Render the report: `R -e 'rmarkdown::render("05.584-PRA2-bassegoda.Rmd")'`
- After installing a new package inside the project, snapshot it: `R -e 'renv::snapshot()'`
- The Rmd's YAML header includes `05.584-PAC-header.html` via `includes: in_header:` — keep that file alongside the Rmd or rendering fails.

`.gitignore` excludes `*.html` and `*.pdf`, so rendered output is intentionally not committed.

### Environment gotchas on this machine (Pop!_OS 24.04 = Ubuntu noble)

- **Pandoc:** command-line R can't find pandoc, so `rmarkdown::render` fails with a "pandoc … required" error. RStudio's Knit button works, but from the terminal export the bundled pandoc first:
  `export RSTUDIO_PANDOC="/usr/lib/rstudio/resources/app/bin/quarto/bin/tools/x86_64"`
- **Package install:** CMake isn't on PATH, so source-compiling `nloptr` (dep of `caret`→`lme4`) fails. Install **pre-compiled binaries** from Posit Public Package Manager — no compile, no sudo:
  `R -e 'options(repos=c(PPM="https://packagemanager.posit.co/cran/__linux__/noble/latest")); renv::install(c(...))'`
  `renv.lock` already records this PPM repo, so a plain `renv::restore()` pulls binaries. (renv rolls back the *whole* transaction on any single compile failure.)

## Data

`clinical_clean.csv` (5927 rows × 78 columns) is the prepared dataset produced in PRA1 and is the input for every exercise. Columns mix:
- Identifiers / timestamps (`caseid`, `casestart`, `opstart`, ...)
- Demographics (`age`, `sex`, `height`, `weight`, `bmi`, `asa`)
- Preoperative labs (`preop_hb`, `preop_plt`, `preop_na`, ...)
- Intraoperative measures (`intraop_ebl`, `intraop_uo`, `intraop_rbc`, ...)
- Outcomes (`icu_admission`, `death_inhosp`, `los_hosp_days`, `op_duration_min`)

When applying k-means / k-medians / DBSCAN, the enunciat requires using **quantitative or binary** variables only, and the dataset has variables on very different scales — normalization is explicitly called out in the Rmd as recommended.

## Structure of the Rmd (what each section is expected to contain)

`05.584-PRA2-bassegoda.Rmd` is pre-scaffolded under `# RESPOSTES` with empty section headings per exercise. Each exercise has its own grading weight (in `avaluacio.txt`) and concrete sub-requirements — read the heading + sub-headings inside the Rmd before writing code into a section, not just the exercise number.

Exercise scope at a glance:
1. (20%) k-means on raw and normalized data, justify k.
2. (10%) k-medians (and optionally PAM / *around medoids*) at the k chosen in 1; cross-tabulate vs. ex.1.
3. (10%) k-means with a **non-Euclidean** distance; compare to ex.1 and ex.2.
4. (10%) DBSCAN and OPTICS, sweep `eps` / `minPts`; compare to ex.1 and ex.2.
5. (5%) Train/test split with justified proportions.
6. (20%) Decision tree with and without pruning; rules in text + plot; confusion matrix and derived metrics.
7. (10%) A second supervised model (different tree criterion or another algorithm); compare to ex.6.
8. (10%) Limitations, risks, and final conclusions (≥300 words).
9. (5%) AI-interaction section — at least 3 documented consultations, classified into the five categories described at the top of the Rmd (Metodologia / Programació / Diagnosi / Interpretació / Extensió) using the 4-step template (Pregunta / Resposta / Decisió / Reflexió crítica).

`avaluacio.txt` is the grading rubric in plain text — useful when verifying a section meets every required bullet (e.g. "interpret centres, group sizes, dispersion" appears in exercises 1-3 and is easy to forget).

## Conventions

- Output language is **Catalan** — narrative prose, headings, and chunk comments should match.
- The Rproj sets 2-space indent, UTF-8 — match this in R code chunks.
- All mining libraries are now installed and snapshotted in `renv.lock`: `cluster`, `dbscan`, `rpart`, `rpart.plot`, `caret`, `randomForest`, `factoextra`, `C50` (loaded in the `llibreries` chunk). After installing anything new, run `renv::snapshot()`.

## Progress and decisions (state of the report)

**Done so far** (all under `# RESPOSTES`, in Catalan):
- `## Descripció del joc de dades` — VitalDB variable glossary + a subsection for the 4 PRA1-derived columns.
- `## Preparació de les dades (comuna als exercicis 1–4)` — shared feature matrix.
- **Exercise 1 (k-means)** — fully written and rendering. Silhouette now on the FULL dataset (k=2 sil 0.37 vs 0.19 at k=3; sizes 685/4564).
- **Exercise 2 (k-medians)** — `flexclust::kcca(family="kmedians")` (median centroid, Manhattan), k=2. Sizes 548/4701, silhouette 0.36, **92.2% concordance with ex1**. Optional PAM `around medoids` included as a contrast (diverges, sil 0.07). Robustness shown via creatinine centre 2.1 (median) vs 2.9 (ex1 mean).
- **Exercise 3 (non-Euclidean k-means)** — cosine/`angle` family of `kcca`, k=2. Sizes 2405/2844 (balanced ~46/54), silhouette 0.16. Splits on a DIFFERENT axis (age/HTN/DM chronic-comorbidity) vs the acute-severity axis of ex1/ex2; 97% of ex1 high-risk nest inside cosine cluster 1.
- **Exercise 4 (DBSCAN + OPTICS)** — `dbscan`/`optics`, minPts=34 (≈dim+1), eps=4 (from kNNdistplot knee). Finds 1 dense cluster (4392) + 16.3% noise; no (eps,minPts) yields 2 balanced clusters (curse of dimensionality). Key finding: noise = high-risk patients (87.6% of ex1 Grup1 flagged as noise vs 5.6% of standard) → the dispersed high-risk group is NOT a dense region, so density methods reframe it as outliers. OPTICS `extractDBSCAN(eps_cl=4)` reproduces DBSCAN exactly. Silhouette N/A (single cluster); quality = noise ratio. Three figures: kNNdistplot, reachability plot, PCA scatter (dense vs noise).
- **Exercise 5 (train/test split)** — start of supervised part. Built a fresh supervised dataset `sup` (re-reads CSV; the ex1 `dades` has emop/htn/dm recoded to 0/1): **42 predictors** (32 numeric + 10 categorical: sex/emop/department/optype/approach/ane_type/position/preop_htn/preop_dm/preop_pft), all known before the ICU decision. Excludes leakage (`icu_days`, `los_hosp_days`, `death_inhosp`), IDs/timestamps, high-cardinality text (`dx`,`opname`), and >20%-missing cols. Trees handle NA via surrogate splits → keep all 5926 rows (no complete-case filter). **Stratified 80/20** via `caret::createDataPartition(seed=42)`: train 4742 (859 pos) / test 1184 (214 pos), 18.1% positive in both. Objects `sup`/`train`/`test` shared by ex6–7.

- **Exercise 6 (decision tree, CART)** — `rpart` (Gini), predict `icu_admission` on `train`, evaluate on `test`. Grow full tree (cp=0, minsplit=20, minbucket=7) = 96 leaves (overfit), prune via **1-SE rule** computed from cptable → **13 leaves**. Pruned generalizes BETTER on test: acc 0.872 vs 0.859, precision 0.68 vs 0.61, kappa 0.531 vs 0.519 (sens slightly lower 0.547 vs 0.598, F1 tied 0.606). Rules text (`rpart.rules`) + plot (`rpart.plot`). Top vars: optype, department, op/ane duration, position, preop_alb. Rules clinically coherent (surgery type/duration/department + frailty). Added a cost-sensitive refinement (loss matrix FN=3 → sens 0.76) since false negatives are clinically worse. **Accuracy misleads at 18% prevalence** (all-No = 82%), so report kappa + sens/spec.

- **Exercise 7 (second supervised model)** — BOTH C5.0 and Random Forest vs ex6 tree. C5.0 (entropy criterion, `trials=10` boosting); RF (`ntree=500`, ensemble). Test metrics: C5.0 tree acc 0.872/kappa 0.552, C5.0 boosting acc 0.894/kappa 0.612, RF acc 0.905/sens 0.621/spec 0.967/prec 0.806/F1 0.702/kappa 0.646 (OOB 0.109). Story = predictive-power vs interpretability tradeoff (CART pruned = readable but weakest; RF = black box but best). All models agree on top vars (optype, durations, department, age) → signal is robust. **Two gotchas:** (1) C5.0 crashes on accented factor level "Sí" (`gsub`/`makeDataFile` wide-string error even in UTF-8 locale) → use ASCII copies ("Sí"→"Si") just for C5.0; (2) randomForest can't handle NA → impute with `na.roughfix` before fit/predict.

- **Exercise 8 (limitations, risks, conclusions)** — prose only (no code). Covers: data limitations (single-center VitalDB → external validity; cross-sectional → associations not causal; high missingness; clustering lost categorical info & is gradient-like; supervised class imbalance + borderline intraop-leakage; RF imputation assumptions), model risks (causality confusion, distribution shift, false negatives ~45% missed, RF black-box/automation bias, fairness), and synthesis conclusions. Conclusions paragraph = 405 words (>300 req met standalone; risks+conclusions = 453).

- **Exercise 9 / "Interacció amb la IA" (5%)** — 7 consultations documented with the 4-step template, weighted toward **Programació (3 entries)** per user request: (1) k-medians via flexclust vs pam, (2) C5.0 accented-level encoding fix, (3) randomForest NA imputation; plus 1 each: Metodologia (normalize + choose k), Diagnosi (DBSCAN 1-cluster = curse of dimensionality), Interpretació (accuracy paradox — framed as JOINT work, not AI-given), Extensió (loss matrix + ensembles + PCA viz). User asked NOT to mention the renv.lock incident (too technical).

**STATUS: report COMPLETE (exercises 1–9 all written, rendering end-to-end).** Remaining = polish/review only if requested. Not yet committed to git (user hasn't asked).

`flexclust` (+ `modeltools`) added to the lockfile for ex2/ex3; `yaml` installed so `renv::snapshot` can parse the Rmd. The whole report renders end-to-end (verified via terminal render with the RSTUDIO_PANDOC export).

**Locked-in modeling decisions** (keep consistent across exercises):
- **Supervised target (ex 5–7): `icu_admission`** (~18% positive). Rejected `death_inhosp` (only 0.8% positive — too imbalanced).
- **Clustering feature set (ex 1–4):** quantitative + binary only; No/Sí flags (`emop`, `preop_htn`, `preop_dm`) recoded to 0/1; **exclude all outcomes** (`icu_admission`, `death_inhosp`, `los_hosp_days`, `icu_days`, `op_duration_min`, `ane_duration_min`), IDs and timestamps. Drop columns with >20% missing → **33 variables, 5249 complete cases**. The shared matrices are `X` (raw) and `Xs` (z-score scaled).
- **Ex 1 outcome:** use **normalized** data, **k = 2**. Silhouette is computed on the **full dataset** (not a subsample): **0.37 at k=2 vs 0.19 at k=3** (an earlier 1500-case subsample overestimated it at ~0.60). Raw-data silhouette looks higher (0.73) but is a scale artifact dominated by `intraop_crystalloid`. Clusters = high-risk/comorbid patients (Grup 1, n≈685: ASA~2.54, creatinine~2.90, emop~0.39) vs standard-risk (Grup 2, n≈4564). Cluster label (1 vs 2) is RNG-order dependent: `km_std` is fit right after `set.seed(42); km_raw`, which stably makes **Grup 1 = high-risk**. This is the **reference partition** for cross-tabulating ex 2–4. Build ex 2 (k-medians via `pam`/`clara`) on the same `Xs` at k=2.
