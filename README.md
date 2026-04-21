# AWS Cloud Automation & CI/CD Pipeline with Python

![Python](https://img.shields.io/badge/Python-3.x-blue?logo=python)
![AWS](https://img.shields.io/badge/AWS-Cloud-orange?logo=amazonaws)
![Jenkins](https://img.shields.io/badge/Jenkins-CI%2FCD-red?logo=jenkins)
![Docker](https://img.shields.io/badge/Docker-Containerization-blue?logo=docker)
![boto3](https://img.shields.io/badge/boto3-AWS%20SDK-yellow)

---

## Project Overview

This project demonstrates end-to-end **cloud infrastructure automation** and **CI/CD pipeline implementation** using Python and AWS. It showcases the ability to programmatically interact with AWS services, automate infrastructure provisioning, and orchestrate containerized application deployments — skills that are central to modern DevOps and Cloud Engineering roles.

The project is split into two areas:

- **AWS Resource Automation** — Python scripts that interact directly with AWS services (EC2, ECR, IAM, VPC) using the `boto3` SDK, enabling infrastructure management without manual console interaction.
- **CI/CD Deployment Pipeline** — A Jenkins pipeline that automates the full lifecycle of deploying a Dockerized Java application from Amazon ECR to an EC2 instance, including deployment validation.

---

## Business Problem Solved

Manual cloud deployments are error-prone, time-consuming, and difficult to audit. This project addresses that by:

- Eliminating manual SSH and Docker commands during deployments
- Providing a controlled, version-selectable deployment process via Jenkins
- Automatically validating that deployments succeed before marking a build as complete
- Enabling infrastructure to be provisioned and monitored through code rather than the AWS console
- Centralizing secrets management through Jenkins credentials — no hardcoded passwords or keys in source code

---

## Architecture

> The full interactive version is available in [architecture-diagram.html](./architecture-diagram.html) — open it in any browser for the best viewing experience.

```
  Developer
     │
     │  git push
     ▼
┌─────────────┐
│   GitHub    │  python-aws-automation repo
│  Repository │
└──────┬──────┘
       │  manual trigger
       ▼
┌─────────────────────────────────────────────────┐
│              Jenkins Server (Docker Container)   │
│                                                 │
│  Stage 1: Select Image Version                  │
│    └── python3 get-images.py                    │
│         └── queries ECR for available tags      │
│         └── prompts operator to select version  │
│                                                 │
│  Stage 2: Deploy Image                          │
│    └── python3 deploy.py                        │
│         └── SSH into EC2 via paramiko           │
│         └── authenticates with ECR registry     │
│         └── pulls and runs selected image       │
│                                                 │
│  Stage 3: Validate Deployment                   │
│    └── python3 validate.py                      │
│         └── HTTP health check on port 8080      │
│         └── confirms 200 OK or reports failure  │
└────────────────────┬────────────────────────────┘
                     │  SSH (port 22) + boto3
                     ▼
       ┌──────────────────────────────┐
       │          AWS Cloud           │
       │       (ca-central-1)         │
       │                              │
       │  ┌───────────────────────┐   │
       │  │      Amazon ECR       │   │
       │  │   Repository:         │   │
       │  │   java-app            │   │
       │  │   (versioned images)  │   │
       │  └──────────┬────────────┘   │
       │             │ docker pull    │
       │             ▼               │
       │  ┌───────────────────────┐   │
       │  │    EC2 Instance       │   │
       │  │    my-server          │   │
       │  │    t2.small           │   │
       │  │    Amazon Linux 2023  │   │
       │  │                       │   │
       │  │   ┌───────────────┐   │   │
       │  │   │ Docker Engine │   │   │
       │  │   │  java-app     │   │   │
       │  │   │  port: 8080   │   │   │
       │  │   └───────────────┘   │   │
       │  └───────────────────────┘   │
       │                              │
       │  ┌───────────────────────┐   │
       │  │   Security Group      │   │
       │  │   Port 22  → Jenkins  │   │
       │  │   Port 8080 → Public  │   │
       │  └───────────────────────┘   │
       └──────────────────────────────┘
```

---

## Tech Stack

| Technology | Role |
|------------|------|
| **Python 3** | Core automation scripting language |
| **boto3** | AWS SDK — interacts with EC2, ECR, IAM, VPC |
| **paramiko** | SSH client — remote command execution on EC2 |
| **requests** | HTTP client — deployment health validation |
| **schedule** | Task scheduler — periodic application monitoring |
| **Jenkins** | CI/CD orchestration server |
| **Docker** | Application containerization |
| **Amazon EC2** | Cloud compute — application hosting |
| **Amazon ECR** | Private Docker image registry |
| **Amazon VPC** | Network isolation and subnet management |
| **AWS IAM** | Identity and access management |
| **GitHub** | Source control and pipeline trigger |

---

## Project Structure

```
python-aws-automation/
├── Jenkinsfile       # Declarative Jenkins CI/CD pipeline
├── deploy.py         # SSH into EC2 and deploy Docker image from ECR
├── validate.py       # HTTP health check to validate successful deployment
├── get-images.py     # Fetch available Docker image tags from ECR
├── ec2-nginx.py      # EC2 provisioning, Docker setup, and application monitoring
├── ecr.py            # List and inspect ECR repositories and images
├── iam.py            # Audit IAM users and identify most recently active user
├── subnets.py        # Discover default VPC subnets across availability zones
└── .gitignore
```

---

## CI/CD Pipeline — Deep Dive

The Jenkins pipeline (`Jenkinsfile`) implements a **human-in-the-loop deployment process**, giving operators control over which version gets deployed while automating all the heavy lifting.

### Stage 1 — Select Image Version
- Executes `get-images.py` which queries the ECR repository for all available Docker image tags
- Presents a dropdown in the Jenkins UI for the operator to select the exact version to deploy
- Constructs the full image URI (`registry/repo:tag`) and passes it to the next stage

### Stage 2 — Deploy Image
- Executes `deploy.py` which uses `paramiko` to establish an SSH connection to the EC2 instance
- Authenticates Docker with the ECR registry using temporary credentials
- Runs the selected Docker image as a container, mapping the configured host and container ports

### Stage 3 — Validate Deployment
- Executes `validate.py` which waits for the application to initialize, then makes an HTTP request
- A `200 OK` response confirms the deployment succeeded
- Any failure is reported, and the build is marked as failed — enabling fast feedback

---

## AWS Automation Scripts — Deep Dive

### `ec2-nginx.py` — EC2 Provisioning & Self-Healing Monitor
Demonstrates full infrastructure automation:
- Checks if an EC2 instance already exists by tag name before creating a new one (idempotent)
- Waits for the instance to pass both instance status and system status checks before proceeding
- Connects via SSH and installs Docker, starts the Docker service, and runs an nginx container
- Programmatically opens port 8080 on the default security group if not already open
- Starts a **scheduled monitor** that checks application health every 10 seconds and automatically restarts the container after 5 consecutive failures

### `ecr.py` — Container Image Management
- Lists all ECR repositories in the account
- Retrieves all images from a target repository and sorts them by push date (newest first)
- Useful for auditing image versions and release history

### `iam.py` — IAM User Auditing
- Lists all IAM users in the AWS account
- Tracks last password usage for each user
- Identifies the most recently active user — useful for security audits and access reviews

### `subnets.py` — VPC Subnet Discovery
- Queries AWS for all subnets in the configured region
- Filters and returns only the default subnets per availability zone
- Useful for dynamic infrastructure provisioning where subnet IDs are needed at runtime

---

## Key DevOps Practices Demonstrated

- **Infrastructure as Code** — AWS resources managed through Python scripts, not manual console clicks
- **CI/CD Automation** — Full deployment pipeline from image selection to health validation
- **Secrets Management** — All credentials stored in Jenkins credential store; none hardcoded in source code
- **Idempotency** — Scripts check existing state before creating resources to avoid duplication
- **Self-Healing Infrastructure** — Automated application monitoring with auto-restart capability
- **Separation of Concerns** — Each script has a single, well-defined responsibility
- **Deployment Validation** — Every deployment is automatically verified before being marked successful

---

## Prerequisites

### Local Machine
- Python 3.x
- AWS CLI configured (`aws configure`)
- Required packages:
  ```bash
  pip install boto3 paramiko requests schedule
  ```

### Jenkins Server
- Jenkins (running as Docker container or standalone)
- Python 3 with packages installed:
  ```bash
  pip3 install boto3 paramiko requests --break-system-packages
  ```
- Jenkins credentials configured:

| Credential ID                   | Type                          | Description                     |
|---------------------------------|-------------------------------|---------------------------------|
| `jenkins_aws_access_key_id`     | Secret Text                   | AWS Access Key ID               |
| `jenkins_aws_secret_access_key` | Secret Text                   | AWS Secret Access Key           |
| `ecr-repo-pwd`                  | Secret Text                   | ECR registry password           |
| `ssh-creds-boto3-server`        | SSH Username with Private Key | EC2 SSH key pair (.pem file)    |

### AWS Resources
- EC2 instance (Amazon Linux 2023, `t2.small`) with Docker installed
- ECR repository with at least one pushed Docker image
- Security group with:
  - Port `22` open to Jenkins server IP
  - Port `8080` open to `0.0.0.0/0`

---

## Environment Variables

For running scripts locally outside of Jenkins:

| Variable         | Description                              |
|------------------|------------------------------------------|
| `EC2_SERVER`     | Public IP address of the EC2 instance    |
| `EC2_USER`       | SSH username (`ec2-user` for Amazon Linux) |
| `SSH_KEY_FILE`   | Absolute path to the `.pem` private key  |
| `ECR_REGISTRY`   | Full ECR registry URL                    |
| `DOCKER_USER`    | Docker login username (`AWS`)            |
| `DOCKER_PWD`     | ECR login password                       |
| `DOCKER_IMAGE`   | Full Docker image URI with tag           |
| `CONTAINER_PORT` | Port the application listens on inside the container |
| `HOST_PORT`      | Port exposed on the EC2 host             |
| `ECR_REPO_NAME`  | Name of the ECR repository               |

Set them before running any script:
```bash
export EC2_SERVER='your-ec2-ip'
export HOST_PORT='8080'
python3 validate.py
```

---

## AWS Region

All resources are provisioned in **ca-central-1 (Canada Central)**.

---

## Author

**Ndubuisi Paul Okoma**
DevOps / Cloud Engineer
[GitHub](https://github.com/OkomaNdu)
