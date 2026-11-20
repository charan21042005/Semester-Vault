# 🔵 UNIT III — PART 2: SCALING AGILE (SAFe, LeSS, AND THE OTHERS)

> **Covered in this document:** why one Scrum team is not enough → the problems of scale (dependencies, integration, alignment, architecture, culture) → **SAFe** in depth (what it is, values, lean-agile mindset, principles, 7 core competencies, configurations, Agile Release Train, Program Increment, PI Planning, roles, Solution Train, Lean Portfolio Management, WSJF, Continuous Delivery Pipeline, DevOps in SAFe, Inspect and Adapt, criticisms) → latest SAFe changes (no more version numbers, AI-Native SAFe) → **LeSS** in depth (principles, LeSS vs LeSS Huge, feature teams, one backlog, Sprint Planning 1 and 2, overall retrospective, Requirement Areas, criticisms) → Scrum@Scale, Nexus, Scrum of Scrums, Spotify model, DAD → SAFe vs LeSS vs others comparison table → how to choose → Conway's Law and team topology → scaling and DevOps/CI/CD link → worked PI-planning dry run → exam answer templates → viva → master sheet.

---

Absolutely mate. 😤🔥 Part 1 covered **one team**. Real products (banking apps, e-commerce giants, telecom platforms) need **dozens or hundreds of people**. That is where **scaling frameworks** come in: your syllabus names **SAFe** and **LeSS**.

---

# 1️⃣ WHY SCALING IS HARD 🏔️

## 1.1 One team vs many teams

```
1 Scrum team  = 5-9 people   → easy coordination
10 teams      = 70-90 people → dependencies, shared code, conflicting priorities, integration pain
```

## 1.2 Scaling problems (list in exams)

| Problem | Example |
|---|---|
| **Dependencies between teams** | Team A's feature needs Team B's API first |
| **Integration complexity** | Ten teams' code must work together in one product |
| **Alignment** | Teams optimize locally; overall goals drift |
| **Shared architecture and tech debt** | Changes ripple across the system |
| **Prioritization** | Many stakeholders, one budget |
| **Governance and compliance** | Large orgs need audit trails, budgeting, risk management |
| **Communication overhead** | N teams → N(N−1)/2 communication paths |
| **Culture and leadership** | Managers used to command-and-control |
| **Release coordination** | Everyone must ship together |

Communication paths rule: 5 teams → 10 paths; 10 teams → 45 paths. 

## 1.3 Two broad philosophies

```
ADD structure/roles/process to coordinate teams       →  SAFe (more prescriptive)
REMOVE structure/roles; keep Scrum minimal, scaled up →  LeSS (more minimalist)
```

---

# 2️⃣ SAFe: SCALED AGILE FRAMEWORK ⭐⭐⭐

## 2.1 What is SAFe?

> **SAFe (Scaled Agile Framework)** is a knowledge base of proven principles, roles, events and practices for achieving **business agility** at enterprise scale, combining **Lean, Agile and DevOps** thinking. Created by Dean Leffingwell (first described around 2007) and maintained by **Scaled Agile, Inc.**

It is the **most widely adopted** scaling framework in large enterprises (banks, telecom, government).

## 2.2 SAFe's foundation: values, mindset, principles

**Four core values:** **Alignment, Transparency, Respect for People, Courage.**

**Lean-Agile Mindset (the "house of Lean"):** goal = deliver the **sustainably shortest lead time** with best quality and value; pillars: respect for people and culture, flow, innovation, relentless improvement; roof: value; foundation: Lean-Agile leadership.

**SAFe Lean-Agile principles (10):**

```
1. Take an economic view                      6. Visualize and limit WIP, reduce batch sizes, manage queue lengths
2. Apply systems thinking                     7. Apply cadence, synchronize with cross-domain planning
3. Assume variability; preserve options       8. Unlock the intrinsic motivation of knowledge workers
4. Build incrementally with fast, integrated  9. Decentralize decision-making
   learning cycles                           10. Organize around value
5. Base milestones on objective evaluation of working systems
```

## 2.3 Seven Core Competencies (SAFe 6.0 structure) ⭐

| # | Core competency | Meaning |
|---|---|---|
| 1 | **Lean-Agile Leadership** | Leaders model and enable the mindset |
| 2 | **Team and Technical Agility** | High-performing teams using Scrum/Kanban/XP + technical practices |
| 3 | **Agile Product Delivery** | Customer-centricity, design thinking, DevOps and release on demand |
| 4 | **Enterprise Solution Delivery** | Building large, complex systems and cyber-physical solutions |
| 5 | **Lean Portfolio Management** | Strategy, funding, governance via value streams |
| 6 | **Organizational Agility** | Lean business operations, strategic agility |
| 7 | **Continuous Learning Culture** | Learning organization, innovation, relentless improvement |

## 2.4 SAFe configurations (levels) ⭐

| Configuration | For | Includes |
|---|---|---|
| **Essential SAFe** | One Agile Release Train (ART) | Teams + ART; the minimum viable SAFe |
| **Large Solution SAFe** | Large systems needing multiple ARTs | + Solution Train, capabilities |
| **Portfolio SAFe** | Strategy, funding and governance | + Lean Portfolio Management, Strategic Themes |
| **Full SAFe** | Entire enterprise | All levels together |

Levels in the Big Picture: **Team → Program (ART) → Large Solution → Portfolio**.

## 2.5 The Agile Release Train (ART) ⭐⭐⭐

> An **Agile Release Train** is a long-lived **team of agile teams** (typically **50-125 people**) that plans, commits, develops and delivers value together on a **fixed cadence**, aligned to a **value stream**.

```
ART  =  5-12 Agile teams  +  shared roles (RTE, Product Management, System Architect)
        all working to the same PI objectives, same cadence, same planning, same demo
```

Train analogy 🚆: the train leaves the station on a fixed schedule (every PI). Teams can't hold the train; features not ready catch the next one.

## 2.6 Program Increment (PI) ⭐⭐⭐

> A **PI** is a fixed time-box (typically **8-12 weeks**, commonly **10 weeks**) containing several iterations (sprints), ending with an **Innovation and Planning (IP) iteration**.

```
PI (10 weeks)
 ┌───────┬───────┬───────┬───────┬──────────────────────┐
 │ Iter1 │ Iter2 │ Iter3 │ Iter4 │ IP iteration         │
 │ 2 wks │ 2 wks │ 2 wks │ 2 wks │ (Innovation+Planning)│
 └───────┴───────┴───────┴───────┴──────────────────────┘
     ▲ System Demo at end of each iteration          ▲ Inspect & Adapt + next PI Planning
```

IP iteration: buffer for innovation, hackathons, learning, tech-debt reduction, and **PI planning** for the next PI.

## 2.7 PI Planning ⭐⭐⭐ (SAFe's heartbeat)

A **face-to-face (or virtual) 2-day event** where all teams on the ART plan the next PI together.

```
DAY 1                                           DAY 2
 • Business context (executives)                 • Planning adjustments (new info, scope changes)
 • Product/solution vision (Product Mgmt)        • Final team plans + risks (ROAM)
 • Architecture/dev practices vision             • Confidence vote (fist of five)
 • Team breakouts: draft plans + PI objectives   • Plan rework if confidence < 3
 • Draft plan review                             • Plan rework/commit
 • Management review + problem solving           • Retrospective of the event
```

Outputs:

| Output | Meaning |
|---|---|
| **Committed PI Objectives** | Each team's business-value-rated goals for the PI |
| **Program Board** | Features by iteration, with **dependencies** shown between teams |
| **Risks (ROAM)** | **R**esolved, **O**wned, **A**ccepted, **M**itigated |
| **Confidence vote** | Fist of five; ≥ 3 average means commit |

## 2.8 SAFe roles ⭐

| Role | Level | Responsibility |
|---|---|---|
| **Release Train Engineer (RTE)** | ART | "Chief Scrum Master" of the train: facilitates PI planning, ART sync, removes impediments |
| **Product Management** | ART | Owns Program Backlog; defines features/vision; customer focus |
| **System Architect/Engineer** | ART | Defines architecture, technical enablers, ensures solution integrity |
| **Business Owners** | ART | Key stakeholders accountable for business outcomes; rate PI objectives |
| **Product Owner (team)** | Team | Team backlog, story acceptance |
| **Scrum Master/Team Coach** | Team | Facilitates team |
| **Agile Team** | Team | Cross-functional, 5-11 people |
| **Solution Train Engineer (STE)** | Large Solution | Coordinates multiple ARTs |
| **Epic Owners, Lean Portfolio Management (LPM)** | Portfolio | Funding, governance, strategy |
| **Shared services / Customers** | — | Specialists and the end users |

## 2.9 Work item hierarchy in SAFe

```
Strategic Themes → Portfolio Epics → Capabilities (Large Solution) → FEATURES (fit in 1 PI) → Stories (fit in 1 iteration) → Tasks
```

| Item | Size | Level |
|---|---|---|
| Epic | Months-years | Portfolio |
| Capability | Spans ARTs | Large Solution |
| Feature | One PI, one ART | Program |
| Story | One iteration | Team |
| Enablers | Architecture/infrastructure/exploration/compliance work | All levels |

## 2.10 WSJF: prioritization in SAFe ⭐⭐

> **WSJF = Cost of Delay ÷ Job Duration (size)**

**Cost of Delay = User/Business Value + Time Criticality + Risk Reduction/Opportunity Enablement** (each scored relatively, e.g., 1,2,3,5,8,13,20).

Example:

| Feature | Business value | Time criticality | Risk reduction | CoD (sum) | Job size | **WSJF** |
|---|---|---|---|---|---|---|
| Payment upgrade | 13 | 8 | 5 | 26 | 8 | **3.25** |
| Wishlist | 8 | 3 | 2 | 13 | 3 | **4.33** ← do first |
| Report export | 5 | 2 | 1 | 8 | 13 | **0.62** |

Do the highest WSJF first: **small, valuable, urgent** items beat big, low-value ones.

## 2.11 Continuous Delivery Pipeline in SAFe (links to Parts 4-5) ⭐

SAFe's **Agile Product Delivery** competency includes the **Continuous Delivery Pipeline (CDP)**:

```
CONTINUOUS EXPLORATION → CONTINUOUS INTEGRATION → CONTINUOUS DEPLOYMENT → RELEASE ON DEMAND
(hypothesize, collaborate,   (develop, build, test      (deploy to staging, verify,   (release to customers
 research, architect, sync)    end-to-end, stage)         deploy to production)         when business wants)
```

DevOps in SAFe is the **CALMR** approach: **C**ulture, **A**utomation, **L**ean flow, **M**easurement, **R**ecovery. Teams use "shift-left" testing and "shift-right" monitoring.

## 2.12 Other SAFe events and ceremonies

| Event | Purpose |
|---|---|
| **Iteration Planning/Review/Retro** | Team-level Scrum events |
| **System Demo** | Integrated demo of all teams' work at end of each iteration |
| **ART Sync** (Scrum of Scrums + PO Sync) | Weekly coordination of dependencies and progress |
| **Inspect and Adapt (I&A)** | End-of-PI workshop: demo, quantitative measurement, retrospective, problem-solving (root cause) |
| **PI Planning** | Start-of-PI alignment |
| **Pre/Post-PI planning** | Prepare/consolidate |
| **Portfolio Sync, Strategic review** | Portfolio-level cadence |

## 2.13 What's new/changed in SAFe (verified notes) ✅

```
• SAFe 6.0 (March 2023): added flow accelerators, OKR guidance, AI/big data/cloud guidance, SAFe Studio platform.
• Scaled Agile later moved away from version numbers; courseware is now identified by release date (e.g., 2026.02.24).
• In June 2026 Scaled Agile unveiled "AI-Native SAFe", a new version of the operating model built for AI-era delivery,
  positioned to sit alongside the existing Core SAFe (it does not replace it).
```

Exam tip: say *"SAFe has evolved through versions (e.g., 6.0 in 2023) and continues to evolve, including AI-focused guidance."* Don't claim an exact current version number.

## 2.14 Benefits and criticisms of SAFe ⚖️

| Benefits | Criticisms |
|---|---|
| Familiar structure for large enterprises; clear roles | Seen as heavy, bureaucratic, "too prescriptive" |
| Aligns strategy, funding and teams | Risk of "agile in name only" (waterfall in disguise: big PI plans) |
| Cadence and synchronization reduce surprises | Many roles/ceremonies; consultant-driven |
| Strong portfolio and governance support | Can reduce team autonomy; management-heavy |
| Training/certification ecosystem | Costly adoption |

---

# 3️⃣ LeSS: LARGE-SCALE SCRUM ⭐⭐⭐

## 3.1 What is LeSS?

> **LeSS (Large-Scale Scrum)** is a scaling framework created by **Craig Larman and Bas Vodde** that applies **Scrum principles to many teams working on one product**, with **as little added process as possible**. Motto: **"More with LeSS."**

Philosophy: *don't add layers of roles and process; **de-scale** the organization*. Scrum is already a complete framework; scaling should be Scrum with more teams, not a new methodology on top.

## 3.2 Two LeSS configurations

| | **LeSS** | **LeSS Huge** |
|---|---|---|
| Teams | **2 to 8** teams (about up to ~50 people) | **More than 8** teams (hundreds) |
| Structure | One Product Owner, one backlog, all teams | Product divided into **Requirement Areas**, each with an **Area Product Owner (APO)** and 4-8 teams |
| Still one | Product Backlog, Definition of Done, Sprint, Potentially Shippable Product Increment | Same |

```
LESS (2-8 teams)                         LESS HUGE (8+ teams)
        Product Owner                          Product Owner (overall)
        One Product Backlog                    ├── Requirement Area 1 (APO) → 4-8 teams
   ┌──────┬──────┬──────┐                      ├── Requirement Area 2 (APO) → 4-8 teams
 Team A  Team B  Team C ...                    └── Requirement Area 3 (APO) → 4-8 teams
   one Sprint • one DoD • one shippable increment
```

## 3.3 Core LeSS structure rules ⭐

| Rule | Detail |
|---|---|
| **One Product Owner** (for LeSS) | Single person prioritizes one Product Backlog for the whole product (works at "product" level, not component level) |
| **One Product Backlog** | Teams pull items from the same ordered list: no per-team backlogs |
| **One Definition of Done** (common minimum) | Shared quality bar (teams may extend it) |
| **One Potentially Shippable Product Increment** | All teams integrate into one product every Sprint |
| **One Sprint** | All teams share the same Sprint cadence |
| **Feature teams** (not component teams) | Cross-functional teams that deliver end-to-end customer features across the whole codebase |
| **Teams are self-managing** | No team leads/project managers by default |
| **Managers become "Scrum-like" Product/Engineering leaders** | Managers' role: improve the system, not direct work |

### Feature team vs component team ⭐

| | **Feature team** | **Component team** |
|---|---|---|
| Owns | Customer-visible features end-to-end | One layer/component (UI team, DB team, API team) |
| Dependencies | Few | Many (hand-offs between teams) |
| Lead time | Short | Long |
| LeSS stance | **Preferred** | Avoided |

## 3.4 LeSS Sprint events ⭐⭐

| Event | LeSS version |
|---|---|
| **Sprint Planning One** | All teams' representatives + PO decide **which items each team will work on** (teams self-select); discuss opportunities for shared work |
| **Sprint Planning Two** | Each team plans **separately** (design, tasks); multi-team planning when needed |
| **Daily Scrum** | Each team has its own |
| **Coordination** | Communicate in code, cross-team meetings, **Communities of Practice**, travelers, open spaces; "Scrum of Scrums" only if needed |
| **Product Backlog Refinement** | Overall PBR (with PO, all teams) + team-level PBR; multi-team refinement to spread knowledge |
| **Sprint Review** | **One** review for the whole product, with stakeholders (often a "bazaar"-style review) |
| **Overall Retrospective** | Product-level retro (after each team's own retro) focused on cross-team systemic issues, with Scrum Masters, PO, selected members, and managers |

## 3.5 LeSS roles

| Role | Detail |
|---|---|
| Product Owner | One PO (LeSS), APOs (Huge) |
| Scrum Master | One SM per 1-3 teams (a teacher/coach; not a manager) |
| Teams | Cross-functional feature teams |
| Managers | Support the system (e.g., "Head of Product Group") |

## 3.6 LeSS rules and guides (vocabulary) 

LeSS has **Rules** (mandatory structure) and **Guides** (suggested experiments). Key ideas: **Systems thinking**, **Lean thinking**, **empirical process control**, **queueing theory**, **customer centricity**, **continuous improvement towards perfection**, **transparency**, **more responsibility to teams**, **whole-product focus**, **"Don't scale prematurely; de-scale first."**

## 3.7 Benefits and criticisms of LeSS ⚖️

| Benefits | Criticisms |
|---|---|
| Simple, close to Scrum; little extra process | Requires deep organizational change (restructure teams, remove roles) |
| Strong team autonomy and customer focus | Hard in organizations with entrenched hierarchy |
| Fewer roles/artifacts = less overhead | Less guidance for portfolio/governance than SAFe |
| Whole-product thinking, fewer hand-offs | Needs highly skilled generalist teams and mature technical practices (CI, TDD) |
| Encourages learning and experimentation | Fewer certifications/consultants than SAFe |

---

# 4️⃣ OTHER SCALING APPROACHES 🧩

| Approach | Creator / body | Idea | Scale |
|---|---|---|---|
| **Scrum of Scrums (SoS)** | Early Scrum practice | Representatives from each team meet regularly to coordinate | A few teams |
| **Nexus** | Scrum.org (Ken Schwaber) | 3-9 Scrum teams on one product; adds a **Nexus Integration Team** to manage dependencies and integration; one Product Backlog, one integrated increment; Nexus events (Nexus Sprint Planning, Daily Scrum, Review, Retro) | 3-9 teams |
| **Scrum@Scale (S@S)** | Jeff Sutherland (Scrum Inc.) | Two cycles: **Scrum Master cycle** (Scrum of Scrums, "how do we do it") and **Product Owner cycle** (MetaScrum, "what do we do"); fractal pattern, scales by replication | Any |
| **Spotify model** | Spotify (2012 whitepaper) | **Squads** (small autonomous teams), **Tribes** (groups of squads), **Chapters** (same-skill people across squads), **Guilds** (communities of interest). **A snapshot of one company's culture, not a framework**; Spotify itself evolved away from it | Culture/inspiration |
| **DAD (Disciplined Agile Delivery)** | PMI (Scott Ambler) | Toolkit of options for different contexts (part of PMI's Disciplined Agile) | Context-based |
| **Enterprise-scale tools** | Various | Agile@Scale, FAST, OKR cascades | — |

```
Spotify model picture:
 Tribe ── Squad 1  Squad 2  Squad 3   (each squad = mini-startup, owns a feature area)
          │  ▲ Chapter (e.g., all backend developers across squads share craft)
          └──── Guild (company-wide community, e.g., "Web performance guild")
```

---

# 5️⃣ COMPARISON: SAFe vs LeSS vs OTHERS ⭐⭐⭐ (WRITE THIS)

| Aspect | **SAFe** | **LeSS** | **Nexus** | **Scrum@Scale** |
|---|---|---|---|---|
| Philosophy | Add structure for coordination and governance | Minimal: Scrum scaled; de-scale organization | Minimal add-on to Scrum for integration | Fractal scaling of Scrum |
| Prescriptiveness | High (many roles, events, artifacts) | Low (few rules) | Low-medium | Low (flexible) |
| Team size | 50-125 per ART; 1000s overall | 2-8 teams (LeSS), 8+ (Huge) | 3-9 teams | Any |
| Cadence | Program Increment (8-12 weeks) with sprints inside | Single Sprint for all | Single Sprint | Sprint |
| Planning | **PI Planning** (2-day big-room) | **Sprint Planning 1 & 2** | Nexus Sprint Planning | Scrum of Scrums + MetaScrum |
| Backlogs | Portfolio → Solution → Program → Team | **One** Product Backlog | One | Product backlog(s) via POs |
| Roles | RTE, Product Mgmt, System Architect, Business Owners, STE... | PO, SM, feature teams (+APOs) | Nexus Integration Team | Chief PO, Chief SM |
| Team structure | Can have component and feature teams | **Feature teams** only | Scrum teams | Scrum teams |
| Governance/Portfolio | Strong (Lean Portfolio Mgmt) | Limited | Limited | Via executive cycles |
| Best for | Large traditional enterprises needing governance and clear roles | Organizations ready to simplify and empower teams | Moderate number of teams on one product | Scrum-experienced orgs |
| Risk | Bureaucracy; "waterfall in agile clothing" | Needs strong culture shift | Integration team can become bottleneck | Needs coaching |

## 5.1 How to choose (decision table)

| Situation | Better fit |
|---|---|
| Large enterprise with 100s-1000s people, regulation, portfolio funding, many stakeholders | **SAFe** |
| One product, 2-8 teams, org willing to flatten and use feature teams | **LeSS** |
| 3-9 teams, struggling with integration of a single product | **Nexus** |
| Strong Scrum culture, want flexible scaling | **Scrum@Scale** |
| Small startup/team | Plain **Scrum/Kanban**; don't scale yet |
| Want inspiration for culture | **Spotify model** (adapt, don't copy) |

Golden rule: **"Don't scale until you must; first make one team great."**

---

# 6️⃣ CONWAY'S LAW AND ORGANIZING TEAMS 🧠

> **Conway's Law:** "Organizations design systems that mirror their own communication structure."

```
Component teams (UI, API, DB)  →  layered architecture with hand-offs, slow delivery
Feature/stream-aligned teams   →  modular architecture, end-to-end ownership, faster flow
```

**Inverse Conway Maneuver:** design the team structure you want, to get the architecture you want (e.g., microservices owned by autonomous teams, like Unit I's Netflix/Amazon case studies: **"two-pizza teams"**).

**Team Topologies** (Skelton & Pais): four team types: **Stream-aligned**, **Enabling**, **Complicated-subsystem**, **Platform**; three interaction modes: collaboration, X-as-a-Service, facilitating. Platform teams offer self-service CI/CD and infrastructure (links to Unit II platform engineering).

---

# 7️⃣ SCALING + DEVOPS + CI/CD 🔗

Scaling agile only works if **integration is continuous**:

```
Many teams → one codebase/product
   ✔ Trunk-based development / short-lived branches
   ✔ Automated CI on every commit (build + tests + scans)
   ✔ Shared pipelines and templates (platform team)
   ✔ Feature flags to decouple deploy from release
   ✔ Contract tests between services
   ✔ Observability and fast rollback
SAFe: Continuous Delivery Pipeline + System Demo each iteration
LeSS: "potentially shippable product increment every Sprint" demands CI/CD maturity
```

Without CI/CD, scaling Agile produces **big-bang integration at the end of the PI**, i.e., waterfall again. (Parts 4-5 cover CI/CD and Jenkins.)

---

# 8️⃣ DRY RUN: A SAFe PI PLANNING IN ACTION 🏃

Context: Bank's Digital Payments ART, 8 teams (~80 people), PI = 10 weeks (4 sprints + IP).

```
Day 1 Morning
  Business Owner: "Competitors launched instant payments; we must ship UPI-style payments this PI."
  Product Mgmt: Top features by WSJF: (1) Instant Pay (WSJF 5.2) (2) Fraud alerts (4.1) (3) Statement export (0.9)
  System Architect: "New payments API requires platform team enabler: event bus upgrade first."
Day 1 Afternoon
  8 teams break out, draft iteration plans on boards (stories mapped to sprints 1-4)
  Team 3: "We need Team 5's fraud-score API by Sprint 2" → red string on Program Board = DEPENDENCY
  Draft review; management reviews; resolves capacity conflict by moving Statement export to next PI
Day 2
  Teams finalize plans + PI Objectives (rated by Business Owners 1-10)
  Risks ROAMed: "Third-party gateway delay" → Mitigated with sandbox + fallback vendor
  Confidence vote: average 4 of 5 → COMMIT
Execution
  Each sprint: team demos; end of sprint: System Demo (integrated)
  Weekly ART Sync tracks dependencies; CI/CD pipeline deploys to staging continuously
  IP iteration: hackathon, tech-debt cleanup, Inspect & Adapt, planning of next PI
```

---

# 9️⃣ EXAM ANSWER TEMPLATES ✍️

## 9.1 "Explain SAFe" (8-10 marks)

```
1. Definition: framework for enterprise-scale agility (Lean + Agile + DevOps) by Scaled Agile Inc.
2. Values/mindset: Alignment, Transparency, Respect for People, Courage; Lean-Agile principles
3. Structure diagram: Team → ART (Program) → Solution Train → Portfolio
4. Key concepts: ART (50-125 people), PI (8-12 weeks + IP iteration), PI Planning (2 days), Program Board
5. Roles: RTE, Product Management, System Architect, Business Owners, STE
6. Prioritization: WSJF = Cost of Delay ÷ Job size
7. Continuous Delivery Pipeline + DevOps (CALMR)
8. Benefits and criticisms; Conclusion
```

## 9.2 "Explain LeSS" (8-10 marks)

```
1. Definition: Large-Scale Scrum, Larman & Vodde; "More with LeSS"
2. Two configurations: LeSS (2-8 teams) and LeSS Huge (8+, requirement areas, APOs)
3. Structural rules: one PO, one Product Backlog, one DoD, one shippable increment, one Sprint, feature teams
4. Events: Sprint Planning One/Two, overall PBR, one Sprint Review, overall retrospective
5. Diagram of teams sharing one backlog
6. Benefits and criticisms; Conclusion
```

## 9.3 "Compare SAFe and LeSS" → use the table in Section 5 (10-12 rows) + when to use + conclusion.

---

# 🔟 VIVA QUESTIONS (PART 2) 🎯

**Q1. Why do we need scaling frameworks?** To coordinate many teams on one product: manage dependencies, integration, alignment and governance.

**Q2. What is SAFe?** Scaled Agile Framework: a set of roles, events and practices that apply Lean-Agile and DevOps principles to large enterprises.

**Q3. What is an Agile Release Train?** A long-lived team of agile teams (50-125 people) delivering value together on a fixed cadence.

**Q4. What is a Program Increment?** A time-box of 8-12 weeks (often 10) with several iterations and an Innovation and Planning iteration.

**Q5. What is PI Planning?** A two-day event where all ART teams plan the PI, create objectives, a Program Board, identify risks and give a confidence vote.

**Q6. What does RTE do?** Release Train Engineer facilitates ART events and removes impediments (chief Scrum Master).

**Q7. What is WSJF?** Weighted Shortest Job First: Cost of Delay divided by job size, used to prioritize.

**Q8. What is LeSS?** Large-Scale Scrum: applying Scrum with minimal extra roles/rules to multiple teams working on one product.

**Q9. LeSS vs LeSS Huge?** LeSS has 2-8 teams with one Product Owner; Huge has 8+ teams grouped into Requirement Areas with Area Product Owners.

**Q10. What are Sprint Planning One and Two in LeSS?** One decides which team takes which items (with PO and reps); Two is each team planning its own work.

**Q11. Feature team vs component team?** Feature teams deliver end-to-end customer features; component teams own one layer and create hand-offs.

**Q12. SAFe vs LeSS in one line?** SAFe adds structure and governance for large enterprises; LeSS keeps Scrum minimal and de-scales the organization.

**Q13. What is Nexus?** Scrum.org framework for 3-9 teams using a Nexus Integration Team for dependencies.

**Q14. What is Conway's Law?** System designs mirror the communication structures of the organizations that build them.

**Q15. Is the Spotify model a framework?** No, it is a description of one company's culture (squads, tribes, chapters, guilds) to be adapted, not copied.

---

# 🧠 PART 2 MASTER SHEET

```
SCALING PROBLEMS: dependencies, integration, alignment, governance, communication overhead N(N-1)/2
SAFe: values (Alignment, Transparency, Respect, Courage) ; 7 core competencies
      configs: Essential | Large Solution | Portfolio | Full
      ART 50-125 people ; PI 8-12 wks (4 sprints + IP) ; PI Planning 2 days (Program Board, PI Objectives, ROAM, confidence vote)
      roles: RTE, Product Mgmt, System Architect, Business Owners, STE ; WSJF = CoD ÷ size ; I&A ; System Demo ; CDP + CALMR
      SAFe 6.0 (2023) → no more version numbers → AI-Native SAFe (June 2026) alongside Core SAFe
LeSS: Larman & Vodde ; LeSS 2-8 teams, Huge 8+ (Requirement Areas, APO)
      one PO, one backlog, one DoD, one shippable increment, one Sprint ; feature teams ; Planning One/Two ; overall retro
Others: Nexus (3-9 teams, Integration Team) | Scrum@Scale (SM cycle + PO cycle) | Scrum of Scrums | Spotify (squad/tribe/chapter/guild) | DAD
Choose: big/regulated → SAFe ; flatten/empower → LeSS ; few teams, integration pain → Nexus ; small → plain Scrum
Conway's Law ; feature teams ; Team Topologies ; scaling needs CI/CD or it becomes waterfall again
```

---

## 🫡 PART 2 STATUS

You can now explain SAFe (ART, PI, PI Planning, WSJF) and LeSS (one backlog, feature teams, Planning One/Two), compare them with Nexus and Scrum@Scale, and choose between them.

**STOP here.** Next up:

> **PART 3 — TDD and BDD (Red-Green-Refactor, unit testing, mocking, Gherkin, Cucumber/behave, test pyramid) 🧪**
