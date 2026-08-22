
<div align="center">

# 🚀 Flask + MongoDB CI/CD Pipeline
### Jenkins → Docker → Amazon ECR → EC2 → Email Notification

**Graded Assignment — CI/CD Pipeline** · Hero Vired DevOps Program

[![Pipeline](https://img.shields.io/badge/pipeline-Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white)](#-pipeline-overview)
[![Docker](https://img.shields.io/badge/container-Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](#-dockerization)
[![AWS ECR](https://img.shields.io/badge/registry-Amazon%20ECR-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)](#-push-to-amazon-ecr)
[![AWS EC2](https://img.shields.io/badge/deploy-Amazon%20EC2-FF9900?style=for-the-badge&logo=amazonec2&logoColor=white)](#-deploy-to-ec2)
[![Tests](https://img.shields.io/badge/tests-pytest-0A9EDC?style=for-the-badge&logo=pytest&logoColor=white)](#-testing)

</div>



<br>

> **📌 What this repo proves:** every push to `main` is automatically tested, containerized, tagged with its Git commit SHA, pushed to a private ECR repository, deployed onto a live EC2 instance by replacing the running container, verified with a real `/health` check, and reported by a **distinct** success or failure email — with zero manual steps in between.

<br>

## 📖 Table of Contents

| | | |
|---|---|---|
| [🏗️ Architecture](#️-architecture) | [⚙️ Pipeline Stages](#️-pipeline-stages) | [☁️ AWS Setup](#️-aws-prerequisites) |
| [🐳 Dockerization](#-dockerization) | [🔐 Secrets & Credentials](#-secrets--credentials) | [📬 Email Notifications](#-email-notifications) |
| [🩺 Health Check Gate](#-health-check-verification-gate) | [🧪 Testing](#-testing) | [🖼️ Evidence / Screenshots](#️-evidence--screenshots) |
| [💻 Local Development](#-local-development-no-docker-no-aws) | [🛠️ Manual Deploy (if Jenkins is down)](#️-manual-deployment-fallback) | [✅ Rubric Compliance](#-rubric-compliance) |

<br>

👉 For quick reference, you can jump directly to the evidence section here:  [🖼️ Evidence / Screenshots](#-evidence--screenshots)

```
Evidence / Screenshots
│
├── I. Implementation Stages 🛠️
│
└── II. Testing Stages with Output 🧪
    │
    ├── Output A - Successful Pipeline Execution ✅
    │
    └── Output B - Pipeline Failing Test ❌
```

## 🏗️ Architecture

```mermaid
flowchart TD
    A["👨‍💻 Developer<br/>git push origin main"] -->|GitHub Webhook| B["🔧 Jenkins Controller<br/>(EC2)"]
    B --> C["1️⃣ Checkout"]
    C --> D["2️⃣ Install Dependencies<br/>pip install -r requirements.txt"]
    D --> E["3️⃣ Test — pytest"]
    E -->|❌ any test fails| STOP["🛑 Pipeline halts<br/>Build / Push / Deploy SKIPPED"]
    STOP --> MAILF["📧 Failure Email<br/>names failed stage"]
    E -->|✅ all pass| F["4️⃣ Build Docker Image<br/>tag = Git commit SHA"]
    F --> G["5️⃣ Push to Amazon ECR"]
    G --> H["6️⃣ Deploy to EC2<br/>SSH → pull → stop old → run new"]
    H --> I["7️⃣ Verify — curl /health"]
    I -->|❌ unhealthy| STOP
    I -->|✅ 200 OK| MAILS["📧 Success Email<br/>SHA · image tag · EC2 target · build link"]

    style A fill:#4C51BF,color:#fff
    style B fill:#D24939,color:#fff
    style STOP fill:#B3261E,color:#fff
    style MAILF fill:#B3261E,color:#fff
    style MAILS fill:#1E7B4D,color:#fff
    style G fill:#FF9900,color:#000
    style H fill:#FF9900,color:#000
```

<table>
<tr><th>Component</th><th>Role</th></tr>
<tr><td><b>GitHub</b></td><td>Source control + webhook trigger on every push to <code>main</code></td></tr>
<tr><td><b>Jenkins (EC2 #1 — controller)</b></td><td>Orchestrates all 7 pipeline stages via a declarative <code>Jenkinsfile</code></td></tr>
<tr><td><b>Docker</b></td><td>Packages the Flask app into a portable, immutable image</td></tr>
<tr><td><b>Amazon ECR</b></td><td>Private registry holding every commit-SHA-tagged image</td></tr>
<tr><td><b>EC2 #2 — app target</b></td><td>Pulls the new image over SSH and runs it as the live container</td></tr>
<tr><td><b>MongoDB Atlas</b></td><td>Managed database the app connects to at runtime</td></tr>
<tr><td><b>Jenkins Email Extension</b></td><td>Sends a content-rich success/failure email on every run</td></tr>
</table>

> 💡 **Why two EC2 instances?** Keeping the Jenkins controller and the app's deploy target separate mirrors how real teams run CI/CD — the build tool and the running application never fight over the same box.

<br>

---

## ⚙️ Pipeline Stages

The `Jenkinsfile` implements these **7 stages, strictly in order** — a failure in any stage halts the pipeline immediately and skips everything after it.

| # | Stage | What happens |
|---|-------|---------------|
| 1️⃣ | **Checkout** | Pulls the latest commit from `main` |
| 2️⃣ | **Install** | `pip install -r requirements.txt` |
| 3️⃣ | **Test** 🧪 | Runs the full `pytest` suite — **hard gate**, nothing proceeds on failure |
| 4️⃣ | **Build** 🐳 | Builds the Docker image, tagged `${GIT_COMMIT}` — never just `latest` |
| 5️⃣ | **Push to ECR** ☁️ | Authenticates to AWS, pushes the SHA-tagged image |
| 6️⃣ | **Deploy to EC2** 🚀 | SSHes into the app target, pulls the new image, **stops + removes** the old container, runs the new one |
| 7️⃣ | **Verify** 🩺 | Curls `/health` on the deployed container — a crash-on-start is still reported as a **failed deployment** |

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GH as GitHub
    participant J as Jenkins
    participant ECR as Amazon ECR
    participant EC2 as EC2 (App)
    participant Mail as Email

    Dev->>GH: git push origin main
    GH->>J: webhook trigger
    J->>J: Checkout → Install → Test
    alt Test fails
        J->>Mail: ❌ Failure email (stage = Test)
    else Test passes
        J->>J: Build image (tag: SHA)
        J->>ECR: docker push
        J->>EC2: SSH — pull, stop old, run new
        EC2->>J: curl /health
        alt Health check fails
            J->>Mail: ❌ Failure email (stage = Verify)
        else Health check 200 OK
            J->>Mail: ✅ Success email (SHA, tag, EC2 host, build URL)
        end
    end
```

<br>

---

## ☁️ AWS Prerequisites

> These are set up **once, manually**, before the pipeline ever runs. The pipeline only deploys into infrastructure that already exists — it never provisions anything.

<table>
<tr><th>Resource</th><th>Purpose</th></tr>
<tr><td>🗂️ <b>ECR repository</b></td><td>Private registry that holds every SHA-tagged image</td></tr>
<tr><td>🖥️ <b>EC2 — <code>jenkins-controller</code></b></td><td>Runs Jenkins itself (t2/t3.medium recommended)</td></tr>
<tr><td>🖥️ <b>EC2 — <code>flask-practice-app</code></b></td><td>The live deploy target (t2/t3.micro, free-tier eligible)</td></tr>
<tr><td>🔑 <b>IAM role</b> on the app target</td><td><code>AmazonEC2ContainerRegistryReadOnly</code> — lets it pull from ECR with no stored keys</td></tr>
<tr><td>👤 <b>IAM user</b> for Jenkins</td><td>Push access to ECR, used only via Jenkins Credentials</td></tr>
<tr><td>🔒 <b>Security groups</b></td><td>See below</td></tr>
</table>

<details>
<summary>🔐 <b>Security group rules (click to expand)</b></summary>

<br>

**`jenkins-controller`**

| Type | Port | Source | Purpose |
|------|------|--------|---------|
| SSH | 22 | Your IP `/32` | Admin access |
| Custom TCP | 8080 | Your IP `/32` | Jenkins web UI |

**`flask-practice-app`**

| Type | Port | Source | Purpose |
|------|------|--------|---------|
| SSH | 22 | **`jenkins-controller`'s security group** (not `0.0.0.0/0`) | Jenkins deploys over SSH |
| Custom TCP | 5000 | `0.0.0.0/0` | Public app access |

> ⚠️ **Never leave port 22 open to `0.0.0.0/0`.** Scoping SSH to the controller's security group ID (not an IP range) is what keeps this safe regardless of where the controller is running.

</details>

<br>

---

## 🐳 Dockerization

<table>
<tr><td>

```dockerfile
FROM python:3.11-slim
WORKDIR /app

# Install dependencies first so Docker can cache this layer
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .
EXPOSE 5000

# Gunicorn — production-grade WSGI server
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

</td></tr>
</table>

**Build & smoke-test locally before it ever touches CI:**

```bash
docker build -t flask-practice:local .
docker run --rm -p 5000:5000 \
  -e MONGO_URI="mongodb+srv://<user>:<pass>@<cluster>/studentdb" \
  flask-practice:local

curl http://localhost:5000/health
# expect: {"status": "ok", "mongo": "connected"}
```

<br>

---

## 🔐 Secrets & Credentials

All sensitive values live in **Jenkins Credentials** — never in the repo, never in `.env` (only `.env.example` with placeholders is committed).

| Credential ID | Kind | Holds |
|---|---|---|
| `aws-creds` | AWS Credentials | Access Key / Secret Key for the `jenkins-cicd` IAM user |
| `ec2-ssh-key` | SSH Username with private key | `ubuntu` + the app target's private key |
| `mongo-uri` | Secret text | MongoDB Atlas connection string |
| `ecr-registry` | Secret text | `<account-id>.dkr.ecr.<region>.amazonaws.com` |
| `ecr-repo-name` | Secret text | e.g. `flask-practice` |
| `ec2-app-host` | Secret text | Public IP/DNS of the app target |
| `notify-email` | Secret text | Where success/failure emails are sent |

> 🚫 **Never do this:** commit an AWS key, SSH key, Mongo URI, or SMTP password to this repo — including inside `Jenkinsfile` itself. If it's sensitive, it belongs in Jenkins Credentials, full stop.

<br>

---

## 📬 Email Notifications

Graded separately from *"a notification exists"* — content must **differ meaningfully by outcome** with real build details, not a generic template.

<table>
<tr>
<td width="50%" valign="top">

### ✅ On Success

```
Subject: [SUCCESS] flask_Practice deployed - <SHA>

Deployment succeeded.
Branch: main
Commit SHA: <full SHA>
Image tag: <registry>/<repo>:<SHA>
EC2 target: <app host>
Run URL: <Jenkins build link>
```

</td>
<td width="50%" valign="top">

### ❌ On Failure

```
Subject: [FAILED] flask_Practice pipeline - <SHA>

Deployment failed.
Branch: main
Commit SHA: <full SHA>
Failed stage: Test / Build / Push / Deploy / Verify
Run URL: <Jenkins console log link>
```

</td>
</tr>
</table>

> 💡 `env.FAILED_STAGE` is set at the **top** of every stage, before its risky command runs — so the failure email always names the exact stage that broke, not just *"the pipeline failed."* Configured via the Jenkins **Email Extension** plugin with Gmail SMTP + an App Password (never the real account password).

<br>

---

## 🩺 Health-Check Verification Gate

```python
@app.route('/health')
def health():
    try:
        client.admin.command("ping")  # cheap, no auth required
        return jsonify(status="ok", mongo="connected"), 200
    except ConnectionFailure as e:
        return jsonify(status="error", mongo="unreachable", detail=str(e)), 503
```

The **Verify** stage curls this endpoint on the freshly-deployed container. A container that starts but then crashes — or can't reach Mongo — still fails the pipeline and triggers the failure email. `docker ps` showing "running" is **not** proof of a healthy deploy; a `200` from `/health` is.

<br>

---

## 🧪 Testing

```bash
pytest -v
```

| Test | Verifies |
|---|---|
| `test_health_ok` | `/health` responds `200` or `503` with a `status` key |
| Core route tests | Success **and** failure cases for existing app routes |

The **Test** stage is a hard gate — `pytest` failing stops the pipeline outright; **Build → Push → Deploy → Verify never run.**

<br>

---

## 🖼️ Evidence / Screenshots

##  "📜 Below are sequential screenshot evidences documenting implementation and testing."


# 🔧  $\Large{\textcolor{#87CEEB}{\textbf{ I.Implementation Stages }}}$

## 1.Forked Repo

<img width="1225" height="672" alt="image" src="https://github.com/user-attachments/assets/7eab88c0-71b7-410d-abb2-6bcdc631198b" />

-----------------
## 2.Performed Git Clone

<img width="1223" height="540" alt="Untitled 4" src="https://github.com/user-attachments/assets/1eec6a40-71dc-4c8a-86dc-5a00446e6eeb" />

----------------

## 3.Added matching pytest case 

```
# test_app.py
def test_health_ok(client):
    resp = client.get("/health")
    assert resp.status_code in (200, 503)   # route responds either way
    assert "status" in resp.get_json()


```

-------------

Dockerfile & .dockerignore

```
Dockerfile
FROM python:3.11-slim
 
WORKDIR /app
 
# Install dependencies first so Docker can cache this layer
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
 
# Now copy the rest of the app
COPY . .
 
EXPOSE 5000
 
# Gunicorn for a production-grade WSGI server instead of Flask's dev server
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]

```

-------------

.dockerignore

```
__pycache__/
*.pyc
.git
.gitignore
.env
.env.example
venv/
*.md
tests/
.pytest_cache/

```

----------

## 4.Create the ECR repository


```
1.AWS Console → search "ECR" → Elastic Container Registry.
2.Repositories → Create repository.
3.Visibility: Private. Name: flask-practice (or match your repo name).
3.Leave "Tag immutability" off during dev; scan-on-push can stay enabled — it's free and useful.
4.Create repository, then copy the URI shown

```

<img width="1676" height="474" alt="image" src="https://github.com/user-attachments/assets/cf70f5b4-6165-4631-97fd-d404d4791cd6" />
<img width="1677" height="516" alt="image" src="https://github.com/user-attachments/assets/79385525-96cb-4fbc-8b5f-6919f39d7f21" />

---------

## 5.Launch the EC2 instances 

One instance runs Jenkins itself (the controller). The other is the deployment target the app actually runs on. Launch both the same way, with different names and security groups.

----------

### Instance 1 - Jenkins-Controller instance 

<img width="1190" height="612" alt="image" src="https://github.com/user-attachments/assets/e53cec22-4cfc-4b66-a193-43f880ae083e" />

------------

### Instance 2 - Flask-practice app Instance 


<img width="1192" height="606" alt="image" src="https://github.com/user-attachments/assets/f0c9c8ab-50f3-4594-9459-6a25c643a860" />

------------
 # 6.Installed  Docker on both instances


 ```
# Update package index
sudo apt-get update -y

# Install Docker and AWS CLI
sudo apt-get install -y docker.io awscli

# Enable and start Docker service
sudo systemctl enable docker --now

# Add the ubuntu user to the docker group (so you can run docker without sudo)
sudo usermod -aG docker ubuntu

```
-------

Java

<img width="1235" height="60" alt="image" src="https://github.com/user-attachments/assets/a863eaf0-d8cf-43dc-8b8e-7812a584da7b" />

-----

Docker

---------
<img width="1233" height="285" alt="image" src="https://github.com/user-attachments/assets/db439897-77db-4fd4-9589-aa933dc62750" />

----------

---------

## 7. IAM role for ECR pull (app target)
------

<img width="1232" height="617" alt="image" src="https://github.com/user-attachments/assets/5ad8c699-1a7c-499e-b205-fc5859ea1e03" />

------

<img width="1235" height="275" alt="image" src="https://github.com/user-attachments/assets/9cade829-955f-4dbc-94ab-1082a98b02d0" />

-----

<img width="1230" height="620" alt="image" src="https://github.com/user-attachments/assets/c4703bbd-f6fa-421c-9aa7-997d3f597afc" />

-----

## 8.Created IAM user for Jenkins (push access)


<img width="1231" height="559" alt="image" src="https://github.com/user-attachments/assets/d3bb1241-e53c-468a-a0ae-6eee9f29e37a" />

-----

<img width="1234" height="524" alt="image" src="https://github.com/user-attachments/assets/ab6bbe3f-adf6-4c84-8e78-f9b166a2f9d1" />

----
<img width="1232" height="601" alt="image" src="https://github.com/user-attachments/assets/1da33f78-2f2a-4fb3-b504-9a6d9a1e662e" />

----

## 9.Install & Configure Jenkins

<img width="1234" height="244" alt="image" src="https://github.com/user-attachments/assets/a5c6f028-5052-4ce2-8b0b-4033fa80c17b" />

---
<img width="1232" height="493" alt="image" src="https://github.com/user-attachments/assets/bf88d8f1-fd18-4c03-be53-9ae1a5e52f21" />

---
-----

## 10.Configure the SMTP server for emailext in Jenkins


```
Manage Jenkins → System → scroll to Extended E-mail Notification.
SMTP server: smtp.gmail.com. SMTP Port: 465. Check Use SSL.
Advanced → check Use SMTP Authentication → enter your Gmail address and the App Password from Section 9.
Default Content Type: HTML. Default Recipients: your notification email.
Save, then use the built-in "test configuration" button to confirm a test email arrives before writing the pipeline.

```

---------
<img width="1228" height="303" alt="image" src="https://github.com/user-attachments/assets/33e19947-8b3b-4d55-aa16-a22c61213b5e" />

----------

------


## 11.Add the webhook on GitHub

---------------------


<img width="1238" height="648" alt="image" src="https://github.com/user-attachments/assets/81a0c9f6-d193-4840-a4d5-a07dd18dd4b8" />


---------
## 12. Updated the Jenkinsfile

<details>
<summary>  📄 <b>Jenkinsfile (click to expand)</b></summary>

<br>

```
pipeline {
  agent any

  environment {
    ECR_REGISTRY = credentials('ecr-registry')
    ECR_REPO     = credentials('ecr-repo-name')
    EC2_HOST     = credentials('ec2-app-host')
    MONGO_URI    = credentials('mongo-uri')
    AWS_REGION   = 'us-east-1'
    NOTIFY_EMAIL = 'shabdadhankkb@gmail.com'
    FAILED_STAGE = ''
  }

  stages {

    stage('Checkout') {
      steps {
        script { env.FAILED_STAGE = 'Checkout' }
        checkout scm
        script {
          env.IMAGE_TAG  = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
          env.GIT_COMMIT = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()
          env.GIT_BRANCH = sh(script: 'git rev-parse --abbrev-ref HEAD', returnStdout: true).trim()
        }
      }
    }

    stage('Install dependencies') {
      steps {
        script { env.FAILED_STAGE = 'Install' }
        sh 'pip install --break-system-packages -r requirements.txt'
      }
    }

    stage('Test') {
      steps {
        script { env.FAILED_STAGE = 'Test' }
        sh 'pytest -v'
      }
    }

    stage('Build image') {
      steps {
        script {
          env.FAILED_STAGE = 'Build'
          docker.build("${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}")
        }
      }
    }

    stage('Push to ECR') {
      steps {
        script {
          env.FAILED_STAGE = 'Push'
          withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                             credentialsId: 'aws-creds']]) {
            sh """
              aws ecr get-login-password --region ${AWS_REGION} | \
                docker login --username AWS --password-stdin ${ECR_REGISTRY}
              docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
            """
          }
        }
      }
    }

    stage('Deploy to EC2') {
      steps {
        script {
          env.FAILED_STAGE = 'Deploy'
          sshagent(credentials: ['ec2-ssh-key']) {
            sh """
              ssh -o StrictHostKeyChecking=no ubuntu@${EC2_HOST} '
                aws ecr get-login-password --region ${AWS_REGION} | \
                  docker login --username AWS --password-stdin ${ECR_REGISTRY} &&
                docker pull ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG} &&
                docker stop flask-app || true &&
                docker rm flask-app || true &&
                docker run -d --name flask-app --restart unless-stopped \
                  -p 5000:5000 \
                  -e MONGO_URI="${MONGO_URI}" \
                  ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
              '
            """
          }
        }
      }
    }

    stage('Verify deployment') {
      steps {
        script {
          env.FAILED_STAGE = 'Verify'
          sh """
            for i in 1 2 3 4 5; do
              sleep 5
              STATUS=\$(curl -s -o /dev/null -w '%{http_code}' http://${EC2_HOST}:5000/health)
              if [ "\$STATUS" = "200" ]; then exit 0; fi
            done
            echo "Health check failed after 5 attempts"
            exit 1
          """
        }
      }
    }
  }

  post {
    success {
      emailext(
        subject: "✅ SUCCESS: flask_Practice #${env.BUILD_NUMBER} deployed",
        to: "${NOTIFY_EMAIL}",
        mimeType: 'text/html',
        body: """
          <h2 style='color:#1E7B4D;'>Deployment Succeeded</h2>
          <p><b>Branch:</b> ${env.GIT_BRANCH}</p>
          <p><b>Commit SHA:</b> ${env.GIT_COMMIT}</p>
          <p><b>Image Tag:</b> ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}</p>
          <p><b>EC2 Target:</b> ${EC2_HOST}</p>
          <p><b>Run URL:</b> <a href='${env.BUILD_URL}'>${env.BUILD_URL}</a></p>
        """
      )
    }
    failure {
      emailext(
        subject: "❌ FAILED: flask_Practice #${env.BUILD_NUMBER} — ${env.FAILED_STAGE} stage",
        to: "${NOTIFY_EMAIL}",
        mimeType: 'text/html',
        body: """
          <h2 style='color:#B3261E;'>Deployment Failed</h2>
          <p><b>Failed Stage:</b> ${env.FAILED_STAGE}</p>
          <p><b>Branch:</b> ${env.GIT_BRANCH}</p>
          <p><b>Commit SHA:</b> ${env.GIT_COMMIT}</p>
          <p><b>Run URL:</b> <a href='${env.BUILD_URL}console'>${env.BUILD_URL}console</a></p>
        """
      )
    }
  }
}
```


</details>

<br>


---------



# 🧪  $\Large{\textcolor{#87CEEB}{\textbf{ II.Testing Stages with output }}}$

## ✅ I have validated the pipeline for both successful and intentional failure test cases.

## i) Output  A - Successfull Pipeline Execution 
## ii) Output B - Pipeline Failing test
 
## $\Large{\textcolor{#87CEEB}{\textbf{ i. Output A }}}$ -  Successfull Pipeline Execution 

## ⬆️ 1.Code committed and pushed from VS Code to GitHub

<img width="1007" height="523" alt="Untitled 5" src="https://github.com/user-attachments/assets/e47b9d5e-b151-48f6-b9f6-8d8899b3f513" />

## 2.Latest commits are propagated to the GitHub repository

<img width="1007" height="556" alt="image" src="https://github.com/user-attachments/assets/4904c310-94a7-4db3-9e3f-dbc2ebcfa21e" />

## 3.GitHub Webhook confirms last delivery was successful

<img width="1010" height="367" alt="image" src="https://github.com/user-attachments/assets/0264ff4a-a261-4699-9d0f-fc8f50425e39" />

## 4.Verified ECR images on the application server

<img width="1390" height="200" alt="Untitled 6" src="https://github.com/user-attachments/assets/67f3c6f5-bd04-4a4f-9bc5-83121cf2b794" />


## 🔎 5.Confirmed ECR images via AWS Console

<img width="1405" height="395" alt="Untitled 7" src="https://github.com/user-attachments/assets/d6a0a944-f745-47df-9c4e-4308494aac07" />


##  ✅ 6.Full pipeline run — all stages green

<img width="1393" height="711" alt="Untitled 8" src="https://github.com/user-attachments/assets/414c6482-e1e8-48a6-b6aa-b99fc3762fb6" />


 ## 📧 7.Successful deployment notification email received
 
<img width="1012" height="355" alt="image" src="https://github.com/user-attachments/assets/91cf709a-6071-412f-afbb-f5b6d9f67ef8" />


## 📜 8.Jenkins job console output screenshot 

<img width="1095" height="576" alt="image" src="https://github.com/user-attachments/assets/404e6d70-e416-4405-93a1-3a1b79398aac" />

##  9.Live application check

## 🌐 Application endpoint verified as reachable

<img width="1630" height="646" alt="image" src="https://github.com/user-attachments/assets/11935ff4-3b37-458b-b5c4-a6079498ed68" />

## 🟢 10.Health check node successfully registered
<img width="1153" height="264" alt="image" src="https://github.com/user-attachments/assets/d1b8c7b0-5b97-45ae-b94c-0baa5271aa61" />

## 📝 11.Student data added via application

<img width="1163" height="400" alt="image" src="https://github.com/user-attachments/assets/2ae7e021-aa02-43bd-a06e-0225956015c7" />

## 📊 12.Verified student record persisted in MongoDB

<img width="1161" height="499" alt="image" src="https://github.com/user-attachments/assets/edf92c25-0e76-413e-87d3-69c0bbab4a94" />

## 🖥️ 13.Jenkins console snippet included as supporting evidence for push, deployment, and verification steps

##1.Docker Push 

Console output confirms image was pushed to ECR.

<img width="863" height="264" alt="image" src="https://github.com/user-attachments/assets/b85fdb8a-536e-413d-9753-5520f8e83bf9" />

<img width="866" height="192" alt="image" src="https://github.com/user-attachments/assets/886db95b-29cc-4b39-b870-5b548cf91cc0" />

## 2.Deployment to EC2 actually replaces the running container
Console logs show the existing container was stopped, removed, and replaced with the latest image.


<img width="1329" height="333" alt="image" src="https://github.com/user-attachments/assets/f61983ea-ff1a-45dc-8aa6-23b3cda22160" />

## 3..Deployment Verification (Health check)

Jenkins console output validates the application health endpoint returned 200 OK.

<img width="1329" height="333" alt="image" src="https://github.com/user-attachments/assets/ec429b4a-13d9-43c4-88de-79f818597cb1" />



## $\Large{\textcolor{#87CEEB}{\textbf{ii.Output B }}}$  -  Pipeline Failing test


### 🛑 Intentionally broken run — pipeline stops early

 1. Set Response code to 999

<img width="720" height="129" alt="image" src="https://github.com/user-attachments/assets/539e6bc5-339a-4e69-a993-2fbc4a0d7a8d" />

 2.  ⬆️ Code committed and pushed from VS Code to GitHub

<img width="1184" height="707" alt="image" src="https://github.com/user-attachments/assets/e5e4f56c-b586-475e-8270-a9f17e6b086e" />

3.  Latest commits are propagated to the GitHub repository

<img width="1193" height="592" alt="image" src="https://github.com/user-attachments/assets/c8cf3514-731c-4a58-95de-0afcb49cea40" />

4. GitHub Webhook confirms last delivery was successful
   
<img width="1191" height="393" alt="image" src="https://github.com/user-attachments/assets/8c7b6146-9270-49d1-8073-862ad088c628" />

5. •  "⚠️ Jenkins job halted at the Testing stage, as shown in the screenshot below."

   <img width="1196" height="674" alt="Untitled 11" src="https://github.com/user-attachments/assets/50131e30-e1b1-4e34-a539-7ea1e5c8c835" />

6. 📧 Failure email — correct failed-stage information

<img width="1202" height="377" alt="image" src="https://github.com/user-attachments/assets/9f76ebc2-1409-491a-bc6c-b24728c1ebca" />


### 5. 🌐 Live application check

<img width="1625" height="449" alt="image" src="https://github.com/user-attachments/assets/d0b23d6c-8934-46d3-a4a6-f25ea32c0ab6" />


---

🛑 Jenkins Console snippet attached to validate failure at the Test stage
"⚠️ Jenkins console snippet highlights the invalid assertion (assert resp.status_code in == 999), aligning with our intentional failure testing."

<img width="1604" height="797" alt="image" src="https://github.com/user-attachments/assets/6300c4ef-8ade-4f67-ad85-46755ace985c" />

<img width="1453" height="363" alt="image" src="https://github.com/user-attachments/assets/41307811-6463-4650-824f-69ddef550001" />



## 💻 Local Development (No Docker, No AWS)

For quick iteration without spinning up any infrastructure:

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# .env (never committed)
echo "MONGO_URI=mongodb://localhost:27017/student_db" > .env

python app.py
# → http://localhost:5000
```

<br>

---

## 🛠️ Manual Deployment (Fallback)

If Jenkins were ever unavailable, here's the exact sequence the pipeline automates — run by hand, straight from the `Jenkinsfile`:

```bash
# On the app target (flask-practice-app)
aws ecr get-login-password --region <region> | \
  docker login --username AWS --password-stdin <ecr-registry>

docker pull <ecr-registry>/<ecr-repo>:<commit-sha>

docker stop flask-app || true
docker rm flask-app || true

docker run -d --name flask-app --restart unless-stopped \
  -p 5000:5000 \
  -e MONGO_URI="<mongo-uri>" \
  <ecr-registry>/<ecr-repo>:<commit-sha>

curl http://localhost:5000/health
```

> **Why SSH instead of SSM?** Chosen for simplicity in a learning assignment — SSM adds IAM/agent setup overhead that isn't necessary here. SSH access is locked down to only the Jenkins controller's security group.

<br>

---

## ✅ Rubric Compliance

| Criterion | Weight | ✔️ Evidence in this repo |
|---|:---:|---|
| Pipeline stages present & passing in correct order (test gates build/deploy) | **25%** | `Jenkinsfile` — sequential `stages{}`, no `continue-on-error`; see [Pipeline Stages](#️-pipeline-stages) |
| Docker image built & tagged with commit SHA | **15%** | `IMAGE_TAG = "${env.GIT_COMMIT}"` in `Jenkinsfile`; see [Dockerization](#-dockerization) |
| Image successfully pushed to ECR | **10%** | `Push to ECR` stage; see [AWS Prerequisites](#️-aws-prerequisites) |
| EC2 deployment replaces the running container | **20%** | `docker stop` + `docker rm` before `docker run` in `Deploy to EC2` stage |
| Health-check step genuinely gates success/failure | **10%** | `/health` route + `Verify` stage; see [Health-Check Verification Gate](#-health-check-verification-gate) |
| Email notification — customized, correct on both outcomes | **15%** | Distinct `post { success / failure }` blocks; see [Email Notifications](#-email-notifications) |
| README documentation quality | **5%** | This file 🙂 |

<br>

<div align="center">

---

**Built for the Hero Vired DevOps Program — CI/CD Pipeline Graded Assignment**
⭐ **If this pipeline helped you understand CI/CD gating logic, consider starring the repo!** ⭐

`Flask` · `MongoDB` · `Docker` · `Jenkins` · `Amazon ECR` · `Amazon EC2`

</div>


