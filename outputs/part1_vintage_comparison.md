# Part 1: Vintage Comparison — 2024 vs 2026

> ## ⚠️ Critical finding — do not switch vintages without reading this
>
> Our fusion pipeline pulls specific pieces of ARIMA data by name (things
> like "table VV_HOV" or "table VV_SHR") and assumes those names always
> mean the same thing. They don't. We checked the 10 ARIMA tables our
> pipeline actually relies on, and **7 of them either now mean something
> completely different in the 2026 data, or don't exist anymore at all** —
> for example, the table that used to hold "does this household have
> children" now holds spray-bottle purchase data instead, and the real
> household/children data quietly moved to a different table name. Only 3
> of the 10 are still safe to use as-is. If we switched the pipeline over
> to 2026 data today without fixing this, it would not throw an error and
> stop — it would keep running and **silently produce wrong fusion
> results**, because in most cases the old table name still exists, it
> just now points to unrelated data. This has to be fixed with an explicit,
> checked mapping before any 2026 cutover — it cannot be fixed by a simple
> find-and-replace, because the renaming isn't a clean one-to-one swap.
> Full detail and the table-by-table evidence is in Part 1.1 below.

Status: **Parts 1.2 (ID continuity), 1.1 (schema diff), 1.4 (variable
mapping diff), and 1.3 (distribution shifts) complete.** Per the project's
sequencing, ID continuity gated interpretation of everything downstream and
was addressed first; distribution shifts (1.3) were addressed last since
they build on 1.1's per-table stability check.

## Data sources used

- **2024 vintage:** `gs://a
rrima-snowflake/CA_2024H2/` — a Snowflake unload
  export. 417 `VV_*` topic tables (long format: `ID, GEO, VV_<TOPIC>_<n>`),
  plus `LOC` (geography dimension table, no person ID) and
  `VARIABLE_MAPPING` (variable dictionary, no person ID).
  - Also inspected, but **not** used as the anchor: `gs://arima-clustering-pipeline/data/200K/`
    (an earlier, superseded candidate 2024 source — see "Superseded
    investigation" below), specifically `arima_200k_all417_decoded.parquet`
    / `arima_200k_all417_raw.parquet` (200,000 rows, 11,879 columns, real
    unique `id` column, range 284–32,433,836) and
    `combined_200k_new_tier1.parquet` / `combined_200k_new_tier3.parquet`
    (200,000 rows each, one-hot encoded, **no ID column at all**).
- **2026 vintage:** `gs://plusco-arima-data-dropzone-prod/ca/2026/`
  - `synthetic-population/dem/`: 27 shards, 33,597,827 rows total, columns
    `id, geo, vv_dem_1..20`.
  - `intact-survey/`: 250 shards, 32,433,918 rows total, columns
    `id, geo, QLANG, PROV, age, ..., ` (~212 columns, the fused client
    survey).

## Part 1.2 — ID continuity

### Documented prior claim (history — superseded, not confirmed)

CLAUDE.md originally recorded: *"old 2024 200K files reportedly have zero ID
overlap with 2026 intact-survey shards, but DO overlap with 2026 `dem/`
files."* This was **not verified** — it was a prior/expert expectation to be
checked, per the project's own agent instructions. Investigation below found
the opposite under raw ID-value comparison, and then found raw ID-value
comparison to not be a meaningful test at all.

### Observed results

**1. Raw ID-value overlap (all417, the 200K anchor, vs. 2026 sources):**

| Comparison | Result | n |
|---|---|---|
| all417 ∩ `dem` | 200,000 / 200,000 (100%) | 200,000 |
| all417 ∩ `intact-survey` | 200,000 / 200,000 (100%) | 200,000 |

**2. Raw ID-value overlap (`VV_ACN`, a 2024H2 `arrima-snowflake` topic table,
vs. `dem`):**

| Comparison | Result | n |
|---|---|---|
| VV_ACN ∩ dem | 32,433,918 / 32,433,918 (100% of VV_ACN) | 32,433,918 |
| dem ids not in VV_ACN | 1,163,909 | — |

Both results initially looked like strong "overlap," consistent with the
prior claim about `dem`. But structural inspection of the ID columns showed
why this is misleading (see next section).

**3. Why raw ID-overlap is structurally inflated, not informative:**

Every ID column checked across both vintages (`all417`, `VV_ACN`, `dem`,
`intact-survey`) is a **dense, gap-free, sequential integer range** — e.g.
`dem` = 1..33,597,827 with zero gaps across all 27 shards; `VV_ACN` (2024H2)
= 1..32,433,918 with zero gaps across all 16 shards; `intact-survey` (2026)
= 0..32,433,917 with zero gaps across 250 shards. Given this, any smaller
export's ID range is close to trivially contained within a larger,
overlapping-range export — a 100% (or near-100%) raw overlap number is
**expected by construction**, regardless of whether the IDs refer to the
same people.

**4. Direct identity spot-check — GEO (n=20 IDs, VV_ACN vs. dem):**

Sampled 20 IDs present in both `VV_ACN` (2024H2) and `dem` (2026) and
compared the `GEO` (postal code) field for each. **19 of 20 (95%) pointed to
different postal codes / provinces** (e.g. id 19236114 = `N0K1N0`, rural
Ontario, in 2024H2 vs. `M9C3J1`, Toronto, in 2026). The one match
(`B0T1K0`, a low-population rural Nova Scotia FSA) is plausibly a
placeholder-geo collision rather than genuine identity — low-cardinality
rural postal assignment is a known source of spurious matches.

**5. Direct identity spot-check — demographics (n=200,000 IDs, all417 vs.
dem and intact-survey, full sample not just a spot-check):**

For all 200,000 `all417` IDs found in both `dem` and `intact-survey`,
compared decoded age (`VV_DEM_1`, 12 brackets) and sex (`VV_DEM_2`) against
`dem`'s raw age/sex codes, and against `intact-survey`'s `age`/`SEX2`
fields, for the *same* ID.

- **Age:** every `dem` age code shows the same ~14–15% share landing in
  `all417`'s "70+" bracket — matching the population base rate for that
  bracket, not a real per-code distinction. If `id` meant the same person,
  each `dem` code should map overwhelmingly to one `all417` age bracket
  (a near-diagonal contingency table); instead the distribution is flat
  across all codes.
- **Sex:** `dem` sex code 21440013 → all417 Female 50,552 / Male 47,527
  (~52%/48%); code 21440014 → Female 52,199 / Male 49,722 (~51%/49%).
  Statistically indistinguishable — no signal at all.
- Same pattern holds for `intact-survey`'s `age` (7 buckets) and `SEX2`:
  proportions are flat across all `all417` age brackets, and sex splits
  ~50/50 regardless of the paired `all417` sex value.

This is a **full-population check (n=200,000), not a small sample** — the
absence of any diagonal/concentration pattern is a strong statistical
signature of independence, i.e. the same `id` value refers to unrelated
individuals across the two sources.

### Methodological assumptions

- Treated `GEO` (postal code) and decoded age/sex as reasonable proxies for
  "same real individual" — collisions are possible (shared postal code in
  low-density areas; shared age/sex by chance) but not at the rate observed
  (95% GEO mismatch on n=20; complete statistical independence on age/sex at
  n=200,000).
- Assumed `dem`'s `vv_dem_1`/`vv_dem_2` and `all417`'s `VV_DEM_1`/`VV_DEM_2`
  encode the same underlying variables (age bracket, sex) — supported by
  matching category cardinalities (12 and 2 respectively) between the two,
  though the manifest for `all417` did not document `VV_DEM_*` at all (a
  separate documentation gap, noted for Part 2).

### Conclusion

**`id` in ARIMA exports is a dense sequential index assigned at
generation/export time — not a stable, cross-vintage person identifier.**
Any raw ID-value overlap between two independently-generated ARIMA exports
(across vintages, or potentially even across two exports of the same
vintage) is expected to be high by construction and is **not evidence of
shared identity** on its own. The originally documented prior finding (zero
overlap with intact-survey, overlap with dem) does not hold under direct
testing and should be treated as unverified/superseded history, not as an
established result — see the correction note left in place in CLAUDE.md.

### Consequence for the fusion pipeline and Parts 1.3/1.4/Part 2

- **Any join across two ARIMA exports keyed on `id`, expecting it to mean
  "same person," is silently wrong.** This applies to the existing fusion
  pipeline if it performs such joins, and to any future work in this
  project (Part 2's VV_ cross-correlation audit and simulation-vs-population
  coherence checks included) that might compare across vintages or across
  independently-generated exports.
- Within a *single* export/vintage, `id` still appears to function correctly
  as a row-level key (e.g. `all417`'s `id` is unique and consistent between
  its own `raw` and `decoded` files; `dem`'s `id` is unique across its own
  27 shards). The stability problem is specifically about carrying `id`
  *across* separately-generated exports.
- Parts 1.3 (distribution shifts) and 1.4 (variable mapping diff) can
  proceed using column-level/aggregate comparisons (which don't require
  row-level ID linkage), but any planned row-level 2024-to-2026 join should
  be reconsidered or explicitly caveated.

## Part 1.1 — Schema diff (2024H2 vs. 2026)

### Data sources

Both vintages' official variable dictionaries, already cached locally:
- 2024H2: `data/CA_2024H2/VARIABLE_MAPPING/` (33,020 rows, columns
  `THEME, TABLE_ID, TABLE_NAME, VAR_ID, DESCRIPTION, VAR_VALUES,
  NATIONAL_COUNT, VALUE_ID`).
- 2026: `data/variable_mapping_2026.csv` (34,234 rows, same fields
  lowercased, plus `dataset`).

### Headline finding: `table_id` is not a stable content key across vintages

**Observed:** 417 `VV_*`-style table_ids appear in the 2024H2 dictionary;
411 table_ids appear in the 2026 dictionary (408 `VV_*` tables plus 3
non-`VV_` administrative entries: `All` — the population-total sentinel row
documented earlier — `loc`, and `pri`). 332 table_ids are present, by name,
in both vintages.

Of those 332 **same-named** tables, a description-set overlap check
(Jaccard similarity of each table's variable descriptions between the two
vintages) found:

**164 of 332 (49%) have a self-similarity of exactly 0.0** — the same
`table_id` refers to entirely different survey content in 2026 than it did
in 2024H2. This is not a formatting artifact: many of these tables have a
**perfect 1.000-similarity match to a differently-named table** in the
other vintage, which would not happen if the mismatch were just inconsistent
text formatting.

**Concrete example — `vv_hov` / `vv_how`:**

| | 2024H2 `vv_hov` | 2026 `vv_hov` | 2024H2 `vv_how` | 2026 `vv_how` |
|---|---|---|---|---|
| Content | Household composition / presence of children (19 vars, e.g. `vv_hov_6` = "Presence Of Children <18", `vv_hov_7` = "Total # of people in hhld") | Spray bottle purchases (2 vars) | Electronics purchase location/brand (Best Buy, Costco, Bose, Samsung...) (22 vars) | Household composition / presence of children (19 vars, e.g. `vv_how_10` = "Presence Of Children <18", `vv_how_11` = "Total # of people in hhld") |

The household-composition content **moved from `vv_hov` to `vv_how`**
between vintages, while both table_ids continued to exist, now holding
unrelated content. This is a rename-plus-reuse pattern, not a simple
addition/removal.

**Other confirmed examples (table_id → best-matching differently-named
table, by description overlap):**

| 2024H2 table_id | 2026 table_id | Jaccard |
|---|---|---|
| `vv_con` | `vv_coo` | 1.000 |
| `vv_mob` | `vv_mod` | 1.000 |
| `vv_cat` | `vv_cav` | 1.000 |
| `vv_cou` | `vv_cow` | 1.000 |
| `vv_cow` | `vv_con` | 1.000 |
| `vv_tra` | `vv_trd` | 1.000 |
| `vv_den` | `vv_dep` | 1.000 |
| `vv_for` | `vv_fot` | 1.000 |
| `vv_hou` | `vv_hox` | 0.840 |
| `vv_ono` | `vv_onn` | 0.933 |
| `vv_cog` | `vv_cof` | 0.911 |
| `vv_rea` | `vv_reb` | 0.800 |
| `vv_dal1` | `vv_dai3` | 0.567 |

Note the non-bijective pattern (`vv_con`→`vv_coo` but `vv_cow`→`vv_con`):
table_ids were reassigned in a way that isn't a clean 1:1 rename mapping,
which rules out a simple "find-and-replace the table prefix" fix.

**Not every table is affected.** Spot-checked against the specific tables
this project's fusion pipeline hardcodes (see "Consequence" below):
`vv_dem`, `vv_res`, and `vv_lux` all have self-similarity = **1.000**
(fully stable content). Category-level values are also stable where the
table itself is stable — e.g. `vv_dem_1` (age) has **identical** category
labels in both vintages (verified via full set comparison, not sampled).

### Consequence for the fusion pipeline (direct, verified impact)

Checked self-similarity specifically for every `VV_*` table hardcoded in
`~/reference/arima_fusion` (`scripts/data_prep/harmonize_shared_vars.py`
and `scripts/novo/expert_priors.py` / `prefill_expert_priors.py`):

| Table | Used for | 2024H2→2026 status |
|---|---|---|
| `vv_dem` | age/gender/income/ethnicity (core shared-variable harmonization) | **Stable** (1.000) |
| `vv_res` | visit-frequency bands | **Stable** (1.000) |
| `vv_lux` | luxury/premium product ownership | **Stable** (1.000) |
| `vv_hov` | `VV_HOV_6` "Presence of Children <18" (hardcoded shared variable) | **Content moved to `vv_how`** — self-similarity 0.0 |
| `vv_mar` | marital status (documented shared-variable prefix) | **Changed** — self-similarity 0.0 |
| `vv_hon` | home/insurance (expert prior pattern `VV_HON\|VV_LIF\|VV_AUU`) | **Changed** — self-similarity 0.0 |
| `vv_lif` | lifestyle (same expert prior pattern) | **Changed** — self-similarity 0.0 |
| `vv_auu` | (same expert prior pattern) | **Changed** — self-similarity 0.0 |
| `vv_fio` | finance/financial attitudes (expert prior pattern `VV_FIO\|VV_SHR`) | **Changed and shrunk** — self-similarity 0.0, 40→1 variables |
| `vv_shr` | shopping/household (same expert prior pattern) | **Removed entirely** — table_id does not exist in 2026 |

**Of the 10 hardcoded table references checked, 3 are stable and 7 are
either semantically changed or removed.** If this fusion pipeline were
pointed at 2026 data using its current hardcoded `VV_*` references without
remapping, most of its cross-domain expert-prior matching and shared-variable
harmonization would either silently operate on the wrong data (for tables
where content moved to a *different but still-existing* table_id, so no
error is raised) or fail outright (`vv_shr`, `KeyError`-style failures for
now-empty/moved variables like `vv_hov_6`). This is exactly the kind of
vintage-switch risk this deliverable is meant to surface, and it is
independent of the ID-stability finding in Part 1.2 — this is a schema/
content-mapping problem, not an identifier problem.

### Methodological assumptions

- Used variable `description` text as the content fingerprint for each
  table (set of distinct descriptions, Jaccard similarity). This assumes
  descriptions are a faithful proxy for "what the table measures" — a
  reasonable assumption given the descriptions are human-readable variable
  labels, not IDs, but not a substitute for full column-value verification
  of every table (only spot-checked, not exhaustive).
- Did not exhaustively verify every one of the 164 flagged tables' new
  location — the "best match" search (by Jaccard against all 411 2026
  tables) was run for a subset (15) as a demonstration of the pattern, not
  a complete remapping table.

## Part 1.4 — Variable mapping diff (2024H2 vs. 2026)

### Observed results

| | 2024H2 | 2026 | Shared | Only 2024H2 | Only 2026 |
|---|---|---|---|---|---|
| Tables (`table_id`) | 417 | 411 (408 `VV_*` + 3 non-VV) | 332 | 85 | 79 |
| Variables (`var_id`) | 11,878 | 10,943 | 5,674 | 6,204 | 5,269 |

- **6,204 variables present in 2024H2 no longer exist (by that `var_id`) in
  2026**; 1,994 of those belong to tables that *still exist* in 2026 under
  the same `table_id` (e.g. `vv_hov_3`..`vv_hov_19`, the household/children
  variables that moved to `vv_how` per Part 1.1 above — these look "removed"
  under `vv_hov` specifically, but the content persists under a different
  table_id).
- **5,269 variables in 2026 have no 2024H2 counterpart**; 1,582 of those
  belong to tables that already existed in 2024H2 (e.g. `vv_con_25`
  through `vv_con_44`, new music/TV/radio attitude items added within an
  existing table).
- **Category-level stability, where checked:** for `vv_dem_1` (age), the
  full set of category labels is byte-identical between vintages (12
  brackets, "18 - 19" through "70+"). This is the one variable checked in
  depth for category drift; it was chosen because Part 1.2 and Task 1 both
  already depend on it being stable, which this confirms. Category drift
  for other shared variables was not exhaustively checked and should not be
  assumed absent.

### Conclusion

The 2024→2026 vintage change is **not** a simple additive schema evolution
(new tables/variables layered on top of a stable core). A substantial
fraction of tables were restructured — content reassigned across table_ids
in a non-bijective way — while variable dictionaries were also updated
independently. Any process that maps 2024-vintage `VV_*` references onto
2026 data (this fusion pipeline, or any future one) needs an explicit,
verified table/variable remapping step; it cannot assume `table_id`/`var_id`
names carry the same meaning across vintages, even when the id string
itself is unchanged.

## Part 1.3 — Distribution shifts (2024H2 vs. 2026)

Notebook: `notebooks/part1_3_distribution_shifts.ipynb`. Figures:
`outputs/figures/part1_3_{vv_dem,vv_res,vv_lux,vv_edu,vv_die,vv_dig}_distribution.png`.

### Scope

Covers a representative subset of 6 tables confirmed content-stable
(self-similarity = 1.000) in Part 1.1 — `vv_dem`, `vv_res`, `vv_lux`,
`vv_edu`, `vv_die`, `vv_dig` — chosen for type diversity (demographics,
binary flags, Likert attitudes, categorical), not exhaustiveness. This is
99 variables out of the 135 tables Part 1.1 confirmed stable; it is not a
complete sweep of every stable table.

### Data sources & sampling methodology (deviation from a full-population comparison)

- 2024H2: `data/all417/arima_200k_all417_decoded.parquet`, a 200,000-row
  sample, already decoded to category labels. Read column-pruned to the
  100 columns these 6 tables need (down from 11,879 total columns in the
  file).
- 2026: `data/part1_3_2026/{res,lux,die,dig,edu}.parquet` and 27
  `data/dem_2026/*.parquet` shards are each the **full 33,597,827-row 2026
  population**, not a sample.

**A first execution attempt loaded these full 2026 tables directly into
pandas and exhausted memory** (an 18GB machine; `res.parquet` alone is
~33.6M rows × 40 `int64` columns, on the order of 10GB once materialized,
and six such tables were being loaded at once). **Fix, and the deviation
from a full-population comparison that results:** each 2026 parquet file
is now read via `pyarrow`'s row-group `iter_batches` streaming interface
and reduced to a random ~200,000-row sample per table — matching the
2024H2 side's order of magnitude — so the full population is never
materialized in memory. Realized sample sizes: 199,520 rows for `res`,
`lux`, `die`, `dig`, `edu`; 200,715 rows for `dem` (sampled proportionally
across its 27 shards). Per-variable `n_2024`/`n_2026` are recorded
alongside every result row.

### Methodology

For each variable: build the non-blank category distribution for both
vintages (`__blank__`/true-NaN excluded from the shape comparison, blank
rate reported separately). A `categories_match` check (set equality of
category labels between vintages) gates the verdict — **a variable having
identical description text in both dictionaries (Part 1.1's basis for
"stable") does not guarantee its response scale is unchanged**, so
variables whose category sets differ are reported as schema mismatches,
not as distribution shifts, regardless of their raw statistics. For
variables that do pass this check: chi-square test of independence plus
**Cramér's V** as the flagging criterion (V > 0.1), since p-values alone
are close to meaningless for judging a real effect at this n.

### Observed results

**Headline: 99 variables checked — 27 OK, 26 flagged distribution shifts,
46 schema mismatches (not comparable).**

| Table | n vars | OK | Flagged shift | Schema mismatch |
|---|---|---|---|---|
| `vv_dem` | 20 | 18 | 1 | 1 |
| `vv_res` | 39 | 0 | 0 | **39 (100%)** |
| `vv_lux` | 15 | 2 | 13 | 0 |
| `vv_edu` | 5 | 1 | 0 | 4 |
| `vv_die` | 9 | 5 | 2 | 2 |
| `vv_dig` | 11 | 1 | 10 | 0 |
| **Total** | **99** | **27** | **26** | **46** |

**Schema mismatches (46) — overlap with Part 1.1's known-unstable tables:
none. All 46 are a new finding, not a rediscovery.** Every one of these 46
variables belongs to one of the 6 tables Part 1.1 explicitly confirmed at
self-similarity = 1.000 (re-verified again at the top of this notebook,
independently). Part 1.1's 164-table "unstable" list is a **table-level**
description-overlap check; it says nothing about whether a table's
individual variables kept the same *response scale* (category/value
structure) across vintages. This notebook checks that different, narrower
axis, and finds it fails often even for tables Part 1.1 called stable:

- **`vv_res`: all 39/39 variables are schema mismatches** — the entire
  table's response scale changed (mostly a 2024H2 4-category structure
  collapsing to a 2026 3-category one; `vv_res_1` goes 2→3,
  `vv_res_18` goes 4→2) even though the table's own description-set
  Jaccard was 1.000. This is a systematic, table-wide re-binning that a
  description-only check cannot see at all.
- **`vv_edu`: 4/5 variables mismatch**, with inconsistent (non-monotonic)
  category-count changes (`vv_edu_2` 10→2, `vv_edu_3` 2→4, `vv_edu_4`
  4→7, `vv_edu_5` 7→10) — not a single consistent re-binning pattern like
  `vv_res`.
- **`vv_dem`: 1/20 mismatches** (`vv_dem_5`: 12→11 categories, blank rate
  0.00%→3.88%) and **`vv_die`: 2/9 mismatches** (`vv_die_2`: 1→2
  categories with blank rate 27.73%→0.00%; `vv_die_9`: 2→1 categories
  with blank rate 0.00%→27.94% — the mirror-image pattern of `vv_die_2`,
  consistent with a category boundary moving relative to the blank
  sentinel in each case, not independently verified further).
- **`vv_lux` and `vv_dig` had zero schema mismatches** — every variable in
  both tables kept the same category set across vintages.

**Flagged distribution shifts (26) — among variables where the schema is
genuinely comparable:**

| Table | Flagged / comparable | Cramér's V range |
|---|---|---|
| `vv_lux` | 13 / 15 (87%) | 0.16 – 0.62 |
| `vv_dig` | 10 / 11 (91%) | 0.11 – 0.45 |
| `vv_die` | 2 / 7 (29%) | 0.16, 0.20 |
| `vv_dem` | 1 / 19 (5%) | 0.25 |
| `vv_edu` | 0 / 1 (0%) | — |
| `vv_res` | 0 / 0 — no comparable variables remain | — |

- **`vv_lux` (luxury/lifestyle attitudes) and `vv_dig` (digital
  attitudes) are the two tables with the largest, most pervasive real
  shifts** — the large majority of variables in both moved (Cramér's V
  up to 0.62 for `vv_lux_12`, 0.45 for `vv_dig_1`), not an isolated
  variable or two.
- The one `vv_dem` shift (`vv_dem_7`, V=0.2465) coincides with a large
  blank-rate change (53.41%→82.26%) — the shift may be substantially
  driven by a coverage/response-rate change rather than a shift among
  respondents who did answer; not disentangled further here.
- `vv_res` has **zero** variables where a genuine shift-vs-stable
  judgment is even possible, because every one of its variables failed
  the schema-match gate — the schema mismatch finding above pre-empts
  any distribution-shift read for this entire table.

### Methodological assumptions

- **Sampling adequacy:** a ~200,000-row random sample per 2026 table is
  assumed to represent that table's true category proportions closely
  enough for this comparison; not validated against the full 33.6M-row
  population for these specific tables (doing so was the original,
  OOM-causing approach). Given the category counts involved (2–17
  categories) and this sample size, sampling error on category
  proportions is expected to be small relative to the effect sizes
  Cramér's V > 0.1 is built to catch, but this has not been independently
  confirmed by comparing sample-based proportions against full-population
  proportions for any of these tables.
- **Sampling representativeness:** per-shard sampling for `dem` (a fixed
  target row count per shard, proportional to nothing but shard count)
  assumes the 27 shards are not systematically different from one another
  in composition; not verified.
- Cramér's V > 0.1 as the flagging threshold is a convention carried over
  from Part 2's cross-correlation audit, chosen to separate "trivial/
  noise-level" from "real" shifts at large n, not a domain-specific
  cutoff derived for this data.
- `categories_match` uses exact set equality of category labels; a
  variable where only the *order* or *encoding* of an unchanged category
  set differs, or where category text differs by formatting alone (not
  checked for), would be misclassified as a schema mismatch rather than a
  true structural change. Not spot-checked for false-positive mismatches
  the way Part 1.1's table-level renames were.

### Conclusion

**Table-level content stability (Part 1.1) and variable-level response-scale
stability (this section) are independent properties, and passing the
former does not imply the latter.** All 6 tables checked here were
selected specifically because Part 1.1 confirmed them content-stable, yet
46 of 99 variables (46%) — including the entirety of `vv_res` — turned
out to have a different category structure in 2026 despite an unchanged
variable description. Any process (this project's fusion pipeline
included) that treats a Part-1.1-style description match as sufficient
evidence a variable is safe to carry across vintages would be wrong for
nearly half of the variables checked here.

Separately, **even among variables that do pass the schema-match check,
real distribution shifts are common and often large** — most pronounced
in the two attitude/Likert tables (`vv_lux`, `vv_dig`), where the large
majority of variables shifted with moderate-to-large effect sizes. This
is a genuine population/measurement change to account for, distinct from
the schema-mismatch problem above, and it means `vv_lux` and `vv_dig`
specifically should not be treated as drop-in stable across a 2024→2026
vintage switch even though Part 1.1 found no *content* change in either
table.

## Superseded investigation (kept for history)

Before `gs://arrima-snowflake/CA_2024H2/` was confirmed as the canonical
2024 source, `gs://arima-clustering-pipeline/data/200K/` was investigated as
a candidate:

- `combined_200k_new_tier1.parquet` (200,000 × 1,350 cols) and
  `combined_200k_new_tier3.parquet` (200,000 × 2,080 cols) are both one-hot
  encoded and have **no ID column of any kind**. Verified tier1's columns
  are a strict subset of tier3's (0 tier1-only, 730 tier3-only, 1,350
  shared). Verified — via 100% positional match across all 1,350 shared
  columns, combined with all 200,000 tier1 rows having unique profiles
  (zero duplicates) — that tier1 and tier3 are the same 200,000
  respondents in the same row order (proof, not assumption, given the
  uniqueness property: a non-identity permutation could not reproduce a
  100% positional match given no duplicate rows exist). They are safe to
  column-concat positionally, but this pair still has no ID and could not
  anchor Part 1.2 regardless.
- `arima_200k_all417_decoded.parquet` / `_raw.parquet` (200,000 × 11,879
  cols) do carry a real `id` column and were used as described above before
  the source decision moved to `arrima-snowflake`.

These files are **not** part of the confirmed 2024 baseline going forward
(see CLAUDE.md "Data source decision"), but the schema/ID findings above
remain valid and are kept for context.
