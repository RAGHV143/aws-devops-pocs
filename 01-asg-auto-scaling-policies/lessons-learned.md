# Lessons Learnt — Auto Scaling Group Policies

This is what I learnt by actually building the POC, beyond what documentation tells you.

## 1. The 5 Auto Scaling Policy types — and when to use each

| Type | Restaurant analogy | Use it when… |
|---|---|---|
| **Target Tracking** | "Keep tables 70% full; you decide how." | 80% of cases — set it once and forget it |
| **Step Scaling** | "50-70% full → call 1 waiter; 70-90% → call 3; >90% → call 5" | You want a bigger reaction for a bigger problem |
| **Simple Scaling** | "If full, call 2 more. Wait 15 min before doing anything else." | Old style. Don't use for new work. |
| **Scheduled** | "Every Fri 7 PM, have 10 ready — wedding night." | Regular events you can mark on a calendar |
| **Predictive** | "Sat lunch always crowded at 12:30. Call 8 waiters at 12:00." | Daily/weekly patterns with at least 24 hours of past data |

## 2. The Deadband — the senior-level concept

A thermostat does not turn the AC on at 24°C and off at 24°C — it would click on/off every 30 seconds as the room temperature jumps around. So a thermostat keeps a **deadband** (a small gap): turn AC on at 24°C, turn off at 21°C. Between 21-24°C, do nothing.

Target Tracking does the exact same thing:

- I set Target = 30% CPU
- AWS automatically created **2 alarms**:
  - **AlarmHigh** at >30% → scale OUT
  - **AlarmLow** at <27% → scale IN
  - 27-30% CPU = **deadband** → do nothing
- The 3% gap stops the system from adding and removing servers every minute as CPU jitters

**One sentence to remember:** *"I set ONE number, AWS auto-created TWO alarms with a 3% gap. That gap is the deadband — same idea as a thermostat — stops the ASG from flapping."*

## 3. Scale-out vs Scale-in is asymmetric — by design

- **Scale-OUT** triggers fast (1-3 minutes after CPU crosses the threshold)
- **Scale-IN** is deliberately slow (10-15 minutes after CPU drops)

Why? A bad scale-out costs an extra ₹5/hour for one extra server. A bad scale-in drops a live server in the middle of serving customers.

AWS picked the right asymmetry — **availability over cost** is the safer default for the 99% case. If you want the opposite (cost over availability), you have to write the logic outside the ASG using Lambda + CloudWatch.

## 4. SSM Session Manager > SSH PEM keys

I configured the Launch Template with:

- **No** SSH key pair
- **No** inbound port 22 in the Security Group
- IAM Instance Profile with `AmazonSSMManagedInstanceCore`

And I could still get a shell on the EC2 — through **Session Manager**.

| Old way (PEM keys) | New way (SSM Session Manager) |
|---|---|
| Hand every engineer a `.pem` file | Add their IAM identity to a group |
| Cannot cancel once copied | Cancel access with 1 IAM click |
| No record of who logged in | Every session logged in CloudTrail |
| Port 22 open = attack surface | No ports open at all |
| Free | Free |

This is literally a separate interview answer — *"How do you give SSH access to an EC2 instance without sharing the PEM key?"* My answer: I don't use SSH at all.

## 5. The IAM Role — Instance Profile pattern

The hidden detail most beginners miss: when you create an IAM Role with EC2 trust, AWS **quietly creates an Instance Profile with the same name**. You attach the **Instance Profile** (not the role directly) to the EC2.

Mental model:

```
IAM Role  ←wrapped in←  Instance Profile  ←attached to←  EC2 Instance
   ↑                                                         ↓
Trust policy: "EC2 is allowed to assume me"   EC2 boots, assumes the role
   ↓
Permissions policy (SSM/S3/etc.)
   ↓
Code on EC2 calls AWS APIs with NO keys stored on disk
```

The 2 sides every role has:
- **Trust policy** = WHO is allowed to assume this role
- **Permissions policy** = WHAT this role can do once it is assumed

Forget either side and the role won't work.

## 6. Two separate IAM permissions for SSM Session Manager

When I was learning SSM, I confused the two sets of IAM permissions involved. Writing this clearly so I don't confuse it again:

| Side | What is needed | Why |
|---|---|---|
| **TARGET EC2** (the one you connect TO) | Instance Profile with `AmazonSSMManagedInstanceCore` | Lets the SSM Agent on that EC2 register with the SSM service |
| **YOUR identity** (IAM user / SSO / source EC2 role) | Policy with `ssm:StartSession` | Lets YOU start a session on that target |

**Both must exist.** Forget either one and the connection fails.

## 7. The bug I hit — and the fix

When I first deployed in `us-east-1`, the ASG kept failing to launch in subnet `us-east-1e`:

```
Your requested instance type (t3.micro) is not supported
in your requested Availability Zone (us-east-1e).
```

**Lesson:** Not every AZ supports every instance type. Always pick subnets explicitly when creating an ASG — don't blindly select all default subnets.

**Fix:** Switched to `ap-south-1` (Mumbai), used 2 AZs explicitly (`ap-south-1a` and `ap-south-1b`), launches went smoothly.

## 8. The order of teardown matters

To delete cleanly without orphans:

1. ASG first (this also auto-deletes the 2 CloudWatch alarms)
2. Launch Template
3. Security Group
4. IAM Role

If you delete the IAM Role first, the running EC2 still works (already has cached credentials for ~1 hour) but new launches and SSM sessions fail confusingly. Order matters.

## 9. Likely follow-up interview questions

After answering the main "What are Auto Scaling policies?" question, expect any of these:

| Follow-up | Short answer |
|---|---|
| Target Tracking vs Step Scaling for a payments API? | Target Tracking on `RequestCountPerTarget` (not CPU — payments wait on database). Add Step Scaling on top for sudden spikes. |
| What happens during scale-in if a request is in the middle of finishing? | 3 tools: **Scale-in Protection** (do not delete this server) + **Lifecycle Hooks** (run cleanup script) + **ALB Connection Draining** (let in-flight requests finish) |
| ASG stuck — desired=5 but only 2 healthy. Where do you look? | 1) Activity history first, 2) IAM Role still exists?, 3) AMI broken?, 4) Subnet ran out of IPs?, 5) Account vCPU limit hit? |
| Compare with K8s HPA + Cluster Autoscaler | "ASG manages servers. HPA manages pods. Cluster Autoscaler is the bridge between them. In EKS we have all three." Mention **Karpenter** as the modern replacement for CA. |
| Multiple policies on one ASG — what if they fight? | Adding servers → the one that wants MORE wins. Removing → the one that wants FEWER wins. Availability over cost. |

## What I would do differently next time

- Build the **same POC in Terraform** to lock the design as code (planned next)
- Use **Karpenter** instead of plain ASG for production-grade EKS clusters (faster, cheaper, picks instance type per pod)
- Layer **Predictive Scaling** on top of Target Tracking for daily/weekly traffic patterns
- Add a **Lifecycle Hook** for graceful scale-in (drain in-flight requests before terminating)
- Put an **Application Load Balancer** in front (so traffic gets distributed across the ASG members)
