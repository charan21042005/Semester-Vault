# 🟢 UNIT III — PART 1: AGILE FROM ABSOLUTE SCRATCH

> **Covered in this document:** Unit III roadmap → how software was built before Agile (Waterfall) and why it failed → Agile Manifesto (4 values, 12 principles) → Agile vs Waterfall → agile frameworks (Scrum, Kanban, XP, Lean) → Scrum in full (roles, events, artifacts, Definition of Done, Sprint Goal) → user stories (INVEST), acceptance criteria, epics, story points, estimation (planning poker) → backlog refinement and prioritization → Kanban (WIP limits, flow, boards) → Extreme Programming practices → agile metrics (velocity, burndown, cycle time, lead time, throughput, CFD) → how Agile and DevOps fit together → a full sprint walk-through (dry run) → common mistakes (anti-patterns) → viva → master sheet.

---

Absolutely mate. 😤🔥 Unit III starts now.

Same exact system as Units I and II:

> Absolute scratch → intuition → concept → diagram → code → dry run → practical thinking → exam/viva → STOP.

Your Unit III syllabus is:

> **Agile and CI/CD Practices for Scalable Product Development:** Scaling Agile (SAFe, LeSS), TDD, BDD, CI/CD, product strategy, distributed teams, A/B testing, technical debt management, Jenkins automation, and best practices for agile collaboration and deployment.

Practical attached to this unit:

```
Practical 6 → Deploy Jenkins, integrate it with GitHub using webhooks, configure an automated CI/CD pipeline,
              execute build and testing stages, and automatically deploy the application upon successful pipeline execution
```

---

# 🚀 UNIT III — ROADMAP

```
PART 1 — Agile from Absolute Scratch ⭐                              (this file)
PART 2 — Scaling Agile: SAFe, LeSS (+ Scrum@Scale, Nexus, Spotify model) 🔥
PART 3 — TDD and BDD (Red-Green-Refactor, Gherkin, test pyramid, code) 🧪
PART 4 — CI/CD Concepts (pipelines, branching, deployment strategies, feature flags, DORA) 🔄
PART 5 — Jenkins Deep Dive (architecture, Jenkinsfile, agents, credentials, webhooks) 🔥
PART 6 — Product Strategy, Distributed Teams, A/B Testing, Technical Debt Management
PART 7 — Best Practices for Agile Collaboration and Deployment
PART 8 — Practical Lab Guide (Jenkins + GitHub webhook + CI/CD pipeline, step by step)

🏆 GRAND CONSOLIDATION — revision, comparison tables, 30 viva Qs, traps, master sheet
```

---

# 1️⃣ THE WORLD BEFORE AGILE: WATERFALL 🌊

## 1.1 What is Waterfall?

A **plan-driven, sequential** method: finish each phase completely before the next begins.

```
Requirements ──▶ Design ──▶ Implementation ──▶ Testing ──▶ Deployment ──▶ Maintenance
  (months)       (months)      (months)         (months)     (finally!)
```

## 1.2 Why it hurt

| Problem | What happened |
|---|---|
| **Late feedback** | Customer sees the product after 12-18 months |
| **Requirements change** | Business needs shift mid-project; the plan can't absorb it |
| **Late testing** | Bugs found at the end are expensive to fix |
| **Big-bang release** | One giant risky launch |
| **Wasted effort** | Features built to a spec that was wrong |
| **Blame culture** | Dev vs QA vs Ops silos |

Restaurant analogy 🍽️: Waterfall = you order a 10-course meal, the chef disappears for 3 hours, and you see the food at the end. Agile = small dishes arrive quickly; you say "less salt next time."

---

# 2️⃣ WHAT IS AGILE? ⭐

> **Agile** is an iterative, incremental approach to building software that values **working software, customer collaboration and responding to change**, delivering in short cycles and learning continuously.

```
Waterfall: plan everything → build everything → deliver once
Agile:     plan a little → build a little → deliver a little → learn → repeat
```

## 2.1 The Agile Manifesto (2001) ⭐⭐⭐ (WRITE THIS IN EXAMS)

Seventeen practitioners met in Utah and wrote four value statements:

```
We value:
  Individuals and interactions        over   processes and tools
  Working software                    over   comprehensive documentation
  Customer collaboration              over   contract negotiation
  Responding to change                over   following a plan

"That is, while there is value in the items on the right, we value the items on the left more."
```

## 2.2 The 12 principles (short form) 📜

| # | Principle (paraphrased) |
|---|---|
| 1 | Satisfy the customer through early and continuous delivery of valuable software |
| 2 | Welcome changing requirements, even late in development |
| 3 | Deliver working software frequently (weeks rather than months) |
| 4 | Business people and developers work together daily |
| 5 | Build projects around motivated individuals; trust them |
| 6 | Face-to-face conversation is the most effective communication |
| 7 | Working software is the primary measure of progress |
| 8 | Sustainable development: maintain a constant pace |
| 9 | Continuous attention to technical excellence and good design |
| 10 | Simplicity: maximize the amount of work *not* done |
| 11 | Best architectures and designs emerge from self-organizing teams |
| 12 | Regularly reflect and adjust (retrospectives) |

Memory trick: **"Deliver early, welcome change, collaborate daily, trust people, keep it simple, reflect and improve."**

## 2.3 Agile vs Waterfall ⭐

| Aspect | Waterfall | Agile |
|---|---|---|
| Approach | Sequential phases | Iterative and incremental |
| Planning | Detailed upfront | Adaptive, just-in-time |
| Requirements | Fixed early | Evolve continuously |
| Delivery | One big release | Frequent small releases |
| Customer involvement | At start and end | Throughout |
| Testing | After build | Continuous, inside every iteration |
| Change | Costly, discouraged | Welcomed |
| Risk | Found late | Found early |
| Documentation | Heavy | Just enough |
| Best for | Stable, well-understood, regulated requirements | Uncertain, fast-changing products |
| Team | Role-siloed | Cross-functional, self-organizing |

---

# 3️⃣ THE AGILE FAMILY OF METHODS 👨‍👩‍👧‍👦

Agile is a mindset; these are concrete frameworks:

| Framework | Core idea | Cadence |
|---|---|---|
| **Scrum** | Time-boxed sprints with defined roles, events, artifacts | Sprints (1-4 weeks) |
| **Kanban** | Visualize work, limit work-in-progress, optimize flow | Continuous flow |
| **XP (Extreme Programming)** | Engineering excellence: TDD, pair programming, CI, refactoring | 1-2 week iterations |
| **Lean software development** | Eliminate waste, amplify learning, deliver fast | Continuous |
| **Scrumban** | Scrum roles + Kanban flow | Hybrid |
| **Crystal, FDD, DSDM** | Lesser-used variants | Varies |

Most teams today use **Scrum and/or Kanban** with **XP engineering practices** (TDD, CI), which is exactly where this unit goes.

---

# 4️⃣ SCRUM IN FULL ⭐⭐⭐

> **Scrum** is a lightweight framework in which a small cross-functional team delivers a potentially releasable product **Increment** every **Sprint** (a fixed time-box of one month or less), inspecting and adapting along the way.

Scrum rests on **three pillars**: **Transparency, Inspection, Adaptation**.
Scrum values: **Commitment, Focus, Openness, Respect, Courage**.

## 4.1 The Scrum Team (Scrum Guide 2020: one team, three accountabilities)

| Accountability | What they do |
|---|---|
| **Product Owner (PO)** | Maximizes the product's value; owns and orders the **Product Backlog**; clearly communicates the **Product Goal** |
| **Scrum Master (SM)** | Ensures Scrum is understood and enacted; removes impediments; coaches the team and organization (servant leader, not a boss) |
| **Developers** | Everyone building the Increment: devs, testers, designers, DevOps engineers; own the Sprint Backlog, quality and the Definition of Done |

Typical team size: **10 or fewer** people. Self-managing and cross-functional.

## 4.2 The five Scrum events ⭐

All events are time-boxed; the **Sprint** contains all the others.

```
┌──────────────────────────── SPRINT (1-4 weeks) ────────────────────────────┐
│ Sprint Planning → [ Daily Scrum every day ] → Sprint Review → Retrospective │
└──────────────────────────────────────────────────────────────────────────────┘
```

| Event | Purpose | Max time (for a 1-month Sprint) | Who |
|---|---|---|---|
| **Sprint** | Container: idea → Increment | ≤ 1 month | Team |
| **Sprint Planning** | Decide WHY (Sprint Goal), WHAT (backlog items), HOW (plan) | 8 hours | Whole team |
| **Daily Scrum** | Inspect progress toward Sprint Goal and adapt the plan | 15 minutes | Developers |
| **Sprint Review** | Inspect the Increment with stakeholders; adapt the backlog; get feedback | 4 hours | Team + stakeholders |
| **Sprint Retrospective** | Inspect how the team worked; plan improvements | 3 hours | Scrum Team |

(Shorter Sprints → shorter events. A 2-week sprint: planning ~4 h, review ~2 h, retro ~1.5 h.)

## 4.3 The three artifacts and their commitments

| Artifact | Commitment | Meaning |
|---|---|---|
| **Product Backlog** | **Product Goal** | Ordered list of everything the product may need |
| **Sprint Backlog** | **Sprint Goal** | Items selected for the Sprint + the plan to deliver them |
| **Increment** | **Definition of Done (DoD)** | A usable, integrated, tested step toward the Product Goal |

### Definition of Done (DoD) ⭐

A shared checklist that says when work is truly done. Example:

```
☐ Code written and peer-reviewed (pull request approved)
☐ Unit tests written and passing; coverage ≥ 80% on changed code
☐ CI pipeline green (build + tests + static analysis + security scan)
☐ Acceptance criteria met and demonstrated
☐ Documentation/README/API docs updated
☐ Deployed to staging; no critical bugs
☐ Product Owner accepted
```

Don't confuse: **DoD** = quality bar for *every* item; **Acceptance criteria** = conditions for *one* specific story.

## 4.4 Scrum cycle diagram (draw in exams) 🔁

```
 Product Backlog ──▶ Sprint Planning ──▶ Sprint Backlog ──▶ [ Sprint: Daily Scrum + building + testing ]
        ▲                                                              │
        │                                                              ▼
   (refinement)            Retrospective ◀── Sprint Review ◀── Increment (potentially releasable)
        └───────────── feedback and new insights ◀────────────────────┘
```

---

# 5️⃣ USER STORIES, EPICS AND ESTIMATION 🧾

## 5.1 User story format

```
As a <type of user>, I want <goal/feature>, so that <benefit/value>.
```

Example: *As a shopper, I want to save items to a wishlist, so that I can buy them later.*

### INVEST checklist for good stories ✅

| Letter | Meaning |
|---|---|
| **I**ndependent | Can be built in any order |
| **N**egotiable | Details discussed, not a fixed contract |
| **V**aluable | Delivers value to a user/customer |
| **E**stimable | Team can size it |
| **S**mall | Fits within one Sprint |
| **T**estable | Clear criteria to verify |

### Acceptance criteria (Given-When-Then; leads into BDD in Part 3)

```
Given I am logged in and viewing a product
When I click "Add to wishlist"
Then the product appears in my wishlist and the heart icon turns red
```

## 5.2 Work hierarchy

```
Theme / Initiative   (strategic goal)
  └── Epic           (big feature, spans many sprints)   e.g., "Wishlist"
        └── User Story  (fits in a sprint)               e.g., "Add item to wishlist"
              └── Task  (technical step, hours)          e.g., "Create wishlist table", "Write API endpoint"
```

## 5.3 Story points and estimation

**Story points** are a *relative* measure of effort + complexity + uncertainty (not hours). Common scale: **Fibonacci** 1, 2, 3, 5, 8, 13, 21.

**Planning Poker:** each member privately picks a card; reveal together; discuss outliers; re-vote until agreement. Avoids anchoring bias.

| Technique | Idea |
|---|---|
| Planning poker | Consensus with cards |
| T-shirt sizing | XS, S, M, L, XL for rough epic sizing |
| Affinity estimation | Sort items by relative size |
| No-estimates / throughput forecasting | Count items instead of points |

## 5.4 Backlog refinement and prioritization

Ongoing activity (not a formal event): break down, clarify and order backlog items so the top is "ready".

| Prioritization method | How |
|---|---|
| **MoSCoW** | Must / Should / Could / Won't |
| **Value vs Effort matrix** | Quick wins first |
| **WSJF** (SAFe) | Weighted Shortest Job First = Cost of Delay ÷ Job size |
| **RICE** | Reach × Impact × Confidence ÷ Effort (Part 6) |
| **Kano model** | Basic / performance / delighters (Part 6) |

---

# 6️⃣ KANBAN 📋

> **Kanban** (Japanese for "signboard") visualizes work on a board, limits work in progress (WIP), and manages flow.

```
 BACKLOG │  TO DO │ IN PROGRESS (WIP ≤ 3) │ IN REVIEW (WIP ≤ 2) │ TESTING │ DONE
 ────────┼────────┼───────────────────────┼─────────────────────┼─────────┼──────
  card   │  card  │  card  card  card     │  card  card         │  card   │ card
```

## 6.1 Kanban principles and practices

```
1. Visualize the workflow (board)         4. Make policies explicit (what "done" means per column)
2. Limit Work In Progress (WIP limits)    5. Implement feedback loops (stand-ups, reviews)
3. Manage flow (measure cycle time)       6. Improve collaboratively, evolve experimentally
```

**Why limit WIP?** Starting more things slows everything (context switching). "Stop starting, start finishing." Little's Law: **Cycle time = WIP ÷ Throughput**. Lower WIP → faster delivery.

## 6.2 Scrum vs Kanban ⭐

| | Scrum | Kanban |
|---|---|---|
| Cadence | Fixed-length sprints | Continuous flow |
| Roles | PO, SM, Developers | No prescribed roles |
| Commitment | Sprint Goal / backlog for the sprint | WIP limits |
| Change mid-cycle | Discouraged during a sprint | Allowed any time (if WIP allows) |
| Metrics | Velocity, burndown | Cycle time, throughput, CFD |
| Best for | Product development with planned increments | Support/ops, maintenance, unpredictable flow |

---

# 7️⃣ EXTREME PROGRAMMING (XP) PRACTICES ⚙️ (links to TDD/CI)

XP is the engineering side of Agile:

| Practice | Meaning |
|---|---|
| **Test-Driven Development (TDD)** | Write the test first (Part 3) |
| **Pair programming** | Two developers, one workstation (driver + navigator) |
| **Continuous Integration** | Integrate and test many times a day (Parts 4-5) |
| **Refactoring** | Improve code structure without changing behavior |
| **Simple design** | Do the simplest thing that works (YAGNI: You Aren't Gonna Need It) |
| **Small releases** | Frequent deployment |
| **Collective code ownership** | Anyone can improve any code |
| **Coding standards** | Shared style |
| **Sustainable pace** | No death marches |
| **On-site customer** | Customer representative always available |

XP values: **Communication, Simplicity, Feedback, Courage, Respect**.

---

# 8️⃣ AGILE METRICS 📊

| Metric | Definition | Use / caution |
|---|---|---|
| **Velocity** | Story points completed per Sprint | Forecast capacity; **never** compare teams or use as a performance target |
| **Burndown chart** | Remaining work vs time in a Sprint | Shows if Sprint Goal is on track |
| **Burnup chart** | Work completed vs total scope over time | Shows scope changes clearly |
| **Cycle time** | Time from "started" to "done" for an item | Flow efficiency |
| **Lead time** | Time from request to delivery | Customer-perceived speed |
| **Throughput** | Items completed per period | Forecasting (Kanban) |
| **Cumulative Flow Diagram (CFD)** | Items per state over time | Spot bottlenecks (widening bands) |
| **Defect escape rate** | Bugs found in production vs total | Quality |
| **Sprint Goal success rate** | % of sprints meeting goal | Predictability |
| **Customer satisfaction/NPS** | User feedback | Value |

Sample burndown (10-day sprint, 40 points):

```
Points left
40 ●
32    ●
28       ●   ← ideal line is straight; actual line tells the story
20          ●
12             ●
 0 ─────────────────● Day 10
```

---

# 9️⃣ AGILE + DEVOPS: HOW THEY FIT TOGETHER 🤝 ⭐

| | Agile | DevOps |
|---|---|---|
| Focus | How teams **plan and build** software iteratively | How teams **deliver and operate** software continuously |
| Boundary | Dev team + business | Dev + Ops + QA + Security |
| Key practices | Sprints, stories, retros | CI/CD, IaC, monitoring, automation |
| Goal | Build the right thing, adapt to change | Ship it safely, quickly and reliably |

```
Agile gives you small, frequent, well-understood changes.
DevOps (CI/CD + IaC + monitoring) gets each change to users safely and quickly.
Together: idea → backlog → code → test → deploy → observe → learn → backlog (continuous loop)
```

Scrum's "potentially releasable Increment every Sprint" is only realistic with **automated builds, tests and deployments**, i.e., CI/CD (Parts 4-5).

---

# 🔟 DRY RUN: A TWO-WEEK SPRINT FROM START TO FINISH 🏃

Team: PO Asha, SM Ravi, 5 Developers. Product: online bookstore. Product Goal: "Increase repeat purchases."

```
Day 0   Backlog refinement: stories sized (wishlist=5, reviews=8, coupon=3, export=13)
Day 1   SPRINT PLANNING
        WHY : Sprint Goal = "Customers can save books and apply coupons"
        WHAT: pulls Wishlist (5) + Coupon (3) + 2 bug fixes (2) = 10 points (team's velocity ~10-12)
        HOW : breaks into tasks; sets up CI pipeline checks
Day 2-9 Daily Scrum 15 min: "progress toward Sprint Goal? any impediments?"
        Developers write tests first (TDD), commit small changes, PR review, Jenkins pipeline runs on every push
        Day 5: PO clarifies coupon rules; scope adjusts without changing Sprint Goal
        Day 7: build breaks → fixed within 10 minutes (team rule: stop the line)
Day 9   Increment deployed to staging automatically; DoD checked
Day 10  SPRINT REVIEW: demo wishlist + coupon to stakeholders; feedback: "add share-wishlist link" → new backlog item
        SPRINT RETROSPECTIVE: Went well: fast PR reviews. Problem: flaky test. Action: fix flaky test, add contract test (owner: Meera, due next sprint)
Next sprint starts immediately; velocity recorded: 10 points
```

---

# 1️⃣1️⃣ COMMON AGILE ANTI-PATTERNS 🚨

| Anti-pattern | Why it's bad | Better |
|---|---|---|
| **"Scrum-fall" / mini-waterfall** (design week, build week, test week) | No real iteration or feedback | Slice vertically: small stories end-to-end |
| **Skipping retrospectives** | No improvement | Keep them; track actions |
| **Velocity as a performance KPI** | Teams inflate estimates | Use for planning only |
| **Scrum Master as project manager** | Kills self-management | Servant leader/coach |
| **Giant stories** | Can't finish in a sprint | Split (INVEST) |
| **No Definition of Done** | "Done" means different things | Written DoD |
| **PO unavailable** | Blocked decisions | Empower PO, daily contact |
| **Daily Scrum as status report to manager** | No collaboration | Team-focused plan for next 24 h |
| **Technical debt ignored** | Slows future sprints | Reserve capacity; see Part 6 |
| **Testing at the end of the sprint** | Late bugs | Test continuously (TDD/BDD/CI) |
| **Agile in name only** ("we do stand-ups") | No change in values | Adopt the mindset and engineering practices |

---

# 1️⃣2️⃣ VIVA QUESTIONS (PART 1) 🎯

**Q1. What is Agile?** An iterative and incremental approach that delivers working software in short cycles with continuous customer feedback and adaptation to change.

**Q2. State the four values of the Agile Manifesto.** Individuals and interactions over processes and tools; working software over comprehensive documentation; customer collaboration over contract negotiation; responding to change over following a plan.

**Q3. Waterfall vs Agile?** Waterfall is sequential with late feedback and fixed requirements; Agile is iterative, welcomes change and delivers frequently.

**Q4. Scrum roles?** Product Owner, Scrum Master, Developers.

**Q5. Scrum events?** Sprint, Sprint Planning, Daily Scrum, Sprint Review, Sprint Retrospective.

**Q6. Scrum artifacts and commitments?** Product Backlog (Product Goal), Sprint Backlog (Sprint Goal), Increment (Definition of Done).

**Q7. What is the Definition of Done?** A shared quality checklist that an item must meet to be considered complete.

**Q8. What is a user story and INVEST?** A short requirement as "As a... I want... so that..."; INVEST = Independent, Negotiable, Valuable, Estimable, Small, Testable.

**Q9. What is velocity?** Story points completed per Sprint, used for forecasting, not performance evaluation.

**Q10. Scrum vs Kanban?** Scrum uses fixed sprints and roles; Kanban uses continuous flow with WIP limits.

**Q11. Why limit WIP?** Reduces context switching and cycle time, exposes bottlenecks (Little's Law).

**Q12. Name XP practices.** TDD, pair programming, CI, refactoring, simple design, small releases, collective ownership.

**Q13. How do Agile and DevOps relate?** Agile organizes iterative development; DevOps automates delivery and operations; together they enable continuous value delivery.

**Q14. What happens in Sprint Retrospective?** The team inspects its process and defines improvements for the next Sprint.

---

# 🧠 PART 1 MASTER SHEET

```
Waterfall (sequential, late feedback) → AGILE (iterative, adaptive, early feedback)
Manifesto: Individuals>Process | Working software>Docs | Collaboration>Contracts | Change>Plan ; 12 principles
SCRUM: 3 accountabilities (PO, SM, Developers) | 5 events (Sprint, Planning, Daily, Review, Retro) | 3 artifacts (PB, SB, Increment)
Commitments: Product Goal | Sprint Goal | Definition of Done ; pillars: Transparency, Inspection, Adaptation
Stories: As a/I want/so that + INVEST + acceptance criteria (Given-When-Then) ; epic > story > task
Estimation: story points (Fibonacci), planning poker ; priority: MoSCoW, WSJF, RICE
KANBAN: visualize, limit WIP, manage flow ; metrics: cycle time, lead time, throughput, CFD
XP: TDD, pair, CI, refactor, simple design, small releases
Metrics: velocity (planning only), burndown/burnup
Agile (build iteratively) + DevOps (deliver continuously) = continuous value
```

---

## 🫡 PART 1 STATUS

You now understand why Agile exists, how Scrum and Kanban work, how stories and estimation are done, and how Agile connects to DevOps.

**STOP here.** Next up:

> **PART 2 — Scaling Agile: SAFe, LeSS (plus Scrum@Scale, Nexus, Spotify model) 🔥**
