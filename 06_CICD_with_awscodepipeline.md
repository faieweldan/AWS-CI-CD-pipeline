# Part 6 — Continuous Delivery Orchestration with AWS CodePipeline

> 6 Day DevOps Challenge — Day 6  
> Focus: pipeline orchestration, GitHub webhooks, CI/CD stage integration, automated deploy trigger, rollback behavior, and troubleshooting.

<img width="684" height="247" alt="image" src="https://github.com/user-attachments/assets/bf218461-7376-45f3-a1cd-cfbd5e44d2bd" />


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

<img width="757" height="467" alt="image" src="https://github.com/user-attachments/assets/030e75e2-3b42-4ce8-b322-220e414f9262" />
<img width="761" height="466" alt="image" src="https://github.com/user-attachments/assets/36c187f9-a7ae-4630-9fee-e2db12604a66" />

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

<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/2e4dbafd-e8e9-4aa8-8b0d-f6e5a3700afe" />
<img width="764" height="466" alt="image" src="https://github.com/user-attachments/assets/64278088-0615-46d6-9915-b56d8bfedf39" />



---

# Step 3 — Configure Build Stage (CodeBuild)

Configured the Build stage to use the existing CodeBuild project:

- **Build provider:** AWS CodeBuild
- **Project name:** `nextwork-devops-cicd`
- **Input artifact:** `SourceArtifact`
- **Defaults kept** for environment variables and build settings

### What happens here
CodeBuild takes the source code from GitHub, runs `buildspec.yml`, and creates a deployable artifact (WAR + appspec + scripts) for the next stage.

<img width="761" height="472" alt="image" src="https://github.com/user-attachments/assets/3244ec35-b332-44bf-9ca4-37e07688c9d0" />


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

<img width="764" height="468" alt="image" src="https://github.com/user-attachments/assets/c8ed309f-5059-49f9-a18c-5fe1f98d29fb" />

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

<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/7f933f28-1215-477e-a480-0fdde6c6ba4f" />


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

Added a visible line in the `<body>` section, then committed and pushed

<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/1f3a9d28-50a7-4c13-816e-8b2990c181f1" />
<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/85f18a74-fc8c-4644-a878-9ecbefc9b894" />

### Result
- GitHub push triggered CodePipeline automatically ✅
- New pipeline execution started without manual intervention ✅
- Source stage displayed the latest commit message ✅

This validated the GitHub webhook integration.

---

# Step 9 — Real Troubleshooting: Deploy Stage Failure (Lifecycle Script)

During deployment testing (including rollback/failure simulation), CodeDeploy failed in:

- **Lifecycle event:** `ApplicationStop`
- **Script:** `scripts/stop_server.sh`

Error observed in CodeDeploy logs:
- script failed
- command not found (`systemctll` typo)

This typo was **intentionally introduced earlier** to validate rollback behavior.

### Important observation
Even after fixing the typo in GitHub and seeing the correct commit flow through:
- **Source stage** ✅ (latest commit shown)
- **Build stage** ✅ (latest commit shown)

the **Deploy stage still failed** with the old behavior on the deployment EC2.

This created confusion because the pipeline clearly showed the newer commit.

<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/98b44037-117e-49af-918e-f2d059e2c060" />

---

# Step 10 — Root Cause and Why It Happened

The issue was not that Source or Build was using the wrong commit.

The real problem was the **deployment target EC2 instance state** and access limitations:

- The deployment EC2 had **no key pair assigned**
- **EC2 Instance Connect** was failing
- I could not SSH in to inspect or manually verify files/services
- The instance had previous deployment state that complicated troubleshooting

This made it difficult to directly confirm what CodeDeploy was using on the target host.

### DevOps lesson
Pipeline stage commit IDs can be correct, but deployment failures may still be caused by:
- target instance state
- lifecycle hook behavior
- deployment cache/state on host
- inability to access the instance for validation

<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/57ad26f1-eebc-4a89-a37c-1fe2acb0fca6" />
<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/5df94e8c-4062-48fe-83ff-82cb0d1c5a6f" />

---

# Step 11 — Fix: Update CloudFormation Stack (Do Not Rebuild Everything Manually)

Instead of deleting and manually rebuilding services, I fixed the infrastructure properly using **CloudFormation stack update**.

### What I changed
I updated the deployment EC2 CloudFormation template to include:

- **EC2 KeyPair parameter**
- **SSH ingress rule (port 22) restricted to My IP**
- Corrected YAML syntax issues during template update

### Why this was the right fix
- Preserved the architecture and IaC approach
- Avoided breaking the rest of the CI/CD chain unnecessarily
- Replaced/updated the deployment EC2 in a controlled way
- Made the instance accessible for SSH if needed later

<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/0d2b5f76-5c36-4711-b6b9-d7a9f5f98f7b" />
<img width="1440" height="900" alt="image" src="https://github.com/user-attachments/assets/6e5cb0f7-0a34-4102-a2c9-e74b1d36bfe8" />

---

# Step 12 — Rerun Pipeline After Infrastructure Fix

After the CloudFormation stack update completed:

1. Confirmed the deployment EC2 was replaced/updated
2. Verified the new instance was the correct deployment target (same tag strategy: `role=webserver`)
3. Returned to CodePipeline
4. Clicked **Release change**

### Result
- Source stage: ✅ Success
- Build stage: ✅ Success
- Deploy stage: ✅ Success

The deployment succeeded after the infrastructure fix and rerun.

<img width="684" height="247" alt="image" src="https://github.com/user-attachments/assets/bf218461-7376-45f3-a1cd-cfbd5e44d2bd" />

---

# Step 13 — Verify Automated Deployment

After the successful pipeline run:

- Opened the Deploy stage details
- Followed the CodeDeploy deployment link
- Located the target EC2 instance
- Retrieved the **Public IPv4 DNS**
- Opened the web app in the browser

Verified that the latest code change (new line in `index.jsp`) was deployed successfully.

This confirmed the full pipeline worked end-to-end:

**GitHub push → CodePipeline → CodeBuild → CodeDeploy → EC2 update**

<img width="790" height="404" alt="image" src="https://github.com/user-attachments/assets/77daf9ec-c533-4af7-9f00-485b6c0170d1" />

---

# Step 14 — Trigger Rollback in CodePipeline (Deploy Stage Only)

To test recovery behavior inside CodePipeline, I manually triggered a rollback on the **Deploy** stage.

### Steps performed
1. Opened the **CodePipeline console**
2. Selected my pipeline (`nextwork-devops-cicd`)
3. Located the **Deploy** stage in the pipeline diagram
4. Clicked the **three dots (...)** on the Deploy stage
5. Selected **Start rollback**
6. In the **Rollback to** dialog, selected the **previous execution ID**
7. Clicked **Start rollback**

### What rollback means
A rollback means reverting to an earlier working deployment version.

In this case, I rolled back **only the Deploy stage**, which means:

- **Source stage** stays on the latest commit
- **Build stage** stays on the latest build
- **Deploy stage** reverts the application version on the EC2 server to a previous successful deployment

### Why this is useful
Deploy-stage-only rollback is useful when:
- source code and build are valid
- but the deployment result causes runtime issues on the target server

Examples:
- A deployment script has a bug (`install_dependencies.sh`, `start_server.sh`, etc.)
- A third-party dependency/service becomes unstable during deployment
- The latest release causes performance/runtime problems that weren’t caught earlier

### Result
- CodePipeline showed **Rollback in progress** under the Deploy stage
- Deploy stage completed successfully again ✅
- The Deploy stage now referenced the **previous successful deployment execution**
- Source/Build still reflected the latest commit (important behavior to understand)

---

# Step 15 — Verify Rollback in the Web Application

To confirm the rollback really worked, I refreshed the same web app URL (same EC2 Public IPv4 DNS).

### Expected behavior after rollback
The extra line added earlier to `index.jsp` was no longer visible:

```html
<p>If you see this line, that means your latest changes are automatically deployed into production by CodePipeline!</p>


