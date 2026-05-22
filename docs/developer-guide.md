# BEM13 ECS CICD Lab — Developer Guide

**Author:** Timothy Nlenjibi  
**Lab:** BEM13 — Running Containers on AWS  
**Stack:** Spring Boot 3.4.5 · Java 21 · Docker · AWS ECS Fargate · CloudFormation GitSync · GitHub Actions OIDC

---

## Table of Contents

1. [What This Project Does](#1-what-this-project-does)
2. [Repository Structure](#2-repository-structure)
3. [The Java Application](#3-the-java-application)
4. [Running the App Locally](#4-running-the-app-locally)
5. [Docker Image](#5-docker-image)
6. [GitHub Actions Workflows](#6-github-actions-workflows)
7. [How OIDC Authentication Works](#7-how-oidc-authentication-works)
8. [AWS Infrastructure — 9-Stack CloudFormation](#8-aws-infrastructure--9-stack-cloudformation)
9. [Network Architecture](#9-network-architecture)
10. [Security Groups](#10-security-groups)
11. [The Full Deployment Pipeline](#11-the-full-deployment-pipeline)
12. [Blue/Green Deployment Explained](#12-bluegreen-deployment-explained)
13. [CloudFormation GitSync](#13-cloudformation-gitsync)
14. [Environment Differences](#14-environment-differences)
15. [One-Time AWS Setup Checklist](#15-one-time-aws-setup-checklist)
16. [Commit and PR Standards](#16-commit-and-pr-standards)
17. [Troubleshooting](#17-troubleshooting)

---

## 1. What This Project Does

This lab deploys a containerized Spring Boot application to **AWS ECS Fargate** using a fully automated CI/CD pipeline. Every push to a branch builds a Docker image, pushes it to ECR, and triggers a zero-downtime **blue/green deployment** — with no manual steps and no long-lived AWS credentials anywhere.

Three environments (`dev`, `test`, `prod`) are separate AWS stacks deployed from the same CloudFormation templates, each in its own isolated VPC.

---

## 2. Repository Structure

This project uses **two separate GitHub repositories**:

### Infrastructure Repository — `ecs-cicd-infra`

```
ecs-cicd-infra/
├── templates/                        # 9 modular CloudFormation stacks
│   ├── network.yaml                  # Stack 1: VPC, subnets, IGW, NAT
│   ├── security.yaml                 # Stack 2: Security groups
│   ├── ecr.yaml                      # Stack 3: ECR repository
│   ├── iam.yaml                      # Stack 4: IAM roles
│   ├── endpoints.yaml                # Stack 5: VPC endpoints
│   ├── alb.yaml                      # Stack 6: ALB + target groups
│   ├── ecs.yaml                      # Stack 7: ECS cluster, service, scaling
│   ├── codedeploy.yaml               # Stack 8: CodeDeploy blue/green
│   └── pipeline.yaml                 # Stack 9: EventBridge + CodePipeline
├── deployments/
│   ├── dev/   01-network.yaml … 09-pipeline.yaml
│   ├── test/  01-network.yaml … 09-pipeline.yaml
│   └── prod/  01-network.yaml … 09-pipeline.yaml
├── parameters/
│   ├── dev.json
│   ├── test.json
│   └── prod.json
├── deployment.yaml                   # GitSync root config
├── architecture.md                   # Mermaid network diagrams
└── docs/
    └── developer-guide.md            # This file
```

### Application Repository — `ecs-cicd-app`

```
ecs-cicd-app/
├── src/
│   ├── main/java/com/example/bem13/
│   │   ├── Bem13Application.java
│   │   ├── controller/
│   │   │   ├── InfoController.java   # GET /api/info
│   │   │   └── QuoteController.java  # GET /api/quotes, /random, /{id}, /health
│   │   └── model/
│   │       ├── AppInfo.java
│   │       └── Quote.java
│   ├── main/resources/
│   │   ├── application.yaml
│   │   └── static/index.html         # Dashboard UI
│   └── test/java/com/example/bem13/
│       ├── QuoteControllerTest.java
│       └── InfoControllerTest.java
├── codedeploy/
│   ├── appspec.yaml                  # CodeDeploy ECS spec
│   └── taskdef.json                  # Task definition (<IMAGE1_NAME> placeholder)
├── pom.xml                           # Spring Boot 3.4.5, Java 21
├── Dockerfile                        # eclipse-temurin:21-jre-alpine
└── .github/workflows/
    ├── ci.yml                        # Build + test on push to dev/test/main
    └── deploy.yml                    # OIDC → ECR push on push to dev/test/main
```

---

## 3. The Java Application

### Technology

| Item | Value |
|---|---|
| Framework | Spring Boot 3.4.5 |
| Java | 21 (LTS) |
| Build | Maven |
| Port | 8080 |
| API docs | SpringDoc OpenAPI (`/swagger-ui.html`) |

### API Endpoints

| Method | Path | Description |
|---|---|---|
| GET | `/` | Dashboard UI (served from `static/index.html`) |
| GET | `/health` | Returns plain `OK` — used by ALB health checks |
| GET | `/api/info` | Returns hostname, environment, Java version, author, lab name, start time |
| GET | `/api/quotes` | All 10 quotes |
| GET | `/api/quotes/random` | One random quote |
| GET | `/api/quotes/{id}` | Quote by ID, 404 if not found |
| GET | `/swagger-ui.html` | Interactive API docs |

### AppInfo Response Example

```json
{
  "hostname": "ip-10-0-3-47",
  "environment": "prod",
  "javaVersion": "21.0.3",
  "author": "Timothy Nlenjibi",
  "labName": "BEM13 ECS CICD LAB",
  "startTime": "2026-05-12T16:02:32.239Z"
}
```

### Active Profile

`SPRING_PROFILES_ACTIVE` is injected by the ECS task definition per environment.
Locally it defaults to `dev` (configured in `application.yaml`).

---

## 4. Running the App Locally

### Prerequisites

- Java 21 (`java -version`)
- Maven 3.9+ (`mvn -version`)

### Commands

```bash
# Build the JAR (skip tests)
mvn clean package -DskipTests -B

# Run
java -jar target/bem13app-0.0.1-SNAPSHOT.jar

# Or via Maven plugin
mvn spring-boot:run
```

Open `http://localhost:8080` to see the dashboard.

### Run the tests

```bash
# All 6 tests
mvn test -B

# Single test class
mvn test -Dtest=QuoteControllerTest -B
mvn test -Dtest=InfoControllerTest  -B
```

### Quick API check

```bash
curl http://localhost:8080/health
# OK

curl http://localhost:8080/api/info
# {"hostname":"localhost","environment":"dev",...}

curl http://localhost:8080/api/quotes/random
# {"id":7,"text":"You build it, you run it.",...}
```

---

## 5. Docker Image

### Dockerfile

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/bem13app-0.0.1-SNAPSHOT.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

`eclipse-temurin:21-jre-alpine` uses the JRE (not JDK) to keep the image small and reduces attack surface.

### Build and run locally with Docker

```bash
mvn clean package -DskipTests -B
docker build -t bem13app:local .
docker run -p 8080:8080 -e SPRING_PROFILES_ACTIVE=dev bem13app:local
```

### Image tagging strategy

| Tag | Example | Purpose |
|---|---|---|
| `{env}-{sha}` | `prod-a1b2c3d4` | Immutable — points to exact commit |
| `{env}-latest` | `prod-latest` | Mutable pointer — watched by EventBridge |

Both tags are pushed on every deploy.

---

## 6. GitHub Actions Workflows

### `ci.yml` — Build and Test

**Triggers:** push to `dev`, `test`, or `main`; pull requests targeting `main`

```
Push code
    ↓
GitHub runner (Ubuntu) spins up
    ↓
Install Java 21 (Temurin) with Maven cache
    ↓
mvn clean verify  (compile + 6 tests)
    ↓
Upload surefire test results as artifact
    ↓
Runner destroyed
```

### `deploy.yml` — Build and Push to ECR

**Triggers:** push to `dev`, `test`, or `main`

```
Push to branch
    ↓
Build JAR: mvn clean package -DskipTests
    ↓
Assume AWS role via OIDC (no passwords — see section 7)
    ↓
Login to ECR
    ↓
Calculate tags:
  dev branch   → dev-{sha}   + dev-latest
  test branch  → test-{sha}  + test-latest
  main branch  → prod-{sha}  + prod-latest
    ↓
IMPORTANT: Upload deploy-artifacts.zip to S3 BEFORE ECR push
  (prevents EventBridge race condition)
    ↓
docker build + push both tags to ECR
    ↓
AWS takes over automatically (see section 11)
```

### Why S3 before ECR?

EventBridge fires the moment the ECR push completes. CodePipeline immediately fetches `{env}/deploy-artifacts.zip` from S3. Uploading to S3 first guarantees the artifacts are always there when the pipeline starts.

---

## 7. How OIDC Authentication Works

OIDC lets GitHub prove to AWS "I am a workflow running in repo X on branch Y" without any stored password or access key.

```
1. GitHub generates a short-lived JWT token
   (contains: repo name, branch, workflow name, run ID)

2. Workflow calls: sts:AssumeRoleWithWebIdentity
   presenting the JWT to the OIDC provider in AWS IAM

3. AWS validates the JWT against GitHub's public keys
   and checks the trust policy conditions:
     aud  = "sts.amazonaws.com"
     sub  = "repo:nlenjibi/ecs-cicd-app:ref:refs/heads/*"

4. AWS returns temporary credentials (15 min TTL)

5. Workflow uses them to push to ECR and upload to S3

6. Credentials expire automatically — nothing to rotate
```

### GitHub secret required

```
Settings → Secrets → Actions → New repository secret
Name:  AWS_ROLE_ARN
Value: arn:aws:iam::<AccountId>:role/<env>-bem13-github-actions-role
```

No other AWS secrets are needed.

---

## 8. AWS Infrastructure — 9-Stack CloudFormation

### Stack dependency order

```
1. network   →  2. security
             →  5. endpoints
             →  6. alb
3. ecr       →  7. ecs       →  8. codedeploy  →  9. pipeline
4. iam       →  7. ecs
             →  8. codedeploy
             →  9. pipeline
             →  6. alb
```

### Stack overview

| # | Template | Creates |
|---|---|---|
| 1 | `network.yaml` | VPC (10.0.0.0/16), 2 public + 2 private subnets across 2 AZs, IGW, NAT, route tables |
| 2 | `security.yaml` | ALB SG (port 80 public, 9090 VPC-only), ECS SG (port 8080 from ALB only), Endpoint SG |
| 3 | `ecr.yaml` | ECR repository `bem13-java-app`, lifecycle rules (keep 15 per env, expire untagged after 1 day) |
| 4 | `iam.yaml` | EcsTaskExecutionRole, EcsTaskRole, CodePipelineRole, CodeDeployRole, EventBridgeRole, GitHubActionsRole |
| 5 | `endpoints.yaml` | VPC interface endpoints: ECR API, ECR DKR, CloudWatch Logs; S3 gateway endpoint |
| 6 | `alb.yaml` | Internet-facing ALB, BlueTargetGroup, GreenTargetGroup, production listener (port 80), test listener (port 9090) |
| 7 | `ecs.yaml` | CloudWatch log group, ECS cluster (Container Insights), task definition, Fargate service (CODE_DEPLOY), auto scaling CPU 70% |
| 8 | `codedeploy.yaml` | CodeDeploy application, ECS blue/green deployment group |
| 9 | `pipeline.yaml` | S3 artifact bucket (versioned + KMS), EventBridge rule (ECR push → start pipeline), CodePipeline |

### Cross-stack reference example

```yaml
# Stack 7 (ecs.yaml) imports from stack 1 (network.yaml):
Subnets:
  - Fn::ImportValue: !Sub "${Environment}-${ProjectName}-PrivateSubnet1Id"
  - Fn::ImportValue: !Sub "${Environment}-${ProjectName}-PrivateSubnet2Id"
```

### Stack naming convention

```
{env}-bem13-{component}

Examples:
  dev-bem13-network
  prod-bem13-ecs
  test-bem13-pipeline
```

---

## 9. Network Architecture

```
Internet
    │
    ▼
Internet Gateway
    │
┌───────────────────────────────────────────┐
│              VPC  10.0.0.0/16             │
│                                           │
│  Public Subnet 1 (10.0.1.0/24) — AZ-a   │  ← ALB node
│  Public Subnet 2 (10.0.2.0/24) — AZ-b   │  ← ALB node
│            NAT Gateway (AZ-a)             │
│                   │                       │
│  Private Subnet 1 (10.0.3.0/24) — AZ-a  │  ← ECS tasks
│  Private Subnet 2 (10.0.4.0/24) — AZ-b  │  ← ECS tasks
│                                           │
│  VPC Endpoints (private subnets):         │
│    ecr.api  · ecr.dkr  · S3  · logs      │
└───────────────────────────────────────────┘
```

**Why VPC endpoints?** ECS tasks in private subnets use VPC endpoints to reach ECR and CloudWatch Logs without leaving the AWS network — no NAT bandwidth cost, faster, more secure.

**User request flow:**
```
User → Internet → ALB (port 80, public) → ECS task (port 8080, private)
```

---

## 10. Security Groups

### ALB Security Group

| Direction | Port | Source | Reason |
|---|---|---|---|
| Inbound | 80 | 0.0.0.0/0 | Public HTTP traffic |
| Inbound | 9090 | VPC CIDR only | CodeDeploy green validation (not public) |
| Outbound | 8080 | VPC CIDR | Forward to ECS tasks |

### ECS Security Group

| Direction | Port | Source | Reason |
|---|---|---|---|
| Inbound | 8080 | ALB SG | Accept traffic from ALB only |

### VPC Endpoint Security Group

| Direction | Port | Source | Reason |
|---|---|---|---|
| Inbound | 443 | ECS SG | Allow ECS tasks to reach ECR/CloudWatch |

---

## 11. The Full Deployment Pipeline

```
Developer pushes to dev / test / main
        ↓
[GitHub Actions — deploy.yml]
  1. Build JAR with Maven
  2. Authenticate to AWS via OIDC (15 min temp token)
  3. Log in to ECR
  4. Upload deploy-artifacts.zip to S3
     (taskdef.json + appspec.yaml)
  5. docker build + push {env}-{sha} and {env}-latest
        ↓
[EventBridge]
  Detects PUSH SUCCESS on tag {env}-latest
        ↓
[CodePipeline]
  Source stage (parallel):
    a) ECR source    → imageDetail.json (new image URI)
    b) S3 source     → deploy-artifacts.zip
        ↓
  Deploy stage:
    CodeDeployToECS — substitutes <IMAGE1_NAME>
    registers new task definition revision
        ↓
[CodeDeploy — blue/green]
  See section 12
        ↓
[ECS Fargate]
  New containers serving live traffic
```

---

## 12. Blue/Green Deployment Explained

### ALB listener setup

```
ALB
 ├── Port 80   (Production Listener) → Blue TG  (current live)
 └── Port 9090 (Test Listener)       → Green TG (new version)
```

### CodeDeploy sequence

```
1. Registers new ECS task definition revision
2. Starts new containers (green slot) in private subnets
3. Registers them with Green Target Group (port 9090)
4. Health checks pass on GET /health
5. Validation window — test new version on port 9090
6. Production listener (port 80) swapped to Green TG
   → Users now hit the new containers
7. Wait BlueTerminationWaitMinutes (5 min dev/test, 10 min prod)
8. Old (blue) containers terminated
```

**Rollback:** if health checks fail at step 4, CodeDeploy automatically keeps the blue containers running and terminates the green ones. Zero downtime.

---

## 13. CloudFormation GitSync

GitSync watches your Git repository and automatically updates CloudFormation stacks when you push changes.

### Deployment file format

```yaml
# deployments/dev/01-network.yaml
template-file-path: templates/network.yaml
stack-name: dev-bem13-network
region: us-east-1
capabilities:
  - CAPABILITY_NAMED_IAM
parameters:
  Environment: dev
  ProjectName: bem13
tags:
  Environment: dev
  ManagedBy: CloudFormation-GitSync
```

Push a change to `templates/network.yaml` → GitSync updates `dev-bem13-network` automatically.

---

## 14. Environment Differences

| Parameter | dev | test | prod |
|---|---|---|---|
| DesiredCount | 1 | 1 | 2 |
| MinCapacity | 1 | 1 | 2 |
| TaskCpu | 512 | 512 | 1024 |
| TaskMemory | 1024 | 1024 | 2048 |
| BlueTerminationWait | 5 min | 5 min | 10 min |
| Image tag prefix | `dev-*` | `test-*` | `prod-*` |
| SPRING_PROFILES_ACTIVE | dev | test | prod |

---

## 15. One-Time AWS Setup Checklist

### 1. Create GitHub OIDC Identity Provider in IAM

```
Provider type: OpenID Connect
Provider URL:  https://token.actions.githubusercontent.com
Audience:      sts.amazonaws.com
```

### 2. Update `templates/iam.yaml` with your GitHub repo

Replace the placeholder on line 193:

```yaml
# Before:
token.actions.githubusercontent.com:sub: "repo:YOUR_GITHUB_ORG/YOUR_APP_REPO:ref:refs/heads/*"

# After:
token.actions.githubusercontent.com:sub: "repo:nlenjibi/ecs-cicd-app:ref:refs/heads/*"
```

### 3. Deploy all 9 stacks in order for each environment

```bash
aws cloudformation deploy \
  --template-file templates/network.yaml \
  --stack-name dev-bem13-network \
  --parameter-overrides Environment=dev ProjectName=bem13 \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
# Repeat for stacks 2–9
```

### 4. Store the GitHub Actions role ARN as a secret

```bash
aws cloudformation describe-stacks \
  --stack-name prod-bem13-iam \
  --query "Stacks[0].Outputs[?OutputKey=='GitHubActionsRoleArn'].OutputValue" \
  --output text
```

In GitHub → Settings → Secrets → Actions:
```
Name:  AWS_ROLE_ARN
Value: arn:aws:iam::<AccountId>:role/prod-bem13-github-actions-role
```

### 5. Push a bootstrap image to ECR

```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin <AccountId>.dkr.ecr.us-east-1.amazonaws.com

mvn clean package -DskipTests -B
docker build -t <AccountId>.dkr.ecr.us-east-1.amazonaws.com/bem13-java-app:prod-latest .
docker push <AccountId>.dkr.ecr.us-east-1.amazonaws.com/bem13-java-app:prod-latest
```

After this, every `git push` triggers the full automated pipeline.

---

## 16. Commit and PR Standards

### Commit format

```
<type>(PROX-<id>): <short summary under 50 chars>

<body: what changed and why, wrapped at 72 chars>

Fixes PROX-<id>
```

**Types:** `feat` · `fix` · `refactor` · `style` · `chore` · `docs` · `test`

### PR description format

```markdown
## What
Brief description of the change.

## Why
The motivation — what problem does this solve?

## How
Key technical decisions.

## Testing Instructions
- [ ] Step 1
- [ ] Step 2

## Related Issue
Fixes PROX-<id>
```

---

## 17. Troubleshooting

### App won't start — port already in use

```bash
# Windows PowerShell — find and kill process on 8080
netstat -ano | findstr :8080
Stop-Process -Id <PID> -Force
```

### GitHub Actions OIDC error: `Not authorized to perform sts:AssumeRoleWithWebIdentity`

1. Confirm the OIDC provider exists in IAM (`token.actions.githubusercontent.com`)
2. Confirm the `sub` condition in `iam.yaml` matches `nlenjibi/ecs-cicd-app` exactly
3. Redeploy the IAM stack after fixing

### CodePipeline: S3 object not found

The deploy-artifacts.zip was not uploaded before the ECR push. Check the GitHub Actions log — the S3 upload step must succeed before `docker push`.

### ECS tasks failing health checks

- Health check: `GET /health` → plain text `OK`
- `HealthCheckGracePeriodSeconds: 60` gives Spring Boot 60s to start
- Check CloudWatch Logs at `/ecs/{env}-bem13` for startup errors

### ECR image pull fails in ECS (private subnet)

Stack 5 (`endpoints.yaml`) must be deployed before ECS tasks can pull images. Without VPC endpoints, image pulls time out in private subnets.
