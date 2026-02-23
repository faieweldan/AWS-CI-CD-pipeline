# Part 2 — Connect a GitHub Repository with AWS (Git + EC2)

> 6 Day DevOps Challenge — Day 2  
> Goal: Store the web application code in GitHub and understand version control fundamentals.

---

# Overview

In Part 2 of the DevOps Challenge, I connected my Java web application (built in Part 1 on EC2) to a GitHub repository.

This step is critical because CI/CD pipelines rely on Git repositories as the **source of truth** for application code.

By the end of this project, I successfully:

- Installed Git on my EC2 instance
- Initialized a local Git repository
- Connected the local repository to GitHub
- Committed and pushed code to GitHub
- Configured Git authentication using a GitHub Personal Access Token
- Created a professional README file

<img width="637" height="384" alt="image" src="https://github.com/user-attachments/assets/04f1b831-c257-4ece-8a23-c0e9daae1c00" />

---

# Step 1 — Install and Configure Git

## What is Git?

Git is a distributed version control system that tracks changes in code.

It allows:
- Tracking changes over time
- Collaboration across teams
- Rolling back to previous versions
- Connecting to remote repositories like GitHub

## Install Git (if not already installed)

```bash
sudo yum install git -y
```

Verify installation:

```bash
git --version
```

---

# Step 2 — Initialize a Local Repository

Inside my EC2 project folder:

```bash
cd ~/nextwork-web-project
```

I initialized Git:

```bash
git init
```

## What does `git init` do?

It creates a new hidden `.git` directory that allows Git to start tracking changes in this folder.

(INSERT IMAGE HERE: git init output)

---

# Step 3 — Understand Branches

After initializing, Git created a default branch (usually `main`).

A **branch** in Git is an independent line of development.

Branches allow developers to:
- Work on features safely
- Avoid breaking production code
- Merge changes later

---

# Step 4 — Stage, Commit, and Push Code

To upload my code to GitHub, I used three core Git commands.

---

## 1️⃣ git add

```bash
git add .
```

This adds all project files to the **staging area**.

### What is the staging area?

The staging area is where Git prepares changes before permanently recording them in a commit.

---

## 2️⃣ git commit

```bash
git commit -m "Initial commit - Java web application setup"
```

The `-m` flag allows me to add a commit message directly in the command.

A commit is a saved snapshot of my project at a specific point in time.

---

## 3️⃣ Connect to GitHub and Push

First, I created a repository on GitHub.

Then I linked my local repo to GitHub:

```bash
git remote add origin https://github.com/<username>/<repository-name>.git
```

Then pushed:

```bash
git push -u origin main
```

The `-u` flag sets the upstream branch, meaning future pushes can be done with:

```bash
git push
```

<img width="651" height="287" alt="image" src="https://github.com/user-attachments/assets/ed0898ff-0b5f-4708-b70a-3df2674704ee" />
<img width="638" height="355" alt="image" src="https://github.com/user-attachments/assets/e02cdc16-7d05-48cd-976e-59e21dd6ff35" />

---

# Step 5 — Authentication and GitHub Token

## Why did password authentication fail?

GitHub no longer allows password authentication for Git operations.

Instead, it requires a **Personal Access Token (PAT)**.

## What is a GitHub Token?

A GitHub token is a secure alternative to a password that allows authenticated access to GitHub repositories.

## How I created the token

- Went to GitHub → Settings
- Developer Settings
- Personal Access Tokens
- Generated new token with repository permissions

Then used that token when prompted during `git push`.

<img width="631" height="392" alt="image" src="https://github.com/user-attachments/assets/68e8bf32-240e-432f-9e41-2b8354324953" />

---

# Step 6 — Configure Git Identity

Git requires a name and email to label commits.

I configured my identity:

```bash
git config --global user.name "Your Name"
git config --global user.email "your-email@example.com"
```

This ensures commits are properly attributed.

I verified commit history with:

```bash
git log
```
---

# Step 7 — Make Changes and Observe Git Tracking

To test Git tracking:

1. I modified `index.jsp`
2. Ran:

```bash
git status
```

Git showed modified files.

3. Then:

```bash
git add .
git commit -m "Updated index.jsp content"
git push
```

After pushing, I refreshed GitHub and saw the updates.

<img width="632" height="332" alt="image" src="https://github.com/user-attachments/assets/4a0199a4-eec5-4caf-9ce4-5070b1199c75" />

---

# Step 8 — Create a README File

As a finishing touch, I created a README file in the repository.

## What is a README?

A README file:
- Explains what the project does
- Documents setup steps
- Improves professionalism
- Helps recruiters understand your work

I created the file:

```bash
touch README.md
```

Then added content using nano or VS Code.

## Why Markdown?

README files use **Markdown (.md)** because it allows:
- Headers (#)
- Bold (**text**)
- Code blocks (```)
- Lists (- item)

My README includes:

- Project overview
- Tools used
- Setup instructions
- Architecture explanation
- Future improvements
- Reflection

<img width="628" height="390" alt="image" src="https://github.com/user-attachments/assets/3f613494-6117-4217-ad14-3ffe61e9513d" />

---

# Key Concepts Learned

- Version control fundamentals
- Local vs remote repositories
- Git staging area
- Branching basics
- Commit tracking
- GitHub authentication using tokens
- Importance of documentation

---

# Why This Step Matters for CI/CD

CI/CD pipelines start with a **Git repository trigger**.

When code is pushed to GitHub:
- CodeBuild will build it
- CodeDeploy will deploy it
- CodePipeline will orchestrate it

Without GitHub integration, automation cannot happen.

This step transformed my project from a local cloud experiment into a source-controlled application ready for automation.

---

# Reflection

This project strengthened my understanding of:

- How distributed version control works
- Why Git is essential for DevOps workflows
- The relationship between EC2 (compute) and GitHub (source control)

Now my application is safely stored in GitHub and ready for the next stage.
