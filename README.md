# 🚀 Flask + MongoDB CI/CD Pipeline — Jenkins Edition

**Automated test → build → deploy → verify → notify pipeline for a Python Flask application**

![Pipeline](https://img.shields.io/badge/pipeline-passing-brightgreen?style=for-the-badge&logo=jenkins&logoColor=white)
![Docker](https://img.shields.io/badge/docker-containerized-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Python](https://img.shields.io/badge/python-3.11-3776AB?style=for-the-badge&logo=python&logoColor=white)
![Flask](https://img.shields.io/badge/flask-app-000000?style=for-the-badge&logo=flask&logoColor=white)
![MongoDB](https://img.shields.io/badge/mongodb-database-47A248?style=for-the-badge&logo=mongodb&logoColor=white)
![Tests](https://img.shields.io/badge/tests-pytest-0A9EDC?style=for-the-badge&logo=pytest&logoColor=white)

>  **CI/CD Pipeline**
> Built and validated end‑to‑end: source control → automated testing → containerization → deployment → health verification → email notification.

---

## 📋 Table of Contents

1. [Objective](#-objective)
2. [Architecture](#-architecture)
3. [Tech Stack](#-tech-stack)
4. [Repository Structure](#-repository-structure)
5. [Prerequisites](#-prerequisites)
6. [Getting Started](#-getting-started)
7. [Pipeline Stages](#-pipeline-stages-in-detail)
8. [Email Notifications](#-email-notifications)
9. [Secrets Management](#-secrets-management)
11. [A full pipeline run Screenshots & Evidence](#A-full-pipeline-run-Screenshots-and-Evidence) ✅
12. [Author](#-author)

---

## 🎯 Objective

This repository implements a fully automated CI/CD pipeline for a Python **Flask + MongoDB** web application. Every push to `main` triggers a Jenkins pipeline that:

- ✅ Installs dependencies and runs the **pytest** suite — a failing test **gates** the pipeline (no build/deploy happens)
- 🏗️ Builds a **Docker image**, tagged with the **Git commit SHA** (never just `latest`) for full traceability
- 📦 Publishes the image
- 🔁 **Deploys** by replacing the currently running container — not a fresh install every time
- ❤️ **Verifies** the deployment with a `/health` endpoint check — this is the true success/failure gate
- 📧 Sends a **customized email** reporting the outcome, with different content for success vs. failure

---

## 🏗️ Architecture

```mermaid
flowchart TD
    A[👨‍💻 Developer pushes to main] --> B[🔔 GitHub Webhook]
    B --> C[⚙️ Jenkins Pipeline Triggered]
    C --> D[📥 Checkout]
    D --> E[📦 Install Dependencies]
    E --> F{🧪 Run pytest}
    F -- ❌ Fail --> G[🛑 Stop Pipeline]
    G --> H1[📧 Failure Email]
    F -- ✅ Pass --> I[🐳 Build Docker Image<br/>tagged with Git SHA]
    I --> J[📤 Push Image]
    J --> K[🔁 Deploy: stop/remove old container<br/>→ run new container]
    K --> L{❤️ /health check}
    L -- ❌ Unhealthy --> H1
    L -- ✅ Healthy --> H2[📧 Success Email]
```

**Flow summary:** `Push → Checkout → Install → Test (gate) → Build (SHA‑tagged) → Push image → Deploy (swap container) → Verify (health gate) → Notify`

---

## 🧰 Tech Stack

| Layer | Technology |
|---|---|
| Application | Python 3.11, Flask |
| Database | MongoDB (Docker container) |
| Testing | pytest |
| Containerization | Docker |
| CI/CD Orchestration | Jenkins (Pipeline as Code — `Jenkinsfile`) |
| Notifications | Jenkins Email Extension (`emailext`), Gmail SMTP |
| Trigger | GitHub Webhook (`push` to `main`) |

---

## 📁 Repository Structure

```text
.
├── app.py                  # Flask application (incl. /health route)
├── test_app.py             # pytest suite — success & failure cases
├── requirements.txt        # Python dependencies
├── Dockerfile              # Container image definition
├── .dockerignore
├── Jenkinsfile             # Pipeline definition (Checkout→Notify)
├── .gitignore
└── README.md                # You are here 📍
```

---

## ✅ Prerequisites

| Requirement | Purpose |
|---|---|
| Docker installed & running | Builds and runs the application + database containers |
| Jenkins with **Docker Pipeline**, **Email Extension**, and **GitHub Integration** plugins | Runs and orchestrates the pipeline |
| A reachable Jenkins URL for the GitHub webhook | Lets GitHub trigger builds automatically on push |
| Jenkins credentials configured (see [Secrets Management](#-secrets-management)) | Keeps DB URI, SMTP, and Git tokens out of the repo |
| Gmail App Password (or your SMTP provider of choice) | Sends success/failure notification emails |

---

## 🏁 Getting Started

<table>
<tr><td>

```bash
# 1️⃣ Clone your fork
git clone https://github.com/<your-username>/flask_Practice.git
cd flask_Practice

# 2️⃣ Create a virtual environment
python3 -m venv venv
source venv/bin/activate

# 3️⃣ Install dependencies
pip install -r requirements.txt

# 4️⃣ Run the test suite locally
pytest -v

# 5️⃣ Build the Docker image
docker build -t flask-student-app:local .
```
</td></tr>
</table>
------------------


## 1.Forked Repo

<img width="1225" height="672" alt="image" src="https://github.com/user-attachments/assets/7eab88c0-71b7-410d-abb2-6bcdc631198b" />

-----------------
## 2.Git Clone

<img width="1223" height="540" alt="Untitled 4" src="https://github.com/user-attachments/assets/1eec6a40-71dc-4c8a-86dc-5a00446e6eeb" />

----------------

## 3.dded matching pytest case 

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

---------

## 5.Launch the EC2 instances 

One instance runs Jenkins itself (the controller). The other is the deployment target the app actually runs on. Launch both the same way, with different names and security groups.

## 🖥️ EC2 Instances Overview

| Instance   | Name                | Type                          | Purpose                                      |
|------------|---------------------|-------------------------------|----------------------------------------------|
| Controller | jenkins-controller  | t2.medium / t3.medium         | Runs Jenkins, builds images, triggers deploys |
| App target | flask-practice-app  | t2.micro / t3.micro (free-tier eligible) | Runs the deployed Flask container |


## 🔐 Security Group Rules

| Type       | Port | Source       | Purpose                                      |
|------------|------|--------------|----------------------------------------------|
| SSH        | 22   | Your IP /32  | You SSH in to install Jenkins and troubleshoot |
| Custom TCP | 8080 | Your IP /32 (or a small trusted range) | Jenkins web UI access |

----------

Jenkins-Controller instance 

<img width="1190" height="612" alt="image" src="https://github.com/user-attachments/assets/e53cec22-4cfc-4b66-a193-43f880ae083e" />

------------

Flask-practice app Instance 


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

## IAM role for ECR pull (app target)

## 🔑 Authentication & Access Approach

| Approach                                | How it works                                                                 | Used in this guide |
|-----------------------------------------|-------------------------------------------------------------------------------|--------------------|
| EC2 instance role on `flask-practice-app` | Lets the EC2 box pull from ECR without storing AWS keys locally               | ✅ Yes — for the `docker pull` during deploy |
| IAM user + access keys (Jenkins Credentials) | Jenkins authenticates to AWS directly to push the image to ECR                | ✅ Yes — for the Build/Push stages |

------

<img width="1232" height="617" alt="image" src="https://github.com/user-attachments/assets/5ad8c699-1a7c-499e-b205-fc5859ea1e03" />

------

<img width="1235" height="275" alt="image" src="https://github.com/user-attachments/assets/9cade829-955f-4dbc-94ab-1082a98b02d0" />

-----

<img width="1230" height="620" alt="image" src="https://github.com/user-attachments/assets/c4703bbd-f6fa-421c-9aa7-997d3f597afc" />

-----

## 7.Created IAM user for Jenkins (push access)


<img width="1231" height="559" alt="image" src="https://github.com/user-attachments/assets/d3bb1241-e53c-468a-a0ae-6eee9f29e37a" />

-----

<img width="1234" height="524" alt="image" src="https://github.com/user-attachments/assets/ab6bbe3f-adf6-4c84-8e78-f9b166a2f9d1" />

----
<img width="1232" height="601" alt="image" src="https://github.com/user-attachments/assets/1da33f78-2f2a-4fb3-b504-9a6d9a1e662e" />

----

## 8.Install & Configure Jenkins

<img width="1234" height="244" alt="image" src="https://github.com/user-attachments/assets/a5c6f028-5052-4ce2-8b0b-4033fa80c17b" />

---
<img width="1232" height="493" alt="image" src="https://github.com/user-attachments/assets/bf88d8f1-fd18-4c03-be53-9ae1a5e52f21" />

---
-----

## 9.Configure the SMTP server for emailext in Jenkins


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


## 10.Add the webhook on GitHub

---------------------


<img width="1238" height="648" alt="image" src="https://github.com/user-attachments/assets/81a0c9f6-d193-4840-a4d5-a07dd18dd4b8" />


---------


---------


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

## 🔐 Secrets Management

All sensitive values live in **Jenkins Credentials** — nothing is committed to the repository:


| Credential ID | Kind                  | Value                                                                 |
|---------------|-----------------------|-----------------------------------------------------------------------|
| aws-creds     | AWS Credentials       | Access Key ID / Secret Access Key from the `jenkins-cicd` IAM user     |
| ec2-ssh-key   | SSH Username + Key    | Username: `ubuntu` · Key: full contents of `flask-practice-key.pem`    |
| mongo-uri     | Secret text           | Your MongoDB Atlas connection string                                   |
| ecr-registry  | Secret text           | `<account-id>.dkr.ecr.<region>.amazonaws.com`                          |
| ecr-repo-name | Secret text           | e.g. `flask-practice`                                                  |
| ec2-app-host  | Secret text           | Public IP or DNS of `flask-practice-app`                               |
| notify-email  | Secret text           | Email address where success/failure notifications should be sent       |



---------
<img width="1231" height="575" alt="image" src="https://github.com/user-attachments/assets/b6807160-0ed3-4df2-bcbf-9477e54d2e36" />

----------


`.env`, credentials, and keys are all listed in `.gitignore` and never appear in pipeline logs or source.

---


# 🔄  $\Large{\textcolor{#87CEEB}{\textbf{A full pipeline run Screenshots and Evidence}}}$
 
## $\Large{\textcolor{#87CEEB}{\textbf{Output A }}}$ -  Successfull Pipeline Execution 

## ⬆️ Code committed and pushed from VS Code to GitHub

<img width="1007" height="523" alt="Untitled 5" src="https://github.com/user-attachments/assets/e47b9d5e-b151-48f6-b9f6-8d8899b3f513" />

## Latest commits are propagated to the GitHub repository

<img width="1007" height="556" alt="image" src="https://github.com/user-attachments/assets/4904c310-94a7-4db3-9e3f-dbc2ebcfa21e" />

## GitHub Webhook confirms last delivery was successful

<img width="1010" height="367" alt="image" src="https://github.com/user-attachments/assets/0264ff4a-a261-4699-9d0f-fc8f50425e39" />

## Verified ECR images on the application server

<img width="1390" height="200" alt="Untitled 6" src="https://github.com/user-attachments/assets/67f3c6f5-bd04-4a4f-9bc5-83121cf2b794" />


## 🔎 Confirmed ECR images via AWS Console

<img width="1405" height="395" alt="Untitled 7" src="https://github.com/user-attachments/assets/d6a0a944-f745-47df-9c4e-4308494aac07" />


#  ✅ Full pipeline run — all stages green

<img width="1393" height="711" alt="Untitled 8" src="https://github.com/user-attachments/assets/414c6482-e1e8-48a6-b6aa-b99fc3762fb6" />


 ## 📧 Successful deployment notification email received
 
<img width="1012" height="355" alt="image" src="https://github.com/user-attachments/assets/91cf709a-6071-412f-afbb-f5b6d9f67ef8" />


## 📜 Jenkins job console output screenshot 

<img width="1095" height="576" alt="image" src="https://github.com/user-attachments/assets/404e6d70-e416-4405-93a1-3a1b79398aac" />

##  Live application check

## 🌐 Application endpoint verified as reachable

<img width="1630" height="646" alt="image" src="https://github.com/user-attachments/assets/11935ff4-3b37-458b-b5c4-a6079498ed68" />

## 🟢 Health check node successfully registered
<img width="1153" height="264" alt="image" src="https://github.com/user-attachments/assets/d1b8c7b0-5b97-45ae-b94c-0baa5271aa61" />

## 📝 Student data added via application

<img width="1163" height="400" alt="image" src="https://github.com/user-attachments/assets/2ae7e021-aa02-43bd-a06e-0225956015c7" />

## 📊 Verified student record persisted in MongoDB

<img width="1161" height="499" alt="image" src="https://github.com/user-attachments/assets/edf92c25-0e76-413e-87d3-69c0bbab4a94" />

## 🖥️ Jenkins console snippet included as supporting evidence for push, deployment, and verification steps

## Docker Push 

Console output confirms image was pushed to ECR.

<img width="863" height="264" alt="image" src="https://github.com/user-attachments/assets/b85fdb8a-536e-413d-9753-5520f8e83bf9" />

<img width="866" height="192" alt="image" src="https://github.com/user-attachments/assets/886db95b-29cc-4b39-b870-5b548cf91cc0" />

## Deployment to EC2 actually replaces the running container
Console logs show the existing container was stopped, removed, and replaced with the latest image.


<img width="1329" height="333" alt="image" src="https://github.com/user-attachments/assets/f61983ea-ff1a-45dc-8aa6-23b3cda22160" />

## Deployment Verification (Health check)

Jenkins console output validates the application health endpoint returned 200 OK.

<img width="1329" height="333" alt="image" src="https://github.com/user-attachments/assets/ec429b4a-13d9-43c4-88de-79f818597cb1" />





## $\Large{\textcolor{#87CEEB}{\textbf{Output B }}}$  -  Pipeline Failing test


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

## 👤 Author

Maintained as part of a graded DevOps CI/CD pipeline assignment.

**Stack:** Flask · MongoDB · Docker · Jenkins · pytest

---

<div align="center">

⭐ **If this pipeline helped you understand CI/CD gating logic, consider starring the repo!** ⭐

</div>
