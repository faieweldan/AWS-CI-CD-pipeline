# AWS-CI-CD-pipeline

A hands-on AWS DevOps learning project that documents my step-by-step journey building a complete CI/CD pipeline for a Java web application using AWS services.

This repository contains project notes and documentation from setting up a web app in the cloud all the way to a fully orchestrated CI/CD pipeline with automated deployment and rollback testing.

---

## Project Overview

This repo documents a multi-part DevOps challenge where I built and connected the following workflow:

**GitHub → CodeBuild → CodeDeploy → CodePipeline → EC2**

Across the project, I practiced:

- Cloud infrastructure setup
- GitHub to AWS integration
- Dependency management with CodeArtifact
- Continuous Integration (CI) with CodeBuild
- Continuous Deployment (CD) with CodeDeploy
- End-to-end CI/CD orchestration with CodePipeline
- Troubleshooting and rollback/recovery workflows

---

## Repository Structure

### `01_setup_webapp.md`
Documents the initial web app setup in AWS (development environment setup and deployment foundation).

### `02_connect_github_to_AWS.md`
Covers connecting GitHub to AWS services so source code can be used in the CI/CD workflow.

### `03_dependencies_CodeArtifact.md`
Documents setting up AWS CodeArtifact for dependency/package management.

### `04_continuous_integration_with_codebuild.md`
Covers Continuous Integration using AWS CodeBuild, including build automation and artifact generation.

### `05_deploy_a_web_app.md`
Documents Continuous Deployment using AWS CodeDeploy:
- CloudFormation infrastructure setup (deployment EC2)
- `appspec.yml` + deployment scripts
- Lifecycle hooks
- Deployment troubleshooting
- Failure simulation and recovery

### `06_CICD_with_awscodepipeline.md`
Documents full CI/CD orchestration using AWS CodePipeline:
- Source → Build → Deploy stages
- GitHub webhook trigger
- Rollback behavior
- Pipeline troubleshooting
- CloudFormation stack update to fix EC2 access issues (SSH/key pair)

---

## AWS Services Used

- **Amazon EC2** (development + deployment servers)
- **AWS CodePipeline** (CI/CD orchestration)
- **AWS CodeBuild** (build automation)
- **AWS CodeDeploy** (deployment automation)
- **AWS CodeArtifact** (package/dependency source)
- **Amazon S3** (artifact storage)
- **AWS CloudFormation** (Infrastructure as Code)
- **AWS IAM** (roles and permissions)
- **AWS CodeConnections** (GitHub integration)

---

## What I Built

By the end of this project, I had a working CI/CD pipeline that:

- Pulls source code from GitHub
- Automatically triggers on push (webhook)
- Builds the application with CodeBuild
- Packages deployment artifacts (`WAR`, `appspec.yml`, scripts)
- Deploys to an EC2 instance using CodeDeploy
- Supports deploy-stage rollback behavior in CodePipeline
- Includes troubleshooting and recovery workflows

---

## Key DevOps Concepts Practiced

- CI/CD pipeline design and orchestration
- Infrastructure as Code (CloudFormation)
- Build artifact flow (Source → Build → Deploy)
- Webhook-driven automation
- CodeDeploy lifecycle hooks (`ApplicationStop`, `BeforeInstall`, `ApplicationStart`)
- Deployment rollback behavior
- Troubleshooting deployment failures using logs
- Safe infrastructure remediation through stack updates
- Separation of development and deployment environments

---

## Notes / Lessons Learned

Some of the most important lessons from this project:

- A pipeline can show the latest commit in Source/Build while deployment still fails due to **target instance state**
- Rollback behavior is useful, but you need to understand **what is actually being rolled back**
- SSH / key pair / security group access matters a lot for debugging EC2 deployment issues
- Updating infrastructure via **CloudFormation stack update** is safer than manually patching resources

---

## Who This Repo Is For

This repo is useful if you are learning:

- AWS DevOps fundamentals
- CI/CD concepts
- CodePipeline / CodeBuild / CodeDeploy integration
- Infrastructure as Code with CloudFormation
- Real-world debugging and recovery in deployment pipelines

---

## Status

✅ Completed through full CI/CD orchestration and rollback testing  
🧪 Includes troubleshooting scenarios and recovery notes for learning purposes

---

## Author

**Rifaie Wildani Nazori**  
Computer Engineering student | Cloud / DevOps learner

---

## Future Improvements (Optional)

- Add architecture diagrams for each part
- Add screenshots for all major steps (Parts 1–6)
- Add a summary table comparing CI vs CD vs CI/CD orchestration
- Add a final “Lessons Learned” page across all parts
