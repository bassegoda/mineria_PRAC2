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
- **Exercise 1 (k-means)** — fully written and rendering.

**Next:** Exercise 2 (k-medians). Exercises 2–9 are still empty scaffolding.

**Locked-in modeling decisions** (keep consistent across exercises):
- **Supervised target (ex 5–7): `icu_admission`** (~18% positive). Rejected `death_inhosp` (only 0.8% positive — too imbalanced).
- **Clustering feature set (ex 1–4):** quantitative + binary only; No/Sí flags (`emop`, `preop_htn`, `preop_dm`) recoded to 0/1; **exclude all outcomes** (`icu_admission`, `death_inhosp`, `los_hosp_days`, `icu_days`, `op_duration_min`, `ane_duration_min`), IDs and timestamps. Drop columns with >20% missing → **33 variables, 5249 complete cases**. The shared matrices are `X` (raw) and `Xs` (z-score scaled).
- **Ex 1 outcome:** use **normalized** data, **k = 2** (silhouette 0.60 vs 0.19 at k=3). Clusters = high-risk/comorbid patients (n≈676) vs standard-risk (n≈4573). This is the **reference partition** for cross-tabulating ex 2–4. Build ex 2 (k-medians via `pam`/`clara`) on the same `Xs` at k=2.
