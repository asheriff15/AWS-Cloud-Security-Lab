# AWS Cloud Troubleshooting Lab — VPC, EC2 & Least-Privilege IAM

![Status](https://img.shields.io/badge/Status-Complete-brightgreen)
![Platform](https://img.shields.io/badge/Platform-AWS-FF9900?logo=amazonaws&logoColor=white)
![Services](https://img.shields.io/badge/Services-VPC%20%7C%20EC2%20%7C%20IAM%20%7C%20S3-blue)
![Focus](https://img.shields.io/badge/Focus-Break%2FFix%20Troubleshooting-red)

## Overview

Built a small AWS environment from scratch — a custom VPC, public subnet, internet gateway, route table, security group and EC2 instance — then **deliberately broke it twice and fixed it**:

1. **Network failure:** removed the security group's SSH rule, diagnosed the resulting connection timeout, and restored access.
2. **Permissions failure:** the instance couldn't reach S3 because it had no credentials. Fixed it with an **IAM role** (no access keys on the box), then proved the role was **least-privilege** — it could *list* S3 but was *denied* creating a bucket.

The point of the lab is the troubleshooting, not the clicking: reading the symptom, identifying which layer is failing (network vs. identity), and fixing it the right way.

---

## Tools & Technologies

| Category | Tool |
|---|---|
| Cloud Provider | AWS (us-east-2, Ohio) |
| Networking | VPC, Subnet, Internet Gateway, Route Table |
| Compute | EC2 `t3.micro`, Amazon Linux 2023 |
| Network Security | Security Group (SSH restricted to my IP `/32`) |
| Identity | IAM Role + Instance Profile, `AmazonS3ReadOnlyAccess` |
| Storage | Amazon S3 |
| Access | SSH with key pair, AWS CLI |

---

## Architecture

```
                         Internet
                            │
                  ┌─────────▼─────────┐
                  │  lab-igw (IGW)    │
                  └─────────┬─────────┘
┌─────────────────── lab-vpc 10.0.0.0/16 ────────────────────┐
│  Route table: 0.0.0.0/0 → lab-igw  |  10.0.0.0/16 → local  │
│                                                             │
│   ┌──────── lab-public-subnet 10.0.1.0/24 (us-east-2a) ───┐ │
│   │                                                       │ │
│   │   lab-sg: inbound SSH 22 from MY_IP/32 only           │ │
│   │   ┌───────────────────────────────┐                   │ │
│   │   │ EC2 lab-instance (t3.micro)   │── IAM role ──────────────► S3
│   │   │ Amazon Linux 2023, public IP  │   lab-s3-role     │ │  (read-only)
│   │   └───────────────────────────────┘                   │ │
│   └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## Project Walkthrough

### Phase 1 — Build the network
- Created **lab-vpc** (`10.0.0.0/16`) and a public subnet **lab-public-subnet** (`10.0.1.0/24`) in us-east-2a
- Created internet gateway **lab-igw** and attached it to the VPC
- Added a `0.0.0.0/0 → lab-igw` route so the subnet can reach the internet

![Create VPC](screenshots/01-create-vpc.png)
![VPC created](screenshots/02-vpc-created.png)
![Create subnet](screenshots/03-create-subnet.png)
![Subnet created](screenshots/04-subnet-created.png)
![Internet gateway created](screenshots/05-igw-created.png)
![Internet gateway attached](screenshots/06-igw-attached.png)
![Route table with default route to the IGW](screenshots/07-route-table.png)

---

### Phase 2 — Lock down access and launch EC2
- Created security group **lab-sg** with a single inbound rule: **SSH (22) from my IP only (`/32`)** — not `0.0.0.0/0`
- Created an RSA key pair and launched a `t3.micro` Amazon Linux 2023 instance into the public subnet with a public IP
- Connected over SSH and confirmed outbound internet access with `ping`

![Create security group](screenshots/08-create-security-group.png)
![Security group created](screenshots/09-security-group-created.png)
![Key pair](screenshots/10-key-pair.png)
![EC2 network settings](screenshots/11-ec2-network-settings.png)
![Instance running](screenshots/12-ec2-launched.png)
![SSH connected](screenshots/13-ssh-connected.png)
![Internet connectivity test](screenshots/14-internet-connectivity.png)

---

### Phase 3 — Break/fix #1: SSH stops working

| | |
|---|---|
| **Break** | Deleted the only inbound rule from `lab-sg` |
| **Symptom** | `ssh: connect to host … port 22: Operation timed out` |
| **Diagnosis** | A **timeout** (not "connection refused") means packets are being silently dropped before they reach the instance — a firewall/network-layer problem, not SSH itself. Security groups drop disallowed traffic silently, which matches. |
| **Fix** | Re-added inbound SSH from my IP `/32` → reconnected immediately |

![Inbound rule removed](screenshots/15-inbound-rule-removed.png)
![SSH timeout](screenshots/16-ssh-timeout.png)
![Inbound rule restored](screenshots/17-inbound-rule-restored.png)
![SSH reconnected](screenshots/18-ssh-reconnected.png)

---

### Phase 4 — Break/fix #2: the instance can't reach S3

| | |
|---|---|
| **Symptom** | `aws s3 ls` → `Unable to locate credentials` |
| **Diagnosis** | The network was fine — this is an **identity** problem. The instance had no IAM role, so the AWS CLI had no credentials to sign requests with. |
| **Wrong fix** | Running `aws configure` and pasting long-lived access keys onto the server |
| **Right fix** | Created IAM role **lab-s3-role** (trusted by `ec2.amazonaws.com`) with `AmazonS3ReadOnlyAccess` and attached it to the instance. The CLI now gets short-lived, auto-rotating credentials from the instance metadata service. |

![S3 access denied - no credentials](screenshots/19-s3-no-credentials.png)
![IAM role created](screenshots/20-iam-role-created.png)
![IAM role attached to EC2](screenshots/21-iam-role-attached.png)

---

### Phase 5 — Prove least privilege
- `aws s3 mb` (create bucket) from the instance → **AccessDenied** — the read-only role correctly can't write
- Created the bucket from the console instead, then ran `aws s3 ls` from the instance → the bucket is listed ✅

The role can do exactly what it needs (read) and nothing more (write).

![CreateBucket denied](screenshots/22-s3-createbucket-denied.png)
![Bucket created in console](screenshots/23-s3-bucket-created.png)
![s3 ls succeeds from EC2](screenshots/24-s3-ls-success.png)

---

## Key Takeaways

| Concept | What I learned |
|---|---|
| Timeout vs. refused | A timeout points to a dropped packet (security group, NACL, route); "refused" means traffic arrived but nothing was listening |
| Security groups | Stateful, deny-by-default — removing the only inbound rule cuts off access instantly, with no error message |
| Network vs. identity | "Can't connect" and "Unable to locate credentials" are different layers — diagnose the layer before changing anything |
| IAM roles over keys | Roles give temporary credentials via instance metadata; no secrets stored on the server |
| Least privilege | Verified the permission boundary by testing an action that *should* fail, not just one that should succeed |

---

## What I'd Do Differently in Production

- **Remove SSH entirely** and use **AWS Systems Manager Session Manager** — no open port 22, no key pairs to manage
- **Use a dedicated public route table** instead of editing the VPC's main route table
- **Scope the IAM policy to one bucket** (e.g., `s3:ListBucket` / `s3:GetObject` on a specific ARN) instead of read access to all of S3
- **Define everything in Terraform** so the environment is reproducible and reviewable — planned as the next iteration of this project

---

## Skills Demonstrated

- AWS networking: VPC design, subnetting, internet gateways, routing
- Security group configuration and network access control
- EC2 provisioning and Linux administration over SSH
- Structured troubleshooting: symptom → layer → root cause → fix → verify
- IAM roles, trust policies and instance profiles
- Least-privilege access design and verification
- Security documentation and GitHub version control

> Screenshots have been redacted to remove the AWS account ID, public IP addresses and local machine details.

---

More of my projects and write-ups: **[asheriff15.github.io](https://asheriff15.github.io)**
