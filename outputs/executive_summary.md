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

## 4. Resolved: Intact's lifestyle survey data can now be matched to Intact's simulated survey data by person — and once matched, it shows the same near-zero pattern as finding #3

Our expert-priors fusion approach for Intact assumes we can line up a
person's lifestyle answers with their simulated survey answers, row by
row. That was blocked: the specific lifestyle file we were using
(`intact_vv_200k`) turned out to describe a different, unrelated set of
people — splitting by gender came out close to a 50/50 coin flip instead
of a clean separation. **That file remains unusable**, but a different,
correct way to make the match was found and independently verified: a
simple `+1` offset on the simulated survey's row numbers lines it up with
ARIMA's own 2024 lifestyle tables — confirmed with a perfect gender
split, a 100% exact match on postal code, and a near-perfect age match,
across all 200,000 people.

With that fixed, we could finally run the same kind of check as finding
#3 on Intact's own data — and got the same headline result: **household
car-insurance coverage vs. actually owning a car came back at r ≈ 0.008,
essentially zero**, for a relationship that should be close to a logical
necessity. This isn't a new, isolated problem — it's the same pattern
from finding #3, now confirmed a second time on independently-verified
data. **Any expert-priors output already produced for Intact using the
old `intact_vv_200k` file should still be treated as unreliable and
re-run using the corrected match.**

- **Full detail:** `outputs/part2_data_quality_scorecard.md`, Task 3 Part
  A, checks 3.3/3.3b ("Resolution"), and Part D, checks 3.18–3.23; also
  updated at the top of `CLAUDE.md`.
- **Figures:** none generated for this specific check.

---

For methodology, sample sizes, and additional lower-priority findings, see
`outputs/part1_vintage_comparison.md` and
`outputs/part2_data_quality_scorecard.md` in full.
