# AWS EKS Platform Engineering – Hands-on Projects

## Overview

This repository contains two practical AWS EKS projects demonstrating Kubernetes application deployment, CI/CD automation, AWS cloud integration, IAM security, and workload identity management.

The projects follow real-world Platform Engineering practices, focusing on automation, scalability, security, and operational reliability.

## Projects

1. **Project 1 – Automated Flask Application Deployment on Amazon EKS**
2. **Project 2 – Secure Amazon S3 Access from Kubernetes Pods using EKS Pod Identity**

---

# Project 1: Automated Flask Application Deployment on Amazon EKS

## Objective

Build an automated CI/CD pipeline to containerize a Flask application, push Docker images to Amazon ECR, and deploy the application to Amazon EKS using GitHub Actions.

Expose the application publicly through an AWS Application Load Balancer (ALB) using Kubernetes Ingress.

## Architecture

```text
Developer
    |
    v
GitHub Repository
    |
    v
GitHub Actions CI/CD
    |
    |-- AWS OIDC Authentication
    |
    |-- Build Docker Image
    |
    |-- Push Image
    v
Amazon ECR
    |
    v
Amazon EKS
    |
    |-- Deployment
    |-- Replica Pods
    |-- ClusterIP Service
    |-- Ingress
    |
    v
AWS Load Balancer Controller
    |
    v
Internet-facing Application Load Balancer
    |
    v
External Users
```

## Technologies Used

| Technology | Purpose |
|---|---|
| AWS EKS | Managed Kubernetes platform |
| Amazon ECR | Docker image repository |
| GitHub Actions | CI/CD automation |
| AWS IAM | Identity and access management |
| GitHub OIDC | Keyless AWS authentication |
| Docker | Application containerization |
| Kubernetes | Container orchestration |
| Helm | AWS Load Balancer Controller installation |
| AWS ALB | External application access |
| kubectl | Kubernetes administration |

## Implementation Details

### 1. Containerization

- Developed a Flask application.
- Created a Dockerfile.
- Built a container image.
- Tested the application locally.

### 2. Amazon ECR Integration

- Created an ECR repository.
- Authenticated Docker with Amazon ECR.
- Pushed application images to ECR.
- Used Git commit SHA tags for image versioning.

### 3. GitHub Actions CI/CD

Created a GitHub Actions workflow to automate:

- Source code checkout.
- AWS authentication using GitHub OIDC.
- Docker image build.
- Image tagging.
- Image push to ECR.
- Kubernetes deployment updates.

### 4. AWS OIDC Authentication

Configured GitHub Actions to assume an AWS IAM role using OpenID Connect.

Benefits:
- No long-lived AWS access keys in GitHub.
- Temporary AWS credentials.
- IAM-controlled permissions.
- Automated authentication from GitHub Actions.

### 5. Kubernetes Deployment

Created Kubernetes manifests for:

- Deployment
- Service
- Ingress

Deployed two Flask application replicas.

Configured the Kubernetes Service as `ClusterIP` to provide internal application connectivity.

### 6. AWS Load Balancer Controller

Installed AWS Load Balancer Controller using Helm.

Configured:
- IAM permissions.
- Kubernetes ServiceAccount.
- IAM Roles for Service Accounts (IRSA).
- ALB Ingress configuration.
- Internet-facing Application Load Balancer.

### 7. Application Exposure

Configured ALB Ingress with:

- Internet-facing scheme.
- IP target type.
- HTTP listener on port 80.
- Kubernetes Service backend.

External users can access the Flask application through the ALB DNS hostname.

## Troubleshooting Experience

Resolved the following real-world issues:

- GitHub Actions OIDC authentication failures.
- EKS access and Kubernetes authorization issues.
- Kubernetes pods failing to schedule.
- AWS Load Balancer Controller installation and webhook issues.
- ALB DNS resolution troubleshooting.
- Kubernetes Service selector mismatch.
- HTTP 503 Service Temporarily Unavailable.
- Application backend endpoint troubleshooting.

## Key Learnings

- End-to-end EKS deployment architecture.
- CI/CD automation using GitHub Actions.
- Secure AWS authentication using OIDC.
- Kubernetes networking and service discovery.
- Ingress and ALB integration.
- IAM permissions and workload access.
- Production-style troubleshooting.

---

# Project 2: Secure Amazon S3 Access from Kubernetes Pods using EKS Pod Identity

## Objective

Implement secure access to Amazon S3 from Kubernetes workloads without storing static AWS access keys inside containers.

Use Amazon EKS Pod Identity to provide temporary AWS credentials to Kubernetes pods through IAM roles.

Demonstrate how an application running inside a Kubernetes pod can create files and upload them to an S3 bucket.

## Architecture

```text
Kubernetes Pod
      |
      | Creates hello.txt
      |
      v
Kubernetes ServiceAccount
      |
      v
EKS Pod Identity Association
      |
      v
EKS Pod Identity Agent
      |
      v
AWS IAM Role
      |
      | Temporary Credentials
      |
      v
Amazon S3
      |
      v
Uploaded hello.txt
```

## Technologies Used

| Technology | Purpose |
|---|---|
| AWS EKS | Kubernetes platform |
| EKS Pod Identity | Workload authentication |
| IAM Role | AWS permissions for workloads |
| IAM Policy | Restrict S3 access |
| Kubernetes ServiceAccount | Pod identity |
| Pod Identity Agent | Credential delivery |
| Amazon S3 | Object storage |
| AWS CLI | S3 operations and identity verification |

## Implementation Details

### 1. Amazon S3 Bucket

Created a dedicated S3 bucket for the EKS workload.

Purpose:
- Store files generated by Kubernetes pods.
- Demonstrate application-to-S3 integration.
- Validate AWS permissions from inside a container.

### 2. IAM Policy

Created a custom IAM policy with restricted permissions.

Permissions:

| IAM Action | Purpose |
|---|---|
| `s3:ListBucket` | List objects in the designated bucket |
| `s3:PutObject` | Upload files |
| `s3:GetObject` | Download files |

The policy is scoped to the dedicated lab bucket instead of granting access to all S3 resources.

### 3. IAM Role

Created a dedicated IAM role for EKS workloads.

Configured the trust relationship with:

`pods.eks.amazonaws.com`

Allowed actions:
- `sts:AssumeRole`
- `sts:TagSession`

Attached the custom S3 access policy to the IAM role.

### 4. Kubernetes ServiceAccount

Created a dedicated Kubernetes ServiceAccount:

`s3-access-sa`

This identity is assigned to the test pod.

### 5. EKS Pod Identity Association

Associated the Kubernetes ServiceAccount with the IAM role using the AWS CLI.

The association connects:

- EKS cluster
- Kubernetes namespace
- Kubernetes ServiceAccount
- AWS IAM role

### 6. Test Pod Deployment

Created a Kubernetes pod using the AWS CLI container image.

Configured the pod to use:

`s3-access-sa`

Verified that EKS Pod Identity injected the required credentials endpoint and token into the pod.

### 7. AWS Identity Verification

Executed the following command inside the pod:

```bash
kubectl exec s3-test-pod -- aws sts get-caller-identity
```

Verified that the pod assumed the dedicated IAM role:

`EKS-Pod-S3-Access-Role`

This confirmed that the pod received temporary AWS credentials without storing static access keys.

### 8. File Creation and S3 Upload

Created a test file inside the container:

```bash
kubectl exec s3-test-pod -- sh -c \
  'echo "Hello from EKS Pod Identity" > /tmp/hello.txt'
```

Read the file:

```bash
kubectl exec s3-test-pod -- sh -c \
  'cat /tmp/hello.txt'
```

Upload the file to S3:

```bash
kubectl exec s3-test-pod -- sh -c \
  'aws s3 cp /tmp/hello.txt \
  s3://YOUR-BUCKET-NAME/hello.txt \
  --region ap-south-1'
```

Verify the uploaded object:

```bash
aws s3 ls s3://YOUR-BUCKET-NAME/
```

## Key Learnings

- EKS Pod Identity architecture.
- Kubernetes ServiceAccount integration with AWS IAM.
- IAM trust relationships.
- Temporary AWS credentials.
- Secure pod-to-AWS authentication.
- Least-privilege IAM permissions.
- S3 object upload and download from Kubernetes workloads.

---

# Repository Structure

```text
Sample-app-Ingress-cntl/
│
├── .github/
│   └── workflows/
│       └── deployment.yml
│
├── app/
│   ├── app.py
│   └── Dockerfile
│
├── k8/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
│
├── s3-test-pod.yaml
├── s3-pod-policy.json
├── pod-identity-trust.json
├── .gitignore
└── README.md
```

*Adjust the filenames above to match the actual files in your repository.*

# Security Best Practices

- Use GitHub OIDC instead of static AWS credentials.
- Use IAM roles for Kubernetes workloads.
- Follow least-privilege IAM permissions.
- Avoid committing AWS credentials, secrets, or sensitive configuration files.
- Use dedicated ServiceAccounts for application workloads.
- Restrict S3 access to specific buckets.
- Use HTTPS and ACM certificates for production ALB endpoints.
- Enable appropriate monitoring and audit logging.

# Future Enhancements

The following improvements are planned:

1. Implement HTTPS using AWS Certificate Manager.
2. Configure Horizontal Pod Autoscaler.
3. Implement Karpenter for EKS node autoscaling.
4. Introduce Argo CD for GitOps-based deployments.
5. Implement Prometheus and Grafana monitoring.
6. Configure Kubernetes RBAC and Network Policies.
7. Add Terraform modules for infrastructure provisioning.
8. Implement automated deployment rollback.
9. Integrate application logging with CloudWatch.
10. Implement S3 lifecycle policies and encryption.

# Overall Learning Outcomes

These two projects demonstrate practical experience with:

- AWS cloud infrastructure.
- Amazon EKS and Kubernetes administration.
- Containerization and orchestration.
- CI/CD pipeline engineering.
- GitHub Actions and OIDC authentication.
- AWS IAM and workload identity.
- Kubernetes networking and Ingress.
- AWS Load Balancer Controller.
- Secure application-to-cloud integration.
- Production troubleshooting.

**Project Goal:** Build secure, automated, scalable, and reliable cloud-native infrastructure using AWS and Kubernetes.

---

*Hands-on Platform Engineering Lab | AWS | Kubernetes | EKS | DevOps | CI/CD*
