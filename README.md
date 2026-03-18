# NodeGoat + Jenkins + Socket Security Demo

Demonstrates Socket's Tier 1 Reachability analysis integrated into a local Jenkins CI pipeline scanning [OWASP NodeGoat](https://github.com/OWASP/NodeGoat) — an intentionally vulnerable Node.js app.

**Example scan results:** https://socket.dev/dashboard/david-s-github/nodegoat-jenkins-socket-demo/main/

---

## What This Shows

- Jenkins pipeline running Socket's Python CLI (`socketcli`) on a real vulnerable app
- **Tier 1 Reachability**: Socket analyzes which CVEs are actually reachable from the codebase's entry points
- Results confirmed from the demo run:
  - 107 total vulnerabilities across the dependency tree
  - **13 reachable** (actually exploitable from this code)
  - **86 unreachable** (present in dep tree but code paths never invoked)
  - **80% noise reduction** — only 13 vulnerabilities actually need attention

---

## Prerequisites

- Docker + Docker Compose
- A Socket API key for your org ([get one here](https://socket.dev))
- Node.js (for local testing without Jenkins)

---

## Reproduce the Demo

### 1. Fork and clone the repo

Fork this repo to your own GitHub account, then clone it:

```bash
git clone https://github.com/YOUR_GITHUB_USER/nodegoat-jenkins-socket-demo.git
cd nodegoat-jenkins-socket-demo
```

### 2. Set environment variables

```bash
# Required: your Socket API key
export SOCKET_API_KEY=sktsec_your_key_here

# Required: point Jenkins at your fork
export GITHUB_REPO_URL=https://github.com/YOUR_GITHUB_USER/nodegoat-jenkins-socket-demo.git

# Optional: customize the repo name in Socket dashboard (defaults to nodegoat-jenkins-socket-demo)
export REPO_NAME=nodegoat-jenkins-socket-demo

# Optional: add a second project (must have its own Jenkinsfile with socketcli)
export GITHUB_REPO_URL_2=https://github.com/YOUR_GITHUB_USER/another-repo.git
export REPO_NAME_2=another-repo
```

### 3. Start the stack (Jenkins + NodeGoat + MongoDB)

```bash
docker compose up -d
```

This builds a custom Jenkins image that includes:
- Python 3 + `socketsecurity` CLI
- Node.js 20 (required for reachability analysis via `@coana-tech/cli`)
- `uv` (Python env manager, also required for reachability)

Jenkins will be available at **http://localhost:8090** (user: `admin`, password: `admin`).

### 4. Trigger a scan

The pipeline job `nodegoat-socket-scan` is auto-created by Jenkins Configuration as Code (JCasC). Trigger it:

```bash
# Get CSRF crumb
curl -s -u admin:admin -c /tmp/j-cookie.txt \
  "http://localhost:8090/crumbIssuer/api/json" -o /tmp/crumb.json

# Trigger build
curl -s -u admin:admin -b /tmp/j-cookie.txt -X POST \
  -H "$(python3 -c "import json; d=json.load(open('/tmp/crumb.json')); print(d['crumbRequestField']+': '+d['crumb'])")" \
  "http://localhost:8090/job/nodegoat-socket-scan/build"
```

Or just click **Build Now** in the Jenkins UI.

### 5. View results

- **Jenkins console**: http://localhost:8090/job/nodegoat-socket-scan/lastBuild/consoleText
- **Socket dashboard**: https://socket.dev/dashboard/your-org/nodegoat-jenkins-socket-demo/main/

---

## How It Works

### Jenkinsfile

```groovy
pipeline {
    agent any
    environment {
        SOCKET_SECURITY_API_TOKEN = credentials('socket-api-key')
    }
    stages {
        stage('Checkout')            { steps { checkout scm } }
        stage('Install Dependencies') { steps { sh 'npm install --production' } }
        stage('Socket Security Scan') {
            steps {
                sh '''
                    socketcli \
                        --target-path . \
                        --repo nodegoat-jenkins-socket-demo \
                        --default-branch \
                        --reach \
                        --reach-ecosystems npm \
                        --disable-blocking \
                        --integration api
                '''
            }
        }
    }
}
```

### Multiple Projects

All Jenkinsfiles live in this repo. Each project gets its own file:

| File | Purpose |
|------|---------|
| `Jenkinsfile` | Scans this repo (NodeGoat) |
| `Jenkinsfile.2` | Clones and scans the repo from `GITHUB_REPO_URL_2` |

To add a third project, copy `Jenkinsfile.2` to `Jenkinsfile.3`, add `GITHUB_REPO_URL_3`/`REPO_NAME_3` env vars, and add a matching job block in `jenkins/casc/jenkins.yaml`.

### Jenkins Configuration as Code (JCasC)

`jenkins/casc/jenkins.yaml` pre-configures:
- Admin credentials (admin/admin)
- Socket API key credential injected from `SOCKET_API_KEY` env var
- Pipeline job pointed at your fork via `GITHUB_REPO_URL` env var
- Optional second job auto-created when `GITHUB_REPO_URL_2` is set

### Docker Compose

| Service | Port | Purpose |
|---------|------|---------|
| Jenkins | 8090 | CI server with socketcli pre-installed |
| NodeGoat | 4000 | Vulnerable Node.js app |
| MongoDB | 27017 | NodeGoat's database |

---

## NodeGoat Vulnerable Dependencies

NodeGoat intentionally ships old packages with known CVEs — perfect for demonstrating Socket's reachability analysis:

| Package | Version | Issue |
|---------|---------|-------|
| `bcrypt-nodejs` | 0.0.3 | Unmaintained, deprecated |
| `marked` | 0.3.5 | Multiple XSS CVEs |
| `needle` | 2.2.4 | SSRF vulnerabilities |
| `mongodb` | 2.1.18 | Old driver with CVEs |
| `express` | 4.13.4 | Older with known issues |

Socket's Tier 1 reachability analysis tells you which of the 107+ CVEs across all transitive dependencies are actually reachable from this app's code — reducing alert noise by 80%.
