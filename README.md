# Claude AI Team Usage Dashboard — Replication Guide

> Built for Dhwani Rural Information Systems | June 2026
> For questions: ravi.jangir@dhwaniris.com
> Live dashboard: https://ravi-dris.github.io/claude-team-dashboard/
> Source code: https://github.com/ravi-dris/claude-team-dashboard

---

## What This Is

A **self-contained HTML dashboard** that visualises your team's Claude AI usage — conversations, messages, Claude Code productivity, inactive members, and seat utilisation. Opens in any browser, no server or install needed. Has a month filter (All Time / monthly) that updates every section dynamically.

---

## Step 1 — Export Your Data from Claude.ai

You need to be a **Team/Enterprise admin** on claude.ai.

### A. Full account data export (ZIP)
1. Go to [claude.ai](https://claude.ai) → **Settings → Account → Privacy**
2. Click **Export Data** → Download the ZIP
3. Contains: `conversations.json`, `users.json`, `projects/`, `memories.json`

### B. Members CSV
1. Go to **Settings → Organization → Members**
2. Click **Export** → Download CSV
3. Columns: Name, Email, Role, Status, Seat Tier

### C. Claude Code CSV (if your team uses Claude Code)
1. Go to **Settings → Organization → Claude Code**
2. Select the month → Click **Export**
3. Columns: User, Lines this Month

---

## Step 2 — Extract the Numbers (Python script)

Run this on the ZIP to get per-user, per-month conversation and message counts:

```python
import zipfile, json
from collections import defaultdict

ZIP = 'your-export.zip'   # path to your downloaded ZIP

with zipfile.ZipFile(ZIP) as z:
    convs = json.load(z.open('conversations.json'))
    users = json.load(z.open('users.json'))

user_map = {u['uuid']: u['email_address'] for u in users}

# Conversations + messages per user per month
monthly = defaultdict(lambda: defaultdict(lambda: {'convs':0,'msgs':0}))
for c in convs:
    email = user_map.get(c['account']['uuid'])
    if not email: continue
    month = c['created_at'][:7]
    monthly[email][month]['convs'] += 1
    monthly[email][month]['msgs']  += sum(1 for m in c.get('chat_messages',[]) if m['sender']=='human')

for email in sorted(monthly):
    for month in sorted(monthly[email]):
        d = monthly[email][month]
        print(f"{email} | {month} | convs={d['convs']} | msgs={d['msgs']}")

# Last active date per user (chat only — update manually for Claude Code users)
last_active = {}
for c in convs:
    email = user_map.get(c['account']['uuid'])
    if not email: continue
    d = c['created_at'][:10]
    if email not in last_active or d > last_active[email]:
        last_active[email] = d

print("\nLast active (chat):")
for email, d in sorted(last_active.items(), key=lambda x: x[1], reverse=True):
    print(f"  {email}: {d}")
```

> **Note on last active dates:** The script captures last chat activity. For developers who use Claude Code (terminal/IDE), manually set their last active date to the end of the month they last used Claude Code — they may have low chat counts but high code lines.

---

## Step 3 — Update the Dashboard HTML

Open `index.html` in any text editor. Find the `// ── DATA` section in the `<script>` block and update:

### 3a. MEMBERS array — one entry per team member
```javascript
var MEMBERS = [
  {name:'Full Name', email:'user@org.com', ini:'AB', role:'User', tier:'Premium'},
  // role: 'User' | 'Owner' | 'Primary Owner'
  // tier: 'Premium' | 'Standard'
  // ini: 2-letter initials for avatar
];
```
> **Order matters** — all other arrays (CONVS, MSGS, CODE, LAST) must have values in the same index order as MEMBERS.

### 3b. CONVS — conversations per user per month
```javascript
var CONVS = {
  'all':     [302, 257, ...],   // all-time totals
  '2026-04': [229,  37, ...],   // April
  '2026-05': [ 69, 207, ...],   // May
  '2026-06': [  4,  13, ...],   // June (partial if mid-month)
};
```

### 3c. MSGS — human messages per user per month
```javascript
var MSGS = {
  'all':     [1771, 1107, ...],
  '2026-04': [1423,  226, ...],
  '2026-05': [ 343,  833, ...],
  '2026-06': [   5,   48, ...],
};
```

### 3d. CODE — Claude Code lines accepted (one month only, 0 if not a code user)
```javascript
var CODE = [0, 105082, 9320, 57984, 4769, 45798, 239916, 0, 0, 165880];
```

### 3e. LAST — last active date per user (chat + Claude Code combined)
```javascript
var LAST = ['2026-06-03', '2026-06-04', ...];
```
> For Claude Code-heavy users with low chat activity, set this to the last day of the month they last used Claude Code.

### 3f. MONTHLY — total convs, msgs, active users per month
```javascript
var MONTHLY = {
  '2026-04': {convs:608,  msgs:5398, users:9},
  '2026-05': {convs:532,  msgs:3322, users:9},
  '2026-06': {convs:49,   msgs:248,  users:7},
};
```

### 3g. DAILY — conversation counts per day
```javascript
var DAILY = {
  '2026-04': { labels:['Apr 02', ...], data:[5, 9, ...] },
  '2026-05': { labels:['May 05', ...], data:[18,23, ...] },
  '2026-06': { labels:['Jun 01', ...], data:[20,13, ...] },
  'all':     { labels:['Apr 02', ...], data:[5, 20, ...] },  // sampled key dates
};
```

### 3h. META — month labels and subtitles
```javascript
var META = {
  'all':     {label:'All Time',   sub:'Apr 2 – Jun 4, 2026 · All Months', tag:'63-day snapshot'},
  '2026-04': {label:'April 2026', sub:'April 1 – 30, 2026',               tag:'Monthly view'},
  '2026-05': {label:'May 2026',   sub:'May 1 – 31, 2026',                 tag:'Monthly view'},
  '2026-06': {label:'June 2026',  sub:'June 1 – 4, 2026',                 tag:'Partial month'},
};
```

### 3i. Update the HTML header
Find these lines near the top of the `<body>` and update for your org:
```html
<div>Your Organisation Name</div>
<div>Your Name, Your Role</div>
<div>Report Date</div>
<div>X members | Plan name</div>
```

Also update the month keys throughout (`2026-04` → your months), and the filter buttons:
```html
<button onclick="go('2026-04')">April 2026</button>
```

---

## Step 4 — Host It (Optional)

The file works locally — just open `index.html` in Chrome/Edge. To give colleagues a shareable URL:

1. Create a free GitHub account at github.com
2. Install GitHub CLI: `winget install GitHub.cli` then `gh auth login`
3. Create a repo and enable Pages:
```bash
mkdir my-dashboard && cd my-dashboard
git init
cp path/to/index.html .
git add . && git commit -m "Add dashboard"
gh repo create my-claude-dashboard --public
git remote add origin https://github.com/YOUR-USERNAME/my-claude-dashboard.git
git branch -M main && git push -u origin main
git checkout -b gh-pages && git push origin gh-pages
```
4. Go to your repo → Settings → Pages → set source to `gh-pages` branch
5. Your URL: `https://YOUR-USERNAME.github.io/my-claude-dashboard/`

---

## Dashboard Sections Explained

| Section | What It Shows | How Calculated |
|---|---|---|
| **Overall Performance** | 5 KPI cards — conversations, messages, AI responses, active/inactive members | Summed from conversations.json for selected period |
| **Inactive Members** | Who hasn't been active recently | 0 convs in period AND last active > 14 days ago |
| **Seat Utilisation** | Active vs inactive seats (doughnut) | Active = any conversation in period |
| **Claude Code Highlights** | Lines accepted, accept rate, top contributor | From Claude Code CSV export (May only) |
| **Daily Trend** | Conversations per day for selected month | Daily count from conversations.json |
| **Monthly Comparison** | All months side-by-side (always visible) | Fixed — not affected by month filter |
| **Conversations per User** | Per-member bar chart, sorted by activity | Updates with month filter |
| **Code Lines per User** | Developer productivity via Claude Code | From CSV — May only, fixed |
| **Member Table** | Full breakdown with rank, convs, messages, avg depth, code, status | Dynamically sorted by selected month |
| **Key Insights** | 3 narrative cards — adoption, ROI, opportunities | Static analysis written at report time |

### Inactive Members Logic
A member is marked inactive when:
- They had **0 conversations** in the selected period, **AND**
- Their **last activity** (chat or Claude Code combined) was **more than 14 days ago**

This prevents recently-active developers (who use Claude Code more than chat) from being wrongly flagged.

---

## Monthly Refresh Checklist

Each month, to update the dashboard:

- [ ] Export new ZIP from claude.ai → Settings → Account → Privacy
- [ ] Export new Members CSV
- [ ] Export new Claude Code CSV for the new month
- [ ] Run the Python script to get new conversation/message counts
- [ ] Add the new month to `CONVS`, `MSGS`, `MONTHLY`, `DAILY`, `META`
- [ ] Update `LAST` dates (especially for Claude Code users)
- [ ] Update `REPORT_DATE` in the JS: `var REPORT_DATE = new Date('YYYY-MM-DD');`
- [ ] Update report date in the HTML header
- [ ] Add new filter button for the new month
- [ ] Push to GitHub → live in ~30 seconds

---

*Built with Claude Code · Anthropic · June 2026*
