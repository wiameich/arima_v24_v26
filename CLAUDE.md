# ARIMA Vintage Comparison & Data Quality Audit

## ✅ RESOLVED — Intact expert-priors fusion blocker (opened 2026-09-15, resolved 2026-09-21)

**Original problem:** `intact_vv_200k.parquet`'s ID space did not link to
`intact_arima_sim_200k.parquet` or `intact_population_200k.parquet`.
Verified via attribute spot-check (Part 2 Task 3, Part A — see
`outputs/part2_data_quality_scorecard.md` checks 3.1-3.3 for the full
trace): gender crosstab between `intact_arima_sim_200k` and `intact_vv_200k`
was statistically flat (~51%/49% either way, not a real linkage), vs. a
perfect bijection between `intact_arima_sim_200k` and
`intact_population_200k`. `intact_vv_200k` was not simply the general
`all417` pull mislabeled either (only 0.57% raw ID overlap there).

**Why this mattered beyond Task 3 itself:**
`~/reference/arima_fusion/doc/expert_priors_methodology.md` describes the
extended copula as conditioning jointly on `[shared + VV_auxiliary]` to
embed cross-domain correlations into the Intact fusion. That precondition
requires `VV_` auxiliary data to be row-linkable to the same individuals
as the survey/population data.

**Resolution (2026-09-21):** the fix was **not** the originally-guessed
re-pull of `intact_vv_200k` — that file remains a dead end (it sits in an
independently-generated ID space unrelated to the other two Intact files,
and is not simply a mislabeled `all417` pull; this diagnosis stands as
documented above). The actual fix: **`intact_arima_sim_200k`'s `id`
column, offset by `+1`, links directly to `arrima-snowflake/CA_2024H2`'s
own `VV_*` tables** — the same 2024H2 bucket used throughout Part 1 of
this project, not the `arima-clustering-pipeline` bucket `intact_vv_200k`
came from. Verified independently (not taken on assertion), to the same
standard used for every ID-based join in this project — attribute
spot-check, not raw ID overlap:

| Check | Before (`intact_vv_200k`) | After (`id + 1` → `CA_2024H2` `VV_*`) |
|---|---|---|
| Gender crosstab | Flat, ~51%/49% either way (no real linkage) | Perfect bijection, zero crossover (96,917 / 103,083 — identical split to the confirmed `intact_population_200k` link) |
| GEO / postal code | Not checked (no meaningful linkage to check) | 100.0000% exact match (n=200,000) |
| Age correlation | r=0.0061 (near-zero) | r=0.978 vs. `AGENUM`, clean non-crossing age-band diagonal |

Full trace, code, and the resulting Intact correlation pairs (Part D) are
in `notebooks/part2_task3_simulation_coherence.ipynb` (Part A "UPDATE" +
Part D) and `outputs/part2_data_quality_scorecard.md` (checks 3.3b,
3.18-3.23). **Any existing `intact_fused_hybrid.parquet` /
`intact_fused_sync.parquet` output built from the old `intact_vv_200k`
linkage remains suspect and should be re-run using the corrected `id + 1`
→ `CA_2024H2` `VV_*` linkage** — this resolution unblocks that work but
does not retroactively fix output already produced under the broken
linkage.

## Context
We fuse Novo Nordisk/Intact client survey data onto ARIMA's synthetic population.
This project validates ARIMA's data itself, independent of our fusion pipeline —
if ARIMA's foundation is shaky, our fusion inherits the problem.

This connects to an open question from the fusion evaluation work: are the
near-zero cross-domain correlations we see (e.g. attitudes vs behavior) a real
feature of the population, or an artifact of ARIMA's generation process
(Gaussian copula decomposition + IPF max-entropy bias)? Part 2 of this project
is designed to help answer that.

> **Strongest evidence gathered so far, pointing toward "generation
> artifact" (2026-09-15):** Part 2 Task 2's VV_ cross-correlation audit
> (`outputs/part2_data_quality_scorecard.md`, `notebooks/part2_task2_vv_correlation_audit.ipynb`)
> tested 24 variable pairs chosen specifically because the expected
> correlation was obvious from the variable descriptions. **71% (17/24)
> came back flagged** — wrong direction or near-zero magnitude. The
> single strongest data point: `vv_aut_1` (household's auto is insured)
> vs. vehicle ownership came back at r=0.014 — essentially zero, for a
> relationship that's close to a logical necessity (you can't insure a
> car you don't own). Every insurance-vs-ownership, travel-attitude-vs-
> frequency, and luxury-attitude-vs-frequency pair showed the same
> pattern (r in the 0.01-0.03 range). This builds on Task 1's
> income-vs-education result (r=0.12 vs. an expected 0.3-0.5) — together
> these are now the strongest evidence gathered on this open question.
>
> **Not everything is flat, so don't overstate this as "all correlations
> are broken":** 5 of 6 digital-attitude-vs-age pairs held up with real,
> correctly-signed magnitude (r=-0.11 to -0.26), and one within-table
> auto pair (mutually exclusive acquisition channels) showed genuine
> structure (r=-0.19). The pattern that's emerging is specific: ARIMA
> appears to preserve **demographic-driven attitude gradients**
> reasonably well, but **attitude↔behavior relationships and even
> logical-necessity constraints collapse toward statistical
> independence**. That's a more precise (and more useful) finding than
> "the population has no correlation structure" — it points at *which*
> part of the generation process (the cross-domain linking step, not the
> demographic conditioning) is the likely source, if this is confirmed to
> be an artifact rather than a real feature.

> **Task 3 finding — updated 2026-09-15 after the isolated-vs-systematic
> follow-up (see below):** Part 2 Task 3
> (`outputs/part2_data_quality_scorecard.md`,
> `notebooks/part2_task3_simulation_coherence.ipynb`) tested whether
> ARIMA's own downstream Novo survey simulation (not just the raw
> synthetic population) preserves cross-domain signal. It found one
> well-established real-world relationship — dieting behavior vs. stated
> weight-loss goal/percentage — where the simulated correlation is not
> just attenuated toward zero, it is **substantially wrong-signed**
> (r ≈ -0.46 where a real, moderate positive relationship is expected;
> n=199,998; both variables verified against source dictionaries — see
> the notebook for full methodological detail: which two variables
> specifically, expected mechanism and its source, and confidence in the
> estimate).
>
> **Follow-up check (recommended next step, now completed): is this
> isolated or systematic?** Tested 7 further plausible-mechanism pairs,
> sourced directly from the Novo survey's own data dictionary
> (`novo_full_data_set.xlsx`'s `Copy of datamap` sheet — stronger sourcing
> than the label-inference used for the original pairs). Result: **4 of 7
> came back correctly signed with real magnitude**, including `BMI`/
> `Weight (KG)` vs. cardiometabolic conditions (high blood pressure, Type
> 2 diabetes, high cholesterol) at r=0.268 and r=0.208 — **the strongest
> cross-domain correlations found anywhere in this entire Part 2 audit**,
> for one of the most textbook-established mechanisms in epidemiology.
>
> **Revised conclusion: the dieting/weight-loss reversal looks isolated,
> not systematic.** It does not generalize to other tested mechanisms.
> ARIMA's Novo simulation shows a genuinely mixed picture — real,
> correctly-signed (even moderately strong) signal for some cross-domain
> mechanisms, near-zero signal for others (consistent with Task 2's
> general pattern), and one substantial, real reversal that appears to be
> a localized defect in that specific pair rather than a general property
> of the simulation. This means neither prior research doc
> (`media_wl_correlation_research.md`'s "near-zero is the genuine answer"
> vs. `vv_dem1_hub_research.md`'s "near-zero is a 5-15x-attenuated
> artifact") is cleanly contradicted by Task 3 — the picture is mechanism-
> dependent, not a single verdict either doc's framework anticipated.

## Data locations
- 2024 vintage: `gs://arrima-snowflake/CA_2024H2/` (confirmed canonical source
  — supersedes the earlier `arima-clustering-pipeline/data/200K/` bucket)
- 2026 vintage: `gs://plusco-arima-data-dropzone-prod/ca/2026/synthetic-population/`
- Variable mapping: `gs://plusco-arima-data-dropzone-prod/ca/2026/variable_mapping.csv`

Prior observation to verify, not assume: old 2024 200K files reportedly have
zero ID overlap with 2026 intact-survey shards, but DO overlap with 2026
`dem/` files.

> **STATUS: unverified / superseded, not confirmed.** Direct testing (see
> "ID stability across vintages" below) found the opposite under raw ID-value
> comparison — 100% overlap with both `dem/` and `intact-survey` — and further
> found that raw ID-value overlap is not a meaningful test at all given how
> ARIMA assigns IDs. This original claim is kept here as history, not as an
> established result. If it needs to be re-checked, use the GEO/attribute
> spot-check method below, not raw ID-value comparison.

## ID stability across vintages — CONFIRMED FINDING (2026-09-10)
`id` in ARIMA exports is a **dense sequential index assigned at generation/
export time, not a stable person identifier carried across vintages.**

- Evidence: `VV_ACN` (2024H2, `arrima-snowflake`) and `dem/` (2026,
  `plusco-arima-data-dropzone-prod`) both have `id` columns that are
  perfectly contiguous, gap-free integer ranges (e.g. `VV_ACN` = 1..32,433,918
  with zero gaps; `dem` = 1..33,597,827 with zero gaps) — the signature of a
  row-generation-time index, not a persistent key.
- Direct test: pulled the full `id` sets from both, found the smaller range
  (2024H2) 100% contained in the larger (2026) — then spot-checked `GEO`
  (postal code) for 20 randomly sampled "matching" IDs. **19 of 20 pointed to
  different postal codes / provinces** across the two vintages (e.g. id
  19236114 = `N0K1N0`, Ontario, in 2024H2 vs. `M9C3J1`, Toronto, in 2026). The
  1 apparent match was in a low-cardinality rural FSA and is plausibly a
  placeholder-geo collision, not genuine identity. Sample size: n=20 IDs,
  drawn from a population of 32,433,918 shared integer values.
- **Conclusion:** the same `id` value refers to different synthetic
  individuals across vintages. Raw ID equality is not evidence of shared
  identity between two ARIMA exports.

**Structural reason any raw ID-overlap % will look inflated:** because IDs
are dense, gap-free integer ranges assigned per export, any smaller export's
ID range is close to trivially contained within a larger export's ID range
covering an overlapping numeric span. A 100% (or near-100%) raw overlap
number between two ARIMA exports is therefore expected by construction and
is **not, by itself, evidence of shared identity** — it must be corroborated
with an attribute-level spot-check (e.g. GEO, demographics) before being
treated as a real finding, the way the check above was done.

**Affects Part 2 too, not just Part 1.2:** any join across two ARIMA exports
keyed on `id` — including within Part 2's VV_ cross-correlation audit or
simulation-vs-population coherence checks, if those ever compare across
vintages or across independently-generated exports — needs this same
caveat. Do not assume `id` equality implies same-person without a spot-check
like the one above.

## Table/variable-content stability across vintages — CONFIRMED FINDING (2026-09-14)
`table_id` is **also not a stable content key across vintages** — a
separate problem from the `id`-column finding above (that one was about
row identity; this one is about what a table/variable *means*).

- Evidence: comparing the 2024H2 and 2026 official variable dictionaries
  (`VARIABLE_MAPPING`), of 332 table_ids present by name in both vintages,
  **130 (39%) have zero content overlap** (Jaccard similarity of variable
  descriptions = 0.0) — the same table_id holds unrelated content in each
  vintage. (Corrected 2026-09-22: this was originally recorded as "164
  (49%)," which was wrong — re-derived independently from both source
  dictionaries and confirmed against the cached self-similarity data; see
  `outputs/part1_vintage_comparison.md` Part 1.1 for the correction note
  and the full 130-table remapping.) Many of these have a **perfect
  1.000-similarity match to a differently-named table** in the other
  vintage (e.g. 2024H2 `vv_hov` "Presence Of Children <18" content moved to
  2026 `vv_how`, while `vv_hov` in 2026 now means spray-bottle purchases).
  The remapping is **not a clean 1:1 rename** (e.g. `vv_con`→`vv_coo` but
  `vv_cow`→`vv_con`), so a simple find-and-replace table-prefix fix will not
  work.
- Not universal: `vv_dem`, `vv_res`, `vv_lux` (and presumably others not yet
  checked) are fully stable (1.000 similarity). Category-level values are
  also stable where checked (`vv_dem_1` age brackets are byte-identical).
  So this must be checked per-table, not assumed either way.
- **Direct, verified impact on this project's own fusion pipeline**
  (`~/reference/arima_fusion`): of 10 `VV_*` tables hardcoded in
  `harmonize_shared_vars.py` / `expert_priors.py`, only 3 (`vv_dem`,
  `vv_res`, `vv_lux`) are stable; 6 changed content (`vv_hov`, `vv_mar`,
  `vv_hon`, `vv_lif`, `vv_auu`, `vv_fio`) and 1 (`vv_shr`) was removed
  entirely. See `outputs/part1_vintage_comparison.md` Part 1.1 for the full
  table and methodology.
- **Correction (2026-09-24): `vv_res` is not actually safe, despite passing
  this table-level check.** Two independent, later checks both fail it: Part
  1.3 found all 39/39 of its variables have a different category/response
  structure across vintages (a 2024H2 4-category scale collapsing to a
  2026 3-category one), and the Part 1.1 variable-level description-diff
  extension (`outputs/part1_vintage_comparison.md`, "Part 1.1 (extension)")
  found all 39/39 of its variables are also internally **reshuffled** —
  same overall set of 39 descriptions in both vintages (hence the 1.000
  table-level score above), but each `var_id` number points at a different
  one of those 39 questions in 2026 than it did in 2024H2. Table-level
  description-set stability (this bullet's original claim) does **not**
  imply a table is safe to reference by `var_id` — it only means the
  table's total content didn't change, not that any specific variable
  kept its meaning. Effectively **2 of the 10** hardcoded tables
  (`vv_dem`, `vv_lux`) are verified safe as-is; `vv_res` needs the same
  explicit remapping treatment as the 6 already-known-unstable tables.
- **Conclusion:** any 2024→2026 vintage switch needs an explicit, verified
  table/variable remapping step. Do not assume a `table_id`/`var_id` string
  carries the same meaning across vintages just because the string itself
  is unchanged — check content (e.g. via description-set overlap) the way
  Part 1.1 did, the same way `id`-column joins need the GEO/attribute
  spot-check from the finding above.

## Deliverables
1. `outputs/part1_vintage_comparison.md` — schema diff, ID continuity, distribution
   shifts, variable mapping changes between 2024 and 2026. Flag anything that
   would break the existing fusion pipeline if we switched vintages.
2. `outputs/part2_data_quality_scorecard.md` — pass/fail/warning table covering
   demographic sanity checks, VV_ cross-correlation audit (20-30 pairs), simulation
   vs population coherence, and missingness patterns. Meant to be reusable for
   future client engagements.

## Agent instructions
Before implementing analysis code:
1. Inspect the repository and existing files.
2. Verify access to the GCS paths and identify the actual file/table structure.
3. Inspect any existing `combined_200k`, simulation, or related analysis files.
4. Do not assume file formats, column names, schemas, or ID locations.
5. Verify the known ID-overlap finding rather than treating it as established fact.
6. Propose an implementation plan for the current sequencing stage before making
   substantial changes.
7. Do not modify raw data.
8. Do not commit raw data, credentials, tokens, or large generated datasets to Git.
9. Keep analysis reproducible and record sample sizes for statistical results.
10. Clearly distinguish:
    - observed results
    - methodological assumptions
    - expert expectations
    - conclusions

## Data source decision (2024 baseline)
Confirmed 2024 source: `gs://arrima-snowflake/CA_2024H2/`. This supersedes the
earlier tier1/tier3 GCS parquet approach below — use this as the canonical
2024 baseline going forward.

Note from earlier investigation (kept for context): neither
`combined_200k_new_tier1.parquet` nor `combined_200k_new_tier3.parquet` carries
a real ID/identifier column, so they cannot anchor the Part 1.2 ID-continuity
check regardless. If they're still used for schema/distribution comparisons,
any row-aligned (positional) join between them rests on an unverified
assumption unless independently checked — flag any output derived that way.

For Part 1.2 (ID continuity vs. 2026), use `arima_200k_all417_manifest.csv`'s
actual IDs as the anchor, not tier1/tier3.

## Working conventions
- Cache GCS pulls locally under `data/` (gitignored) — don't re-fetch on every run.
- Correlation audits: report both direction and magnitude, and flag expected-vs-
  actual mismatches explicitly rather than just listing numbers.
- Keep scorecard entries atomic (one check = one row) so it can be extended later.
- Note sample sizes alongside any correlation — some VV_ variables may be sparse
  (68 health_and_wellness tables were already excluded for this reason; check for
  others in Part 2 Task 4).
- Plots: side-by-side histograms for distribution shift comparisons, saved to
  `outputs/figures/`.

## Sequencing (don't run top-to-bottom by task number)
1. ID continuity (Part 1.2) — gates interpretation of everything downstream
2. Schema diff + variable mapping diff (Part 1.1, 1.4)
3. Distribution shifts (Part 1.3)
4. Missingness audit (Part 2.4) — informs which VV_ pairs are worth testing
5. Demographic sanity checks (Part 2.1)
6. VV_ cross-correlation audit (Part 2.2)
7. Simulation vs population coherence (Part 2.3) — compare against 2.2's baseline