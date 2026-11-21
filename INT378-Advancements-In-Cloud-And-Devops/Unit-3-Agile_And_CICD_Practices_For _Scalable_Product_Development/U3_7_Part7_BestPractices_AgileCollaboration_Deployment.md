# 🤝 UNIT III — PART 7: BEST PRACTICES FOR AGILE COLLABORATION AND DEPLOYMENT

> **Covered in this document:** the mindset (people + process + automation) → **Part A: agile collaboration best practices** (clear goals, Definition of Ready and Done, vertical slicing, working agreements, ceremonies done right, WIP limits and visual management, pair/mob programming and swarming, **code review** best practices with PR template and CODEOWNERS, commit hygiene, blameless culture and psychological safety, retrospectives with formats and action tracking, knowledge sharing and docs-as-code, stakeholder and customer feedback loops, conflict and decision-making, team health checks, metrics that help vs hurt incl. SPACE) → **Part B: deployment best practices** (small batches, trunk-based, build once, immutable artifacts, IaC and environment parity, test pyramid and quality gates, shift-left security, secrets and config, feature flags and progressive delivery, automated rollback, observability and SLOs, database migrations, release notes and versioning, lightweight change management, incident response and blameless postmortems, runbooks, deployment timing, cost and compliance) → ready-to-use checklists (PR, pre-deploy, go-live, rollback) → templates (PR template, CODEOWNERS, working agreement, postmortem) → maturity model → anti-patterns → exam template → viva → master sheet.

---

Absolutely mate. 😤🔥 This is the last theory block of Unit III. Your syllabus line: **"best practices for agile collaboration and deployment."** Examiners usually ask for **8-10 well-explained best practices with reasons** and often a checklist. This file gives you the full toolbox.

```
Great delivery = PEOPLE (collaboration) + PROCESS (agile flow) + AUTOMATION (CI/CD) + FEEDBACK (monitoring/metrics)
```

---

# 🅐 AGILE COLLABORATION BEST PRACTICES

## A1. Start with shared understanding

| Practice | Why it matters | How |
|---|---|---|
| **Clear Product Goal and Sprint Goal** | Everyone knows what success means | PO states the goal; the team pins it on the board |
| **Well-written user stories** | Prevents rework | INVEST + acceptance criteria (Given-When-Then, Part 3) |
| **Definition of Ready (DoR)** | Stories enter a sprint only when understood | Checklist below |
| **Definition of Done (DoD)** | One quality bar for all | Checklist below |
| **Three Amigos** (business + dev + test) before coding | Misunderstandings found early | 20-30 min per story or batch |
| **Vertical slicing** | Deliver value end-to-end each sprint | Slice by user flow, not by layer |

### Definition of Ready (example)

```
☐ Story has a clear user value ("As a... I want... so that...")
☐ Acceptance criteria written (Given-When-Then)
☐ Dependencies identified/unblocked; designs or API contracts available
☐ Estimated by the team; small enough for one Sprint
☐ Test data/environment needs known
```

### Definition of Done (example, Part 1 expanded)

```
☐ Code reviewed and merged via PR (≥ 1 approval)    ☐ Unit + acceptance tests written and passing in CI
☐ Static analysis/security scans pass quality gate   ☐ Deployed to staging automatically; smoke tests pass
☐ Docs/README/API spec/changelog updated             ☐ Monitoring/alerts/logging added for the feature
☐ Feature flag configured (if partial rollout)       ☐ Product Owner accepted
```

### Vertical vs horizontal slicing ⭐

```
HORIZONTAL (bad):  Sprint 1 = database tables, Sprint 2 = API, Sprint 3 = UI → nothing usable until sprint 3
VERTICAL (good):   Sprint 1 = "add item to wishlist" across DB+API+UI (tiny but working), Sprint 2 = "view wishlist"...
Slicing techniques: by workflow step, by business rule, by data variation, by happy path first then edge cases, by simple → complex
```

## A2. Make ceremonies valuable (not boring rituals)

| Ceremony | Best practice |
|---|---|
| **Sprint Planning** | Start from the Sprint Goal; pull only what the team can finish; plan to ~70-80% of capacity for surprises |
| **Daily Scrum** | 15 min, standing at the **board** (walk the work right-to-left: what can we finish today?), no status reports to managers; park deep discussions after |
| **Refinement** | Ongoing, ~10% of sprint capacity; keep 1-2 sprints of "ready" stories at the top |
| **Sprint Review** | Demo **working software**, not slides; invite real users/stakeholders; capture feedback as backlog items |
| **Retrospective** | Safe space, varied formats, **max 1-3 concrete actions with owners and due dates**, review last actions first |

## A3. Visual management and limiting work in progress

```
One shared board (Jira/Trello/GitHub Projects) = single source of truth for work
Columns mirror the real workflow; WIP limits per column; blocked items flagged in red
"Stop starting, start finishing": swarm on stuck work before pulling new items
Track cycle time and aging WIP (items open longest), not just velocity
```

## A4. Collaboration techniques: pairing, mobbing, swarming 👥

| Technique | What | Benefits | Watch out |
|---|---|---|---|
| **Pair programming** | Driver (types) + navigator (thinks, reviews); swap often | Instant review, knowledge sharing, fewer bugs | Needs rapport; schedule breaks |
| **Mob programming** | Whole team, one keyboard, one problem | Deep knowledge spreading, great for hard/critical code | Energy-intensive; use selectively |
| **Swarming** | Several members pile onto one story to finish it | Reduces WIP, speeds flow | Needs clear coordination |
| **Pair/mob remotely** | VS Code Live Share, Tuple, screen share | Works for distributed teams (Part 6) | Audio/latency quality |

## A5. CODE REVIEW BEST PRACTICES ⭐⭐⭐ (very common interview topic)

### Why review?

```
Catch bugs and design flaws early • share knowledge • enforce standards • build collective ownership • security checks
```

### Author's checklist (before opening a PR)

```
☐ Small, focused change (ideally < 300-400 lines; one purpose)    ☐ Self-review the diff first
☐ Tests added/updated and passing locally                          ☐ Meaningful title + description (what, why, how to test)
☐ No secrets, debug code, or commented-out code                    ☐ Linked to the ticket/issue; screenshots for UI
```

### Reviewer's checklist

```
☐ Correctness: does it do what the story says? edge cases? error handling?
☐ Design: simple, readable, consistent with architecture; no duplication
☐ Tests: meaningful, cover new behavior and edge cases
☐ Security: input validation, auth checks, no hard-coded secrets, dependency risks
☐ Performance/scalability concerns; logging/metrics added
☐ Naming, comments for "why", docs updated
☐ Style: leave nitpicks to linters/formatters (automate them!)
```

### Review etiquette ✅

```
Review within the agreed SLA (e.g., 1 working day) — slow reviews kill flow
Comment on the CODE, not the person; ask questions ("What happens if X is null?") instead of commands
Label comments: [blocking] vs [nit] vs [question] vs [praise]
Approve when good enough, not perfect; don't block on personal style
Move long debates to a call; summarize the outcome in the PR
Automate everything automatable: formatting, linting, tests, security scans (CI status checks)
```

### Pull request template (`.github/pull_request_template.md`)

```markdown
## What and why
<!-- Describe the change and link the ticket: Closes #123 -->

## How to test
<!-- Steps or commands; screenshots for UI changes -->

## Checklist
- [ ] Tests added/updated and passing
- [ ] No secrets or debug code
- [ ] Docs/changelog updated
- [ ] Backward compatible / migration considered
- [ ] Monitoring/alerts considered
- [ ] Feature flag used if partially complete
```

### CODEOWNERS (`.github/CODEOWNERS`): automatic reviewer assignment

```
# pattern            owners
*                    @myorg/dev-team
/infra/              @myorg/platform-team
/src/payments/       @asha-dev @myorg/payments-team
*.tf                 @myorg/platform-team
Jenkinsfile          @myorg/devops-team
```

### Branch protection (GitHub → Settings → Branches)

```
☑ Require a pull request before merging (≥ 1 approval; dismiss stale approvals)
☑ Require status checks to pass (Jenkins/GitHub Actions: build, test, lint, scan)
☑ Require branches to be up to date; require conversation resolution
☑ Restrict force pushes and deletions; optionally require signed commits and linear history
```

## A6. Commit and branch hygiene 🌿

```
Small, atomic commits (one logical change); commit often, push daily
Conventional Commit messages (Part 4):  feat: add wishlist endpoint   |   fix: round coupon to 2 decimals
Meaningful branch names: feature/wishlist-api, fix/coupon-rounding, chore/upgrade-flask
Short-lived branches; rebase or merge main frequently; squash noisy WIP commits when merging
Never commit secrets, large binaries, or generated files (.gitignore, pre-commit hooks, secret scanners)
Pre-commit hooks: formatter (black), linter (flake8), secret scan (gitleaks/detect-secrets)
```

## A7. Culture: psychological safety and blameless practice 🧠

> **Psychological safety** = people feel safe to speak up, ask questions, admit mistakes and disagree without fear of punishment. Google's Project Aristotle found it the strongest factor in effective teams.

Practices: **blameless retrospectives and postmortems** (focus on system causes, "how did our process allow this?"); leaders model vulnerability; celebrate learning from failure; reward helping others; avoid hero culture; sustainable pace (no constant overtime).

## A8. Retrospective best practices ⭐

| Format | Questions |
|---|---|
| **Start-Stop-Continue** | What should we start / stop / continue doing? |
| **4Ls** | Liked, Learned, Lacked, Longed for |
| **Sailboat** | Wind (helps), anchors (slows), rocks (risks), island (goal) |
| **Mad-Sad-Glad** | Emotions-based feedback |
| **Timeline** | Key events of the sprint with highs/lows |
| **5 Whys / fishbone** | Root-cause analysis for a problem |

Retro structure: **Set the stage → Gather data → Generate insights → Decide actions → Close**. Keep actions **SMART** (specific, owner, due date), limit to 1-3, put them in the next sprint's backlog, and **review them first** next retro. Rotate facilitators; vary formats; ensure everyone speaks (anonymous input for sensitive topics).

Action template:

```
Problem: Flaky test blocks pipeline 2-3 times a week
Root cause: shared test data and time-dependent assertion
Action: isolate fixtures + freeze time in tests   Owner: Meera   Due: next Sprint day 5   Success: 0 flaky failures in 2 weeks
```

## A9. Knowledge sharing and documentation 📚

```
Docs-as-code: README, runbooks, ADRs, API specs (OpenAPI) live in the repo, reviewed via PRs
ADR (Architecture Decision Record): short doc per decision (context, options, decision, consequences)
Onboarding guide + "make setup"/devcontainer so a new joiner runs the app in < 1 hour
Communities of practice/guilds, brown-bag sessions, tech talks, rotating on-call and pairing to spread knowledge
Avoid single points of knowledge ("bus factor" ≥ 2 for every component)
```

ADR template:

```
# ADR-012: Use PostgreSQL for orders
Status: Accepted (2026-03-10)
Context: Need transactions and relational integrity; team knows SQL.
Options: PostgreSQL | MongoDB | DynamoDB
Decision: PostgreSQL on RDS Multi-AZ.
Consequences: + ACID, mature tooling ; − vertical scaling limits; revisit if write volume > X.
```

## A10. Stakeholders, customers and feedback loops 🔁

```
Short feedback loops: demo every sprint; release behind flags to real users early
Direct customer contact: interviews, usability sessions, support-ticket review by the whole team
Transparent roadmap and progress (open board, release notes, burnup charts)
Manage expectations with outcomes/ranges, not fixed feature promises
Say "no" or "not now" with data (RICE/WSJF, Part 6)
```

## A11. Decision-making and conflict handling

```
Disagree and commit • prefer data/experiments over opinion • time-box debates • use ADRs for big decisions
Fist-of-five/dot voting for quick consensus • escalate blockers early, not late
```

## A12. Team health and metrics that help (not harm) 📊

| Use | Avoid |
|---|---|
| DORA metrics (team-level trends) | Individual productivity rankings |
| Cycle time, WIP age, throughput | Velocity as a target or cross-team comparison |
| Quality: escaped defects, MTTR, flaky-test rate | Lines of code, commit counts, hours online |
| Team health checks (Spotify-style: easy to release, fun, learning, pawns/players...) | Using surveys punitively |
| **SPACE framework**: **S**atisfaction, **P**erformance, **A**ctivity, **C**ommunication & collaboration, **E**fficiency & flow | Single-metric thinking |

Goodhart's law: *"When a measure becomes a target, it ceases to be a good measure."*

---

# 🅑 DEPLOYMENT BEST PRACTICES

## B1. The 12 golden rules ⭐⭐⭐

```
 1. Small batches, frequent releases      7. Feature flags + progressive delivery
 2. Trunk-based development               8. Automated rollback and quick recovery
 3. Automate everything repeatable        9. Observability: logs, metrics, traces, SLOs
 4. Build once, deploy many              10. Backward-compatible DB migrations
 5. Immutable, versioned artifacts       11. Blameless incident response + postmortems
 6. Infrastructure as Code, env parity   12. Shift-left security and compliance in the pipeline
```

## B2. Detail by practice

### 1) Small batch size and high frequency

```
Small change = easier to review, test, deploy, debug, roll back
Big release = big risk. "If it hurts, do it more often."
Aim for deploys of hours-days of work, not weeks (DORA: smaller batches correlate with better stability)
```

### 2) Trunk-based development + short-lived branches (Part 4)

```
Merge to main at least daily; protect main with PR + CI; hide unfinished work behind flags
```

### 3) Automate the whole path (CI/CD)

```
Commit → build → test → scan → package → deploy → verify → monitor with no manual copy/paste steps
Pipeline as code (Jenkinsfile in repo, reviewed like app code)
```

### 4) Build once, deploy many; immutable artifacts

```
Same Docker image (myapp:1.7.3-4f2a9c1) promoted dev → staging → prod; only config differs
Never patch servers by hand; replace with a new version (immutable infrastructure, Unit II)
Avoid the "latest" tag in production; tag with version + commit SHA
```

### 5) Infrastructure as Code and environment parity

```
Define VPCs, servers, pipelines in Terraform/CloudFormation (Unit II); same templates for all environments with different variables
Containers (Unit I) make runtime identical from laptop to production
Config in environment variables / parameter stores (12-factor, Unit VI), not baked into the image
```

### 6) Test strategy and quality gates

```
Test pyramid: many unit, some integration/contract, few E2E (Part 3) ; fast tests first
Gates: tests pass, coverage on new code, static analysis (SonarQube), dependency + image scans, performance budget
Smoke tests immediately after each deployment (GET /health, key user flow)
Fix or quarantine flaky tests immediately
```

### 7) Shift-left security (DevSecOps preview, Unit V) 🔐

```
SAST (code), SCA/dependency scanning, container image scanning (Trivy), IaC scanning (Checkov), secret scanning (gitleaks)
DAST on staging; least-privilege IAM for pipelines (OIDC, short-lived credentials)
Secrets only in secret stores (Jenkins credentials, Vault, AWS Secrets Manager) — never in repo or logs
Sign and verify artifacts; keep an SBOM; pin and update dependencies (Dependabot/Renovate)
```

### 8) Progressive delivery: flags, canary, blue-green

```
Deploy ≠ Release: deploy dark with the flag off; release gradually (1% → 10% → 50% → 100%) while watching metrics
Blue-green for instant switch/rollback; canary for risk reduction; A/B for product decisions (Part 6)
Automate promotion/abort based on SLO metrics (error rate, latency)
```

### 9) Fast, tested rollback and roll-forward

```
Keep previous artifacts; practice rollback in staging; automate (one command / one click)
Backward-compatible changes so old and new versions can coexist
Decide quickly: rollback for clear regressions, roll forward for small safe fixes
```

### 10) Observability and SLOs (Unit IV link) 📈

| Pillar | Meaning | Tools |
|---|---|---|
| **Metrics** | Numeric time series (CPU, request rate, errors, latency) | Prometheus, CloudWatch |
| **Logs** | Event records (structured JSON recommended) | ELK/OpenSearch, Loki, CloudWatch Logs |
| **Traces** | Request journey across services | Jaeger, OpenTelemetry, X-Ray |
| **Dashboards/alerts** | Visualize and notify | Grafana, Alertmanager |

```
Golden signals: Latency, Traffic, Errors, Saturation (Google SRE)
SLI (indicator) → SLO (objective, e.g., 99.9% of requests succeed) → SLA (contract) ; Error budget = 100% − SLO
Alert on symptoms users feel (SLO burn), not every CPU blip; every alert actionable with a runbook
Deployment markers on dashboards (see "what changed" instantly)
```

### 11) Database migrations done safely

```
Versioned migration scripts (Flyway, Liquibase, Alembic) in the repo, run by the pipeline
Expand → migrate → contract (Part 4); avoid destructive changes in the same release as code that needs them
Backups before risky migrations; test migrations on production-like data; plan reversibility
```

### 12) Versioning, release notes and traceability

```
Semantic Versioning + Git tags; automatic changelog from Conventional Commits (semantic-release, release-please)
Every deployed version traceable: commit SHA ↔ build number ↔ ticket ↔ artifact
Release notes for users; internal deployment log in Slack/Teams channel (ChatOps)
```

### 13) Lightweight change management

```
Replace heavy Change Advisory Boards (CAB) with: automated checks + peer review + small changes + audit trail from the pipeline
(Research shows heavyweight external approvals correlate with slower delivery without better stability)
Keep approvals where regulation demands (Continuous Delivery with an approval gate)
```

### 14) Deployment timing and communication

```
Deploy during business hours when the team is available (avoid "Friday evening" deploys unless your automation and rollback are excellent)
Announce in a deployment channel; have an owner "on watch" for 30-60 minutes after release
Freeze periods for peak business events (festival sales) if needed; keep pipelines ready for emergency hotfixes
```

### 15) Incident response and blameless postmortems 🚨

```
Detect (alerts) → Triage (severity) → Mitigate (rollback/flag off) → Communicate (status page) → Resolve → Learn
On-call rotation, clear roles (incident commander, communicator), runbooks
Postmortem within days: timeline, impact, root causes (5 Whys), what went well/badly, action items with owners
```

Postmortem template:

```
Title / date / severity / duration / authors
Summary & customer impact
Timeline (UTC): detection → escalation → mitigation → resolution
Root cause(s) & contributing factors (system/process, not individuals)
What went well • What went wrong • Where we got lucky
Action items (owner, due date, ticket) — prevent, detect, mitigate faster
```

### 16) Cost, performance and compliance

```
Tag resources; budgets and alerts (Unit VI); right-size; autoscale; remove idle environments (ephemeral preview environments)
Keep audit trails: who deployed what, when, with which approvals (pipeline logs, Git history, CloudTrail)
Compliance as code (policy checks in pipeline); retention of logs/artifacts per regulation
```

---

# ✅ READY-TO-USE CHECKLISTS

## C1. Pre-merge (PR) checklist

```
☐ Small, focused PR with description and linked ticket     ☐ CI green: build, unit tests, lint, security scans
☐ At least 1 approving review; comments resolved           ☐ New/changed behavior tested; docs updated
☐ No secrets/debug code; migrations backward compatible    ☐ Feature flag ready if incomplete
```

## C2. Pre-deployment checklist

```
☐ Artifact built once, versioned, scanned, stored in registry   ☐ Staging deployment + acceptance/smoke tests passed
☐ DB migration plan reviewed (expand/contract) and backed up    ☐ Monitoring/alerts/dashboards ready for the change
☐ Rollback plan known and tested (previous version available)   ☐ Feature flags configured; canary % planned
☐ On-call/owner available; stakeholders informed                ☐ Runbook updated; release notes drafted
```

## C3. Post-deployment checklist

```
☐ Health check/smoke test OK       ☐ Error rate, latency, saturation normal vs baseline
☐ Business metric sanity check     ☐ No spike in logs/alerts       ☐ Canary ramped or rolled back as decided
☐ Release notes published          ☐ Ticket/changelog updated      ☐ Remove obsolete flags/old versions after confidence
```

## C4. Rollback checklist

```
☐ Declare rollback (owner)         ☐ Switch traffic back (blue-green) / redeploy previous artifact / turn flag off
☐ Verify health and metrics        ☐ Communicate status                   ☐ Preserve logs/evidence for the postmortem
☐ Create follow-up tickets         ☐ Fix forward in a new, tested change
```

---

# 🧱 TEAM WORKING AGREEMENT TEMPLATE

```
TEAM CHARTER: Payments Squad
Core hours: 13:00-17:00 IST overlap | Response time: Slack 4 h, PR review 1 working day
Meetings: Daily 10-min stand-up (async if absent), Planning Mon, Review+Retro every 2nd Friday
Quality: DoD in repo/docs; no merge without CI green + 1 approval; boy-scout refactors welcome
Communication: threads over DMs; decisions in ADRs; blockers raised within 2 hours
Pairing: 2 pairing sessions/week; knowledge-sharing demo monthly
Incidents: blameless; on-call rotation weekly; postmortem within 3 working days
```

---

# 🪜 DEVOPS/AGILE MATURITY LADDER

| Level | Description |
|---|---|
| **1. Ad hoc** | Manual builds/deploys, long-lived branches, rare releases |
| **2. Repeatable** | CI server builds and runs unit tests; some scripts for deployment |
| **3. Defined** | Pipeline as code; automated tests; staging deploy; PR reviews; Scrum/Kanban stable |
| **4. Managed** | Continuous Delivery, IaC, quality gates, security scanning, monitoring, DORA tracking, feature flags |
| **5. Optimizing** | Continuous Deployment/progressive delivery, SLO-driven automation, experimentation, platform engineering, continuous improvement culture |

Use it to say where a team is and the next step (don't jump from level 1 to 5).

---

# 🚨 ANTI-PATTERNS (WHAT NOT TO DO)

| Anti-pattern | Fix |
|---|---|
| Giant PRs reviewed in minutes ("LGTM") | Small PRs, checklists, review SLAs |
| Heroics: one person deploys manually | Automate; everyone can deploy |
| Long-lived branches/big-bang merges | Trunk-based + flags |
| "Works in staging" but different from prod | IaC + containers, env parity |
| Deploying without monitoring | Dashboards, alerts, post-deploy checks |
| Secrets in code/CI logs | Secret managers, masking, scanning |
| No rollback plan | Versioned artifacts, blue-green/canary, rehearsed rollback |
| Blame after incidents | Blameless postmortems |
| Velocity worship / individual metrics | Team outcome metrics (DORA, cycle time) |
| Skipping retrospective actions | Track actions, review first thing next retro |
| Meetings overload / no focus time | Async-first, meeting-free blocks |
| Documentation only in people's heads | Docs-as-code, ADRs, runbooks |
| Manual approvals for everything | Automate low-risk checks; humans for high-risk |
| Ignoring flaky tests | Fix/quarantine, stop-the-line |

---

# ✍️ EXAM ANSWER TEMPLATE

**Q. Discuss best practices for agile collaboration and deployment.**

```
1. Intro: delivering software fast and reliably needs both team collaboration practices and deployment engineering practices.
2. COLLABORATION (6-8 bullets, 1-2 lines each): clear goals + DoR/DoD; vertical slicing; effective ceremonies; WIP limits/visual boards;
   code reviews + PR hygiene; pair/mob programming; blameless culture/retrospectives; documentation/ADRs; customer feedback; healthy metrics
3. DEPLOYMENT (8-10 bullets): small batches; trunk-based; CI/CD automation; build once deploy many; IaC/environment parity; test pyramid + quality gates;
   shift-left security & secrets; feature flags + canary/blue-green; fast rollback; observability/SLOs; safe DB migrations; blameless incident response
4. Small checklist or table (PR checklist / pre-deploy checklist)
5. Metrics: DORA (+ SPACE)
6. Conclusion: People + process + automation + feedback → faster, safer, higher-quality delivery.
```

---

# 🎯 VIVA QUESTIONS (PART 7)

**Q1. What is the Definition of Done vs Definition of Ready?** DoD is the quality checklist for completed work; DoR is the checklist for a story to be ready to enter a sprint.

**Q2. What is vertical slicing?** Splitting work into thin end-to-end slices delivering user value, instead of layer-by-layer.

**Q3. Give 4 code review best practices.** Small PRs, automated checks first, constructive comments on code not people, timely reviews (SLA).

**Q4. What is CODEOWNERS?** A file that automatically requests reviews from designated owners for matching paths.

**Q5. What is pair vs mob programming?** Two people at one keyboard vs the whole team working on one problem together.

**Q6. What is psychological safety?** A team climate where people can speak up and admit mistakes without fear.

**Q7. What makes a good retrospective action?** Specific, owned, time-bound, limited in number, and reviewed next retro.

**Q8. Why deploy small and often?** Lower risk, easier debugging and rollback, faster feedback.

**Q9. What is "build once, deploy many"?** Create one immutable artifact and promote it through environments with environment-specific configuration only.

**Q10. How do feature flags help deployment?** They decouple deployment from release, allow gradual rollout and instant kill switches.

**Q11. How do you ensure safe database migrations?** Versioned, automated, backward-compatible (expand/contract), tested, with backups.

**Q12. What are golden signals?** Latency, traffic, errors and saturation.

**Q13. SLI, SLO, SLA and error budget?** SLI is the measured indicator; SLO the internal target; SLA the customer contract; error budget is the allowed unreliability (100% − SLO).

**Q14. What is a blameless postmortem?** An incident review focused on system and process causes and learning, not on blaming individuals.

**Q15. Name useful vs harmful team metrics.** Useful: DORA, cycle time, escaped defects; harmful: individual output rankings, velocity targets, lines of code.

**Q16. How should secrets be handled in pipelines?** Stored in secret managers/credential stores, injected at runtime, masked, never committed.

**Q17. Why avoid Friday evening deployments?** Reduced staff availability for incident response; only deploy then with excellent automation and rollback.

---

# 🧠 PART 7 MASTER SHEET

```
COLLABORATION: clear goals • DoR/DoD • vertical slicing • Three Amigos • effective ceremonies • WIP limits + visual board
  • code review (small PRs, SLA, labels, automation, PR template, CODEOWNERS, branch protection) • pair/mob/swarm
  • commit hygiene (conventional commits, hooks) • psychological safety + blameless • retros (SMART actions)
  • docs-as-code, ADRs, bus factor ≥ 2 • customer feedback loops • healthy metrics (DORA, SPACE), avoid Goodhart traps
DEPLOYMENT: small batches • trunk-based • automate everything • build once deploy many • immutable artifacts • IaC + env parity
  • test pyramid + quality gates • shift-left security + secrets • feature flags + canary/blue-green • fast rollback
  • observability (metrics, logs, traces; golden signals; SLI/SLO/SLA/error budget) • safe DB migrations (expand/contract)
  • versioning + release notes • lightweight change mgmt • incident response + blameless postmortems • cost/compliance
CHECKLISTS: PR • pre-deploy • post-deploy • rollback ; MATURITY: ad hoc → repeatable → defined → managed → optimizing
```

---

## 🫡 PART 7 STATUS

You can now articulate and apply best practices for agile collaboration (reviews, DoD/DoR, retros, blameless culture) and for deployment (automation, progressive delivery, observability, safe rollbacks), with checklists and templates.

**STOP here.** Next up:

> **PART 8 — Practical Lab Guide: Jenkins + GitHub webhook + automated CI/CD pipeline (build, test, auto-deploy), step by step 🧪**
