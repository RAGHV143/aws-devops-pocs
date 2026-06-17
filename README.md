# AWS DevOps — Hands-on POCs

Hi, I am **Raghvendra Sharma** — a DevOps engineer with 4.9 years of experience at Amdocs (Hyderabad). This repository is my personal lab for AWS, Kubernetes and Terraform.

## What you will find here

Each folder is one **end-to-end Proof of Concept (POC)**:

- The problem I wanted to understand
- How I built it (Console clicks and/or Terraform)
- What I learnt — including the senior-level points docs don't tell you
- Screenshots from the actual run on my AWS account
- Total cost + total time taken

## POCs in this repo

| # | Title | Service focus | Status |
|---|---|---|---|
| 01 | [Auto Scaling Group Policies — Target Tracking](./01-asg-auto-scaling-policies/) | EC2, ASG, IAM, SSM, CloudWatch | ✅ Done (Console) — Terraform version planned |

(More coming — Application Load Balancer, S3 lifecycle, EKS + IRSA, Karpenter, Disaster Recovery…)

## Why I build in public

For DevOps roles, the best resume is working code with a story. Each POC here was hand-built on my own AWS account and torn down right after — total spend stays under ₹50.

I do every POC the same way:

1. **Read the question** (interview question or real-world problem)
2. **Build it** on AWS — Console first to understand each click, then Terraform to lock it as code
3. **Break it on purpose** — change one knob, see what happens
4. **Tear it down** — every minute paid is a minute learnt
5. **Write the lesson** — a 1-page note that I can read 6 months later and still remember why each choice was made

## How to use this repo

- **Recruiters / hiring managers** — read the POC's `README.md` and `lessons-learned.md`. ~5 min each.
- **Engineers reproducing the lab** — follow `console-walkthrough.md` step by step. ~45 min on free tier.

## Connect

- LinkedIn: *(add your LinkedIn URL here before pushing)*
- Email: *(add your email here before pushing)*

---

*All POCs are built on a personal AWS account. No client data, no employer code. Opinions are my own.*
