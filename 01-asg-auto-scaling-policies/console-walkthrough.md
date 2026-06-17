# Console Walkthrough — Step by Step

This is exactly what I clicked on AWS Console to build the POC end-to-end.

> **Region:** `ap-south-1` (Mumbai)
> **Total time:** ~45 minutes
> **Total cost:** under ₹10 if you tear down right after

---

## Part A — Create the IAM Role (so EC2 can use SSM)

1. AWS Console → **IAM** → **Roles** → **Create role**
2. **Trusted entity type:** AWS service
3. **Use case:** EC2 → Next
4. **Permissions:** search for `AmazonSSMManagedInstanceCore` → tick the box → Next
5. **Role name:** `asg-poc-ssm-role` → Create role

> 📝 **Note:** AWS quietly creates an **Instance Profile** with the same name. Console hides this. You will use this name in Part B.

**Why this role?**
- The **trust policy** says *"EC2 service is allowed to assume me"*
- The **permissions policy** lets the EC2 talk to SSM service for shell access
- Both sides must exist or the role is useless

---

## Part B — Create the Launch Template

1. EC2 Console → **Launch Templates** → **Create launch template**
2. **Name:** `asg-poc-lt`
3. **Auto Scaling guidance:** tick *"Provide guidance to help me set up a template..."*
4. **AMI:** Amazon Linux 2023 (latest version in your region)
5. **Instance type:** `t3.micro` (free tier eligible)
6. **Key pair:** **Don't include in launch template** *(no SSH keys — SSM gives us shell)*
7. **Network settings → Security groups:** create a new SG `asg-poc-sg` with:
   - **Inbound rules:** empty *(no port 22, no anything)*
   - **Outbound rules:** All traffic to `0.0.0.0/0` *(needed so SSM Agent can reach AWS API)*
8. **Advanced details → IAM instance profile:** pick `asg-poc-ssm-role`
9. **Advanced details → Auto-assign public IP:** **Enable** *(needed for SSM to reach internet — or use VPC Endpoints in private subnet)*
10. Create launch template

---

## Part C — Create the Auto Scaling Group with Target Tracking

1. EC2 Console → **Auto Scaling Groups** → **Create Auto Scaling group**
2. **Name:** `asg-poc-asg`
3. **Launch template:** pick `asg-poc-lt` (Latest version) → Next
4. **VPC:** Default VPC
5. **Availability Zones and subnets:** pick subnets in 2 AZs (e.g., `ap-south-1a` and `ap-south-1b`) → Next
6. **Load balancer:** None for this POC → Next
7. **Group size:**
   - Desired capacity: `1`
   - Minimum capacity: `1`
   - Maximum capacity: `3`
8. **Scaling policies:** select **Target tracking scaling policy**
   - Policy name: `target-tracking-cpu`
   - Metric type: **Average CPU utilization**
   - **Target value:** `30` *(I picked 30 instead of 70 only so we can trigger scale-out easily during the demo. In production, use 60-70%.)*
   - Instances need: `60` seconds to warm up
9. Skip notifications + tags → Review → Create Auto Scaling group

> ⏱️ Wait ~60 seconds. ASG launches 1 EC2.

---

## Part D — Verify and Stress Test

### D.1 — Confirm the alarms were auto-created

1. CloudWatch Console → **Alarms**
2. Search for `TargetTracking-asg-poc-asg`
3. You should see **exactly 2 alarms:**
   - `*AlarmHigh*` — threshold = `30`, *Greater than* → triggers scale-OUT
   - `*AlarmLow*` — threshold = `27`, *Less than* → triggers scale-IN

🎯 **Senior-level observation:** I set ONE number (target = 30). AWS automatically created TWO alarms with a 3% gap. That gap is the **deadband** — it stops the system from adding and removing servers when CPU jitters around the target. (Same idea as a thermostat: turn AC on at 24°C, off at 21°C.)

### D.2 — Connect to the EC2 via SSM (no SSH key needed)

1. EC2 → Instances → click the instance → **Connect** button
2. Tab: **Session Manager** → **Connect**
3. A browser-based shell opens. **No keys, no port 22, no bastion server.**

> ⚠️ **If Ping status shows `Offline`:** check that the EC2 has a **Public IPv4** address. If empty, stop the EC2 → Networking → enable auto-assign public IP → start it again. Without internet path, the SSM Agent cannot reach AWS API and login fails.

### D.3 — Spike the CPU

In the SSM session:

```bash
sudo dnf install -y stress-ng
stress-ng --cpu $(nproc) --timeout 600s &
top -bn1 | head -10
```

You should see CPU at ~100%. Type `exit` to leave the SSM session — **the load keeps running on the EC2.**

If `dnf install -y stress-ng` fails (not in default AL2023 repo), fallback:

```bash
# Pure shell, zero install — pegs every CPU core
for i in $(seq 1 $(nproc)); do yes > /dev/null & done
# To stop later: pkill yes
```

### D.4 — Watch the ASG scale OUT

1. EC2 → Auto Scaling Groups → `asg-poc-asg` → **Activity** tab
2. Within 3-6 minutes you will see new rows like:
   - *"Launching a new EC2 instance: i-0yyy..."*
   - *"Cause: ...changing the desired capacity from 1 to 2..."*
3. Possibly a third row taking it to `desired = 3`

📸 **Take a screenshot of this Activity table** — this is your proof-of-work for the portfolio. Save it as `screenshots/02-scaling-activity-1-to-3.png`.

### D.5 — Stop the load and watch it scale IN

1. SSM back into any of the running instances
2. Run: `sudo pkill stress-ng` (or `pkill yes` if you used the fallback) → `exit`
3. **Wait 10-15 minutes** — scale-in is deliberately slow
4. Activity table will show: *"Terminating EC2 instance..."* twice, back to `desired = 1`

📝 **Why scale-in is slow:** AWS protects against a brief CPU dip causing a server to die mid-request. A bad scale-out costs an extra ₹5/hour. A bad scale-in drops live customers. AWS picked the right asymmetry.

---

## Part E — Teardown (do this or pay the meter)

Delete in this exact order to avoid orphans:

1. EC2 → **Auto Scaling Groups** → `asg-poc-asg` → Delete → confirm
2. EC2 → **Launch Templates** → `asg-poc-lt` → Delete
3. EC2 → **Security Groups** → `asg-poc-sg` → Delete
4. IAM → **Roles** → `asg-poc-ssm-role` → Delete
5. CloudWatch → **Alarms** — the 2 Target Tracking alarms are auto-deleted when the ASG dies (verify they are gone)

Verify nothing is left from the AWS CLI:

```bash
aws autoscaling describe-auto-scaling-groups --region ap-south-1 \
  --query 'AutoScalingGroups[?contains(AutoScalingGroupName, `asg-poc`)].AutoScalingGroupName'
# Should return:  []
```

**Total spend:** under ₹10.

---

## Common gotchas (and how I fixed each)

| Issue | What I saw | Fix |
|---|---|---|
| ASG won't launch instance | Error: *"t3.micro not supported in us-east-1e"* | Pick subnets explicitly — skip `us-east-1e` for `t3.micro` |
| SSM Session Manager: Ping status `Offline` | i/o timeout to `ssm.<region>.amazonaws.com` | EC2 had no Public IP. Stop → enable auto-assign public IP → start |
| `stress-ng: command not found` | New AL2023 has no stress-ng pre-installed | `sudo dnf install -y stress-ng` (or fallback to `yes > /dev/null` loop) |
| CloudWatch alarms in `INSUFFICIENT_DATA` for 3 min | Default 3 × 60s data points | Wait. Don't stress until both alarms show `OK`. |
