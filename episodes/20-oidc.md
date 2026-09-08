# Episode 20 — OIDC: Stop Using Long-Lived Cloud Keys

| | |
| :--- | :--- |
| **YouTube title** | OIDC: Stop Using Long-Lived Cloud Keys |
| **Film order** | 20 of 33 · Phase 3 · Week 20 |
| **Duration** | 10 minutes |
| **Lab repo** | `supercheck-io/supercheck` |
| **Supercheck files** | `GitHub OIDC → GHCR; no AWS keys in GitHub Secrets` |
| **Next** | [21-tofu-state.md](21-tofu-state.md) |

## Overview

NIS2/DORA: short-lived tokens. Supercheck images push to ghcr.io/supercheck-io/supercheck.

Film only Supercheck names: `supercheck-app`, `supercheck-worker-eu`, `supercheck-execution`, Traefik, BullMQ, PlanetScale/Postgres, Redis Sentinel. No toy `demo-api`.


## Episode 20: OIDC: Stop Using Long-Lived Cloud Keys

### 1. The Production Problem
*"A company kept `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` stored in GitHub repository secrets. A supply-chain vulnerability in an open-source GitHub Action leaked all environment variables to a rogue external IP. By morning, attackers had provisioned hundreds of GPU instances across 5 AWS regions, costing $80,000."*

### 2. Deep Technical Breakdown
1. **Cryptographic Web Identity Federation:** GitHub acts as an OpenID Connect Identity Provider (IdP). When a job requests credentials, GitHub mints an ephemeral JSON Web Token (JWT) signed by GitHub's private RSA key.
2. **AWS STS Validation:** AWS Security Token Service (STS) receives the JWT via `AssumeRoleWithWebIdentity`. STS verifies GitHub's public certificate thumbprint, validates that the audience is `sts.amazonaws.com`, and checks the subject (`sub`) claim.
3. **Branch-Level Least Privilege:** The IAM Trust Policy enforces:
   `repo:prodlinehq/supercheck-app:ref:refs/heads/main`
   Developers on feature branches or pull requests cannot assume production deployment permissions!
4. **Zero Persistent Storage:** Credentials exist only in runner memory as temporary session tokens (`ASIA...`) that automatically expire in 3,600 seconds.

### 3. Architecture & Token Exchange Sequence

```mermaid
sequenceDiagram
    autonumber
    participant Runner as GitHub Actions Runner
    participant IdP as GitHub OIDC Token Service
    participant STS as AWS Security Token Service (STS)
    participant ECR as Amazon Elastic Container Registry

    Runner->>IdP: Request OIDC JWT (Audience: sts.amazonaws.com)
    IdP-->>Runner: Return signed JWT with claims (sub: repo:prodlinehq/supercheck-app:ref:refs/heads/main)
    Runner->>STS: AssumeRoleWithWebIdentity(RoleARN, JWT)
    Note over STS: 1. Validate GitHub public key signature<br/>2. Verify 'sub' claim matches main branch<br/>3. Verify 'aud' claim matches sts.amazonaws.com
    STS-->>Runner: Return temporary AWS credentials (Key, Secret, SessionToken, 1hr TTL)
    Runner->>ECR: Authenticate via temporary token
    Runner->>ECR: Push scanned container image
    Note over Runner: Job finishes; temporary credentials expire permanently
```

### 4. AWS IAM Trust Policy & Production Workflow

#### AWS IAM Trust Policy Configuration
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:prodlinehq/supercheck-app:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

#### Production Deployment Workflow (.github/workflows/deploy-prod.yml)
```yaml
name: Production Release & Deploy (OIDC)

on:
  push:
    branches: [main]

permissions:
  id-token: write   # Required to request the OIDC JWT token
  contents: read    # Required to checkout code

jobs:
  release-and-push:
    name: Build, Authenticate & Push to AWS ECR
    runs-on: ubuntu-latest
    steps:
    - name: Checkout Code
      uses: actions/checkout@v4

    # 1. Passwordless Authentication via OIDC
    - name: Configure AWS Credentials via OIDC
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/github-actions-supercheck-app-prod
        aws-region: eu-west-1
        audience: sts.amazonaws.com

    - name: Log in to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2

    # 2. BuildKit Container Build & Push
    - name: Build & Push Image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.login-ecr.outputs.registry }}/supercheck-app:${{ github.sha }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
```

---


## European SRE Compliance & Security Interview Signals

| Interview Question | Junior Candidate Answer | Senior SRE Winning Answer |
| :--- | :--- | :--- |
| **"How do you secure CI/CD pipelines against secret leakage?"** | Put access keys in GitHub Secrets. | **OIDC Federation:** Eliminate static credentials, mint ephemeral STS tokens valid for 1 hour, and enforce branch-level least-privilege scoping via IAM trust policy conditions. |
| **"What is NIS2 / DORA compliance in European SRE?"** | "Not familiar." | "European cybersecurity directives requiring strict third-party supply chain verification, elimination of unrotated static credentials, and cryptographic auditability (SBOM & OIDC)." |
| **"How do you coordinate CI build outputs with GitOps deployments?"** | CI executes `kubectl apply` directly. | CI builds the container, scans it with Trivy, pushes to ECR, and commits the new image tag to the GitOps configuration repo via an automated pull request. **ArgoCD manages cluster deployment.** |

---
