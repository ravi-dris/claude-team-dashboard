# Claude AI Team Usage Dashboard — Replication Guide

> Built for Dhwani Rural Information Systems | June 2026
> For questions: ravi.jangir@dhwaniris.com
> Live dashboard: https://ravi-dris.github.io/claude-team-dashboard/
> Source code: https://github.com/ravi-dris/claude-team-dashboard

---

## What This Is

A **self-contained HTML dashboard** for reporting Claude AI usage to leadership. Covers multiple teams with a dual filter (Team + Month). Opens in any browser — no server, no install.

**Built for:** Dhwani RIS — Product Team (10 members) and Services Team (11 members)  
**Data period:** April 2 – June 4, 2026  
**Total members:** 21 across both teams

---

## Dashboard Sections (in order)

| # | Section | What It Shows |
|---|---|---|
| 1 | Overall Performance | 5 KPI cards: Conversations, Messages, AI Responses, Active Members, Inactive Members |
| 2 | Key Insights | 6 org-level insight cards + 21 per-member spotlight cards |
| 3 | Claude Code Highlights | Lines accepted, accept rate, top contributor, active code users — updates with filters |
| 4 | Activity Trends | Daily conversations chart + Monthly comparison (Product vs Services) |
| 5 | User Engagement | Conversations per user + Code lines per user — both filterable |
| 6 | Member Activity Breakdown | Sortable table — click any column header to sort |
| 7 | Inactive Members + Seat Utilisation | Who's inactive + doughnut chart |

---

## Filters

**Row 1 — Team:** Both Teams · Product Team · Services Team  
**Row 2 — Month:** All Time · April · May · June

Every section updates when either filter changes. Charts colour-code by team: **blue = Product, orange = Services**.

---

## Active vs Inactive Definition

A member is **Active** if ANY of:
1. Had **conversations > 0** in the selected period
2. Had **code lines > 0** in the selected period (Claude Code users)
3. **Last activity ≤ 14 days ago** (covers recently active members in partial months)

A member is **Inactive** only if ALL three conditions fail — 0 convs, 0 code, AND last active > 14 days ago.

> This is why Ankit Jangir shows Active in June (0 chats but 60,324 code lines), and why Abhijit Nair shows Active (165K code lines, 3 chats). The same rule applies to the KPI card, the Inactive Members panel, and the Seat Utilisation doughnut — they are all consistent.

---

## Step 1 — Export Data from Claude.ai

You need **Team/Enterprise admin** access on claude.ai.

### A. Full data export (ZIP)
Settings → Account → Privacy → **Export Data**  
Contains: `conversations.json`, `users.json`, `projects/`

### B. Members CSV
Settings → Organization → Members → **Export**  
Columns: Name, Email, Role, Status, Seat Tier

### C. Claude Code CSVs (one per month)
Settings → Organization → Claude Code → select month → **Export**  
Columns: User, Lines this Month  
Export one CSV per month you want to cover.

---

## Step 2 — Extract the Numbers

Run this Python script on each team's ZIP:

```python
import zipfile, json
from collections import defaultdict

ZIP = 'your-export.zip'

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
    monthly[email][month]['msgs'] += sum(1 for m in c.get('chat_messages',[]) if m['sender']=='human')

for email in sorted(monthly):
    for month in sorted(monthly[email]):
        d = monthly[email][month]
        print(f"{email} | {month} | convs={d['convs']} | msgs={d['msgs']}")

# Last active date per user (chat only)
last_active = {}
for c in convs:
    email = user_map.get(c['account']['uuid'])
    if not email: continue
    d = c['created_at'][:10]
    if email not in last_active or d > last_active[email]:
        last_active[email] = d

print("\nLast active:")
for email, d in sorted(last_active.items(), key=lambda x: x[1], reverse=True):
    print(f"  {email}: {d}")
```

> **Important for Claude Code users:** The script only captures last *chat* activity. For developers who primarily use Claude Code (terminal/IDE), manually set their `last` date to the last day of the month they had code output. Example: if Abhijit had 165K lines in May, set his last date to `2026-05-31`.

---

## Step 3 — Update the Dashboard HTML

Open `index.html`. There are **two team data objects** at the top of the `<script>` block: `var P = {...}` (Product) and `var S = {...}` (Services). Update both.

Each team object has these fields:

### members — one entry per person
```javascript
{name:'Full Name', email:'user@org.com', ini:'AB', role:'User', tier:'Premium', team:'product'}
// role: 'User' | 'Owner' | 'Primary Owner'
// tier: 'Premium' | 'Standard'
// team: 'product' or 'services'
// ini: 2-letter initials shown in avatar
```
> **Index order matters** — all arrays (convs, msgs, code, last) must be in the same order as members.

### convs — conversations per user per month
```javascript
convs: {
  'all':     [302, 257, ...],   // all-time totals, one per member
  '2026-04': [229,  37, ...],   // April
  '2026-05': [ 69, 207, ...],   // May
  '2026-06': [  4,  13, ...],   // June (partial)
}
```

### msgs — human messages per user per month
```javascript
msgs: {
  'all':     [1771, 1107, ...],
  '2026-04': [1423,  226, ...],
  '2026-05': [ 343,  833, ...],
  '2026-06': [   5,   48, ...],
}
```

### code — Claude Code lines per user per month
```javascript
code: {
  'all':     [0, 134460, 9608, ...],   // sum across all months
  '2026-04': [0,  28417,  288, ...],
  '2026-05': [0, 105082, 9320, ...],
  '2026-06': [0,    961,    0, ...],
}
```

### last — last active date per user (chat + Claude Code combined)
```javascript
last: ['2026-06-04', '2026-05-01', '2026-06-04', ...]
```

### monthly — totals per month
```javascript
monthly: {
  '2026-04': {c:608, m:5398, u:9},   // c=convs, m=msgs, u=active users
  '2026-05': {c:532, m:3322, u:9},
  '2026-06': {c:49,  m:248,  u:7},
}
```

### daily — conversations per day
```javascript
daily: {
  '2026-04': { L:['Apr 02','Apr 03',...], D:[5,9,...] },
  '2026-05': { L:['May 05','May 06',...], D:[18,23,...] },
  '2026-06': { L:['Jun 01','Jun 02',...], D:[20,13,...] },
  'all':     { L:[sampled key dates],     D:[sampled counts] }
}
```

### Also update these global objects:

**META** — month labels:
```javascript
var META = {
  'all':     {label:'All Time',   sub:'Apr 2 – Jun 4, 2026'},
  '2026-04': {label:'April 2026', sub:'April 2026'},
  '2026-05': {label:'May 2026',   sub:'May 2026'},
  '2026-06': {label:'June 2026',  sub:'June 1–4, 2026'},
};
```

**CODE_HL** — Claude Code highlight cards per team per month:
```javascript
var CODE_HL = {
  both:     { all:{lines,rate,top,topWho,users}, '2026-04':{...}, ... },
  product:  { all:{...}, '2026-04':{...}, ... },
  services: { all:{...}, '2026-04':{...}, ... },
};
```

**REPORT_DATE** — used for "days since last active" calculation:
```javascript
var REPORT_DATE = new Date('2026-06-05');  // day after report date
```

### Update HTML header
```html
<div>Your Organisation Name</div>
<div>Prepared for: Your Name, Your Role</div>
<div>Report Date: June 4, 2026</div>
```

### Add/update filter buttons for months:
```html
<button onclick="setMonth('2026-07')">July 2026</button>
```

---

## Step 4 — Per-Member Spotlight Cards

The Key Insights section includes a **Member Spotlight** — one card per person with their actual usage pattern and what they use Claude for. These are based on real conversation topic names from `conversations.json`.

To update them: extract conversation names from the ZIP and rewrite the static HTML cards in the `<!-- Per-member spotlight -->` section. Each card follows this structure:

```html
<div style="...border-left:3px solid [colour]...">
  <!-- Header: avatar + name + team badge -->
  <!-- Stats row: convs, msgs, avg, code -->
  <!-- Usage archetype badge -->
  <!-- 2-3 sentence description of actual usage -->
</div>
```

Colour by archetype:
- `var(--blue)` = Deep Collaborator (high avg msgs/conv)
- `var(--green)` = Volume User (high conversation count)
- `var(--purple)` = Silent Coder (low convs, high code lines)
- `var(--orange)` = Specialist (domain-specific use)
- `var(--red)` = Needs Activation (low usage, inactive)
- `var(--muted)` = Growing User (increasing trend)

---

## Step 5 — Host on GitHub Pages

```bash
# One-time setup
gh repo create my-claude-dashboard --public
cd my-dashboard-folder
git init
git remote add origin https://github.com/YOUR-USERNAME/my-claude-dashboard.git

# Each update
git add index.html
git commit -m "Update dashboard — July 2026"
git push origin gh-pages
# Live at: https://YOUR-USERNAME.github.io/my-claude-dashboard/
```

First push: also run `git checkout -b gh-pages` before pushing.

---

## Monthly Refresh Checklist

- [ ] Export new ZIP (both teams if applicable) from claude.ai admin
- [ ] Export new Members CSVs
- [ ] Export Claude Code CSVs for the new month (both teams)
- [ ] Run Python script on both ZIPs → get new convs/msgs/last dates
- [ ] Add new month key to `convs`, `msgs`, `code`, `monthly`, `daily` in both P and S objects
- [ ] Update `last` dates for Claude Code-active members
- [ ] Update `CODE_HL` with new month's code totals
- [ ] Add filter button for new month
- [ ] Update `META` with new month label
- [ ] Update `REPORT_DATE` and HTML header date
- [ ] Update per-member spotlight cards if usage patterns changed
- [ ] `git add index.html && git commit && git push origin gh-pages`

---

## Excluded Users (not in members list)

These appeared in Claude Code CSV exports but were not in the members CSV. Their lines are excluded from totals. Investigate and add them if they are current team members.

| User | Team | Months with activity |
|---|---|---|
| om.prakash@dhwaniris.com | Product | April (14,681 lines) |
| fathima.nihala@dhwaniris.com | Product | April (601 lines) |
| shlok@dhwaniris.com | Services | April (39,541 lines) |
| vivek.kumar@dhwaniris.com | Services | April (1,453 lines), May (2,315 lines) |

---

*Built with Claude Code · Anthropic · June 2026*
