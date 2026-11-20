# 🔧 UNIT I — PART 6: DOCKER HANDS-ON (DOCKERFILE, DOCKER HUB, VOLUMES, NETWORKS, COMPOSE)

> **Covered in this document:** essential Docker commands (images, containers, cleanup) → Dockerfile from scratch with every instruction explained → a first Dockerfile (static site on Nginx) → a second one (Python Flask app) → a third (Node.js app) → build cache and layer order → `.dockerignore` → port mapping (`-p`) → environment variables → volumes and bind mounts (persistence) → Docker networks (bridge, host, none, user-defined) → Docker Hub deep dive (account, repos, tags, login, access tokens, push, pull, public vs private, official images, rate limits) → full push/pull dry run → multi-stage builds → Docker Compose → best practices → troubleshooting table → viva → master sheet.

---

Absolutely mate. 😤🔥 Part 5 was theory. Part 6 is **hands on keyboard**. After this you can finish Practical 4 without guessing.

Make sure Docker works first:

```bash
docker --version
docker run hello-world
```

---

# 1️⃣ ESSENTIAL DOCKER COMMANDS (CHEAT SHEET)

## 1.1 Images

| Command | Meaning |
|---|---|
| `docker images` (or `docker image ls`) | List local images |
| `docker pull nginx:1.27` | Download an image |
| `docker build -t myapp:1.0 .` | Build image from Dockerfile in current folder |
| `docker tag myapp:1.0 user/myapp:1.0` | Add another name/tag |
| `docker push user/myapp:1.0` | Upload to registry |
| `docker rmi myapp:1.0` | Remove an image |
| `docker history myapp:1.0` | Show layers |
| `docker image prune` | Remove dangling images |
| `docker search nginx` | Search Docker Hub from CLI |

## 1.2 Containers

| Command | Meaning |
|---|---|
| `docker run nginx` | Run (foreground) |
| `docker run -d nginx` | **d**etached: run in background |
| `docker run -d --name web -p 8080:80 nginx` | Name it and map host:container ports |
| `docker run -it ubuntu bash` | **i**nteractive + **t**ty: get a shell |
| `docker run --rm alpine echo hi` | Auto-remove after exit |
| `docker ps` / `docker ps -a` | List running / all containers |
| `docker logs web` / `docker logs -f web` | View / follow logs |
| `docker exec -it web sh` | Open a shell inside a running container |
| `docker stop web` / `docker start web` / `docker restart web` | Lifecycle |
| `docker rm web` / `docker rm -f web` | Remove (force) |
| `docker inspect web` | Full JSON details (IP, mounts, config) |
| `docker stats` | Live CPU/RAM usage |
| `docker cp web:/etc/nginx/nginx.conf .` | Copy files out/in |

## 1.3 Cleanup

```bash
docker container prune          # remove all stopped containers
docker image prune -a           # remove unused images
docker system prune             # remove stopped containers, unused networks, dangling images
docker system df                # how much disk Docker uses
```

---

# 2️⃣ THE DOCKERFILE (RECIPE FOR AN IMAGE) 📜

A **Dockerfile** is a plain text file named `Dockerfile` (no extension) listing steps to build an image. Each instruction creates a layer.

## 2.1 Every instruction you need

| Instruction | Purpose | Example |
|---|---|---|
| `FROM` | Base image (always first) | `FROM python:3.12-slim` |
| `WORKDIR` | Set working directory inside image | `WORKDIR /app` |
| `COPY` | Copy files from host to image | `COPY . .` |
| `ADD` | Like COPY plus URL/tar extraction (prefer COPY) | `ADD app.tar.gz /app` |
| `RUN` | Execute command **at build time** | `RUN pip install -r requirements.txt` |
| `ENV` | Set environment variable | `ENV PORT=5000` |
| `ARG` | Build-time variable | `ARG VERSION=1.0` |
| `EXPOSE` | Document the port the app listens on (does not publish it) | `EXPOSE 5000` |
| `USER` | Switch to non-root user | `USER appuser` |
| `VOLUME` | Declare a mount point | `VOLUME /data` |
| `CMD` | Default command **at run time** (overridable) | `CMD ["python","app.py"]` |
| `ENTRYPOINT` | Fixed executable; CMD supplies default args | `ENTRYPOINT ["python"]` |
| `HEALTHCHECK` | Command to test container health | `HEALTHCHECK CMD curl -f http://localhost:5000/ || exit 1` |
| `LABEL` | Metadata | `LABEL maintainer="you"` |

### RUN vs CMD vs ENTRYPOINT ⭐ (very common viva)

```
RUN         → runs while BUILDING the image (installs things); result baked into a layer
CMD         → runs when the container STARTS; easily overridden by docker run args
ENTRYPOINT  → the main program; harder to override; CMD gives its default arguments
```

### COPY vs ADD

Use **COPY** for plain file copying (clear and predictable). Use ADD only when you need auto-extraction of a local tar.

### Shell form vs exec form

```
CMD python app.py              ← shell form (runs via /bin/sh -c)
CMD ["python", "app.py"]       ← exec form (preferred; proper signal handling for docker stop)
```

---

# 3️⃣ DOCKERFILE #1 — STATIC WEBSITE ON NGINX (SIMPLEST) 🌐

Folder structure:

```
mysite/
├── Dockerfile
└── index.html
```

`index.html`:

```html
<!DOCTYPE html>
<html>
  <head><title>My Docker Site</title></head>
  <body>
    <h1>Hello from my Docker container 🐳</h1>
    <p>Served by Nginx inside a container.</p>
  </body>
</html>
```

`Dockerfile`:

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

Only 3 lines. Nginx image already has `CMD ["nginx","-g","daemon off;"]`.

Build and run:

```bash
docker build -t mysite:1.0 .
docker run -d --name mysite -p 8080:80 mysite:1.0
# open http://localhost:8080  (or http://<EC2-public-IP>:8080 on a server)
docker logs mysite
docker stop mysite && docker rm mysite
```

### What does `-p 8080:80` mean?

```
-p  HOST_PORT : CONTAINER_PORT
    8080 on your machine  →  80 inside the container
```

Container ports are private until you **publish** them. `EXPOSE` alone does not publish.

---

# 4️⃣ DOCKERFILE #2 — PYTHON FLASK APP 🐍

```
flaskapp/
├── Dockerfile
├── app.py
├── requirements.txt
└── .dockerignore
```

`app.py`:

```python
from flask import Flask
import os, socket

app = Flask(__name__)

@app.route("/")
def home():
    name = os.getenv("APP_NAME", "Docker Demo")
    return f"<h1>Hello from {name}</h1><p>Container hostname: {socket.gethostname()}</p>"

@app.route("/health")
def health():
    return {"status": "ok"}

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)   # 0.0.0.0 is REQUIRED inside containers
```

⚠️ **Classic bug:** if Flask binds to `127.0.0.1`, the app is reachable only inside the container and the published port will not work. Always bind to `0.0.0.0`.

`requirements.txt`:

```
flask==3.0.3
```

`Dockerfile`:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

# copy requirements first → better layer caching
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# then copy the rest of the code
COPY . .

EXPOSE 5000
CMD ["python", "app.py"]
```

`.dockerignore`:

```
__pycache__
*.pyc
.git
.env
venv
```

Build, run, test:

```bash
docker build -t flaskapp:1.0 .
docker run -d --name flask1 -p 5000:5000 -e APP_NAME="Practical 4" flaskapp:1.0
curl http://localhost:5000
curl http://localhost:5000/health
docker logs flask1
```

## 4.1 DRY RUN OF THE BUILD (what Docker prints/does)

```
Step 1  FROM python:3.12-slim     → pulls base image layers (cached next time)
Step 2  WORKDIR /app              → creates /app, new layer metadata
Step 3  COPY requirements.txt .   → layer with one file
Step 4  RUN pip install ...       → installs Flask; layer holds site-packages
Step 5  COPY . .                  → layer with your code
Step 6  EXPOSE / CMD              → metadata only
Result: image flaskapp:1.0 with ID, size ~ 150 MB
```

---

# 5️⃣ DOCKERFILE #3 — NODE.JS APP ⬢

`server.js`:

```javascript
const http = require("http");
const port = process.env.PORT || 3000;

http.createServer((req, res) => {
  res.writeHead(200, { "Content-Type": "text/plain" });
  res.end("Hello from Node inside Docker!\n");
}).listen(port, "0.0.0.0", () => console.log("Listening on " + port));
```

`package.json`:

```json
{ "name": "node-demo", "version": "1.0.0", "main": "server.js",
  "scripts": { "start": "node server.js" } }
```

`Dockerfile`:

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY . .
EXPOSE 3000
USER node
CMD ["npm", "start"]
```

```bash
docker build -t nodeapp:1.0 .
docker run -d -p 3000:3000 --name node1 nodeapp:1.0
curl localhost:3000
```

---

# 6️⃣ BUILD CACHE AND LAYER ORDER ⚡

Docker reuses a cached layer if the instruction and its inputs are unchanged. **A changed layer invalidates all layers after it.**

```
BAD order                            GOOD order
COPY . .                             COPY requirements.txt .
RUN pip install -r requirements.txt  RUN pip install -r requirements.txt
                                     COPY . .
Any code edit → reinstalls packages  Code edit → only last layer rebuilds (seconds)
```

Rules:

```
Put rarely-changing steps first, frequently-changing last
Combine related RUN commands with && to reduce layers
Use small base images (alpine, slim)
Use .dockerignore to keep junk out of the build context
```

---

# 7️⃣ ENVIRONMENT VARIABLES

```bash
docker run -e APP_NAME="Test" -e PORT=5000 flaskapp:1.0
docker run --env-file .env flaskapp:1.0     # many variables from a file
```

⚠️ Do not `COPY .env` into the image and do not hardcode passwords in the Dockerfile. Pass secrets at runtime or use a secrets manager (Unit V).

---

# 8️⃣ VOLUMES AND BIND MOUNTS (PERSISTENCE) 💾

Container filesystem is **ephemeral**. Remove the container and the data vanishes. Fix with:

| Type | Syntax | Where data lives | Use |
|---|---|---|---|
| **Named volume** | `-v mydata:/var/lib/mysql` | Managed by Docker | Databases, production data |
| **Bind mount** | `-v $(pwd)/site:/usr/share/nginx/html` | A folder on the host | Development (live edit) |
| **tmpfs** | `--tmpfs /tmp` | RAM | Temporary/secret scratch data |

```bash
docker volume create appdata
docker run -d --name db -v appdata:/data alpine sleep 3600
docker exec db sh -c "echo saved > /data/note.txt"
docker rm -f db
docker run --rm -v appdata:/data alpine cat /data/note.txt     # prints: saved  ✅ data survived

docker volume ls
docker volume inspect appdata
docker volume rm appdata
```

---

# 9️⃣ DOCKER NETWORKING 🌐

| Network driver | Meaning |
|---|---|
| **bridge** (default) | Private network on the host; containers get internal IPs; outside access via `-p` |
| **host** | Container shares the host's network directly (no isolation, no port mapping needed) |
| **none** | No networking |
| **overlay** | Multi-host networks (Swarm/Kubernetes-like setups) |
| **user-defined bridge** | Custom bridge with **automatic DNS by container name** ✅ |

```bash
docker network create appnet
docker run -d --name api --network appnet flaskapp:1.0
docker run -it --rm --network appnet alpine sh -c "apk add -q curl && curl http://api:5000/health"
# containers reach each other by NAME on a user-defined network
```

```
Browser ──8080──▶ Host ──(port mapping)──▶ [container web :80]
                                              │ network appnet
                                           [container api :5000] [container db :5432]
```

---

# 🔟 DOCKER HUB DEEP DIVE ⭐⭐⭐ (SYLLABUS: "Docker Hub")

## 10.1 What is Docker Hub?

> Docker Hub is Docker's cloud-based **registry service** where you find, store, share and distribute container images.

Like **GitHub for images**: repositories, versions (tags), public/private, teams, automation.

URL: `hub.docker.com`

## 10.2 Terms

| Term | Meaning |
|---|---|
| **Registry** | Server that stores images (Docker Hub, ECR, ACR, GHCR, Google Artifact Registry, self-hosted) |
| **Repository** | Collection of related image versions: `username/myapp` |
| **Tag** | Version label: `username/myapp:1.0`, `:latest` |
| **Namespace** | Your username or organization |
| **Official Images** | Curated, maintained images (`nginx`, `python`, `mysql`, `node`, `redis`): written without namespace (`library/nginx`) |
| **Verified Publisher** | Trusted vendor images |
| **Digest** | Immutable content hash (`sha256:...`) uniquely identifying an image |

Full image name anatomy:

```
docker.io / username / myapp : 1.0
   │          │         │      └ tag (default: latest)
   │          │         └ repository
   │          └ namespace (account/org)
   └ registry host (default docker.io)
```

## 10.3 Public vs Private repositories

| | Public | Private |
|---|---|---|
| Visible to | Everyone | You and collaborators |
| Cost | Free | Free tier allows a limited number; more via paid plans (check current Docker pricing) |
| Use | Open source, sharing | Company code |

## 10.4 Rate limits

Docker Hub limits image pulls for anonymous and free users (a cap of pulls per time window per IP/account; exact numbers change, so check Docker's current docs). Logging in raises limits. In CI or on shared networks, you may see `toomanyrequests`: log in or use a mirror/private registry.

## 10.5 STEP-BY-STEP: PUSH YOUR IMAGE TO DOCKER HUB (PRACTICAL 4 CORE)

**Step 1: Create account** at hub.docker.com (note your username, say `rahul123`).

**Step 2: Create access token (better than password)**
Account Settings → Personal access tokens (Security) → New token → copy it once.

**Step 3: Login from terminal**

```bash
docker login -u rahul123
# paste the access token when asked for password
# Login Succeeded
```

**Step 4: Build with your namespace in the name (or tag later)**

```bash
docker build -t rahul123/flaskapp:1.0 .
# or tag an existing image
docker tag flaskapp:1.0 rahul123/flaskapp:1.0
```

**Step 5: Push**

```bash
docker push rahul123/flaskapp:1.0
# Pushing layers... "1.0: digest: sha256:... size: ..."
# also tag latest if you want
docker tag rahul123/flaskapp:1.0 rahul123/flaskapp:latest
docker push rahul123/flaskapp:latest
```

**Step 6: Verify** on hub.docker.com → Repositories → `rahul123/flaskapp` → Tags.

**Step 7: Pull and run on ANOTHER system** (another laptop, another EC2 instance)

```bash
docker pull rahul123/flaskapp:1.0
docker run -d -p 5000:5000 --name flask-remote rahul123/flaskapp:1.0
curl http://localhost:5000        # works ✅ same behaviour = portability proven
```

**Step 8: Logout (shared machines!)**

```bash
docker logout
```

## 10.6 DRY RUN OF `docker push`

```
Local image rahul123/flaskapp:1.0 has layers L1..L5
docker push → client asks registry "which layers do you already have?"
  • L1 (python base) already on Hub → "Layer already exists"   (skipped)
  • L2..L5 not present                → uploaded
Registry stores a manifest listing layers → tag 1.0 → digest sha256:abc...
On pull: another machine downloads only layers it lacks → fast
```

## 10.7 Other registries (same workflow, different login)

```bash
# AWS ECR
aws ecr create-repository --repository-name flaskapp
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <account>.dkr.ecr.ap-south-1.amazonaws.com
docker tag flaskapp:1.0 <account>.dkr.ecr.ap-south-1.amazonaws.com/flaskapp:1.0
docker push <account>.dkr.ecr.ap-south-1.amazonaws.com/flaskapp:1.0

# Azure ACR
az acr login --name myregistry
docker tag flaskapp:1.0 myregistry.azurecr.io/flaskapp:1.0
docker push myregistry.azurecr.io/flaskapp:1.0
```

---

# 1️⃣1️⃣ MULTI-STAGE BUILDS (SMALLER, SAFER IMAGES)

Build with heavy tools in stage 1, copy only the result into a tiny final stage.

```dockerfile
# Stage 1: build
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build                # outputs /app/dist

# Stage 2: serve static files with tiny nginx
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

Final image has no Node, no source, no dev dependencies: smaller and less attack surface.

---

# 1️⃣2️⃣ DOCKER COMPOSE (MULTI-CONTAINER IN ONE FILE) 🎼

When your app needs web + database + cache, use `docker-compose.yml` (modern command: `docker compose`, with a space).

```yaml
services:
  web:
    build: .
    ports:
      - "5000:5000"
    environment:
      - APP_NAME=Compose Demo
      - REDIS_HOST=cache
    depends_on:
      - cache
  cache:
    image: redis:7-alpine
    volumes:
      - redisdata:/data

volumes:
  redisdata:
```

```bash
docker compose up -d        # build + create network + start everything
docker compose ps
docker compose logs -f
docker compose down         # stop and remove containers/network (add -v to remove volumes)
```

Compose automatically creates a network so `web` can reach Redis at hostname `cache`.

---

# 1️⃣3️⃣ BEST PRACTICES ✅

```
Use official, small base images (alpine/slim); pin versions (python:3.12-slim, not :latest)
One process per container; stateless containers; data in volumes
Order Dockerfile for caching; use .dockerignore
Run as non-root (USER)
No secrets in images; use env vars/secret managers at runtime
Scan images (docker scout cves IMAGE / trivy image IMAGE)
Add HEALTHCHECK; set restart policies and resource limits
Tag meaningfully (1.0.3, git commit SHA), avoid relying on :latest
Clean up unused images/containers regularly
```

---

# 1️⃣4️⃣ TROUBLESHOOTING TABLE 🛠️

| Problem | Likely cause | Fix |
|---|---|---|
| `permission denied ... docker.sock` | User not in docker group | `sudo usermod -aG docker $USER`, re-login (or use sudo) |
| `Cannot connect to the Docker daemon` | Daemon not running | `sudo systemctl start docker` / start Docker Desktop |
| Page not opening on published port | App bound to 127.0.0.1; wrong `-p`; firewall/SG closed | Bind `0.0.0.0`; check `docker ps` PORTS; open port in Security Group |
| `port is already allocated` | Host port used | Use another host port, e.g., `-p 8081:80` |
| Container exits immediately | Main process ended/crashed | `docker logs <name>`; ensure foreground process (`daemon off`) |
| `denied: requested access to the resource is denied` on push | Not logged in or image name lacks your username | `docker login`; retag `username/repo:tag` |
| `repository does not exist` | Repo/namespace typo | Check Hub; create repo or fix name |
| `toomanyrequests` | Pull rate limit | Login, wait, or use mirror |
| `no space left on device` | Old images/containers | `docker system prune` |
| Image huge | Big base, no .dockerignore | Slim base, multi-stage |
| `exec format error` | CPU architecture mismatch (ARM vs x86) | `docker build --platform linux/amd64` or use buildx multi-arch |

---

# 1️⃣5️⃣ VIVA QUESTIONS (PART 6) 🎯

**Q1. What is a Dockerfile?** Text file with instructions to build an image.

**Q2. RUN vs CMD vs ENTRYPOINT?** RUN executes at build time; CMD sets default run-time command; ENTRYPOINT sets the fixed executable.

**Q3. COPY vs ADD?** COPY copies files; ADD also extracts tar archives and fetches URLs; prefer COPY.

**Q4. EXPOSE vs `-p`?** EXPOSE documents the port; `-p` actually publishes it to the host.

**Q5. Why copy requirements.txt before the code?** To use layer caching so dependencies don't reinstall on every code change.

**Q6. What is Docker Hub?** Cloud registry for storing, sharing and distributing images.

**Q7. Steps to push an image to Docker Hub?** Create account/token, `docker login`, tag as `username/repo:tag`, `docker push`.

**Q8. Volume vs bind mount?** Volume is Docker-managed storage; bind mount maps a host path.

**Q9. Default Docker network?** Bridge; user-defined bridges add DNS by container name.

**Q10. What is Docker Compose?** Tool to define and run multi-container apps with a YAML file.

**Q11. What is a multi-stage build?** Using multiple FROM stages to produce a small final image without build tools.

**Q12. How do you verify image portability?** Pull the image on a different machine and run it; the behavior is identical.

**Q13. What is an image digest?** Immutable sha256 hash identifying exact image content.

---

# 🧠 PART 6 MASTER SHEET

```
Build:   docker build -t user/app:tag .
Run:     docker run -d --name n -p HOST:CONT -e K=V -v vol:/path image
Inspect: ps, logs, exec -it, inspect, stats
Share:   login → tag user/app:tag → push → (other machine) pull → run
Dockerfile: FROM, WORKDIR, COPY, RUN, ENV, EXPOSE, USER, CMD/ENTRYPOINT, HEALTHCHECK
Order for cache: base → deps manifest → install → source code
Bind app to 0.0.0.0 ; publish with -p ; EXPOSE is only documentation
Persist: volumes ; Networks: bridge default, user-defined = DNS by name
Docker Hub = public registry ; official images ; tokens > passwords ; rate limits
Compose = multi-container ; multi-stage = small images
```

---

## 🫡 PART 6 STATUS

You can write Dockerfiles, build, run, persist data, network containers, and publish/pull images via Docker Hub.

**STOP here.** Next up:

> **PART 7 — Case Studies: Netflix, Amazon, Uber 🌍**
