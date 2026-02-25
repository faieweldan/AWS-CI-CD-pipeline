# Part 6 — Continuous Delivery Orchestration with AWS CodePipeline

> 6 Day DevOps Challenge — Day 6  
> Focus: pipeline orchestration, GitHub webhooks, CI/CD stage integration, automated deploy trigger, rollback behavior, and troubleshooting.

---

# Overview

In this project, I connected all the components from previous parts into a single **end-to-end CI/CD pipeline** using **AWS CodePipeline**.

The goal was to:

- Orchestrate Source → Build → Deploy in one workflow
- Trigger the pipeline automatically from GitHub pushes
- Reuse the existing CodeBuild and CodeDeploy setup
- Validate deploy-stage rollback behavior
- Troubleshoot deployment failures in a real pipeline execution
- Fix infrastructure access limitations (SSH / EC2 access) safely using CloudFormation stack updates

This project transformed the setup from:

Separate AWS services (manually triggered) → **One automated CI/CD pipeline**

---

# Architecture Overview

Pipeline flow:

GitHub (master branch) → CodePipeline Source Stage → CodeBuild Build Stage → CodeDeploy Deploy Stage → EC2 (Deployment Server)

Integrated services:

- **GitHub (via GitHub App / CodeConnections)** — source code and webhook trigger
- **CodePipeline** — orchestration and stage execution
- **CodeBuild** — build/package artifact creation
- **CodeArtifact** — dependency source/cache for build process
- **CodeDeploy** — in-place deployment to EC2
- **EC2 (deployment instance)** — production-like deployment target
- **S3 (artifact store)** — pipeline/build artifacts between stages

---

# Step 1 — Create a New CodePipeline

Started in the **CodePipeline console** and selected:

- **Create pipeline**
- **Build custom pipeline**

Pipeline settings configured:

- **Pipeline name:** `rifaie-devops-cicd` (or `nextwork-devops-cicd` depending on your setup)
- **Pipeline type:** V2
- **Execution mode:** `SUPERSEDED`
- **Service role:** New service role (default generated)
- **Artifact store / encryption / variables:** left as default

Why `SUPERSEDED` mode?

- Ensures only the latest code change runs
- Cancels older in-progress executions
- Reduces wasted build/deploy runs during rapid commits

---

# Step 2 — Configure Source Stage (GitHub)

Configured the Source stage to pull code from GitHub:

- **Source provider:** GitHub (via GitHub App)
- **Connection:** Existing CodeConnection (GitHub connection)
- **Repository:** `<github-username>/<repo-name>`
- **Default branch:** `master`
- **Output artifact format:** CodePipeline default (`CODE_ZIP`)
- **Detect changes:** Webhook events enabled ✅

### Why this matters
The Source stage is what makes the pipeline “continuous.”  
Every push to `master` automatically triggers a new pipeline execution.

---

# Step 3 — Configure Build Stage (CodeBuild)

Configured the Build stage to use the existing CodeBuild project:

- **Build provider:** AWS CodeBuild
- **Project name:** `nextwork-devops-cicd`
- **Input artifact:** `SourceArtifact`
- **Defaults kept** for environment variables and build settings

### What happens here
CodeBuild takes the source code from GitHub, runs `buildspec.yml`, and creates a deployable artifact (WAR + appspec + scripts) for the next stage.

---

# Step 4 — Skip Test Stage

For this project, I selected:

- **Skip test stage**

### Why skip?
This challenge focuses on pipeline orchestration (Source → Build → Deploy).  
In production, a Test stage is strongly recommended (unit/integration/UI tests).

---

# Step 5 — Configure Deploy Stage (CodeDeploy)

Configured Deploy stage with existing CodeDeploy resources:

- **Deploy provider:** AWS CodeDeploy
- **Input artifact:** `BuildArtifact`
- **Application name:** `nextwork-devops-cicd`
- **Deployment group:** `nextwork-devops-cicd-deploymentgroup`
- **Automatic rollback on stage failure:** Enabled ✅

### What this stage does
CodePipeline passes the build artifact to CodeDeploy, which deploys the application to the EC2 deployment instance and runs lifecycle hooks (`ApplicationStop`, `BeforeInstall`, `ApplicationStart`).

---

# Step 6 — Review and Create Pipeline

Reviewed the pipeline configuration:

## Pipeline settings
- Pipeline type: V2
- Execution mode: SUPERSEDED
- Service role: New service role
- Artifact store: Default location

## Source stage
- GitHub (via GitHub App)
- Webhook enabled
- Default branch: `master`

## Build stage
- AWS CodeBuild
- Existing project selected

## Deploy stage
- AWS CodeDeploy
- Existing application + deployment group selected
- Auto rollback enabled

Created the pipeline.  
CodePipeline automatically started the first execution.

---

# Step 7 — Verify First Pipeline Execution

After creation, the pipeline executed automatically.

Observed the stage progress in the pipeline diagram:

- **Source** → ✅ Success
- **Build** → ✅ Success
- **Deploy** → (initially failed during troubleshooting scenario, then later succeeded)

I also reviewed execution details under:

- **Executions tab**
- Stage-specific details
- Source commit reference (linked back to GitHub commit)

This confirmed the pipeline was correctly pulling the latest commit from GitHub and triggering downstream stages.

---

# Step 8 — Test Auto-Trigger with a Code Change

To test webhook automation, I made a code change in:

`src/main/webapp/index.jsp`

Added a visible line in the `<body>` section, then committed and pushed:

```bash
git add .
git commit -m "Update index.jsp with a new line to test CodePipeline"
git push origin master
