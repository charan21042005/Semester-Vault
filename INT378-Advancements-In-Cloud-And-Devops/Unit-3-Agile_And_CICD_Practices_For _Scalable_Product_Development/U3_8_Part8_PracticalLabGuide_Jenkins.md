# 🧪 UNIT III — PART 8: PRACTICAL LAB GUIDE (JENKINS + GITHUB WEBHOOK + AUTOMATED CI/CD PIPELINE)

> **Covered in this document:** lab goal and architecture → prerequisites and cost rules → **Step 1** create the GitHub repository and the full sample application (Flask app, tests, requirements, Dockerfile, `.gitignore`, `.dockerignore`) → **Step 2** launch an Ubuntu EC2 server and security group (including GitHub webhook IP ranges) → **Step 3** install Java, Jenkins, Git, Python and Docker, and give Jenkins Docker access → **Step 4** first-run setup, plugins, Jenkins URL → **Step 5** create the Pipeline job (Pipeline script from SCM, webhook trigger) → **Step 6** the complete `Jenkinsfile` (checkout, setup, lint, test with reports, Docker build, deploy, smoke test, automatic rollback) explained line by line → **Step 7** first manual build and what to inspect → **Step 8** add the GitHub webhook → **Step 9** prove automation: push a change and watch auto build + auto deploy → **Step 10** prove "deploy only on success": break a test, break the health check (auto-rollback) → evidence checklist → bonus variations (Multibranch/PR builds, Docker Hub push, manual approval gate = Continuous Delivery, SSH deploy to a second server, private repo, Poll SCM fallback, ngrok for local Jenkins) → troubleshooting flow and table → cleanup → report template → practical viva → quick checklist.

---

Absolutely mate. 😤🔥 This is **Practical 6**. Your syllabus line:

> **P6:** *Deploy Jenkins, integrate it with GitHub using webhooks, configure an automated CI/CD pipeline, execute build and testing stages, and automatically deploy the application upon successful pipeline execution.*

We'll build exactly that, end to end, and then **prove** each requirement.

---

# 0️⃣ GOAL, ARCHITECTURE AND RULES

## 0.1 What you will build

```
 Developer laptop
      │  git push
      ▼
 ┌──────────┐   webhook (POST /github-webhook/)    ┌───────────────────────────── EC2 (Ubuntu) ────────────────────────────┐
 │  GitHub  │ ───────────────────────────────────▶ │  JENKINS :8080                                                          │
 │   repo   │                                      │   1 Checkout → 2 Setup → 3 Lint → 4 Unit tests (+reports)                │
 └──────────┘                                      │   5 Docker build → 6 Deploy container (port 80) → 7 Smoke test           │
                                                   │   on smoke-test failure → automatic rollback to previous container       │
                                                   │  DOCKER: container "flask-cicd-demo"  ← http://<EC2_IP>/  (users)        │
                                                   └─────────────────────────────────────────────────────────────────────────┘
```

| Requirement from syllabus | How we satisfy it |
|---|---|
| Deploy Jenkins | Install Jenkins on an EC2 Ubuntu server |
| Integrate with GitHub using webhooks | GitHub webhook → `/github-webhook/`; job trigger "GitHub hook trigger for GITScm polling" |
| Configure an automated CI/CD pipeline | `Jenkinsfile` (pipeline as code) in the repo |
| Execute build and testing stages | Setup/Lint/pytest with JUnit and coverage reports; Docker build |
| Automatically deploy upon success | Deploy + Smoke test stages run only if earlier stages pass |

## 0.2 Prerequisites

```
☐ AWS account (budget alert set) and permission to launch EC2     ☐ GitHub account
☐ A laptop with Git, a code editor, and (optionally) Python 3.12+ and Docker to test locally
☐ Your public IP: https://checkip.amazonaws.com
☐ A strong Jenkins admin password you'll remember
```

## 0.3 Cost and safety rules 🚨

```
• Use t3.small (2 GB) or t3.medium (4 GB). t3.micro (1 GB) is usually too small for Jenkins + Docker builds.
• EC2 and public IPv4 addresses cost money while running. STOP or TERMINATE when done (Section 14).
• Don't leave Jenkins open to the world: restrict port 8080 to your IP + GitHub's webhook IP ranges (Step 2).
• Never commit secrets. Use a strong admin password. Delete tokens after the lab.
• Free Tier terms for new AWS accounts changed in 2025: check your Billing console.
```

---

# 1️⃣ STEP 1: CREATE THE GITHUB REPOSITORY AND THE APPLICATION

## 1.1 Create the repo

GitHub → **New repository** → name `flask-cicd-demo` → **Public** (simplest; private variant in Bonus) → add nothing else → Create.

## 1.2 Project structure

```
flask-cicd-demo/
├── app.py
├── test_app.py
├── requirements.txt            ← runtime dependencies (goes into the Docker image)
├── requirements-dev.txt        ← test/lint tools (used by Jenkins)
├── Dockerfile
├── Jenkinsfile
├── .gitignore
├── .dockerignore
└── README.md
```

## 1.3 `app.py`

```python
import os
import socket

from flask import Flask, jsonify

app = Flask(__name__)


@app.route("/")
def home():
    version = os.getenv("APP_VERSION", "dev")
    return (
        "<h1>Hello from Jenkins CI/CD 🚀</h1>"
        f"<p>Version (build number): {version}</p>"
        f"<p>Container: {socket.gethostname()}</p>"
    )


@app.route("/health")
def health():
    return jsonify(status="ok")


@app.route("/version")
def version():
    return jsonify(version=os.getenv("APP_VERSION", "dev"))


@app.route("/add/<int:a>/<int:b>")
def add(a, b):
    return jsonify(result=a + b)


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

(`host="0.0.0.0"` is mandatory inside Docker, Unit I Part 6.)

## 1.4 `test_app.py`

```python
import pytest

from app import app


@pytest.fixture
def client():
    app.config["TESTING"] = True
    with app.test_client() as c:
        yield c


def test_home_page(client):
    resp = client.get("/")
    assert resp.status_code == 200
    assert b"Hello from Jenkins CI/CD" in resp.data


def test_health_endpoint(client):
    resp = client.get("/health")
    assert resp.status_code == 200
    assert resp.get_json() == {"status": "ok"}


def test_version_endpoint(client):
    resp = client.get("/version")
    assert resp.status_code == 200
    assert "version" in resp.get_json()


def test_add_endpoint(client):
    assert client.get("/add/2/3").get_json()["result"] == 5
```

## 1.5 `requirements.txt` and `requirements-dev.txt`

`requirements.txt`:

```
flask==3.0.3
```

`requirements-dev.txt`:

```
-r requirements.txt
pytest==8.3.3
pytest-cov==5.0.0
flake8==7.1.1
```

## 1.6 `Dockerfile` (Unit I, upgraded)

```dockerfile
FROM python:3.12-slim

ARG BUILD_VERSION=dev
ENV APP_VERSION=${BUILD_VERSION}

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

# run as a non-root user (container security)
RUN useradd --create-home appuser
USER appuser

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=3s \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:5000/health')" || exit 1

CMD ["python", "app.py"]
```

## 1.7 `.gitignore` and `.dockerignore`

`.gitignore`:

```
venv/
__pycache__/
*.pyc
.pytest_cache/
reports/
.coverage
.prev_image
.env
```

`.dockerignore`:

```
venv
__pycache__
*.pyc
.git
reports
Jenkinsfile
test_app.py
requirements-dev.txt
```

## 1.8 Test locally first ✅ (optional but wise)

```bash
python3 -m venv venv && . venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements-dev.txt
flake8 app.py test_app.py --max-line-length=100     # no output = clean
pytest -v                                           # 4 passed
docker build -t flask-cicd-demo:local . && docker run --rm -p 5000:5000 flask-cicd-demo:local
# open http://localhost:5000
```

## 1.9 Push to GitHub (the `Jenkinsfile` comes in Step 6; you can add it now or later)

```bash
git init -b main
git add .
git commit -m "feat: initial Flask app with tests and Dockerfile"
git remote add origin https://github.com/<your-username>/flask-cicd-demo.git
git push -u origin main
```

(GitHub no longer accepts account passwords for pushes over HTTPS: use a **Personal Access Token** as the password, SSH keys, or GitHub CLI `gh auth login`.)

---

# 2️⃣ STEP 2: LAUNCH THE EC2 SERVER AND SECURITY GROUP

## 2.1 Console settings

| Field | Value |
|---|---|
| Name | `jenkins-server` |
| AMI | **Ubuntu Server 24.04 LTS** |
| Instance type | `t3.small` (or `t3.medium`) |
| Key pair | `lab-key` (create if needed; Unit I Part 8) |
| Network | Default VPC, public subnet, **auto-assign public IP: enable** |
| Storage | **20 GiB** gp3 |
| Security group | New: `jenkins-sg` (below) |

## 2.2 Security group `jenkins-sg` (inbound) ⭐

| Type | Port | Source | Purpose |
|---|---|---|---|
| SSH | 22 | **My IP** | Admin |
| Custom TCP | 8080 | **My IP** | You use the Jenkins UI |
| Custom TCP | 8080 | **GitHub webhook IP ranges** (below) | GitHub can reach Jenkins |
| HTTP | 80 | Anywhere (0.0.0.0/0) | The deployed app (public demo) |

### Adding GitHub's webhook IP ranges (IPv4)

GitHub publishes them at `https://api.github.com/meta` (the `hooks` array; the list changes occasionally, so fetch it fresh):

```bash
# run on your laptop with AWS CLI + jq; replace SG_ID with your jenkins-sg ID
SG_ID=sg-0123456789abcdef0
for cidr in $(curl -s https://api.github.com/meta | jq -r '.hooks[]' | grep -v ':'); do
  aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 8080 --cidr "$cidr"
done
```

(IPv6 entries are skipped by `grep -v ':'`. If you hit the security-group rule limit, remove other rules or temporarily allow 8080 from `0.0.0.0/0` **only for the duration of the lab** with a strong password, then close it.)

## 2.3 Connect

```bash
chmod 400 lab-key.pem
ssh -i lab-key.pem ubuntu@<EC2_PUBLIC_IP>
```

📸 Screenshot: running instance + security group rules.

---

# 3️⃣ STEP 3: INSTALL JAVA, JENKINS, GIT, PYTHON AND DOCKER ON THE SERVER

Run on the EC2 instance.

## 3.1 System update and tools

```bash
sudo apt update && sudo apt -y upgrade
sudo apt install -y git curl python3 python3-venv python3-pip fontconfig openjdk-21-jre
java -version          # openjdk 21...
git --version
python3 --version
```

## 3.2 Install Jenkins (LTS) ⚠️ use the current commands from jenkins.io

Jenkins' package-signing key is rotated from time to time. Open **https://www.jenkins.io/doc/book/installing/linux/** → *Debian/Ubuntu* → *Long Term Support release* and copy the three commands. They look like this (the key file name, e.g. `jenkins.io-2026.key`, may differ when you do the lab):

```bash
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2026.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" \
  | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install -y jenkins
sudo systemctl enable --now jenkins
sudo systemctl status jenkins            # active (running)
```

## 3.3 Install Docker and give Jenkins access ⭐

```bash
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker jenkins          # lets the Jenkins service user run docker
sudo usermod -aG docker ubuntu           # (optional) your SSH user
sudo systemctl restart jenkins           # REQUIRED so the new group membership takes effect
docker --version
sudo -u jenkins docker ps                # should print an (empty) table, not "permission denied"
```

⚠️ Security note: membership in the `docker` group is effectively root on the host. Acceptable for a lab; see Unit V for hardening (rootless Docker, separate build agents).

## 3.4 Get the initial admin password

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

📸 Screenshot: `systemctl status jenkins`.

---

# 4️⃣ STEP 4: JENKINS FIRST-RUN SETUP

1. Browser: **`http://<EC2_PUBLIC_IP>:8080`** → **Unlock Jenkins** → paste the password.
2. **Install suggested plugins** (includes Git, Pipeline, GitHub, Credentials Binding, JUnit...) → wait.
3. **Create First Admin User** (strong password) → Save and Continue.
4. **Instance Configuration → Jenkins URL:** `http://<EC2_PUBLIC_IP>:8080/` → Save and Finish → **Start using Jenkins**.
5. Verify plugins: **Manage Jenkins → Plugins → Installed plugins** → search for **Git**, **GitHub**, **Pipeline**, **JUnit**. If missing, install from *Available plugins* and restart.
6. (Good practice) **Manage Jenkins → Nodes → Built-In Node → Configure**: leave executors at 2 for this single-server lab. (In production set it to 0 and use agents, Part 5.)

If your public IP changes after a stop/start, update **Manage Jenkins → System → Jenkins URL** and the GitHub webhook URL (use an Elastic IP to avoid this; note public IPv4 addresses are billed).

📸 Screenshot: Jenkins dashboard.

---

# 5️⃣ STEP 5: CREATE THE PIPELINE JOB

**Dashboard → New Item**

```
Name:  flask-cicd-pipeline      Type: Pipeline      → OK
```

Configure:

| Section | Setting |
|---|---|
| **General** | (optional) ☑ GitHub project → `https://github.com/<you>/flask-cicd-demo/` |
| **Build Triggers** | ☑ **GitHub hook trigger for GITScm polling** |
| **Pipeline → Definition** | **Pipeline script from SCM** |
| SCM | **Git** |
| Repository URL | `https://github.com/<you>/flask-cicd-demo.git` |
| Credentials | None (public repo) |
| Branches to build | `*/main` |
| Script Path | `Jenkinsfile` |
| (Lightweight checkout) | leave ticked |

→ **Save**.

---

# 6️⃣ STEP 6: THE COMPLETE `Jenkinsfile` (EXPLAINED) ⭐⭐⭐

Create `Jenkinsfile` in the repo root:

```groovy
pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        APP_NAME  = 'flask-cicd-demo'
        IMAGE     = 'flask-cicd-demo'
        TAG       = "${env.BUILD_NUMBER}"
        HOST_PORT = '80'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1 --pretty=format:"Commit: %h by %an - %s"'
            }
        }

        stage('Setup Python Env') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements-dev.txt
                '''
            }
        }

        stage('Lint') {
            steps {
                sh '''
                    . venv/bin/activate
                    flake8 app.py test_app.py --max-line-length=100
                '''
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                    . venv/bin/activate
                    mkdir -p reports
                    pytest -v \
                        --junitxml=reports/pytest.xml \
                        --cov=app --cov-report=xml:reports/coverage.xml --cov-report=term-missing \
                        --cov-fail-under=70
                '''
            }
            post {
                always {
                    junit 'reports/pytest.xml'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build --build-arg BUILD_VERSION=${TAG} \
                        -t ${IMAGE}:${TAG} -t ${IMAGE}:latest .
                    docker images ${IMAGE}
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    # remember what is running now (for rollback)
                    PREV_IMAGE=$(docker inspect --format '{{.Config.Image}}' ${APP_NAME} 2>/dev/null || true)
                    echo "Previous image: ${PREV_IMAGE:-none}"
                    echo "${PREV_IMAGE}" > .prev_image

                    # replace the running container with the new version
                    docker rm -f ${APP_NAME} 2>/dev/null || true
                    docker run -d --name ${APP_NAME} --restart unless-stopped \
                        -p ${HOST_PORT}:5000 ${IMAGE}:${TAG}
                '''
            }
        }

        stage('Smoke Test') {
            steps {
                sh '''
                    for i in 1 2 3 4 5 6 7 8 9 10; do
                        if curl -fsS http://localhost:${HOST_PORT}/health; then
                            echo
                            echo "Smoke test passed. Deployed build #${TAG}"
                            curl -fsS http://localhost:${HOST_PORT}/version
                            exit 0
                        fi
                        echo "Waiting for the app... (${i}/10)"
                        sleep 3
                    done
                    echo "Smoke test FAILED"
                    exit 1
                '''
            }
            post {
                failure {
                    echo 'Smoke test failed: rolling back to the previous version'
                    sh '''
                        PREV=$(cat .prev_image 2>/dev/null || true)
                        docker rm -f ${APP_NAME} 2>/dev/null || true
                        if [ -n "$PREV" ]; then
                            docker run -d --name ${APP_NAME} --restart unless-stopped \
                                -p ${HOST_PORT}:5000 "$PREV"
                            echo "Rolled back to $PREV"
                        else
                            echo "No previous version to roll back to"
                        fi
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true
            sh 'docker image prune -f || true'
        }
        success {
            echo "✅ Pipeline succeeded: build #${env.BUILD_NUMBER} is live at port ${HOST_PORT}"
        }
        failure {
            echo "❌ Pipeline failed at build #${env.BUILD_NUMBER}. Check the failing stage in Stage View."
        }
    }
}
```

Commit and push it:

```bash
git add Jenkinsfile
git commit -m "ci: add Jenkins pipeline (lint, test, docker build, deploy, smoke test, rollback)"
git push
```

## 6.1 Line-by-line explanation ✍️ (for viva)

| Part | Explanation |
|---|---|
| `agent any` | Run on any available node (here the built-in node) |
| `options { ... }` | `timestamps` adds time to logs; `timeout` aborts stuck builds; `disableConcurrentBuilds` avoids two deploys colliding; `buildDiscarder` keeps last 10 builds |
| `environment { TAG = "${env.BUILD_NUMBER}" }` | The Jenkins build number becomes the image version tag (traceability) |
| **Checkout** | `checkout scm` fetches the commit that triggered the build; prints last commit |
| **Setup** | Creates a Python virtual environment and installs dev dependencies; uses `.` (dot) because `sh` is dash |
| **Lint** | `flake8` enforces style; non-zero exit fails the stage → pipeline stops |
| **Unit Tests** | pytest produces JUnit XML + coverage; `--cov-fail-under=70` is a **quality gate**; `junit` (in `post always`) publishes results even on failure |
| **Build Docker Image** | Image tagged with build number and `latest`; `BUILD_VERSION` build-arg bakes the version into the app |
| **Deploy** | Saves the currently running image name to `.prev_image`, removes the old container, runs the new one mapping host port 80 → container 5000 |
| **Smoke Test** | Polls `/health` up to 10 times; passes only if the app is really serving; prints version |
| Stage `post { failure }` | **Automatic rollback**: restart the previous image if the smoke test fails |
| Pipeline `post` | Always archives reports and prunes dangling images; success/failure messages |
| Sequential stage flow | A failure in any stage stops later stages: **this is why deployment happens only on success** |

---

# 7️⃣ STEP 7: FIRST MANUAL BUILD AND WHAT TO INSPECT

Jenkins → `flask-cicd-pipeline` → **Build Now**.

Open the build (`#1`) and inspect:

| Where | What to see |
|---|---|
| **Stage View** (job page) | Boxes for each stage turning green; durations |
| **Console Output** | Full logs, git commit line, pytest output, docker build, smoke test "Smoke test passed" |
| **Test Result** (build page) | 4 tests passed; trend graph after more builds |
| **Artifacts** | `reports/pytest.xml`, `coverage.xml` |
| **Workspace** | `/var/lib/jenkins/workspace/flask-cicd-pipeline/` |

Verify the app:

```bash
curl http://localhost/health                 # {"status":"ok"}  (on the EC2)
docker ps                                    # container flask-cicd-demo, PORTS 0.0.0.0:80->5000/tcp
```

Browser: **`http://<EC2_PUBLIC_IP>/`** → "Hello from Jenkins CI/CD 🚀 · Version (build number): 1".

📸 Screenshots: Stage View (all green), console output (smoke test passed), app in browser, test result page.

---

# 8️⃣ STEP 8: ADD THE GITHUB WEBHOOK ⭐⭐⭐

GitHub repo → **Settings → Webhooks → Add webhook**

| Field | Value |
|---|---|
| **Payload URL** | `http://<EC2_PUBLIC_IP>:8080/github-webhook/` (**trailing slash required**) |
| **Content type** | `application/json` |
| **Secret** | (optional in lab) |
| **SSL verification** | Disable warning is shown for http (fine for lab) |
| **Which events?** | **Just the push event** |
| **Active** | ☑ |

→ **Add webhook**. GitHub immediately sends a **ping**.

Verify: Webhooks → your hook → **Recent Deliveries** → a delivery with a green tick and **Response 200**.

📸 Screenshot: webhook page with the green tick.

If the ping fails, use the troubleshooting table in Section 12 (usually Security Group or wrong URL).

---

# 9️⃣ STEP 9: PROVE THE AUTOMATION: PUSH → AUTO BUILD → AUTO DEPLOY 🚀

## 9.1 Make a visible change

Edit `app.py` text:

```python
"<h1>Hello from Jenkins CI/CD 🚀 - Practical 6 by YOUR NAME</h1>"
```

and update `test_app.py` expectation if you changed the checked phrase (the test looks for `b"Hello from Jenkins CI/CD"`, still present).

```bash
git add app.py
git commit -m "feat: personalise home page"
git push
```

## 9.2 Watch (do not click Build Now!)

```
Within seconds: Jenkins dashboard shows a new build #2 started "Started by GitHub push by <you>"
Stage View: Checkout → Setup → Lint → Unit Tests → Build → Deploy → Smoke Test all green
Refresh http://<EC2_PUBLIC_IP>/ → new heading and Version: 2
```

✅ This single experiment demonstrates: **webhook integration + automated build + automated test + automated deployment.**

📸 Screenshots: build #2 "Started by GitHub push", browser showing new version, GitHub webhook delivery for the push (200).

---

# 🔟 STEP 10: PROVE "DEPLOY ONLY ON SUCCESS" AND ROLLBACK 🛡️

## 10.1 Experiment A: a failing test blocks deployment

Break a test on purpose (`test_app.py`):

```python
def test_add_endpoint(client):
    assert client.get("/add/2/3").get_json()["result"] == 6      # wrong on purpose
```

```bash
git commit -am "test: intentionally break a test"
git push
```

Expected:

```
Build #3: Checkout ✅ → Setup ✅ → Lint ✅ → Unit Tests ❌ (red) → Build/Deploy/Smoke test SKIPPED
Test Result: 3 passed, 1 failed ; build FAILED
Browser: app STILL shows Version 2 (the broken change was never deployed) ✔
```

Fix it (`== 5`), commit, push → build #4 green → deployed.

## 10.2 Experiment B: the app fails its health check → automatic rollback

Temporarily break `/health` so tests still pass but the smoke test fails. For example, change the health route to return HTTP 500 while the unit test is relaxed... simplest way to demonstrate: **make the container crash at startup** after tests pass. In `app.py` add at the bottom of the main block:

```python
if __name__ == "__main__":
    raise SystemExit("simulated startup crash")      # tests import the module, so they still pass
    app.run(host="0.0.0.0", port=5000)
```

(Remove it after the demo. Tests pass because pytest imports `app` without running the `__main__` block, but the container executes `python app.py` and exits at once.)

Push. Expected:

```
Stages up to Deploy ✅ (container starts then exits immediately)
Smoke Test ❌ ("Waiting for the app... (1/10)" ... "Smoke test FAILED")
Stage post → "Rolled back to flask-cicd-demo:4" ; build marked FAILED
Browser: site still works on the previous version ✔  (docker ps shows the old image)
```

Then remove the crash line, push → new build deploys normally.

📸 Screenshots: red Unit Tests stage; red Smoke Test with rollback message; browser still working.

## 10.3 What you demonstrated

```
CI:  every push is built and tested automatically
CD:  successful builds are deployed automatically (Continuous Deployment style)
Quality gates: lint + tests + coverage threshold
Safety: failed tests stop deployment; failed smoke test rolls back
Traceability: image tag = Jenkins build number = version shown in the app
```

---

# 1️⃣1️⃣ EVIDENCE CHECKLIST FOR YOUR REPORT 📸

```
☐ EC2 instance + security group (22/8080/80)           ☐ java -version, jenkins service active
☐ Jenkins dashboard / plugins                           ☐ Job configuration (webhook trigger + SCM + Jenkinsfile path)
☐ Jenkinsfile (code screenshot)                         ☐ First manual build: Stage View all green + console output
☐ App running in browser (version 1)                    ☐ GitHub webhook page (green tick, Recent Deliveries 200)
☐ Push → automatic build "Started by GitHub push"       ☐ New version visible in browser (version 2)
☐ Test result page + coverage/artifacts                 ☐ Failed-test build (deployment skipped)
☐ Smoke-test failure + rollback log                     ☐ docker ps / docker images output
☐ Cleanup completed (instance terminated)
```

---

# 1️⃣2️⃣ TROUBLESHOOTING (FLOW + TABLE) 🛠️

## 12.1 "Webhook sent but nothing happens" flow

```
GitHub Recent Deliveries shows green 200?
 ├─ NO (timeout/connection refused) → Security Group allows 8080 from GitHub ranges? Public IP changed? Jenkins running? URL has /github-webhook/ ?
 └─ YES → Job has "GitHub hook trigger for GITScm polling" ticked? Repo URL in the job exactly matches the GitHub repo URL? Branch spec */main?
          → open Jenkins "GitHub Hook Log" (job page) / Manage Jenkins → System Log for errors
```

## 12.2 Table

| Symptom | Cause | Fix |
|---|---|---|
| Can't open `:8080` | SG closed/Jenkins down | Open 8080 for your IP; `sudo systemctl status jenkins` |
| Jenkins won't start / "failed to find a valid Java installation" | Java missing or installed after Jenkins | `sudo apt install openjdk-21-jre`, restart Jenkins |
| `apt update` signature/GPG error for Jenkins repo | Rotated signing key | Use the current key file from jenkins.io instructions |
| `python3: venv not available` / `ensurepip` error | `python3-venv` missing | `sudo apt install python3-venv` |
| `source: not found` | Used `source` in `sh` | Use `. venv/bin/activate` |
| `flake8` fails (E501 line too long, F401 unused import) | Style/unused code | Fix the code or adjust flags |
| `docker: permission denied ... docker.sock` | `jenkins` not in docker group or Jenkins not restarted | `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins` |
| `docker: command not found` in pipeline | Docker not installed | `sudo apt install docker.io` |
| `port is already allocated` (80) | Other process on 80 | `sudo ss -ltnp | grep :80`; stop it or change `HOST_PORT` (and SG) |
| App unreachable in browser but smoke test passes | SG missing port 80 / using https | Open 80; use `http://` |
| `ERROR: Couldn't find any revision to build. Verify the repository and branch configuration` | Branch spec `*/master` vs `*/main`; wrong URL | Set `*/main`; check URL |
| `Jenkinsfile not found` | Script Path wrong / file not pushed | Path `Jenkinsfile`; push the file |
| Webhook 403/404 | Wrong URL (missing trailing slash) / blocked | Use `http://IP:8080/github-webhook/` |
| Build starts only by polling/manual | Hook trigger not ticked | Tick trigger; re-save job |
| Smoke test always fails | App crashes / port mismatch / container name clash | `docker logs flask-cicd-demo`; check `-p 80:5000` |
| `--cov-fail-under` fails | Coverage < 70% | Add tests or lower threshold deliberately |
| Out-of-memory / extremely slow | t3.micro | Use t3.small/t3.medium; add swap |
| Disk full | Old images/workspaces | `docker system prune -af`, `buildDiscarder`, bigger disk |
| Jenkins URL/webhook broke after restart | Public IP changed | Update Jenkins URL + webhook, or use Elastic IP |

---

# 1️⃣3️⃣ BONUS VARIATIONS (EXTRA MARKS) ⭐

## 13.1 Multibranch Pipeline + Pull Request builds

New Item → **Multibranch Pipeline** → Branch Sources: GitHub → repo URL (+ credentials for private) → Behaviours: discover branches, discover PRs → Scan triggers. Webhook still points to `/github-webhook/`. Now every branch/PR gets its own pipeline; only `main` should deploy:

```groovy
stage('Deploy') {
    when { branch 'main' }
    steps { /* same as before */ }
}
```

In GitHub: **Settings → Branches → Branch protection rule** for `main` → require PR + require the Jenkins status check → **a PR can't merge unless CI is green** (Part 7).

## 13.2 Continuous *Delivery* with a manual approval gate

Add before `Deploy`:

```groovy
stage('Approval') {
    steps {
        timeout(time: 30, unit: 'MINUTES') {
            input message: "Deploy build #${env.BUILD_NUMBER} to production?", ok: 'Deploy'
        }
    }
}
```

Now the pipeline pauses for a human: **Continuous Delivery**; removing it gives **Continuous Deployment** (Part 4).

## 13.3 Push the image to Docker Hub (links to Unit I Practical 4)

Credentials: **Manage Jenkins → Credentials → Add** → *Username with password* → username = Docker Hub user, password = **access token**, ID = `dockerhub-creds`.

```groovy
environment { DH = credentials('dockerhub-creds') }
stage('Push Image') {
    steps {
        sh '''
            echo "$DH_PSW" | docker login -u "$DH_USR" --password-stdin
            docker tag ${IMAGE}:${TAG} $DH_USR/${IMAGE}:${TAG}
            docker push $DH_USR/${IMAGE}:${TAG}
            docker logout
        '''
    }
}
```

## 13.4 Deploy to a second server over SSH

Install the **SSH Agent** plugin; add an *SSH Username with private key* credential `deploy-key`; the target server needs Docker and the pushed image available (Docker Hub):

```groovy
stage('Deploy to Remote') {
    steps {
        sshagent(credentials: ['deploy-key']) {
            sh '''
                ssh -o StrictHostKeyChecking=no ubuntu@<TARGET_IP> "
                    docker pull <dockerhub-user>/flask-cicd-demo:${TAG} &&
                    docker rm -f flask-cicd-demo || true &&
                    docker run -d --name flask-cicd-demo --restart unless-stopped -p 80:5000 <dockerhub-user>/flask-cicd-demo:${TAG}"
            '''
        }
    }
}
```

## 13.5 Private repository

GitHub → fine-grained **Personal Access Token** (read access to repo contents) → Jenkins credential *Username with password* (username = GitHub user, password = token) → select it in the job's SCM **Credentials** field.

## 13.6 No public IP? Two fallbacks

```
• Poll SCM (no webhook needed): Build Triggers → Poll SCM → schedule "H/2 * * * *" (checks every ~2 minutes)
• Local Jenkins: run ngrok (ngrok http 8080) and use https://<id>.ngrok-free.app/github-webhook/ as the Payload URL
```

## 13.7 Notifications

Email Extension or Slack plugin in `post { failure { ... } }`; or simple email step: `mail to:'you@example.com', subject:"Build ${env.BUILD_NUMBER} failed", body:"${env.BUILD_URL}"` (needs SMTP configured).

## 13.8 Add a BDD acceptance stage (links to Part 3)

```groovy
stage('Acceptance (BDD)') {
    steps {
        sh '''
            . venv/bin/activate
            pip install behave requests
            behave --junit --junit-directory reports/behave
        '''
    }
    post { always { junit 'reports/behave/*.xml' } }
}
```

---

# 1️⃣4️⃣ CLEANUP 🧹 (MANDATORY)

```bash
# on the EC2 server (optional, before terminating)
docker rm -f flask-cicd-demo
docker system prune -af
```

```
☐ GitHub: delete the webhook (Settings → Webhooks) and revoke any Personal Access Tokens used
☐ Jenkins: delete credentials you created (Docker Hub token etc.)
☐ AWS: TERMINATE the jenkins-server instance (or STOP it if you'll continue; EBS storage still costs a little)
☐ AWS: release any Elastic IP; delete the jenkins-sg security group; delete the key pair if unused
☐ Check Billing/Cost Explorer next day for stray charges
```

---

# 1️⃣5️⃣ LAB REPORT TEMPLATE 📝

```
Title:        Practical 6: Jenkins CI/CD pipeline with GitHub webhook
Aim:          (copy syllabus line)
Requirements: AWS EC2 (Ubuntu), Jenkins LTS, Java 21, Git, Python 3, Docker, GitHub account
Theory:       CI, CD (Delivery vs Deployment), Jenkins architecture, Jenkinsfile, webhook, pipeline stages, quality gates, rollback
Architecture: (diagram from Section 0)
Procedure:    Steps 1-10 with commands (summarize)
Pipeline code: Jenkinsfile with stage explanations
Output:       screenshots from the evidence checklist
Observations: automatic trigger time, number of tests, coverage, build duration, effect of failing test and rollback
Result:       Pipeline automatically built, tested and deployed the application on every push; failures prevented deployment.
Precautions:  restricted security group, no secrets in repo, cleanup of resources
Conclusion + Viva answers
```

---

# 1️⃣6️⃣ PRACTICAL VIVA QUESTIONS 🎯

**Q1. Why did you choose Pipeline over Freestyle?** Pipeline as code in a Jenkinsfile is versioned, reviewable and supports complex flows; Freestyle is UI-configured and not versioned.

**Q2. How does GitHub trigger Jenkins?** A webhook sends an HTTP POST to `/github-webhook/` on push; the job with the GitHub hook trigger starts a build.

**Q3. Why webhook over polling?** Instant, event-driven, no repeated repository checks.

**Q4. What happens if a unit test fails?** The Unit Tests stage fails, later stages (build, deploy) are skipped, and the old version keeps running.

**Q5. How is deployment automated here?** The Deploy stage replaces the Docker container with the new image on the server after all earlier stages pass.

**Q6. What is the smoke test and why use it?** A quick post-deploy check of `/health`; it catches deployments that start but don't work, and triggers rollback.

**Q7. How does your rollback work?** Before deploying, the pipeline records the running image; if the smoke test fails, the stage `post { failure }` restarts that previous image.

**Q8. What is the role of `.` instead of `source`?** `sh` runs dash which lacks `source`; the dot command activates the virtualenv.

**Q9. Why must the jenkins user be in the docker group?** To run Docker commands without root; requires a Jenkins restart after the change.

**Q10. What is `junit` step used for?** Publishes test results and trends; marks the build unstable on test failures.

**Q11. How did you version the image?** Tag = Jenkins build number, shown in the app, giving traceability from deployment to build.

**Q12. How would you make this Continuous Delivery instead of Deployment?** Add an `input` approval stage before Deploy.

**Q13. How would you secure this setup?** Restrict security group, HTTPS reverse proxy, no anonymous access, credentials store, run builds on agents, update plugins.

**Q14. What are the quality gates in your pipeline?** Lint success, all tests passing, coverage ≥ 70%, smoke test.

**Q15. How would you scale to many branches/PRs?** Multibranch pipeline with PR checks and branch protection requiring CI success.

---

# ✅ QUICK CHECKLIST

```
Repo (app + tests + Dockerfile + Jenkinsfile) → EC2 Ubuntu t3.small + SG (22, 8080, 80) → Java 21 + Jenkins + Git + Python venv + Docker
→ jenkins in docker group + restart → unlock + suggested plugins + admin + URL → Pipeline job (SCM + hook trigger)
→ Build Now (manual proof) → GitHub webhook (…/github-webhook/, JSON, push) → git push = auto build + deploy
→ break test = no deploy ; break startup = rollback → evidence → CLEANUP (terminate EC2, delete webhook/tokens)
```

---

## 🫡 PART 8 STATUS

Practical 6 is fully scripted: a working application, a production-style Jenkinsfile with quality gates, automatic deployment, smoke test and rollback, a GitHub webhook, proof experiments, bonus variations, troubleshooting and cleanup.

**STOP here.** Last file:

> **🏆 GRAND CONSOLIDATION: Unit III full revision, comparison tables, 30 viva Qs, traps, what to write in exams, master sheet**
