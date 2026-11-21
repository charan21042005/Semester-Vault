# 🏆 UNIT III — GRAND CONSOLIDATION

## Agile and CI/CD Practices for Scalable Product Development — From Scratch → End-to-End

> **Covered in this document:** the whole unit in one sentence → the complete story → part-by-part revision (Parts 1-8) → master comparison tables (Waterfall vs Agile, Scrum vs Kanban, SAFe vs LeSS vs Nexus vs Scrum@Scale, TDD vs BDD vs ATDD, CI vs Delivery vs Deployment, branching strategies, deployment strategies, Freestyle vs Pipeline, Declarative vs Scripted, Jenkins vs GitHub Actions vs GitLab CI, A/B vs canary, prioritization frameworks) → command and syntax cheat sheets (Git, pytest, behave, Docker, Jenkins, Jenkinsfile) → decision tables → common errors table → CI/CD security checklist → lab ideas → 3 real-world scenarios → 30 best viva questions with answers → common traps → what to write (and not write) in an exam → master sheet → memory maps → how Unit III connects to Units I, II, IV, V and VI.

---

# 🚀 THE ENTIRE UNIT III IN ONE SENTENCE

> **Agile teams (scaled with SAFe or LeSS when many teams work on one product) build the right thing in small increments guided by product strategy, A/B tests and managed technical debt; they build it right with TDD and BDD; and a CI/CD pipeline (such as a Jenkins `Jenkinsfile` triggered by a GitHub webhook) automatically builds, tests, deploys and monitors every change using best practices for collaboration, progressive delivery and fast recovery.**

---

# 1️⃣ THE COMPLETE UNIT III STORY

```
WHY AGILE        Waterfall: late feedback, big-bang risk → Agile: small increments, change welcomed        [Part 1]
HOW (1 TEAM)     Scrum (roles, events, artifacts, DoD) + Kanban (WIP) + XP engineering                      [Part 1]
HOW (MANY TEAMS) SAFe (ART, PI, PI Planning, WSJF) vs LeSS (one backlog, feature teams, Sprint Planning 1/2) [Part 2]
BUILD IT RIGHT   TDD (Red-Green-Refactor) + BDD (Given-When-Then, Three Amigos) + test pyramid             [Part 3]
SHIP IT          CI → Continuous Delivery → Continuous Deployment ; pipelines ; branching ; blue-green/canary/flags [Part 4]
THE TOOL         Jenkins: controller+agents, Jenkinsfile, credentials, webhooks, multibranch, security      [Part 5]
BUILD THE RIGHT THING + KEEP IT HEALTHY  Product strategy, distributed teams, A/B testing, technical debt  [Part 6]
DO IT WELL       Collaboration best practices (reviews, DoD, retros) + deployment best practices (SLOs, rollback) [Part 7]
PROVE IT         Practical 6: GitHub push → webhook → Jenkins lint/test/build/deploy/smoke test/rollback     [Part 8]
```

---

# 2️⃣ PART-BY-PART REVISION

## Part 1: Agile from scratch
```
Waterfall (sequential, late feedback) vs Agile (iterative, adaptive) ; Manifesto: 4 values, 12 principles
SCRUM: PO, Scrum Master, Developers | Sprint, Planning, Daily Scrum, Review, Retrospective | Product Backlog (Product Goal),
       Sprint Backlog (Sprint Goal), Increment (Definition of Done) ; pillars: transparency, inspection, adaptation
Stories: As a/I want/so that + INVEST + acceptance criteria ; epic > story > task ; story points, planning poker
KANBAN: visualize, WIP limits, flow ; cycle time = WIP ÷ throughput (Little's Law)
XP: TDD, pair programming, CI, refactoring, simple design, small releases
Metrics: velocity (planning only), burndown/burnup, cycle/lead time, throughput, CFD ; Agile + DevOps = continuous value
```

## Part 2: Scaling Agile
```
Problems: dependencies, integration, alignment, governance, communication paths N(N-1)/2
SAFe: values (Alignment, Transparency, Respect for People, Courage) ; 7 core competencies ; configs: Essential, Large Solution, Portfolio, Full
      ART (50-125 people) ; PI (8-12 weeks, 4 sprints + IP iteration) ; PI Planning (2 days: objectives, Program Board, ROAM, confidence vote)
      roles: RTE, Product Mgmt, System Architect, Business Owners, STE ; WSJF = Cost of Delay ÷ job size ; I&A ; System Demo ; CDP + CALMR
      SAFe 6.0 (Mar 2023) → no version numbers → AI-Native SAFe (Jun 2026) alongside Core SAFe
LeSS (Larman & Vodde): LeSS (2-8 teams) / LeSS Huge (8+, Requirement Areas, APOs) ; one PO, one backlog, one DoD, one shippable increment, one Sprint ;
      feature teams ; Sprint Planning One/Two ; overall retrospective ; "More with LeSS"
Others: Nexus (3-9 teams, Integration Team) | Scrum@Scale | Scrum of Scrums | Spotify model (squad/tribe/chapter/guild) | DAD ; Conway's Law
```

## Part 3: TDD and BDD
```
Test pyramid: many unit, some integration, few UI ; FIRST ; Arrange-Act-Assert
TDD: Red → Green → Refactor ; Three Laws ; pytest (fixture, parametrize, raises) ; mocks/stubs/fakes ; coverage is a guide not a goal
BDD: Dan North ; Three Amigos ; Given-When-Then ; Gherkin (.feature) ; Cucumber/behave/SpecFlow ; living documentation ; ATDD
TDD = build it right (code) | BDD = build the right thing (behavior) | double loop: BDD outer + TDD inner ; tests run in CI and publish JUnit reports
```

## Part 4: CI/CD concepts
```
CI: merge often + automated build + tests + fix broken build now ; Continuous Delivery (manual prod approval) vs Continuous Deployment (automatic)
Pipeline: Source → Build → Test → Package → Deploy staging → Verify → [approve] → Prod → Monitor ; build once deploy many ; immutable versioned artifacts
Pipeline as code ; branching: GitFlow | GitHub Flow | Trunk-based (+flags) ; SemVer ; Conventional Commits ; PR + status checks + branch protection
Deploy strategies: recreate, rolling, blue-green, canary, shadow, A/B, feature flags ; expand/contract DB migrations ; rollback vs roll-forward
Tools: Jenkins, GitHub Actions, GitLab CI, AWS CodePipeline, Argo CD, Spinnaker ; DORA: deployment frequency, lead time, change failure rate, time to restore (+ rework)
```

## Part 5: Jenkins
```
Controller (UI, config, scheduling, plugins) + Agents (executors, labels, workspace) ; JENKINS_HOME ; plugins ; LTS releases
Install: Java 17/21 first → Jenkins repo (current key) → :8080 → initialAdminPassword → suggested plugins
Jobs: Freestyle | Pipeline | Multibranch | Folder ; Declarative pipeline{agent, environment, options, parameters, triggers, stages, post, when, parallel, input}
Credentials store + withCredentials ; triggers: webhook (/github-webhook/), pollSCM, cron ; Docker agents ; shared libraries ; JCasC ; security hardening
```

## Part 6: Strategy, distributed teams, A/B testing, technical debt
```
STRATEGY: vision → strategy → roadmap → backlog ; outcome > output ; MVP, Build-Measure-Learn, pivot ; OKRs, North Star, AARRR ; RICE, ICE, MoSCoW, Kano, WSJF
DISTRIBUTED: async-first, overlap hours, working agreements, adapted ceremonies, CI/CD + docs + IaC, security, wellbeing
A/B: random split → metric → stats (α 0.05, p-value, power 80%, CI) → example 5.0% vs 5.8%: z≈2.50, p≈0.012 ; pitfalls: peeking, SRM, multiple tests ; vs canary
TECH DEBT: Cunningham metaphor (principal + interest) ; Fowler quadrant ; measure (Sonar debt ratio, complexity, churn) ; manage (visible register, Boy Scout rule,
           10-20% capacity, refactor, strangler fig, quality gates, prioritize pain×frequency×risk÷effort)
```

## Part 7: Best practices
```
Collaboration: clear goals, DoR/DoD, vertical slicing, effective ceremonies, WIP limits, code review (small PRs, SLA, templates, CODEOWNERS, branch protection),
  pair/mob, commit hygiene, blameless culture, SMART retro actions, docs-as-code/ADRs, feedback loops, metrics (DORA, SPACE; avoid Goodhart traps)
Deployment: small batches, trunk-based, automate, build once, immutable, IaC + parity, test pyramid + gates, shift-left security + secrets, flags + canary/blue-green,
  rollback, observability (golden signals, SLI/SLO/SLA, error budget), safe migrations, versioning, light change mgmt, blameless incident response
```

## Part 8: Practical 6
```
Repo (app, tests, Dockerfile, Jenkinsfile) → EC2 Ubuntu + SG (22, 8080, 80) → Java 21 + Jenkins + Git + Python + Docker → jenkins in docker group + restart
→ Pipeline job (SCM + hook trigger) → manual build → GitHub webhook (…/github-webhook/) → git push = auto build/test/deploy
→ failing test = no deploy ; failed smoke test = auto rollback → evidence → cleanup
```

---

# 3️⃣ MASTER COMPARISON TABLES

## 3.1 Waterfall vs Agile

| | Waterfall | Agile |
|---|---|---|
| Approach | Sequential phases | Iterative/incremental |
| Requirements | Fixed early | Evolving |
| Delivery | One big release | Frequent small releases |
| Feedback | Late | Continuous |
| Change | Costly | Welcomed |
| Risk | Found late | Found early |
| Best for | Stable, regulated, well-known scope | Uncertain, fast-changing products |

## 3.2 Scrum vs Kanban

| | Scrum | Kanban |
|---|---|---|
| Cadence | Fixed Sprints | Continuous flow |
| Roles | PO, SM, Developers | None prescribed |
| Limits | Sprint scope | WIP limits |
| Change | Not mid-Sprint | Anytime (WIP allowing) |
| Metrics | Velocity, burndown | Cycle time, throughput, CFD |

## 3.3 Scaling frameworks

| | SAFe | LeSS | Nexus | Scrum@Scale |
|---|---|---|---|---|
| Philosophy | Add structure/governance | Minimal: de-scale | Add integration team | Fractal Scrum |
| Size | 50-125 per ART; thousands overall | 2-8 teams (Huge: 8+) | 3-9 teams | Any |
| Planning | PI Planning (2 days) | Sprint Planning One/Two | Nexus Sprint Planning | Scrum of Scrums + MetaScrum |
| Backlog | Portfolio→Solution→Program→Team | One Product Backlog | One | Via POs |
| Teams | Feature or component | Feature teams only | Scrum teams | Scrum teams |
| Best for | Large regulated enterprises | Orgs willing to simplify | Few teams, integration pain | Scrum-mature orgs |

## 3.4 TDD vs BDD vs ATDD

| | TDD | BDD | ATDD |
|---|---|---|---|
| Focus | Code correctness | System behavior | Acceptance criteria |
| Audience | Developers | Whole team | Team + customer |
| Language | Code asserts | Gherkin Given-When-Then | Acceptance tests |
| Level | Unit | Acceptance/feature | Acceptance |
| Tools | pytest, JUnit | Cucumber, behave | FitNesse, Robot |

## 3.5 CI vs Continuous Delivery vs Continuous Deployment

| | CI | Continuous Delivery | Continuous Deployment |
|---|---|---|---|
| Scope | Merge + build + test | + always release-ready | + automatic release |
| Production release | No | **Manual approval** | **Automatic** |
| Needs | Tests | Staging, automation | Excellent tests, monitoring, rollback |

## 3.6 Branching strategies

| | GitFlow | GitHub Flow | Trunk-based |
|---|---|---|---|
| Long-lived branches | develop, release | main only | trunk only |
| Branch life | Days-weeks | Hours-days | ≤ 1-2 days |
| Release | Scheduled | Continuous | Continuous |
| CI fit | Lower | Good | **Best** (with flags) |

## 3.7 Deployment strategies

| Strategy | Downtime | Rollback | Cost | Note |
|---|---|---|---|---|
| Recreate | Yes | Redeploy | Low | Simplest |
| Rolling | No | Slow | Low | Two versions coexist |
| Blue-Green | No | **Instant** | 2× infra | Switch traffic |
| Canary | No | Fast | Medium | % of users + metrics |
| Shadow | No | n/a | Higher | Mirror traffic |
| A/B | No | Flag off | Medium | Compare business metric |
| Feature flag | No | Instant | Low | Deploy ≠ release |

## 3.8 Jenkins job and syntax comparisons

| Freestyle vs Pipeline | Freestyle | Pipeline |
|---|---|---|
| Defined in | UI | Jenkinsfile (code) |
| Versioned | No | Yes |
| Complex flows | Limited | Rich |

| Declarative vs Scripted | Declarative | Scripted |
|---|---|---|
| Syntax | `pipeline { }` structured | `node { }` Groovy |
| Learning | Easier | Harder |
| Validation | Up front | Runtime |

## 3.9 Jenkins vs GitHub Actions vs GitLab CI

| | Jenkins | GitHub Actions | GitLab CI |
|---|---|---|---|
| Hosting | You run it | Hosted (or self-hosted runners) | GitLab SaaS/self-managed |
| Config | Jenkinsfile/UI | YAML workflows | `.gitlab-ci.yml` |
| Setup effort | High | Very low | Low |
| Flexibility | Very high (plugins) | High (marketplace) | High (built-in) |
| Maintenance | You (upgrades, security) | Vendor | Vendor/you |

## 3.10 A/B testing vs canary vs feature flag

| | A/B test | Canary | Feature flag |
|---|---|---|---|
| Question | Is it **better**? | Is it **safe**? | Mechanism to control exposure |
| Measures | Business metric (stats) | Errors/latency | — |
| Duration | Until planned sample reached | Minutes-hours | Any |

## 3.11 Prioritization frameworks

| Framework | Formula/idea | Use |
|---|---|---|
| **RICE** | (Reach × Impact × Confidence) ÷ Effort | Product features |
| **ICE** | Impact × Confidence × Ease | Quick experiments |
| **WSJF** | Cost of Delay ÷ Job size | SAFe |
| **MoSCoW** | Must/Should/Could/Won't | Fixed-scope releases |
| **Kano** | Basic/Performance/Delighters | Satisfaction analysis |
| **Value vs Effort** | 2×2 matrix | Visual triage |

## 3.12 Technical debt quadrant

| | Reckless | Prudent |
|---|---|---|
| **Deliberate** | "No time for design" | "Ship now, fix later (planned)" |
| **Inadvertent** | "What's layering?" | "Now we know a better way" |

---

# 4️⃣ COMMAND AND SYNTAX CHEAT SHEETS ⌨️

## 4.1 Git

```bash
git clone URL ; git checkout -b feature/x ; git add . ; git commit -m "feat: ..." ; git push -u origin feature/x
git pull --rebase ; git merge main ; git rebase main ; git log --oneline --graph ; git status ; git diff
git tag -a v1.2.0 -m "release" && git push --tags ; git revert <sha> ; git stash
```

## 4.2 pytest and behave

```bash
pytest -v ; pytest -k "coupon" ; pytest --junitxml=reports/pytest.xml ; pytest --cov=app --cov-report=term-missing --cov-fail-under=70
behave ; behave --tags=@smoke ; behave --junit --junit-directory reports/behave
flake8 app.py --max-line-length=100
```

## 4.3 Docker (from Unit I)

```bash
docker build -t name:tag . ; docker run -d --name n -p 80:5000 name:tag ; docker ps ; docker logs n ; docker rm -f n ; docker image prune -f
```

## 4.4 Jenkins on Linux

```bash
sudo systemctl status|start|stop|restart jenkins ; sudo journalctl -u jenkins -f
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
sudo usermod -aG docker jenkins && sudo systemctl restart jenkins
```

## 4.5 Declarative Jenkinsfile skeleton

```groovy
pipeline {
  agent any
  options { timestamps(); timeout(time: 20, unit: 'MINUTES'); disableConcurrentBuilds() }
  environment { TAG = "${env.BUILD_NUMBER}" }
  stages {
    stage('Checkout') { steps { checkout scm } }
    stage('Build')    { steps { sh 'make build' } }
    stage('Test')     { steps { sh 'make test' } post { always { junit 'reports/*.xml' } } }
    stage('Approve')  { when { branch 'main' } steps { input message: 'Deploy?' } }     // Continuous Delivery gate
    stage('Deploy')   { when { branch 'main' } steps { sh './deploy.sh' } }
  }
  post { success { echo 'OK' } failure { echo 'FAILED' } always { archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true } }
}
```

## 4.6 Webhook essentials

```
Payload URL:  http://<JENKINS_HOST>:8080/github-webhook/    (trailing slash)
Content type: application/json    Event: push    Job: "GitHub hook trigger for GITScm polling"
```

---

# 5️⃣ DECISION TABLES 🎯

## 5.1 Which agile approach?

| Situation | Choose |
|---|---|
| One product team building features in planned increments | Scrum |
| Support/ops/maintenance with unpredictable flow | Kanban |
| Strong engineering rigor needed | Add XP practices (TDD, pairing, CI) |
| Large regulated enterprise, many teams, portfolio governance | SAFe |
| 2-8 teams, one product, willing to flatten roles | LeSS |
| 3-9 teams, integration pain | Nexus |

## 5.2 Which test practice?

| Need | Choose |
|---|---|
| Design better code and get fast regression safety | TDD |
| Align business and tech on behavior | BDD |
| Verify feature meets acceptance criteria with customer | ATDD/BDD |
| Verify services' API compatibility | Contract tests |
| Check deployed app is alive | Smoke test |

## 5.3 Which deployment strategy?

| Need | Choose |
|---|---|
| Simple app, downtime acceptable | Recreate |
| No downtime, low cost | Rolling |
| Instant rollback | Blue-green |
| Reduce blast radius with real users | Canary |
| Business decision between versions | A/B test |
| Hide unfinished work / kill switch | Feature flag |

## 5.4 Which CI/CD tool?

| Situation | Choose |
|---|---|
| Code on GitHub, minimal ops | GitHub Actions |
| Need full control, on-prem/air-gapped, lots of plugins | Jenkins |
| Using GitLab end to end | GitLab CI |
| AWS-native shop | CodePipeline/CodeBuild |
| Kubernetes GitOps | Argo CD/Flux |

## 5.5 Which debt first?

```
Priority = (Pain × Change frequency × Risk) ÷ Effort → fix hot, risky, frequently changed code first; accept or delete stable, low-risk debt
```

---

# 6️⃣ COMMON ERRORS TABLE 🛠️

| Symptom | Cause | Fix |
|---|---|---|
| Jenkins won't start / Java error | Java missing or installed after Jenkins | Install Java 17/21 first; restart |
| Jenkins apt GPG error | Rotated signing key | Use current key from jenkins.io |
| `source: not found` in `sh` | dash shell | Use `. venv/bin/activate` |
| `docker: permission denied` in pipeline | `jenkins` not in docker group / no restart | `usermod -aG docker jenkins` + restart |
| Webhook delivered but no build | Trigger not ticked / URL mismatch / branch spec | Tick trigger; match repo URL; `*/main` |
| Webhook timeout | SG/port/IP changed | Open 8080 for GitHub ranges; update URL |
| `Couldn't find any revision to build` | Branch `*/master` vs `*/main` | Fix branch spec |
| Deploy ran despite failing tests | Tests not failing the stage / `|| true` | Remove `|| true`; non-zero exit must fail |
| Flaky tests breaking CI | Time/order/shared state | Isolate, freeze time, fix or quarantine |
| Slow pipelines | Serial stages, no cache | Parallelize, cache deps, fast tests first |
| Secrets visible in logs | Double-quoted Groovy interpolation / echo | Single quotes with credentials; mask; don't echo |
| App unreachable after deploy | Binding 127.0.0.1 / port not open | `0.0.0.0`; open SG; `-p` mapping |
| Merge conflicts everywhere | Long-lived branches | Trunk-based, small PRs |
| A/B test "winner" disappears | Peeking/multiple testing/novelty | Predefine sample; correct; run full cycles |
| Velocity dropping | Technical debt | Register, prioritize, allocate capacity |

---

# 7️⃣ CI/CD SECURITY CHECKLIST 🔐 (BRIDGE TO UNIT V)

```
☐ Jenkins: no anonymous access, RBAC, HTTPS via reverse proxy, restricted port 8080, regular plugin/core updates
☐ Controller runs no builds (executors 0) in production; use isolated agents or ephemeral Docker/K8s agents
☐ Secrets only in credential stores/vaults; never in repo, Jenkinsfile text, or logs; rotate regularly; prefer short-lived tokens/OIDC/IAM roles
☐ Docker socket/group access treated as root-equivalent; scan images (Trivy); run containers as non-root
☐ Scan code (SAST), dependencies (SCA), IaC (Checkov), secrets (gitleaks) in the pipeline; quality gates block risky changes
☐ Branch protection + code review + signed commits for pipeline/infra changes; restrict who can edit Jenkinsfiles
☐ Audit logs: who deployed what, when, which artifact (traceability from commit to production)
☐ Webhook secret/IP allow-list; keep Jenkins backups (JENKINS_HOME) and test restore
```

---

# 8️⃣ LAB IDEAS BEYOND THE SYLLABUS

| Lab | Idea |
|---|---|
| A | Convert the Jenkinsfile into a **GitHub Actions** workflow and compare |
| B | **Multibranch pipeline** with PR builds + branch protection requiring CI success |
| C | Add **SonarQube** (Docker) with a Quality Gate stage |
| D | **Blue-green** with two containers and an Nginx switch; script the cut-over and rollback |
| E | **Canary** with Nginx weighted upstream (90/10) and a metrics check |
| F | Add a **BDD acceptance stage** (behave) and publish its JUnit report |
| G | Provision the Jenkins EC2 + security group with **Terraform/CloudFormation** (Unit II) |
| H | Run the pipeline on a **Docker agent** (`agent { docker { image 'python:3.12-slim' } }`) |
| I | Add **feature flag + A/B bucketing** (hash-based) to the Flask app and compute the z-test from simulated data |
| J | Add **Prometheus + Grafana** to watch the deployed container (Unit IV preview) |

---

# 9️⃣ THREE REAL-WORLD SCENARIOS

## Scenario A: "Releases take two weeks and always break something"
```
Problem: long-lived branches, manual testing, manual deployment, big batches
Fix: trunk-based + short PRs; CI on every commit (lint, unit tests, scans); pipeline as code; build once/deploy many;
     staging with automated acceptance tests; canary + feature flags; automated rollback; DORA metrics to track improvement
Concepts: CI/CD, branching, deployment strategies (Parts 4, 5, 7)
```

## Scenario B: "Twelve teams, one product, constant integration chaos"
```
Problem: dependencies, misaligned priorities, late integration
Fix: choose SAFe (if governance/portfolio needed) with ARTs, PI Planning and System Demos, or LeSS (if the org can flatten) with one backlog and feature teams;
     either way: shared CI/CD, trunk-based dev, contract tests, platform team for pipelines
Concepts: scaling Agile, Conway's Law, CI/CD (Parts 2, 4)
```

## Scenario C: "Velocity keeps falling and bugs keep rising"
```
Problem: technical debt and weak tests
Fix: measure (Sonar debt ratio, hotspots, DORA), register debt, reserve 15% capacity, Boy Scout rule, TDD on new code, quality gates on new code,
     strangler fig for the worst legacy module, communicate cost in lead-time terms to the business, A/B tests to focus features on proven value
Concepts: technical debt, TDD, product strategy, A/B (Parts 3, 6)
```

---

# 🔟 30 BEST UNIT III VIVA QUESTIONS 🔥 (WITH ANSWERS)

**Agile and scaling**

1. **Agile Manifesto values?** Individuals and interactions over processes and tools; working software over comprehensive documentation; customer collaboration over contract negotiation; responding to change over following a plan.
2. **Scrum roles/events/artifacts?** Roles: PO, Scrum Master, Developers. Events: Sprint, Planning, Daily Scrum, Review, Retrospective. Artifacts: Product Backlog, Sprint Backlog, Increment (with commitments Product Goal, Sprint Goal, DoD).
3. **Scrum vs Kanban?** Sprints and roles vs continuous flow with WIP limits.
4. **What is the Definition of Done?** Shared checklist defining when an item is complete and releasable.
5. **Why scale Agile and what are the challenges?** To coordinate many teams on one product; dependencies, integration, alignment, governance.
6. **Explain SAFe's ART and PI.** ART: long-lived team of agile teams (50-125 people); PI: 8-12 week planning increment with several iterations and an IP iteration.
7. **What is PI Planning?** Two-day event where all ART teams plan the PI, creating objectives, a Program Board, risks and a confidence vote.
8. **What is WSJF?** Cost of Delay ÷ Job size, used to prioritize.
9. **LeSS in brief?** Large-Scale Scrum: one PO, one backlog, one DoD, one shippable increment, feature teams; LeSS for 2-8 teams, LeSS Huge for more.
10. **SAFe vs LeSS?** SAFe adds structure/governance for large enterprises; LeSS keeps Scrum minimal and de-scales the organization.

**TDD, BDD and CI/CD**

11. **Explain Red-Green-Refactor.** Write a failing test, write minimal code to pass, then refactor with tests green.
12. **Test pyramid?** Many fast unit tests, fewer integration tests, very few UI/E2E tests.
13. **What is BDD and Gherkin?** Behavior described in plain-language Given-When-Then scenarios (Gherkin) that become executable tests, built by the Three Amigos.
14. **TDD vs BDD?** TDD is developer-level unit testing in code; BDD is team-level behavior specs in plain language, often using TDD underneath.
15. **What is CI?** Frequent merging into a shared mainline with automatic build and tests on every commit.
16. **Continuous Delivery vs Deployment?** Delivery: always release-ready with manual production approval; Deployment: every passing change goes to production automatically.
17. **Stages of a pipeline?** Source, build, test, package, deploy to staging, verify, (approve), deploy to production, monitor.
18. **What does build once, deploy many mean?** One immutable artifact promoted through all environments.
19. **Blue-green vs canary?** Blue-green switches all traffic between two environments with instant rollback; canary gradually exposes a small percentage first.
20. **What is a feature flag?** Runtime toggle that decouples deployment from release and enables gradual rollout and kill switch.

**Jenkins**

21. **Jenkins architecture?** Controller manages UI, config, scheduling and plugins; agents with executors run builds; `JENKINS_HOME` stores state.
22. **Freestyle vs Pipeline?** UI-configured non-versioned job vs code-defined Jenkinsfile with complex flow support.
23. **Declarative pipeline structure?** `pipeline { agent; environment; options; stages { stage { steps } }; post }`.
24. **How does the GitHub webhook trigger a build?** Push → GitHub POSTs to `/github-webhook/` → Jenkins triggers jobs with the hook trigger enabled.
25. **How do you handle secrets in Jenkins?** Credentials store with `credentials()`/`withCredentials`, masked in logs, not hard-coded.

**Strategy, A/B, debt, best practices**

26. **RICE?** (Reach × Impact × Confidence) ÷ Effort.
27. **What is A/B testing and its key pitfalls?** Randomized comparison of control and variant on a metric using statistics; pitfalls: peeking, multiple comparisons, SRM, underpowered tests.
28. **What is technical debt and how do you manage it?** Future rework cost from shortcuts; manage via visible backlog, capacity allocation, refactoring with tests, quality gates, strangler fig, prioritization by interest and risk.
29. **Best practices for distributed teams?** Async-first written communication, overlap hours, working agreements, adapted ceremonies, small PRs with CI, documentation, security and wellbeing.
30. **Name key deployment best practices.** Small batches, trunk-based development, automation, immutable artifacts, IaC, quality gates, shift-left security, progressive delivery, fast rollback, observability/SLOs, blameless postmortems.

---

# 1️⃣1️⃣ COMMON UNIT III TRAPS 🚨

```
1. Confusing Continuous Delivery (manual approval) with Continuous Deployment (automatic)
2. Treating velocity as a performance target or comparing teams
3. Saying "SAFe has a single latest version number" (it dropped version numbers; say "evolves, e.g. SAFe 6.0 in 2023, AI-Native SAFe in 2026")
4. Mixing LeSS (one backlog, feature teams, minimal roles) with SAFe (many roles, ARTs, PI Planning)
5. Writing tests after code and calling it TDD
6. Thinking 100% coverage means no bugs
7. Calling BDD "just a testing tool" (it is a collaboration practice)
8. Believing CI = running Jenkins nightly (CI = integrate and test every commit, multiple times a day)
9. Rebuilding artifacts per environment instead of promoting one
10. Long-lived branches in a "CI/CD" claim
11. Saying canary and A/B are the same (safety vs business metric)
12. Hard-coding secrets in a Jenkinsfile; echoing credentials; double-quoted secret interpolation
13. Installing Jenkins before Java; running builds on the controller in production
14. Forgetting `/github-webhook/` trailing slash and the hook trigger checkbox
15. Using `source` in Jenkins `sh`; forgetting the jenkins user's docker group + restart
16. Peeking at A/B results and stopping early; ignoring guardrail metrics
17. Treating all technical debt as bad (prudent deliberate debt can be a smart trade-off if repaid)
18. Presenting exact market shares/versions as permanent facts
19. Leaving cloud resources running after the lab
```

---

# 1️⃣2️⃣ WHAT TO WRITE IN AN EXAM ✍️

| Question type | Structure |
|---|---|
| **Define Agile / compare with Waterfall** | Definition + manifesto values + 6-8 row table + example |
| **Explain Scrum** | Roles, events, artifacts + cycle diagram + DoD + benefits |
| **Explain SAFe / LeSS / compare** | Definition, structure diagram, key concepts (ART, PI / one backlog, feature teams), roles, events, benefits, criticisms, comparison table, when to use |
| **TDD / BDD** | Red-Green-Refactor diagram + tiny code example + Gherkin example + Three Amigos + comparison table + CI link |
| **CI/CD** | Definitions, Delivery vs Deployment table, pipeline diagram with stages, practices, tools, benefits, DORA |
| **Branching/deployment strategies** | Diagram per strategy + pros/cons table + when to use |
| **Jenkins** | Architecture diagram, installation steps, Jenkinsfile example, webhook flow, credentials, security |
| **A/B testing** | Definition + diagram + process + stats terms + numeric example + pitfalls + vs canary |
| **Technical debt** | Definition (principal/interest) + quadrant + types + measurement + management strategies + prioritization |
| **Best practices** | 8-10 numbered practices each with a one-line reason + checklist + conclusion |
| **Practical 6** | Aim, requirements, architecture diagram, numbered steps with commands, Jenkinsfile, outputs, rollback demo, cleanup, result |

Writing habits: underline keywords (Sprint, PI, ART, Red-Green-Refactor, Gherkin, pipeline, webhook, Jenkinsfile, canary, p-value), draw a diagram in every process answer, add one concrete example and one tool name per idea.

---

# 1️⃣3️⃣ WHAT NOT TO WRITE ❌

```
✘ Vague essays without frameworks, diagrams, or technical keywords
✘ "Agile means no documentation/no planning"
✘ Claims that one framework is "best" without context
✘ Exact numbers for tools/market share/versions stated as permanent facts
✘ Insecure sample code (passwords in Jenkinsfiles, open 0.0.0.0/0 on 22/8080 as "recommended")
✘ Mixing Jenkins freestyle UI steps with "pipeline as code" claims
✘ Describing CI/CD without mentioning automated tests, quality gates and rollback
✘ Ignoring human factors (collaboration, culture) in a "best practices" answer
```

---

# 1️⃣4️⃣ UNIT III MASTER SHEET 🧠🔥

```
AGILE      Manifesto (4 values, 12 principles) ; Scrum (3 roles, 5 events, 3 artifacts, DoD) ; Kanban (WIP) ; XP ; stories (INVEST) ; metrics
SCALING    SAFe: ART, PI (8-12 wks), PI Planning, RTE, WSJF, CDP/CALMR, 4 configs, 7 competencies, no version numbers → AI-Native SAFe (2026)
           LeSS: 1 PO / 1 backlog / 1 DoD / feature teams / Planning One+Two / LeSS Huge ; Nexus ; Scrum@Scale ; Spotify ; Conway's Law
TESTING    Pyramid ; FIRST ; TDD Red-Green-Refactor ; mocks ; BDD Given-When-Then ; Gherkin ; Three Amigos ; ATDD ; coverage ≠ quality
CI/CD      CI → Delivery (manual approve) → Deployment (auto) ; pipeline stages ; build once ; pipeline as code ; trunk-based ; SemVer ;
           rolling/blue-green/canary/shadow/A-B/flags ; expand-contract ; rollback ; DORA (4 + rework)
JENKINS    Controller + agents ; Jenkinsfile (Declarative) ; stages/steps/post/when/parallel/input ; credentials ; webhook /github-webhook/ ;
           multibranch ; Docker agents ; shared libs ; JCasC ; security ; Java before Jenkins ; `. venv/bin/activate` ; docker group + restart
PRODUCT    vision→strategy→roadmap→backlog ; MVP ; Build-Measure-Learn ; OKR ; AARRR ; RICE/ICE/MoSCoW/Kano
TEAMS      distributed: async-first, overlap, working agreements ; A/B: hypothesis, α, p, power, SRM, peeking ; vs canary
DEBT       principal+interest ; quadrant ; measure ; Boy Scout ; 10-20% capacity ; strangler fig ; priority = pain×freq×risk÷effort
BEST PRACT reviews + DoD + retros + blameless + docs ; small batches + automation + gates + flags + observability + rollback
P6         repo → EC2 + Jenkins + Docker → Pipeline from SCM + hook trigger → webhook → push = auto lint/test/build/deploy/smoke/rollback → cleanup
```

---

# 1️⃣5️⃣ MEMORY MAPS 🗺️

## Unit map
```
Unit III ─┬─ PEOPLE & PROCESS: Agile → Scrum/Kanban/XP → Scaling (SAFe, LeSS) → Strategy → Distributed teams
          ├─ QUALITY:          TDD → BDD → Test pyramid → Technical debt management
          ├─ DELIVERY:         CI → CD (Delivery/Deployment) → Pipelines → Branching → Deployment strategies → Jenkins
          └─ LEARNING:         A/B testing → Metrics (DORA, OKR) → Retros → Best practices
```

## Pipeline map
```
git push ─webhook▶ Jenkins ─▶ Checkout ─▶ Setup ─▶ Lint ─▶ Test(+reports) ─▶ Build image ─▶ Deploy ─▶ Smoke test ─▶ (rollback on failure)
                                                                      ▲ fail = stop, notify, nothing deployed
```

## Test map
```
BDD scenario (outer, business) ─▶ TDD unit cycles (inner, code) ─▶ CI runs all ─▶ quality gate ─▶ deploy
```

## Scaling map
```
1 team: Scrum ─▶ few teams: Nexus / LeSS (2-8) ─▶ many teams: SAFe ART(s) ─▶ enterprise: SAFe Portfolio
Always underneath: CI/CD + trunk-based + automated tests + feature flags
```

---

# 1️⃣6️⃣ HOW UNIT III CONNECTS TO THE REST OF INT378 🔗

| Unit | Connection |
|---|---|
| **Unit I** Cloud infrastructure and Docker | EC2 hosts Jenkins; Docker images are the pipeline's artifact; Netflix/Amazon/Uber case studies use microservices, small teams and continuous deployment |
| **Unit II** Infrastructure as Code | Provision the Jenkins server and environments with Terraform/CloudFormation; pipelines run `terraform plan/apply` or `deploy`; JCasC is IaC for Jenkins |
| **Unit IV** Monitoring and observability | Post-deploy verification, canary analysis, SLOs, alerts, DORA dashboards; Prometheus/Grafana watch Jenkins and the app |
| **Unit V** DevSecOps and cloud security | Add SAST/SCA/image/IaC scans and secrets management to the Jenkins pipeline (Practical 7); secure Jenkins itself; IAM for pipelines |
| **Unit VI** Cloud-native and cost | 12-factor config, serverless pipelines, cost of CI infrastructure, tagging and budgets, governance and compliance evidence from pipeline logs |

```
Unit I (what runs) → Unit II (define as code) → Unit III (deliver continuously) → Unit IV (observe) → Unit V (secure) → Unit VI (optimize and govern)
```

---

# 🏆 UNIT III — FINAL TAKEAWAY

```
1. Agile + DevOps: small, frequent, well-understood changes delivered safely through automation.
2. Scale carefully: SAFe adds structure and governance; LeSS de-scales and keeps Scrum simple; both need CI/CD underneath.
3. Quality comes from TDD (right code) and BDD (right behavior) running automatically in the pipeline.
4. CI/CD pipelines build once, test everything, deploy with progressive strategies, verify, and roll back fast.
5. Jenkins implements this as pipeline-as-code, triggered by GitHub webhooks, with credentials, agents and security.
6. Strategy, A/B experiments and technical-debt management keep the team building the right thing in a healthy codebase.
7. Best practices = collaboration culture + deployment engineering + feedback metrics.
8. You can do it yourself: push to GitHub → Jenkins lints, tests, builds, deploys, smoke-tests and rolls back automatically.
```

---

## 🫡 UNIT III STATUS

✅ Part 1 Agile from scratch
✅ Part 2 Scaling Agile (SAFe, LeSS)
✅ Part 3 TDD and BDD
✅ Part 4 CI/CD concepts
✅ Part 5 Jenkins deep dive
✅ Part 6 Product strategy, distributed teams, A/B testing, technical debt
✅ Part 7 Best practices for collaboration and deployment
✅ Part 8 Practical lab guide (Jenkins + GitHub webhook pipeline)
✅ Grand Consolidation

## Next up in your syllabus: UNIT IV, Monitoring and Observability (Prometheus, Grafana, metrics, alerting, dashboards, scaling monitoring, advanced observability) 🚀

Send the go-ahead and I'll build Unit IV the same way.
