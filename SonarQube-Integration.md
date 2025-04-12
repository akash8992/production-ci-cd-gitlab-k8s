# Automated SonarQube Integration with GitLab for Branch-Level Code Analysis

To set up an automated SonarQube scanning process that analyzes code whenever developers push to their branches and sends reports back to them, follow this comprehensive guide:

## Prerequisites
- GitLab self-hosted instance (already set up)
- Server with sufficient resources (SonarQube requires at least 4GB RAM for small teams)
- Docker installed (recommended for SonarQube deployment)
- Admin access to GitLab

## Step 1: Install and Configure SonarQube

### Option A: Docker Installation (Recommended)
```bash
# Create directories for persistent data
mkdir -p /opt/sonarqube/data /opt/sonarqube/extensions /opt/sonarqube/logs

# Set proper permissions
chmod -R 777 /opt/sonarqube

# Run SonarQube with Docker
docker run -d --name sonarqube \
  -p 9000:9000 \
  -v /opt/sonarqube/data:/opt/sonarqube/data \
  -v /opt/sonarqube/extensions:/opt/sonarqube/extensions \
  -v /opt/sonarqube/logs:/opt/sonarqube/logs \
  sonarqube:latest
```

### Option B: Manual Installation
1. Download SonarQube from [official site](https://www.sonarqube.org/downloads/)
2. Unzip to `/opt/sonarqube`
3. Configure `sonar.properties`:
   ```
   sonar.jdbc.url=jdbc:postgresql://localhost/sonar
   sonar.jdbc.username=sonar
   sonar.jdbc.password=sonar
   sonar.path.data=/opt/sonarqube/data
   sonar.path.temp=/opt/sonarqube/temp
   ```
4. Start SonarQube:
   ```bash
   ./bin/linux-x86-64/sonar.sh start
   ```

## Step 2: Initial SonarQube Setup

1. Access SonarQube at `http://your-server:9000`
2. Login with admin/admin (change password immediately)
3. Generate a token:
   - Go to Administration → Security → Users
   - Click on your admin user → Tokens → Generate
   - Save this token securely (you'll need it for GitLab integration)

## Step 3: Install and Configure SonarQube Scanner

```bash
# Download scanner
wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.7.0.2747-linux.zip
unzip sonar-scanner-cli-4.7.0.2747-linux.zip
mv sonar-scanner-4.7.0.2747-linux /opt/sonar-scanner

# Add to PATH
echo 'export PATH=$PATH:/opt/sonar-scanner/bin' >> /etc/profile
source /etc/profile
```

## Step 4: GitLab Integration

### 1. Install GitLab SonarQube Exporter (for reports in GitLab UI)
```bash
# On GitLab server
sudo gitlab-rails runner "Feature.enable(:sonarqube_integration)"
```

### 2. Configure GitLab to Connect to SonarQube
1. Go to Admin Area → Settings → CI/CD
2. Expand "Continuous Integration and Deployment"
3. Add SonarQube configuration:
   - SonarQube URL: `http://your-sonarqube:9000`
   - Authentication token: [the token you generated earlier]

### 3. Set Up Project-Level Integration
For each project (or group settings if you want to apply to multiple projects):

1. Go to Project → Settings → CI/CD
2. Add variables:
   - `SONAR_HOST_URL`: `http://your-sonarqube:9000`
   - `SONAR_TOKEN`: [project-specific token from SonarQube]
   - `SONAR_PROJECT_KEY`: `your-project-key` (unique per project)
   - `SONAR_USER_HOME`: `${CI_PROJECT_DIR}/.sonar`

## Step 5: Configure GitLab CI/CD Pipeline

Create or modify `.gitlab-ci.yml` in each project:

```yaml
stages:
  - test
  - sonarqube-check

sonarqube-check:
  stage: sonarqube-check
  image:
    name: sonarsource/sonar-scanner-cli:latest
    entrypoint: [""]
  variables:
    SONAR_USER_HOME: "${CI_PROJECT_DIR}/.sonar"
    GIT_DEPTH: 0
  script:
    - sonar-scanner
      -Dsonar.projectKey=${SONAR_PROJECT_KEY}
      -Dsonar.projectName=${CI_PROJECT_NAME}
      -Dsonar.projectVersion=${CI_COMMIT_SHORT_SHA}
      -Dsonar.sources=.
      -Dsonar.host.url=${SONAR_HOST_URL}
      -Dsonar.login=${SONAR_TOKEN}
      -Dsonar.branch.name=${CI_COMMIT_REF_NAME}
      -Dsonar.qualitygate.wait=true
      -Dsonar.gitlab.project_id=${CI_PROJECT_ID}
      -Dsonar.gitlab.commit_sha=${CI_COMMIT_SHA}
      -Dsonar.gitlab.ref_name=${CI_COMMIT_REF_NAME}
  allow_failure: true
  only:
    - branches
  except:
    - master
    - main
```

## Step 6: Configure Quality Gate Notifications

1. In SonarQube, go to Administration → Configuration → General Settings → GitLab
   - Enable GitLab authentication
   - Set GitLab URL to your GitLab instance
   - Add GitLab Application ID and Secret (create a GitLab OAuth application first)

2. Configure notifications:
   - Go to Administration → Configuration → Notifications
   - Add email notifications for "Quality Gate changes"
   - Configure to send to "Issue Assignees"

## Step 7: Set Up Branch Analysis and Developer Reporting

### 1. Configure Branch Analysis in SonarQube
1. Go to Project Settings → General Settings → Branches
2. Enable "Branch analysis"
3. Set:
   - Branch type: Both long-lived and short-lived branches
   - Automatic deletion of inactive branches: Enable (set to 30 days)

### 2. Create GitLab Webhook for Push Events
1. Go to Project → Settings → Webhooks
2. Add webhook:
   - URL: `http://your-sonarqube:9000/api/gitlab/webhook`
   - Trigger: Push events
   - Secret token: [generate and save securely]
3. In SonarQube, go to Project Settings → GitLab Integration
   - Add the same secret token
   - Enable "Automatic analysis on push"

### 3. Customize Developer Notifications
Create a script to parse SonarQube reports and send to developers:

```bash
#!/bin/bash

# This script would be triggered by SonarQube webhook
# Place it in /usr/local/bin/sonarqube_notify.sh

# Parse webhook JSON
PROJECT_KEY=$(jq -r '.project.key' $SONARQUBE_WEBHOOK_PAYLOAD)
BRANCH=$(jq -r '.branch.name' $SONARQUBE_WEBHOOK_PAYLOAD)
COMMIT_SHA=$(jq -r '.commit.sha' $SONARQUBE_WEBHOOK_PAYLOAD)

# Get GitLab commit info to find pusher
GITLAB_USER_EMAIL=$(curl --header "PRIVATE-TOKEN: $GITLAB_API_TOKEN" \
  "$GITLAB_URL/api/v4/projects/$CI_PROJECT_ID/repository/commits/$COMMIT_SHA" | \
  jq -r '.author_email')

# Get SonarQube analysis results
ANALYSIS_ID=$(curl --header "Authorization: Bearer $SONAR_TOKEN" \
  "$SONAR_HOST_URL/api/project_analyses/search?project=$PROJECT_KEY&branch=$BRANCH" | \
  jq -r '.analyses[0].key')

ISSUES=$(curl --header "Authorization: Bearer $SONAR_TOKEN" \
  "$SONAR_HOST_URL/api/issues/search?componentKeys=$PROJECT_KEY&branch=$BRANCH")

# Format message
MESSAGE="SonarQube analysis for your branch $BRANCH:\n"
MESSAGE+="Total issues: $(echo $ISSUES | jq '.total')\n"
MESSAGE+="Critical: $(echo $ISSUES | jq '.issues[] | select(.severity=="CRITICAL") | .key' | wc -l)\n"
MESSAGE+="Major: $(echo $ISSUES | jq '.issues[] | select(.severity=="MAJOR") | .key' | wc -l)\n"
MESSAGE+="View full report: $SONAR_HOST_URL/dashboard?id=$PROJECT_KEY&branch=$BRANCH"

# Send email to developer
echo -e "$MESSAGE" | mail -s "SonarQube Analysis Report" "$GITLAB_USER_EMAIL"
```

## Step 8: Final Configuration and Testing

1. Set up the webhook script as a service:
   ```bash
   sudo chmod +x /usr/local/bin/sonarqube_notify.sh
   ```

2. Create a systemd service:
   ```ini
   [Unit]
   Description=SonarQube GitLab Notifier
   After=network.target

   [Service]
   ExecStart=/usr/local/bin/sonarqube_notify.sh
   Restart=always
   User=root
   Environment=SONAR_TOKEN=your_sonarqube_token
   Environment=GITLAB_API_TOKEN=your_gitlab_token
   Environment=GITLAB_URL=https://your.gitlab.instance
   Environment=SONAR_HOST_URL=http://your-sonarqube:9000

   [Install]
   WantedBy=multi-user.target
   ```

3. Enable and start the service:
   ```bash
   sudo systemctl enable sonarqube-notifier
   sudo systemctl start sonarqube-notifier
   ```

## Verification

1. Have a developer push to a feature branch
2. Verify:
   - Pipeline runs in GitLab
   - SonarQube analysis is triggered
   - Report appears in SonarQube with branch-specific data
   - Developer receives email notification
   - Issues appear in GitLab merge request UI

## Troubleshooting Tips

1. Check SonarQube logs: `/opt/sonarqube/logs/sonar.log`
2. Verify webhook deliveries in GitLab (Project → Settings → Webhooks)
3. Check scanner output in GitLab CI/CD job logs
4. Ensure tokens have correct permissions in both systems
