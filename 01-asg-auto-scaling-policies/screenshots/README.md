# Screenshots

Drop the following 3 screenshots from your AWS Console run here:

| Filename | What it shows | Why it matters |
|---|---|---|
| `01-cloudwatch-alarms-deadband.png` | The 2 auto-created CloudWatch alarms with thresholds 30 and 27 | Proves the **deadband** concept visually — shows the 3% gap between AlarmHigh and AlarmLow |
| `02-scaling-activity-1-to-3.png` | The ASG Activity tab showing instance launches (1 → 2 → 3) | Proves the policy fired and the group actually scaled out |
| `03-stress-ng-cpu-spike.png` | `top` output during stress test, all cores near 100% | Shows the trigger that caused the scaling event |

## Before you commit screenshots — sanitize first

Recruiters and the public will see these. Before pushing, blur or crop:

- ❌ **AWS Account ID** (the 12-digit number, e.g. `852921658206`)
- ❌ **Instance IDs** you don't want public (`i-xxxxx`)
- ❌ **Personal email** in top-right corner of Console
- ❌ Any **internal company name or VPN hostname**
- ❌ **Public IP addresses** of your instances

## Tools to sanitize quickly

- **Windows Snip & Sketch** — built-in, free, has highlight/blur tool
- **GIMP / Photoshop** → Filter → Blur → Gaussian Blur on selected region
- **macOS Preview** → mark up tool → black rectangle

## Naming convention

Use the prefix numbers `01-`, `02-`, `03-` so screenshots line up with the order they appear in the [console-walkthrough.md](../console-walkthrough.md).
