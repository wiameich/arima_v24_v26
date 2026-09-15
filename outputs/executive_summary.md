# Executive Summary — ARIMA Data Quality & Vintage Comparison

We checked ARIMA's synthetic population data itself — separate from our own
fusion pipeline — by comparing the 2024 and 2026 versions of the data and
testing whether relationships that should obviously exist in the data
actually do. Four findings matter most, in order of importance. **This is a
short pointer to two detailed reports, not a replacement for them.**

## 1. A row's "ID" number does not identify the same person across different ARIMA data pulls — and this showed up twice, independently

Every time ARIMA data is exported, each row is just numbered 1, 2, 3...
starting fresh. If you assume "ID 500" in one ARIMA file is the same
person as "ID 500" in another file, you'd be wrong — we proved this by
checking whether people with the "same" ID had matching basic details
(location, age, gender), and they didn't. This isn't a one-off: we found
it comparing the 2024 vs. 2026 population data, **and separately, again,
inside our own live Intact fusion pipeline's data** — two unrelated
places turning up the same problem, which is why we treat it as a
structural property of ARIMA, not a fluke.

- **Full detail:** `outputs/part1_vintage_comparison.md`, Part 1.2 ("ID
  continuity"); `outputs/part2_data_quality_scorecard.md`, Task 3 Part A
  (checks 3.1–3.3).
- **Figures:** none — this finding is evidenced by ID-overlap and
  attribute-matching tables in the reports, not charts.

## 2. Switching to the new (2026) ARIMA data today would make 7 of our 10 depended-on tables silently return wrong data — with no error to warn us

Our fusion pipeline looks up specific named data tables (e.g. "table
VV_HOV") assuming the name always means the same thing. It doesn't. Of
the 10 tables the pipeline depends on, **7 either now hold completely
different content in 2026, or no longer exist.** Because the old table
names mostly still exist (just pointing at unrelated data now), the
pipeline would keep running without crashing — it would just quietly
produce wrong results. Only 3 of the 10 are safe to use as-is.

- **Full detail:** `outputs/part1_vintage_comparison.md`, Part 1.1
  ("Schema diff"), especially the "Consequence for the fusion pipeline"
  table.
- **Figures:** none — this is a table-name/content-mapping finding, not a
  plotted result.

## 3. ARIMA's population shows almost no relationship between things that should be near-certainties together — this caps what any fusion method can achieve for those variables

We tested 24 pairs of variables where common sense says they should
clearly move together — e.g. "has car insurance" and "owns a car" (you
basically can't have one without the other). **17 of 24 pairs (71%)
showed essentially no relationship.** The starkest case: insurance vs.
ownership came out at r = 0.014 — almost exactly zero — for something
close to a logical necessity. This matters beyond this one check: if the
underlying population data doesn't carry these relationships, no amount
of downstream fusion work — ours included — can recover them. It sets a
hard ceiling on what's achievable for these kinds of variables.

- **Figures:** `outputs/figures/task2_correlation_audit_bars.png` (all 24
  pairs, insurance-vs-ownership is the near-zero standout);
  `outputs/figures/task2_category_means.png` (shows the
  insurance-vs-ownership category averaging near flat vs. other
  categories).
- **Full detail:** `outputs/part2_data_quality_scorecard.md`, Task 2
  ("VV_ cross-correlation audit"), check 2.12.

## 4. Active blocker: Intact's lifestyle survey data can't currently be matched to Intact's simulated survey data by person — this needs to be fixed before more Intact expert-priors work proceeds

Our expert-priors fusion approach for Intact assumes we can line up a
person's lifestyle answers with their simulated survey answers, row by
row. Right now that's not true: splitting by gender should produce a
clean separation if the rows really were the same people, but instead it
comes out close to a 50/50 coin flip either way — a strong sign the two
files describe different, unrelated sets of people. **Any expert-priors
output already produced for Intact using this file should be treated as
unreliable until this is resolved.** Likely fix: re-pull the lifestyle
data filtered to the same 200K people already used elsewhere in the
Intact pipeline, rather than as an independent sample — not yet done.

- **Full detail:** `outputs/part2_data_quality_scorecard.md`, Task 3 Part
  A, check 3.3 ("Root cause of 3.3"); also flagged as the active blocker
  note at the top of `CLAUDE.md`.
- **Figures:** none generated for this specific check.

---

For methodology, sample sizes, and additional lower-priority findings, see
`outputs/part1_vintage_comparison.md` and
`outputs/part2_data_quality_scorecard.md` in full.
