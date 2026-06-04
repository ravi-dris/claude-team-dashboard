# Claude AI Team Usage Dashboard — Replication Guide

> Built for Dhwani Rural Information Systems | June 2026
> For questions: ravi.jangir@dhwaniris.com

---

## What This Is

A **self-contained HTML dashboard** that visualises your team's Claude AI usage — conversations, messages, Claude Code productivity, inactive members, and seat utilisation. Opens in any browser, no server or install needed.

---

## Step 1 — Export Your Data from Claude.ai

You need to be a **Team/Enterprise admin** on claude.ai.

### A. Full account data export (ZIP)
1. Go to [claude.ai](https://claude.ai) → **Settings → Account → Privacy**
2. Click **Export Data** → Download the ZIP file
3. The ZIP contains `conversations.json`, `users.json`, `projects/`, `memories.json`

### B. Members CSV
1. Go to **Settings → Organization → Members**
2. Click **Export** → Download CSV
3. Columns: Name, Email, Role, Status, Seat Tier

### C. Claude Code CSV (if you use Claude Code)
1. Go to **Settings → Organization → Claude Code**
2. Select the month → Click **Export**
3. Columns: User, Lines this Month

---

## Step 2 — Extract the Key Numbers

Open a terminal and run this Python script to extract metrics from the ZIP:

```python
import zipfile, json
from collections import defaultdict, Counter

ZIP = 'your-export.zip'   # path to your downloaded ZIP

with zipfile.ZipFile(ZIP) as z:
    convs = json.load(z.open('conversations.json'))
    users = json.load(z.open('users.json'))

user_map = {u['uuid']: u['email_address'] for u in users}

# Conversations per user per month
monthly = defaultdict(lambda: defaultdict(lambda: {'convs':0,'msgs':0}))
for c in convs:
    uid   = c['account']['uuid']
    email = user_map.get(uid)
    if not email: continue
    month = c['created_at'][:7]
    monthly[email][month]['convs'] += 1
    monthly[email][month]['msgs']  += sum(1 for m in c.get('chat_messages',[]) if m['sender']=='human')

# Print summary
for email in sorted(monthly.keys()):
    for month in sorted(monthly[email].keys()):
        d = monthly[email][month]
        print(f"{email} | {month} | convs={d['convs']} | msgs={d['msgs']}")

# Last active date per user
last_active = {}
for c in convs:
    uid   = c['account']['uuid']
    email = user_map.get(uid)
    if not email: continue
    d = c['created_at'][:10]
    if email not in last_active or d > last_active[email]:
        last_active[email] = d

print("\nLast active:")
for email, d in sorted(last_active.items(), key=lambda x: x[1], reverse=True):
    print(f"  {email}: {d}")
```

---

## Step 3 — Plug Your Numbers into the Dashboard

Open `dhwaniris_claude_report.html` in a text editor and update the **DATA section** at the top of the `<script>` block:

### 3a. Update MEMBERS array
```javascript
var MEMBERS = [
  {name:'Full Name', email:'user@org.com', ini:'AB', role:'User', tier:'Premium'},
  // one entry per team member
  // role: 'User' | 'Owner' | 'Primary Owner'
  // tier: 'Premium' | 'Standard'
];
```

### 3b. Update CONVS (conversations per user per month)
```javascript
var CONVS = {
  'all':     [302, 257, ...],   // all-time total, one number per MEMBERS entry
  '2026-04': [229,  37, ...],   // April only
  '2026-05': [ 69, 207, ...],   // May only
  '2026-06': [  4,  13, ...],   // June only (partial if mid-month)
};
```
> Order must match the MEMBERS array exactly.

### 3c. Update MSGS (human messages per user per month)
```javascript
var MSGS = {
  'all':     [1771, 1107, ...],
  '2026-04': [1423,  226, ...],
  '2026-05': [ 343,  833, ...],
  '2026-06': [   5,   48, ...],
};
```

### 3d. Update CODE (Claude Code lines, one month only)
```javascript
var CODE = [0, 105082, 9320, 57984, ...];   // index matches MEMBERS, 0 if not a code user
```

### 3e. Update LAST (last active date per user)
```javascript
var LAST = ['2026-06-03', '2026-06-04', ...];   // ISO date, index matches MEMBERS
```

### 3f. Update MONTHLY totals
```javascript
var MONTHLY = {
  '2026-04': {convs:608,  msgs:5398, users:9},
  '2026-05': {convs:532,  msgs:3322, users:9},
  '2026-06': {convs:49,   msgs:248,  users:7},
};
```

### 3g. Update DAILY data
```javascript
var DAILY = {
  '2026-04': { labels:['Apr 02', ...], data:[5, 9, ...] },
  '2026-05': { labels:['May 05', ...], data:[18,23, ...] },
  '2026-06': { labels:['Jun 01', ...], data:[20,13, ...] },
  'all':     { labels:['Apr 02', ...], data:[5, 20, ...] },  // sampled points
};
```

### 3h. Update report metadata
Find and change these lines near the top of the HTML:
```html
<div>Dhwani Rural Information Systems</div>   ← your org name
<div>Ravi Jangir, Strategy & Ops</div>         ← your name
<div>June 4, 2026</div>                        ← report date
<div>10 members | Claude Team</div>            ← team size & plan
```
Also update the month keys (`2026-04`, `2026-05`, etc.) and the `META` object to match your reporting period.

---

## Step 4 — Open & Share

1. Save the file
2. Open in any browser (Chrome, Edge, Firefox) — double-click the `.html` file
3. Share the `.html` file with anyone — it is **fully self-contained**, no internet needed for the layout (Chart.js loads from CDN, requires internet for first open)

---

## Dashboard Sections

| Section | What It Shows |
|---|---|
| Month Filter (sticky) | Switch between All Time / Apr / May / Jun |
| 5 KPI Cards | Conversations, Messages, AI Responses, Active Members, Inactive Count |
| Inactive Members Panel | Names + last active date + days since, color-coded urgency |
| Seat Utilisation Chart | Doughnut — active vs inactive seats |
| Claude Code Highlights | Lines accepted, accept rate, top contributor (May only) |
| Daily Trend Chart | Conversations per day for selected month |
| Monthly Comparison | All 3 months side-by-side (always visible) |
| User Conv Chart | Per-member bar chart, sorted by activity |
| Claude Code Lines Chart | Per-developer code output (May only) |
| Member Table | Full breakdown — rank, convs, messages, avg depth, code lines, status |
| Key Insights | 3 narrative cards — adoption, ROI, opportunities |

---

## Tips for Next Month's Report

1. Re-export from claude.ai (Steps 1A–1C)
2. Re-run the Python script (Step 2)  
3. Add the new month's data to all arrays (shift 'all' totals, add new month key)
4. Update `REPORT_DATE` in the JS to the new report date
5. Save and share

---

*Built with Claude Code · Anthropic · June 2026*
