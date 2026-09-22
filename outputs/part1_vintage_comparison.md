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

**130 of 332 (39%) have a self-similarity of exactly 0.0** — the same
`table_id` refers to entirely different survey content in 2026 than it did
in 2024H2. This is not a formatting artifact: many of these tables have a
**perfect 1.000-similarity match to a differently-named table** in the
other vintage, which would not happen if the mismatch were just inconsistent
text formatting.

> **Correction (2026-09-22):** earlier drafts of this report, and the
> cross-reference in `CLAUDE.md`, stated this count as "164 (49%)." That
> number was wrong — re-derived directly from both vintages' source
> dictionaries (`data/CA_2024H2/VARIABLE_MAPPING/`,
> `data/variable_mapping_2026.csv`) and cross-checked against the
> already-cached `data/part1_1_table_self_similarity.csv`, both independent
> computations agree on **130**, not 164. The underlying finding
> (a large share of same-named tables hold unrelated content across
> vintages) is unchanged; only the count was corrected.

**Concrete example — `vv_hov` / `vv_how`:**

| | 2024H2 `vv_hov` | 2026 `vv_hov` | 2024H2 `vv_how` | 2026 `vv_how` |
|---|---|---|---|---|
| Content | Household composition / presence of children (19 vars, e.g. `vv_hov_6` = "Presence Of Children <18", `vv_hov_7` = "Total # of people in hhld") | Spray bottle purchases (2 vars) | Electronics purchase location/brand (Best Buy, Costco, Bose, Samsung...) (22 vars) | Household composition / presence of children (19 vars, e.g. `vv_how_10` = "Presence Of Children <18", `vv_how_11` = "Total # of people in hhld") |

The household-composition content **moved from `vv_hov` to `vv_how`**
between vintages, while both table_ids continued to exist, now holding
unrelated content. This is a rename-plus-reuse pattern, not a simple
addition/removal.

**Complete remapping (all 130 flagged tables) — updated 2026-09-22:**
for each of the 130 tables with self-similarity 0.0, this ran the same
best-match search (Jaccard similarity of variable descriptions) against
**all 411 2026 tables**, not a demonstration subset. Confidence is flagged
per row, since a "1.000 Jaccard" from a 1-2-variable table is much weaker
evidence than a "1.000 Jaccard" from a 20-variable table.

**Read this as two different kinds of result, not one scale of
"confidence" — the trustworthy rows and the unreliable rows are answering
different questions:**

**Trustworthy — treat these as usable findings:**

- **HIGH (76 tables):** unique best match, ≥5 shared descriptions, ≥0.5
  Jaccard — a confident content-move match. Safe to cite "content moved
  from X to Y" for these.
- **NO MATCH — likely genuinely removed (4 tables):** `vv_cry`, `vv_tvc`,
  `vv_tvs`, `vv_wef` — best Jaccard against *any* of the 411 2026 tables is
  0.0 (the nominal "best match" is the `all`/`loc` administrative sentinel
  row, which is not a real content match). **This is just as trustworthy
  as a HIGH match, not a weaker version of one — it's a confirmed negative
  result** (checked against all 411 candidates and none overlap at all),
  not a guess. Unlike the other 126 tables, these show no evidence their
  content moved anywhere; they look genuinely discontinued, not renamed.

**Not reliable for content tracing — treat only as "this table's content
is gone under its old name," nothing more:**

- **LOW (41 tables):** either the best match is tied with a second
  candidate at the identical score (11 tables — genuinely ambiguous, the
  search cannot distinguish which is right), or the match rests on fewer
  than 5 shared description strings (30 tables — a 1.000 score here can be
  a coincidental match on generic wording, e.g. two unrelated
  single-variable tables that happen to share one description string).
  **Do not treat the "best-matching 2026 `table_id`" listed for these rows
  as where the content actually went** — the only thing confirmed is that
  the 2024H2 table's content no longer exists under its 2024H2 name; the
  destination is unknown with confidence, and the listed candidate may
  simply be the least-wrong guess among many weak options.
- **MODERATE (9 tables):** a real but partial overlap (0.2-0.5 Jaccard) —
  plausible as a lead worth manually checking, but not a confirmed match
  on its own; sits between the two tiers above rather than in either.

Note the non-bijective pattern among the HIGH matches (`vv_con`→`vv_coo`
but `vv_cow`→`vv_con`; `vv_hou`→`vv_hox` but `vv_hox`→`vv_hou`): table_ids
were reassigned in a way that isn't a clean 1:1 rename mapping, which
rules out a simple "find-and-replace the table prefix" fix.

| 2024H2 `table_id` | Best-matching 2026 `table_id` | Jaccard | Confidence |
|---|---|---|---|
| `vv_air` | `vv_ais` | 0.800 | HIGH |
| `vv_ais` | `vv_air` | 1.000 | HIGH |
| `vv_all` | `vv_alm` | 1.000 | HIGH |
| `vv_alm` | `vv_all` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_aut` | `vv_auw` | 0.978 | HIGH |
| `vv_auu` | `vv_aut` | 1.000 | HIGH |
| `vv_auw` | `vv_auv` | 1.000 | HIGH |
| `vv_bus` | `vv_buu` | 1.000 | HIGH |
| `vv_buu` | `vv_buv` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_buv` | `vv_bus` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_cas` | `vv_cat` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_cat` | `vv_cav` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_cau` | `vv_cas` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_che` | `vv_chf` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_chf` | `vv_che` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_chi` | `vv_acn` | 1.000 | LOW -- tied with another candidate |
| `vv_chj` | `vv_chi` | 1.000 | HIGH |
| `vv_cho` | `vv_chp` | 1.000 | HIGH |
| `vv_chp` | `vv_cho` | 0.800 | HIGH |
| `vv_cof` | `vv_cog` | 0.200 | LOW -- small description set (n<5), match may be coincidental |
| `vv_cog` | `vv_cof` | 0.911 | HIGH |
| `vv_col` | `vv_com` | 0.761 | HIGH |
| `vv_com` | `vv_cop` | 1.000 | HIGH |
| `vv_con` | `vv_coo` | 1.000 | HIGH |
| `vv_coo` | `vv_coq` | 1.000 | HIGH |
| `vv_cop` | `vv_cos` | 1.000 | HIGH |
| `vv_cor` | `vv_cou` | 1.000 | HIGH |
| `vv_cos` | `vv_cov` | 0.920 | HIGH |
| `vv_cou` | `vv_cow` | 1.000 | HIGH |
| `vv_cov` | `vv_cox` | 0.273 | MODERATE -- partial content overlap only |
| `vv_cow` | `vv_con` | 1.000 | HIGH |
| `vv_cox` | `vv_bou` | 1.000 | LOW -- tied with another candidate |
| `vv_cre` | `vv_crg` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_cry` | `all` | 0.000 | NO MATCH -- likely removed |
| `vv_dai` | `vv_dam` | 0.392 | MODERATE -- partial content overlap only |
| `vv_dai1` | `vv_dam` | 0.331 | MODERATE -- partial content overlap only |
| `vv_daj3` | `vv_daj` | 0.340 | MODERATE -- partial content overlap only |
| `vv_dal` | `vv_dai2` | 0.316 | MODERATE -- partial content overlap only |
| `vv_dal1` | `vv_dai3` | 0.567 | HIGH |
| `vv_den` | `vv_dep` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_dep` | `vv_deq` | 1.000 | HIGH |
| `vv_dog` | `vv_doh` | 1.000 | HIGH |
| `vv_doh` | `vv_dog` | 1.000 | HIGH |
| `vv_ele` | `vv_elf` | 0.871 | HIGH |
| `vv_elf` | `vv_ele` | 1.000 | HIGH |
| `vv_faf` | `vv_fae` | 1.000 | HIGH |
| `vv_fin` | `vv_fio` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_fio` | `vv_fin` | 1.000 | HIGH |
| `vv_fla` | `vv_flb` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_flb` | `vv_fla` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_foo` | `vv_for` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_fop` | `vv_fos` | 1.000 | LOW -- tied with another candidate |
| `vv_for` | `vv_fot` | 1.000 | HIGH |
| `vv_frr` | `vv_bat` | 1.000 | LOW -- tied with another candidate |
| `vv_hom` | `vv_hoq` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_hon` | `vv_hor` | 1.000 | HIGH |
| `vv_hop` | `vv_hoo` | 1.000 | HIGH |
| `vv_hou` | `vv_hox` | 0.840 | HIGH |
| `vv_hov` | `vv_how` | 1.000 | HIGH |
| `vv_how` | `vv_hon` | 1.000 | HIGH |
| `vv_hox` | `vv_hou` | 0.978 | HIGH |
| `vv_ice` | `vv_icf` | 0.755 | HIGH |
| `vv_icf` | `vv_ice` | 1.000 | HIGH |
| `vv_int` | `vv_inz` | 0.496 | MODERATE -- partial content overlap only |
| `vv_inu` | `vv_int` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_inv` | `vv_inu` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_inw` | `vv_inx` | 1.000 | LOW -- tied with another candidate |
| `vv_inz` | `vv_inx` | 1.000 | LOW -- tied with another candidate |
| `vv_lei` | `vv_lek` | 0.667 | HIGH |
| `vv_lei1` | `vv_lek1` | 0.655 | HIGH |
| `vv_lif` | `vv_lig` | 0.925 | HIGH |
| `vv_lig` | `vv_lif` | 1.000 | HIGH |
| `vv_mag` | `vv_mal` | 0.947 | HIGH |
| `vv_mah` | `vv_mag` | 1.000 | HIGH |
| `vv_maj` | `vv_mai` | 0.538 | HIGH |
| `vv_mak` | `vv_man` | 1.000 | HIGH |
| `vv_mam` | `vv_mah` | 0.351 | MODERATE -- partial content overlap only |
| `vv_mar` | `vv_mas` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_mas` | `vv_mat` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_mat` | `vv_mau` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_mau` | `vv_mar` | 1.000 | HIGH |
| `vv_mea` | `vv_meb` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_meb` | `vv_mea` | 1.000 | HIGH |
| `vv_men` | `vv_fos` | 1.000 | LOW -- tied with another candidate |
| `vv_meo` | `vv_men` | 1.000 | HIGH |
| `vv_mob` | `vv_mod` | 1.000 | HIGH |
| `vv_moc` | `vv_mob` | 0.667 | HIGH |
| `vv_nai` | `vv_han` | 1.000 | LOW -- tied with another candidate |
| `vv_naj` | `vv_nai` | 1.000 | HIGH |
| `vv_nex` | `vv_mak` | 1.000 | LOW -- tied with another candidate |
| `vv_non` | `vv_noo` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_noo` | `vv_non` | 0.571 | HIGH |
| `vv_onl` | `vv_ono` | 1.000 | HIGH |
| `vv_onm` | `vv_onl` | 1.000 | HIGH |
| `vv_ono` | `vv_onn` | 0.933 | HIGH |
| `vv_oth` | `vv_oti` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_oti` | `vv_oth` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_pas` | `vv_pam` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_pat` | `vv_pas` | 1.000 | HIGH |
| `vv_pes` | `vv_peu` | 1.000 | HIGH |
| `vv_pet` | `vv_pew` | 0.833 | HIGH |
| `vv_peu` | `vv_pes` | 1.000 | HIGH |
| `vv_pew` | `vv_pev` | 1.000 | HIGH |
| `vv_plb` | `vv_plc` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_prf` | `vv_prg` | 1.000 | LOW -- tied with another candidate |
| `vv_pri` | `vv_prj` | 0.200 | LOW -- small description set (n<5), match may be coincidental |
| `vv_prj` | `vv_pri` | 1.000 | HIGH |
| `vv_pro` | `vv_prp` | 1.000 | HIGH |
| `vv_prp` | `vv_pro` | 0.312 | MODERATE -- partial content overlap only |
| `vv_pub` | `vv_puc` | 0.600 | HIGH |
| `vv_puc` | `vv_pub` | 0.950 | HIGH |
| `vv_rad` | `vv_raf` | 1.000 | HIGH |
| `vv_rae` | `vv_rad` | 0.942 | HIGH |
| `vv_rea` | `vv_reb` | 0.800 | HIGH |
| `vv_reb` | `vv_rea` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_sal` | `vv_eye` | 1.000 | LOW -- tied with another candidate |
| `vv_sho` | `vv_shp` | 0.183 | MODERATE -- partial content overlap only |
| `vv_shp` | `vv_shq` | 1.000 | LOW -- small description set (n<5), match may be coincidental |
| `vv_spo` | `vv_spq` | 0.667 | HIGH |
| `vv_spp` | `vv_spo` | 0.559 | HIGH |
| `vv_spr` | `vv_spp` | 1.000 | HIGH |
| `vv_tra` | `vv_trd` | 1.000 | HIGH |
| `vv_trb` | `vv_trc` | 1.000 | HIGH |
| `vv_trd` | `vv_trb` | 1.000 | HIGH |
| `vv_tvc` | `all` | 0.000 | NO MATCH -- likely removed |
| `vv_tvs` | `all` | 0.000 | NO MATCH -- likely removed |
| `vv_vei` | `vv_vek` | 1.000 | HIGH |
| `vv_vek` | `vv_vej` | 0.764 | HIGH |
| `vv_wee` | `vv_wef` | 1.000 | HIGH |
| `vv_wef` | `all` | 0.000 | NO MATCH -- likely removed |

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
- **Updated 2026-09-22: the "best match" search (Jaccard against all 411
  2026 tables) has now been run for all 130 flagged tables** — see the
  complete remapping table above, not a demonstration subset. Best-match
  confidence still varies by row (flagged inline): a 1.000 Jaccard from a
  1-2-variable table is materially weaker evidence than one from a
  20-variable table, and this was not resolved by running the search
  exhaustively — only by scoring more tables the same, imperfect way.

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

### Domain clustering of added/removed tables — by `theme`

Grouped the 85 only-2024H2 tables by their 2024H2 `THEME` field, and the
79 only-2026 tables by their 2026 `theme` field, to check whether table
churn is spread evenly across content domains or concentrated in a few.

**85 only-2024H2 tables (content dropped between vintages), by theme:**

| Theme | Count | % of 85 |
|---|---|---|
| Media Usage | 42 | 49.4% |
| Shopping | 22 | 25.9% |
| Characteristics/Views | 7 | 8.2% |
| Lifestyle | 5 | 5.9% |
| Health and Wellness | 4 | 4.7% |
| Home | 3 | 3.5% |
| Professional Life | 1 | 1.2% |
| Food and Drinks | 1 | 1.2% |
| **Total** | **85** | **100.0%** |

**79 only-2026 tables (new content added between vintages), by theme:**

| Theme | Count | % of 79 |
|---|---|---|
| Media Usage | 36 | 45.6% |
| Shopping | 13 | 16.5% |
| Health and Wellness | 10 | 12.7% |
| Lifestyle | 7 | 8.9% |
| Professional Life | 4 | 5.1% |
| Characteristics/Views | 2 | 2.5% |
| Food and Drinks | 2 | 2.5% |
| Home | 2 | 2.5% |
| Location | 1 | 1.3% |
| Segments | 1 | 1.3% |
| `all` (no theme — population-total administrative sentinel, not real content) | 1 | 1.3% |
| **Total** | **79** | **100.0%** |

**Both directions cluster heavily in the same two domains: Media Usage and
Shopping.** Media Usage accounts for 42/85 (49.4%) of dropped tables and
36/79 (45.6%) of added tables; Shopping accounts for 22/85 (25.9%) dropped
and 13/79 (16.5%) added. Together these two themes are 64/85 (75.3%) of
what was dropped and 49/79 (62.0%) of what was added — table churn is not
evenly spread across ARIMA's content domains, it's concentrated in the same
two areas on both sides of the vintage change. This is consistent with Part
1.1's finding that content *within* Media Usage/Shopping tables was itself
heavily restructured, not just added/removed wholesale: `vv_dai` and
`vv_maj` (newspaper/magazine batteries) and `vv_shp`/`vv_puc` (shopping and
transit batteries) all appear directly in Part 1.1's 130-table remap list
above (self-similarity 0.0 -- same table_id, different content across
vintages), on top of the wholesale add/remove churn counted here. The two
findings point at the same underlying area of the schema being in the most
flux between vintages, via two different mechanisms (tables renamed/reused,
and whole sub-tables added or dropped).

Two of the only-2026 tables (`loc`, `pri`) are themselves non-`VV_` structural
dimension tables (geography and population-segment lookups, per Part 1.1's
"3 non-VV administrative entries" note), not new survey content — they land
under `Location` and `Segments` respectively (1 each) and are noted here so
they aren't mistaken for genuinely new content domains.

**Methodological note:** `theme` is a per-variable field in both
dictionaries; this used each table's single theme value directly since none
of these specific 165 tables (85+79) span more than one theme (2 tables
elsewhere in the dictionary do — `vv_cog`/`vv_tea` in 2024H2, `vv_cof`/`vv_tea`
in 2026 — but neither is in the only-2024/only-2026 sets, so this doesn't
affect the counts above).

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
independently). Part 1.1's 130-table "unstable" list is a **table-level**
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
