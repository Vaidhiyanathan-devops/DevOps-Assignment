
<img width="940" height="513" alt="image" src="https://github.com/user-attachments/assets/c75f3230-670d-44f6-82c6-fb86ba64aeea" />


<img width="1448" height="1032" alt="Infra-design" src="https://github.com/user-attachments/assets/77e40d07-5602-42ca-9c04-2978f8659258" />


https://github.com/user-attachments/assets/ab896d09-2701-4530-9a1f-755d923d99d8


AWS APPLICATION URL : https://k8s.vaidhi.sbs



---

## 📁 Repository Structure

```
.
├── frontend/                  # React frontend application
├── backend/                   # Node.js/Express backend
│   ├── GET /api/health        # → {"status":"healthy","message":"Backend is running successfully"}
│   └── GET /api/message       # → {"message":"You've successfully integrated the backend!"}
├── terraform/
│   ├── aws/                   # AWS infrastructure (IaC)
│   │   ├── environments/
│   │   │   ├── dev/
│   │   │   ├── staging/
│   │   │   └── prod/
│   │   └── modules/
│   │       ├── vpc/
│   │       ├── ec2/
│   │       ├── nginx/
│   │       └── route53/
│   └── gcp/                   # GCP infrastructure (IaC) — coming in next submission
├── .github/
│   └── workflows/             # CI/CD pipelines
└── README.md
```

---

## 🏗️ Architecture Overview

### AWS Deployment (ap-south-1 — Mumbai)

Traffic flows from client → domain → AWS → VPC in a clean public/private subnet split:

```
Client
  │
  ▼
k8s.vaidhi.sbs (Hostinger Domain)
  │
  ▼
AWS Nameserver → Route 53 (vaidhi.sbs hosted zone)
  │
  ▼
A/CNAME Record → NGINX EC2 EIP
  │
  ▼
Internet Gateway (IGW)
  │
  ├──► Public Subnet (ap-south-1a)
  │        ├── NGINX EC2 (Load Balancer + Reverse Proxy, EIP attached)
  │        └── Frontend EC2
  │
  └──► Private Subnet (ap-south-1b)
           ├── Backend EC2
           └── NAT Gateway (for egress-only internet access)
```

**Key design decisions:**
- NGINX on EC2 acts as both the ingress point and reverse proxy — no AWS ALB (cost-justified for this scale)
- Backend lives in a private subnet with no public IP — only reachable from the NGINX proxy
- NAT Gateway allows backend EC2 to pull updates/dependencies without exposing it publicly
- EIP on NGINX EC2 ensures a stable IP for the Route 53 A record


## ☁️ Infrastructure — AWS (Terraform)

### Architecture
- **Region:** `ap-south-1` (Mumbai) — chosen for proximity to Indian user base, cost efficiency, and personal AWS familiarity
- **Compute:** EC2 instances (t3.micro for dev/t3.small for prod) — Kubernetes was deliberately avoided; this workload does not justify the operational overhead
- **Networking:** Custom VPC with public + private subnets across availability zones
- **Ingress:** Self-managed NGINX (no managed ALB — cost-justified at this scale)
- **DNS:** Route 53 hosted zone with A record pointing to NGINX EIP

### Terraform Usage

```bash
cd terraform/aws/environments/prod

# Initialize (remote state on S3)
terraform init

# Preview changes
terraform plan -var-file="prod.tfvars"

# Apply
terraform apply -var-file="prod.tfvars"
```

### State Management

| Concern | Implementation |
|---|---|
| State backend | AWS S3 bucket (`pgagi-tfstate-<env>`) |
| State locking | DynamoDB table (`pgagi-tf-lock`) |
| Env isolation | Separate S3 prefix per environment (`dev/`, `staging/`, `prod/`) |
| Recovery | S3 versioning enabled — roll back to any prior state |

> ⚠️ State buckets and lock tables are bootstrapped separately via `terraform/aws/bootstrap/` — they are not self-referential.

### Environment Differences

| Feature | dev | staging | prod |
|---|---|---|---|
| Instance type | t3.micro | t3.micro | t3.small |
| NGINX rate limiting | Disabled | Enabled (relaxed) | Enabled (strict) |
| Deletion protection | Off | Off | On |
| Backups | None | None | Enabled |
| Monitoring | Basic | CloudWatch | CloudWatch + Alerts |

---

## 🔐 Security Design

- **No secrets in Git** — all credentials injected via GitHub Actions secrets or AWS SSM Parameter Store
- **Backend EC2 has no public IP** — only accessible through NGINX proxy in the public subnet
- **IAM roles** scoped to least-privilege per component (EC2 instance roles, CI/CD deploy role)
- **Security Groups** enforce allow-only rules:
  - NGINX SG: inbound 80/443 from `0.0.0.0/0`, outbound to backend SG only
  - Backend SG: inbound only from NGINX SG on app port
- **SSH access** locked to specific IP ranges (not `0.0.0.0/0`)

---

## 📈 Scalability & Availability

| Concern | Current Behavior |
|---|---|
| Frontend/Backend scale | Manual — single EC2 per service |
| NGINX scale | Single instance (SPOF at this scale — acceptable for assignment scope) |
| Traffic spikes | Handled by NGINX buffering + upstream keepalives |
| AZ failure | NAT Gateway in ap-south-1b; NGINX in ap-south-1a — partial resilience |
| Recovery | EC2 auto-recovery via CloudWatch alarm + instance recovery action |

**10× traffic scenario:** Add an ASG behind NGINX for frontend and backend. NGINX itself becomes the next bottleneck — replace with AWS ALB at that point. Terraform modules are written to support this evolution without major rewrites.

---

## 🚢 Deployment Strategy

1. CI pipeline builds and tests the app (GitHub Actions)
2. Docker image pushed to ECR (or artifact to S3)
3. Terraform apply updates the Launch Template / user data
4. EC2 instance replaced via rolling update (zero-downtime for stateless services)
5. NGINX health checks gate traffic — no requests route to unhealthy instances
6. **Rollback:** `terraform apply` with prior `.tfvars` + previous image tag — takes ~3 minutes

**Downtime expectation:** Near-zero for rolling deploys. ~30 seconds for full instance replacement if needed.

---

## ⚠️ Failure & Operational Thinking

| Scenario | Impact | Recovery |
|---|---|---|
| Backend EC2 crashes | 502 from NGINX | EC2 auto-recovery → ~2 min |
| NGINX EC2 crashes | Full outage | Manual restart or AMI replacement |
| NAT Gateway fails | Backend loses internet egress | Restore NAT GW via Terraform |
| Route 53 misconfiguration | DNS resolution fails | Fix record + TTL propagation |
| AZ failure (ap-south-1a) | Full outage (single-AZ public) | Multi-AZ is the upgrade path |

**Alerting:** CloudWatch alarms on EC2 status checks + CPU threshold. SNS → email for `prod`. No 2 AM pages for `dev/staging`.

---

## ❌ What We Did NOT Do (and Why)

| Skipped | Reason |
|---|---|
| Kubernetes (EKS) | Massive operational overhead for a 2-service app. Cost: ~$150+/mo vs ~$20/mo for EC2. No justification. |
| AWS ALB | NGINX on EC2 is sufficient and cheaper at this traffic level. ALB is the upgrade path. |
| RDS / managed DB | No database in this application. |
| Multi-AZ prod | Single-AZ is an acceptable risk for this assignment scope. Documented upgrade path exists. |
| WAF | Not warranted for a demo app. Would add for any real production workload. |
| CDN (CloudFront) | Frontend is served directly. CDN is the obvious next step for global latency. |
| Auto Scaling Groups | Single EC2 per service. ASG is wired into the Terraform modules but not activated. |

---

## ✅ Assignment Checklist

- [x] Repository is a fork of the original
- [x] No commits pushed to the original repository
- [x] Documentation link provided (Google Docs)
- [x] Application deployed on AWS (`k8s.vaidhi.sbs`)
- [x] Second cloud (GCP) — *submitted separately*
- [x] Cloud & region choices clearly justified
- [x] Environments: dev / staging / prod exist with meaningful differences
- [x] Infrastructure provisioned using Terraform (IaC)
- [x] State strategy documented (S3 + DynamoDB)
- [x] Security & identity decisions explained
- [x] Failure & operational behavior documented
- [x] Future growth scenario addressed
- [x] "What we did NOT do" section included
- [x] Demo video link provided
- [x] Hosted URLs confirmed live

---


---

*Built for the PGAGI DevOps Full-Time Role assignment. Infrastructure-first, complexity-averse, production-minded.*

