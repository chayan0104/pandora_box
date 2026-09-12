# 1) Target Design (BFSI Pattern, Small Resources)

## Environments
- **UAT (On‑Prem, RHEL)**: runs “legacy + UAT stack”
  - Deploy via **AWS CodeDeploy to on‑prem instance**
  - Runtime: **Docker Compose** (microservices) + **WildFly/JBoss (WAR)** (legacy style)
  - DB: **Oracle XE** (Docker) (cost-free, fits your Oracle skill)
- **PROD (AWS)**: runs “cloud-native production”
  - **EKS + Helm + ArgoCD** (GitOps CD)
  - Images stored in **ECR**
  - DB: **RDS** (use **RDS Oracle** if budget permits; otherwise keep DB in Oracle XE on EC2 for demo and still use RDS for a smaller auxiliary DB—details below)

## Shared DevOps Services (AWS)
- **Jenkins (EC2)**: CI + orchestration + approvals + promotion
- **JFrog Artifactory (EC2)**: artifact repo + Docker registry for “pre‑prod/UAT build outputs”
- **SonarQube (EC2 or Docker Compose)**: SAST/quality gate
- **Observability**
  - **CloudWatch + SNS** for AWS infra alerts
  - **Prometheus + Grafana** for Kubernetes/app metrics
  - **Dynatrace** for APM (trial/tenant)

## BFSI Controls you will demonstrate
- Environment segregation (UAT on-prem, PROD on AWS)
- Artifact immutability (build once, promote same tag)
- Approval gates (CAB-like “UAT signoff → PROD promote”)
- Audit trail (Artifactory build-info + Jenkins logs + Git history + CloudWatch alarms)

---

# 2) Repo Layout (Keep it Interview-Friendly)

Use 2 repos (this helps you explain CI vs CD cleanly):

### Repo A: `app-platform` (code + build + UAT deploy package)
```
app-platform/
  services/              # open-source microservices code
  legacy-war/            # one Java module packaged as WAR for WildFly/WebLogic style deploy
  docker/                # Dockerfiles per service
  compose/uat/           # docker-compose.yaml for UAT
  codedeploy/uat/        # appspec.yml + scripts for on-prem deployment
  jenkins/Jenkinsfile
  terraform/             # AWS infra (Jenkins, Artifactory, EKS, etc.)
  scripts/               # bash/python helpers, DB deploy scripts
```

### Repo B: `gitops-prod` (PROD Kubernetes desired state)
```
gitops-prod/
  helm/yourapp/          # Helm chart (or references to chart path)
  envs/prod/values.yaml  # only prod values
  argocd/app.yaml        # ArgoCD Application
```

**Why this matters in interviews:**  
Jenkins does CI + promotion decision; **ArgoCD owns PROD deployment state**. That’s modern DevOps.

---

# 3) AWS (PROD + Tooling) — Terraform Roadmap

You will create with Terraform (HCL), using remote state in S3.

## 3.1 Terraform remote state (S3 + DynamoDB lock)
- S3 bucket for state
- DynamoDB table for state locking

## 3.2 Network (VPC)
- VPC, 2 public subnets, 2 private subnets, IGW, NAT (single NAT to keep cost low)
- Security groups for Jenkins, Artifactory, EKS nodes

## 3.3 Compute (EC2)
- **EC2 Jenkins** (t3.medium)
- **EC2 Artifactory OSS** (t3.medium; Artifactory likes RAM)
- Optional **EC2 SonarQube** (or run SonarQube via Docker Compose on Jenkins EC2)

## 3.4 EKS
- EKS cluster (small)
- Managed node group: 2× `t3.medium` (min=2, max=4)

## 3.5 ECR
- ECR repositories: one per microservice (recommended)

## 3.6 RDS
Two practical options:

**Option A (Most aligned to BFSI + your Oracle skill):**
- **RDS Oracle** (create, test, destroy quickly to control cost)

**Option B (Cost-minimizing fallback):**
- Use **RDS (any small engine you choose)** only to demonstrate RDS provisioning/backup/alarms
- Keep the app’s main DB as **Oracle XE** (on-prem UAT + optionally EC2 for prod demo)
> In interviews, you can still say “Prod uses managed RDS in real; I used XE for cost control in the lab.”

## 3.7 SNS + CloudWatch
- SNS topic + email subscription
- CloudWatch alarms (EC2 CPU high, EKS node CPU, RDS CPU/storage)

**Troubleshooting (common Terraform issues)**
- `DependencyViolation` on destroy → destroy in reverse order: EKS/RDS/EC2 → VPC
- EKS nodes `NotReady` → missing IAM policies on node role or SG rules
- RDS subnet group errors → ensure DB subnets are private subnets

---

# 4) On‑Prem UAT (RHEL) Setup (No extra tools)

You need 1–2 RHEL servers/VMs.

## 4.1 Base OS hardening-ish (what to say in BFSI interviews)
- Dedicated user accounts
- Firewall only for required ports
- Keep configs under `/opt/uat/` with restricted permissions
- Keep deployment logs under `/var/log/uat-deploy/`

## 4.2 Install Docker + Docker Compose (UAT runtime)
On RHEL:
```bash
sudo yum install -y docker
sudo systemctl enable --now docker
# Docker Compose v2 is "docker compose" plugin; install if missing from distro
```

## 4.3 Oracle XE on UAT (Docker)
```bash
docker run -d --name oracle-xe \
  -p 1521:1521 \
  -e ORACLE_PASSWORD=OraclePwd123 \
  gvenzl/oracle-xe:21-slim
```

## 4.4 WildFly/JBoss (legacy deploy target)
Install Java + WildFly and run it as a service. Your CodeDeploy deployment will drop a WAR and restart/reload.

**Why:** BFSI has legacy Java EE appservers; your resume includes WildFly/JBoss and WebLogic.

> WebLogic: if you have access to developer edition, repeat similar deploy steps. If not, you still explain the process (WAR staging, restart domains, datasource/JNDI configs, WLST automation).

---

# 5) Artifactory (JFrog) Setup (AWS EC2)

## 5.1 Create these repositories in Artifactory
- **maven-local** (for jar/war)
- **docker-uat-local** (UAT Docker images)
- (optional) **helm-local** (Helm chart hosting; or store Helm chart in Git)

**Why Artifactory in this project**
- BFSI teams want **a controlled artifact system** (retention, promotion, checksums).
- Store **build outputs** centrally; UAT pulls from Artifactory; PROD uses ECR.

---

# 6) Jenkins Setup (AWS EC2) + Required Integrations

## 6.1 Jenkins plugins
- Pipeline
- Artifactory
- Kubernetes CLI (or just AWS CLI + kubectl installed on Jenkins)
- AWS Steps (optional)

## 6.2 Credentials in Jenkins
- `artifactory-creds`
- `aws-creds` (or attach IAM role to Jenkins EC2)
- `sonarqube-token`
- Git PAT for pushing to `gitops-prod`

**Why Jenkins stays even with ArgoCD**
- Jenkins = CI, testing, packaging, and **promotion control + approvals**
- ArgoCD = PROD CD and drift management

This separation is a modern, defensible story.

---

# 7) CI/CD Flow (This is the Core “Explainable” BFSI Pipeline)

## 7.1 CI: Build → Quality → Package → Publish to Artifactory

### Build outputs
- Microservices: Docker images
- Legacy module: WAR artifact

### Minimum CI stages (Jenkinsfile)
1. Checkout
2. Build + unit test
3. **SonarQube** scan (SAST/quality gate)
4. Publish WAR/JAR to **Artifactory (maven-local)**
5. Build Docker images
6. Push Docker images to **Artifactory docker-uat-local**
7. (Security gates you listed)
   - **Trivy** image scan
   - **Checkov** Terraform scan
   - OWASP (DAST) against UAT URL after deploy
   - Checkmarx/Fortify/Black Duck: integrate if you have servers/licenses; otherwise keep pipeline stubs and explain where they plug in

### Tool usage mapping (what you will say)
- **SonarQube**: SAST + code quality baseline + enforce coverage/bugs
- **Trivy**: container vulnerability scan before promotion
- **Checkov**: IaC scanning on Terraform plans
- **OWASP**: DAST baseline scan on running UAT endpoint

## 7.2 UAT Deploy (On‑Prem) via AWS CodeDeploy

This is a strong differentiator: **CodeDeploy supports on‑prem instances**.

### Steps
1. Install CodeDeploy agent on UAT RHEL
2. Register on‑prem instance in CodeDeploy (AWS CLI)
3. Create CodeDeploy Application + Deployment Group for UAT
4. Jenkins packages the UAT bundle (compose + scripts + WAR)
5. Jenkins uploads bundle to S3
6. Jenkins triggers CodeDeploy deployment
7. Post-deploy smoke test + optional OWASP DAST baseline

### What CodeDeploy does on the UAT server
- Pull new Docker images from Artifactory
- `docker compose up -d` for microservices
- Copy WAR to WildFly deployments directory
- Restart/reload WildFly

**Why this is BFSI-real**
- On-prem environments often still use agent-based deployment with strict control.

## 7.3 Promotion Gate (BFSI Change Control)
After UAT is green, enforce a manual approval:

- Jenkins `input` step: “Approve PROD release?”
- Capture approver username in build logs (audit)

## 7.4 PROD Deploy (AWS EKS) via ArgoCD (GitOps)

### Promotion mechanics (clean and explainable)
- Jenkins does **promotion**, not “deploy”:
  1. Pull image from Artifactory
  2. Retag and push to **ECR**
  3. Update `gitops-prod` values.yaml with the new ECR tag
  4. Commit/push to Git
- ArgoCD detects Git change and syncs to EKS

**Why ArgoCD**
- Drift detection + self-heal + auditability via Git history
- In interviews: “We prevent manual kubectl changes in PROD; Git is the source of truth.”

---

# 8) EKS (PROD) Setup Details (Helm + ArgoCD)

## 8.1 Install ArgoCD on EKS
(ArgoCD is the only “extra” not in your resume, but you required it.)

## 8.2 Deploy microservices via Helm chart
- One Helm chart with:
  - Deployments
  - Services
  - ConfigMaps
  - Secrets (K8s Secrets; store values securely in Jenkins credentials and inject at deploy-time, or keep encrypted outside Git)

## 8.3 “High-load path” on small AWS
You simulate BFSI load behavior using:
- **HPA** (Kubernetes feature; no new tool)
- Proper CPU/memory requests/limits
- Python-based load generator (uses your Python skill; no extra tool)

### HPA example
- Set requests/limits in Helm values
- Create HPA for key services

### Load test (Python)
Write a simple asyncio load script to hit the PROD ALB/Service endpoint and watch:
- `kubectl get hpa -w`
- Prometheus/Grafana dashboards
- CloudWatch alarms (if nodes spike)

**What you’ll explain**
- “Even with small nodes, I validated scaling logic end-to-end: HPA reacts to CPU, replicas scale, service stays available.”

---

# 9) Observability (Use Everything You Listed)

## 9.1 CloudWatch + SNS (AWS infra)
- CloudWatch alarms → SNS email
- EKS control plane logs → CloudWatch
- EC2 CPU/mem alarms (mem via CloudWatch agent if you choose; if not, just CPU)

## 9.2 Prometheus + Grafana (K8s + app)
- Install via Helm on EKS
- Dashboards:
  - Node CPU/mem
  - Pod restarts
  - Request latency if your app exports metrics (many OSS apps do)

## 9.3 Dynatrace
- PROD: Dynatrace K8s integration/OneAgent to EKS nodes (Dynatrace docs)
- UAT on-prem: OneAgent on RHEL server
- Use Dynatrace for:
  - service flow
  - response time
  - error rate
  - infra utilization

**Interview-ready explanation**
- Prometheus/Grafana = metrics + dashboards (platform)
- Dynatrace = APM deep visibility (transactions, service flow, RCA acceleration)
- CloudWatch = AWS-native infra + alerting to SNS

---

# 10) Database + Release Automation (Use Your Oracle/SQL skills)

## UAT DB
- Oracle XE in Docker
- Jenkins runs migration scripts before app deploy (or as a dedicated stage)

## PROD DB
- If using RDS Oracle: run migration scripts against RDS endpoint

## DB deploy automation (BFSI-grade idea)
Use your existing pattern:
- SQL scripts in Git
- Jenkins stage validates ordering (DDL/DML/GRANT)
- Execute with Java/Python script
- Log applied scripts with timestamp + build number (audit trail)

Add a “DB approval gate” before PROD DDL.

---

# 11) Troubleshooting Runbook (Must-Have)

## 11.1 Jenkins
- Docker permission denied:
  - `sudo usermod -aG docker jenkins && sudo systemctl restart jenkins`
- AWS auth failures:
  - prefer EC2 instance profile/IAM role; otherwise rotate access keys
- Pipeline stuck at approval:
  - add timeout around `input {}`

## 11.2 Artifactory Docker push/pull
- Auth fails:
  - verify correct docker registry URL from “Set Me Up”
- Docker insecure registry (if HTTP):
  - configure docker daemon `insecure-registries` (UAT server + Jenkins server)

## 11.3 CodeDeploy to on-prem
- Agent not running:
  - `sudo systemctl status codedeploy-agent`
- Deployment fails:
  - check `/var/log/aws/codedeploy-agent/codedeploy-agent.log`
- Hooks fail:
  - check scripts permissions `chmod +x`
  - validate working directory paths in `appspec.yml`

## 11.4 EKS issues
- Pods `ImagePullBackOff`:
  - ECR auth via node role; ensure node IAM has ECR read permissions
- HPA not scaling:
  - install metrics server; ensure CPU requests are set
- ArgoCD OutOfSync:
  - check rendered Helm manifests; fix values; re-sync

## 11.5 Observability
- No Prometheus data:
  - check `kubectl get pods -n monitoring`
  - check scrape targets
- CloudWatch alarm not firing:
  - verify dimensions and evaluation periods
- Dynatrace not showing services:
  - validate OneAgent injection/host install and network to Dynatrace tenant

---

# 12) What You Will Be Able to Explain (Tool-by-Tool “Why/How”)

- **Terraform**: reproducible, environment-separated AWS infra (VPC/EKS/EC2/ECR/RDS/S3/SNS)
- **Jenkins (Groovy)**: orchestrates CI + UAT deploy + approval + promotion to PROD
- **Artifactory**: single source for build artifacts + UAT registry; traceability and retention
- **AWS CodeDeploy**: agent-based deployment to **on-prem UAT** (very BFSI-real), rollback-capable
- **Docker + Compose**: on-prem UAT runtime; simple controlled environment
- **Kubernetes (EKS) + Helm**: PROD orchestration + packaging + environment values separation
- **ArgoCD**: GitOps CD + drift control + audit via Git (modern DevOps)
- **ECR**: PROD registry; EKS pulls from ECR securely
- **RDS + Oracle/SQL**: managed DB story + DB change automation + tuning narrative
- **SonarQube/Trivy/Checkov/OWASP (+ Checkmarx/Fortify/Black Duck)**: DevSecOps gates and how they reduce production incidents
- **Prometheus/Grafana/Dynatrace/CloudWatch**: layered observability and faster RCA
- **RHEL/on-prem**: ops discipline, patching mindset, service mgmt, troubleshooting
- **Incident management/RCA**: runbooks + rollback paths (CodeDeploy rollback, ArgoCD rollback via git revert)

---

## One clarifying question (so I can make this truly “end-to-end” with exact files)
For your open-source app choice, will you use:
1) **Mostly Java** microservices (best to align with WildFly/WebLogic), or  
2) **Polyglot** microservices (fine, but legacy WAR will be a separate small Java module)?

If you answer 1 or 2, I’ll give you:
- a concrete `Jenkinsfile` (UAT + PROD promotion)
- `appspec.yml` + CodeDeploy hook scripts for Compose + WildFly WAR
- Helm chart skeleton + ArgoCD Application YAML
- Terraform module skeleton for your exact component list (EC2/EKS/ECR/RDS/S3/SNS/CloudWatch)
