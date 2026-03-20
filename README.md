# Jenkins + Socket Security Demo

Demonstrates Socket's Tier 1 Reachability analysis integrated into a local Jenkins CI pipeline. The job template is parameterized so you can point it at any repository.

---

## Prerequisites

- Docker + Docker Compose
- A Socket API key for your org ([get one here](https://socket.dev))

---

## Quick Start

### 1. Clone the repo

```bash
git clone https://github.com/YOUR_GITHUB_USER/jenkins-socket-demo.git
cd jenkins-socket-demo
```

### 2. Set environment variables

```bash
# Required: your Socket API key
export SOCKET_API_KEY=sktsec_your_key_here

# Required: point Jenkins at your fork (repo containing the Jenkinsfile)
export GITHUB_REPO_URL=https://github.com/YOUR_GITHUB_USER/jenkins-socket-demo.git
```

### 3. Start the stack

```bash
docker compose up -d
```

Jenkins will be available at **http://localhost:8090** (user: `admin`, password: `admin`).

### 4. Create new Jenkins Job with cloning the Tier-1-Reachability-Job-Template 

1. Click on **+ New Item**
2. Edit **Copy from** set to **Tier-1-Reachability-Job-Template** 
3. **Enter an item name** give it a name (i.e., **go-dvwa-jenkins-demo**)
4. Click **OK**
### Within the job configuration
5. Change the **Defaut Value** field for parameterized **REPO_NAME** (i.e., **set it to the same as the job name**)
6. Enter the **Repository URL**  (i.e., **https://github.com/YOR_GITHUB_USER/go-dvwa-gh**)
4. Click **Save**

### 5. Trigger a scan

1. Click to Open the job (i.e., **go-dvwa-jenkins-demo**) in Jenkins
2. Click **Build with Parameters**
3. Enter `REPO_NAME` (defaults to the job name if left blank)
4. Click **Build**

### 6. View results

- **Jenkins console**: Check the build output in the Jenkins UI
- **Socket dashboard**: https://socket.dev/dashboard/your-org/REPO_NAME/main/

---

## How It Works

### Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `REPO_NAME` | Job name | Repository name for Socket security scan |
| `SOCKET_API_CREDENTIAL_ID` | `socket-api-key` | Override Socket API token credential ID |

### Jenkinsfile

```groovy
pipeline {
    agent any
    environment {
        SOCKET_SECURITY_API_TOKEN = credentials('socket-api-key')
        REPO_NAME = "${params.REPO_NAME ?: env.JOB_NAME}"
    }
    stages {
        stage('Install Dependencies') {
            steps { sh 'npm install --ignore-scripts' }
        }
        stage('Socket Security Scan') {
            steps {
                sh '''
                    socketcli \
                        --target-path . \
                        --repo ${REPO_NAME} \
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

### Jenkins Configuration as Code (JCasC)

`jenkins/casc/jenkins.yaml` pre-configures:
- Admin credentials (admin/admin)
- Socket API key credential injected from `SOCKET_API_KEY` env var
- Parameterized pipeline job with `REPO_NAME` defaulting to the job name
- Job pointed at your fork via `GITHUB_REPO_URL` env var

### Docker Compose

| Service | Port | Purpose |
|---------|------|---------|
| Jenkins | 8090 | CI server with socketcli pre-installed |
| MongoDB | 27017 | Available for app databases |

---

## Clean Restart

To wipe all persistent data and start fresh:

```bash
docker compose down -v
docker compose up -d
```

**Note:** The `jobs:` key in `jenkins/casc/jenkins.yaml` must be an explicit empty list (`jobs: []`), not null/blank. The job-dsl `SeedJobConfigurator` throws a NullPointerException on a null value.
