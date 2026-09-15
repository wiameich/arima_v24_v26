# Part 2: Data Quality Scorecard

Status: **All four tasks (1: demographic sanity, 2: VV_ cross-correlation
audit, 3: simulation vs. population coherence, 4: missingness) complete.**

Meant to be reusable for future client engagements — each row is one
atomic check, so this table can be extended without restructuring it.

All checks below use the **2026 ARIMA synthetic population**
(`plusco-arima-data-dropzone-prod/ca/2026/synthetic-population/`),
n = 33,597,827 unless noted otherwise. Full methodology, code, and
visuals are in `notebooks/part2_task1_demographic_sanity.ipynb` and
`notebooks/part2_task4_missingness.ipynb`.

## Legend
- **PASS** — matches expectation, no concern.
- **WARNING** — deviates from expectation but has a plausible benign
  explanation, or is narrow in scope; worth knowing, not urgent.
- **FAIL** — a genuine defect (e.g. a logical impossibility) or a result
  that materially undermines a stated assumption (e.g. real-world
  correlation expectations).

## Task 1 — Demographic sanity checks

| # | Check | n | Observed | Expectation | Status | Notes |
|---|---|---|---|---|---|---|
| 1.1 | Income vs. education correlation | 33,597,827 | Spearman r = 0.1197 | Real-world literature: moderate-to-strong (r ~ 0.3–0.5) | **WARNING** | One of the most robust real-world demographic relationships comes out weak in ARIMA. Relevant to the open question in `CLAUDE.md` about whether ARIMA's near-zero cross-domain correlations are real or a generation artifact (Gaussian copula / IPF max-entropy bias) — this is evidence toward "artifact." |
| 1.2 | Income vs. education monotonicity | 33,597,827 | Mean income rank does NOT increase monotonically with education rank | Should increase monotonically | **WARNING** | Same underlying signal as 1.1, not an independent defect. |
| 1.3 | Household size vs. age life-cycle pattern | 33,597,827 | Peak mean household size at 18–19 (3.47), declining thereafter with a mild bump at 35–39 (2.95) | Peak expected in 30s–40s (family-forming years) | **WARNING (methodology caveat, not a clear defect)** | The hardcoded expectation used here didn't account for 18–19-year-olds often still living in larger family households — the observed pattern (high-young → dip → family bump → decline) is plausible on its own. Flagged as a check-design note, not a confirmed data issue. |
| 1.4 | Age 18–19 + advanced-degree education (implausible combination) | 33,597,827 (1,124,485 in the 18–19 band) | 56,344 rows = 5.008% of the 18–19 band | Near-zero (physically implausible to complete by that age) | **WARNING** | Elevated well beyond a rare-tail rate; consistent with 1.1/1.2's weak age↔education constraint in ARIMA's generation. |
| 1.5 | Household size = 1 with > 1 income contributor (logical contradiction) | 33,597,827 (8,890,265 size-1 households) | 2,622,653 rows = 29.500% of size-1 households | Exactly 0% (logically impossible, not just implausible) | **FAIL** | The most severe Task 1 finding — this is a real internal contradiction (can't have more income earners than household members), not a matter of real-world plausibility. |
| 1.6 | Age 18–19 + $200k+ personal income (implausible combination) | 33,597,827 (1,124,485 in the 18–19 band) | 17,461 rows = 1.552% of the 18–19 band | Small rare tail | **WARNING** | Elevated but plausible as a genuine rare tail (young entrepreneurs/athletes/etc. exist in reality), lower severity than 1.4. |

## Task 2 — VV_ cross-correlation audit (24 pairs)

**Headline result: `vv_aut_1` (household's auto is covered by insurance)
vs. `owns_any_vehicle` (derived from `vv_vek`'s vehicle-ownership
indicators) — Spearman r = 0.0141, n = 33,596,974.** This pair is about as
close to a logical necessity as exists in survey data (you cannot have
auto insurance coverage without owning a vehicle), yet ARIMA's 2026
synthetic population shows essentially **no relationship at all**. This is
the single strongest data point gathered in this audit and is flagged
**FAIL**, not WARNING — a near-tautological relationship collapsing to
zero is a materially different (and more severe) finding than a merely
weaker-than-expected real-world correlation.

**Overall: 17 of 24 pairs (71%) flagged** — wrong direction or
suspiciously low magnitude (|r| < 0.05, chosen because these pairs were
selected specifically for *obvious* expected relationships). All 6
tables used were verified against their actual 2026 variable descriptions
before selection (not assumed from table code), per Part 1.1's finding
that many `VV_*` table_ids changed content between vintages — see
`notebooks/part2_task2_vv_correlation_audit.ipynb` for the verification
table. Full code and visuals in that notebook.

| # | Pair | Category | n | Spearman r | Expected | Status |
|---|---|---|---|---|---|---|
| 2.1 | `vv_aut_1` × `vv_aut_2` (covered × acquired via broker) | auto | 33,596,974 | 0.0460 | + | WARNING (low magnitude) |
| 2.2 | `vv_aut_1` × `vv_aut_3` (covered × acquired via bank) | auto | 33,596,974 | -0.0272 | + | WARNING (wrong direction, but noise-level magnitude) |
| 2.3 | `vv_aut_1` × `vv_aut_4` (covered × acquired via insurance co.) | auto | 33,596,974 | 0.0060 | + | WARNING (low magnitude) |
| 2.4 | `vv_aut_2` × `vv_aut_4` (broker × insurance co., mutually exclusive) | auto | 33,597,827 | -0.1887 | − | **PASS** |
| 2.5 | `vv_aut_2` × `vv_aut_3` (broker × bank, mutually exclusive) | auto | 33,597,827 | -0.0786 | − | **PASS** |
| 2.6 | `vv_dig_10` (checks social media daily) × age | digital_vs_demo | 33,522,967 | -0.1134 | − | **PASS** |
| 2.7 | `vv_dig_11` (internet = main news source) × age | digital_vs_demo | 33,523,441 | -0.1780 | − | **PASS** |
| 2.8 | `vv_dig_9` (researches online before buying) × age | digital_vs_demo | 33,523,064 | -0.1274 | − | **PASS** |
| 2.9 | `vv_dig_2` (life w/o internet less fun) × age | digital_vs_demo | 33,492,197 | -0.2620 | − | **PASS** |
| 2.10 | `vv_dig_5` (internet = social belonging) × age | digital_vs_demo | 33,523,224 | -0.2466 | − | **PASS** |
| 2.11 | `vv_dig_3` (privacy/data concern) × age | digital_vs_demo | 33,522,845 | -0.2038 | + (low-confidence going in) | WARNING (wrong direction vs. a stated low-confidence prior; real literature is mixed on this direction, so not necessarily a defect) |
| 2.12 | `vv_aut_1` (insured) × `owns_any_vehicle` | insurance_vs_ownership | 33,596,974 | **0.0141** | + | **FAIL — headline finding** |
| 2.13 | `vv_aut_1` × `vv_vek_2` (owns Compact) | insurance_vs_ownership | 33,596,974 | 0.0194 | + | WARNING (low magnitude) |
| 2.14 | `vv_aut_1` × `vv_vek_5` (owns Midsize) | insurance_vs_ownership | 33,596,974 | 0.0137 | + | WARNING (low magnitude) |
| 2.15 | `vv_tra_21` (passionate about travel) × `vv_trd_2` (# trips/yr) | travel | 9,596,757 | 0.0164 | + | WARNING (low magnitude) |
| 2.16 | `vv_tra_2` (vacation = escape) × `vv_trd_2` | travel | 9,596,757 | 0.0209 | + | WARNING (low magnitude) |
| 2.17 | `vv_tra_17` (seeks holiday inspiration) × `vv_trd_2` | travel | 9,596,757 | 0.0085 | + | WARNING (low magnitude) |
| 2.18 | `vv_tra_21` (passionate about travel) × `vv_trc_2` (took overnight trip) | travel | 33,597,827 | 0.0177 | + | WARNING (low magnitude) |
| 2.19 | `vv_auv_9` (would always choose luxury auto) × `owns_luxury_vehicle` | luxury | 33,597,827 | 0.0303 | + | WARNING (low magnitude) |
| 2.20 | `vv_auv_15` (prefers luxury vehicle) × `owns_luxury_vehicle` | luxury | 33,597,827 | 0.0283 | + | WARNING (low magnitude) |
| 2.21 | `vv_tra_10` (wants luxurious vacation) × `vv_trd_2` | luxury | 9,596,757 | 0.0150 | + | WARNING (low magnitude) |
| 2.22 | `vv_tra_10` × `vv_trc_2` | luxury | 33,597,827 | 0.0183 | + | WARNING (low magnitude) |
| 2.23 | `vv_auv_9` (luxury auto preference, cross-domain) × `vv_trd_2` | luxury | 9,596,757 | 0.0202 | + | WARNING (low magnitude) |
| 2.24 | `vv_auv_15` (cross-domain) × `vv_trd_2` | luxury | 9,596,757 | 0.0157 | + | WARNING (low magnitude) |

**What still holds — not everything is flat:** 5 of 6 digital-attitude-vs-age
pairs (2.6–2.10) came back correctly signed with real magnitude
(r = -0.11 to -0.26), and one within-table auto pair (2.4, mutually
exclusive acquisition channels) showed genuine structure (r = -0.19).
ARIMA appears to preserve **demographic-driven attitude gradients**
reasonably well; it is specifically **attitude↔behavior relationships and
logical-necessity constraints** (insurance/ownership, travel/luxury
attitude vs. frequency, and the covered→acquisition-channel hierarchy)
that collapse toward statistical independence. `vv_trd_2` (# trips/year)
has a 71.4% blank rate, so pairs using it (2.15–2.17, 2.21, 2.23–2.24) run
on n≈9.6M rather than the full 33.6M population — still an enormous
sample, so this is not a small-sample-noise explanation for the low r.

## Task 3 — Simulation vs. population coherence

Checks whether ARIMA's own simulated Intact and Novo surveys correlate
with `VV_*` lifestyle data for the same person. Full code and visuals in
`notebooks/part2_task3_simulation_coherence.ipynb`. Uses the 2024H2 200K
population (`gs://arima-clustering-pipeline/data/data_fusion/`), not the
2026 population Task 2 used.

**Part A — ID linkage (required before computing anything, per the same
standard as every prior ID-based join this session):**

| # | Check | Result | Status |
|---|---|---|---|
| 3.1 | Novo: `novo_arima_simulated_200k` vs. `all417_decoded` | Age band exact match rate = 100.0000% (n=200,000); gender is a perfect bijection (zero crossover) | **PASS — confirmed trustworthy** |
| 3.2 | Intact: `intact_arima_sim_200k` vs. `intact_population_200k` | Perfect gender bijection (96,917/96,917, 103,083/103,083, zero crossover) | **PASS** |
| 3.3 | Intact: `intact_arima_sim_200k` vs. `intact_vv_200k` | Flat gender split (~51%/49% either way); age correlation r=0.0061 | **FAIL — blocked** |

**Root cause of 3.3, traced specifically:** `intact_vv_200k` sits in an
independently-generated ID space from the other two Intact tables — it is
not simply the general `all417` pull mislabeled either (only 0.57% raw ID
overlap with `all417`, and no signal even within that sliver). No
`VV_*` lifestyle file currently available is ID-linked to the Intact
simulation, so **no Intact-side correlation was computed** — doing so
would silently mix unrelated individuals, the same failure mode
documented in Part 1.2, now found within a client-facing deliverable.

**Part B — Novo (8 pairs, n≈200,000): 0/8 pairs OK, all flagged.**
Variables verified against the 2024H2 dictionary and, for `Q19 (KGS)`,
independently confirmed against `novo_full_data_set.xlsx`'s `Copy of
datamap` sheet — the actual Novo survey data dictionary (entry:
`Q19_KGS: How much weight would you like to lose overall?`) — the
strongest available source, superseding an earlier, weaker cross-reference
to the Sequential PMM documentation (initially inferred only from a
hardcoded label in `novo_fusion.py`, flagged as an open caveat at the
time, now resolved).

| # | Pair | n | Spearman r | Expected | Status |
|---|---|---|---|---|---|
| 3.4 | `Q19 (KGS)` (weight-loss goal, kg) × dieting | 199,998 | **-0.4616** | + | **FAIL — wrong direction, real/substantial, confirmed genuine** |
| 3.5 | `Weight Loss (%)` × dieting | 199,998 | **-0.4611** | + | **FAIL — wrong direction, same** |
| 3.6 | `BMI` × dieting | 199,998 | -0.0150 | + | WARNING (wrong direction, but noise-level, not meaningfully inverted) |
| 3.7 | `Weight (KG)` × sports_regularly | 199,990 | 0.0601 | − | WARNING (wrong direction, but noise-level, not meaningfully inverted) |
| 3.8 | `BMI` × sports_regularly | 199,990 | -0.0319 | − | WARNING (low magnitude) |
| 3.9 | `Q19 (KGS)` × sports_regularly | 199,990 | -0.0103 | − | WARNING (low magnitude) |
| 3.10 | `BMI` × exercise_important_belief | 200,000 | -0.0055 | − | WARNING (low magnitude) |

**Full methodological detail for 3.4/3.5** (the headline reversal —
documented to the same standard as the ID-continuity and Task 2 findings):

| | 3.4: `Q19 (KGS)` × `dieting` | 3.5: `Weight Loss (%)` × `dieting` |
|---|---|---|
| Variable 1 | `Q19 (KGS)` — Novo survey, numeric, kg. Datamap label: "How much weight would you like to lose overall?" | `Weight Loss (%)` — a derived/simulated numeric field; **no matching raw-question entry found** in the Novo datamap, so its exact construction is not independently confirmed (flagged, not assumed) |
| Variable 2 | `dieting` = `VV_DIE_1` ("Control Diet Currently", Yes/No), decoded via the CA_2024H2 `VARIABLE_MAPPING` dictionary | same |
| Expected sign / mechanism | + — selection-into-dieting: people currently managing their diet should, on average, have set some weight-loss goal. General epidemiological/survey pattern, **not a specific cited study** — a weaker-confidence prior than the cardiometabolic mechanism in 3.11/3.12 below | Same mechanism, applied to a percentage rather than kg |
| n | 199,998 | 199,998 |
| Observed r | **-0.4616** | **-0.4611** |
| Confidence in the estimate | High confidence in the number itself (n≈200K, both variables verified against source dictionaries — not a measurement artifact). Lower confidence in *why*: the expected-sign mechanism is a reasonable prior, not a certainty — see 3.11-3.17, which found this reversal does not generalize | Same |

**Part B.2 — Follow-up (recommended next step, now completed): is the
reversal isolated or systematic?** 7 further pairs, sourced directly from
`novo_full_data_set.xlsx`'s `Copy of datamap` sheet (stronger sourcing
than the original 8 pairs' label-inference).

| # | Pair | n | Spearman r | Expected | Status | Mechanism / source |
|---|---|---|---|---|---|---|
| 3.11 | `BMI` × `Q15r1` (cardiometabolic: high BP, Type 2 diabetes, high cholesterol) | 200,000 | **0.2680** | + | **PASS** | Textbook obesity-comorbidity link; datamap: "Have you been diagnosed with or do you regularly experience any of the following health conditions?" |
| 3.12 | `Weight (KG)` × `Q15r1` | 200,000 | **0.2075** | + | **PASS** | Same mechanism |
| 3.13 | `BMI` × `Q15r2` (mechanical: joint/back pain, restricted mobility) | 200,000 | 0.0772 | + | **PASS** | Well-established excess-weight/joint-stress link, per datamap |
| 3.14 | `Weight (KG)` × `Q15r2` | 200,000 | 0.0513 | + | **PASS** | Same mechanism |
| 3.15 | `BMI` × `Q11r5` (exercises ≥weekly, 0-10 agreement) | 193,144 | -0.0324 | − | WARNING (low magnitude) | Established exercise-BMI link, per datamap |
| 3.16 | `Weight (KG)` × `Q11r5` | 193,144 | 0.0075 | − | WARNING (wrong direction, noise-level) | Same mechanism |
| 3.17 | `BMI` × `Q28r13` (new diagnosis, e.g. Type 2 Diabetes/Sleep Apnea, as weight-loss trigger) | 200,000 | 0.0098 | + | WARNING (low magnitude) | Same obesity-comorbidity mechanism as 3.11, per datamap |

**Verdict on 3.11-3.17: the 3.4/3.5 reversal is isolated, not
systematic.** 4 of 7 follow-up pairs are correctly signed with real
magnitude. `BMI`/`Weight (KG)` × cardiometabolic conditions (3.11, 3.12)
are **the strongest cross-domain correlations found anywhere in this
entire Part 2 audit** — stronger than anything in Task 2, and stronger
than the rest of Task 3 — for one of the most textbook-established
mechanisms in epidemiology. This does not generalize the 3.4/3.5 reversal
into a broader pattern; it looks like a localized defect in that specific
pair.

**Part C — Direct comparison against Task 2 (the "ceiling" finding):**

| | Mean \|r\| | % flagged |
|---|---|---|
| Task 2 (raw VV_ vs. VV_, 2026 population) | 0.0715 | 71% (17/24) |
| Task 3 original 8 pairs (Novo simulation vs. VV_, 2024H2 population) | 0.1344 | 100% (8/8) |
| Task 3 follow-up 7 pairs (3.11-3.17) | 0.0938 | 43% (3/7) |

**The higher mean \|r\| in the original Task 3 8 pairs is not evidence of
better correlation preservation** — it is driven entirely by the two
wrong-signed pairs (3.4, 3.5); excluding those two, the remaining 6 are in
the same near-zero range Task 2 found. **The ceiling is uneven, not
flat:** Novo's own ARIMA-generated simulation can preserve real,
correctly-signed, even moderately strong cross-domain signal for some
mechanisms (3.11, 3.12), while showing near-zero signal for others
(3.6-3.10, 3.15-3.17, consistent with Task 2's general pattern) and one
substantial, real reversal for a specific pair (3.4, 3.5) that does not
generalize. A fusion method's achievable ceiling therefore likely varies
by mechanism rather than being uniformly near-zero — but it is still
bounded by what ARIMA's own simulation already contains, whether that's
strong, weak, or (in one case) wrong-signed.

## Task 4 — Missingness audit

| # | Check | n | Observed | Expectation | Status | Notes |
|---|---|---|---|---|---|---|
| 4.1 | Real parquet null rate | 10,937 columns sampled across 409 tables (1 shard each) | 0.0% uniformly, everywhere (max observed = 0.0%) | N/A — structural finding | **PASS (informational)** | Not a defect: this data model encodes missingness via explicit sentinel categories (e.g. `__blank__`), never real parquet NULLs. A naive `.isna()` check on this data would wrongly conclude there's no missingness at all — noted so downstream work doesn't make that mistake. |
| 4.2 | Sparse domains (variables with blank/non-response rate > 50%) | 10,943 variables (exact, full-population, via `variable_mapping_2026.csv`'s `national_count`) | 1,869 variables (17%) exceed 50% blank; 104 tables have ≥50% of their own variables over that threshold | N/A — exploratory flag | **WARNING** | Mostly plausible niche/low-incidence content (e.g. specific regional newspapers, specific TV shows, city-specific transit sub-questions) — checked variable descriptions directly to confirm this rather than assume it. Not evidence of broken data on its own, but these variables carry little signal and should be deprioritized or excluded in Task 2's correlation-pair selection. |
| 4.3 | Fully unpopulated (100.000000% blank) individual variables | 10,943 variables checked | Exactly 5 variables, in only 2 tables (`vv_puc`, `vv_shp` — both otherwise well-populated): `vv_puc_60`, `vv_puc_64`, `vv_puc_68`, `vv_puc_83`, `vv_shp_17` | Should be rare/near-zero | **WARNING (narrow)** | Likely dead/unused sub-options (e.g. a specific city's transit "never used" sub-code, a residual "Other" shopping category) rather than a structural defect — both host tables are otherwise healthy. An earlier pass overstated this as "15 tables 100% blank across every variable," which was a misreading of a per-table aggregate statistic (fraction of a table's variables exceeding the 50% threshold, not the table's actual blank rate); corrected here after re-checking the real per-variable numbers directly. |
| 4.4 | Constant-column detection: dictionary vs. real files | 10,943 variables (dictionary) vs. 10,937 columns (409-table file scan, 1 shard each) | Dictionary flags 0 constant variables; direct file scan flags 69 | Should broadly agree | **WARNING** | 0/69 agreement — the variable dictionary's category list reflects the *possible* answer set, not the *observed* distribution, so it cannot be relied on alone to find practically-constant (zero-variance) variables. Relevant to Task 2: these 69 columns should be excluded from correlation analysis (undefined/zero correlation for a constant variable) using the file-scan result, not the dictionary. |

No fixes applied to the underlying data anywhere in this scorecard — every
row reports an observed result, not a correction.
