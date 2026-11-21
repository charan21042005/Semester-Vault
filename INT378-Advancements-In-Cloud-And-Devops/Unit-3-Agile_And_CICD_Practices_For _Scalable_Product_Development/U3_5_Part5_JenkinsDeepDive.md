# 🔧 UNIT III — PART 5: JENKINS DEEP DIVE (AUTOMATION SERVER FOR CI/CD)

> **Covered in this document:** what Jenkins is and why it is still everywhere → history and ecosystem → **architecture** (controller, agents/nodes, executors, labels, workspace, `JENKINS_HOME`, build queue, plugins) → installation (Ubuntu with Java 21, Docker, Amazon Linux notes, ports, first-run wizard, initial admin password) → UI tour and core vocabulary → job types (Freestyle, Pipeline, Multibranch, Folders) → **Freestyle vs Pipeline** → **Declarative Pipeline syntax in full** (agent, environment, options, parameters, triggers, tools, stages, steps, when, post, parallel, input, matrix) → Scripted Pipeline → a complete Python CI Jenkinsfile → a Docker build/push/deploy Jenkinsfile → built-in environment variables → **credentials** (types, `withCredentials`, masking) → **triggers** (webhooks, SCM polling, cron, upstream, parameters) → GitHub integration and the webhook flow → Multibranch pipelines and PR builds → agents and distributed builds (SSH agent, Docker agents, Kubernetes) → test reports, artifacts, notifications → shared libraries, JCasC, Blue Ocean status → **Jenkins security hardening and backup** → performance and scaling → troubleshooting table → viva → master sheet.

---

Absolutely mate. 😤🔥 Part 4 explained **what** CI/CD is. Part 5 is the **tool you'll actually use in Practical 6: Jenkins.**

---

# 1️⃣ WHAT IS JENKINS? ⭐

> **Jenkins** is an open-source **automation server** written in Java that orchestrates building, testing and deploying software. You describe a **pipeline** (build → test → package → deploy), and Jenkins runs it automatically whenever code changes.

Key facts:

```
Open source (MIT license)           Written in Java (runs on any OS with a supported JDK)
Self-hosted: you install and run it Huge plugin ecosystem (1,800+ plugins) → integrates with almost everything
Pipeline as Code (Jenkinsfile)      Distributed builds: one controller, many agents
Web UI on port 8080 by default      Governed by the Continuous Delivery Foundation (CDF)
```

## 1.1 History in 3 lines

```
2004  Kohsuke Kawaguchi creates "Hudson" at Sun Microsystems
2011  After a dispute with Oracle, the community forks it as "Jenkins"
Today Jenkins is one of the most widely used CI/CD servers, with LTS (Long-Term Support) releases every ~12 weeks plus weekly releases
```

## 1.2 Why Jenkins is still used

| Strength | Detail |
|---|---|
| Total control | Runs anywhere: on-prem, AWS EC2, Kubernetes, air-gapped networks |
| Flexibility | Any language/tool via plugins and shell commands |
| Mature | Years of docs, community answers, enterprise adoption |
| Pipeline as Code | Jenkinsfile reviewed in Git like application code |
| Free software | You pay for servers and admin time, not licences |

Honest downsides: you maintain it (upgrades, plugins, security), plugin compatibility issues, Groovy learning curve, older UI feel. (Hosted tools like GitHub Actions are easier to start with; see Part 4.)

---

# 2️⃣ JENKINS ARCHITECTURE ⭐⭐⭐ (DRAW THIS)

```
                         ┌─────────────────────────────────────────────┐
  Developer ─ git push ─▶│ GitHub / GitLab / Bitbucket                 │
                         └───────────────┬─────────────────────────────┘
                                         │ webhook (HTTP POST)
                                         ▼
 ┌────────────────────────────── JENKINS CONTROLLER ──────────────────────────────┐
 │ Web UI + REST API │ Job/pipeline configs │ Build queue │ Scheduler │ Plugins    │
 │ Credentials store │ Users & security     │ JENKINS_HOME (/var/lib/jenkins)      │
 └───────────────┬───────────────────────────────┬────────────────────────────────┘
                 │ assigns builds to agents       │
        ┌────────▼────────┐             ┌────────▼────────┐          ┌──────────────┐
        │ AGENT 1 (Linux) │             │ AGENT 2 (Docker)│          │ AGENT 3 (Win)│
        │ executors: 2    │             │ executors: 4    │          │ executors: 2 │
        │ labels: linux   │             │ labels: docker  │          │ labels: win  │
        │ workspace/      │             │ workspace/      │          │ workspace/   │
        └─────────────────┘             └─────────────────┘          └──────────────┘
```

| Component | Meaning |
|---|---|
| **Controller** (formerly "master") | The central brain: UI, configuration, scheduling, plugins, credentials. **Best practice: don't run builds on the controller** in production (set its executors to 0) |
| **Agent / Node** | A machine (or container) that actually executes build steps; connects to the controller |
| **Executor** | A slot on a node that runs one build at a time; a node with 2 executors can run 2 builds in parallel |
| **Label** | A tag on nodes (`linux`, `docker`, `gpu`) so pipelines can request matching agents |
| **Workspace** | A directory on the agent where the job checks out code and builds |
| **Build queue** | Pending builds waiting for a free executor |
| **`JENKINS_HOME`** | Directory storing all config, jobs, build history, plugins, secrets (`/var/lib/jenkins` on Linux packages, `/var/jenkins_home` in Docker). **This is what you back up** |
| **Plugin** | Extension adding features (Git, Pipeline, Docker, credentials, email, SonarQube...) |
| **Job/Project** | A configured automation unit |
| **Build** | One execution of a job (numbered `#1`, `#2`...) |
| **Stage / Step** | Pipeline structure: stages group steps; steps are actual commands |

Controller-agent connection methods: **SSH** (controller connects out), **inbound/WebSocket (JNLP)** (agent connects in), **Docker/Kubernetes dynamic agents** (spun up per build, destroyed after).

---

# 3️⃣ INSTALLING JENKINS 💻

## 3.1 Sizing

| Use | Minimum | Recommended |
|---|---|---|
| Tiny lab | 1 GB RAM | **2 GB+ RAM (t3.small)**; Jenkins + Docker builds are happier with **4 GB (t3.medium)** |
| Disk | 10 GB | 20-50 GB |
| Java | Java 17 or Java 21 (current Jenkins LTS no longer supports Java 11) | **Java 21** |

⚠️ A `t2.micro/t3.micro` (1 GB) often struggles (out-of-memory, slow). For Practical 6 use `t3.small` or `t3.medium` on EC2 and **stop/terminate when done** to control cost.

## 3.2 Option A: Ubuntu 22.04/24.04 (apt) ⭐ (used in Practical 6)

⚠️ Jenkins' apt repository signing keys are **rotated periodically** (a 2026 key is in use). **Always copy the exact commands from https://www.jenkins.io/doc/book/installing/linux/ ("Debian/Ubuntu" → "Long Term Support release")**. The shape of the commands is:

```bash
# 1. Java FIRST (Jenkins may fail to start if Java is installed after it)
sudo apt update
sudo apt install -y fontconfig openjdk-21-jre
java -version

# 2. Add Jenkins LTS repository and signing key (use the key file name shown on the Jenkins docs page)
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# 3. Install and start
sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins          # should show "active (running)"
```

(If `apt update` complains about an invalid/expired GPG key, fetch the current key name from the Jenkins page above.)

Also install tools the pipelines will need on the Jenkins machine:

```bash
sudo apt install -y git python3 python3-venv python3-pip curl
```

## 3.3 Option B: Docker (fast, disposable) 🐳

```bash
docker run -d --name jenkins -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts-jdk21
docker logs jenkins          # initial admin password is printed here
```

(For pipelines that build Docker images, you'd also mount the Docker socket or use a Docker-in-Docker/agent approach; the apt install on a VM is simpler for beginners.)

## 3.4 Option C: Amazon Linux 2023 / RHEL family

```bash
sudo dnf install -y fontconfig java-21-amazon-corretto     # or java-21-openjdk
# add the Jenkins RPM repo (copy the wget/repo + rpm --import lines from jenkins.io: Red Hat section)
sudo dnf install -y jenkins
sudo systemctl enable --now jenkins
```

## 3.5 Ports and security group

```
8080  → Jenkins web UI (open in the EC2 Security Group: restrict to YOUR IP while learning;
        for GitHub webhooks GitHub must reach it → either open 8080 to GitHub's hook IP ranges or use a tunnel/reverse proxy)
22    → SSH (My IP only)
80/443→ if you put Nginx/ALB with HTTPS in front (recommended in real life)
50000 → inbound agents (only if using inbound agents)
```

## 3.6 First-run wizard ⭐

```
1. Open  http://<SERVER_PUBLIC_IP>:8080
2. "Unlock Jenkins" → get the password:
      sudo cat /var/lib/jenkins/secrets/initialAdminPassword          (Docker: docker logs jenkins)
3. Choose "Install suggested plugins" (Git, Pipeline, credentials, etc.) → wait
4. Create the first admin user (username, strong password, email)
5. Instance configuration: Jenkins URL (e.g., http://<IP>:8080/) → Save and Finish → "Start using Jenkins"
```

## 3.7 Useful system commands

```bash
sudo systemctl start|stop|restart jenkins
sudo journalctl -u jenkins -f              # live service logs
ls /var/lib/jenkins                        # JENKINS_HOME: jobs/, plugins/, secrets/, config.xml, workspace/
sudo cat /var/log/jenkins/jenkins.log      # (path varies by install)
```

---

# 4️⃣ JENKINS UI TOUR AND VOCABULARY 🧭

| Where | What |
|---|---|
| **Dashboard** | List of jobs with status icons and weather (build health) |
| **New Item** | Create a job: Freestyle, Pipeline, Multibranch Pipeline, Folder... |
| **Manage Jenkins** | Plugins, Credentials, System config, Tools, Nodes, Security, Script Console, System Log |
| **Job page** | Build Now, Configure, Build History, Console Output, Workspace, Replay (pipeline) |
| **Build status colors** | 🔵/🟢 success, 🔴 failed, 🟡 unstable (tests failed but build ran), ⚪ not built/aborted |
| **Pipeline Syntax** (job → Pipeline Syntax) | **Snippet Generator**: pick a step, fill the form, get ready-to-paste Groovy ✅ |
| **Global Tool Configuration (Tools)** | JDK, Git, Maven, Node installations |

Build results: **SUCCESS**, **UNSTABLE** (e.g., test failures with `junit`), **FAILURE**, **ABORTED**, **NOT_BUILT**.

---

# 5️⃣ JOB TYPES ⭐

| Type | Description | Use |
|---|---|---|
| **Freestyle project** | Click-configured steps in the UI (SCM, triggers, build steps, post-build actions) | Quick demos, simple tasks; **not version-controlled** |
| **Pipeline** | Pipeline defined as code (Jenkinsfile in repo or pasted in UI) | **Recommended standard** |
| **Multibranch Pipeline** | Scans a repo; auto-creates a pipeline job per branch/PR that contains a Jenkinsfile | Feature-branch/PR workflows |
| **Organization Folder** | Scans an entire GitHub org/Bitbucket project for repos with Jenkinsfiles | Many repos |
| **Folder** | Groups jobs and can scope credentials/permissions | Organization |
| **Maven project, external job, etc.** | Legacy/specialized | — |

## 5.1 Freestyle vs Pipeline ⭐⭐ (viva favourite)

| Aspect | Freestyle | Pipeline |
|---|---|---|
| Defined in | Web UI forms | Code (`Jenkinsfile`, Groovy-based DSL) |
| Version control | ❌ (config lives in Jenkins) | ✅ (in Git with the app) |
| Complex flows (parallel, conditions, loops) | Limited | ✅ Rich |
| Survives controller restarts mid-build | No | ✅ (durable pipelines) |
| Reusability | Copy/clone jobs | Shared libraries, templates |
| Code review of pipeline changes | ❌ | ✅ via PRs |
| Visualization | Basic | Stage View, Blue Ocean/Pipeline Graph |
| Best for | Small one-off jobs | All real CI/CD |

---

# 6️⃣ DECLARATIVE PIPELINE SYNTAX ⭐⭐⭐

Two syntaxes exist: **Declarative** (structured, recommended, beginner-friendly) and **Scripted** (full Groovy, flexible). Start with Declarative.

## 6.1 Skeleton

```groovy
pipeline {
    agent any                       // where to run

    environment { APP_NAME = 'myapp' }       // env variables
    options { timeout(time: 15, unit: 'MINUTES'); buildDiscarder(logRotator(numToKeepStr: '10')) }
    triggers { pollSCM('H/5 * * * *') }      // optional
    parameters { string(name: 'ENV', defaultValue: 'staging', description: 'Target env') }

    stages {
        stage('Build') {
            steps {
                echo "Building ${APP_NAME}"
                sh 'echo hello from the shell'
            }
        }
        stage('Test') {
            steps { sh 'echo running tests' }
        }
    }

    post {                           // runs after stages
        always   { echo 'Always runs' }
        success  { echo 'Build succeeded' }
        failure  { echo 'Build failed' }
    }
}
```

## 6.2 Directives and sections explained

| Section | Purpose |
|---|---|
| `pipeline { }` | Required top-level wrapper |
| `agent` | Where to run: `any`, `none`, `label 'linux'`, `docker { image '...' }`, `dockerfile true`, `kubernetes {...}` (per pipeline or per stage) |
| `stages { stage('X') { steps { ... } } }` | **Required.** Ordered stages, each with `steps` |
| `steps` | Actual commands: `sh`, `bat`, `echo`, `git`, `checkout scm`, `junit`, `archiveArtifacts`, `input`, `sleep`, `retry`, `timeout`... |
| `environment { }` | Variables; can pull secrets via `credentials('id')` |
| `options { }` | `timeout`, `retry`, `timestamps()`, `buildDiscarder`, `disableConcurrentBuilds()`, `skipDefaultCheckout()` |
| `parameters { }` | Build-time inputs: `string`, `choice`, `booleanParam`, `password` |
| `triggers { }` | `cron('H 2 * * *')`, `pollSCM('H/5 * * * *')`, `upstream(...)`; webhooks configured separately (Section 10) |
| `tools { }` | Use tools configured in Global Tool Configuration (jdk, maven, nodejs) |
| `when { }` | Conditional stage: `branch 'main'`, `expression { }`, `environment`, `not`, `allOf`, `anyOf`, `changeRequest()`, `tag` |
| `post { }` | Conditions: `always`, `success`, `failure`, `unstable`, `changed`, `fixed`, `regression`, `aborted`, `cleanup` |
| `parallel { }` | Run stages/branches simultaneously |
| `input { }` | Pause for human approval (Continuous Delivery gate) |
| `matrix { }` | Run the same stage across combinations (e.g., Python 3.11/3.12 × OS) |

## 6.3 `when`, `parallel`, `input` example

```groovy
pipeline {
    agent any
    stages {
        stage('Checks') {
            parallel {
                stage('Lint')   { steps { sh 'echo flake8 .' } }
                stage('Unit')   { steps { sh 'echo pytest' } }
                stage('Scan')   { steps { sh 'echo trivy fs .' } }
            }
        }
        stage('Deploy to Staging') {
            when { branch 'main' }                       // only on main
            steps { echo 'Deploying to staging' }
        }
        stage('Approve Production') {
            when { branch 'main' }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: 'Deploy to PRODUCTION?', ok: 'Yes, deploy', submitter: 'admin,release-managers'
                }
            }
        }
        stage('Deploy to Production') {
            when { branch 'main' }
            steps { echo 'Deploying to production' }
        }
    }
}
```

`input` makes it **Continuous Delivery**; removing the approval stage gives **Continuous Deployment**.

## 6.4 Parameters and conditions

```groovy
pipeline {
    agent any
    parameters {
        choice(name: 'ENV', choices: ['dev', 'staging', 'prod'], description: 'Environment')
        booleanParam(name: 'RUN_TESTS', defaultValue: true, description: 'Run tests?')
    }
    stages {
        stage('Test') {
            when { expression { return params.RUN_TESTS } }
            steps { echo "Testing for ${params.ENV}" }
        }
    }
}
```

## 6.5 Post actions with reports ✅

```groovy
post {
    always {
        junit allowEmptyResults: true, testResults: 'reports/*.xml'      // publish test results (trend graph)
        archiveArtifacts artifacts: 'dist/**', fingerprint: true, allowEmptyArchive: true
        cleanWs()                                                        // clean workspace (needs Workspace Cleanup plugin)
    }
    failure { echo "Pipeline FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}" }
}
```

## 6.6 Matrix example

```groovy
stage('Test matrix') {
    matrix {
        axes { axis { name 'PY'; values '3.11', '3.12' } }
        stages { stage('Run') { steps { echo "Testing on Python ${PY}" } } }
    }
}
```

## 6.7 Scripted Pipeline (for comparison)

```groovy
node('linux') {
    stage('Checkout') { checkout scm }
    stage('Build')    { sh 'make build' }
    try {
        stage('Test') { sh 'make test' }
    } catch (e) {
        currentBuild.result = 'FAILURE'
        throw e
    } finally {
        junit 'reports/*.xml'
    }
}
```

| | Declarative | Scripted |
|---|---|---|
| Structure | Strict `pipeline { }` | Free-form Groovy in `node { }` |
| Learning curve | Easier | Harder |
| Flexibility | Moderate (use `script { }` block for Groovy) | Maximum |
| Validation | Syntax checked up front | Errors at runtime |
| Recommendation | **Default choice** | Advanced needs |

`script { }` block inside a declarative step lets you use Groovy logic when needed.

---

# 7️⃣ A COMPLETE PYTHON CI JENKINSFILE 🐍 ⭐⭐

Project (from Part 3): `app.py`, `test_app.py`, `requirements.txt`, `Jenkinsfile`.

`requirements.txt`:

```
flask==3.0.3
pytest==8.3.3
pytest-cov==5.0.0
flake8==7.1.1
```

`Jenkinsfile`:

```groovy
pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 15, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    environment {
        APP_NAME = 'flask-cicd-demo'
        VENV     = 'venv'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm                         // gets the repo/branch that triggered the build
                sh 'git log -1 --pretty=format:"%h %an %s"'
            }
        }

        stage('Setup') {
            steps {
                sh '''
                    python3 -m venv ${VENV}
                    . ${VENV}/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Lint') {
            steps {
                sh '''
                    . ${VENV}/bin/activate
                    flake8 app.py test_app.py --max-line-length=100
                '''
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                    mkdir -p reports
                    . ${VENV}/bin/activate
                    pytest --junitxml=reports/pytest.xml --cov=app --cov-report=xml:reports/coverage.xml -v
                '''
            }
            post {
                always { junit 'reports/pytest.xml' }
            }
        }
    }

    post {
        success { echo "✅ ${APP_NAME} build #${BUILD_NUMBER} passed" }
        failure { echo "❌ ${APP_NAME} build #${BUILD_NUMBER} failed" }
        always  { archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true }
    }
}
```

⚠️ **Shell gotcha:** Jenkins `sh` runs `/bin/sh` (often `dash`), where `source venv/bin/activate` fails. Use the dot command: `. venv/bin/activate`.

---

# 8️⃣ A DOCKER BUILD, PUSH AND DEPLOY JENKINSFILE 🐳 ⭐⭐ (LINKS TO UNIT I)

```groovy
pipeline {
    agent any

    environment {
        IMAGE_NAME  = 'yourdockerhubuser/flask-cicd-demo'
        IMAGE_TAG   = "${env.BUILD_NUMBER}"
        REGISTRY_CR = credentials('dockerhub-creds')     // Username with password → sets REGISTRY_CR_USR / REGISTRY_CR_PSW
    }

    stages {
        stage('Checkout') { steps { checkout scm } }

        stage('Test') {
            steps {
                sh '''
                    python3 -m venv venv && . venv/bin/activate
                    pip install -r requirements.txt
                    pytest -q
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest .'
            }
        }

        stage('Docker Push') {
            when { branch 'main' }
            steps {
                sh '''
                    echo "$REGISTRY_CR_PSW" | docker login -u "$REGISTRY_CR_USR" --password-stdin
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Deploy') {
            when { branch 'main' }
            steps {
                sh '''
                    docker rm -f flask-app || true
                    docker run -d --name flask-app --restart unless-stopped -p 80:5000 ${IMAGE_NAME}:${IMAGE_TAG}
                    sleep 5
                    curl -f http://localhost:80/health
                '''
            }
        }
    }

    post {
        always { sh 'docker logout || true' }
    }
}
```

Requirements: Docker installed on the Jenkins machine and **the `jenkins` user in the `docker` group** (`sudo usermod -aG docker jenkins && sudo systemctl restart jenkins`). Security warning: access to Docker = effectively root on that host; fine for a lab, harden in production. (`when { branch 'main' }` only works in **Multibranch** jobs; in a plain Pipeline job use `when { expression { env.GIT_BRANCH == 'origin/main' } }` or drop the condition.)

---

# 9️⃣ BUILT-IN ENVIRONMENT VARIABLES 🌍

| Variable | Meaning |
|---|---|
| `BUILD_NUMBER` | Sequential build number |
| `BUILD_ID`, `BUILD_URL` | ID and URL of the build |
| `JOB_NAME` | Job's name |
| `WORKSPACE` | Absolute path of the workspace |
| `JENKINS_URL` | Jenkins root URL |
| `NODE_NAME` | Node running the build |
| `GIT_COMMIT`, `GIT_BRANCH`, `GIT_URL` | From the Git plugin |
| `BRANCH_NAME`, `CHANGE_ID`, `CHANGE_TARGET` | Multibranch: branch name and PR info |
| `BUILD_TAG` | `jenkins-JOBNAME-BUILDNUMBER` |

Use in Groovy as `env.BUILD_NUMBER`, `${BUILD_NUMBER}`; in `sh` as `$BUILD_NUMBER`. See the full list at `http://<jenkins>/pipeline-syntax/globals`.

---

# 🔟 CREDENTIALS MANAGEMENT 🔐 ⭐⭐⭐ (NEVER HARD-CODE SECRETS)

**Manage Jenkins → Credentials → System → Global credentials → Add Credentials.**

| Kind | Use |
|---|---|
| **Username with password** | Docker Hub login, DB, Git over HTTPS (use tokens, not passwords) |
| **Secret text** | API tokens (GitHub PAT, Slack) |
| **SSH Username with private key** | Git over SSH, deploy to servers |
| **Secret file** | kubeconfig, certificates |
| **Certificate** | PKCS#12 |
| **AWS credentials** (plugin) | AWS access keys (prefer IAM roles on EC2) |

Each credential has an **ID** (e.g., `dockerhub-creds`) you reference in pipelines.

```groovy
// Option 1: environment helper (Username/password → NAME_USR and NAME_PSW)
environment { DOCKER = credentials('dockerhub-creds') }
steps { sh 'echo "$DOCKER_PSW" | docker login -u "$DOCKER_USR" --password-stdin' }

// Option 2: withCredentials (scoped to a block)
withCredentials([string(credentialsId: 'github-token', variable: 'GH_TOKEN')]) {
    sh 'curl -H "Authorization: Bearer $GH_TOKEN" https://api.github.com/user'
}

withCredentials([sshUserPrivateKey(credentialsId: 'deploy-key', keyFileVariable: 'KEY', usernameVariable: 'USER')]) {
    sh 'ssh -i $KEY -o StrictHostKeyChecking=no $USER@10.0.1.20 "docker ps"'
}
```

Rules: Jenkins **masks** secret values in console logs (but don't echo them on purpose or write them to files); use **single quotes** around `sh` strings with secrets (`sh '...$PSW...'`), because Groovy double-quote interpolation can leak secrets; scope credentials to folders; rotate regularly; prefer short-lived tokens/IAM roles (Unit V).

---

# 1️⃣1️⃣ TRIGGERS: HOW BUILDS START ⭐⭐

| Trigger | How | Pros/Cons |
|---|---|---|
| **GitHub webhook** ("GitHub hook trigger for GITScm polling") | GitHub POSTs to `http://JENKINS/github-webhook/` on push/PR | **Instant**, efficient ✅ (needs Jenkins reachable from GitHub) |
| **Poll SCM** | Jenkins checks the repo on a schedule (`H/2 * * * *`) | Works behind firewalls; delay + load |
| **Build periodically** (cron) | `H 2 * * *` nightly | Scheduled jobs |
| **Upstream/downstream** | Run after another job finishes | Chained jobs |
| **Remote trigger** (token) | `curl http://JENKINS/job/NAME/build?token=XYZ` | Scripts/integrations |
| **Generic Webhook Trigger plugin** | Any JSON webhook | Flexible |
| **Manual / Parameterized** | "Build with Parameters" | On-demand deploys |

### Cron syntax in Jenkins (five fields)

```
MINUTE HOUR DOM MONTH DOW
H/5 * * * *       → every ~5 minutes (H = hashed offset to spread load)
H 2 * * 1-5       → once between 02:00-02:59 on weekdays
```

---

# 1️⃣2️⃣ GITHUB INTEGRATION AND WEBHOOK FLOW ⭐⭐⭐ (PRACTICAL 6 CORE)

## 12.1 The flow

```
1. Developer: git push to GitHub
2. GitHub sends HTTP POST (JSON payload) → http://<JENKINS_PUBLIC_IP>:8080/github-webhook/
3. Jenkins GitHub plugin receives it → finds jobs for that repo with the webhook trigger enabled
4. Jenkins schedules a build → agent checks out the commit → runs the Jenkinsfile
5. Jenkins reports result (and optionally sets commit status ✅/❌ on GitHub)
```

## 12.2 Setup checklist

| Step | Where |
|---|---|
| Plugins: **Git**, **GitHub**, **GitHub Branch Source** (for multibranch), **Pipeline** | Manage Jenkins → Plugins (suggested plugins already include most) |
| Create a **Personal Access Token** (or deploy key) for private repos | GitHub → Settings → Developer settings → Fine-grained PAT (repo contents read; add commit status if needed) |
| Add it as Jenkins **credential** | Manage Jenkins → Credentials |
| Create **Pipeline job** → "Pipeline script from SCM" → Git → repo URL → credentials → branch → Script Path `Jenkinsfile` | Job config |
| Tick **"GitHub hook trigger for GITScm polling"** | Job config → Build Triggers |
| GitHub repo → **Settings → Webhooks → Add webhook**: Payload URL `http://<IP>:8080/github-webhook/`, Content type `application/json`, events: *Just the push event* | GitHub |
| Open port 8080 in Security Group; Jenkins URL must be reachable from the internet | AWS |
| Test: push a commit → build starts within seconds; GitHub webhook "Recent Deliveries" shows **200** | GitHub |

## 12.3 Local Jenkins behind NAT? Use a tunnel

GitHub can't reach `localhost:8080`. Use **ngrok** or **Cloudflare Tunnel**: `ngrok http 8080` → use the `https://xxxx.ngrok-free.app/github-webhook/` URL as the Payload URL (set Jenkins URL accordingly). Fine for labs; free URLs change on restart.

## 12.4 Webhook troubleshooting

| Symptom (GitHub "Recent Deliveries") | Cause | Fix |
|---|---|---|
| Timeout / couldn't connect | Port 8080 closed / wrong IP / Jenkins down | Security Group, public IP, `systemctl status jenkins` |
| 403 | CSRF/auth blocking (anonymous disabled and request not recognized) | Use correct endpoint `/github-webhook/` (trailing slash), keep GitHub plugin installed; or use "Generic Webhook Trigger" with token |
| 302/404 | Wrong URL | `http://IP:8080/github-webhook/` exactly |
| 200 but no build | Trigger not enabled on job / repo URL mismatch / branch filter | Tick hook trigger; check repo URL and branch spec (`*/main`) |
| Build starts but checkout fails | Missing/incorrect credentials | Add credential, check repo access |

---

# 1️⃣3️⃣ MULTIBRANCH PIPELINES AND PULL REQUESTS 🌿

**New Item → Multibranch Pipeline** → Branch Sources: **GitHub** → repo + credentials → discover branches and PRs → scan triggers (periodic + webhook).

```
Jenkins scans repo → finds branches containing a Jenkinsfile → creates one job per branch/PR automatically
   feature/wishlist  → build runs tests only
   PR #12            → build runs on merge-preview; reports status to GitHub PR
   main              → build, test, push image, deploy (use `when { branch 'main' }`)
Branch deleted → job disappears
```

Great for GitHub Flow / trunk-based workflows (Part 4). Branch protection in GitHub can **require Jenkins' status check** to pass before merging.

---

# 1️⃣4️⃣ AGENTS AND DISTRIBUTED BUILDS 🖥️

## 14.1 Why agents?

```
Isolation (don't run untrusted builds on the controller)      Scale (parallel builds)
Different OS/tools (Windows, macOS, GPU)                        Security (separate credentials/networks)
```

## 14.2 Adding an SSH agent (steps)

```
1. Prepare agent machine: install Java (same major as controller), git, tools; create user 'jenkins'; add controller's public SSH key
2. Manage Jenkins → Nodes → New Node → name, "Permanent Agent"
3. Remote root directory: /home/jenkins/agent ; Labels: linux docker ; Usage: only build jobs with matching labels
4. Launch method: "Launch agents via SSH" → host, credentials (SSH key) → Save → status "online"
5. In Jenkinsfile: agent { label 'docker' }
```

## 14.3 Docker agents (clean, reproducible build environment) ⭐

```groovy
pipeline {
    agent { docker { image 'python:3.12-slim' } }       // build runs INSIDE this container (needs Docker Pipeline plugin + Docker on node)
    stages {
        stage('Test') {
            steps { sh 'pip install -r requirements.txt && pytest -q' }
        }
    }
}
```

Benefit: no need to install Python/Node/Java on the Jenkins machine; each build gets a fresh, versioned toolchain.

## 14.4 Kubernetes dynamic agents

With the Kubernetes plugin, each build launches a **pod** as an agent and deletes it afterward: elastic and cost-efficient (cloud-native Jenkins).

---

# 1️⃣5️⃣ REPORTS, ARTIFACTS AND NOTIFICATIONS 📬

| Feature | Step/Plugin |
|---|---|
| Test results and trend graph | `junit 'reports/*.xml'` |
| Coverage report | Coverage plugin / Cobertura / `recordCoverage` |
| Store build outputs | `archiveArtifacts artifacts: 'dist/*'` |
| Pass files between stages/agents | `stash name:'build', includes:'dist/**'` → `unstash 'build'` |
| Email | `mail to: 'team@example.com', subject: "Build ${env.BUILD_NUMBER}", body: "..."` (configure SMTP in system settings) or Email Extension plugin |
| Slack/Teams | Slack Notification plugin |
| Commit status on GitHub | GitHub plugin / Checks API plugin |
| Static analysis gates | SonarQube Scanner + `waitForQualityGate()` (Unit V link) |
| Dashboards | Stage View, Pipeline Graph View, Blue Ocean (maintenance mode: no new features; many teams use the newer Pipeline Graph View) |

---

# 1️⃣6️⃣ SHARED LIBRARIES, JCasC AND PLUGINS 🧩

## 16.1 Shared libraries (reuse pipeline code)

```
repo: jenkins-shared-lib
  └── vars/
        └── pythonCi.groovy      ← global function used by many projects

// pythonCi.groovy
def call(Map cfg = [:]) {
    pipeline {
        agent any
        stages {
            stage('Test') { steps { sh "pytest ${cfg.testPath ?: '.'}" } }
        }
    }
}

// Jenkinsfile in any project
@Library('jenkins-shared-lib') _
pythonCi(testPath: 'tests')
```

Teams share standard pipelines (security scans, deploy steps) instead of copy-paste.

## 16.2 Jenkins Configuration as Code (JCasC)

YAML file describing Jenkins itself (users, security, credentials references, nodes): reproducible Jenkins, ties to Unit II (IaC!).

```yaml
jenkins:
  systemMessage: "CI server managed by JCasC"
  numExecutors: 0
  securityRealm:
    local:
      allowsSignup: false
      users:
        - id: "admin"
          password: "${ADMIN_PASSWORD}"
```

## 16.3 Plugin management tips

```
Install only what you need (every plugin = maintenance + attack surface)
Keep plugins and Jenkins core updated (security advisories)
Test updates on a staging Jenkins; pin versions with plugins.txt (jenkins-plugin-cli) for Docker images
Common plugins: Git, GitHub, GitHub Branch Source, Pipeline, Credentials Binding, Docker Pipeline, Pipeline Stage View,
                JUnit, Email Extension, Slack Notification, SonarQube Scanner, Workspace Cleanup, Role-based Authorization, Blue Ocean (legacy)
```

---

# 1️⃣7️⃣ SECURITY HARDENING ✅ (BRIDGE TO UNIT V)

| Area | Recommendation |
|---|---|
| **Access** | Disable anonymous access; use **Matrix/Role-based authorization**; integrate with LDAP/SSO; least privilege |
| **Transport** | Put Jenkins behind a reverse proxy (Nginx/ALB) with **HTTPS**; don't expose 8080 publicly long-term |
| **Controller** | Set controller executors to **0**; build only on agents |
| **Credentials** | Use the credential store; no secrets in Jenkinsfiles/repos; rotate |
| **Plugins/Core** | Update regularly; follow Jenkins security advisories |
| **CSRF protection** | Keep enabled (default) |
| **Script Console** | Admin only: it can run arbitrary Groovy on the controller |
| **Agent security** | Agent-to-controller access controls on by default; isolate agents |
| **Pipelines** | Restrict who can edit Jenkinsfiles; review PRs that change pipelines; script security sandbox for Groovy |
| **Network** | Security Groups: restrict 8080/22; webhooks via allowed IP ranges or secret tokens |
| **Audit** | Audit Trail plugin; log shipping |
| **Docker access** | `docker` group membership = root-equivalent; consider rootless/DinD/agents |

## 17.1 Backup and restore

```
Backup:   JENKINS_HOME (/var/lib/jenkins) – especially jobs/, config.xml, credentials.xml, secrets/, plugins/ list
Tools:    ThinBackup plugin, snapshots (EBS), rsync/S3 sync, JCasC + Jenkinsfiles in Git (so Jenkins is rebuildable)
Test restores regularly.
```

---

# 1️⃣8️⃣ PERFORMANCE AND SCALING TIPS ⚡

```
Run builds on agents, not the controller          Cache dependencies (pip/npm/Maven caches, Docker layer cache)
Parallelize independent stages                    Use lightweight checkout / shallow clones for big repos
Clean old builds (buildDiscarder) and workspaces  Give the controller enough RAM (JVM heap: JAVA_OPTS="-Xmx2g")
Use ephemeral Docker/K8s agents                   Avoid heavy polling: use webhooks
Keep pipelines fast: fail early, fast tests first Monitor Jenkins (Prometheus plugin → Grafana, Unit IV)
```

---

# 1️⃣9️⃣ TROUBLESHOOTING TABLE 🛠️

| Symptom | Cause | Fix |
|---|---|---|
| `jenkins: failed to find a valid Java installation` | Java missing/wrong version | Install Java 17/21 **before** Jenkins; `java -version` |
| Can't open `http://IP:8080` | Security Group / service down | Open 8080; `sudo systemctl status jenkins` |
| `apt update` GPG/signature error for Jenkins repo | Rotated/expired signing key | Download the current key from jenkins.io install page |
| "Unlock Jenkins" password not found | Wrong path | `sudo cat /var/lib/jenkins/secrets/initialAdminPassword` |
| Plugin install fails/offline | No internet / update center | Check outbound access; retry; proxy settings |
| `git: command not found` / `python3: not found` / `venv` missing | Tools not on the agent | `sudo apt install git python3 python3-venv python3-pip` |
| `source: not found` in `sh` | Using `source` under dash | Use `. venv/bin/activate` |
| `docker: permission denied ... /var/run/docker.sock` | `jenkins` user not in docker group | `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins` |
| `No such property: X for class` / Groovy errors | Syntax/scope | Use Pipeline Syntax snippet generator; check quotes |
| `WorkflowScript: ... Expected a step` | Wrong declarative structure | Steps must be in `steps { }` |
| Build stuck "Waiting for next available executor" | No free executor / label mismatch | Add executors/agents; fix label |
| `ERROR: Couldn't find any revision to build` | Wrong branch spec (`*/master` vs `*/main`) | Set branch to `*/main` |
| Webhook delivered, no build | Trigger not enabled/URL mismatch | See Section 12.4 |
| `credentials 'x' not found` | Wrong ID/scope | Check credential ID and folder scope |
| Out of memory / very slow | Low RAM (1 GB instance) | Use ≥2-4 GB; add swap; tune heap |
| Disk full | Old builds/workspaces/Docker images | `buildDiscarder`, `cleanWs`, `docker system prune` |
| Pipeline can't push image: `denied` | Not logged in / wrong repo name | Docker credentials, `user/repo:tag` |

---

# 2️⃣0️⃣ VIVA QUESTIONS (PART 5) 🎯

**Q1. What is Jenkins?** An open-source Java-based automation server used to implement CI/CD pipelines.

**Q2. Explain Jenkins architecture.** A controller manages configuration, scheduling, plugins and UI; agents with executors run builds; `JENKINS_HOME` stores state; plugins extend functionality.

**Q3. What is an executor?** A slot on a node that runs one build at a time.

**Q4. Freestyle vs Pipeline job?** Freestyle is UI-configured and not versioned; Pipeline is code in a Jenkinsfile, versioned and supports complex flows.

**Q5. Declarative vs Scripted pipeline?** Declarative has a strict structured syntax (`pipeline { }`) and is easier; Scripted is free-form Groovy in `node { }` with more flexibility.

**Q6. What is a Jenkinsfile?** A text file in the repository defining the pipeline stages and steps.

**Q7. What do `agent`, `stages`, `steps`, `post` do?** Where to run; ordered work groups; commands; actions after completion based on result.

**Q8. How do you store secrets in Jenkins?** In the Credentials store, referenced by ID with `credentials()` or `withCredentials`, masked in logs.

**Q9. How does the GitHub webhook trigger Jenkins?** GitHub POSTs to `/github-webhook/` on push; Jenkins' GitHub plugin triggers matching jobs with the hook trigger enabled.

**Q10. Webhook vs Poll SCM?** Webhook is event-driven and instant; Poll SCM checks periodically and adds delay and load.

**Q11. What is a Multibranch Pipeline?** A job that auto-creates pipelines for each branch/PR containing a Jenkinsfile.

**Q12. Why shouldn't builds run on the controller?** Security and stability; use agents with controller executors set to 0.

**Q13. What does `junit` step do?** Publishes JUnit XML test results, giving pass/fail counts and trends, and marks build UNSTABLE on failures.

**Q14. How do you add `input` approval, and which delivery model does it represent?** `input message:` step pauses for human approval, representing Continuous Delivery.

**Q15. What is a shared library?** Reusable pipeline code in a separate repo used across Jenkinsfiles with `@Library`.

**Q16. How do you secure Jenkins?** Disable anonymous, role-based access, HTTPS, update plugins, credentials store, no controller builds, restrict script console.

---

# 🧠 PART 5 MASTER SHEET

```
JENKINS: open-source Java automation server ; Hudson fork (2011) ; LTS every ~12 weeks ; plugins (1800+)
ARCHITECTURE: Controller (UI, config, scheduling, plugins) + Agents/Nodes (executors, labels, workspace) ; JENKINS_HOME ; build queue
INSTALL: Java 17/21 FIRST → add Jenkins repo (current key from jenkins.io) → apt install jenkins → :8080 → initialAdminPassword → suggested plugins → admin user
JOBS: Freestyle (UI) | Pipeline (Jenkinsfile) | Multibranch (per branch/PR) | Folder/Org folder
DECLARATIVE: pipeline{ agent, environment, options, parameters, triggers, tools, stages{ stage{ when, steps, post } }, post }
  post: always/success/failure/unstable/changed/fixed/cleanup ; parallel ; input (approval) ; matrix ; script{}
STEPS: checkout scm, sh, echo, junit, archiveArtifacts, withCredentials, input, timeout, retry, stash/unstash, cleanWs
CREDENTIALS: store → ID → credentials('id') / withCredentials ; masked ; single quotes for secrets ; rotate
TRIGGERS: GitHub webhook (/github-webhook/, JSON) | pollSCM | cron (H) | upstream | remote token | parameters
ENV: BUILD_NUMBER, JOB_NAME, WORKSPACE, GIT_COMMIT, BRANCH_NAME, BUILD_URL
AGENTS: SSH/inbound/Docker/Kubernetes ; agent{ docker{ image } } ; controller executors = 0
EXTRAS: shared libraries (@Library), JCasC, Blue Ocean (legacy), SonarQube gate
SECURITY: no anonymous, RBAC, HTTPS, update plugins, credentials store, docker group = root-equivalent, backup JENKINS_HOME
GOTCHAS: Java before Jenkins ; `. venv/bin/activate` not `source` ; jenkins user in docker group ; branch */main
```

---

## 🫡 PART 5 STATUS

You understand Jenkins architecture, installation, Declarative Pipeline syntax, credentials, triggers and the GitHub webhook flow, agents, security and troubleshooting, ready for Practical 6.

**STOP here.** Next up:

> **PART 6 — Product Strategy, Distributed Teams, A/B Testing and Technical Debt Management 🎯**
