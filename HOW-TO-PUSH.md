# How to push this repo to GitHub

> This file is for you (Raghvendra) only — **delete it before pushing** or it goes public.

## Step 1 — Sanity check (5 min)

Before anything goes online, do these checks:

```bash
cd "C:\Users\raghvens\OneDrive - AMDOCS\Cursor\React-Application-Dex\Resume_GenAI_Devops\aws-devops-pocs"

# 1. No AWS account ID hardcoded anywhere
grep -r "852921658206" .          # Should return nothing

# 2. No personal email or phone
grep -ri "amdocs" .                # Check what shows up — sanitize if needed
grep -ri "@gmail\|@amdocs" .      # Should return nothing

# 3. No PEM files, no .tfstate
find . -name "*.pem"               # Empty
find . -name "*.tfstate*"          # Empty
```

If anything is found — fix before pushing.

## Step 2 — Add your contact info to the top-level README

Open `README.md` and replace these placeholders:

```
LinkedIn: *(add your LinkedIn URL here before pushing)*
Email: *(add your email here before pushing)*
```

with your actual details.

## Step 3 — Add the screenshots

Save your 3 sanitized screenshots into `01-asg-auto-scaling-policies/screenshots/` with these exact filenames:

- `01-cloudwatch-alarms-deadband.png`
- `02-scaling-activity-1-to-3.png`
- `03-stress-ng-cpu-spike.png`

(See the `screenshots/README.md` for what each should show.)

## Step 4 — Create the GitHub repo (browser)

1. Go to <https://github.com/new>
2. **Repository name:** `aws-devops-pocs`
3. **Description:** *"Hands-on AWS DevOps POCs — each folder is one end-to-end lab with notes, screenshots, and lessons learnt."*
4. **Public** ✅
5. **Add a README:** ❌ (we already have one)
6. **Add .gitignore:** ❌ (we already have one)
7. **Add a license:** ✅ → **MIT License**
8. Click **Create repository**

GitHub now shows you the empty repo with push instructions.

## Step 5 — Push from your laptop (terminal)

Open PowerShell or Git Bash in this folder:

```bash
cd "C:\Users\raghvens\OneDrive - AMDOCS\Cursor\React-Application-Dex\Resume_GenAI_Devops\aws-devops-pocs"

# Delete this how-to file before pushing
rm HOW-TO-PUSH.md

# First-time git setup (only if you've never done this on this laptop)
git config --global user.name "Raghvendra Sharma"
git config --global user.email "your-public-email@example.com"

# Initialize and push
git init
git add .
git commit -m "Add POC 01: AWS Auto Scaling Group Policies (Target Tracking)"
git branch -M main
git remote add origin https://github.com/<your-github-username>/aws-devops-pocs.git
git push -u origin main
```

Replace `<your-github-username>` with your actual GitHub username.

## Step 6 — Polish the GitHub profile

After the push works:

1. Go to your GitHub profile page
2. Click **Customize your pins** in the right sidebar
3. **Pin** the `aws-devops-pocs` repo
4. (Optional but recommended) Edit your GitHub profile bio:
   *"DevOps Engineer @ Amdocs · 4.9 yrs · AWS, Kubernetes, Terraform · Building hands-on POCs in public"*
5. Add a profile picture if you don't have one (recruiters check)

## Step 7 — Share on LinkedIn

Sample post (edit before sharing):

> Just shipped my first POC in a new "build in public" series — **AWS Auto Scaling Group with Target Tracking policy.**
>
> Built it twice — once on AWS Console to understand each click, then I'm doing it in Terraform to lock the design as code.
>
> The senior-level point that surprised me: I set ONE target number (CPU at 30%), and AWS automatically created TWO CloudWatch alarms with a 3% gap. That gap is the **deadband** — same idea as a thermostat — and it's what stops the ASG from flapping.
>
> Repo (with screenshots, console walkthrough, and lessons learnt):
> 👉 https://github.com/<your-username>/aws-devops-pocs
>
> Up next: Application Load Balancer in front of this same ASG.
>
> #AWS #DevOps #BuildInPublic #LearningOutLoud

## Step 8 — Delete this file

```bash
rm HOW-TO-PUSH.md
git add . && git commit -m "Remove internal notes" && git push
```

(Or just delete it before the first push and you're done.)

---

## What if I want to add more POCs later?

Easy. Each new POC = one new top-level folder.

```bash
cd aws-devops-pocs
mkdir 02-alb-path-host-routing
cd 02-alb-path-host-routing
# ... create README.md, console-walkthrough.md, lessons-learned.md, architecture.md, screenshots/
git add .
git commit -m "Add POC 02: ALB path and host based routing"
git push
```

Then update the table in the top-level `README.md`.

That's it. Each POC keeps its own self-contained chapter.
