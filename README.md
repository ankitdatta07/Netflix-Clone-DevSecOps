# 🎬 Netflix Clone — DevSecOps CI/CD Pipeline

![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)
![SonarQube](https://img.shields.io/badge/SonarQube-4E9BCD?style=for-the-badge&logo=sonarqube&logoColor=white)
![Trivy](https://img.shields.io/badge/Trivy-1904DA?style=for-the-badge&logo=aquasecurity&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-232F3E?style=for-the-badge&logo=amazonaws&logoColor=white)

> A production-grade CI/CD pipeline for a Netflix Clone application, built on a foundation of security-first engineering. This project walks through every stage from source code to a live Kubernetes deployment — with automated security scanning baked in at each gate, not bolted on at the end.

---

## 📐 Architecture Overview

The pipeline is orchestrated entirely by Jenkins, which acts as the central controller managing every stage from code checkout to Kubernetes deployment. I deliberately followed a **"Shift-Left Security"** philosophy — meaning security checks (SonarQube static analysis, OWASP Dependency-Check, and Trivy scans) are placed *before* the Docker build and deployment stages. If the code doesn't pass the security gate, the pipeline fails fast and the deployment never happens.

The high-level flow is:

```
GitHub (Source) → Jenkins (Orchestrator)
    ├── Static Code Analysis  →  SonarQube Quality Gate
    ├── Dependency Audit      →  OWASP Dependency-Check
    ├── Build & Push          →  Docker → Docker Hub (dock279/netflix)
    ├── Image Scan            →  Trivy (Filesystem + Image)
    └── Deploy                →  Kubernetes (NodePort: 30009)
                                     ↓
                               Monitoring: Prometheus + Node Exporter → Grafana
```

> 
> ![Architecture Diagram](Docs/arch.png)
> 

The infrastructure runs on AWS EC2. The Jenkins controller, SonarQube instance, and Kubernetes master (`K8s-Master`) were provisioned on separate nodes to avoid resource contention — a lesson I learned the hard way when SonarQube's embedded database started competing with Jenkins' build workers for heap space.

---

## 🛠️ Tech Stack & Engineering Rationale

| Tool | Role | Why I Chose It |
|:---|:---|:---|
| **Jenkins** | CI/CD Orchestrator | I chose Jenkins over managed solutions (like GitHub Actions) specifically to gain experience with self-hosted automation controllers and Groovy-based pipeline-as-code. Writing a `Jenkinsfile` teaches you what the abstractions are hiding. |
| **SonarQube** | Static Code Analysis (SAST) | SonarQube gave me a quality gate I could fail the build on. Rather than just generating a report, it integrates as a pipeline-gate — the build proceeds only if the analysis passes. |
| **OWASP Dependency-Check** | Vulnerable Library Scanning | Supply chain attacks are a real vector. OWASP DC scans the project's NPM dependencies against the NVD (National Vulnerability Database) and surfaces CVEs early, before they're ever packaged into an image. |
| **Trivy** | Container & Filesystem Scanning | I ran Trivy twice: once on the filesystem (pre-build) and once on the final Docker image. This two-pass approach catches vulnerabilities introduced by the base OS layers in the Docker image itself, not just the application code. |
| **Docker** | Containerization | Containerizing the app guarantees environment parity — "works on my machine" stops being a valid excuse. The image built on the Jenkins agent behaves identically in Kubernetes. |
| **Docker Hub** `dock279/netflix` | Image Registry | Chosen for simplicity and its free tier. The image is tagged `latest` and pulled directly by the Kubernetes manifest during deployment. |
| **Kubernetes** | Container Orchestration | K8s handles rolling deployments and ensures the app stays available even if a pod is rescheduled. Using a `NodePort` service exposed the app on port `30009` for external access without requiring a cloud load balancer. |
| **Prometheus + Node Exporter** | Metrics Collection | Prometheus scrapes system-level metrics (CPU, memory, I/O) from Node Exporter running on the K8s master node (`localhost:9100`). This gives me visibility into the infrastructure health post-deployment. |
| **Grafana** | Observability Dashboard | I imported the "Node Exporter Full" dashboard to visualize the metrics Prometheus collected. Seeing RAM usage at 91.6% during a heavy pipeline run was a concrete reminder of why right-sizing instances matters. |

---

## 🚀 The Build Journey

### Step 1: Jenkins Environment & Plugin Setup

Setting up Jenkins was the most time-consuming part of the project — not because it's hard, but because getting the plugins right requires patience. I configured SMTP-based email notifications so that every build result (pass or fail) fires off an alert. This matters because in a real team, a broken build that nobody notices is worse than a build that was never triggered.

The key plugins I configured: Blue Ocean (for the stage view), SonarQube Scanner, OWASP Dependency-Check, Docker Pipeline, and the Kubernetes CLI plugin. Managing plugin version compatibility was the first real debugging exercise of the project.
 
> 
> ![Jenkins Setup](Docs/jenkins.png)
> 

---

### Step 2: Security & Quality Gates

This stage is the heart of the "shift-left" approach. Before a single line of application code is compiled or packaged, two independent security checks run:

**SonarQube Static Analysis:** I configured a SonarQube project (`Netflix`) manually within the SonarQube Community Edition (v9.9.8). The Jenkins pipeline integrates via a server token, and after the scan, Jenkins waits for the Quality Gate webhook to confirm a `Passed` status before proceeding. On the final run (Build #22), the Netflix project passed with an A-rating on Bugs, Vulnerabilities, and Code Smells — with 0 new issues on new code since April 12, 2026.

**OWASP Dependency-Check:** This stage scans the project's `package.json` dependencies and checks each library's CVE history. The Dependency-Check trend graph in Jenkins shows consistently ~8 High-severity findings across builds — these are known vulnerabilities in the project's third-party dependencies, which I documented rather than suppressed.

> 
> ![SonarQube Quality Gate](Docs/Sonarqube2.png)
> ![SonarQube Quality Gate](Docs/sonarqube3.png)
> 

---

### Step 3: Containerization with Docker

Once the code cleared the security gates, I built the Docker image and pushed it to Docker Hub (`dock279/netflix:latest`). The image was built from a custom Dockerfile targeting the Netflix Clone's TypeScript/Node.js codebase (3.2k lines of TypeScript as reported by SonarQube).

I kept the image lean by ensuring the build stage only copies what's needed for production — avoiding the common mistake of shipping `devDependencies` and build toolchains into the final image. The image has been pulled 26 times from Docker Hub, confirming it's being successfully retrieved by the Kubernetes deployment manifest.

> 
> ![Docker Hub](Docs/Dockerhub.png)
>

---

### Step 4: Image Scanning with Trivy

After the Docker image was pushed, Trivy scanned it for OS-level CVEs — vulnerabilities in the base image's system packages that have nothing to do with the application code itself. Running this scan *after* the build (rather than just a filesystem scan) catches a different class of vulnerability: anything introduced by the `FROM` base image in the Dockerfile.

I configured Trivy in the pipeline to report findings without hard-failing the build on this stage (a deliberate trade-off documented in the next section), so the pipeline would continue to deployment while the report was archived as a build artifact.
 
> 
> ![Final Pipeline Scan](Docs/Jenkins-pipeline.png)
> 

---

### Step 5: Kubernetes Deployment

The final stage applied a Kubernetes manifest to a single-node cluster (`K8s-Master`, IP: `172-31-45-218`). The manifest defines a `Deployment` and a `NodePort` Service. The deployment spun up a pod (`netflix-app-5b9c665f4d`) exposing port 80 internally and mapping it to `NodePort 30009` for external traffic.

The app was accessible at `54.149.242.216:30009/browse`, serving the full Netflix Clone UI including the TMDB-powered movie catalog (Popular Movies, Top Rated, Now Playing). At the time of the screenshots, I observed two pods in `Pending` state due to resource constraints on the single-node cluster — this is captured honestly as a known limitation of a single-node K8s setup.

> ![K8s-pods](Docs/Kubernetes.png)
> ![App Live](Docs/Netflix.png)
> ![App Live](Docs/Netflix2.png)
> 

---

## 📊 Monitoring & Observability

After deployment, I wired up a monitoring stack to observe the infrastructure:

- **Prometheus** (`localhost:9090`) scrapes metrics from both itself and Node Exporter (`localhost:9100`), both confirmed healthy in the Prometheus targets dashboard (last scrape latency: 16.5ms for node_exporter).
- **Grafana** (`54.245.12.254:3000`) hosts two dashboards: "Node Exporter Full" (tagged `linux`) and "Jenkins: Performance and Health Overview".
- The Node Exporter Full dashboard during an active pipeline run showed CPU at 2.9%, RAM utilization at 91.6%, and SWAP at 73.6% — clear indicators that the instance was under pressure during heavy build stages.

> 
> ![Grafana Dashboard](Docs/Grafana.png)
> ![Monitoring](Docs/Monitoring.png)
> ![Prometheus-Targets](Docs/prometheus-targets.png)
> 

---

## 🔗 Engineering Trade-offs & Lessons Learned

This section is the most honest part of the documentation — the decisions I'd revisit and the things I'd automate next.

**1. Trivy Gate Severity Threshold**
I configured Trivy to report without blocking the build on image scan findings. In a production environment this is the wrong call — a proper pipeline should fail on `CRITICAL` severity CVEs. I made this trade-off to complete the end-to-end deployment flow for the project, but I documented the findings. The right fix is a `--exit-code 1 --severity CRITICAL` flag that turns Trivy into a hard gate.

**2. SonarQube Embedded Database**
SonarQube displayed a warning throughout the project: *"Embedded database should be used for evaluation purposes only."* This is a real constraint — the H2 embedded DB doesn't support concurrent access at scale and can't be migrated to another engine. For anything beyond a personal project, I'd configure an external PostgreSQL instance from day one.

**3. Single-Node Kubernetes**
Running a single-node K8s cluster meant that when the node itself was under pressure (RAM at 91.6%), pods entered `Pending` state because the scheduler had nowhere to place them. The fix is a proper multi-node cluster — either a managed service (EKS/GKE) or dedicated worker nodes.

**4. Cost & Instance Lifecycle**
I terminated EC2 instances after each significant testing session to avoid ongoing charges. This is realistic for a student project, but it highlights why **Terraform** is critical. Without IaC, tearing down and rebuilding the environment is a manual, error-prone process. My next step is to define the entire infrastructure in Terraform so I can `apply` and `destroy` with a single command.

**5. What I'd Automate Next**

- **Terraform** for infra provisioning (EC2, Security Groups, IAM roles)
- **Helm Charts** for the Kubernetes deployment instead of raw manifests — Helm makes version management and rollbacks tractable
- **Slack/PagerDuty webhook** integration for Grafana alerts on RAM/CPU threshold breaches
- **Automated SonarQube Quality Profile** tuning via the SonarQube API so gate criteria are version-controlled alongside the code

---

## 📈 Final Reflection

Before this project, CI/CD was an abstract concept. After building it from scratch — configuring Jenkins plugins, debugging why the SonarQube webhook wasn't firing, watching a Trivy scan surface a CVE in a base image I didn't write — it became a concrete engineering discipline.

The most important thing I internalized is that **a CI/CD pipeline is only as useful as the gates it enforces**. A pipeline that always passes teaches you nothing and protects nothing. The deliberate decision to make SonarQube and OWASP hard gates, rather than optional reports, is what gives this pipeline its actual value.

I'm entering the industry with hands-on experience in the full DevSecOps loop: source → static analysis → dependency audit → container build → image scan → orchestrated deployment → infrastructure monitoring. That end-to-end ownership is what I was building toward.

---

## 🔗 References & Tools Used

- Jenkins — [jenkins.io](https://www.jenkins.io)
- SonarQube Community Edition v9.9.8 — [sonarqube.org](https://www.sonarqube.org)
- OWASP Dependency-Check — [owasp.org](https://owasp.org/www-project-dependency-check/)
- Trivy by Aqua Security — [aquasecurity.github.io/trivy](https://aquasecurity.github.io/trivy)
- Docker Hub (`dock279/netflix`) — [hub.docker.com](https://hub.docker.com)
- Prometheus — [prometheus.io](https://prometheus.io)
- Grafana — [grafana.com](https://grafana.com)
- Kubernetes — [kubernetes.io](https://kubernetes.io)

---

*Deployed on AWS EC2 &nbsp;|&nbsp; K8s Master Node: `ip-172-31-45-218` &nbsp;|&nbsp; App URL: `54.149.242.216:30009/browse` &nbsp;|&nbsp; Last successful build: **#22***
