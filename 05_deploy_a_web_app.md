# Part 5 — Continuous Deployment with AWS CodeDeploy

> 6 Day DevOps Challenge — Day 5  
> Focus: Infrastructure as Code, automated deployment, lifecycle hooks, rollback, and recovery.

---

# Overview

In this project, I extended the CI pipeline into a full **Continuous Deployment (CD) pipeline**.

The goal was to:

- Provision production infrastructure using CloudFormation
- Deploy the built WAR artifact from S3
- Configure CodeDeploy lifecycle hooks
- Automate service installation and startup
- Simulate deployment failure
- Perform manual recovery

This project transformed the pipeline from:

Build Automation → Fully Automated Deployment.

---

# Architecture Overview

Pipeline flow:

GitHub → CodeBuild → S3 Artifact → CodeDeploy → EC2 (Production)

Infrastructure provisioning:

CloudFormation → VPC + Subnet + Security Group + EC2

The production EC2 instance is completely separate from the development instance.

---

# Step 1 — Provision Production Infrastructure (CloudFormation)

Instead of manually launching EC2, I used **Infrastructure as Code**.

Created a CloudFormation stack:

`NextWorkCodeDeployEC2Stack`

Resources provisioned:

- VPC
- Public Subnet
- Route Tables
- Internet Gateway
- Security Group
- EC2 instance

Security configuration:

- Port 22 (SSH) restricted to My IP (/32)
- Port 80 (HTTP) open for application access

Why CloudFormation?

- Infrastructure is reproducible
- Environment can be recreated in minutes
- Stack can be deleted cleanly
- Follows Infrastructure as Code best practices

Stack reached: ✅ CREATE_COMPLETE

---

# Step 2 — Prepare Deployment Scripts

To allow CodeDeploy to automate server configuration, I created a `scripts/` folder.

### install_dependencies.sh

- Installs Tomcat
- Installs Apache (httpd)
- Configures reverse proxy (Apache → Tomcat)

Purpose:
Prepare the server environment during deployment.

---

### start_server.sh

- Starts Tomcat
- Starts Apache
- Enables services on boot

Purpose:
Ensure application runs after deployment.

---

### stop_server.sh

- Checks if services are running
- Stops Tomcat safely
- Stops Apache safely

Purpose:
Prevent file conflicts during redeployment.

---

# Step 3 — Create appspec.yml

Created `appspec.yml` at the project root.

```yaml
version: 0.0
os: linux
files:
  - source: /target/nextwork-web-project.war
    destination: /usr/share/tomcat/webapps/
hooks:
  ApplicationStop:
    - location: scripts/stop_server.sh
      timeout: 300
      runas: root
  BeforeInstall:
    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: root
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 300
      runas: root
```

This file tells CodeDeploy:

- Where to copy the WAR file
- Which scripts to run
- In what order
- With which permissions

---

# Step 4 — Update buildspec.yml for Deployment Packaging

Modified artifact section:

```yaml
artifacts:
  files:
    - target/nextwork-web-project.war
    - appspec.yml
    - scripts/**/*
  discard-paths: no
```

This ensures:

- WAR file
- appspec.yml
- Deployment scripts

are all packaged inside the deployment artifact.

Rebuilt using CodeBuild.

Artifact verified in S3.

---

# Step 5 — Create CodeDeploy Application

Created:

Application Name:
`nextwork-devops-cicd`

Compute platform:
EC2/On-premises

---

# Step 6 — Create Deployment Group

Deployment group name:
`nextwork-devops-cicd-deploymentgroup`

Configuration:

- Deployment type: In-place
- Deployment setting: CodeDeployDefault.AllAtOnce
- Load balancing: Disabled
- Agent update: Every 14 days

Target instances selected via tag:

Key:
role

Value:
webserver

CodeDeploy detected:
✅ 1 matched EC2 instance

---

# Step 7 — Create First Deployment

Deployment configuration:

- Revision type: Amazon S3
- Revision file type: .zip
- Artifact location: S3 build artifact

Deployment lifecycle executed:

1. ApplicationStop
2. BeforeInstall
3. Install
4. ApplicationStart

Result:

❌ Deployment failed

---

# Step 8 — Diagnose Failure

Failure occurred because:

The S3 artifact did not contain:

- appspec.yml
- deployment scripts

Reason:

CodeBuild had not been rebuilt after adding deployment files.

---

# Step 9 — Rebuild Artifact

Triggered new CodeBuild run.

Build completed successfully.

Verified:

- appspec.yml present
- scripts folder included
- WAR file packaged

---

# Step 10 — Redeploy

Created new deployment using updated artifact.

Lifecycle events executed successfully.

Deployment status:

✅ SUCCEEDED

Accessed EC2 Public IPv4 DNS via HTTP.

Application loaded successfully.

Continuous Deployment pipeline operational.

---

# Step 11 — Simulate Deployment Failure

To test recovery, I intentionally broke:

`stop_server.sh`

Changed:

systemctl → systemctll

Committed and pushed to GitHub.

Rebuilt artifact via CodeBuild.

Created new deployment.

Result:

❌ Deployment failed during ApplicationStop

Error:

ScriptFailed — command not found

---

# Step 12 — Observe Rollback Behavior

Rollback was enabled.

However:

Rollback failed.

Reason:

CodeDeploy reuses latest artifact, not previous successful artifact.

This demonstrated an important DevOps lesson:

Rollback configuration ≠ Artifact version rollback.

---

# Step 13 — Manual Recovery

Steps taken:

1. Fixed typo in stop_server.sh
2. Pushed changes to GitHub
3. Triggered new CodeBuild
4. Retried deployment

Deployment status:

✅ SUCCEEDED

Application restored.

---

# Final Result

✔ Infrastructure provisioned via CloudFormation  
✔ Production EC2 separated from development  
✔ Artifact automatically deployed from S3  
✔ Lifecycle hooks automated installation and startup  
✔ Failure simulated successfully  
✔ Manual recovery executed correctly  
✔ Full CI → CD pipeline implemented  

---

# Key DevOps Concepts Practiced

- Infrastructure as Code
- Automated deployment lifecycle
- EC2 tagging strategy
- Service automation with systemctl
- Reverse proxy configuration
- Deployment troubleshooting
- Rollback limitations
- Manual recovery process
- Separation of dev and prod environments

---

# Reflection

This project demonstrated:

- Deployment automation requires correct artifact packaging.
- Lifecycle scripts control real server behavior.
- IAM permissions must align across services.
- Rollback strategies require version-aware pipelines.
- Recovery procedures are critical DevOps skills.

The pipeline now supports:

Build → Package → Deploy → Recover

Continuous Integration has successfully evolved into Continuous Deployment.
