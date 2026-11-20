# 🐳 UNIT I — PART 5: CONTAINERIZATION + DOCKER ARCHITECTURE + LIFE CYCLE

> **Covered in this document:** the "works on my machine" problem → what a container is → VM vs container (deep table) → how containers work inside Linux (namespaces, cgroups, union filesystem) → what Docker is → Docker architecture (client, daemon, containerd, runc, registry, objects) → images and layers → containers → image vs container → OCI standards → Docker container life cycle (create, start, run, pause, unpause, stop, restart, kill, remove) with state diagram → image life cycle (Dockerfile → build → tag → push → pull → run) → dry run of `docker run hello-world` → Docker Desktop vs Engine → containers in the cloud (ECS, AKS, Fargate) → orchestration intro (Kubernetes) → pros and cons → viva → master sheet.

---

Absolutely mate. 😤🔥 Now the second big pillar of Unit I: **containerization**. This is where modern DevOps starts, and where **Practical 4** lives.

---

# 1️⃣ THE PROBLEM CONTAINERS SOLVE

## 1.1 "It works on my machine!" 😭

```
Developer laptop:  Python 3.11, library X v2.1, Ubuntu
Test server:       Python 3.8,  library X v1.9, CentOS
Production:        Python 3.10, library X missing

Result: app works on laptop, breaks everywhere else.
```

Cause: the **environment** (OS libs, runtime versions, config) differs.

## 1.2 The shipping container analogy 🚢

Before standard shipping containers, goods were loaded loose: slow, damaged, different at every port. Then came **one standard box**: pack once, ship by truck, train or ship, open anywhere.

```
Software container = standard box that holds
                     app + runtime + libraries + config
Runs the same on laptop, test server, AWS, Azure.
```

## 1.3 Definition (exam)

> A **container** is a lightweight, isolated, portable package that bundles an application with all its dependencies and runs on a shared host operating-system kernel.

Containerization = packaging software this way.

---

# 2️⃣ VIRTUAL MACHINE vs CONTAINER ⭐⭐⭐

```
VIRTUAL MACHINES                       CONTAINERS
┌────────┬────────┬────────┐          ┌────────┬────────┬────────┐
│ App A  │ App B  │ App C  │          │ App A  │ App B  │ App C  │
│ Libs   │ Libs   │ Libs   │          │ Libs   │ Libs   │ Libs   │
│ Guest  │ Guest  │ Guest  │          ├────────┴────────┴────────┤
│ OS     │ OS     │ OS     │          │   Container runtime       │
├────────┴────────┴────────┤          │   (Docker Engine)         │
│       Hypervisor          │          ├──────────────────────────┤
│       Host OS / Hardware  │          │   Host OS (shared kernel) │
└──────────────────────────┘          │   Hardware                │
                                       └──────────────────────────┘
```

| Aspect | Virtual Machine | Container |
|---|---|---|
| Virtualizes | Hardware | Operating system (process level) |
| Own OS kernel? | Yes (full guest OS) | No, shares host kernel |
| Size | GBs | MBs (often tens to hundreds) |
| Startup | Minutes | Seconds or less |
| Overhead | High | Low |
| Density per host | Few | Many (dozens-hundreds) |
| Isolation | Strong (separate kernel) | Process-level (weaker than VM) |
| Portability | Heavier to move | Very portable via images |
| Best for | Different OS needs, strong isolation | Microservices, CI/CD, fast scaling |
| Example tech | VMware, KVM, EC2 | Docker, containerd, Podman |

💡 Containers do **not** replace VMs. In the cloud, containers usually run **inside** VMs (or on serverless platforms like Fargate). Both coexist.

⚠️ A Linux container needs a Linux kernel. Docker Desktop on Windows/Mac quietly runs a small Linux VM underneath.

---

# 3️⃣ HOW CONTAINERS WORK INSIDE (LINUX MAGIC) 🔬

A container is not a tiny VM. It is a **normal process with fences** around it. Three Linux features create the fences:

## 3.1 Namespaces: isolation ("what you can SEE")

| Namespace | Isolates |
|---|---|
| **PID** | Process IDs (container sees its own process 1) |
| **NET** | Network interfaces, IPs, ports |
| **MNT** | Filesystem mounts |
| **UTS** | Hostname |
| **IPC** | Inter-process communication |
| **USER** | User/group IDs |

## 3.2 cgroups (control groups): limits ("what you can USE")

Restrict and measure CPU, memory, disk I/O, network for a container.

```
docker run --memory=256m --cpus=0.5 myapp
→ cgroup caps this container to 256 MB RAM and half a CPU core
```

## 3.3 Union / layered filesystem (OverlayFS)

Images are built in **read-only layers**; a thin **writable layer** is added on top for a running container. Shared layers save disk and speed up downloads.

```
Container writable layer   (changes made at runtime; lost when container removed)
─────────────────────────
Layer 4: COPY app code     ┐
Layer 3: RUN pip install   │ read-only image layers
Layer 2: install Python    │ (shared between containers)
Layer 1: base OS (alpine)  ┘
```

---

# 4️⃣ WHAT IS DOCKER? 🐳

> Docker is an open-source platform to **build, ship and run** applications in containers.

- Released 2013; made containers easy for everyone.
- Provides: CLI, image format, build tool, registry (Docker Hub), runtime.
- Docker Inc. is the company; **Docker Engine** is the core software; Docker Desktop is the GUI app for Windows/Mac/Linux.

Slogan: **Build once, run anywhere.**

## 4.1 Benefits

```
Consistency across environments      Fast startup
Lightweight (many per server)        Easy CI/CD (same image from dev to prod)
Isolation of dependencies            Microservice friendly
Version-controlled images            Efficient scaling
```

---

# 5️⃣ DOCKER ARCHITECTURE ⭐⭐⭐ (DRAW THIS IN EXAMS)

## 5.1 The big picture (client–server)

```
┌────────────────┐  REST API   ┌───────────────────────────────────┐        ┌───────────────┐
│ Docker CLIENT  │ ─────────▶  │         DOCKER HOST                │        │   REGISTRY    │
│ docker build   │ (unix sock/ │  ┌───────────────────────────┐    │ pull   │ (Docker Hub,  │
│ docker pull    │  TCP)       │  │ Docker DAEMON (dockerd)   │◀───┼────────│  ECR, ACR,    │
│ docker run     │             │  │  manages:                 │ push   ───▶ │  GHCR)        │
└────────────────┘             │  │  Images  Containers       │    │        └───────────────┘
                               │  │  Networks  Volumes        │    │
                               │  └──────────┬────────────────┘    │
                               │        containerd → runc          │
                               │  [Container1] [Container2] ...    │
                               └───────────────────────────────────┘
```

## 5.2 Components explained

| Component | Role |
|---|---|
| **Docker Client (`docker`)** | The CLI you type into. Sends commands to the daemon via REST API. Can talk to a remote daemon too |
| **Docker Daemon (`dockerd`)** | Background service that does the work: builds images, runs containers, manages networks/volumes. Listens on the API |
| **containerd** | Industry-standard container runtime manager used by Docker (and Kubernetes). Handles image transfer, storage, container lifecycle |
| **runc** | Low-level OCI runtime that actually creates the container process using namespaces/cgroups |
| **Registry** | Storage/distribution for images. **Docker Hub** is the default public registry |
| **Docker Objects** | Images, Containers, Networks, Volumes, Plugins |

Flow of `docker run nginx`:

```
Client ──▶ Daemon: "run nginx"
Daemon checks local images → not found
Daemon ──▶ Registry (Docker Hub): pull nginx
Daemon ──▶ containerd ──▶ runc: create container (namespaces + cgroups)
Container starts → process running → output/ports available
```

## 5.3 Docker objects in detail

| Object | Meaning |
|---|---|
| **Image** | Read-only template (layers) used to create containers; built from a Dockerfile |
| **Container** | Runnable instance of an image |
| **Network** | Lets containers talk (bridge, host, none, overlay) |
| **Volume** | Persistent storage that survives container removal |
| **Registry / Repository** | Registry = server (Docker Hub). Repository = collection of image versions (e.g., `nginx`) |
| **Tag** | Version label (`nginx:1.27`, `latest`) |

## 5.4 Image vs Container (classic viva) ⭐

```
Image     = recipe / class / blueprint / read-only template
Container = cooked dish / object / running instance

One image → many containers
```

| | Image | Container |
|---|---|---|
| State | Static, read-only | Running or stopped, has writable layer |
| Created by | `docker build` / `docker pull` | `docker run` / `docker create` |
| Stored in | Local cache and registry | On host only |

---

# 6️⃣ OCI — OPEN STANDARDS

**OCI (Open Container Initiative)** defines standards so tools interoperate:

- **Image spec:** how images are structured
- **Runtime spec:** how containers are run
- **Distribution spec:** how images are shared

Because of OCI, images built with Docker run on Podman, containerd, Kubernetes, AWS Fargate, Azure ACI.

---

# 7️⃣ DOCKER CONTAINER LIFE CYCLE ⭐⭐⭐ (SYLLABUS TOPIC)

## 7.1 States

```
                  docker create
   (image) ───────────────────────▶ CREATED
                                      │ docker start
                                      ▼
   docker run  = create + start ▶  RUNNING ◀──── docker unpause ───┐
                                      │  │                          │
                       docker pause   │  └──────────▶ PAUSED ───────┘
                                      │
                 docker stop / kill   │         process exits on its own
                                      ▼                │
                                    STOPPED (exited) ◀─┘
                                      │  │
                         docker start │  │ docker rm
                          (again)     ▼  ▼
                                  RUNNING  REMOVED (deleted)
```

## 7.2 Commands per stage

| Stage | Command | What happens |
|---|---|---|
| **Create** | `docker create nginx` | Container made from image, not started |
| **Start** | `docker start <id>` | Starts a created/stopped container |
| **Run** | `docker run nginx` | Pull (if needed) + create + start in one go |
| **Pause** | `docker pause <id>` | Freezes processes (cgroup freezer), memory kept |
| **Unpause** | `docker unpause <id>` | Resumes |
| **Stop** | `docker stop <id>` | Graceful: sends SIGTERM, waits (default 10 s), then SIGKILL |
| **Restart** | `docker restart <id>` | Stop then start |
| **Kill** | `docker kill <id>` | Immediate SIGKILL |
| **Remove** | `docker rm <id>` | Deletes stopped container and its writable layer |
| **Inspect** | `docker ps`, `docker ps -a`, `docker logs`, `docker inspect`, `docker stats` | Observe state |

## 7.3 Important lifecycle facts

```
• A container lives only as long as its main process (PID 1) runs.
  If the process exits → container stops.
• stop = graceful (SIGTERM), kill = forceful (SIGKILL).
• Data in the writable layer is LOST on rm → use volumes for persistence.
• "Exited" containers still occupy disk until removed (docker container prune).
• Restart policies: no | on-failure | always | unless-stopped
   e.g.  docker run --restart unless-stopped nginx
```

## 7.4 Lifecycle walkthrough (DRY RUN, try this yourself)

```bash
docker create --name web -p 8080:80 nginx   # state: Created
docker ps -a                                 # STATUS: Created
docker start web                             # state: Running
docker ps                                    # STATUS: Up 5 seconds
docker pause web                             # state: Paused
docker unpause web                           # Running again
docker stop web                              # state: Exited (0)
docker start web                             # Running again
docker rm -f web                             # Removed (-f = force even if running)
```

---

# 8️⃣ IMAGE LIFE CYCLE (BUILD → SHIP → RUN)

```
Dockerfile ──docker build──▶ Image (local)
                               │ docker tag
                               ▼
                       user/app:1.0
                               │ docker push
                               ▼
                    Registry (Docker Hub / ECR / ACR)
                               │ docker pull   (on any other machine)
                               ▼
                         Image (local) ──docker run──▶ Container
```

The whole flow is **Practical 4**. Part 6 writes the Dockerfile; Part 8 gives the exact lab.

---

# 9️⃣ DRY RUN: `docker run hello-world`

```
$ docker run hello-world

Step 1  Client sends "run hello-world" to daemon
Step 2  Daemon: image not in local cache
        → prints "Unable to find image 'hello-world:latest' locally"
Step 3  Daemon pulls from Docker Hub library/hello-world (a tiny image, layer downloaded)
Step 4  Daemon creates container from image (writable layer, network, namespaces, cgroups)
Step 5  Container starts, prints "Hello from Docker!" explaining these steps
Step 6  Process ends → container status = Exited (0)

$ docker ps -a      → shows the exited container
$ docker images     → shows hello-world image cached locally
```

---

# 🔟 INSTALLING DOCKER (QUICK GUIDE)

| Platform | How |
|---|---|
| **Windows / macOS** | Docker Desktop (needs WSL2/Hyper-V on Windows; includes Engine + CLI + Compose) |
| **Ubuntu / Debian** | Docker's apt repository → `sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin` (follow docs.docker.com for current steps) |
| **Amazon Linux 2023 (EC2)** | `sudo dnf install -y docker && sudo systemctl enable --now docker && sudo usermod -aG docker ec2-user` (re-login after) |

Verify:

```bash
docker --version
docker info
docker run hello-world
```

Licensing note: Docker Desktop requires a paid subscription for larger companies; Docker Engine on Linux is free/open source. Alternatives: Podman, Rancher Desktop.

---

# 1️⃣1️⃣ CONTAINERS IN THE CLOUD ☁️

| Need | AWS | Azure |
|---|---|---|
| Registry | ECR | ACR |
| Run a container simply, no cluster | ECS on Fargate, App Runner | Azure Container Instances, Container Apps |
| Orchestrate at scale | ECS / EKS | AKS |
| Run Docker on a VM yourself | EC2 + Docker | VM + Docker |

## 11.1 Why orchestration? (intro to Kubernetes)

One container is easy. 500 containers across 50 servers need automation for:

```
Scheduling (which server runs what)    Self-healing (restart failed containers)
Scaling in/out                         Rolling updates, rollbacks
Service discovery + load balancing     Config and secrets
```

**Kubernetes (K8s)** is the standard orchestrator: Pods → Deployments → Services, running on a cluster of nodes. Docker Swarm and Amazon ECS are alternatives. (Kubernetes security comes in Unit V; here know the idea.)

```
Docker       = build and run containers (one host)
Kubernetes   = manage thousands of containers across many hosts
```

---

# 1️⃣2️⃣ CONTAINER SECURITY AT A GLANCE (PREVIEW OF UNIT V)

```
Use trusted/official minimal base images (alpine, slim, distroless)
Don't run as root inside the container (USER instruction)
Scan images for vulnerabilities (Docker Scout, Trivy)
Never bake secrets into images
Keep images updated; pin versions, avoid :latest in production
Limit resources and capabilities
```

---

# 1️⃣3️⃣ PROS AND CONS

| Pros | Cons |
|---|---|
| Portable and consistent | Weaker isolation than VMs (shared kernel) |
| Fast and light | Persistent data needs volumes |
| Efficient resource use | Orchestration adds complexity |
| Perfect for microservices and CI/CD | Security needs care (images, privileges) |
| Easy rollback by image tag | Networking can confuse beginners |

---

# 1️⃣4️⃣ VIVA QUESTIONS (PART 5) 🎯

**Q1. What is a container?** Lightweight isolated package of an app and its dependencies sharing the host kernel.

**Q2. Container vs VM?** Containers virtualize at OS level sharing the kernel, small and fast; VMs virtualize hardware with full guest OS, heavier but stronger isolation.

**Q3. What technologies isolate containers?** Namespaces (isolation), cgroups (resource limits), union filesystems (layers).

**Q4. Explain Docker architecture.** Client sends REST commands to the Docker daemon, which manages images, containers, networks, volumes using containerd/runc and pulls/pushes images from a registry.

**Q5. Image vs container?** Image = read-only template; container = running instance.

**Q6. What is Docker Hub?** Default public registry to store and share images.

**Q7. Explain container life cycle.** Created → Running → Paused → Stopped → Removed, via create/start/pause/unpause/stop/kill/rm; `run` = create + start.

**Q8. stop vs kill?** stop sends SIGTERM then SIGKILL after timeout; kill sends SIGKILL immediately.

**Q9. Why is data lost when a container is removed?** Writable layer is deleted with the container; use volumes/bind mounts.

**Q10. What is OCI?** Open standards for images and runtimes ensuring portability.

**Q11. What are image layers?** Read-only filesystem diffs stacked and cached; shared across images.

**Q12. Why Kubernetes?** Orchestrates many containers: scheduling, scaling, self-healing, rolling updates.

---

# 🧠 PART 5 MASTER SHEET

```
Container = app + deps, shares host kernel; Docker = build/ship/run
VM: hardware virtualization, guest OS, GBs, minutes | Container: OS-level, MBs, seconds
Isolation: namespaces (see) + cgroups (use) + union FS (layers)
Architecture: Client → REST → Daemon (dockerd) → containerd → runc ; Registry (Hub)
Objects: image, container, network, volume
Image = read-only template ; Container = running instance (+ writable layer)
Life cycle: create → start → (pause/unpause) → stop/kill → rm ; run = pull+create+start
Container lives while PID 1 runs ; data persists only with volumes
Orchestration: Kubernetes / ECS / AKS
```

---

## 🫡 PART 5 STATUS

You understand what containers are, why they beat "works on my machine," how Docker is built, and how containers live and die.

**STOP here.** Next up:

> **PART 6 — Docker Hands-on: Dockerfile, Docker Hub, Volumes, Networks, Compose 🔧**
