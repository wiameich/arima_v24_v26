# ARIMA Synthetic Population — Data Quality Reference

**What this is:** a general reference on the quality and characteristics
of ARIMA's synthetic Canadian population data, for anyone in the group
working with it — regardless of which project or pipeline you're using it
in. It's built from a structured audit that compared ARIMA's 2024 and 2026
data releases ("vintages") against each other, and separately tested
whether relationships that should obviously exist inside a single release
actually do.

**How to use this document:** each section states an observed finding,
what would normally be expected, and what that gap means in practice. Full
methodology, code, sample sizes, and every individual result live in two
companion documents — `outputs/part1_vintage_comparison.md` (vintage-to-
vintage comparison) and `outputs/part2_data_quality_scorecard.md`
(within-release data quality) — this document synthesizes and reframes
their findings for a general audience rather than replacing them.

**A note on how to read correlation numbers below:** most findings here
use Spearman's r, a standard measure of how strongly two things move
together, ranging from -1 (perfectly opposite) through 0 (no relationship)
to +1 (perfectly together). As a rough guide, |r| below ~0.1 is
negligible, 0.1-0.3 is weak, 0.3-0.5 is moderate, and above 0.5 is strong.
A few checks use related measures (Cramér's V for category-vs-category
comparisons, mutual information for catching non-straight-line
relationships) — each is explained in plain terms where it first appears.

---

## 1. Vintage instability — the 2024 and 2026 releases don't line up the way you'd assume

If your work touches both the 2024 and 2026 ARIMA releases — or you're
migrating something from one to the other — four separate problems apply,
and they're independent of each other: fixing one doesn't fix the others.

### 1a. The same ID number does not mean the same person across releases

ARIMA assigns each row a plain sequential number (1, 2, 3, ...) at export
time — it is not a persistent identifier tied to a real (synthetic)
person. Two exports of ARIMA data, even from the same underlying
population, each start their own numbering. This was confirmed directly:
the full ID range from the 2024H2 release turned out to be completely
contained within the 2026 release's ID range (as raw numbers, "ID 19236114"
exists in both), but spot-checking 20 of those "matching" IDs against
independent details like postal code found **19 of 20 pointed to different
people** — different provinces, different postal codes, no relationship
between the two rows beyond sharing a number.

This isn't a rare edge case — it's structurally guaranteed to look
misleading. Because IDs are dense, gap-free counting numbers assigned per
export, any smaller export's ID range will almost always sit entirely
inside a larger export's range purely by coincidence of size. **A high
percentage of "matching" IDs between two ARIMA exports is not, by itself,
evidence that they refer to the same people** — it has to be checked
against something independent (age, gender, location) before it means
anything.

**The practical rule this leads to:** never join or compare two ARIMA
exports — across vintages, or across two separately-generated pulls of
the "same" vintage — by matching ID numbers alone. Always verify with an
independent attribute spot-check first (a handful of records, checked for
matching age band, gender, and geography, is usually enough to tell within
minutes whether a proposed linkage is real). This came up twice, in two
unrelated places in this audit — once comparing 2024 vs. 2026 directly,
and once inside a separate simulated-survey file that turned out to sit in
an entirely disconnected numbering space from the files it was assumed to
line up with. In that second case, the file that looked like it should
link up (matching table names, matching row count) was a dead end — the
real linkage needed an unintuitive "+1" offset into a completely different
source file, and was only trusted once it produced a **perfect match on
gender split, a 100% exact match on postal code, and a near-perfect age
correlation (r = 0.978)** across all 200,000 records. The lesson
generalizes: a linkage that looks structurally obvious (same table names,
same row counts) can be entirely wrong, and a linkage that looks
unintuitive (an offset, a different source table) can be the correct one
— the only way to tell the difference is to check it against independent
attributes, every time, never by assumption.

### 1b. The catalog of tables and variables changed substantially, not additively

Comparing the two releases' data dictionaries directly:

| | 2024 | 2026 | Present in both | Only in 2024 | Only in 2026 |
|---|---|---|---|---|---|
| Tables | 417 | 411 | 332 | 85 | 79 |
| Variables | 11,878 | 10,943 | 5,674 | 6,204 | 5,269 |

This is not a case of the newer release simply adding new content on top
of an unchanged core — more than half of all 2024 variables (6,204 of
11,878) don't exist under that name in 2026, and a comparable number of
genuinely new 2026 variables (5,269) have no 2024 equivalent. Some of what
looks "removed" actually just moved: about a third of the only-in-2024
variables (1,994 of 6,204) belong to tables that still exist in 2026, just
holding different content (see 1c below) — the underlying questions
persisted, but under a different table name.

**This churn isn't spread evenly across content areas.** Grouping the
added and removed tables by subject area, two domains account for the
large majority of the change on both sides: **Media Usage** (49% of
dropped tables, 46% of added tables) and **Shopping** (26% of dropped,
17% of added) — together roughly three-quarters of what disappeared and
around two-thirds of what's new. Every other content domain (demographics,
health, lifestyle, professional/financial, food & drink) saw comparatively
minor churn. If your work depends on Media Usage or Shopping content
specifically, treat the 2024→2026 transition as a near-total re-check;
for most other domains, the core inventory is comparatively stable.

### 1c. Even a table or variable that "still exists" by the same name can silently mean something different

This is the single most important finding for anyone planning to treat
2024 and 2026 data as interchangeable by name.

Comparing the actual content (the set of variable descriptions) inside
every table present in both releases under the same name: **130 of 332
shared tables (39%) have zero content overlap between vintages** — same
table name, completely different subject matter. For example, the table
that used to hold "presence of children under 18 in the household" now
holds spray-bottle purchase data instead in 2026; the real
household/children content quietly moved to a different table name
entirely. **This is not a clean, one-to-one renaming** — one table's old
content might move to table X, while table X's old content moves to a
third table, not back to the original — so a simple find-and-replace on
table names will not fix it. Any two-vintage comparison, migration, or
data-fusion step that references a table by name needs an explicit,
independently-verified content mapping, not an assumption that the name
still means what it used to.

**A second, subtler version of the same problem exists even in tables
that pass the check above.** 135 of the 332 shared tables were confirmed
to have *fully unchanged* content (the exact same set of questions used in
both vintages). But checking those tables one variable at a time revealed
that in **56 of those 135 "fully stable" tables (42%), the specific
numbered variable no longer points at the same individual question** —
the table's overall content is unchanged, but which question is labeled
"variable #6" versus "variable #12" has been reshuffled between vintages.
In the worst-affected tables, *every single variable* is reshuffled: a
movie-attendance table, for instance, has its "how recently did you last
go" questions and its "what type of movie did you attend" questions using
the exact same overall set of 33 questions in both years, but a specific
variable number that meant "went in the past 2 months" in 2024 means
"attended a family/children's movie" in 2026. Across the full set of
5,674 variables shared by name between the two releases, only **17.8%
have byte-identical question wording**; **59.6% differ enough to represent
a substantively different question**, not just a rewording of the same
one.

**The practical rule:** neither "the table name is unchanged" nor "the
table's overall content passed a stability check" is sufficient evidence
that a specific numbered variable means the same thing across releases.
The only check that holds up is comparing that specific variable's actual
question text directly, every time.

### 1d. Even when a variable's wording is genuinely unchanged, its answer options might not be

Checking a representative set of 99 variables across 6 tables independently
confirmed to have identical question wording in both vintages: **46 of
those 99 variables (46%) — including every single variable in one entire
table — turned out to use a different set of answer categories in 2026**
than in 2024, despite the question text matching word-for-word. In one
case, a 4-option answer scale in 2024 collapsed to a 3-option scale in
2026; in another, the number of answer options changed inconsistently
across variables in the same table (10→2 for one, 2→4 for another, with
no single consistent pattern). Comparing raw statistics across these
variables without first checking whether the answer categories still
match would produce a meaningless result dressed up as a "distribution
shift."

**Among the variables where the answer categories genuinely do still
match**, real shifts in how people answered are common, and concentrated
in specific content areas: 87% of checked lifestyle/luxury-attitude
questions and 91% of checked digital-attitude questions showed a
meaningful shift in response pattern between vintages (using Cramér's V, a
standard 0-to-1 measure of how different two category-distributions are;
values above ~0.1 are considered a real, non-trivial difference — several
of these reached 0.45-0.62). Demographic variables were far more stable
(only 1 of 19 checked shifted meaningfully). **The practical rule:** for
lifestyle and digital-attitude content specifically, assume the population
that answered these questions looks meaningfully different between 2024
and 2026, even where the question itself is unchanged.

---

## 2. Cross-domain correlation weakness — relationships that should obviously exist often don't

Separately from the vintage comparison, this audit tested whether
straightforward, common-sense relationships hold up *within* a single
ARIMA release. The pattern is consistent and specific, not random: some
kinds of relationships hold up fine, and one particular kind consistently
does not.

### 2a. Demographic sanity checks

Testing basic demographic relationships and internal consistency in the
2026 population (n = 33,597,827 throughout this subsection):

- **Income vs. education** comes out weak (r = 0.12) against a real-world
  expectation of moderate-to-strong (typically 0.3-0.5 in actual survey
  data) — one of the most robust, well-established demographic
  relationships in any population data comes out noticeably muted here.
- **Household size vs. age** shows a plausible life-cycle shape on
  inspection (largest average households among 18-19-year-olds, a dip,
  then a mild bump back up in the mid-30s before declining with age) —
  flagged only because it didn't match a specific hardcoded expectation
  (a single peak in the 30s-40s) that turned out not to account for
  younger adults often still living in larger family households. Noted
  here as a check-design lesson, not a confirmed data defect.
- **A genuine logical contradiction, not just an implausible pattern:**
  among single-person households, **29.5% report more than one income
  earner in the household** — a mathematical impossibility, since a
  household of one person cannot contain more than one income-earning
  member. This is the most severe finding in this category, because it
  isn't a matter of real-world plausibility (like a very young person
  holding an advanced degree) — it's an internal contradiction within the
  generated data itself.
- Two "implausible but not impossible" combinations were also elevated
  beyond a reasonable rare tail: 5.0% of 18-19-year-olds report having
  completed an advanced degree (physically difficult to achieve by that
  age), and 1.6% of that same age band report $200,000+ personal income
  (a genuine rare tail in reality, but elevated here).

### 2b. Cross-domain attitude/behavior correlations

24 pairs of variables were tested, each chosen specifically because the
expected relationship is close to common sense or logical necessity (e.g.
you can't insure a car you don't own). **17 of 24 pairs (71%) came back
either in the wrong direction or at a negligible magnitude.**

The single strongest example: **whether a household's car is insured,
compared to whether the household owns a vehicle at all, came back at
r = 0.014 — essentially zero** — for a relationship that is close to a
logical necessity in the real world. Every insurance-vs-ownership,
travel-attitude-vs-frequency, and luxury-attitude-vs-frequency pair tested
showed the same pattern, in the r = 0.01-0.03 range.

**This did not generalize to every kind of relationship tested** — see
Section 3 for what held up.

### 2c. Does ARIMA's own simulated survey data preserve these relationships?

Beyond the raw synthetic population, two independently-generated simulated
survey datasets built on top of it were tested the same way, to check
whether the pattern above is specific to the base population or also
shows up downstream.

**First result (8 pairs, ~200,000 records):** every single pair came back
flagged. The standout: a well-established real-world relationship — people
currently managing their diet, compared to how large a weight-loss goal
they report — came back **substantially wrong-signed** (r ≈ -0.46, where
a real, moderate positive relationship is expected), not merely
attenuated toward zero.

**Follow-up (7 additional pairs, sourced directly from the underlying
survey's own data dictionary): is that reversal an isolated defect or a
systemic problem?** Result: **4 of 7 came back correctly signed with real
magnitude**, including two of the strongest cross-domain correlations
found anywhere in this entire audit (r = 0.27 and r = 0.21, both for the
well-established link between body weight/BMI and cardiometabolic health
conditions like high blood pressure and Type 2 diabetes). **Conclusion:
the dieting/weight-loss reversal looks like an isolated defect in that
specific pair, not a general property of the simulation** — the broader
picture is genuinely mixed (some real signal, some near-zero signal,
consistent with 2b above, and one localized reversal), not a single
verdict in either direction.

**A separate, general lesson from this same work, worth stating on its
own:** person-level linkage between a simulated survey dataset and a
separate lifestyle/attitude dataset can sometimes be recovered even when
the first file you'd naturally reach for turns out to be wrong — but this
must be independently verified every time, never assumed from matching
file names, table names, or row counts. In this case, one candidate
linkage that looked exactly right (same table structure, same expected row
count) turned out to represent an entirely different, unrelated set of
people (a near-random ~50/50 split on gender, where a genuine match should
be a clean bijection). The correct linkage required an unintuitive
numeric offset into a different source table entirely, and was only
trusted after it produced a perfect gender match, a 100% exact postal-code
match, and a near-perfect age correlation across all 200,000 records. Once
that corrected linkage was in place, testing the same insurance-vs-
ownership question that stood out in 2b produced the same near-zero result
(r ≈ 0.008) on this independently-verified data — the finding replicates,
it isn't an artifact of one specific dataset.

### 2d. Missing-data patterns

- **Missingness in this data is not represented as an actual database
  NULL — it's coded as an explicit answer category** (e.g. a literal
  "blank" label sitting alongside real answer options). A naive check for
  NULL values would wrongly conclude this data has no missingness at all.
  Anyone querying this data directly needs to know to check for the blank
  sentinel category, not rely on a standard null-check.
- **17% of all variables (1,869 of 10,943) have a non-response rate above
  50%.** On inspection, this is mostly explainable: niche or low-incidence
  content (specific regional newspapers, specific TV shows, city-specific
  transit sub-questions that only apply to people who live in that city).
  It's not on its own evidence of broken data, but these variables carry
  little usable signal and should generally be deprioritized in any
  analysis.
- A small number of variables (5, across just 2 otherwise well-populated
  tables) are **100% blank for every single record** — these look like
  dead/unused answer sub-options (e.g. a "never used" code specific to a
  smaller transit system) rather than a sign that the whole table is
  broken.
- **The variable dictionary cannot be relied on, by itself, to identify
  variables with no real variation.** Comparing what the dictionary
  implies should be a fixed/constant variable against what the actual data
  files show: **zero agreement** — the dictionary describes the *possible*
  answer set, not what was *actually* generated, so a variable can be
  effectively constant in the real data even though the dictionary lists
  multiple valid answer options for it (69 such variables were found this
  way, that the dictionary gave no indication of).

---

## 3. What does work well — this isn't a blanket "the data is broken" story

The pattern that emerges across every check above is specific, not
uniform, and it's worth stating plainly so Section 2 isn't over-read:

- **Demographic-driven attitude gradients hold up well.** Five of six
  tested "digital attitude vs. age" relationships came back correctly
  signed with real, meaningful magnitude (r = -0.11 to -0.26) — younger
  people really do show more social-media/digital-native attitudes in this
  data, at a strength consistent with what you'd expect in a real
  population.
- **Genuinely mutually-exclusive categories within the same subject area
  hold up.** A pair of auto-insurance-acquisition-channel variables that
  should be largely exclusive of each other (you typically buy insurance
  through a broker *or* a bank, not both) showed real structure
  (r = -0.19).
- **The strongest cross-domain correlations found anywhere in this audit
  are a textbook-established health relationship**, preserved cleanly in
  ARIMA's simulated survey data: body weight/BMI vs. cardiometabolic
  conditions (r = 0.21-0.27) — a reminder that ARIMA's weak cross-domain
  signal is not universal.
- **Some tables are genuinely, fully stable across the 2024→2026
  transition at every level checked** — not just matching table content,
  but every individual variable's wording and, where checked, its answer
  categories too. Core demographics and luxury-attitude content are
  examples of this, independently re-verified rather than assumed. This
  matters because it shows the instability in Section 1 is a real,
  checkable property that varies table by table — it is not true that
  "nothing is stable," and a table passing every level of this check is
  meaningful, positive evidence, not an oversight.
- **Once an ID-based linkage between two files is correctly identified and
  independently verified, it holds up cleanly** — the corrected Intact
  linkage in 2c produced a perfect gender match, a 100% exact geography
  match, and a near-perfect age correlation across 200,000 records. The
  ID-instability problem in Section 1 is a *verification* problem, not an
  unsolvable one — a linkage that passes an independent spot-check can be
  trusted.
- **A dedicated follow-up specifically designed to catch relationships
  that simpler methods might miss did not find hidden structure** — see
  Section 4 for what this means for the open question below. On its own,
  this is a reassuring result in a narrow sense: it means the weak
  correlations reported in Section 2 are not simply an artifact of using
  too blunt a statistical method.

---

## 4. Open question: is the missing cross-domain structure a real feature of the population, or an artifact of how ARIMA generates data?

ARIMA builds its synthetic population using a statistical technique
(a Gaussian copula, combined with a max-entropy balancing step) that is
known, in principle, to be capable of understating relationships between
variables from different subject domains, even when a real population
would show them. The findings above raise a genuine, unresolved question:
is the weak cross-domain correlation seen throughout Section 2 a true
feature of the population ARIMA is modeling, or a side effect of how that
population was generated?

**Evidence pointing toward "generation artifact":**
- A near-textbook-necessity relationship (insurance requires ownership)
  collapses to essentially zero (2b).
- 71% of deliberately-obvious cross-domain pairs came back flagged (2b).
- One of the most robust demographic relationships in any real population
  (income vs. education) comes out muted (2a).

**Evidence complicating a simple "everything is broken" reading:**
- Demographic-driven attitude gradients hold up with real, correctly-
  signed strength (Section 3).
- The single strongest cross-domain correlation found anywhere in the
  audit is a textbook health mechanism, preserved cleanly (Section 3).
- The one substantial, wrong-signed reversal found in the simulated survey
  data (dieting vs. weight-loss goal) did not generalize to six other
  tested mechanisms — it looks like an isolated defect in that specific
  pair, not a systemic problem (2c).

**A dedicated three-part follow-up was run specifically to test whether
better statistical methods would surface relationships that the simpler
methods above might be missing** — because a genuinely real but
non-obvious relationship (for example, a U-shaped rather than straight-
line pattern, or a relationship that only shows up within a specific
subgroup, or one that only appears once a third variable is accounted for)
could look like "no relationship" under the simpler tests in Section 2
without actually being absent from the data:

1. **Mutual information** (a statistical measure that can catch
   non-straight-line relationships that a standard correlation would miss
   entirely) was computed for 18 pairs spanning nine different content
   domains — food & drink, lifestyle, health & wellness, professional
   life, shopping, media, auto, travel, and digital. It **never once
   rescued a pair** that the simpler method had already called
   near-zero: 14 of 16 newly-tested pairs came back flagged the same way,
   extending the pattern from Section 2 into domains not previously
   tested. One genuine exception was found — a real, non-straight-line
   relationship between news-checking frequency and treating the internet
   as a primary news source — but see point 3 below for what that
   turned out to be.
2. **Subgroup analysis** checked whether any of these weak overall
   relationships were hiding a much stronger relationship within a
   specific subgroup (income bracket, province, household size, or age
   band) that gets averaged away when looking at everyone together.
   Across 24 such checks, **none** showed a subgroup effect large enough
   to materially change the conclusion — the weak overall pattern is
   fairly uniform, not a mask over strong pockets.
3. **Joint-dependence testing** checked whether a third variable was
   hiding or distorting a relationship between two others — testing 7
   trios of related variables across combinations like food/drink ×
   lifestyle × health, professional life × shopping, and media × shopping.
   Six of the seven showed no meaningful effect. The one exception was the
   non-straight-line relationship flagged in point 1: once age was
   accounted for, that relationship's already-modest strength dropped by
   about 97% — meaning it was really an age-driven pattern all along (a
   textbook example of what statisticians call Simpson's paradox — a
   relationship that appears in the pooled data but disappears once you
   separate out the group that's actually driving it), not evidence of a
   genuine, direct link between the two behaviors.

**Where this leaves the open question:** it is not resolved — this kind
of testing cannot, on its own, prove whether a relationship's absence
reflects the true underlying population or a limitation of how ARIMA
generates it. But it **does rule out one specific alternative
explanation**: that the weak correlations in Section 2 are simply an
artifact of using correlation methods too blunt to detect real structure
that's actually there. Three purpose-built methods, designed specifically
to catch exactly that kind of hidden structure, were applied across a wide
span of content domains and found almost none. The pattern that survives
every test run in this audit is specific and consistent: **ARIMA appears
to preserve demographic-driven gradients (age, income, and similar
"backbone" variables shaping attitudes and behavior) reasonably well,
while relationships between attitudes and behaviors — and even outright
logical-necessity relationships — collapse toward statistical
independence far more often than a real population would.** That's a more
precise, and more actionable, finding than either "the data has no
correlation structure" or "the data is fine" — and it should inform how
much confidence to place in any cross-domain relationship (as opposed to
a demographic-driven one) discovered in this data going forward.
