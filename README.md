<div align="center">
  
![GitHub Actions](https://img.icons8.com/color/96/github.png)
![AWS IAM](https://img.icons8.com/color/96/amazon-web-services.png)

## GitHub Actions OIDC Authentication with AWS – Secure CI/CD Without Static Credentials

**Updated: January 14, 2026**

[![Follow @nicoleepaixao](https://img.shields.io/github/followers/nicoleepaixao?label=Follow&style=social)](https://github.com/nicoleepaixao)
[![Star this repo](https://img.shields.io/github/stars/nicoleepaixao/github-actions-aws-oidc?style=social)](https://github.com/nicoleepaixao/github-actions-aws-oidc)
[![Medium Article](https://img.shields.io/badge/Medium-12100E?style=for-the-badge&logo=medium&logoColor=white)](https://nicoleepaixao.medium.com/)

<p align="center">
  <a href="README-PT.md">🇧🇷</a>
  <a href="README.md">🇺🇸</a>
</p>


</div>

---

<p align="center">
  <img src="img/aws-oidc.png" alt="OIDC Architecture" width="1800">
</p>

## **The Problem**

Traditional AWS authentication in CI/CD pipelines relies on **long-lived credentials** stored as GitHub Secrets. Every deployment workflow uses the same static **Access Key** and **Secret Access Key**, which introduces significant security risks:

- **Credentials never expire** unless manually rotated
- **High risk of leakage** through logs, errors, or compromised repositories
- **No auditability** of which workflow or job used the credentials
- **Shared secrets** across multiple workflows and environments

This project solves that problem by implementing **OIDC-based authentication**, eliminating static credentials entirely. GitHub Actions requests temporary tokens from AWS STS, which automatically expire after the job completes, ensuring **zero long-lived secrets**.

---

## **How It Works**

Using **OpenID Connect (OIDC)**, this solution enables:

- **Temporary credentials** issued by AWS STS on every workflow run
- **No Access Keys stored** in GitHub Secrets
- **Automatic expiration** of credentials (15 minutes to 12 hours)
- **Fine-grained access control** via IAM Role trust policies restricted to specific repositories and branches

The GitHub Actions workflow requests an OIDC token, AWS validates it against the configured trust policy, and returns temporary credentials that work only for that specific job execution.

---

## **Prerequisites**

| **Requirement** | **Details** |
|-----------------|-------------|
| **AWS Account** | With IAM permissions to create OIDC providers and roles |
| **GitHub Repository** | Public or private repo with Actions enabled |
| **AWS CLI** | Configured with appropriate credentials |

---

## **Step-by-Step Implementation**

**Step 1: Create/Validate OIDC Identity Provider**

The GitHub OIDC provider configuration:
- **URL:** `https://token.actions.githubusercontent.com`
- **Audience:** `sts.amazonaws.com`

Check if provider exists:

```bash
aws iam list-open-id-connect-providers
```

If you see:
```text
arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com
```

✅ Provider already exists, skip to step 2.

Create provider (if needed):

Recommended: Use AWS Console (IAM → Identity providers → Add provider)
- Provider URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

Via CLI (requires thumbprint):
```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list <THUMBPRINT>
```

---

**Step 2: Create IAM Role for GitHub Actions**

Create `github-oidc-trust.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<OWNER>/<REPO>:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

**Replace:**
- `<ACCOUNT_ID>` - Your AWS account ID
- `<OWNER>` - GitHub organization or username (e.g., `nicoleepaixao`)
- `<REPO>` - Repository name (e.g., `pipeline-nginx`)

Create IAM Role:

```bash
aws iam create-role \
  --role-name github-oidc-deploy \
  --assume-role-policy-document file://github-oidc-trust.json
```

Save the ARN returned:
```
arn:aws:iam::<ACCOUNT_ID>:role/github-oidc-deploy
```

---

**Step 3: Attach Permissions to Role**

The role needs permissions for what your pipeline does:

```bash
# ECR permissions (push images)
aws iam attach-role-policy \
  --role-name github-oidc-deploy \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser

# ECS permissions (deploy tasks)
aws iam attach-role-policy \
  --role-name github-oidc-deploy \
  --policy-arn arn:aws:iam::aws:policy/AmazonECSFullAccess
```

⚠️ **For production:** Replace with a custom least-privilege policy.

---

## **GitHub Actions Configuration**

**Step 4: Configure Repository Variables**

In your GitHub repository:

1. Go to **Settings** → **Secrets and variables** → **Actions** → **Variables**
2. Add variables:

| **Variable Name** | **Value** |
|------------------|-----------|
| `AWS_ROLE_ARN_HOMOL` | `arn:aws:iam::<ACCOUNT_ID>:role/github-oidc-deploy-homol` |
| `AWS_ROLE_ARN_PROD` | `arn:aws:iam::<ACCOUNT_ID>:role/github-oidc-deploy-prod` |
| `AWS_REGION` | `us-east-1` |

**Note:** ARNs are not sensitive - use Variables instead of Secrets.

---

**Step 5: Workflow Configuration**

Complete workflow example:

```yaml
name: Deploy to AWS ECS

on:
  push:
    branches:
      - main

permissions:
  id-token: write   # Required for OIDC token
  contents: read    # Required for checkout

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN_HOMOL }}
          aws-region: ${{ vars.AWS_REGION }}
      
      - name: Verify AWS identity
        run: aws sts get-caller-identity
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Build and push Docker image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          ECR_REPOSITORY: my-app
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
      
      - name: Deploy to ECS
        env:
          CLUSTER_NAME: my-cluster
          SERVICE_NAME: my-service
        run: |
          aws ecs update-service \
            --cluster $CLUSTER_NAME \
            --service $SERVICE_NAME \
            --force-new-deployment
```

---

## **Multi-Environment Setup**

**Separate Roles for Homol and Prod**

Homol Trust Policy (accepts `main` branch):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<OWNER>/<REPO>:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

Prod Trust Policy (accepts `release` branch only):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:<OWNER>/<REPO>:ref:refs/heads/release"
        }
      }
    }
  ]
}
```

---

## **Validation and Testing**

**Verify OIDC Configuration:**

```bash
# Check if OIDC provider exists
aws iam list-open-id-connect-providers

# Verify role trust policy
aws iam get-role --role-name github-oidc-deploy
```

**Workflow Validation Step:**

Add this to your workflow to confirm identity:

```yaml
- name: Verify AWS Identity
  run: |
    aws sts get-caller-identity
    echo "Assumed role successfully!"
```

Expected output:
```json
{
  "UserId": "AROAXXXXXXXXX:github-actions-session",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/github-oidc-deploy/github-actions-session"
}
```

---

## **Troubleshooting**

| **Error** | **Cause** | **Solution** |
|-----------|-----------|-------------|
| `Could not assume role with OIDC` | Missing `id-token: write` permission | Add to workflow `permissions` section |
| `AccessDenied` during deployment | Role lacks necessary permissions | Attach appropriate IAM policies |
| `Invalid identity token` | Trust policy mismatch | Verify `sub` claim matches repo/branch exactly |
| `Provider not found` | OIDC provider doesn't exist | Create provider in IAM console |

---

## **Security Best Practices**

| **Practice** | **Implementation** |
|--------------|-------------------|
| **Least Privilege** | Grant only required permissions to role |
| **Branch Restrictions** | Limit trust policy to specific branches |
| **Environment Protection** | Use GitHub Environments with approvals for prod |
| **Audit Logs** | Enable CloudTrail for all AssumeRole calls |
| **Separate Roles** | One role per environment (homol/prod) |

**Custom Least-Privilege Policy Example:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecs:UpdateService",
        "ecs:DescribeServices",
        "ecs:RegisterTaskDefinition",
        "ecs:DescribeTaskDefinition"
      ],
      "Resource": [
        "arn:aws:ecs:us-east-1:<ACCOUNT_ID>:service/my-cluster/*",
        "arn:aws:ecs:us-east-1:<ACCOUNT_ID>:task-definition/my-app:*"
      ]
    }
  ]
}
```

---

## **Key Benefits**

| **Benefit** | **Description** |
|-------------|-----------------|
| **No Long-Lived Credentials** | No Access Keys or Secrets stored in GitHub |
| **Automatic Rotation** | Tokens expire automatically (1 hour) |
| **Least Privilege** | Restrict access by repository and branch |
| **Audit Trail** | All actions logged in CloudTrail with context |
| **Secure by Default** | Industry best practice recommended by AWS and GitHub |

---

## **Additional Resources**

- [GitHub OIDC Documentation](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-amazon-web-services) - Official guide
- [AWS IAM OIDC Identity Providers](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc.html) - Provider setup
- [aws-actions/configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials) - GitHub Action
- [AWS Security Token Service (STS)](https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html) - STS reference

---

## **Connect & Follow**

Stay updated with DevOps automation and best practices:

<div align="center">

[![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/nicoleepaixao)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?logo=linkedin&logoColor=white&style=for-the-badge)](https://www.linkedin.com/in/nicolepaixao/)
[![Medium](https://img.shields.io/badge/Medium-12100E?style=for-the-badge&logo=medium&logoColor=white)](https://medium.com/@nicoleepaixao)

</div>

---

## **Disclaimer**

This configuration provides temporary credentials with automatic expiration. Always follow the principle of least privilege and restrict role assumptions to specific repositories and branches. Test in non-production environments before applying to production workloads. Consult official AWS and GitHub documentation for the latest security recommendations.

---

<div align="center">

**Happy automating your deployments!**

*Document Created: January 14, 2026*

Made with ❤️ by [Nicole Paixão](https://github.com/nicoleepaixao)

</div>
