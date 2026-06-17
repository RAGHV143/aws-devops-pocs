# Architecture

```
                    ┌─────────────────────────────────────┐
                    │   AWS Account  (region: ap-south-1) │
                    │                                     │
   ┌───────────┐    │   ┌─────────────────────────────┐   │
   │ Engineer  │────┼──→│  AWS Console / SSM Service  │   │
   │ (browser) │    │   └──────────────┬──────────────┘   │
   └───────────┘    │                  │ "open shell"     │
                    │                  ↓                  │
                    │   ┌─────────────────────────────┐   │
                    │   │  Auto Scaling Group         │   │
                    │   │  min=1, desired=1, max=3    │   │
                    │   └─────────────┬───────────────┘   │
                    │                 │ launches          │
                    │                 ↓                   │
                    │   ┌─────────────────────────────┐   │
                    │   │  Launch Template            │   │
                    │   │  - t3.micro                 │   │
                    │   │  - Amazon Linux 2023        │   │
                    │   │  - IAM Instance Profile     │   │
                    │   │    (AmazonSSMManagedInstance│   │
                    │   │     Core)                   │   │
                    │   │  - Auto-assign public IP    │   │
                    │   │  - SG: outbound 443 only    │   │
                    │   │       (no inbound rules)    │   │
                    │   └─────────────┬───────────────┘   │
                    │                 │                   │
                    │       ┌─────────┴─────────┐         │
                    │       ↓                   ↓         │
                    │   ┌──────┐            ┌──────┐      │
                    │   │ EC2  │            │ EC2  │      │
                    │   │ AZ-a │            │ AZ-b │      │
                    │   └───┬──┘            └───┬──┘      │
                    │       │                   │         │
                    │       └─────────┬─────────┘         │
                    │                 │ CPU metric        │
                    │                 ↓                   │
                    │   ┌─────────────────────────────┐   │
                    │   │  CloudWatch Alarms          │   │
                    │   │  (auto-created by Target    │   │
                    │   │   Tracking policy)          │   │
                    │   │                             │   │
                    │   │  - AlarmHigh > 30% → out    │   │
                    │   │  - AlarmLow  < 27% → in     │   │
                    │   │                             │   │
                    │   │  Gap of 3% = "deadband"     │   │
                    │   │  prevents flap              │   │
                    │   └─────────────────────────────┘   │
                    └─────────────────────────────────────┘
```

## Components — what each piece does

| Component | Purpose |
|---|---|
| **Launch Template** | Recipe for new servers (image, type, IAM profile, network settings) |
| **Auto Scaling Group** | Group manager that adds/removes servers based on policy |
| **Target Tracking Policy** | Rule: *"keep CPU at 30%"* — AWS handles the math |
| **CloudWatch Alarms** | Auto-created by the policy — the eyes that trigger scale events |
| **IAM Instance Profile** | Lets the EC2 talk to AWS APIs without storing keys on disk |
| **SSM Session Manager** | Browser-based shell on the EC2 — no SSH, no PEM key, no port 22 |

## What is NOT in this diagram (and would be in production)

These are intentionally left out to keep the POC focused on Auto Scaling itself. Each of these is a planned future POC:

- **Application Load Balancer (ALB)** in front of the ASG — distributes traffic, removes/adds targets automatically as ASG members come and go
- **Lifecycle Hooks** — graceful scale-in (drain in-flight requests before terminating)
- **VPC Endpoints** for SSM — so private-subnet EC2s without internet can still use Session Manager
- **CloudWatch Log Group** for SSM session recording — every command typed in any SSM session is logged for audit
- **Predictive Scaling** layered on top of Target Tracking — for daily/weekly traffic patterns where ML can pre-warm servers

## Data flow during a scale-out event

1. Engineer runs `stress-ng --cpu $(nproc)` on one EC2 via SSM
2. EC2 reports CPU = 95% to CloudWatch every 60 seconds
3. CloudWatch sees 3 consecutive data points above 30 → AlarmHigh transitions to `ALARM`
4. AlarmHigh notifies the ASG: *"add a server"*
5. ASG asks Launch Template to create a new EC2
6. EC2 boots in another AZ, takes ~60 seconds to be `InService`
7. Average CPU across the 2 EC2s drops below 30 → AlarmHigh back to `OK` → no more launches
8. Total time: 3-6 minutes from stress start to second EC2 ready
