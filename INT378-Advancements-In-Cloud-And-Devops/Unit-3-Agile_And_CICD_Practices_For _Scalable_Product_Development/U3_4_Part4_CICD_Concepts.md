# 🔄 UNIT III — PART 4: CI/CD CONCEPTS (CONTINUOUS INTEGRATION, DELIVERY AND DEPLOYMENT)

> **Covered in this document:** the pain CI/CD removes (integration hell, manual releases) → **Continuous Integration** (definition, practices, benefits, the CI loop) → **Continuous Delivery vs Continuous Deployment** (the classic confusion, table) → the anatomy of a **pipeline** (source, build, test, package, release, deploy, verify, monitor) → environments and promotion → artifacts, versioning and registries → **pipeline as code** → **branching strategies** (GitFlow, GitHub Flow, trunk-based) and pull-request workflow → semantic versioning and conventional commits → **deployment strategies** (recreate, rolling, blue-green, canary, shadow, feature flags/dark launches) with diagrams → database migrations (expand/contract) → rollback vs roll-forward → quality gates → CI/CD tools landscape and comparison (Jenkins, GitHub Actions, GitLab CI, AWS CodePipeline, Argo CD...) → a pipeline in GitHub Actions YAML for comparison → **DORA metrics** → a full dry run of one commit's journey → common mistakes → exam answer template → viva → master sheet.

---

Absolutely mate. 😤🔥 Parts 1-3 gave us the **people practices** (Agile, scaling) and **code practices** (TDD/BDD). Part 4 is the **delivery engine**: CI/CD. This is the heart of Unit III and the theory behind **Practical 6 (Jenkins)**.

---

# 1️⃣ THE PAIN CI/CD REMOVES 😫

## 1.1 Before CI/CD

```
Developers work on separate branches/copies for weeks
        │
        ▼
"INTEGRATION DAY": merge everything → hundreds of conflicts, nothing compiles, tests fail → "integration hell" 🔥
        │
        ▼
Manual build on someone's laptop → manual testing → copy files to server by FTP → "release weekend" with downtime
        │
        ▼
Something breaks in production → nobody knows what changed → panic rollback
```

| Problem | Effect |
|---|---|
| Big-bang integration | Painful merges, late bug discovery |
| Manual builds/deploys | Slow, error-prone, "works on my machine" |
| Infrequent releases | Big risky releases, long feedback loops |
| No traceability | Don't know which commit is in production |
| Fear of releasing | Release becomes an event, not a routine |

## 1.2 The CI/CD idea

> **Integrate small changes often, verify every change automatically, and make releasing a boring, repeatable, push-button (or fully automatic) process.**

```
Small batch size  +  automation  +  fast feedback  =  low risk, high speed
```

---

# 2️⃣ CONTINUOUS INTEGRATION (CI) ⭐⭐⭐

## 2.1 Definition

> **Continuous Integration** is the practice of developers **merging their changes into a shared mainline frequently (at least daily)**, with each merge **automatically built and tested** by a CI server to detect integration problems early. (Popularized by Martin Fowler / XP.)

## 2.2 The CI loop

```
Developer ──commit/push──▶ Version control (Git)
                               │ webhook / poll
                               ▼
                        CI SERVER (Jenkins, GitHub Actions...)
                               │
          ┌────────────────────┼──────────────────────────────┐
          ▼                    ▼                              ▼
     1. Checkout          2. Build/compile            3. Run automated tests
                                                      (unit, lint, static analysis)
                               │
                   ┌───────────┴───────────┐
                  PASS ✅                 FAIL ❌
        produce build artifact     notify developer immediately (email/Slack)
        (ready for CD)             → fix the build FIRST (stop the line)
```

## 2.3 CI practices (Fowler's checklist) ✅

| Practice | Meaning |
|---|---|
| **Maintain a single source repository** | Everything needed to build is in version control |
| **Automate the build** | One command builds the product |
| **Make the build self-testing** | Tests run in the build |
| **Everyone commits to mainline every day** | Small, frequent merges |
| **Every commit triggers a build on an integration machine** | Not on dev laptops |
| **Keep the build fast** | Ideal: under ~10 minutes |
| **Fix broken builds immediately** | Top priority for the team |
| **Test in a clone of production** | Similar environment (containers help: Unit I) |
| **Make it easy to get the latest executable** | Artifacts accessible |
| **Everyone can see what's happening** | Dashboards, notifications |

## 2.4 Benefits of CI

```
✔ Bugs found within minutes, not weeks         ✔ Small, easy-to-understand changes
✔ Always-working main branch                    ✔ Less merge pain
✔ Faster, more confident development            ✔ Automated, reproducible builds
✔ Visibility of project health                  ✔ Foundation for delivery automation
```

---

# 3️⃣ CONTINUOUS DELIVERY vs CONTINUOUS DEPLOYMENT ⭐⭐⭐ (CLASSIC EXAM QUESTION)

The "CD" in CI/CD means **two different things**:

| | **Continuous Delivery** | **Continuous Deployment** |
|---|---|---|
| Every change that passes the pipeline… | Is **ready to be released** to production | Is **automatically released** to production |
| Final production release | **Manual approval/button** (business decides when) | **No human step**: fully automatic |
| Risk appetite | Moderate; compliance/regulation friendly | High maturity; needs excellent tests, monitoring, rollback |
| Release frequency | On business demand (daily-weekly...) | Many times a day |
| Typical users | Banks, enterprises, regulated industries | Netflix, Amazon, Etsy, SaaS products |

```
CODE → BUILD → TEST → STAGING ─────────▶ [ manual approval ] ──▶ PRODUCTION     = CONTINUOUS DELIVERY
CODE → BUILD → TEST → STAGING ─────────▶ (automatic, if all checks pass) ──▶ PRODUCTION = CONTINUOUS DEPLOYMENT
```

Memory trick: **Delivery = deployable at any time. Deployment = deployed automatically every time.**

The full picture:

```
Continuous Integration  ⊂  Continuous Delivery  ⊂  Continuous Deployment
(merge+build+test)         (+ always release-ready)    (+ auto-release to prod)
```

---

# 4️⃣ ANATOMY OF A CI/CD PIPELINE 🏗️ ⭐⭐⭐

> A **pipeline** is an automated, ordered set of stages that takes a code change from commit to production, giving fast feedback at each step.

```
┌────────┐  ┌───────┐  ┌───────┐  ┌─────────┐  ┌─────────┐  ┌────────┐  ┌─────────┐  ┌────────┐
│ SOURCE │─▶│ BUILD │─▶│ TEST  │─▶│ PACKAGE │─▶│ DEPLOY  │─▶│ VERIFY │─▶│ DEPLOY  │─▶│MONITOR │
│ commit │  │compile│  │ unit  │  │ docker  │  │ to dev/ │  │ smoke/ │  │  to     │  │ metrics│
│ webhook│  │ deps  │  │ lint  │  │ image / │  │ staging │  │ accept.│  │  PROD   │  │ alerts │
│        │  │       │  │ scan  │  │ artifact│  │         │  │ tests  │  │(+approve)│ │        │
└────────┘  └───────┘  └───────┘  └─────────┘  └─────────┘  └────────┘  └─────────┘  └────────┘
        ◀──────── fail fast: any red stage stops the pipeline and notifies ────────▶
```

## 4.1 Stage by stage

| Stage | What happens | Examples |
|---|---|---|
| **Source/Trigger** | Commit, PR, tag or schedule starts the pipeline | Git webhook |
| **Build** | Compile, resolve dependencies | `mvn package`, `npm ci && npm run build`, `pip install -r requirements.txt` |
| **Static analysis / lint** | Code quality, style, bugs | flake8, pylint, ESLint, SonarQube |
| **Unit tests** | Fast tests; produce reports/coverage | pytest, JUnit, Jest |
| **Security scan** | Dependency + code + image scanning | OWASP Dependency-Check, Trivy, Snyk (Unit V) |
| **Package/Publish** | Create **immutable artifact** and store it | Docker image → registry, JAR/wheel → Nexus/Artifactory/S3 |
| **Deploy to test/staging** | Release artifact to non-prod (often via IaC/containers) | Docker run, Kubernetes, Terraform (Unit II) |
| **Integration/acceptance/E2E tests** | Verify deployed system | behave/Cucumber, Selenium, Postman/Newman |
| **Performance test (optional)** | Load tests | k6, JMeter |
| **Approval gate** (Continuous Delivery) | Human sign-off | Jenkins `input`, GitHub Environments |
| **Deploy to production** | Rolling/blue-green/canary strategies | See Section 8 |
| **Post-deploy verification and monitoring** | Smoke tests, metrics, alerts, auto-rollback | Prometheus/Grafana (Unit IV) |

## 4.2 Golden principle: **build once, deploy many**

```
Build the artifact (e.g., Docker image myapp:1.4.2-ab12cd3) ONCE.
Promote THE SAME artifact through dev → test → staging → prod.
Only configuration (env vars/secrets) differs per environment.
✘ Never rebuild for each environment (you'd test one thing and ship another).
```

## 4.3 Environments and promotion

| Environment | Purpose | Deploy trigger |
|---|---|---|
| **Local/dev** | Developer machine, quick feedback | Manual |
| **CI** | Build + unit tests on every commit | Every commit |
| **Dev/Integration** | Shared integration | Merge to main |
| **QA/Test** | Acceptance/regression | Automatic |
| **Staging/Pre-prod** | Production-like final check | Automatic/manual |
| **Production** | Real users | Manual approval (Delivery) or automatic (Deployment) |

Keep environments **as similar as possible** (containers + IaC from Units I-II) to avoid "works in staging, fails in prod."

## 4.4 Artifacts, versioning and registries

| Item | Detail |
|---|---|
| **Artifact** | The deployable output (Docker image, JAR/WAR, wheel, zip, AMI) |
| **Artifact repository** | Nexus, JFrog Artifactory, AWS ECR/S3/CodeArtifact, Docker Hub, GitHub Packages |
| **Versioning** | Semantic Versioning `MAJOR.MINOR.PATCH` (2.5.1); plus build number/commit SHA: `myapp:2.5.1-b57-ab12cd3` |
| **Immutability** | Never overwrite a published version; avoid relying on `latest` in production |
| **Traceability** | Every deployed artifact maps back to a commit and a pipeline run |

### Semantic Versioning (SemVer) ⭐

```
MAJOR.MINOR.PATCH
  │     │     └── bug fixes, backward compatible        (1.4.2 → 1.4.3)
  │     └──────── new features, backward compatible     (1.4.3 → 1.5.0)
  └────────────── breaking changes                       (1.5.0 → 2.0.0)
```

### Conventional Commits (helps automate versioning and changelogs)

```
feat: add wishlist endpoint          → MINOR bump
fix: correct coupon rounding         → PATCH bump
feat!: change API response format    → MAJOR bump (breaking)
docs:, chore:, refactor:, test:, ci:
```

---

# 5️⃣ PIPELINE AS CODE 📜 ⭐

> **Pipeline as Code**: the pipeline definition lives in a text file **in the same repository as the application** (version-controlled, reviewed, reusable), like IaC in Unit II.

| Tool | File |
|---|---|
| **Jenkins** | `Jenkinsfile` (Declarative or Scripted Groovy) |
| **GitHub Actions** | `.github/workflows/*.yml` |
| **GitLab CI** | `.gitlab-ci.yml` |
| **CircleCI** | `.circleci/config.yml` |
| **Azure Pipelines** | `azure-pipelines.yml` |
| **AWS CodeBuild** | `buildspec.yml` |
| **Bitbucket** | `bitbucket-pipelines.yml` |

Benefits: versioned history, code review for pipeline changes, per-branch pipelines, easy recreation, no "click-configured snowflake" jobs.

---

# 6️⃣ BRANCHING STRATEGIES ⭐⭐⭐

How you use Git determines how well CI works.

## 6.1 Git basics refresher

```
git clone URL           git checkout -b feature/wishlist     git add . && git commit -m "feat: wishlist"
git push -u origin feature/wishlist     git pull --rebase    git merge / git rebase
Pull Request (PR) / Merge Request (MR) = ask to merge a branch, with review + automated checks
```

## 6.2 GitFlow 🌊 (heavy, release-oriented)

```
main      ●───────────────────●(v1.0)────────────────●(v1.1)
                               ▲                      ▲
release                 ●──●──┘ (release/1.0)   ●──●──┘
                        ▲                        ▲
develop  ●──●──●──●──●──┴──●──●──●──●──●──●──●───┘
           ▲  ▲        ▲
feature    └──┘        └── feature/xyz
hotfix                      (from main, merged into main + develop)
```

| Branch | Purpose |
|---|---|
| `main` | Production-ready code (tagged releases) |
| `develop` | Integration branch for next release |
| `feature/*` | New features (long-lived possible) |
| `release/*` | Stabilize a release |
| `hotfix/*` | Urgent production fixes |

Pros: clear structure, good for scheduled releases/versioned products. Cons: long-lived branches → merge pain, **slows CI/CD**; considered heavy for continuously delivered web apps.

## 6.3 GitHub Flow 🐙 (simple)

```
main (always deployable)
  └── short-lived feature branch → Pull Request → review + CI checks → merge → auto-deploy
```

Rules: `main` is always deployable; branch from main; open PR early; merge after review and green CI; deploy immediately after merge.

## 6.4 Trunk-Based Development 🌳 ⭐ (preferred for high-performing CI/CD teams)

```
trunk (main) ●─●─●─●─●─●─●─●─●─●   ← everyone commits here (or via branches living < 1-2 days)
              short-lived branches (hours to 1-2 days) merged quickly
```

| Practice | Why |
|---|---|
| Very short-lived branches | Avoid merge pain |
| Small commits, merge daily | True continuous integration |
| **Feature flags** hide unfinished work | Deploy ≠ release |
| Strong automated tests | Safe to integrate often |
| Release from trunk (or short release branches) | Simplicity |

DORA research associates trunk-based development with higher delivery performance.

## 6.5 Comparison table ⭐

| | GitFlow | GitHub Flow | Trunk-Based |
|---|---|---|---|
| Long-lived branches | Many (develop, release) | Only `main` | Only trunk |
| Branch lifetime | Days-weeks | Hours-days | Hours (≤ 1-2 days) |
| Release style | Scheduled, versioned | Continuous | Continuous/very frequent |
| Complexity | High | Low | Low (needs discipline) |
| CI friendliness | Lower | Good | Best |
| Needs feature flags | Less | Sometimes | **Yes** |
| Best for | Packaged software, multiple supported versions | Web apps/SaaS, small teams | High-velocity teams, microservices |

## 6.6 Pull request workflow with CI ✅

```
1. Create branch  2. Commit small changes  3. Push  4. Open PR
5. CI runs automatically on the PR (build, tests, lint, scans) → status checks
6. Reviewers comment/approve (at least 1) 7. All checks green + approvals → merge (squash/merge/rebase)
8. CD pipeline deploys main  9. Delete the branch
```

**Branch protection rules:** require PR, require passing status checks, require approvals, block force-push, linear history optional.

---

# 7️⃣ QUALITY GATES AND SHIFT-LEFT 🚧

A **quality gate** is a condition that must be satisfied to proceed.

| Gate | Example threshold |
|---|---|
| Build success | Must compile |
| Unit tests | 100% pass |
| Code coverage | ≥ 80% (on new code) |
| Static analysis | No new critical/blocker issues (SonarQube Quality Gate) |
| Security scan | No critical/high vulnerabilities |
| Performance | p95 latency < 300 ms |
| Manual approval | Release manager sign-off (Delivery) |

**Shift-left** = move testing, security and quality checks earlier in the pipeline (cheaper to fix). **Shift-right** = test/monitor in production (canary, observability).

---

# 8️⃣ DEPLOYMENT STRATEGIES ⭐⭐⭐ (HOW TO RELEASE WITHOUT DOWNTIME)

| Strategy | How it works | Pros | Cons |
|---|---|---|---|
| **Recreate** (big bang) | Stop old version, start new | Simple, cheap | **Downtime** |
| **Rolling update** | Replace instances gradually (e.g., 1-2 at a time) | No downtime, low extra cost | Two versions run together; slower rollback |
| **Blue-Green** | Two identical environments; deploy to idle one; switch traffic at once | Instant switch and **instant rollback** | Double infrastructure cost; DB compatibility |
| **Canary** | Release to small % of users (1-5%), watch metrics, ramp up | Real-user validation, low blast radius | Needs good monitoring and traffic routing |
| **Shadow (dark traffic)** | Mirror production traffic to new version without affecting users | Safe real-world load test | Complex; side-effects must be isolated |
| **A/B testing** | Different versions to different user groups to compare business metrics | Data-driven product decisions | Needs experiment platform (Part 6) |
| **Feature flags / dark launch** | Code deployed but hidden behind a toggle; enable per user/percentage | Decouples deploy from release; instant kill switch | Flag debt; testing combinations |

## 8.1 Diagrams

```
ROLLING
 before:  [v1][v1][v1][v1]
 step 1:  [v2][v1][v1][v1]
 step 2:  [v2][v2][v1][v1]
 after :  [v2][v2][v2][v2]

BLUE-GREEN
 Users ──▶ Load balancer ──▶ BLUE (v1, live)        GREEN (v2, idle, being tested)
 switch:   Users ──▶ Load balancer ──▶ GREEN (v2, live)   BLUE (v1 kept for instant rollback)

CANARY
 Users ──▶ LB ──95%──▶ v1 fleet
              └─5%───▶ v2 canary ──metrics OK?── yes → 25% → 50% → 100%
                                                   no  → rollback to 0%
```

## 8.2 Feature flags (toggles) 🚩 ⭐

```python
# simple example
if flags.is_enabled("new_checkout", user_id=user.id):
    return new_checkout_flow()
return old_checkout_flow()
```

| Flag type | Purpose |
|---|---|
| Release toggle | Hide incomplete work (supports trunk-based development) |
| Experiment toggle | A/B tests |
| Ops toggle | Kill switch under load |
| Permission toggle | Premium/beta users |

Tools: LaunchDarkly, Unleash, Flagsmith, Split, AWS AppConfig, simple config/env vars. **Rule:** remove flags after rollout (flag debt = technical debt).

## 8.3 Database changes and zero-downtime releases (expand/contract)

Never break the old version while deploying the new one:

```
1. EXPAND   : add new column/table (nullable/default) → old code unaffected
2. MIGRATE  : deploy new code that writes both/reads new; backfill data
3. CONTRACT : remove old column in a later release after everything uses the new one
```

Tools: Flyway, Liquibase, Alembic, Django/Rails migrations. Make migrations **versioned, automated, backward-compatible**.

## 8.4 Rollback vs roll-forward

| | Rollback | Roll-forward |
|---|---|---|
| Action | Redeploy previous known-good artifact | Fix quickly and ship a new version |
| When | Clear regression; fast recovery | Small fix easy; DB changes irreversible |
| Enabler | Immutable versioned artifacts, blue-green, flags | Fast pipeline, small changes |

Rollback is easy when you **keep old artifacts**, use **blue-green/canary**, and keep DB changes **backward compatible**.

---

# 9️⃣ CI/CD TOOLS LANDSCAPE 🧰

| Tool | Type | Notes |
|---|---|---|
| **Jenkins** | Self-hosted automation server | Open source, huge plugin ecosystem, `Jenkinsfile`; **your Practical 6** (Part 5) |
| **GitHub Actions** | Hosted (and self-hosted runners) | YAML workflows in repo; marketplace actions; tight GitHub integration |
| **GitLab CI/CD** | Built into GitLab | `.gitlab-ci.yml`; integrated registry, security scanning |
| **CircleCI, Travis CI, Bitbucket Pipelines** | SaaS CI | Easy setup |
| **Azure DevOps Pipelines** | Microsoft | YAML + classic; strong Azure integration |
| **AWS CodePipeline + CodeBuild + CodeDeploy** | AWS-native | Fits AWS stack (Unit I/II) |
| **Argo CD / Flux** | GitOps CD for Kubernetes | Pull-based deployment |
| **Spinnaker** | Multi-cloud CD (from Netflix, Unit I case study) | Canary/blue-green pipelines |
| **Tekton** | Kubernetes-native CI/CD | Cloud-native pipelines |
| **TeamCity, Bamboo, Octopus Deploy** | Commercial options | Enterprise |

### Jenkins vs GitHub Actions vs GitLab CI (quick comparison) ⭐

| | Jenkins | GitHub Actions | GitLab CI |
|---|---|---|---|
| Hosting | You host/maintain | GitHub-hosted runners (or self-hosted) | GitLab.com or self-managed |
| Setup effort | High (install, plugins, upgrades, security) | Very low | Low |
| Config | `Jenkinsfile` (Groovy) / UI | YAML | YAML |
| Extensibility | 1,800+ plugins; very flexible | Marketplace actions | Built-in features + templates |
| Cost | Free software; infra + maintenance cost | Free tier; usage-based paid | Free tier; paid tiers |
| Strength | Total control, any environment, legacy support | Convenience, ecosystem, PR integration | All-in-one DevOps platform |
| Weakness | Plugin/maintenance burden, security upkeep | Vendor lock-in to GitHub | Platform lock-in |

---

# 🔟 THE SAME PIPELINE IN GITHUB ACTIONS (FOR COMPARISON) 🆚

`.github/workflows/ci.yml`:

```yaml
name: ci
on:
  push:
    branches: [main]
  pull_request:

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -r requirements.txt pytest pytest-cov
      - run: pytest --junitxml=reports/pytest.xml --cov=.
      - uses: actions/upload-artifact@v4
        with:
          name: test-reports
          path: reports/

  docker:
    needs: build-test               # runs only if tests passed
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USER }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: myuser/myapp:${{ github.sha }}
```

Note the same ideas: trigger, stages (jobs), dependencies (`needs`), secrets, artifacts. The Jenkins equivalent (a `Jenkinsfile`) is built step by step in Part 5 and Part 8.

---

# 1️⃣1️⃣ DORA METRICS: MEASURING DELIVERY PERFORMANCE 📈 ⭐⭐

The **DORA** (DevOps Research and Assessment, Google Cloud) research identified metrics that predict software delivery and organizational performance:

| Metric | Measures | Meaning |
|---|---|---|
| **Deployment Frequency** | Throughput | How often you deploy to production |
| **Lead Time for Changes** | Throughput | Time from commit to running in production |
| **Change Failure Rate** | Stability | % of deployments causing failures needing a fix/rollback |
| **Time to Restore Service** (recent DORA reports call it *failed deployment recovery time*) | Stability | How fast you recover from a failed deployment/incident |

Recent DORA reports have also discussed an additional measure, **rework rate** (how often unplanned deployments are needed to fix problems), and group teams into performance levels rather than a simple "elite/high/medium/low" label in some editions. Exact thresholds change between reports, so quote **ideas** (speed *and* stability go together), not fixed numbers.

```
Key finding: speed and stability are NOT trade-offs. High performers deploy more often AND break less.
```

Practices linked with better DORA metrics: **CI/CD, trunk-based development, test automation, loosely coupled architecture, small batch sizes, monitoring, team autonomy, and a generative culture.**

---

# 1️⃣2️⃣ DRY RUN: ONE COMMIT'S JOURNEY TO PRODUCTION 🏃

```
10:02  Dev Meera finishes "coupon rounding fix" with a failing test first (TDD), commits to branch fix/coupon-rounding
10:03  git push → opens Pull Request → webhook triggers CI
10:03  CI: checkout → pip install → flake8 (ok) → pytest: 42 tests, 41 passed, 1 FAILED ❌ (her own refactor broke a rule)
10:04  Slack/GitHub status red; she sees the failing test log in 30 seconds; fixes; pushes again
10:09  CI green ✅ (tests 42/42, coverage 88%, Sonar gate passed, Trivy scan: no critical) ; teammate approves PR
10:11  Merge to main → pipeline: build Docker image myapp:1.7.3-4f2a9c1 → push to registry
10:14  Auto-deploy to STAGING → behave acceptance tests (BDD) 12/12 passed ; smoke test /health = 200
10:15  CONTINUOUS DELIVERY: manager clicks "Approve" ; (Continuous Deployment would skip this click)
10:16  Canary: 5% of users on 1.7.3 → Prometheus watches error rate and latency for 10 minutes
10:26  Metrics healthy → ramp to 100% ; old version retained for instant rollback
10:27  Total time from commit to production: ~25 minutes; change failure: none ✔
```

---

# 1️⃣3️⃣ COMMON CI/CD MISTAKES 🚨

| Mistake | Impact | Fix |
|---|---|---|
| Slow pipelines (30-60 min) | Developers stop waiting, batch changes | Parallelize, cache dependencies, run fast tests first |
| Flaky tests | Distrust, ignored red builds | Fix/quarantine; no retries as a crutch |
| Rebuilding per environment | Tested ≠ shipped | Build once, promote artifact |
| Long-lived branches | Merge hell | Trunk-based, short branches, flags |
| Secrets in repo/pipeline logs | Breach | Credential stores, masked logs, vaults (Unit V) |
| Manual steps in the "automated" pipeline | Errors, slowness | Automate everything repeatable |
| No rollback plan | Long outages | Keep artifacts, blue-green/canary, tested rollback |
| Broken main tolerated | Blocks everyone | Stop the line; revert fast |
| Pipeline config clicked in UI only | Not reproducible | Pipeline as code |
| No monitoring after deploy | Failures discovered by users | Post-deploy checks, alerts (Unit IV) |
| Using `latest` tags | Unclear what's running | Immutable version tags |
| Treating CI/CD as only a tool | Culture unchanged | Small batches, ownership, blameless culture |

---

# 1️⃣4️⃣ EXAM ANSWER TEMPLATE ✍️

**Q. Explain CI/CD with a pipeline diagram. Differentiate Continuous Delivery and Deployment.**

```
1. Intro: CI/CD automates integrating, testing and releasing software frequently and reliably.
2. CI: definition + loop (commit → build → test → feedback) + benefits.
3. CD: Delivery vs Deployment (table; manual approval vs automatic).
4. Pipeline diagram with stages: Source → Build → Test → Package → Deploy (staging) → Verify → Deploy (prod) → Monitor.
5. Key practices: build once deploy many, pipeline as code, trunk-based/short branches, quality gates, artifact versioning.
6. Deployment strategies (rolling, blue-green, canary, feature flags) in a small table.
7. Tools: Jenkins, GitHub Actions, GitLab CI, AWS CodePipeline.
8. Benefits (speed, quality, low risk) + DORA metrics; Conclusion.
```

---

# 1️⃣5️⃣ VIVA QUESTIONS (PART 4) 🎯

**Q1. What is Continuous Integration?** Developers merge to a shared mainline frequently, each merge automatically built and tested to find problems early.

**Q2. Continuous Delivery vs Continuous Deployment?** Delivery keeps every change release-ready with a manual approval for production; Deployment releases every passing change automatically.

**Q3. What is a pipeline?** An automated sequence of stages (build, test, package, deploy) that moves a change from commit to production with feedback at each step.

**Q4. What does "build once, deploy many" mean?** Create one immutable artifact and promote the same artifact through all environments.

**Q5. What is pipeline as code?** Defining the pipeline in a versioned file like `Jenkinsfile` stored with the application.

**Q6. GitFlow vs trunk-based development?** GitFlow uses long-lived develop/release branches; trunk-based uses a single trunk with very short-lived branches and feature flags, enabling better CI.

**Q7. What is a quality gate?** A condition (tests, coverage, security, static analysis) that must be met to proceed in the pipeline.

**Q8. Explain blue-green deployment.** Two identical environments; deploy to the idle one and switch traffic; rollback by switching back.

**Q9. Explain canary deployment.** Release to a small percentage of users, monitor, then gradually increase or roll back.

**Q10. What is a feature flag?** A toggle that enables or hides functionality at runtime, decoupling deployment from release.

**Q11. What are the DORA metrics?** Deployment frequency, lead time for changes, change failure rate, and time to restore (failed deployment recovery time).

**Q12. What is semantic versioning?** MAJOR.MINOR.PATCH to signal breaking changes, new features and bug fixes.

**Q13. How do you handle database changes in zero-downtime deployments?** Backward-compatible migrations with expand/contract.

**Q14. Why are artifacts immutable?** To ensure what was tested is exactly what's deployed and allow reliable rollback.

**Q15. Why is trunk-based development good for CI?** Frequent small merges avoid long-lived branch divergence and integration pain.

---

# 🧠 PART 4 MASTER SHEET

```
CI = merge often + automatic build + automatic tests + fast feedback + fix broken build immediately
CD (Delivery) = always release-ready, manual prod approval | CD (Deployment) = fully automatic to prod
Pipeline: Source → Build → Test(lint, unit, scan) → Package(artifact/image) → Deploy staging → Verify(acceptance/smoke) → [approve] → Prod → Monitor
Principles: build once deploy many ; immutable versioned artifacts ; pipeline as code ; fail fast ; quality gates ; shift-left
Branching: GitFlow (heavy) | GitHub Flow (simple) | Trunk-based (best for CI; short branches + feature flags)
SemVer MAJOR.MINOR.PATCH ; conventional commits ; PR + status checks + branch protection
Deploy strategies: recreate | rolling | blue-green | canary | shadow | A/B | feature flags ; expand/contract DB migrations
Rollback (redeploy old artifact) vs roll-forward (ship fix)
Tools: Jenkins, GitHub Actions, GitLab CI, CircleCI, Azure Pipelines, AWS CodePipeline, Argo CD, Spinnaker, Tekton
DORA: deployment frequency, lead time, change failure rate, time to restore (+ rework rate) → speed AND stability
```

---

## 🫡 PART 4 STATUS

You understand CI, Continuous Delivery vs Deployment, pipeline anatomy, branching strategies, deployment strategies, feature flags, tools and DORA metrics.

**STOP here.** Next up:

> **PART 5 — Jenkins Deep Dive (architecture, installation, plugins, Freestyle vs Pipeline, Jenkinsfile, agents, credentials, GitHub webhooks, shared libraries, security) 🔥**
