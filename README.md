# Tetris — DevSecOps CI/CD Pipeline

A React-based Tetris application with an automated CI/CD pipeline built using Jenkins. The pipeline performs code quality analysis, dependency and vulnerability scanning, builds a Docker image, and deploys the application to a local Docker container.

![Tetris DevSecOps Pipeline](docs/pipeline-flow.png)

## Overview

This project demonstrates how application code moves from source control to a running container through an automated Jenkins pipeline, with code quality and security checks integrated before deployment.

## Pipeline Flow

1. **Checkout Source Code** — Jenkins checks out the repository.
2. **Install Dependencies** — Installs Node.js packages cleanly using `npm ci`.
3. **SonarQube Analysis** — Analyzes code quality, bugs, and code smells.
4. **Quality Gate** — Checks the SonarQube result and stops the pipeline if the gate fails.
5. **OWASP Dependency-Check** — Scans third-party project dependencies for known CVEs.
6. **Trivy Filesystem Scan** — Scans the repository filesystem for vulnerable packages.
7. **Build Docker Image** — Builds a multi-stage production Docker image (`tetris-app:local`).
8. **Deploy** — Replaces the `tetris-app-local` container and serves the app via Nginx on port 3000.

## Technology Stack

| Tool | Purpose |
|---|---|
| Jenkins | CI/CD pipeline orchestration |
| Git / GitHub | Source control |
| Node.js / npm | Application build and package management |
| SonarQube | Code quality analysis and Quality Gate enforcement |
| OWASP Dependency-Check | Software Composition Analysis (SCA) for known CVEs |
| Trivy | Filesystem and package vulnerability scanning |
| Docker | Multi-stage image build and containerized deployment |
| React | Tetris game application |
| Nginx | Production web server hosting the compiled React app |

## Application Access

Once deployed, the services are available at:

- **Tetris app:** http://localhost:3000
- **Jenkins:** http://localhost:8080
- **SonarQube:** http://localhost:9000

---

## Getting Started: Setting Up for Yourself

If you clone this repository to run the pipeline in your own environment, follow these configuration steps:

### 1. Clone the Repository

```bash
git clone https://github.com/harshaarmin/tetris-devsecops.git
cd tetris-devsecops
```

### 2. Jenkins Prerequisites & Plugins

Ensure the following plugins are installed in Jenkins (**Manage Jenkins** → **Plugins** → **Available Plugins**):
- **Pipeline**
- **NodeJS Plugin**
- **SonarQube Scanner for Jenkins**
- **OWASP Dependency-Check Plugin**
- **Docker** and **Docker Pipeline**

### 3. Global Tool Configuration

Configure these tools in Jenkins under **Manage Jenkins** → **Tools**:

| Tool | Name in Jenkins | Version / Notes |
|---|---|---|
| **NodeJS** | `nodejs` | NodeJS 18.x or 20.x |
| **SonarQube Scanner** | `SonarScanner` | Install automatically |
| **Dependency-Check** | `DP-Check` | Install automatically (latest version) |

### 4. Security & Credentials Setup

The pipeline expects the following credentials and integrations:

1. **SonarQube Server Configuration:**
   - In SonarQube, go to **My Account** → **Security** → **Generate Token** (type: User Token).
   - In Jenkins, go to **Manage Jenkins** → **System** → **SonarQube installations**:
     - Name: `SonarQube`
     - Server URL: `http://localhost:9000` (or your SonarQube host)
     - Server authentication token: Add your SonarQube token as **Secret text**.

2. **SonarQube Webhook (Required for Quality Gate):**
   - In SonarQube, navigate to **Administration** → **Configuration** → **Webhooks**.
   - Create a webhook:
     - Name: `Jenkins`
     - URL: `http://<jenkins-ip-or-host>:8080/sonarqube-webhook/`
   > **Note:** Without this webhook, the Quality Gate stage will wait until its 5-minute timeout and abort.

3. **NVD API Key (`nvd-api-key`):**
   - Request a free API key from the [National Vulnerability Database (NVD)](https://nvd.nist.gov/developers/request-an-api-key) to avoid strict rate limiting.
   - In Jenkins (**Manage Jenkins** → **Credentials** → **System** → **Global credentials**):
     - Kind: **Secret text**
     - Secret: `<your-nvd-api-key>`
     - ID: `nvd-api-key`

4. **Trivy CLI & Docker Access on Jenkins Agent:**
   - Install Trivy on the Jenkins host or agent machine:
     ```bash
     # macOS (Homebrew)
     brew install trivy

     # Debian/Ubuntu
     sudo apt-get install wget apt-transport-https gnupg lsb-release -y
     wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
     echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
     sudo apt-get update && sudo apt-get install trivy -y
     ```
   - Ensure the `jenkins` user has permission to run Docker commands:
     ```bash
     sudo usermod -aG docker jenkins
     sudo systemctl restart jenkins
     ```

---

## Run the Pipeline

1. In Jenkins, create a new **Pipeline** job (e.g. `tetris-devsecops`).
2. Under **Pipeline Definition**, select **Pipeline script from SCM**:
   - **SCM**: Git
   - **Repository URL**: `https://github.com/harshaarmin/tetris-devsecops.git`
   - **Branch Specifier**: `*/master`
   - **Script Path**: `Tetris-V1/Jenkinsfile`
3. Click **Save** and select **Build Now**.
4. Monitor the build logs across all 8 stages.
5. When complete, navigate to http://localhost:3000 to play Tetris!

---

## Running Locally (Without Jenkins)

If you just want to run the application or Docker container directly:

### Run with Node.js
```bash
cd Tetris-V1
npm install
npm start
# Opens at http://localhost:3000
```

### Run with Docker
```bash
cd Tetris-V1
docker build -t tetris-app:local .
docker run -d --name tetris-app-local -p 3000:80 tetris-app:local
# Available at http://localhost:3000
```

---

## Project Structure

```text
.
├── Tetris-V1/
│   ├── Jenkinsfile                       # 8-stage DevSecOps CI/CD pipeline
│   ├── Dockerfile                        # Multi-stage build (Node 22 -> Nginx)
│   ├── dependency-check-suppressions.xml # OWASP false positive suppressions
│   ├── package.json
│   ├── package-lock.json
│   └── src/
├── docs/
│   └── pipeline-flow.png                 # Architecture and pipeline flow diagram
└── README.md
```

## Notes & Security Best Practices

- The pipeline enforces a strict Quality Gate: any blocker issues or failed quality metrics stop the build immediately.
- Dependencies are scanned against the NVD database; non-actionable false positives are documented in `dependency-check-suppressions.xml`.
- Sensitive credentials (NVD key, SonarQube tokens) are injected securely via Jenkins credentials and never stored in the repository.

## Future Improvements

- Push verified Docker images to a container registry (Docker Hub / AWS ECR).
- Integrate Trivy container image scanning after the Docker build step.
- Add automated unit tests and test coverage reporting.
- Deploy to Kubernetes (EKS / Minikube) using GitOps (ArgoCD).
