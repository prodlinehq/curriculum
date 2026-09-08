# Episode 21 — Terraform Remote State and Locking

| | |
| :--- | :--- |
| **YouTube title** | Terraform Remote State and Locking |
| **Film order** | 21 of 33 · Phase 3 · Week 21 |
| **Duration** | 10 minutes |
| **Lab repo** | `supercheck-io/supercheck` |
| **Supercheck files** | `supercheck-ee/deploy OpenTofu + Infisical` |
| **Next** | [22-tofu-modules.md](22-tofu-modules.md) |

## Overview

Two applies must not corrupt state. Supercheck prod is Hetzner K3s; state still needs lock + encryption.

Film only Supercheck names: `supercheck-app`, `supercheck-worker-eu`, `supercheck-execution`, Traefik, BullMQ, PlanetScale/Postgres, Redis Sentinel. No toy `demo-api`.


## Episode 21: Terraform Remote State and Locking

### 1. The Production Problem
*"Two engineers applied Terraform concurrently from their laptops. One was adding a subnet while the other was attaching an Internet Gateway. The state file suffered a race collision, leaving half the VPC resources orphaned in AWS and generating unrecoverable state corruption."*

### 2. Deep Technical Breakdown
1. **The State File is Plaintext:** Regardless of whether your code uses environment variables or secrets managers, the compiled `.tfstate` contains **raw passwords, TLS private keys, and API tokens in unencrypted JSON**. State files must be stored in encrypted, IAM-restricted object storage (S3 + SSE-KMS) and never committed to Git.
2. **DynamoDB Distributed Locking Mechanics:** To prevent two engineers or CI pipelines from modifying infrastructure at the same time, Terraform acquires a lock before every command. DynamoDB uses a table with a string partition key named `LockID`. Terraform performs a conditional write:
   `attribute_not_exists(LockID)`
   If another process holds the lock, DynamoDB rejects the write, and Terraform halts with a lock acquisition failure.
3. **S3 Object Versioning as Disaster Recovery:** If a state write is interrupted by a network blip or corrupted by a faulty tool, S3 versioning allows restoring the exact previous state JSON in seconds.

### 3. Concurrency Race Condition vs Distributed Lock Sequence

```mermaid
sequenceDiagram
    autonumber
    actor EngineerA as Engineer A (Feature: VPC Peering)
    actor EngineerB as Engineer B (Feature: EKS Addon)
    participant DDB as DynamoDB Lock Table
    participant S3 as S3 State Storage

    EngineerA->>DDB: PutItem(LockID="prod/terraform.tfstate")
    DDB-->>EngineerA: 200 OK (Lock Acquired)
    EngineerA->>S3: Download latest state.json

    EngineerB->>DDB: PutItem(LockID="prod/terraform.tfstate")
    Note over DDB: ConditionalCheckFailedException:<br/>Lock already held by Engineer A!
    DDB-->>EngineerB: Error: State locked by Engineer A (Lock Info: Created 10:02 UTC)
    Note over EngineerB: Engineer B execution halts cleanly; zero state corruption!

    EngineerA->>S3: Upload updated state.json
    EngineerA->>DDB: DeleteItem(LockID="prod/terraform.tfstate")
    DDB-->>EngineerA: Lock Released
```

### 4. Production Remote Backend Configuration (`backend.tf`)
```hcl
terraform {
  required_version = ">= 1.8.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.50"
    }
  }

  backend "s3" {
    bucket         = "prodline-terraform-state-eu-west-1"
    key            = "platform/prod/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "prodline-terraform-locks"
    encrypt        = true
  }
}
```

---


## Remote Backend Comparison Matrix

| Backend Type | Locking Supported? | Encryption at Rest | Access Audit Logging | Cost | Best For |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **S3 + DynamoDB** | **Yes (DynamoDB)** | **Yes (SSE-KMS)** | **CloudTrail + S3 Access Logs** | **<$1/month** | **Standard AWS production foundation** |
| **GCS (Google Cloud)** | Yes (Native) | Yes (Cloud KMS) | Cloud Audit Logs | <$1/month | GCP production workloads |
| **Terraform Cloud** | Yes (Native) | Yes | Full platform audit logs | Expensive ($0.00014/hr/res) | Teams requiring commercial UI/governance |
| **Spacelift / Scalr** | Yes (Native) | Yes | Enterprise audit logs | Tiered pricing | Complex multi-cloud enterprise policy orchestration |

---

## State Disaster Recovery & Forensics Matrix

| Disaster Scenario | Underlying Cause | Recovery Runbook |
| :--- | :--- | :--- |
| **Stale Lock Deadlock** | CI runner crashed or was cancelled mid-apply | Run `terraform force-unlock <LOCK_ID>`. Verify no other pipeline is running first! |
| **Out-of-Band State Drift** | Engineer modified AWS console resource manually | Run `terraform plan -detailed-exitcode` (Exit 2 indicates drift). Reconcile by applying or importing. |
| **State File Corruption** | Interrupted write or network timeout | S3 Bucket Versioning enables instant recovery: restore the previous S3 object version. |
| **Accidental Deletion** | Resource removed from HCL without planning | Leverage `lifecycle { prevent_destroy = true }` on databases, VPCs, and storage buckets. |

---

## European SRE OpenTofu & IaC Interview Signals

| Interview Question | Junior Candidate Answer | Senior SRE Winning Answer |
| :--- | :--- | :--- |
| **"Terraform vs OpenTofu: What is the current enterprise status?"** | "They are the exact same tool." | "Post HashiCorp's BSL licensing change, the Linux Foundation created OpenTofu. European enterprises (especially banking, fintech, and telecommunications) have strict open-source governance and frequently mandate OpenTofu to eliminate commercial licensing risk." |
| **"Where do secrets live in Terraform state files?"** | "They are encrypted by AWS." | "Terraform state stores resources (including DB passwords and TLS private keys) in **plain text JSON** inside `.tfstate`. It must be secured with KMS encryption, strict IAM bucket policies, and never committed to version control." |
| **"How do you prevent Terraform from accidentally destroying critical databases?"** | "Be careful before typing 'yes'." | "Enforce `lifecycle { prevent_destroy = true }` in HCL, configure Sentinel / Open Policy Agent (OPA) gates in CI to reject destructive plans, and maintain automated point-in-time database snapshots." |

---

