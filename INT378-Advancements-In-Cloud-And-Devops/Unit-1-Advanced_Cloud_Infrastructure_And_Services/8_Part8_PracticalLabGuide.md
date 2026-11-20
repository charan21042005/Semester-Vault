# 🧪 UNIT I — PART 8: PRACTICAL LAB GUIDE (EC2 + VPC + NGINX, AND DOCKER BUILD/PUSH/PULL)

> **Covered in this document:** lab safety and billing rules → **Practical 1** (AWS account, region, VPC check/creation, security group, key pair, launch EC2, SSH/RDP, install Nginx/Apache, deploy a simple web app, verify with public IP, cleanup) → both console path and CLI path → user-data automation → **Practical 4** (install Docker, write app, Dockerfile, build, run, expose ports, push to Docker Hub, pull on another system, container lifecycle operations, verification) → using Docker on EC2 (combining both practicals) → expected outputs → troubleshooting flowcharts → lab report template → viva for practicals → quick checklists.

---

Absolutely mate. 😤🔥 Time to convert theory into running things. This file is written like a lab manual: follow the steps in order, note the expected output, and clean up at the end.

---

# 0️⃣ LAB RULES (READ FIRST) 🚨

```
1. Set a BILLING ALERT / budget before launching anything (Billing → Budgets).
2. Use the Free Tier eligible instance type shown in the launch wizard (commonly t2.micro / t3.micro depending on region/account).
3. NEVER open SSH (22) to the whole internet in a real project: use "My IP". (In labs, use My IP too.)
4. Keep the .pem key file private. Never commit it to GitHub.
5. When finished: TERMINATE instances, release Elastic IPs, delete extra volumes/snapshots, delete the VPC items you created.
6. Free Tier terms changed for new AWS accounts in 2025 (credit-based plan). Check the current terms in your Billing console.
```

---

# 🅰️ PRACTICAL 1 — EC2 + VPC + SECURITY GROUP + SSH + WEB SERVER

> **Syllabus text:** *Create an AWS free-tier account. Launch an EC2 instance, configure VPC networking and security groups, connect remotely using SSH/RDP, install and configure a web server (Apache/Nginx), deploy a simple web application, and verify accessibility using the public IP address.*

## 1.1 Objective and mental model

```
You (laptop) ──SSH :22──▶ EC2 (in public subnet of a VPC) ◀──HTTP :80── Browser
                          │
                  Security Group controls both
                  Internet Gateway + route table make it reachable
```

## 1.2 Step 1: Create the AWS account

1. Go to aws.amazon.com → **Create an AWS account**.
2. Provide email, account name, verify email, set root password.
3. Add contact info, payment method (card required for verification), phone verification, choose a support plan (**Basic/Free**).
4. Sign in to the **Console** and choose a region near you (e.g., **Asia Pacific (Mumbai) ap-south-1**).
5. Secure root: **Security credentials → enable MFA** on the root user.
6. Create a Budget: Billing → **Budgets** → create a small monthly cost budget with email alert.

## 1.3 Step 2: Check networking (VPC)

New accounts have a **default VPC** with public subnets, an Internet Gateway and a route table, which is enough for the lab.

Console: **VPC → Your VPCs** → note the default VPC (CIDR usually `172.31.0.0/16`).
Check: **Route tables** → default route has `0.0.0.0/0 → igw-xxxx`. This is what makes the subnet public.

### Optional (for deeper marks): build your own VPC

Console: **VPC → Create VPC → "VPC and more"** (wizard) with:

```
Name:            lab-vpc
IPv4 CIDR:       10.0.0.0/16
AZs:             2
Public subnets:  2  (10.0.1.0/24, 10.0.2.0/24)
Private subnets: 0 or 2
NAT gateways:    None   (NAT Gateway costs money; skip for lab)
```

CLI equivalent (core pieces):

```bash
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query Vpc.VpcId --output text)
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames

SUBNET_ID=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 \
            --availability-zone ap-south-1a --query Subnet.SubnetId --output text)
aws ec2 modify-subnet-attribute --subnet-id $SUBNET_ID --map-public-ip-on-launch

IGW_ID=$(aws ec2 create-internet-gateway --query InternetGateway.InternetGatewayId --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

RT_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID --query RouteTable.RouteTableId --output text)
aws ec2 create-route --route-table-id $RT_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $RT_ID --subnet-id $SUBNET_ID
```

## 1.4 Step 3: Create the Security Group

Console: **EC2 → Security Groups → Create security group**

```
Name:        web-sg
Description: Allow SSH and HTTP
VPC:         (default or lab-vpc)

Inbound rules:
  Type SSH   | Protocol TCP | Port 22 | Source: My IP
  Type HTTP  | Protocol TCP | Port 80 | Source: Anywhere-IPv4 (0.0.0.0/0)
  (RDP 3389 only for Windows instances, source My IP)

Outbound: default (all traffic allowed)
```

CLI:

```bash
SG_ID=$(aws ec2 create-security-group --group-name web-sg --description "SSH and HTTP" \
        --vpc-id $VPC_ID --query GroupId --output text)
MYIP=$(curl -s https://checkip.amazonaws.com)
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22 --cidr ${MYIP}/32
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0
```

## 1.5 Step 4: Create a Key Pair

Console: **EC2 → Key Pairs → Create key pair** → name `lab-key`, type RSA, format **.pem** (OpenSSH; use **.ppk** only if using PuTTY) → file downloads once.

```bash
# Linux/Mac: restrict permissions (SSH refuses open keys)
chmod 400 lab-key.pem

# CLI way
aws ec2 create-key-pair --key-name lab-key --query KeyMaterial --output text > lab-key.pem
chmod 400 lab-key.pem
```

## 1.6 Step 5: Launch the EC2 instance

Console: **EC2 → Launch instance**

| Field | Value |
|---|---|
| Name | `web-server-1` |
| AMI | **Amazon Linux 2023** (or Ubuntu Server LTS) marked Free tier eligible |
| Instance type | `t2.micro` / `t3.micro` (Free tier eligible shown in wizard) |
| Key pair | `lab-key` |
| Network settings → Edit | VPC: your VPC; Subnet: a **public** subnet; **Auto-assign public IP: Enable**; Security group: select existing `web-sg` |
| Storage | 8 GiB gp3 (default) |
| Advanced → User data | (optional automation script below) |

Click **Launch instance** → wait for state **Running** and **2/2 status checks passed**.

### Optional: USER DATA to auto-install the web server at first boot

Amazon Linux 2023:

```bash
#!/bin/bash
dnf update -y
dnf install -y nginx
systemctl enable --now nginx
cat > /usr/share/nginx/html/index.html <<'EOF'
<!DOCTYPE html>
<html><head><title>EC2 Practical</title></head>
<body style="font-family:sans-serif;text-align:center;margin-top:60px">
  <h1>Hello from my EC2 instance ☁️</h1>
  <p>Nginx on Amazon Linux, deployed via user data.</p>
</body></html>
EOF
```

Ubuntu version:

```bash
#!/bin/bash
apt-get update -y
apt-get install -y nginx
systemctl enable --now nginx
echo "<h1>Hello from Ubuntu EC2</h1>" > /var/www/html/index.html
```

CLI launch:

```bash
aws ec2 run-instances --image-id <AMI_ID> --instance-type t3.micro \
   --key-name lab-key --security-group-ids $SG_ID --subnet-id $SUBNET_ID \
   --associate-public-ip-address --user-data file://userdata.sh \
   --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-server-1}]'
```

Get the public IP:

```bash
aws ec2 describe-instances --filters Name=tag:Name,Values=web-server-1 \
   --query "Reservations[].Instances[].PublicIpAddress" --output text
```

## 1.7 Step 6: Connect remotely

### Linux/Mac/Windows PowerShell (OpenSSH)

```bash
ssh -i lab-key.pem ec2-user@<PUBLIC_IP>      # Amazon Linux user: ec2-user
ssh -i lab-key.pem ubuntu@<PUBLIC_IP>        # Ubuntu user: ubuntu
```

First time: type `yes` to trust the host fingerprint.

### Windows with PuTTY

Convert `.pem` to `.ppk` with PuTTYgen → PuTTY → Host `ec2-user@<IP>` → Connection → SSH → Auth → select `.ppk` → Open.

### Alternative without SSH: **EC2 Instance Connect** (console → Connect button) or **Session Manager**.

### RDP (Windows Server instance)

Launch Windows AMI → Console **Connect → RDP client** → decrypt the Administrator password with your `.pem` → open Remote Desktop Connection to the public IP → port 3389 must be allowed from My IP.

## 1.8 Step 7: Install and configure the web server (manual way)

Skip if you used user data; otherwise run after SSH:

### Nginx on Amazon Linux

```bash
sudo dnf update -y
sudo dnf install -y nginx
sudo systemctl enable --now nginx
systemctl status nginx
```

### Apache on Amazon Linux

```bash
sudo dnf install -y httpd
sudo systemctl enable --now httpd
# web root: /var/www/html
```

### Nginx/Apache on Ubuntu

```bash
sudo apt update && sudo apt install -y nginx        # or apache2
sudo systemctl enable --now nginx                   # or apache2
```

## 1.9 Step 8: Deploy a simple web application

### Option A: static page

```bash
sudo bash -c 'cat > /usr/share/nginx/html/index.html <<EOF
<html><body>
<h1>Cloud Practical 1</h1>
<p>Name: YOUR NAME | Reg No: XXXX</p>
<p>Server: $(hostname)</p>
</body></html>
EOF'
```

(Apache web root: `/var/www/html/index.html`.)

### Option B: tiny dynamic app (Python Flask behind Nginx, or direct)

```bash
sudo dnf install -y python3-pip
pip3 install flask
cat > app.py <<'EOF'
from flask import Flask
app = Flask(__name__)
@app.route("/")
def home():
    return "<h1>Flask app on EC2 ☁️</h1>"
app.run(host="0.0.0.0", port=5000)
EOF
nohup python3 app.py > app.log 2>&1 &
```

If using port 5000 directly, **add inbound rule TCP 5000** in the security group. Better practice: configure Nginx as a **reverse proxy** to port 5000:

```nginx
# /etc/nginx/conf.d/app.conf
server {
    listen 80;
    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
    }
}
```

```bash
sudo nginx -t              # test config
sudo systemctl reload nginx
```

(If the default Nginx server block still answers on port 80, remove/disable it or adjust `default_server`.)

## 1.10 Step 9: Verify accessibility ✅

```
Browser:  http://<PUBLIC_IP>        → your page appears
Terminal: curl http://<PUBLIC_IP>   → prints the HTML
On server: curl localhost           → works locally
```

Use **HTTP**, not HTTPS, unless you configured TLS (a "not secure" warning is normal in the lab).

Evidence to attach to your report: screenshots of (1) running instance with public IP, (2) security group rules, (3) SSH session, (4) browser showing your page.

## 1.11 Troubleshooting flow: "My site isn't opening" 🔍

```
Is the instance RUNNING with 2/2 checks?                         no → wait/reboot
Does it have a PUBLIC IPv4?                                      no → enable auto-assign / attach Elastic IP
Does the subnet route table have 0.0.0.0/0 → IGW?                no → add route, attach IGW
Does the Security Group allow inbound TCP 80 from 0.0.0.0/0?     no → add rule
Does the NACL allow 80 inbound + ephemeral ports outbound?       no → fix NACL (default allows)
Is the web server running on the instance? (systemctl status)    no → start it
Does `curl localhost` work on the instance?                      no → app/web server config issue
Using https:// by mistake?                                       yes → use http://
```

| SSH problem | Cause / fix |
|---|---|
| `Permission denied (publickey)` | Wrong user name (ec2-user vs ubuntu) or wrong key |
| `UNPROTECTED PRIVATE KEY FILE` | `chmod 400 lab-key.pem` |
| `Connection timed out` | SG doesn't allow 22 from your current IP (IP changed), no public IP, or no IGW route |
| `Connection refused` | sshd not running (rare), wrong port |

## 1.12 Step 10: Cleanup (MANDATORY) 🧹

```bash
aws ec2 terminate-instances --instance-ids <INSTANCE_ID>
# wait for "terminated"
aws ec2 delete-security-group --group-id $SG_ID            # after instance is gone
aws ec2 delete-key-pair --key-name lab-key
# if you created a custom VPC: detach+delete IGW, delete subnets, route tables, then VPC
# release any Elastic IP:
aws ec2 release-address --allocation-id <eipalloc-id>
```

Console check: EC2 → Instances (none running), Volumes (none), Elastic IPs (none), Billing dashboard shows ~0.

---

# 🅱️ PRACTICAL 4 — DOCKER IMAGE, CONTAINER, DOCKER HUB, PULL ON ANOTHER SYSTEM

> **Syllabus text:** *Build a Docker image for a web application, execute it within a Docker container, expose required ports, push the image to Docker Hub, pull it onto another system, and verify successful deployment and container lifecycle operations.*

## 2.1 Objective and flow

```
SYSTEM A (your laptop or EC2-1)                   SYSTEM B (another PC or EC2-2)
 app code + Dockerfile                             
   │ docker build                                   
   ▼                                                
 image ── docker push ──▶ DOCKER HUB ── docker pull ──▶ image ── docker run ──▶ container ✅
```

## 2.2 Step 1: Install Docker (System A)

Ubuntu (recommended: follow docs.docker.com "Install Docker Engine on Ubuntu" for the latest repository commands) or Amazon Linux 2023 on EC2:

```bash
sudo dnf install -y docker
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user      # then log out and back in
docker --version
docker run hello-world                # smoke test
```

Windows/Mac: install Docker Desktop and start it.

## 2.3 Step 2: Create the web application

```bash
mkdir ~/docker-lab && cd ~/docker-lab
```

`app.py`:

```python
from flask import Flask
import socket, os

app = Flask(__name__)

@app.route("/")
def home():
    return f"""
    <h1>Docker Practical 4 🐳</h1>
    <p>Student: {os.getenv('STUDENT_NAME','YOUR NAME')}</p>
    <p>Served from container: {socket.gethostname()}</p>
    """

@app.route("/health")
def health():
    return {"status": "healthy"}

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

`requirements.txt`:

```
flask==3.0.3
```

`.dockerignore`:

```
__pycache__
*.pyc
.git
.env
```

## 2.4 Step 3: Write the Dockerfile

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

## 2.5 Step 4: Build the image

```bash
docker build -t dockerlab:1.0 .
docker images                    # confirm dockerlab 1.0 exists
docker history dockerlab:1.0     # show layers (good screenshot)
```

## 2.6 Step 5: Run the container and expose the port

```bash
docker run -d --name lab1 -p 5000:5000 -e STUDENT_NAME="Your Name" dockerlab:1.0
docker ps
curl http://localhost:5000
curl http://localhost:5000/health
docker logs lab1
```

If on EC2: add **inbound TCP 5000** (or map `-p 80:5000` and use port 80) in the Security Group, then open `http://<EC2_PUBLIC_IP>:5000`.

## 2.7 Step 6: Demonstrate container lifecycle operations (marks!) ⭐

```bash
docker ps                      # Running
docker pause lab1              # Paused → curl will hang/timeout
docker ps                      # STATUS: Up ... (Paused)
docker unpause lab1            # Running again
docker stop lab1               # Exited
docker ps -a                   # shows Exited (143 or 0) container
docker start lab1              # Running again
docker restart lab1
docker exec -it lab1 sh        # enter container: ls, whoami, exit
docker stats --no-stream       # resource usage snapshot
docker kill lab1               # immediate stop
docker rm lab1                 # removed
docker run -d --name lab2 -p 5001:5000 dockerlab:1.0   # create another from same image: one image → many containers
```

Record each command with its output in the report.

## 2.7.1 Optional persistence demo

```bash
docker volume create labdata
docker run -d --name vol1 -v labdata:/data alpine sh -c "echo hello > /data/f.txt && sleep 300"
docker rm -f vol1
docker run --rm -v labdata:/data alpine cat /data/f.txt     # data survived
```

## 2.8 Step 7: Push to Docker Hub

```bash
docker login -u <your-dockerhub-username>        # use an access token as the password
docker tag dockerlab:1.0 <username>/dockerlab:1.0
docker push <username>/dockerlab:1.0
docker tag dockerlab:1.0 <username>/dockerlab:latest
docker push <username>/dockerlab:latest
```

Verify in browser: hub.docker.com → your repository `dockerlab` → Tags (screenshot).

## 2.9 Step 8: Pull and run on ANOTHER system

On System B (second EC2 instance, a friend's laptop, or after removing local images on the same machine to simulate):

```bash
# simulate a fresh machine on the same system:
docker rm -f $(docker ps -aq) 2>/dev/null
docker rmi <username>/dockerlab:1.0 dockerlab:1.0

# the real test:
docker pull <username>/dockerlab:1.0
docker run -d --name remote1 -p 5000:5000 <username>/dockerlab:1.0
curl http://localhost:5000
```

Expected: the same page appears, with a **different container hostname**. This proves portability: build once, run anywhere.

## 2.10 Step 9: Verify successful deployment ✅

```
☐ docker ps shows STATUS "Up" and PORTS 0.0.0.0:5000->5000/tcp
☐ curl/browser returns the page
☐ /health returns {"status":"healthy"}
☐ docker logs shows request lines (GET / 200)
☐ Image visible on Docker Hub with the tag
☐ Pulled and ran on a second system successfully
```

## 2.11 Cleanup

```bash
docker rm -f $(docker ps -aq)
docker system prune -a
docker logout
# EC2: terminate instances and clean the Security Group
```

---

# 🅲 COMBINED MINI-PROJECT: RUN YOUR DOCKER IMAGE ON EC2 🚀

This links both practicals into one story:

```
1. Launch EC2 (Practical 1 steps) with SG: 22 (My IP), 80 (anywhere)
2. SSH in; install Docker
3. docker run -d -p 80:5000 --name web <username>/dockerlab:1.0
4. Open http://<EC2_PUBLIC_IP>  → your containerized app, served from AWS
5. Terminate EC2 when done
```

User data version (fully automated):

```bash
#!/bin/bash
dnf install -y docker
systemctl enable --now docker
docker run -d --restart unless-stopped -p 80:5000 <username>/dockerlab:1.0
```

---

# 📝 LAB REPORT TEMPLATE

```
Title:        <Practical name>
Aim:          <from syllabus line>
Requirements: AWS account, laptop, Docker, Docker Hub account, etc.
Theory:       (5-8 lines: EC2, VPC, SG, key pair / Docker image, container, registry)
Procedure:    numbered steps with commands
Output:       screenshots + pasted command outputs
Result:       one sentence confirming success
Precautions:  cleanup, key safety, least-open ports
Conclusion + Viva answers
```

---

# 🎯 PRACTICAL VIVA QUESTIONS

**Q1. Why did SSH fail with "Connection timed out"?** Security group not allowing port 22 from your IP, no public IP, or missing internet gateway route.

**Q2. What does the key pair do?** AWS stores the public key in the instance; your private `.pem` proves your identity for SSH.

**Q3. Why 0.0.0.0/0 on port 80 but My IP on port 22?** A public website must be reachable by all; admin access should be restricted.

**Q4. What is user data?** Script run on first boot for automated configuration.

**Q5. Default user names?** `ec2-user` (Amazon Linux), `ubuntu` (Ubuntu), `Administrator` (Windows).

**Q6. Is the default VPC public?** Its subnets route 0.0.0.0/0 to an Internet Gateway, so yes.

**Q7. Why does the Flask app need `host=0.0.0.0` in Docker?** Otherwise it listens only on container loopback and published ports won't reach it.

**Q8. How do you prove the image is portable?** Pull and run it on a different system with identical results.

**Q9. Difference between `docker stop` and `docker kill`?** Graceful SIGTERM with timeout vs immediate SIGKILL.

**Q10. What if `docker push` says "access denied"?** Not logged in or image not named `username/repo:tag`.

**Q11. Why must you terminate instances after the lab?** Running resources continue to incur charges.

**Q12. Static IP for the instance?** Allocate an Elastic IP (release when unused).

---

# ✅ QUICK CHECKLISTS

**Practical 1:** account + MFA + budget → VPC/IGW/route → SG (22 My IP, 80 all) → key pair → launch → SSH → install Nginx → deploy page → test via public IP → cleanup.

**Practical 4:** install Docker → app + requirements → Dockerfile → build → run with `-p` → lifecycle commands → login → tag → push → pull on second system → verify → cleanup.

---

## 🫡 PART 8 STATUS

You now have a complete, runnable lab manual for both Unit I practicals, with troubleshooting and viva prep.

**STOP here.** Last file:

> **🏆 GRAND CONSOLIDATION — Unit I full revision, comparison tables, 30 viva questions, traps, what to write in exams, master sheet**
