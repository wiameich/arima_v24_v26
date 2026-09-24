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
| 3.3 | Intact: `intact_arima_sim_200k` vs. `intact_vv_200k` | Flat gender split (~51%/49% either way); age correlation r=0.0061 | **FAIL — `intact_vv_200k` unusable** (see resolution below — this file specifically is not the fix) |
| 3.3b | Intact (corrected linkage): `intact_arima_sim_200k`'s `id + 1` vs. `arrima-snowflake/CA_2024H2`'s own `VV_*` tables | Perfect gender bijection (96,917/103,083, zero crossover, same split as 3.2); 100.0000% GEO/postal-code exact match (n=200,000); clean, non-crossing age-band diagonal, Spearman r=0.978 vs. `AGENUM` | **PASS — RESOLVED 2026-09-21** |

**Root cause of 3.3, traced specifically:** `intact_vv_200k` sits in an
independently-generated ID space from the other two Intact tables — it is
not simply the general `all417` pull mislabeled either (only 0.57% raw ID
overlap with `all417`, and no signal even within that sliver). `intact_vv_200k`
itself remains unusable and this finding stands.

**Resolution (2026-09-21):** a corrected linkage was found and verified
independently, to the same standard as every other ID-based join in this
project (gender bijection + GEO/age spot-check, not raw ID overlap — see
3.3b above and `notebooks/part2_task3_simulation_coherence.ipynb`'s "UPDATE"
cells in Part A). It has nothing to do with `intact_vv_200k`:
**`intact_arima_sim_200k`'s `id` column, offset by `+1`, links directly to
`arrima-snowflake/CA_2024H2`'s own `VV_*` tables** — the same 2024H2 bucket
used throughout Part 1, not the `arima-clustering-pipeline` bucket
`intact_vv_200k` came from. This unblocks Intact-side correlation work; see
Part D below.

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

**Part D — Intact (corrected, 6 pairs, n up to 200,000):** now that 3.3 is
resolved (3.3b above), this repeats Part B's exercise for Intact using the
`id + 1` linkage to `CA_2024H2`'s `VV_*` tables.

**Variable-meaning confirmation:** Intact's own `Q10`–`Q58` survey columns
have no data dictionary anywhere in this project's usual sources (checked:
local files, both GCS buckets used elsewhere in this project — including
`code/`, `dashboards/`, `dec_results/` — and `~/reference/arima_fusion`).
One was located externally — `data/intact_datamap/Azimut_template_mapped_TopLevel_only.xlsx`
— but its own README describes it as a **dashboard-to-template crosswalk,
not a native survey codebook**, and it labels its Q-variable rows
`'2026 Survey'`, a different vintage than the 2024H2 data this simulation
links to. Given that, it was checked for internal consistency before being
trusted: 128/157 (81.5%) of Intact's `Q`-variables show an exact
category-count match against what's actually observed in
`intact_arima_sim_200k`; most of the 29 discrepancies are explainable as
dashboard-side binning of continuous/count variables (e.g. `Q16r*`, `Q31`,
`Q34`, `Q58r*`), not real content drift. Two genuine gaps treated as
unconfirmed: `Q41` (only exists as a nested grid in the real data, not the
flat item the datamap lists) and `Q44` (no question text at all in the
datamap). Separately: **this datamap never gives an explicit numeric
code→label order**, only category content/counts — so for binary items,
direction was independently cross-validated against a known anchor (e.g.
`Q18` "has driver's licence": code 1 dominates and rises with age,
consistent with code 1 = Yes at an 82.6% rate); for 5-point Likert/attitude
batteries (`Q33`, `Q35`, `Q37`, `Q40`, `Q57`, etc.) no such anchor exists,
so **Part D is restricted to binary, count, and continuous variables with
independently confirmed direction** — Likert pairs are excluded from this
pass rather than risk a wrong-direction verdict built on a guessed code
order.

| # | Pair | n | Spearman r | Expected | Status |
|---|---|---|---|---|---|
| 3.18 | `VV_AUU_1` (Hhld. Auto Is Covered, CA_2024H2 via `id+1`) × `Q17r1`–`r5` (owns/leases any car type, Intact) | 177,214 | **0.0079** | + | **WARNING — essentially zero (household-level vs. personal-level variable; some gap may reflect real multi-person-household effects, not purely an ARIMA artifact — see caveat below)** |
| 3.19 | `VV_HOV_6` (# children <18, CA_2024H2 via `id+1`) × `Q48r4` (life event: birth of a child, last 12mo) | 200,000 | 0.1054 | + | **PASS** |
| 3.20 | `AGENUM` × `Q18` (has valid driver's licence) | 200,000 | 0.1420 | + | **PASS** |
| 3.21 | `AGENUM` × `Q12` (owns primary residence) | 200,000 | 0.1740 | + | **PASS** |
| 3.22 | `Q48r5` (life event: purchased a car, last 12mo) × `Q31` (# quotes obtained at last shopping) | 200,000 | 0.0483 | + | **PASS (weak)** |
| 3.23 | `VV_HOV_6` (# children <18, CA_2024H2 via `id+1`) × `Q48r7` (life event: child/spouse obtained driver's licence) | 200,000 | 0.0487 | + | **PASS (weak)** |

**Verdict on 3.18–3.23: 5/6 correctly signed; 3.18 is the standout.**
Demographic gradients (3.20, 3.21) hold up well, consistent with Task 2's
general pattern. **3.18 is the main reason this fix mattered**: household
auto insurance coverage (from genuinely-linked CA_2024H2 data) vs. owning
or leasing any car type comes back at **r≈0.008 — essentially zero**. The
row-normalized crosstab confirms this isn't a rare-category artifact: 77.1%
of car owners are insured vs. 75.9% of non-owners — barely any difference.
This is the same "logical-necessity variables collapse toward
independence" pattern Task 2 flagged for `vv_aut_1` × vehicle ownership
(r=0.014, checks 2.1–2.3 area) — except that finding was on a linkage never
confirmed reliable, while 3.18 is on data independently verified above
(perfect gender bijection, 100% GEO match). **Caveat:** `VV_AUU_1` is a
household-level variable and `Q17r*` is asked at the personal level ("cars
**you** currently own"), so some of this gap could reflect other household
members owning/insuring a car this respondent doesn't personally hold — a
real feature of multi-person households, not necessarily an ARIMA
artifact. That confound wasn't present in Task 2's original (both
household-level) pair, so 3.18 should be read as *consistent with, not a
clean replication of* Task 2's finding. 3.19/3.22/3.23 (cross-domain and
behaviour-behaviour pairs) are correctly signed but modest, which is itself
expected given the timeframe/level mismatches involved (a standing
household attribute vs. a specific last-12-months event).

## Task 4 — Missingness audit

| # | Check | n | Observed | Expectation | Status | Notes |
|---|---|---|---|---|---|---|
| 4.1 | Real parquet null rate | 10,937 columns sampled across 409 tables (1 shard each) | 0.0% uniformly, everywhere (max observed = 0.0%) | N/A — structural finding | **PASS (informational)** | Not a defect: this data model encodes missingness via explicit sentinel categories (e.g. `__blank__`), never real parquet NULLs. A naive `.isna()` check on this data would wrongly conclude there's no missingness at all — noted so downstream work doesn't make that mistake. |
| 4.2 | Sparse domains (variables with blank/non-response rate > 50%) | 10,943 variables (exact, full-population, via `variable_mapping_2026.csv`'s `national_count`) | 1,869 variables (17%) exceed 50% blank; 104 tables have ≥50% of their own variables over that threshold | N/A — exploratory flag | **WARNING** | Mostly plausible niche/low-incidence content (e.g. specific regional newspapers, specific TV shows, city-specific transit sub-questions) — checked variable descriptions directly to confirm this rather than assume it. Not evidence of broken data on its own, but these variables carry little signal and should be deprioritized or excluded in Task 2's correlation-pair selection. |
| 4.3 | Fully unpopulated (100.000000% blank) individual variables | 10,943 variables checked | Exactly 5 variables, in only 2 tables (`vv_puc`, `vv_shp` — both otherwise well-populated): `vv_puc_60`, `vv_puc_64`, `vv_puc_68`, `vv_puc_83`, `vv_shp_17` | Should be rare/near-zero | **WARNING (narrow)** | Likely dead/unused sub-options (e.g. a specific city's transit "never used" sub-code, a residual "Other" shopping category) rather than a structural defect — both host tables are otherwise healthy. An earlier pass overstated this as "15 tables 100% blank across every variable," which was a misreading of a per-table aggregate statistic (fraction of a table's variables exceeding the 50% threshold, not the table's actual blank rate); corrected here after re-checking the real per-variable numbers directly. |
| 4.4 | Constant-column detection: dictionary vs. real files | 10,943 variables (dictionary) vs. 10,937 columns (409-table file scan, 1 shard each) | Dictionary flags 0 constant variables; direct file scan flags 69 | Should broadly agree | **WARNING** | 0/69 agreement — the variable dictionary's category list reflects the *possible* answer set, not the *observed* distribution, so it cannot be relied on alone to find practically-constant (zero-variance) variables. Relevant to Task 2: these 69 columns should be excluded from correlation analysis (undefined/zero correlation for a constant variable) using the file-scan result, not the dictionary. |

### Detail for 4.3 — the 5 fully-blank (100.000000%) variables, with contrast

Pulled directly from `data/variable_mapping_2026.csv`'s `description` field (literal text, not paraphrased), with `blank_pct` recomputed fresh from the same `national_count`-based method as 4.2/4.3 (summing blank-labeled categories' `national_count` over the variable's total, n=33,597,827 throughout — the full 2026 population, not a sample):

| Variable ID | Table | Description | Blank % |
|---|---|---|---|
| `vv_puc_60` | `vv_puc` | When Last time used (Ottawa) - Never used | 100.0000% |
| `vv_puc_64` | `vv_puc` | When Last time used (Calgary) - Never used | 100.0000% |
| `vv_puc_68` | `vv_puc` | When Last time used (Edmonton) - Never used | 100.0000% |
| `vv_puc_83` | `vv_puc` | Number of Times Boarded Last Day (Vancouver CMA) - The West Coast Express | 100.0000% |
| `vv_shp_17` | `vv_shp` | Categories Shop Most Often - Other | 100.0000% |

**Contrast — other variables in the same two tables, for the "otherwise well-populated" claim in 4.3:**

| Variable ID | Table | Description | Blank % |
|---|---|---|---|
| `vv_puc_1` | `vv_puc` | When Last time used (Toronto CMA) - Yesterday - TTC Subway | 0.0000% |
| `vv_puc_57` | `vv_puc` | When Last time used (Ottawa) - Yesterday | 99.9617% |
| `vv_puc_91` | `vv_puc` | When Last time used Bus - Summary | 90.6646% |
| `vv_puc_93` | `vv_puc` | When Last time used - Yesterday | 0.0000% |
| `vv_shp_1` | `vv_shp` | $ Spent Online Past Month | 34.5945% |
| `vv_shp_11` | `vv_shp` | Categories Shop Most Often - Groceries | 26.0034% |
| `vv_shp_65` | `vv_shp` | How Often Shop Online | 14.8894% |
| `vv_shp_93` | `vv_shp` | Purchase Method Personally Used - Credit Card | 0.0000% |

`vv_puc` (Public Transit Usage) is a 100-variable table structured as repeated city-by-mode-by-recency batteries (Toronto/Montreal/Vancouver/Ottawa/Calgary/Edmonton × subway/bus/LRT/etc. × yesterday/past week/longer ago/never used). Its fully-populated variables (0.0000% blank, e.g. `vv_puc_1`, `vv_puc_93`) are the higher-level "when last used" summary items everyone answers; `vv_puc_60`/`vv_puc_64`/`vv_puc_68` are the "never used" sub-code specifically for three smaller-transit-system cities (Ottawa/Calgary/Edmonton) — a residual bucket that this synthetic population simply never assigns anyone to, alongside `vv_puc_57`-`vv_puc_59` (same three cities' other recency codes) sitting at 99.9%+ blank, not 100% — confirming this is a gradient of rarity, not a single broken table. `vv_shp` (Shopping) is a 100-variable table mixing well-populated purchase-method/event flags (0.0000% blank, e.g. `vv_shp_93`, purchase method used) with sparser "shop most often" category picks (`vv_shp_1`, `vv_shp_11`); `vv_shp_17` ("Other" shopping category) is the one true zero within that mix — again a residual catch-all bucket, not evidence the table itself is unpopulated.

### Detail for 4.2 — representative sample of >50%-blank variables (18 of 1,869), literal descriptions

Stratified sample spanning the full blank-rate range (50.5%-99.97%) and 18 distinct tables (of the 233 tables that contain at least one >50%-blank variable), drawn from `data/variable_mapping_2026.csv`'s literal `description` field rather than paraphrased into categories. Confirms the category-level pattern claimed in 4.2 (regional newspapers, niche magazines, specific TV shows, city-specific transit sub-questions) with citable, verbatim examples:

| Variable ID | Table | Description | Blank % |
|---|---|---|---|
| `vv_mah1_86` | `vv_mah1` | CAA Saskatchewan - Devices Used to Access Sometimes | 99.9712% |
| `vv_daj2_96` | `vv_daj2` | The Windsor Star - How Last Weekday Issue Obtained | 99.7773% |
| `vv_daj1_70` | `vv_daj1` | The Edmonton Sun - Time Spent with Last Saturday Issue (in min) | 99.6072% |
| `vv_dai8_66` | `vv_dai8` | The Standard - Time Spent On Last Day (in min) | 99.4451% |
| `vv_mai1_22` | `vv_mai1` | Les Idees de ma Maison - # of Occasions Read a Typical Issue | 99.1447% |
| `vv_dai11_15` | `vv_dai11` | Winnipeg Free Press - How Often Access Publication's Digital Content | 98.8471% |
| `vv_weg1_30` | `vv_weg1` | French - Personally Watched on Any Screen/Device per Month - NCIS (S+) | 98.2282% |
| `vv_mai1_9` | `vv_mai1` | Hello! Canada - Percentage Read | 97.5015% |
| `vv_mai1_74` | `vv_mai1` | Zoomer Magazine - Percentage Read | 96.7730% |
| `vv_dam1_48` | `vv_dam1` | When Last Time Action Taken - Recommended the advertised product/brand/service | 95.8308% |
| `vv_tvc_64` | `vv_tvc` | English - Personally Watch on Any Screen/Any Device Per Week - National Geographic | 94.2990% |
| `vv_fly_13` | `vv_fly` | How Often Personally Use Print/Digital to Plan/Make Purchases - Sports Equipment | 92.3955% |
| `vv_cox_2` | `vv_cox` | Formats personally use - Others Sometimes | 89.6371% |
| `vv_out1_23` | `vv_out1` | When Last Time Action Taken - Downloaded Coupon | 84.8701% |
| `vv_res_26` | `vv_res` | Type of Food Used Past 30 Days - Ice Cream | 79.7042% |
| `vv_but_4` | `vv_but` | Equipment/Distribution:  Shipping/Transportation/Distribution Services/Construction | 74.1293% |
| `vv_inz_64` | `vv_inz` | Online Activities by Device Past 30 Days - Watched Long Form Videos(Longer than 21 min) | 64.4288% |
| `vv_lot_26` | `vv_lot` | $ Spent/Average Month | 50.5391% |

Sample drawn by taking every ~104th row (evenly spaced by rank) of the 1,778 >50%-blank variables outside `vv_puc`/`vv_shp` (already detailed above), sorted descending by blank %, so the sample spans the full range rather than clustering at either extreme. Full 1,869-row list is reproducible from `data/variable_mapping_2026.csv` via the method in `notebooks/part2_task4_missingness.ipynb`, not attached in full here to keep this document a readable size.

No fixes applied to the underlying data anywhere in this scorecard — every
row reports an observed result, not a correction.

## Task 2/3 Follow-up — Mutual Information, Subgroup, and Joint-Dependence Analysis

Three follow-up analyses requested on top of Task 2's VV_ cross-correlation
audit and Task 3's simulation-coherence audit: (1) mutual information, to
check whether Spearman's rank correlation is missing non-monotonic
structure; (2) subgroup/interaction analysis, to check whether a pairwise
correlation changes materially within income/province/household-size/age
subgroups; (3) joint-dependence trios, to check whether a pairwise result
is actually driven by a third variable invisible to a pairwise test. Full
code in `notebooks/part2_task5_mi_subgroup_joint.ipynb`.

**Data source and a methodological correction made before running anything:**
the 16 new pairs below use `data/all417/arima_200k_all417_decoded.parquet`
(200K rows, 2024H2 vintage, ~380 VV_ tables spanning every domain). Its `id`
was verified this session to link *directly* (no offset) to `CA_2024H2`'s
own tables — a 20,000-id sample joined on raw `id` against `CA_2024H2/VV_DEM`
produced a perfect gender bijection, the same attribute-spot-check standard
used everywhere else in this project. Because this file is 2024H2-vintage,
every variable was decoded/labeled using `CA_2024H2`'s own dictionary, not
`variable_mapping_2026.csv` — this caught real problems before they became
errors: `vv_ele`, `vv_inv`, `vv_wor` (as "days worked"), `vv_med` (as
"conditions"), and `vv_buu` (as "business trips") all looked like clean
candidate variables under their 2026 table names but hold unrelated content
in the 2024H2 dictionary (e.g. `vv_ele` is TV-set ownership in 2024H2, not
electric-vehicle intent), consistent with CLAUDE.md's standing finding that
39% of table_ids have drifted content across vintages. They were dropped
from the candidate list rather than mis-decoded. Separately, `VV_DEM_4`
("# of Household Income Contributors") was initially assumed to be
household size; `VV_HOV_7` ("Total # of people in hhld") is the actual
household-size variable and is what's used below.

### F1 — Mutual information

Two anchors (Task 2's `vv_aut_1` × `owns_any_vehicle`, r=0.0141, n=33,596,974;
Task 3's `VV_AUU_1` × `owns_any_car`, r=0.0079, n=177,214) were rebuilt from
their original sources to compute normalized mutual information (NMI,
`sklearn.metrics.normalized_mutual_info_score` on integer-factorized
category codes) alongside their known Spearman r, giving a **near-independence
NMI baseline of ~0.0001–0.0002**. 16 new pairs were then tested, at least one
per domain named in the request (food/drinks, lifestyle, health/wellness,
professional life, shopping, media, auto, travel, digital), chosen the same
way as Task 2's original pairs (obvious expected direction from the variable
descriptions).

| # | Pair | Domain | n | Spearman r | Expected | NMI | Status |
|---|---|---|---|---|---|---|---|
| F1.1 | `VV_ORG_1` (organic food, hhld) × `VV_VIT_1` (vitamins) | food_drink × health | 179,686 | -0.0114 | + | 0.0001 | **FAIL — wrong direction** |
| F1.2 | `VV_DIE_1` (control diet) × `VV_VEG_1` (vegan products) | food_drink × health | 175,861 | 0.0140 | + | 0.0002 | WARNING (low magnitude) |
| F1.3 | `VV_FIT_1` (fitness club member) × `VV_DIE_1` (control diet) | lifestyle × health | 199,993 | 0.0132 | + | 0.0002 | WARNING (low magnitude) |
| F1.4 | `VV_FIT_1` (fitness club member) × `VV_SMO_1` (smoked, last 6mo) | lifestyle × health | 194,705 | 0.0091 | − | 0.0001 | **FAIL — wrong direction** |
| F1.5 | `VV_TOT_1` (investment $ bracket) × `VV_ORG_1` (organic food) | professional_life × shopping | 151,731 | -0.0345 | + | 0.0005 | **FAIL — wrong direction** |
| F1.6 | `VV_ADI_1` (searched online after ad) × `VV_SHP_2` (online shopping freq.) | media × shopping | 133,940 | -0.0087 | + | 0.0001 | **FAIL — wrong direction** |
| F1.7 | `VV_TRU_1` (news-checking freq.) × `VV_DIG_11` (internet = main news source) | media × digital | 138,722 | **-0.0930** | + | **0.0039** | **FAIL — wrong direction, non-trivial magnitude (see below)** |
| F1.8 | `VV_ADB_1` (ad blocker) × `VV_BIN_1` (binge-watch freq.) | digital × media/lifestyle | 128,801 | 0.0102 | + | 0.0001 | WARNING (low magnitude) |
| F1.9 | `VV_AUT_1` (vehicle purchase, past 12mo) × `VV_DEM_3` (income) | auto × demographics | 164,248 | -0.0183 | + | 0.0002 | **FAIL — wrong direction** |
| F1.10 | `VV_TRB_1` (vacation trip, past 12mo) × `VV_DEM_3` (income) | travel × demographics | 200,000 | 0.0232 | + | 0.0003 | WARNING (low magnitude) |
| F1.11 | `VV_VAC_1` (vacation trip, past 12mo) × `VV_RES_1` (dined out, past 30d) | travel × lifestyle | 199,992 | 0.0063 | + | 0.0000 | WARNING (low magnitude) |
| F1.12 | `VV_LOT_1` (bought lottery ticket) × `VV_DEM_3` (income) | shopping × demographics | 197,112 | 0.0012 | − | 0.0003 | **FAIL — wrong direction (trivial magnitude)** |
| F1.13 | Employed (`VV_WOR_1`) × `VV_MOT_2` (career-orientation) | professional_life × media/attitude | 199,223 | **0.0636** | + | **0.0047** | **PASS (weak) — strongest new pair** |
| F1.14 | `VV_POD_13` (health/fitness podcast) × `VV_FIT_1` (fitness club) | media × health | 199,995 | 0.0137 | + | 0.0002 | WARNING (low magnitude) |
| F1.15 | `VV_SHP_2` (online shopping freq.) × `VV_ADB_1` (ad blocker) | shopping × digital | 133,494 | 0.0025 | + | 0.0000 | WARNING (low magnitude) |
| F1.16 | `VV_TRB_1` (vacation trip) × `VV_LUX_2` ("worth paying extra for quality") | lifestyle/travel × shopping-attitude | 199,993 | 0.0008 | + | 0.0001 | WARNING (low magnitude) |

**Observed:** 14/16 new pairs (87.5%) are flagged (wrong-signed or
low-magnitude), replicating Task 2's original 71% (17/24) and Task 3's
finding, now demonstrated across professional life, shopping, media, auto,
travel, and digital — domains Task 2/3 had not directly tested. Only two
pairs (F1.7, F1.13) have non-trivial NMI, and both are also the two pairs
with the largest raw |r| — **MI and Spearman agree throughout on which pairs
have real structure; MI did not rescue any pair Spearman called near-zero.**

**F1.7 deserves a closer look, and it's genuinely interesting:** a crosstab
of `VV_TRU_1` against `VV_DIG_11` (Fig. `task5_dig11_tru1_nonmonotonic.png`)
shows a **hump-shaped, non-monotonic** relationship — both people who never
follow the news and people who check daily show *lower* agreement that "the
internet is my main news source" (22.1% and 18.5% "Completely Agree") than
people in the middle of the frequency scale (~26% at rank 3-4). Spearman's
r=-0.093 is driven by the high end and mischaracterizes this as a simple
downward trend. **Assumption/verification:** this was checked against the
MI-is-a-finite-sample-biased-upward-estimator concern by inspecting the
full crosstab, not just the summary statistic — the pattern is a smooth,
interpretable hump across all 7 categories at n=138,722, not a single noisy
category, so it reads as genuine structure, not an artifact. By contrast,
F1.13 and the wrong-signed F1.5 (also inspected via crosstab) are both
**cleanly monotonic** — real relationships Spearman already measures
correctly, so their elevated NMI corroborates rather than reveals anything.

### F2 — Subgroup / interaction analysis

8 pairs (both anchors on the demographic fields their own source supports —
age only for Task 2's anchor; province and age for Task 3's anchor — plus 6
of the F1 pairs spanning the widest range of domains/outcomes: F1.13, F1.7,
F1.5, F1.4, F1.10, F1.12) checked across income tercile, province (top 6 by
n, else grouped), household size (`VV_HOV_7`: 1 / 2 / 3+), and age band
(<35 / 35-54 / 55+). A subgroup is flagged if |Δr| > 0.10 vs. the overall r,
or if it reverses sign (with both r's required to exceed 0.02 to avoid
flagging noise around zero); n≥1,000 required per subgroup.

**Observed:** zero of the 6 new pairs × 4 dimensions (24 checks) crossed the
formal |Δr| > 0.10 threshold — full detail (all 24 × subgroup-level rows) is
in the notebook. The near-zero (or modest) cross-domain relationships found
in F1 are not masking a strong subgroup-specific effect hiding underneath an
averaged-out null; this pattern is fairly uniform across income, province,
household size, and age. Both anchors showed the same pattern on their
available dimensions (no material subgroup differences from their ~0.008–0.014
overall r).

The largest deltas, while under the formal threshold, are still informative:
F1.7 (`VV_TRU_1` × `VV_DIG_11`) swings from r=-0.093 overall to essentially
flat within every age band (-0.0007 to -0.0205) — the closest thing to a
real subgroup effect found here. Rather than over-claim this as a "flagged"
result under the pre-registered cutoff, it's followed up properly as a
joint-dependence trio below (F3.6).

### F3 — Joint-dependence (3+ variables)

7 trios spanning food/drink × lifestyle × health, professional life ×
shopping, media × shopping, auto × environment × demographics, travel ×
professional life × demographics, media × digital × age (F3.6, chosen
*because* F2 surfaced it — not picked in advance), and shopping × digital ×
media. **Method (assumption made explicit):** the standard first-order
partial-correlation formula, `r_AB|C = (r_AB − r_AC·r_BC) / √((1−r_AC²)(1−r_BC²))`,
applied to Spearman correlations — an approximation valid to the extent the
rank-transformed relationships are approximately linear. Every result below
is corroborated independently by recomputing A×B's Spearman r within each
tercile of C.

| # | Trio (A × B \| C) | Domain | n | r(A,B) raw | r(A,B\|C) partial | Conclusion |
|---|---|---|---|---|---|---|
| F3.1 | `VV_ORG_1` × `VV_FIT_1` \| `VV_DIE_1` | food_drink × lifestyle × health | 179,685 | 0.0234 | 0.0233 | No material change — not confounded by diet control |
| F3.2 | `VV_TOT_1` × `VV_ORG_1` \| income | professional_life × shopping | 151,731 | -0.0345 | -0.0338 | No material change — not explained by income |
| F3.3 | `VV_ADI_1` × `VV_SHP_2` \| `VV_DIG_11` | media × shopping | 133,496 | -0.0088 | -0.0082 | No material change — not explained by digital engagement |
| F3.4 | `VV_AUT_1` × `VV_ENV_2` \| income | auto × environment × demographics | 164,248 | -0.0035 | -0.0036 | No material change |
| F3.5 | `VV_TRB_1` × employed \| income | travel × professional_life × demographics | 199,228 | 0.0326 | 0.0299 | No material change |
| F3.6 | `VV_TRU_1` × `VV_DIG_11` \| age | media × digital × age | 138,722 | **-0.0930** | **-0.0028** | **Confound confirmed — age composition drives the pooled result (~97% reduction)** |
| F3.7 | `VV_SHP_2` × `VV_ADB_1` \| `VV_BIN_1` | shopping × digital × media | 86,498 | 0.0001 | 0.0003 | No material change |

**Observed:** 1 of 7 trios (F3.6) shows real joint structure a pairwise test
would miss. The raw r=-0.093 collapses to a partial correlation of -0.0028
(~97% reduction) once age is controlled for, and the tercile-stratified
check agrees independently — within the youngest age tercile the correlation
is essentially exactly zero (-0.0005, n=55,545). **Age composition, not a
direct link between news-checking frequency and internet-as-news-source
agreement, produces the pooled correlation** — a Simpson's-paradox-style
confound, the clearest example in this follow-up of a relationship that
looks real pairwise but isn't. The other 6 trios show no material difference
between the raw and partial correlations — the hypothesized third-variable
explanations do not explain away F1's near-zero pairwise results; those
results were not simply hiding structure a trio analysis would reveal.

### Conclusion — does this support or complicate the "generation artifact" hypothesis?

These three follow-ups **support, and sharpen, the generation-artifact
reading rather than complicating it.** If ARIMA's near-zero cross-domain
correlations were principally a *measurement* artifact — Spearman missing
non-monotonic structure, or a pooled-subgroup average masking real
within-group effects — this follow-up's three purpose-built methods should
have surfaced that structure somewhere across 18 MI pairs, 24 subgroup
checks, and 7 trios. They found exactly one such case (F3.6's age confound),
and it explains a relationship *away* rather than resurrecting one — a
demographic-composition effect, consistent with CLAUDE.md's standing finding
that ARIMA preserves **demographic-driven gradients** reasonably well while
cross-domain **attitude/behavior links collapse toward independence**. The
one non-monotonic pattern confirmed as genuine here (F1.7's news-checking
hump) is itself demographic in origin (age-driven, per F3.6), not evidence
of the richer cross-domain dependency structure the generation-artifact
hypothesis says is missing.

**This does not resolve the open question** — it cannot distinguish "real
population feature" from "synthesis artifact" any more than Task 2/3 could —
but it **does rule out one specific alternative explanation**: that better
statistics (MI instead of Spearman, subgroup-aware analysis, joint-dependence
testing) would have found the missing structure. They did not, across a much
wider span of domains than previously tested. That narrows the open question
rather than complicating it.

No fixes applied to the underlying data anywhere in this scorecard — every
row reports an observed result, not a correction.
