# Methodology

Where every number in the scoring model comes from, and how much confidence
it deserves.

For how the code implements this, see [architecture.md](architecture.md).
A condensed version of this page ships inside the app, under
Settings → About → How scoring works.

---

## What the score is

A weighted checklist informed by mortality research. Each category earns a
sub-score from 0 to 10 based on how much of that category's achievable
benefit you captured this week; the sub-scores are combined using weights
that reflect how much each category appears to matter.

## What the score is not

**It is not a risk reduction:**

1. **The evidence is observational.** Every source below is a cohort study or
   a meta-analysis of cohort studies. People who exercise, sleep well, and
   have close friends differ from people who don't in a hundred unmeasured
   ways (income, education, baseline health, whether they are already ill).
2. **The categories are not independent, yet they are added in your score.**
   Someone who exercises *and* sleeps well *and* sees friends doesn't get
   three people's worth of benefit.
3. **The weights carry a judgment.** The credibility factors below are
   defensible, but not objective. Two reasonable people would pick different
   ones.

What the score is good for: Tracking that you have not slept properly in
three weeks, or that your socialization has become unsatisfying, or that
you might not be getting as much cardio as you thought. Generally trying
to encourage you to prioritize at least three of the five pillars of 
longevity (Sleep, Exercise, and Social Health). Nutrition and Emotional
Health are the other categories, and though they are not in the model yet,
they may be someday!

---

## The method

```
weight = RRR × credibility
```

**RRR** is the category's maximum Relative Reduction in mortality Risk, in
percentage points. **Credibility** discounts that for how far the source sits
from what this app actually measures.

The weights therefore sum to a maximum total, and each reads as "percentage
points of reduced mortality risk this category can buy you."

### Two conversion rules

**Everything is a risk *reduction*, not an increase.** A source reporting
"34% higher mortality" (HR 1.34) describes a reduction of `1 − 1/1.34` = 25%,
not 34. Reading an increase as though it were a reduction overstates it, and
the error grows with the effect size.

**Everything is a *risk* ratio, not an odds ratio.** Odds ratios overstate
risk ratios whenever the outcome is common, and death over a long follow-up
is very common. The conversion is:

```
RR = OR / (1 − p₀ + (p₀ × OR))
```

where `p₀` is the outcome rate in the reference group. Two weights below
depend on this and are roughly halved by it.

---

## Category by category
### Moderate physical activity — weight 35

| | |
| :--- | :--- |
| Source | Lee et al. 2022, *Circulation* |
| Design | 116,221 US adults, 2 cohorts, 30 years of follow-up |
| Outcome | All-cause mortality |
| Effect | HR **0.65** at 900 min/week → **RRR 35** |
| Credibility | **1.0** |
| Weight | **35** |

The strongest evidence in the set, and the only category taken at full
credibility: the largest and longest-running cohort here, a clean
dose-response with no plateau inside the observed range, and all-cause
mortality as the endpoint — which is the endpoint the composite is built on.

Curve anchors (HR → score at `k = 0.35`):

| min/week | 0 | 75 | 160 | 250 | 900 |
| --- | --- | --- | --- | --- | --- |
| HR | 1.00 | 0.92 | 0.82 | 0.78 | 0.65 |
| score | 0 | 2.29 | 5.14 | 6.29 | 10 |

Monotonic — moderate activity does not turn on you anywhere in the observed
range, so the curve simply clamps past 900.

### Socializing — weight 24

| | |
| :--- | :--- |
| Source | Holt-Lunstad et al. 2010, *PLoS Medicine* |
| Design | Meta-analysis, 148 studies, 308,849 people, mean 7.5y follow-up |
| Outcome | Survival (~29% mortality across studies) |
| Effect | OR **1.50** for survival → **RRR 26.2** |
| Credibility | **0.9** |
| Weight | **24** |

The headline "50% greater likelihood of survival" is an **odds** ratio for
**survival**, and needs both conversions:

```
OR for death   = 1 / 1.50            = 0.667
RR             = 0.667 / (1 − 0.29 + 0.29 × 0.667) = 0.738
RRR            = 1 − 0.738           = 26.2%
```

So the famous 50% figure is a 26-point risk reduction on a like-for-like
scale — comparable to moderate exercise, not half again as large. Taking the
50 at face value was the single largest error in the original weighting.

Credibility 0.9 rather than 1.0: an enormous and well-conducted meta-analysis,
but the exposure is self-reported and social connection is strongly
confounded by socioeconomic status.

**Why the target is user-configurable.** How much company a week needs is the
one target here that does not generalise. An introvert and an extrovert are 
not under-served by the same number.

**Satisfaction mode** replaces the stopwatch entirely with one question a
week. Under it the sub-score is 10 for yes and 0 for no or unanswered, and it
never prorates, as there is no such thing as three sevenths of being satisfied.

### Sleep — weight 17

| | |
| :--- | :--- |
| Source | *GeroScience* 2025, "Imbalanced sleep increases mortality risk by 14–34%" |
| Design | Meta-analysis, 79 cohort studies |
| Outcome | All-cause mortality, reference 7–8 h |
| Effect | Short (<7h) HR **1.14** → RRR 12.3 · Long (≥9h) HR **1.34** → RRR 25.4 |
| Credibility | **0.9** |
| Weight | **17** |

Both arms are *increases* and need inverting: 1.34 is a 25-point reduction,
not 34. The RRR is the **midpoint of the two arms**, 18.8 — not the long
arm's 25.4, which would take the least trustworthy number of the pair at face
value. (An earlier version of this table used a rounder 20, which was not the
midpoint of anything and quietly inflated the arm we trust least.)

Credibility 0.9 — level with socializing, behind only moderate PA — is
deliberate. The source
matches what the app measures more closely than almost any other here: 79
cohort studies, all-cause mortality, and an exposure that is literally hours
slept, the number the user types in. It is docked from 1.0 only for the
heterogeneity of pooling 79 studies, and because one arm of the curve is
genuinely suspect.

**That suspect arm is discounted twice already, and must not be charged a
third time.** Long sleep is the arm most contaminated by reverse causation
(illness causes oversleeping at least as much as oversleeping causes illness).
Taking the midpoint rather than the long arm discounts it once; the asymmetric
slope below discounts it again. An earlier version of this table also applied
a 0.8 credibility on the same grounds, which was that one concern charged
three times over.

**The band is 7–9 hours**, taken directly from how the source defines its
buckets: under 7 is short, 9 or more is long, and the range between carries no
measured penalty.

**The penalty is asymmetric.** On the raw numbers an hour of oversleep is
worth about twice an hour of undersleep (25.4 / 12.3 = 2.06). The shipped
ratio is **1.5** — short 2×, long 3× — discounted from 2.06 for the reverse
causation above. The discount sits in the slope rather than in the weight
because the weight would discount both tails equally and only one tail has the
problem.

```
A(h) = 7.5 − 2(7 − h)   if h < 7
     = 7.5              if 7 ≤ h ≤ 9
     = 7.5 − 3(h − 9)   if h > 9
```

### Vigorous physical activity — weight 11

| | |
| :--- | :--- |
| Source | Lee et al. 2022, *Circulation* |
| Outcome | **Cardiovascular mortality** |
| Effect | HR **0.85** at 215 min/week → **RRR 15**, rising back to 0.90 by 900 |
| Credibility | **0.7** (substitution) |
| Weight | **11** (10.5, rounded up) |

**This is the one category scored on CVD mortality rather than all-cause.** 
The reverse-J benefit, peaking around 215 min/week and declining after,
is the reason vigorous activity is scored separately from moderate, and it 
exists *only* in the CVD data. The all-cause curve for vigorous PA is 
monotonic to 900 min (HR 0.81).

**The cost:** a 15% reduction in cardiovascular mortality is not 15 
percentage points of all-cause mortality. CVD is roughly a third of
deaths in these populations, so on a strict all-cause scale this category
would be worth nearer 5. Keeping it at 11 is a product judgment based on
other research that demonstrates a monotonic longevity increase with 
cardiovascular fitness. This is the least defensible number in the table 
on purely evidential grounds.

The credibility factor of 0.7 is **substitution, not quality**: the same
cohort describes moderate and vigorous activity as "an equivalent
combination," so paying both out in full double-counts a single exercise
habit.

### Resistance training — weight 6

| | |
| :--- | :--- |
| Source | Momma et al. 2022, *British Journal of Sports Medicine* |
| Design | Meta-analysis, 16 cohorts; 7 for all-cause (42,133 deaths / 263,058 people) |
| Outcome | All-cause mortality |
| Effect | RR **0.83** at ~40 min/week → **RRR 17** |
| Certainty | **GRADE: very low** (authors' own rating) |
| Credibility | **0.33** |
| Weight | **6** |

The authors rate their own evidence "very low" certainty, chiefly for
indirectness — most included studies were conducted in the USA — alongside
self-reported exposure and an inability to test for publication bias.

### Time in nature — weight 5

| | |
| :--- | :--- |
| Source | White et al. 2019, *Scientific Reports* |
| Outcome | **Self-reported good health** — mortality was never studied |
| Effect | OR **1.59** at 120–179 min/week → RR ≈ 1.15 → **15** |
| Credibility | **0.33** |
| Weight | **5** |

The headline 59% is an odds ratio for a **common** outcome, so the conversion
is super important:

```
RR = 1.59 / (1 − 0.65 + 0.65 × 1.59) = 1.15
```

A 15% relative increase in the likelihood of reporting good health — not 59%.
The study also reports that time in nature explained roughly **1% of
variance**, and its authors explicitly note that prospective studies are
needed before clinical recommendations can be made.

Credibility 0.33 acknowledges that this is not a mortality finding at all and
is being imported onto a mortality scale. It shares that factor with resistance
training, and the logic is comparative: if evidence of very low certainty
*about mortality* is worth a third, evidence about a different outcome entirely
cannot be worth more. Resistance still outranks nature, but on the size of its
effect rather than the quality of its evidence. The **120-minute target** is the one
part of this category that is directly evidenced.

### Flexibility and balance — weight 4

| | |
| :--- | :--- |
| Source | Das et al. 2024, *Research on Aging* |
| Design | Meta-analysis, community-dwelling **elderly** |
| Outcome | All-cause mortality |
| Effect | HR **1.14** for inability to complete a static balance test → **RRR 12** |
| Credibility | **0.33** |
| Weight | **4** |

The smallest weight in the table — though not because of the discount, which
it shares with resistance and nature. Its RRR is simply the lowest. Three
compounding reasons for the discount:

- **It measures a capacity.** The source says people who cannot balance die 
  sooner. It says nothing about whether logging minutes of balance work changes 
  that.
- **It is a frailty marker.** Failing a balance test is a *consequence* of being 
  unwell, which plausibly makes this finding reverse-causal.
- **The population is elderly**, not the general adult population using this
  app.

It stays in the model because balance work is cheap, harmless, and probably good
for you, especially as you age.

---

## Summary

| Category | Source effect | Outcome | RRR | Cred. | Weight |
| --- | --- | --- | --- | --- | --- |
| Moderate PA | HR 0.65 @ 900 min | all-cause | 35 | 1.0 | **35** |
| Socializing | OR 1.50 survival | all-cause | 26.2 | 0.9 | **24** |
| Sleep | HR 1.14 / 1.34 | all-cause | 18.8 | 0.9 | **17** |
| Vigorous PA | HR 0.85 @ 215 min | **CVD** | 15 | 0.7 | **11** |
| Resistance | RR 0.83 @ 40 min | all-cause | 17 | 0.33 | **6** |
| Nature | OR 1.59 | **self-rated health** | 15 | 0.33 | **5** |
| Flexibility | HR 1.14 (balance test) | all-cause | 12 | 0.33 | **4** |
| | | | | | **102** |

Sections above run in this order — heaviest first — so the categories doing
most of the work are the ones read first. The enum in the code is ordered
differently and cannot be changed to match: Drift persists those ordinals, so
reordering it would silently recategorise every stored row.

Weights are **relative**. Any of them can be changed in Settings → Advanced, 
and doing so rescores your whole history rather than only future weeks.

### A structural caveat about the categories themselves

**The number of categories is itself a weighting decision.** Physical activity
is split four ways here and socializing is not split at all, so exercise
accounts for over half the composite partly by construction. Had socializing
been divided into partner, friends, family, and community, it would dominate
instead. There is no neutral way to carve this up, and this particular carving 
reflects what is practical to log as much as what the research says. Remember
that the score is merely a way to measure your achievement in each specific
category.

---

## Sources

- Holt-Lunstad J, Smith TB, Layton JB (2010). Social relationships and
  mortality risk. *PLoS Medicine*.
  <https://pmc.ncbi.nlm.nih.gov/articles/PMC2910600/>
- Lee DH et al. (2022). Long-term leisure-time physical activity intensity and
  all-cause and cause-specific mortality. *Circulation*.
  <https://pmc.ncbi.nlm.nih.gov/articles/PMC10121111/>
- Momma H et al. (2022). Muscle-strengthening activities and risk of
  all-cause mortality. *British Journal of Sports Medicine*.
  <https://pmc.ncbi.nlm.nih.gov/articles/PMC9209691/>
- *GeroScience* (2025). Imbalanced sleep increases mortality risk by 14–34%.
  <https://pubmed.ncbi.nlm.nih.gov/40072785/>
- White MP et al. (2019). Spending at least 120 minutes a week in nature is
  associated with good health and wellbeing. *Scientific Reports*.
  <https://pmc.ncbi.nlm.nih.gov/articles/PMC6565732/>
- Das S et al. (2024). Performance in a balance test and prediction of
  all-cause mortality. *Research on Aging*.
  <https://doi.org/10.1177/01640275241232392>

Nothing here is medical advice. If a number in this document matters to a
decision you are making about your health, talk to a doctor rather than to an
app that keeps a weekly tally.
