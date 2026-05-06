# Calico AI — Instagram Outbound Process

## Overview

Automated Instagram DM outreach to real estate photographers and media companies. The agent works city by city, scraping Instagram search results, qualifying profiles, checking for prior contact, selecting the best-fit message, sending the DM, and logging everything to a CSV tracker.

---

## Inputs Required

| Input | Example | Notes |
|-------|---------|-------|
| **City** | `Kansas City` | Used to construct search queries |
| **Hashtag / search query** | `#realestatephotography #kansascityphotographer` | Agent constructs this from city + niche |

---

## Search Query Construction

Base URL:
```
https://www.instagram.com/explore/search/keyword/?q=<query>
```

Query strategy — combine niche + city signal:
- Primary: `#realestatephotography` + city name variations
- Secondary: `#[city]photographer`, `#[city]realestate`, `#[city]listing`
- Fallback: `real estate photography [city]`

Example for Kansas City:
```
https://www.instagram.com/explore/search/keyword/?q=%23realestatephotographykansascity
https://www.instagram.com/explore/search/keyword/?q=%23kansascityphotographer
https://www.instagram.com/explore/search/keyword/?q=%23kansascityrealestate
```

Collect unique post URLs from the search results page. Extract profile handles from each post before visiting individual profiles.

---

## Per-Profile Workflow

For each profile found in search results, execute this sequence in order:

### Step 1 — Early exit: check the CSV tracker

Before loading the profile, look up the username in `outreach-log.csv`.

- If a row exists with that username → **skip**, move to next profile
- If no row → continue to Step 2

This avoids loading a profile page we've already processed.

### Step 2 — Load the profile and qualify

Navigate to `https://www.instagram.com/<username>/`

Read the profile:
- Bio text
- Account name / handle
- Recent post captions and hashtags
- Follower count / following count
- Any business category label Instagram shows

**Qualification criteria** (based on `calico-ai-customer-research.md`):

| Signal | Qualifies | Does Not Qualify |
|--------|-----------|-----------------|
| Bio mentions real estate, listing, property photography | ✅ | General lifestyle, portrait, wedding only |
| Posts show interior/exterior listing photos | ✅ | No property content visible |
| Account type | Solo photographer, small studio, media company | Agent/broker, stager, unrelated business |
| Follower range | 200 – 50,000 | <200 (likely inactive) or >100K (likely brand account) |
| Location signal | Target city or nearby market | Clearly different market with no overlap |
| Activity | Posted within last 90 days | Dormant account |

**ICP segment to assign** (feeds Step 5):
- `solo` — solo freelance RE photographer
- `studio` — small RE photography studio (2-5 people)
- `media_company` — RE media company (team, bundled services, drone, video mentioned)

If the profile does **not** qualify:
- Log a row to `outreach-log.csv` with `status = disqualified` and `disqualify_reason`
- Move to next profile

### Step 3 — Check Instagram DM inbox for prior contact

Navigate to the Instagram DM inbox (`https://www.instagram.com/direct/inbox/`) and search for the username.

- If a conversation exists → **skip**, log `status = already_contacted`, move to next profile
- If no conversation → continue to Step 4

This is the authoritative check. The CSV is a local cache; the inbox is the source of truth.

### Step 4 — Select the best-fit message

Use the ICP segment from Step 2 and the offer mapping table from `calico-ai-offer-constructs.md`:

| ICP Segment | Primary Offer | Script to Use |
|-------------|--------------|---------------|
| `solo` | Offer 1: "The $300 You're Not Charging" | Blitz Script 1 or targeted Version A/B |
| `studio` | Offer 2: "Add Video to Your Menu This Week" | Blitz Script 1 or Offer 2 Version A/B |
| `media_company` | Offer 4: "Scale Without Hiring" | Blitz Script 3 or Offer 4 Version A/B |

Secondary signals that override offer selection:
- Profile posts show heavy editing work → layer in Offer 3 (editing pain angle)
- Profile bio or location mentions NY or CA → open with Offer 6 (compliance angle)
- Profile has drone/video content already → use Offer 4 or 5 (scale or brokerage angle)

Replace `{{first_name}}` with the first name from the profile name field, or the handle if name is not available.

### Step 5 — Send the DM

Navigate to the profile and open a new DM conversation.

Send the selected message exactly as written (with `{{first_name}}` substituted).

Verify the message was sent by confirming it appears in the conversation thread.

### Step 6 — Log to CSV

Append a row to `outreach-log.csv` with:

| Column | Value |
|--------|-------|
| `timestamp` | ISO 8601 datetime, e.g. `2026-04-27T14:32:00` |
| `username` | Instagram handle (no @) |
| `profile_name` | Display name from profile |
| `icp_segment` | `solo`, `studio`, or `media_company` |
| `offer_used` | Offer number and name, e.g. `Offer 1 - The $300 You're Not Charging` |
| `message_sent` | Full text of the message sent |
| `status` | `sent`, `disqualified`, `already_contacted`, or `error` |
| `disqualify_reason` | If disqualified, brief reason; else empty |
| `city` | Target city for this run |

### Step 7 — Wait, then continue

Wait a **randomized delay of 8–25 minutes** before processing the next profile.

This mimics human pacing and reduces the risk of rate limiting or account flags.

---

## CSV Tracker

**File:** `outreach-log.csv`  
**Location:** Project root (`/Users/lucas/Desktop/projects/real-estate-photographer-ig-outbound/outreach-log.csv`)

**Schema:**
```
timestamp,username,profile_name,icp_segment,offer_used,message_sent,status,disqualify_reason,city
```

If the file does not exist, create it with the header row before writing the first entry.

---

## Full Loop Pseudocode

```
configure(city)
queries = build_search_queries(city)

for query in queries:
    profiles = scrape_instagram_search(query)
    
    for username in profiles:
        # Step 1 — fast exit
        if username in csv_tracker:
            continue
        
        # Step 2 — load and qualify
        profile = load_instagram_profile(username)
        icp_segment = qualify_profile(profile)
        
        if not icp_segment:
            log_csv(username, status="disqualified", ...)
            continue
        
        # Step 3 — check inbox
        if conversation_exists_in_inbox(username):
            log_csv(username, status="already_contacted", ...)
            continue
        
        # Step 4 — pick message
        message = select_message(icp_segment, profile)
        
        # Step 5 — send DM
        send_dm(username, message)
        
        # Step 6 — log
        log_csv(username, status="sent", message=message, ...)
        
        # Step 7 — wait
        sleep(random(8, 25) minutes)
```

---

## Safety and Rate Limiting Notes

- **Do not send more than ~20-30 DMs per session.** Instagram flags accounts for high-volume identical messages.
- **Vary message wording slightly** between sends where possible — use alternate script versions (A vs. B).
- **Stop immediately** if Instagram shows a "Try Again Later" or "Action Blocked" notice. Log the blocked state and surface it.
- **Never send to the same username twice.** The CSV check in Step 1 and the inbox check in Step 3 are both guards.
- If the Instagram session appears logged out or access-restricted, stop and ask the user to re-authenticate before continuing.

---

## Files in This Project

| File | Purpose |
|------|---------|
| `calico-ai-customer-research.md` | ICP definitions, pain points, qualification criteria |
| `calico-ai-offer-constructs.md` | All offer framings and DM scripts |
| `outbound-process.md` | This file — full process spec |
| `outreach-log.csv` | Tracker — created on first run |
