---
name: ig-outbound
description: |
  Run the Calico AI Instagram outbound process for a given city. Scrapes Instagram search results for real estate photographers, qualifies each profile against ICP criteria, checks for prior contact, selects the best-fit DM script, sends the message, logs to CSV, and waits a randomized 8-25 minutes before the next lead. Trigger by providing a city name (e.g. "Austin, Texas"). Source of truth: /Users/lucas/Desktop/projects/real-estate-photographer-ig-outbound/outbound-process.md
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
---

# Instagram Outbound Skill — Calico AI

## Role

You are running the Calico AI Instagram outreach process. Your job is to find real estate photographers in the given city, qualify them against ICP criteria, avoid double-contacting anyone, select the right DM script, send it via the browser harness, log the result, and pace yourself to avoid triggering Instagram's rate limits.

**Project root:** `/Users/lucas/Desktop/projects/real-estate-photographer-ig-outbound/`
**CSV tracker:** `/Users/lucas/Desktop/projects/real-estate-photographer-ig-outbound/outreach-log.csv`
**Process spec:** `outbound-process.md` (already read — use this skill as the authoritative summary)
**Offer scripts:** `calico-ai-offer-constructs.md`
**ICP research:** `calico-ai-customer-research.md`

---

## Safety Rules (non-negotiable)

1. **Never send more than 20-30 DMs per session.** Keep a running count. Stop at 20 unless the user explicitly raises the cap.
2. **Never contact the same username twice.** Check CSV first (Step 1), then the actual DM inbox (Step 3).
3. **Stop immediately** if Instagram shows "Try Again Later", "Action Blocked", or any rate-limit warning. Log the blocked state and surface it to the user.
4. **If Instagram appears logged out or access-restricted, stop and ask the user to re-authenticate.** Do not attempt to log in.
5. **Randomized 8–25 minute wait between every DM send.** This is mandatory — not optional.
6. **Never skip the inbox check** (Step 3). The CSV is a local cache; the inbox is the source of truth.

---

## Step 0 — Setup

### 0a. Read the city input
The user provides a city. Parse city name and state/region from it.

### 0b. Ensure CSV tracker exists
```python
import csv, os
CSV_PATH = "/Users/lucas/Desktop/projects/real-estate-photographer-ig-outbound/outreach-log.csv"
HEADERS = ["timestamp","username","profile_name","icp_segment","offer_used","message_sent","status","disqualify_reason","city"]
if not os.path.exists(CSV_PATH):
    with open(CSV_PATH, "w", newline="") as f:
        csv.writer(f).writerow(HEADERS)
```

### 0c. Load already-contacted usernames into memory
```python
import csv
contacted = set()
with open(CSV_PATH, newline="") as f:
    for row in csv.DictReader(f):
        contacted.add(row["username"].lower())
```

---

## Step 1 — Build search queries

For a city like "Austin, Texas":
- Slug variants: `austin`, `austintx`, `austintexas`
- Queries to run (in order):
  1. `#realestatephotographyaustin`
  2. `#austinrealestatephotographer`
  3. `#austinrealestatephotography`
  4. `#austinphotographer`
  5. `#austinrealestatelisting`
  6. `real estate photography austin texas`

Construct Instagram search URLs:
```
https://www.instagram.com/explore/search/keyword/?q=<url-encoded-query>
```

---

## Step 2 — Scrape search results via browser harness

Use the browser harness to load each search URL and extract profile handles from post thumbnails. Do **not** navigate in the user's active tab — always use `new_tab()`.

```bash
browser-harness <<'PY'
new_tab("https://www.instagram.com/explore/search/keyword/?q=%23realestatephotographyaustin")
wait_for_load()
import time; time.sleep(3)  # let JS render
capture_screenshot("/tmp/ig_search.png")
# Extract post links and handles via JS
handles = js("""
  Array.from(document.querySelectorAll('a[href*="/"]'))
    .map(a => a.href.match(/instagram\\.com\\/([^/?#]+)/)?.[1])
    .filter(h => h && !['explore','p','reel','stories','direct','accounts'].includes(h))
    .filter((v,i,a) => a.indexOf(v) === i)
""")
print(handles)
PY
```

Collect all unique handles across all queries. Deduplicate.

---

## Step 3 — Per-profile loop

For each handle collected, execute the sequence below. Maintain a `dm_count` counter. Stop the loop if `dm_count >= 20`.

### 3a — CSV early exit (Step 1 of process spec)
```python
if handle.lower() in contacted:
    print(f"SKIP {handle} — already in CSV")
    continue
```

### 3b — Load and qualify the profile (Step 2 of process spec)

Navigate to `https://www.instagram.com/<handle>/`:

```bash
browser-harness <<'PY'
new_tab(f"https://www.instagram.com/{handle}/")
wait_for_load()
import time; time.sleep(2)
capture_screenshot("/tmp/ig_profile.png")
profile_data = js("""
  ({
    bio: document.querySelector('span.-vDIg, div._aa_c, header section div')?.innerText || '',
    name: document.querySelector('h1, header h2')?.innerText || '',
    meta: document.querySelector('header section ul')?.innerText || '',
    posts_preview: Array.from(document.querySelectorAll('article img, ._aagv img')).slice(0,9).map(i=>i.alt||'')
  })
""")
print(profile_data)
PY
```

Screenshot and read the profile carefully. Evaluate against ALL qualification criteria:

| Signal | Qualifies | Does Not Qualify |
|--------|-----------|-----------------|
| Bio mentions real estate, listing, property photography | ✅ | General lifestyle, portrait, wedding only |
| Posts show interior/exterior listing photos | ✅ | No property content visible |
| Account type | Solo photographer, small studio, media company | Agent/broker, stager, unrelated business |
| Follower range | 200–50,000 | <200 (likely inactive) or >100K (likely brand account) |
| Location signal | Target city or nearby market | Clearly different market |
| Activity | Posted within last 90 days | Dormant |

**Assign ICP segment:**
- `solo` — solo freelance RE photographer
- `studio` — small RE photography studio (2-5 people)
- `media_company` — RE media company (team, bundled services, drone, video)

If not qualified → log to CSV with `status=disqualified` and reason → continue to next handle.

### 3c — Check DM inbox for prior contact (Step 3 of process spec)

Navigate to `https://www.instagram.com/direct/inbox/` and search for the handle.

```bash
browser-harness <<'PY'
new_tab("https://www.instagram.com/direct/inbox/")
wait_for_load()
import time; time.sleep(2)
capture_screenshot("/tmp/ig_inbox.png")
# Search for the handle in the search box
search_box = js("document.querySelector('input[placeholder*=\"Search\"], input[name=\"queryBox\"]')")
# click the search input, type the handle, check if a conversation appears
PY
```

Take a screenshot after typing the handle. If a conversation thread appears → log `status=already_contacted` → continue.

If no conversation found → proceed to message selection.

### 3d — Select message (Step 4 of process spec)

**Primary mapping:**

| ICP | Offer | Script |
|-----|-------|--------|
| `solo` | Offer 1: "The $300 You're Not Charging" | Blitz Script 1 or Offer 1 Version A/B |
| `studio` | Offer 2: "Add Video to Your Menu This Week" | Blitz Script 1 or Offer 2 Version A/B |
| `media_company` | Offer 4: "Scale Without Hiring" | Blitz Script 3 or Offer 4 Version A/B |

**Secondary signal overrides:**
- Heavy editing content visible → layer in Offer 3 (editing pain angle)
- Profile bio/location mentions NY or CA → use Offer 6 (compliance angle) as opener
- Drone/video content already present → use Offer 4 or 5

**Alternate between A and B versions** across sends to vary wording.

Extract first name from the display name field. Fall back to the handle if no name is available.

Substitute `{{first_name}}` in the chosen script.

### 3e — Send the DM (Step 5 of process spec)

Navigate to the profile and open a new DM:

```bash
browser-harness <<'PY'
new_tab(f"https://www.instagram.com/{handle}/")
wait_for_load()
import time; time.sleep(2)
capture_screenshot("/tmp/before_dm.png")
# Find and click the Message button
# After clicking, wait for DM modal to open
# Type the message, then send
# Take a screenshot to confirm the message appears in the thread
PY
```

**Critically: verify the message appears sent** (screenshot the thread after sending). If anything looks wrong (action blocked, error), stop immediately and report to the user.

### 3f — Log to CSV (Step 6 of process spec)

```python
import csv, datetime
with open(CSV_PATH, "a", newline="") as f:
    csv.writer(f).writerow([
        datetime.datetime.now().isoformat(timespec='seconds'),
        handle,
        profile_name,
        icp_segment,
        offer_name,          # e.g. "Offer 1 - The $300 You're Not Charging"
        message_sent,
        "sent",
        "",                  # disqualify_reason empty on success
        city,
    ])
contacted.add(handle.lower())
dm_count += 1
```

### 3g — Wait randomized delay (Step 7 of process spec)

```python
import random, time
delay_minutes = random.uniform(8, 25)
delay_seconds = delay_minutes * 60
print(f"Waiting {delay_minutes:.1f} minutes before next profile...")
time.sleep(delay_seconds)
```

**This wait is mandatory.** Do not skip it or reduce it. It mimics human pacing and protects the account.

---

## Error Handling

| Situation | Action |
|-----------|--------|
| Instagram "Try Again Later" or "Action Blocked" | Stop immediately. Log current state. Surface to user. |
| Instagram login wall / logged out | Stop. Ask user to re-authenticate in their browser. |
| Profile loads but no bio/content visible | Screenshot and skip with `status=disqualified, disqualify_reason=profile_not_readable` |
| DM send appears to fail / no confirmation | Log `status=error`. Do NOT retry. Continue to next profile after normal wait. |
| `dm_count` reaches 20 | Stop the loop. Report final counts to user. Ask if they want to continue in a new session. |

---

## Session Summary

After the loop ends (or is stopped), print a summary:
- Total profiles evaluated
- DMs sent this session
- Disqualified count (with top reasons)
- Already-contacted skips
- Any errors or blocks encountered
- Current `outreach-log.csv` path

---

## Execution Instructions

When the user gives you a city name:

1. Read this skill file (already done).
2. Read `calico-ai-offer-constructs.md` to load all DM scripts into context.
3. Read `outreach-log.csv` (if it exists) to load prior contacts.
4. Build search queries for the city.
5. Open the browser harness. Navigate to the first search URL using `new_tab()`.
6. Extract handles. Deduplicate. Build your queue.
7. Execute the per-profile loop, following Steps 3a–3g in strict order.
8. Never skip a safety check. Never skip the delay.
9. Report progress to the user at each DM send (handle, ICP segment, offer used, delay until next).
10. Report the session summary when done.
