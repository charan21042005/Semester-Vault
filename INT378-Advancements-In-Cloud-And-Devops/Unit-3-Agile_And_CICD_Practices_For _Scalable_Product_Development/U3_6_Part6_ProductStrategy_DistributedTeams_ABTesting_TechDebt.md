# 🎯 UNIT III — PART 6: PRODUCT STRATEGY, DISTRIBUTED TEAMS, A/B TESTING AND TECHNICAL DEBT MANAGEMENT

> **Covered in this document:** **(A) Product strategy** (vision → strategy → roadmap → backlog, product-market fit, MVP and Lean Startup's Build-Measure-Learn, dual-track discovery and delivery, OKRs, North Star metric, AARRR pirate metrics, roadmap styles, prioritization frameworks with numeric examples: RICE, ICE, MoSCoW, Kano, Value-vs-Effort) → **(B) Distributed teams** (models, challenges, practices, async-first, overlap hours, working agreements, adapting Scrum ceremonies, tools, security, culture, a sample multi-time-zone schedule) → **(C) A/B testing** (what and why, process, hypothesis, metrics and guardrails, statistics basics, a worked z-test with numbers, Python code, sample-size rule of thumb, deterministic bucketing code, pitfalls such as peeking and SRM, A/B vs canary vs feature flags, tools) → **(D) Technical debt management** (metaphor, Fowler's quadrant, types, causes, interest, measuring with SonarQube-style metrics, strategies: Boy Scout rule, refactoring, strangler fig, capacity allocation, debt backlog, prioritization table, communicating to business) → how all four connect to CI/CD → exam templates → viva → master sheet.

---

Absolutely mate. 😤🔥 Parts 1-5 covered **how** to build and ship. Part 6 covers four syllabus items that decide **what to build, who builds it, how to prove it works, and how to keep the code healthy**:

```
(A) Product strategy       → WHAT to build and WHY
(B) Distributed teams      → HOW people collaborate across places and time zones
(C) A/B testing            → PROOF (data) that a change helps users
(D) Technical debt mgmt    → KEEPING the codebase fast to change
```

---

# 🅐 PRODUCT STRATEGY

## A1. Why product strategy matters

Agile and CI/CD make you **fast at building**. Strategy makes sure you build **the right thing**. Shipping features quickly in the wrong direction just wastes money faster.

```
Output  = features shipped         (what you built)
Outcome = user/business behavior changed (what happened because of it)
Strategy focuses on OUTCOMES, not output.
```

## A2. The strategy hierarchy ⭐⭐⭐

```
VISION      "Why do we exist? What future do we want?"           (3-10 years)  e.g., "Make learning accessible to everyone"
   ▼
STRATEGY    "How will we win? Which bets, which customers?"      (1-3 years)   e.g., "Win Indian college students via affordable mobile courses"
   ▼
ROADMAP     "What themes/initiatives, in what order?"            (quarters)    e.g., Q1: onboarding, Q2: offline mode, Q3: group study
   ▼
BACKLOG     "What exactly do we build next?"                     (sprints)     user stories, ordered by value
   ▼
SPRINTS / CI/CD  → increments reach users → metrics → learning → feeds back up the chain
```

| Level | Owner | Detail |
|---|---|---|
| **Vision** | Leadership/CPO | Inspirational, stable direction |
| **Strategy** | Product leadership | Target market, differentiation, goals, trade-offs ("what we will NOT do") |
| **Roadmap** | Product Manager | Themes and outcomes, not fixed feature promises |
| **Backlog** | Product Owner | Detailed, ordered, refined stories |

A good strategy answers: **Who is the customer? What problem? Why us (differentiator)? How do we measure success? What are we not doing?**

## A3. Product-market fit (PMF) and MVP ⭐

> **Product-market fit:** a product satisfies a strong market demand: users genuinely need it, keep using it, and recommend it.

> **MVP (Minimum Viable Product):** the **smallest version** of a product that lets you **learn the most** about customers with the least effort. (Not "a bad product": minimal but valuable.)

MVP examples: landing page + sign-up form (smoke test), concierge MVP (manually deliver the service), "Wizard of Oz" (looks automated, human behind), single-feature app. (Dropbox's explainer video, Zappos' founder buying shoes from stores.)

### Lean Startup: Build-Measure-Learn ⭐

```
   IDEAS ──▶ BUILD (MVP/experiment) ──▶ PRODUCT ──▶ MEASURE (data) ──▶ LEARN (insights)
     ▲                                                                       │
     └───────────── persevere / PIVOT / kill ◀────────────────────────────────┘
```

Concepts: **validated learning**, **hypothesis-driven development** ("We believe that [change] will result in [outcome] for [users]. We'll know we're right when [metric] moves by [amount]"), **pivot** (change strategy based on evidence), **innovation accounting**.

## A4. Discovery and delivery (dual-track Agile) 🧭

```
DISCOVERY TRACK (what to build?)             DELIVERY TRACK (build it right)
 user interviews, prototypes, usability tests,     Scrum/Kanban sprints, TDD/BDD, CI/CD,
 data analysis, experiments (A/B)                  releases, monitoring
        └────────── validated ideas flow into the backlog ──────────▶
```

Teams that skip discovery build features nobody uses (studies of products often find a large share of features are rarely or never used).

## A5. Goals and metrics

### OKRs (Objectives and Key Results) ⭐

```
OBJECTIVE (qualitative, inspiring):     "Make onboarding delightful for new students"
KEY RESULTS (measurable outcomes):      KR1: Increase day-1 activation from 35% → 50%
                                        KR2: Reduce time-to-first-lesson from 8 → 3 minutes
                                        KR3: Raise onboarding NPS from 20 → 40
```

Rules: 3-5 objectives, 2-5 KRs each; ambitious (~70% attainment is healthy); reviewed regularly; **KRs are outcomes, not tasks**. (SAFe 6.0 expanded OKR guidance, Part 2.)

### North Star Metric

One metric capturing the core value delivered: e.g., Spotify "time spent listening", Airbnb "nights booked", WhatsApp "messages sent".

### AARRR "Pirate metrics" (growth funnel) 🏴‍☠️

| Stage | Question | Example metric |
|---|---|---|
| **A**cquisition | How do users find us? | Visits, installs, CAC |
| **A**ctivation | Do they have a great first experience? | Sign-up → first key action % |
| **R**etention | Do they come back? | D1/D7/D30 retention, churn |
| **R**evenue | Do they pay? | Conversion, ARPU, LTV |
| **R**eferral | Do they tell others? | Invites, NPS, viral coefficient |

Other common metrics: **DAU/MAU**, **conversion rate**, **churn rate**, **LTV:CAC**, **NPS**, **CSAT**, **task success**, **error rate/latency (guardrails)**.

## A6. Roadmap styles

| Style | Description | Pro | Con |
|---|---|---|---|
| **Feature/date roadmap** | "Feature X ships March 15" | Familiar | False certainty, output-focused |
| **Theme/outcome roadmap** | Goals per quarter ("improve activation") | Flexible, outcome-driven | Needs trust |
| **Now-Next-Later** | Three horizons, no dates beyond "now" | Honest about uncertainty | Stakeholders want dates |
| **Goal-oriented (GIST, OKR-linked)** | Goals → ideas → steps → tasks | Aligns teams | Needs discipline |

## A7. Prioritization frameworks ⭐⭐⭐ (WITH NUMBERS)

You always have more ideas than capacity. Frameworks make trade-offs explicit.

### 1) RICE

```
RICE score = (Reach × Impact × Confidence) ÷ Effort
Reach      = users affected per period (e.g., per quarter)
Impact     = 3 massive, 2 high, 1 medium, 0.5 low, 0.25 minimal
Confidence = 100%, 80%, 50% (how sure are we?)
Effort     = person-months
```

| Feature | Reach | Impact | Confidence | Effort | RICE |
|---|---|---|---|---|---|
| One-tap checkout | 8,000 | 2 | 80% | 2 | (8000×2×0.8)/2 = **6,400** |
| Dark mode | 5,000 | 0.5 | 100% | 1 | (5000×0.5×1)/1 = **2,500** |
| Referral program | 3,000 | 3 | 50% | 3 | (3000×3×0.5)/3 = **1,500** |

Order: One-tap checkout → Dark mode → Referral.

### 2) ICE (quick): **Impact × Confidence × Ease** (each 1-10)

### 3) MoSCoW: **M**ust have / **S**hould have / **C**ould have / **W**on't have (this time): great for fixed-scope releases

### 4) Kano model ⭐

| Category | If present | If absent | Example |
|---|---|---|---|
| **Basic (must-be)** | Neutral (expected) | Very unhappy | Login works, data not lost |
| **Performance (one-dimensional)** | More = happier | Less = unhappier | Faster page loads, more storage |
| **Delighters (attractive)** | Delighted | Nobody notices | Smart suggestions, playful animation |
| **Indifferent / Reverse** | No effect / annoys some | — | Unused settings |

Over time delighters become basics (e.g., touch screens).

### 5) Value vs Effort matrix

```
 High value │ Quick wins (DO FIRST)   │ Big bets (PLAN)
            │─────────────────────────┼──────────────────────
 Low value  │ Fill-ins (maybe)        │ Money pits (AVOID)
            └──────── Low effort ─────┴──── High effort ───
```

### 6) WSJF (SAFe) from Part 2: **Cost of Delay ÷ Job size**.

## A8. Linking strategy to Agile and CI/CD 🔗

```
OKR → Product Goal (Scrum) → roadmap themes → ordered Product Backlog → Sprint Goal → stories
→ CI/CD ships small increments → feature flags + A/B tests measure impact → metrics update the OKR progress → re-prioritize
```

**Build vs Buy vs Partner:** build what differentiates you; buy/integrate commodities (payments, auth, email).

---

# 🅑 DISTRIBUTED TEAMS

## B1. What is a distributed team?

> A **distributed team** has members working in **different locations (cities, countries, time zones)** rather than one office, collaborating mostly through digital tools.

| Model | Description |
|---|---|
| **Co-located** | Everyone in one office |
| **Hybrid** | Mix of office and remote days |
| **Remote-first** | Remote is the default; office optional; processes designed for remote |
| **Fully distributed** | No office at all (e.g., GitLab, Automattic-style) |
| **Follow-the-sun** | Work passes between time zones to achieve ~24-hour progress (support, ops, some development) |
| **Offshore/nearshore outsourcing** | Partner teams in other countries (a different dynamic: vendor management) |

## B2. Challenges ⭐

| Challenge | Example |
|---|---|
| **Time-zone gaps** | India vs US West Coast: little or no overlap; waiting 24 h for a review reply |
| **Communication loss** | No hallway chats; tone misread in text; slow clarification |
| **Trust and visibility** | "Are they working?"; hidden blockers |
| **Cultural/language differences** | Directness, hierarchy, holidays, idioms |
| **Onboarding** | New joiners feel isolated; knowledge locked in heads |
| **Tool sprawl and meeting overload** | Slack + email + docs + 10 meetings a day |
| **Hand-off delays** | Dev in one zone, QA in another → long cycles |
| **Security and compliance** | Home networks, personal devices, data residency |
| **Isolation and burnout** | Blurred work-life boundaries |
| **Inconsistent environments** | "Works on my laptop" gets worse remotely |

## B3. Principles and practices that work ✅

### 1) Async-first, written culture ⭐

```
Default to written, asynchronous communication; use meetings only when real-time adds value
Write decisions down: Architecture Decision Records (ADRs), RFCs/design docs, meeting notes, README/runbooks
"If it isn't written down, it didn't happen."
```

### 2) Overlap hours and time-zone design

```
Find 2-4 hours of overlap for real-time collaboration (planning, pairing, retros)
Rotate inconvenient meeting times fairly; record sessions; share notes
Clear "response time expectations" (e.g., PR review within one working day)
```

### 3) Working agreements (team charter) ⭐

Short, visible rules the team creates together, for example:

```
• Core overlap hours: 13:00-16:00 UTC          • Daily Scrum: written async update OR 10-minute call
• PRs reviewed within 24 h; small PRs (<300 lines)  • Use threads; no decisions in DMs
• Camera optional; agenda required for meetings   • Status in Slack: working hours visible
• Definition of Done and DoR documented            • Escalation path for blockers
```

### 4) Adapt Agile ceremonies for distributed teams

| Ceremony | Distributed adaptation |
|---|---|
| Daily Scrum | Async written stand-up in Slack/Teams (yesterday/today/blockers) or a short video call at overlap time |
| Sprint Planning | Pre-refine backlog async; plan in overlap hours using shared board and estimation tools (planning poker online) |
| Sprint Review | Recorded demo + live Q&A; share working link (staging URL) |
| Retrospective | Online board (Miro, Mural, FunRetro, EasyRetro), anonymous input, rotating facilitator |
| Refinement | Docs + comments before, short call after |
| SAFe PI Planning | Virtual PI planning with breakout rooms and digital Program Board |

### 5) Technical practices that make distributed work smooth (CI/CD again!)

```
Trunk-based development, small PRs, automated CI on every PR → review feedback is mostly automatic
IaC + containers (Units I-II) → identical environments for everyone
Automated deployments + feature flags → no "deployment window" dependence on a person in a specific zone
Good documentation, runbooks, ADRs, onboarding scripts (`make setup`)
Self-service platforms; clear ownership (CODEOWNERS)
Remote pair/mob programming (VS Code Live Share, Tuple, Zoom screen share)
Observability dashboards visible to all; follow-the-sun on-call rotations
```

### 6) Culture and wellbeing

```
Psychological safety (blameless retros/postmortems)    Virtual coffee, team rituals, occasional in-person meetups
Over-communicate context and "why"                      Respect local holidays and working hours
Recognize remote contributions publicly                 Outcome-based performance, not hours online
Mentoring and buddy system for new joiners
```

## B4. Tools by purpose 🧰

| Purpose | Tools |
|---|---|
| Chat | Slack, Microsoft Teams, Discord |
| Video | Zoom, Google Meet, Teams |
| Work tracking | Jira, Azure Boards, Linear, Trello, GitHub Projects |
| Documentation | Confluence, Notion, Google Docs, GitHub Wiki/Markdown in repo |
| Whiteboarding | Miro, Mural, FigJam |
| Code and reviews | GitHub, GitLab, Bitbucket |
| CI/CD | Jenkins, GitHub Actions, GitLab CI |
| Async video | Loom |
| Time zones | World Time Buddy, calendar working-hours settings |
| Security | VPN/zero-trust access (SSO + MFA), device management, secrets managers |

## B5. Security for distributed work 🔐 (links to Unit V)

```
SSO + MFA everywhere                      Zero-trust access, least privilege IAM
VPN or identity-aware proxy               Encrypted devices, patch management
No secrets in chat/email                  Audit logs, DLP where required
Data residency/compliance by region       Secure CI/CD runners and secrets (credentials store)
```

## B6. Sample schedule: team across India (IST), Europe (CET) and US East (ET)

```
UTC       IST(+5:30)  CET(+1)   ET(-5)     Activity
06:30     12:00       07:30     01:30      ← India working, others asleep (deep work, async)
09:00     14:30       10:00     04:00      Europe starts ; India afternoon → India-Europe overlap (planning, reviews, pairing)
13:00-14:30  18:30-20:00 14:00-15:30 08:00-09:30  ← THREE-WAY OVERLAP (short): weekly planning/retro/demos
14:30     20:00       15:30     09:30      US morning; India winds down → hand-off notes (follow-the-sun)
```

Tips: keep the three-way overlap for **high-value** meetings only (1-2 per week); everything else async.

## B7. Measuring distributed team health

```
DORA metrics (lead time, deployment frequency...)   PR review turnaround time
Cycle time and WIP                                   Engagement/satisfaction surveys, eNPS
Meeting load (hours/week)                            Onboarding time to first merged PR
```

---

# 🅒 A/B TESTING

## C1. What is A/B testing? ⭐⭐⭐

> **A/B testing** (split testing) is a **controlled experiment** where users are **randomly assigned** to see **version A (control)** or **version B (variant)**, and a **pre-defined metric** is compared to determine statistically whether B performs better.

```
        100% of users  (randomly split)
        ┌──────────────┴──────────────┐
   50% → A (control: current)      50% → B (variant: new)
        └──── measure same metric: conversion / click-through / revenue / retention ────┘
                              compare with statistics → ship B, keep A, or iterate
```

Why: **opinions don't equal evidence**. Small changes (button text, price display, ranking algorithm) can move metrics by large amounts, and intuition is often wrong.

Examples: checkout button colour/text, recommendation algorithm, pricing page layout, onboarding flow, email subject line, search ranking, a new API caching strategy (latency metric).

## C2. A/B vs related terms ⭐

| Term | Meaning |
|---|---|
| **A/B test** | Two versions, compare a metric statistically |
| **A/B/n test** | Several variants (A, B, C...) |
| **Multivariate test (MVT)** | Test combinations of multiple elements at once (needs lots of traffic) |
| **Split URL test** | Variants on different URLs/pages |
| **Canary release** | Gradual rollout to detect **technical** problems (errors, latency); not designed to compare business metrics |
| **Feature flag** | Toggle that enables the variant for a user group (the **mechanism** for experiments) |
| **Multi-armed bandit** | Algorithm that shifts traffic towards the better variant during the test (explore-exploit) |
| **Dark launch/shadow** | Run new code without exposing to users |

Canary asks **"is it safe?"**; A/B asks **"is it better?"**

## C3. The experiment process (step by step) ✅

```
1. OBSERVE & HYPOTHESIZE    "Users abandon checkout at the shipping step"
   Hypothesis: "Showing delivery cost earlier will increase checkout completion by ≥ 5% (relative)."
2. DEFINE METRICS           Primary: checkout completion rate. Secondary: revenue/order. Guardrails: page load time, error rate, refund rate
3. DESIGN                   Unit of randomization (user, not session), traffic split, duration, minimum detectable effect (MDE), sample size
4. IMPLEMENT                Feature flag + variant code + event tracking; QA both variants (CI tests!)
5. RUN                      Random assignment; don't change the experiment mid-way; wait for planned sample size/duration (≥ 1-2 full weekly cycles)
6. ANALYZE                  Check data quality (SRM), compute difference, p-value / confidence interval, segments (carefully)
7. DECIDE                   Ship B / keep A / iterate; consider practical significance (is the lift worth the cost?)
8. DOCUMENT & CLEAN UP      Record learning; remove losing code and flags
```

## C4. Statistics basics (just enough for exams and interviews) 📐

| Term | Meaning |
|---|---|
| **Null hypothesis (H0)** | No difference between A and B |
| **Alternative (H1)** | There is a difference |
| **Significance level α** | Probability of a false positive we accept (commonly **0.05**) |
| **p-value** | Probability of seeing a difference at least this large **if H0 were true**. If p < α → "statistically significant" |
| **Type I error (false positive)** | Declare a winner when there is none (probability α) |
| **Type II error (false negative)** | Miss a real effect (probability β) |
| **Power (1 − β)** | Chance of detecting a real effect (commonly **80%**) |
| **Confidence interval (CI)** | Range likely to contain the true effect (95% CI) |
| **MDE** | Smallest effect worth detecting |
| **Sample size** | Users needed per variant to reach desired power for the MDE |
| **Statistical vs practical significance** | A tiny lift can be significant with huge traffic but not worth shipping |

⚠️ A p-value is **not** "the probability that B is better".

### Sample-size rule of thumb (80% power, α = 0.05, two-sided)

```
n per variant ≈ 16 × p(1 − p) / δ²
p = baseline conversion rate, δ = absolute change to detect

Example: baseline 5% (0.05), detect +0.5 percentage points (δ = 0.005)
n ≈ 16 × 0.05 × 0.95 / 0.005² = 16 × 0.0475 / 0.000025 ≈ 30,400 users per variant
```

Lesson: detecting small lifts needs **lots of traffic**; low-traffic sites should test bigger changes.

## C5. WORKED EXAMPLE (numbers) 🧮

```
A (control): 10,000 visitors, 500 conversions → 5.00%
B (variant): 10,000 visitors, 580 conversions → 5.80%
Difference = +0.80 percentage points (16% relative lift)
```

Two-proportion z-test:

```
Pooled p̂ = (500 + 580) / 20,000 = 0.054
SE = √( p̂(1−p̂)(1/nA + 1/nB) ) = √(0.054 × 0.946 × 0.0002) ≈ 0.003196
z = 0.008 / 0.003196 ≈ 2.50
Two-sided p-value ≈ 0.0123   → p < 0.05 → statistically significant
95% CI for the difference ≈ 0.008 ± 1.96 × 0.003196 ≈ [0.17%, 1.43%]
```

Interpretation: B's improvement is unlikely to be chance; the true lift is plausibly between ~0.2 and ~1.4 percentage points. Decide using business value (is +0.2 enough?) and guardrail metrics.

### Python: compute it yourself (no external library needed)

```python
import math

def two_prop_ztest(conv_a, n_a, conv_b, n_b):
    p_a, p_b = conv_a / n_a, conv_b / n_b
    p_pool = (conv_a + conv_b) / (n_a + n_b)
    se_pool = math.sqrt(p_pool * (1 - p_pool) * (1 / n_a + 1 / n_b))
    z = (p_b - p_a) / se_pool
    p_value = math.erfc(abs(z) / math.sqrt(2))            # two-sided
    se_diff = math.sqrt(p_a * (1 - p_a) / n_a + p_b * (1 - p_b) / n_b)
    ci = ((p_b - p_a) - 1.96 * se_diff, (p_b - p_a) + 1.96 * se_diff)
    return {"rate_a": p_a, "rate_b": p_b, "lift_abs": p_b - p_a,
            "lift_rel": (p_b - p_a) / p_a, "z": z, "p_value": p_value, "ci95": ci}

print(two_prop_ztest(500, 10000, 580, 10000))
# → z ≈ 2.50, p_value ≈ 0.0123, ci95 ≈ (0.0017, 0.0143)
```

With a library:

```python
from statsmodels.stats.proportion import proportions_ztest
stat, p = proportions_ztest(count=[580, 500], nobs=[10000, 10000])    # pip install statsmodels
```

## C6. Implementing experiments: deterministic bucketing with a feature flag 🚩

Users must see the **same variant every time** (consistent assignment) and assignment must be **random and uniform**: use a **hash**, not random() per request.

```python
import hashlib

def assign_variant(user_id: str, experiment: str, split: float = 0.5) -> str:
    """Return 'A' or 'B' deterministically for a user and experiment."""
    digest = hashlib.sha256(f"{experiment}:{user_id}".encode()).hexdigest()
    bucket = int(digest[:8], 16) / 0xFFFFFFFF          # uniform number in [0, 1]
    return "B" if bucket < split else "A"

# in a Flask route
@app.route("/checkout")
def checkout():
    variant = assign_variant(current_user.id, "ship-cost-early-v1")
    log_event("exposure", user=current_user.id, experiment="ship-cost-early-v1", variant=variant)
    return render_template("checkout_b.html" if variant == "B" else "checkout_a.html")
```

Then log **conversion events** with user id and variant; analyze counts per variant.

```
Exposure event → Conversion event → data warehouse (BigQuery/Redshift) → stats → dashboard
```

## C7. Pitfalls (very common exam/interview points) 🚨

| Pitfall | Problem | Remedy |
|---|---|---|
| **Peeking** | Checking p-value daily and stopping when "significant" inflates false positives | Fix sample size/duration in advance; use sequential testing methods if you must peek |
| **Multiple comparisons** | Testing many metrics/variants → some "win" by chance | Pre-register one primary metric; apply corrections (Bonferroni/FDR) |
| **Sample Ratio Mismatch (SRM)** | Split is not the planned 50/50 (e.g., 51.5/48.5 with big N) → assignment/tracking bug | Chi-square check of counts; fix before trusting results |
| **Novelty/primacy effect** | Users react to "new" then settle | Run ≥ 2 weekly cycles; look at trends over time |
| **Seasonality/day-of-week** | Weekends differ from weekdays | Full-week multiples |
| **Underpowered tests** | Too little traffic → inconclusive | Compute sample size first |
| **Interference (network effects)** | Users influence each other (marketplaces, social) | Cluster/geo randomization |
| **Simpson's paradox/segments** | Aggregate vs subgroup contradictions | Careful, pre-planned segmentation |
| **Ignoring guardrails** | Conversion up but latency/refunds worse | Always monitor guardrail metrics |
| **Optimizing local metrics** | Short-term clicks hurt long-term retention | Choose metrics tied to long-term value |
| **Ethics/consent** | Dark patterns, harming users | Review experiments; avoid manipulative tests; privacy law compliance (GDPR/DPDP) |
| **Flag/code debt** | Dead variants stay forever | Remove after decision |

SRM quick check (Python):

```python
import math
def srm_pvalue(n_a, n_b, expected_ratio=0.5):
    total = n_a + n_b
    exp_a, exp_b = total * expected_ratio, total * (1 - expected_ratio)
    chi2 = (n_a - exp_a)**2 / exp_a + (n_b - exp_b)**2 / exp_b
    return math.erfc(math.sqrt(chi2 / 2))       # chi-square df=1 survival function
print(srm_pvalue(10150, 9850))                  # small p → suspicious assignment
```

## C8. Tools 🧰

| Tool | Notes |
|---|---|
| **Optimizely, VWO, AB Tasty** | Web/product experimentation platforms |
| **LaunchDarkly, Statsig, Split, Eppo** | Feature flags + experimentation |
| **GrowthBook, Unleash, Flagsmith, PostHog** | Open-source/self-hostable options |
| **AWS CloudWatch Evidently / AppConfig** | AWS-native flags and experiments (check current status/availability) |
| **Google Optimize** | Was discontinued by Google in 2023 |
| **Analytics/warehouse** | GA4, Mixpanel, Amplitude, BigQuery, Redshift |

## C9. A/B testing inside the CI/CD flow 🔗

```
Idea (discovery) → hypothesis → implement variant behind a feature flag (TDD/BDD tests for both paths)
→ CI/CD deploys to production with flag OFF (deploy ≠ release) → ramp 1% → 50/50 experiment
→ metrics + guardrails monitored (Unit IV) → decision → roll out 100% or remove → delete flag (cleanup)
```

---

# 🅓 TECHNICAL DEBT MANAGEMENT

## D1. What is technical debt? ⭐⭐⭐

> **Technical debt** (a metaphor coined by **Ward Cunningham**) is the **implied cost of future rework** caused by choosing an easy or quick solution now instead of a better approach that would take longer.

```
Like financial debt:
  PRINCIPAL = the work needed to fix/clean the shortcut
  INTEREST  = extra effort you pay EVERY time you touch that code (slower changes, more bugs)
  If you never repay, interest compounds until development nearly stops ("bankruptcy" → rewrite)
```

Not all debt is bad: like a loan, **taking prudent debt consciously to hit a market window** can be smart **if you repay it**.

## D2. Fowler's Technical Debt Quadrant ⭐⭐

|  | **Reckless** | **Prudent** |
|---|---|---|
| **Deliberate** | "We don't have time for design." (bad) | "We must ship now and deal with the consequences." (conscious trade-off) |
| **Inadvertent** | "What's layering?" (lack of skill/knowledge) | "Now we know how we should have done it." (learning over time) |

Teams should aim to avoid **reckless** debt and **manage prudent** debt visibly.

## D3. Types of technical debt

| Type | Examples |
|---|---|
| **Code debt** | Duplication, long methods, poor naming, complex conditionals, "code smells" |
| **Design/architecture debt** | Monolith tangled modules, wrong boundaries, tight coupling, no layering |
| **Test debt** | Missing/flaky tests, low coverage, slow suites, manual regression |
| **Documentation/knowledge debt** | Outdated docs, tribal knowledge, no ADRs |
| **Build/CI-CD/infrastructure debt** | Manual deployments, snowflake servers, no IaC, slow pipelines, config drift (Unit II) |
| **Dependency/platform debt** | Outdated libraries/frameworks/runtimes, end-of-life versions |
| **Security debt** | Known vulnerabilities, weak auth, missing patches (Unit V) |
| **Data debt** | Poor schemas, inconsistent data, missing migrations |
| **Process/people debt** | Unclear ownership, no code review, knowledge silos |

## D4. Causes ⚠️

```
Deadline pressure / "ship now" culture      Changing or unclear requirements
Lack of skills or standards                 No refactoring time in plans; "feature factory" mindset
Weak or no automated tests                  Short-term vendor/library choices; copy-paste reuse
Fast growth without architecture work       Leaving team members / lost knowledge
Ignoring warnings (static analysis)         "Temporary" hacks that become permanent
```

## D5. Symptoms (how debt shows up) 🩺

```
Simple changes take long and break unrelated things       Bug rates rising, "fear of touching X"
Slow onboarding of new developers                           Long build/test times, flaky tests
Many TODO/FIXME/HACK comments                               Velocity declining sprint after sprint
Hot-fix culture; repeated incidents in the same area        Rewrites proposed as the only way out
```

## D6. Measuring technical debt 📏 ⭐

You can't manage what you can't see.

| Measure | Meaning |
|---|---|
| **SonarQube/SonarCloud Technical Debt (remediation effort)** | Estimated time to fix all maintainability issues (e.g., "12 days") |
| **Technical Debt Ratio** | Remediation cost ÷ development cost of the code (e.g., 5% = good; Sonar ratings A-E) |
| **Code smells, bugs, vulnerabilities, duplications** | Counts and density per KLOC |
| **Cyclomatic/cognitive complexity** | Branching complexity per function/file |
| **Code coverage** (on new code) | Risk indicator |
| **Code churn × complexity (hotspots)** | Files changed often and complex → highest interest |
| **Dependency age / outdated count** | Platform debt |
| **Build/test duration and flakiness** | Delivery debt |
| **Change failure rate, lead time, MTTR** (DORA) | Debt's impact on delivery |
| **Defect trends, rework %, time spent on unplanned work** | Interest actually being paid |
| **Developer surveys** | "How painful is X?" |

Maintainability rating (Sonar-style): A (≤5% ratio) ... E (>50%). Use **Quality Gates on new code** (the "Clean as You Code" approach): don't let *new* debt in, even if legacy debt exists.

SonarQube in a Jenkins pipeline (preview; Unit V expands):

```groovy
stage('Static Analysis') {
    steps {
        withSonarQubeEnv('sonar-server') {
            sh 'sonar-scanner -Dsonar.projectKey=flask-demo -Dsonar.sources=. -Dsonar.python.coverage.reportPaths=reports/coverage.xml'
        }
    }
}
stage('Quality Gate') {
    steps { timeout(time: 5, unit: 'MINUTES') { waitForQualityGate abortPipeline: true } }
}
```

## D7. Managing technical debt: strategies ⭐⭐⭐

### 1) Make debt **visible**
```
Keep a Technical Debt Register/backlog (items with: description, location, impact, estimated effort, risk, owner)
Label tickets (tech-debt); track with dashboards; discuss in refinement and retrospectives
Add TODOs with ticket IDs (TODO(JIRA-123)) not anonymous TODOs
```

### 2) Prevent new reckless debt
```
Definition of Done includes tests, review, static analysis, docs
Code reviews / pair programming, coding standards, linters/formatters in CI
TDD/BDD + CI quality gates (coverage, Sonar, security scans) on new code
Architecture guidelines, ADRs, design reviews for big changes
```

### 3) Pay down continuously ("repay little and often")

| Technique | Description |
|---|---|
| **Boy Scout Rule** | "Leave the code cleaner than you found it": small refactors with every change |
| **Refactoring** | Behavior-preserving improvement protected by tests (Part 3) |
| **Capacity allocation** | Reserve a fixed share of each Sprint/iteration (commonly **10-20%**) for debt/quality; or a "tech-debt story" every sprint |
| **Dedicated hardening/IP iteration** | SAFe's Innovation & Planning iteration or occasional debt-focused sprints (use sparingly: avoid making debt a "later" bucket) |
| **Opportunistic repayment** | Fix debt in areas you're about to change anyway (highest ROI) |
| **Upgrades on a schedule** | Dependabot/Renovate PRs; monthly dependency day |
| **Automate** | Auto-formatters, auto-fix linters, automated dependency updates |

### 4) Large-scale: **Strangler Fig pattern** 🌿 (replace legacy gradually, not a big-bang rewrite)

```
Put a facade/proxy in front of the legacy system
  → build new functionality as new services behind the facade
  → route traffic feature by feature to new implementation (feature flags/canary)
  → retire legacy pieces gradually until nothing is left
```

Why not rewrite from scratch? The "Second System" risk: long, expensive, repeats old mistakes, no value delivered until the end.

### 5) Prioritize which debt to pay first ⭐

Score each item by **interest (how much it hurts)**, **impact/risk**, **frequency of change**, and **effort**:

| Debt item | Pain (1-5) | Change frequency (1-5) | Risk (1-5) | Effort (days) | Priority = (Pain × Freq × Risk) ÷ Effort |
|---|---|---|---|---|---|
| Payment module has no tests | 5 | 4 | 5 | 8 | 100/8 = **12.5** ← do first |
| Outdated web framework (EOL) | 4 | 2 | 5 | 20 | 40/20 = **2.0** |
| Inconsistent naming in reports | 2 | 2 | 1 | 3 | 4/3 = **1.3** |
| Duplicate CSV export code | 3 | 3 | 2 | 2 | 18/2 = **9.0** |

Debt in **hot, risky, frequently-changed** code is expensive; debt in stable, rarely touched code can wait ("don't polish the basement").

### 6) Decide: **repay, refinance, or accept**

| Decision | When |
|---|---|
| **Repay now** | High interest, high risk, blocking |
| **Schedule** | Important, can plan with upcoming work |
| **Accept consciously** | Low impact/stable code; document the decision |
| **Delete** | Dead code/features (best debt is code that doesn't exist: YAGNI) |

### 7) Communicate with the business 💬

Translate debt into **money and risk**, not technical jargon:

```
"Each new payment feature currently takes 3 weeks instead of 1 because of tangled code (interest ≈ 2 extra weeks per feature).
 A 6-week cleanup reduces that to 1 week. We'll break even after ~3 features."
Use metrics: lead time, defect rate, incident count, onboarding time, DORA, cost of delay
```

### 8) Governance in SAFe/Scrum

```
Scrum: debt items are Product Backlog items ordered by the Product Owner; DoD protects quality; Retros surface debt
SAFe: Enablers (architectural/infrastructure/technical-debt work) are first-class backlog items, prioritized with WSJF;
      IP iteration and Inspect & Adapt address systemic debt
LeSS: "One Definition of Done" shared across teams raises the quality bar; refactoring is part of normal work
```

## D8. Debt and DevOps/IaC/CI-CD 🔗

```
CI/CD pipeline = automated debt gatekeeper (lint, tests, Sonar gate, dependency scan on every commit)
IaC (Unit II) reduces infrastructure/config debt: snowflakes and drift disappear
Feature flags create "flag debt": clean them up after rollout
Fast pipelines and good tests make repayment (refactoring) safe and cheap
```

## D9. Anti-patterns 🚨

| Anti-pattern | Why bad | Better |
|---|---|---|
| "We'll fix it later" with no ticket | Debt invisible; never repaid | Log it with owner and impact |
| 100% feature work, 0% debt | Interest compounds | Reserve capacity |
| Big-bang rewrite | High risk, no value until end | Strangler fig, incremental refactoring |
| Refactoring without tests | Breaks behavior | Add characterization tests first |
| Debt "sprints" every 6 months as the only repayment | Huge backlog, demoralizing | Continuous repayment + occasional focus |
| Treating all debt equally | Wasted effort | Prioritize by interest and risk |
| Blaming individuals | Hides problems | Blameless culture; systemic fixes |
| Measuring only with tools | Misses design/knowledge debt | Combine metrics + developer insight |

---

# 🔗 HOW A-B-C-D CONNECT

```
STRATEGY (A) decides the outcome to chase (OKR: raise activation) →
DISTRIBUTED TEAMS (B) collaborate async with CI/CD to build small increments →
A/B TESTS (C) prove which change moves the metric (feature flags + canary + monitoring) →
TECH DEBT MANAGEMENT (D) keeps the codebase fast to change so experiments and releases stay cheap
All sit on the pipeline from Parts 4-5.
```

---

# ✍️ EXAM ANSWER TEMPLATES

**A/B testing (8 marks):** definition + diagram (split → metric → compare) + process steps (hypothesis, metric, sample size, run, analyze, decide) + stats terms (p-value, α, power, CI) + small numeric example (5.0% vs 5.8%, z≈2.5, p≈0.012) + pitfalls (peeking, SRM) + A/B vs canary + tools + conclusion.

**Technical debt (8 marks):** definition (Ward Cunningham, principal/interest) + quadrant diagram + types + causes + measurement (Sonar debt ratio, complexity, churn) + management strategies (visible backlog, Boy Scout, capacity %, refactoring, strangler fig, quality gates) + prioritization + conclusion.

**Distributed teams (6 marks):** definition/models + challenges (time zones, communication) + practices (async-first, overlap, working agreements, tools, CI/CD, docs) + security + conclusion.

**Product strategy (6 marks):** vision→strategy→roadmap→backlog + MVP/Lean + OKR + prioritization (RICE/MoSCoW/Kano) + linking to agile delivery.

---

# 🎯 VIVA QUESTIONS (PART 6)

**Q1. Difference between product vision, strategy and roadmap?** Vision is the long-term why; strategy is how we will win; roadmap sequences themes/initiatives over time; backlog holds the detailed work.

**Q2. What is an MVP?** The smallest product version that enables maximum validated learning with minimum effort.

**Q3. What is RICE?** Reach × Impact × Confidence ÷ Effort for prioritizing features.

**Q4. What are OKRs?** Objectives with measurable Key Results that express outcomes.

**Q5. Output vs outcome?** Output is what you ship; outcome is the change in user/business behavior that results.

**Q6. What is the Kano model?** Classifies features as basic, performance and delighters by their effect on satisfaction.

**Q7. Name challenges of distributed teams.** Time zones, communication gaps, trust, onboarding, tool overload, security, culture.

**Q8. How do you run Agile in a distributed team?** Async stand-ups, overlap-hour planning, online boards/retros, strong documentation, small PRs, CI/CD automation, working agreements.

**Q9. What is follow-the-sun?** Handing work across time zones to progress nearly 24 hours a day.

**Q10. What is A/B testing?** Randomized controlled experiment comparing a control and a variant on a metric to see if the variant is statistically better.

**Q11. Explain p-value, α and power.** p-value: probability of at least this extreme a result under no difference; α: accepted false-positive rate (0.05); power: probability (1-β), typically 80%, of detecting a real effect.

**Q12. What is peeking and why is it bad?** Stopping a test when it first looks significant inflates false positives; fix sample size in advance.

**Q13. What is SRM?** Sample Ratio Mismatch: observed group sizes deviate from the planned split, signalling assignment/tracking bugs.

**Q14. A/B test vs canary release?** A/B tests whether a variant improves a business metric; canary tests whether a release is technically safe.

**Q15. What is technical debt?** The future rework cost incurred by choosing quick or poor solutions now.

**Q16. Explain principal and interest in technical debt.** Principal is the effort to fix the shortcut; interest is the extra effort paid each time you work around it.

**Q17. How do you measure technical debt?** Remediation time and debt ratio (SonarQube), complexity, duplication, coverage, churn hotspots, dependency age, DORA metrics, rework.

**Q18. Strategies to manage it?** Visible debt backlog, Boy Scout rule, refactoring with tests, capacity allocation (10-20%), quality gates, strangler fig, prioritization by interest and risk.

**Q19. What is the Strangler Fig pattern?** Gradually replacing a legacy system by routing functionality to new components behind a facade until the legacy is retired.

**Q20. How do feature flags relate to debt?** They enable safe releases and experiments but create flag debt if not removed.

---

# 🧠 PART 6 MASTER SHEET

```
(A) PRODUCT STRATEGY: vision → strategy → roadmap → backlog ; outcome > output ; MVP + Build-Measure-Learn + pivot ;
    dual-track (discovery + delivery) ; OKRs ; North Star ; AARRR ; roadmaps (feature, outcome, Now-Next-Later)
    prioritization: RICE=(R×I×C)/E, ICE, MoSCoW, Kano (basic/performance/delighters), Value-Effort, WSJF
(B) DISTRIBUTED TEAMS: remote-first/hybrid/follow-the-sun ; challenges: time zones, comms, trust, onboarding, security
    practices: async-first + written culture, overlap hours, working agreements, adapted ceremonies, CI/CD + IaC + docs, tools, wellbeing
(C) A/B TESTING: random split → compare metric ; hypothesis → metrics (primary + guardrails) → sample size → run → analyze → decide → cleanup
    stats: H0/H1, α=0.05, p-value, power 80%, CI, MDE ; n ≈ 16·p(1−p)/δ² ; example 5.0% vs 5.8% → z≈2.50, p≈0.012
    pitfalls: peeking, multiple testing, SRM, novelty, seasonality, underpowered ; deterministic hash bucketing ; flags
    A/B (is it better?) vs canary (is it safe?)
(D) TECH DEBT: Ward Cunningham ; principal + interest ; Fowler quadrant (deliberate/inadvertent × reckless/prudent)
    types: code, design, test, doc, infra/CI-CD, dependency, security ; measure: Sonar debt ratio, complexity, churn hotspots, DORA
    manage: visible register, prevent (DoD, reviews, gates), repay continuously (Boy Scout, refactor, 10-20% capacity),
            strangler fig, prioritize (pain×frequency×risk÷effort), communicate in money/time
```

---

## 🫡 PART 6 STATUS

You can now explain product strategy and prioritization, run and analyze an A/B test with numbers, structure and manage distributed agile teams, and measure and pay down technical debt.

**STOP here.** Next up:

> **PART 7 — Best Practices for Agile Collaboration and Deployment 🤝🚀**
