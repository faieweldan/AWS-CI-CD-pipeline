# Part 3 — Secure Packages with AWS CodeArtifact

> 6 Day DevOps Challenge — Day 3  
> Focus: Private dependency management + IAM security + package publishing

---

# Overview

In this project, I set up **AWS CodeArtifact** as a private artifact repository for managing Java dependencies.

Instead of pulling dependencies directly from public sources, I configured a secure, controlled repository inside AWS — then connected Maven to it.

By the end of this project, I:

- Created a CodeArtifact domain
- Created a Maven repository with an upstream connection
- Configured IAM policies and roles for EC2 access
- Resolved a permissions error
- Connected Maven using `settings.xml`
- Verified dependency downloads
- Published and retrieved my own custom package

---

# CodeArtifact Repository Setup

## What is CodeArtifact?

**AWS CodeArtifact** is a managed artifact repository service that stores software packages.

Engineering teams use artifact repositories to:

- Control dependency versions
- Improve build reliability
- Cache public packages
- Enforce security policies
- Prevent direct internet downloads in production environments

---

## Create CodeArtifact Repository

### 1️⃣ Create Repository


::contentReference[oaicite:0]{index=0}


Configuration:

- Repository name: `nextwork-devops-cicd`
- Domain name: `nextwork`
- Upstream repository: `maven-central-store`

### Why use an upstream repository?

An upstream repository allows CodeArtifact to:

- Fetch packages from Maven Central
- Cache them locally
- Improve build speed
- Maintain reliability even if Maven Central goes down

This creates the flow:

Application → CodeArtifact → Maven Central (if needed)

<img width="764" height="444" alt="image" src="https://github.com/user-attachments/assets/60caa682-0823-4b43-a43d-19791e5c34ad" />


---

# CodeArtifact Security — IAM Configuration

When attempting to retrieve an authorization token, I received:

```
Unable to locate credentials
```

This happened because my EC2 instance did not yet have permission to access CodeArtifact.

AWS follows the **Principle of Least Privilege**, meaning no service has access unless explicitly granted.

---

## Create IAM Policy


::contentReference[oaicite:1]{index=1}


I created a new IAM policy:

**Policy Name:** `codeartifact-nextwork-consumer-policy`

### Policy JSON

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "codeartifact:GetAuthorizationToken",
        "codeartifact:GetRepositoryEndpoint",
        "codeartifact:ReadFromRepository"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "sts:GetServiceBearerToken",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "sts:AWSServiceName": "codeartifact.amazonaws.com"
        }
      }
    }
  ]
}
```

### What this policy grants

- Retrieve authorization tokens
- Access repository endpoints
- Read packages
- Obtain temporary service bearer tokens

---

## Create IAM Role for EC2


::contentReference[oaicite:2]{index=2}


Steps:

1. Created a new role for EC2
2. Attached `codeartifact-nextwork-consumer-policy`
3. Named role: `EC2-instance-nextwork-cicd`
4. Attached role to my EC2 instance

### Why use IAM roles instead of access keys?

IAM roles:
- Provide temporary credentials
- Rotate automatically
- Avoid hardcoding secrets
- Follow AWS security best practices

After attaching the role, I re-ran the token command — and it succeeded.

<img width="759" height="454" alt="image" src="https://github.com/user-attachments/assets/d5eefefd-0753-48e7-a1e8-fabf9cfcd05a" />


---

# Connecting Maven to CodeArtifact

To allow Maven to authenticate with CodeArtifact, I configured a `settings.xml` file.

The file included:

- `<servers>` for authentication
- `<profiles>` for repository configuration
- `<mirrors>` for fallback routing

Then I compiled using:

```bash
mvn -s settings.xml compile
```

<img width="767" height="430" alt="image" src="https://github.com/user-attachments/assets/68693de0-4f20-4192-9334-66fb378b96c3" />

---

# Verify Maven Integration


::contentReference[oaicite:3]{index=3}


During compilation, I observed:

```
Downloading from nextwork-devops-cicd
```

Then:

```
BUILD SUCCESS
```

After refreshing CodeArtifact, I saw Maven dependencies automatically stored in the repository.

This confirmed:

- Authentication worked
- IAM role worked
- Upstream repository worked
- Package caching worked

<img width="764" height="444" alt="image" src="https://github.com/user-attachments/assets/d07a410e-ead3-4fe5-8d66-c2f1991391bf" />


---

Publish My Own Package

To experience the full lifecycle of CodeArtifact, I created and uploaded a custom package.

---

## Create Package

```bash
echo "Hellooooo this is a test package!" > secret-mission.txt
tar -czvf secret-mission.tar.gz secret-mission.txt
```

---

## Generate Security Hash

```bash
export ASSET_SHA256=$(sha256sum secret-mission.tar.gz | awk '{print $1;}')
```

A SHA256 hash ensures:

- File integrity
- No tampering
- Secure upload validation

---

## Publish Package

```bash
aws codeartifact publish-package-version \
  --domain nextwork \
  --repository nextwork-devops-cicd \
  --format generic \
  --namespace secret-mission \
  --package secret-mission \
  --package-version 1.0.0 \
  --asset-content secret-mission.tar.gz \
  --asset-name secret-mission.tar.gz \
  --asset-sha256 $ASSET_SHA256
```

<img width="761" height="445" alt="image" src="https://github.com/user-attachments/assets/4c43499a-9b58-4edf-bfc0-d691b5592948" />


---

## Verify and Download Package


::contentReference[oaicite:4]{index=4}


Downloaded it back:

```bash
aws codeartifact get-package-version-asset \
  --domain nextwork \
  --repository nextwork-devops-cicd \
  --format generic \
  --namespace secret-mission \
  --package secret-mission \
  --package-version 1.0.0 \
  --asset secret-mission.tar.gz \
  secret-mission.tar.gz
```

Extracted:

```bash
tar -xzvf secret-mission.tar.gz
cat secret-mission.txt
```

Successfully retrieved my original message.

Full lifecycle complete:
Publish → Store → Retrieve → Validate

---

# Key Concepts Learned

- Artifact repository architecture (Domain → Repository → Package → Version)
- IAM policy vs IAM role
- Principle of Least Privilege
- Temporary token authentication
- Maven repository configuration
- Dependency caching strategy
- Package publishing and integrity validation

---

# Why This Matters in CI/CD

CodeArtifact enables:

- Reproducible builds
- Controlled dependency versions
- Secure package distribution
- Enterprise-grade DevOps workflows
