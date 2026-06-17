# POC 01 — AWS Auto Scaling Group Policies (Target Tracking)

## The problem I wanted to solve

Interview question: *"What are the policies in Auto Scaling?"*

Reading documentation gives me the names of the 5 policy types. Building the lab gives me:

- The **deadband concept** — why AWS auto-creates 2 alarms when I set 1 number
- The **asymmetric** scale-out vs scale-in timing — and why it's the right default
- The **IAM Role + Instance Profile** pattern that makes SSM Session Manager work without any SSH key

These are the senior-level talking points. None of them are obvious from docs alone.

## What I built

A small EC2 Auto Scaling Group across 2 AZs with a Target Tracking policy on **Average CPU at 30%**. When I peg the CPU on one server using `stress-ng`, the ASG launches more servers automatically. When the load drops, it scales back in.

See [`architecture.md`](./architecture.md) for the diagram.

## Stack

| Layer | Service | Notes |
|---|---|---|
| Compute | **EC2** (`t3.micro`, Amazon Linux 2023) | Free tier eligible |
| Scaling | **Auto Scaling Group** + **Launch Template** + **Target Tracking Policy** | All from EC2 Console |
| Access | **SSM Session Manager** | No SSH, no PEM key, no inbound port 22 |
| Identity | **IAM Role** + **Instance Profile** | `AmazonSSMManagedInstanceCore` policy |
| Monitoring | **CloudWatch Alarms** | Auto-created by the Target Tracking policy |
| Region | `ap-south-1` (Mumbai) | |

## How to reproduce on your own AWS account

Step-by-step Console clicks: [`console-walkthrough.md`](./console-walkthrough.md)

## What I learnt — the 3 senior-level points

1. **Target Tracking auto-creates 2 CloudWatch alarms** — I set ONE number (target = 30), AWS gave me TWO thresholds (30 for scale-out and 27 for scale-in). Most people don't notice this until they see the alarms in the console.
2. **The 3% gap is called the "deadband"** — same idea as a thermostat (turn AC on at 24°C, off at 21°C). Stops the ASG from adding/removing servers every minute as CPU jitters around the target.
3. **Scale-out is fast, scale-in is slow — by design.** A bad scale-out costs an extra ₹5/hour. A bad scale-in drops a live customer mid-request. AWS picked the right asymmetry — availability over cost.

Detailed notes: [`lessons-learned.md`](./lessons-learned.md)

## Cost + time

- **Total AWS spend:** under ₹10 (1-3 × t3.micro × 30 min, free-tier eligible)
- **Build + test time:** ~45 minutes end-to-end
- **Teardown time:** ~2 minutes

## Skills demonstrated

- AWS EC2 Auto Scaling Groups + Launch Templates + Target Tracking math
- IAM — Roles, Instance Profiles, Trust vs Permissions policy split
- SSM Session Manager (passwordless, keyless EC2 access)
- CloudWatch Alarms — auto-created by Target Tracking, reading the deadband
- Linux load testing with `stress-ng`
- Cost-aware AWS engineering — free-tier instances + clean teardown order

## Bug I hit during the POC (and the fix)

First attempt was in `us-east-1`. ASG kept failing to launch in subnet `us-east-1e` with this error:

```
Your requested instance type (t3.micro) is not supported in your
requested Availability Zone (us-east-1e). Please retry by choosing
us-east-1a, us-east-1b, us-east-1c, us-east-1d, us-east-1f.
```

**Lesson:** Not every AZ supports every instance type. Always pick subnets explicitly when creating an ASG — don't blindly select all default subnets.

**Fix:** Switched the POC to `ap-south-1` (Mumbai), picked 2 AZs explicitly (`ap-south-1a` + `ap-south-1b`), launches succeeded.

## Screenshots

See [`screenshots/`](./screenshots/) folder.
