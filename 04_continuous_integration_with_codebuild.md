# Part 4 — Continuous Integration with AWS CodeBuild

> 6 Day DevOps Challenge — Day 4  
> Focus: Build automation, artifact packaging, IAM permissions, and CI testing.

---

# Overview

In this project, I built a complete **Continuous Integration (CI) pipeline** using AWS CodeBuild.

The goal was to:

- Automatically pull source code from GitHub
- Compile and package the Java web application
- Store build artifacts in Amazon S3
- Integrate AWS CodeArtifact for dependency management
- Automate testing inside the pipeline
- Troubleshoot IAM permission failures

This project transformed the application from manually built → automatically built and tested.

---

# Architecture Overview


::contentReference[oaicite:0]{index=0}


Pipeline flow:

GitHub → CodeBuild → CodeArtifact (dependencies) → S3 (artifact storage)

<img width="853" height="261" alt="image" src="https://github.com/user-attachments/assets/df62c7cc-1744-4d4c-8380-3159492641b5" />


---

# Step 1 — Create CodeBuild Project

I created a new CodeBuild project with the following configuration:

**Project Name:** `nextwork-devops-cicd`  
**Source Provider:** GitHub  
**Credential Type:** GitHub App (via AWS CodeConnections)  
**Environment:**  
- Managed Image  
- Amazon Linux  
- Standard runtime  
- Corretto 8  
- On-demand compute  

Why Managed Image?
- No need to configure build servers
- Pre-installed runtimes
- Faster setup
- Fully managed by AWS

<img width="635" height="385" alt="image" src="https://github.com/user-attachments/assets/d8b3ee42-37b3-46b1-a377-8d9d69580857" />

---

# Step 2 — Connect GitHub via CodeConnections


::contentReference[oaicite:1]{index=1}


Instead of using Personal Access Tokens, I used **GitHub App authentication** through AWS CodeConnections.

Benefits:
- Secure authentication
- No manual token rotation
- AWS-managed integration
- Recommended production approach

<img width="631" height="386" alt="image" src="https://github.com/user-attachments/assets/3025874d-6a8e-41b4-a1bb-7a0fbcca1e12" />

---

# Step 3 — Configure Build Environment

Environment configuration:

- Provisioning model: On-demand
- Compute type: EC2
- Image: `aws/codebuild/amazonlinux-x86_64-standard:corretto8`
- New service role auto-created

CloudWatch logging enabled for monitoring build phases.

---

# Step 4 — Configure Artifact Storage in S3


::contentReference[oaicite:2]{index=2}


I created a dedicated S3 bucket:

`nextwork-devops-cicd-<myname>`

Artifact settings:

- Type: Amazon S3
- Packaging: Zip
- Artifact name: `nextwork-devops-cicd-artifact`

The output artifact is a `.war` file packaged inside a `.zip` file.

---

# Step 5 — First Build Attempt (Expected Failure)

When running the first build:

Build failed with:

```
YAML_FILE_ERROR: YAML file does not exist
```

This happened because CodeBuild expects a `buildspec.yml` file in the root of the repository.

This failure confirmed:
- Source connection worked
- CodeBuild triggered correctly
- The issue was configuration-related

---

# Step 6 — Create buildspec.yml

I created `buildspec.yml` in the project root.

```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      java: corretto8

  pre_build:
    commands:
      - echo Logging in to AWS CodeArtifact...
      - CODEARTIFACT_AUTH_TOKEN=`aws codeartifact get-authorization-token --domain nextwork --domain-owner <ACCOUNT_ID> --region <REGION> --query authorizationToken --output text`
      - export CODEARTIFACT_AUTH_TOKEN

  build:
    commands:
      - echo Build started on `date`
      - mvn clean install -s settings.xml

  post_build:
    commands:
      - echo Packaging artifact
      - mvn package -s settings.xml

artifacts:
  files:
    - target/*.war
```

After pushing to GitHub, I reran the build.

<img width="759" height="469" alt="image" src="https://github.com/user-attachments/assets/08f81fcf-7e04-4751-aa9a-4d9c729da672" />


---

# Step 7 — Second Build Failure (IAM Issue)

The build failed again.

This time, CodeBuild could not access CodeArtifact.

Root cause:
The CodeBuild service role did not have permission to read from CodeArtifact.

---

# Step 8 — Fix CodeBuild IAM Permissions


::contentReference[oaicite:3]{index=3}


I attached:

`codeartifact-nextwork-consumer-policy`

to the CodeBuild service role:

`codebuild-nextwork-devops-cicd-service-role`

After updating permissions, I reran the build.

Result: ✅ SUCCESS

<img width="760" height="460" alt="image" src="https://github.com/user-attachments/assets/5ad46844-3c62-4694-b811-d979255c7477" />


---

# Step 9 — Verify Artifact in S3


::contentReference[oaicite:4]{index=4}


Inside S3:

- Found artifact zip file
- Downloaded artifact
- Confirmed `.war` file present inside

This validated:

- Build completed
- Packaging worked
- Artifact uploaded correctly

---

# Step 10 — Automate Testing

To extend CI beyond compilation, I created:

`run-tests.sh`

```bash
#!/bin/bash
echo "Running validation tests..."

if [ -d "src" ]; then
  echo "PASS: src directory exists"
else
  exit 1
fi

if [ -f "src/main/webapp/index.jsp" ]; then
  echo "PASS: index.jsp exists"
else
  exit 1
fi

exit 0
```

Updated `buildspec.yml`:

```yaml
  build:
    commands:
      - chmod +x run-tests.sh
      - ./run-tests.sh
      - mvn -s settings.xml compile
```

Now CodeBuild:

1. Runs tests
2. Fails pipeline if tests fail
3. Builds only if tests pass

This upgraded the project from build automation → true Continuous Integration.

<img width="761" height="474" alt="image" src="https://github.com/user-attachments/assets/91635f14-18c2-49b8-b380-b94c51bfad46" />


---

# Final Result

✔ CodeBuild pulls from GitHub  
✔ Dependencies resolved via CodeArtifact  
✔ IAM permissions configured correctly  
✔ Automated tests executed  
✔ Artifact packaged as WAR  
✔ Artifact stored in S3  
✔ CloudWatch logs enabled  

CI pipeline fully operational.

---

# Key DevOps Concepts Practiced

- CI architecture
- Infrastructure as configuration
- GitHub App authentication
- IAM role troubleshooting
- Build automation
- Artifact packaging
- S3 artifact storage
- Automated testing inside CI
- Debugging failed pipeline phases

---

# Reflection

This project demonstrated how:

- Small misconfigurations (like missing IAM permissions) can break pipelines.
- CI pipelines require secure service-to-service communication.
- Automated testing increases reliability.
- Infrastructure and code must be aligned.

The application is now automatically built, tested, and packaged on demand.

Deploy the packaged artifact using AWS CodeDeploy and move from CI → CD (Continuous Deployment).

---
