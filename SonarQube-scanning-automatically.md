## ✅ What You Need to Set Up

### 🧱 1. **Install SonarQube on Your GitLab Server**
You can install SonarQube using Docker or manually. Here’s a Docker-based setup (recommended):

```bash
docker run -d --name sonarqube \
  -p 9000:9000 \
  -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true \
  sonarqube:community
```

> Access it at `http://<your-server-ip>:9000`

Create:
- A **SonarQube project** for each GitLab repo (or one that handles all branches)
- A **SonarQube token** for GitLab runner

---

### 🔁 2. **GitLab CI/CD Integration with SonarQube**

#### a. **Add `sonar-project.properties` to Your Repo**

```properties
sonar.projectKey=your-project
sonar.sources=.
sonar.host.url=http://<your-server-ip>:9000
sonar.login=${SONAR_TOKEN}
sonar.branch.name=${CI_COMMIT_REF_NAME}
```

#### b. **Create a Secret Token in GitLab CI/CD Variables**
In GitLab UI:
- Go to your project → Settings → CI/CD → Variables
- Add: `SONAR_TOKEN` → paste your SonarQube token

---

### 🔧 3. **Configure `.gitlab-ci.yml` for Auto Scan**

Add this job in your `.gitlab-ci.yml`:

```yaml
stages:
  - sonar

sonarqube-check:
  image: sonarsource/sonar-scanner-cli:latest
  stage: sonar
  script:
    - sonar-scanner
  only:
    - branches
```

This will:
- Run on **every push to any branch**
- Use the `sonar-scanner` to scan the code and push the report to SonarQube

---

### 📬 4. **Send Scan Report to Developer Automatically**

We’ll automate emails to the **commit author**.

#### Method A: Simple Email in CI Job (Linux Tools)

Add this to the end of your `.gitlab-ci.yml` job:

```yaml
  after_script:
    - apt-get update && apt-get install -y mailutils
    - |
      AUTHOR_EMAIL=$(git log -1 --pretty=format:'%ae')
      REPORT_URL="http://<your-server-ip>:9000/dashboard?id=your-project&branch=${CI_COMMIT_REF_NAME}"
      echo "SonarQube scan completed for branch ${CI_COMMIT_REF_NAME}. View report: $REPORT_URL" \
        | mail -s "SonarQube Report for ${CI_COMMIT_REF_NAME}" "$AUTHOR_EMAIL"
```

> Make sure your GitLab runner has `mailutils` and can send mail (via local SMTP or external).

---

### 🛠️ Alternative Notification: Slack or GitLab Comment
Instead of email, you can:
- Use [Slack webhook](https://api.slack.com/messaging/webhooks) to ping developers
- Use GitLab API to post a comment on the commit/pipeline with the report link

---

### 🔐 Bonus: Secure the Setup
- Use **SonarQube authentication token** in CI secrets
- Use a **reverse proxy (NGINX)** and TLS (SSL) for SonarQube server

---

### ✅ Final Flow

1. Developer pushes code to their branch
2. GitLab CI triggers and runs SonarQube scan
3. Report is generated per branch in SonarQube
4. Email is sent to the developer who made the push, with report link

---

Would you like:
- A ready-to-paste `.gitlab-ci.yml` file tailored to your project?
- Help setting up SMTP for GitLab Runner to send emails?
- A Slack-based notification version instead?
