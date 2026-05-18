---
name: link-building-outreach
description: >
  Runs a full link-building outreach campaign for Final Round AI (or any domain).
  Use this skill whenever the user says "run outreach", "find backlink opportunities",
  "competitor backlink analysis", "who links to our competitors", "link building",
  "find sites to pitch", or "send outreach emails". The skill pulls Ahrefs referring-domain
  data for competitors, finds gap domains (sites linking to competitors but not to us),
  researches each site for a content angle, finds contact emails, writes personalised
  pitches, and sends them via Outlook — all in one automated flow.
---

# Link-Building Outreach Skill

## What this skill does

End-to-end link-building pipeline:

1. Pull referring domains for all competitor sites via Ahrefs API v3
2. Pull referring domains for the target site (e.g. finalroundai.com)
3. Find **gap domains** — sites linking to competitors but NOT to the target site
4. Filter for quality (DR ≥ 25, dofollow, not spam, not generic/social)
5. Research each gap domain's content angle (what blog topics they cover)
6. Find a contact email for each domain
7. Write a personalised outreach email per domain
8. Send all emails via Outlook MCP from the specified sender address
9. Output a full summary report

---

## Inputs to collect before starting

Ask the user for any that are missing:

| Input | Default / Example |
|---|---|
| `AHREFS_KEY` | Ahrefs API v3 bearer token |
| `TARGET_DOMAIN` | `finalroundai.com` |
| `COMPETITOR_DOMAINS` | List of competitor domains (see defaults below) |
| `SENDER_EMAIL` | e.g. `alex@finalroundai.net` |
| `MIN_DR` | `25` (minimum domain rating to include) |
| `MAX_EMAILS` | `20` (safety cap — don't send more than this without user confirmation) |

**Default competitor domains for Final Round AI:**
```
interviewsidekick.com
interviewcoder.co
parakeet-ai.com
vervecopilot.com
senseicopilot.com
interview-assistant-ai.com
lockedinai.com
beyz.ai
linkjob.ai
```

---

## Phase 1 — Pull Ahrefs referring domains

Use `curl` via the Bash tool. The Ahrefs API v3 returns max 100 referring domains per request on most plans — that's fine, we focus on the highest-DR sites.

### Endpoint
```
GET https://api.ahrefs.com/v3/site-explorer/refdomains
```

### Required parameters
| Param | Value |
|---|---|
| `target` | domain to analyse (no `https://`) |
| `select` | `domain_rating,domain,traffic_domain,dofollow_links,is_spam` |
| `limit` | `100` |
| `order_by` | `domain_rating:desc` |

### Auth header
```
Authorization: Bearer <AHREFS_KEY>
```

### Bash helper — pull one domain
```bash
fetch_refdomains() {
  local label=$1   # short name for the file, e.g. "sidekick"
  local target=$2  # e.g. "interviewsidekick.com"
  curl -s \
    "https://api.ahrefs.com/v3/site-explorer/refdomains?target=${target}&select=domain_rating,domain,traffic_domain,dofollow_links,is_spam&limit=100&order_by=domain_rating:desc" \
    -H "Authorization: Bearer ${AHREFS_KEY}" \
    > "/tmp/outreach/${label}.json"
  echo "Fetched ${label}: $(jq '.refdomains | length' /tmp/outreach/${label}.json) domains"
}
```

### Run in parallel
```bash
mkdir -p /tmp/outreach

# All competitors + own domain simultaneously
fetch_refdomains "own"      "finalroundai.com" &
fetch_refdomains "sidekick" "interviewsidekick.com" &
fetch_refdomains "coder"    "interviewcoder.co" &
fetch_refdomains "parakeet" "parakeet-ai.com" &
fetch_refdomains "verve"    "vervecopilot.com" &
fetch_refdomains "sensei"   "senseicopilot.com" &
fetch_refdomains "intasst"  "interview-assistant-ai.com" &
fetch_refdomains "lockedin" "lockedinai.com" &
fetch_refdomains "beyz"     "beyz.ai" &
fetch_refdomains "linkjob"  "linkjob.ai" &
wait
echo "All Ahrefs data fetched."
```

---

## Phase 2 — Build the gap list

Extract unique domains from each competitor file, then subtract the target site's own referring domains.

```bash
# Build FRAI's domain set
jq -r '.refdomains[].domain' /tmp/outreach/own.json | sort -u > /tmp/outreach/own_domains.txt

# Generic/social sites to always exclude
EXCLUDE="youtube.com google.com google.co.uk github.com wikipedia.org linkedin.com
         twitter.com x.com facebook.com instagram.com reddit.com tiktok.com
         apple.com microsoft.com amazon.com trustpilot.com g2.com capterra.com
         producthunt.com ycombinator.com web.archive.org"

# Competitor domains themselves (to avoid outreaching a competitor)
COMPETITORS="interviewsidekick.com interviewcoder.co parakeet-ai.com
             vervecopilot.com senseicopilot.com interview-assistant-ai.com
             lockedinai.com beyz.ai linkjob.ai finalroundai.com"

ALL_EXCLUDE="$EXCLUDE $COMPETITORS"

# Collect all competitor domains with metadata
for f in sidekick coder parakeet verve sensei intasst lockedin beyz linkjob; do
  jq -r --arg src "$f" \
    '.refdomains[] | select(.is_spam == false) |
     [$src, .domain, (.domain_rating|tostring), (.traffic_domain|tostring), (.dofollow_links|tostring)] | join("\t")' \
    /tmp/outreach/${f}.json
done | sort -t$'\t' -k3 -rn > /tmp/outreach/all_competitor_domains.tsv

# Filter: not already linking to FRAI, not excluded, DR >= MIN_DR
MIN_DR=25
while IFS=$'\t' read -r src domain dr traffic dofollow; do
  grep -qx "$domain" /tmp/outreach/own_domains.txt && continue
  skip=false
  for excl in $ALL_EXCLUDE; do
    [[ "$domain" == "$excl" || "$domain" == *".$excl" ]] && { skip=true; break; }
  done
  $skip && continue
  dr_int=$(echo "$dr" | cut -d. -f1)
  [[ $dr_int -ge $MIN_DR ]] && echo -e "$domain\t$dr\t$traffic\t$dofollow\t$src"
done < /tmp/outreach/all_competitor_domains.tsv \
  | sort -t$'\t' -k2 -rn \
  | awk -F'\t' '!seen[$1]++' \
  > /tmp/outreach/gap_domains.tsv

echo "Gap domains found: $(wc -l < /tmp/outreach/gap_domains.tsv)"
```

Read `gap_domains.tsv` and show the user a preview table (top 20 by DR). Columns: Domain | DR | Monthly Traffic | Dofollow Links | Found via Competitor.

---

## Phase 3 — Research each gap domain

For each domain in `gap_domains.tsv` (up to `MAX_EMAILS`), do the following research in parallel using WebSearch and WebFetch:

### 3a. Understand their content angle
Search: `site:<domain> interview OR "job search" OR "AI tools" OR "career advice"`

Look for:
- Do they publish listicles / "best tools" roundups?
- Do they cover job-search or interview prep content?
- Do they have a "Write for us" / guest post page?
- What's their primary audience (students, developers, HR professionals, job seekers)?

### 3b. Find the most relevant page on their site
If they have a roundup like "Best AI interview tools" or "Top interview prep resources", that's the ideal page to pitch adding a Final Round AI mention.

If they have a blog, find their most relevant recent post topic.

### 3c. Identify the pitch angle
Match the domain's content to one of these Final Round AI value props:
- **AI Interview Copilot** — real-time answer suggestions during live interviews
- **AI Mock Interview** — practice with realistic AI interviewers
- **AI Resume Builder** — tailored resume generation
- **Interview Question Bank** — 10,000+ company-specific questions
- **Coding Interview prep** — for technical roles

Store per domain: `domain | content_angle | relevant_page_url | pitch_angle`

---

## Phase 4 — Find contact email

For each domain, try these in order (stop at first success):

### Method 1: Search
```
WebSearch: "<domain> contact email editor OR "write for us" OR "submit" site:<domain>"
```

### Method 2: Common paths (try via WebFetch)
Try these URLs and look for an email address or contact form:
- `https://<domain>/contact`
- `https://<domain>/about`
- `https://<domain>/write-for-us`
- `https://<domain>/contribute`

### Method 3: Hunter.io pattern guess
If domain is e.g. `techblog.com`, try:
- `editor@techblog.com`
- `hello@techblog.com`
- `contact@techblog.com`

Record: `domain | email | confidence (found/guessed)`

If no email found at all, mark as `MANUAL_FOLLOWUP` — include in report but don't send.

---

## Phase 5 — Write personalised outreach emails

Write one email per domain. Each email must feel hand-written, not templated. Follow these rules:

### Email structure
```
Subject: [Short, specific, curiosity-driven — mention their site name or specific article]

Hi [Name if found, else "there"],

Opening line: Reference something specific about THEIR site or content
(e.g., "I came across your roundup of AI interview tools on [page] — really solid list.")

Value bridge: Connect their audience to what Final Round AI offers
(e.g., "Since you cover tools for job seekers, I thought your readers might find
Final Round AI useful — it gives real-time coaching during live interviews, which
a lot of candidates don't know exists.")

The ask: Make it low friction
Options (choose what fits the site):
  - "Would you be open to including it in your [specific article/roundup]?"
  - "Happy to write a short guest section or provide a quote for your [topic] post."
  - "We also have a data study on [relevant topic] that could make a good exclusive."

Close: Keep it easy to say yes
(e.g., "No worries if it's not a fit — just thought it was worth a shot!")

[Name]
Final Round AI
[alex@finalroundai.net]
[https://www.finalroundai.com]
```

### Personalisation checklist (every email must have):
- [ ] Reference to a specific page, post title, or content theme on their site
- [ ] One sentence about their audience (not about FRAI)
- [ ] The correct pitch angle for that domain's content type
- [ ] A single, clear, low-friction ask
- [ ] No attachments, no "please find enclosed", no corporate speak

### Tone
Casual, direct, human. Like an email from a person, not a marketing team. Around 150–200 words. No bullet points in the email body.

---

## Phase 6 — Send via Outlook MCP

Use the `mcp__outlook__send_email` tool.

Before sending:
1. Show the user a summary table: Domain | DR | Email | Subject | First line of email body
2. Ask: "Ready to send these [N] emails? Or would you like to review/edit any first?"
3. Wait for explicit confirmation.

After confirmation, send each email:
```
From: alex@finalroundai.net
To: <contact email>
Subject: <generated subject>
Body: <generated email body>
```

Log each send result (success / failed) as you go.

If `mcp__outlook__send_email` is not available, fall back to saving each email as a `.txt` draft in `/tmp/outreach/drafts/` and tell the user where to find them.

---

## Phase 7 — Summary report

After all emails are sent (or drafted), output this report in the conversation:

```
## Outreach Campaign Summary — [DATE]

**Target:** finalroundai.com  
**Competitors analysed:** 9  
**Total gap domains found:** [N] (DR 25+)  
**Emails sent:** [N]  
**Needs manual follow-up:** [N] (no email found)

### Emails Sent
| # | Domain | DR | Email | Subject |
|---|---|---|---|---|
| 1 | ... | ... | ... | ... |

### Manual Follow-up Needed (no email found)
| Domain | DR | Site | Why valuable |
|---|---|---|---|
| ... | ... | ... | ... |

### Domains Skipped (already linking to FRAI)
Listed for reference: [comma-separated list]

### Recommended Next Steps
- Follow up on non-replies in 5–7 days
- For manual follow-up sites, try LinkedIn to find the editor/owner
- Consider creating a data study or free tool to attract links from DR 60+ domains
```

---

## Error handling

| Situation | Action |
|---|---|
| Ahrefs returns 401 | Tell user the API key is invalid or expired |
| Ahrefs returns only 0 domains | Domain may not be in Ahrefs index — skip it |
| No email found for domain | Mark as MANUAL_FOLLOWUP, include in report |
| Outlook send fails | Save draft to `/tmp/outreach/drafts/<domain>.txt` |
| User has fewer than 5 gap domains | Relax DR filter to 15 and re-run the gap analysis |

---

## Quick reference — Verified API calls

```bash
# Domain rating check
curl -s "https://api.ahrefs.com/v3/site-explorer/domain-rating?target=DOMAIN&date=$(date +%Y-%m-%d)" \
  -H "Authorization: Bearer $AHREFS_KEY"
# → {"domain_rating": {"domain_rating": 65.0, "ahrefs_rank": 178550}}

# Referring domains (top 100 by DR)
curl -s "https://api.ahrefs.com/v3/site-explorer/refdomains?target=DOMAIN&select=domain_rating,domain,traffic_domain,dofollow_links,is_spam&limit=100&order_by=domain_rating:desc" \
  -H "Authorization: Bearer $AHREFS_KEY"
# → {"refdomains": [{...}, ...]}

# Available fields for refdomains:
# domain_rating, domain, traffic_domain, dofollow_links, is_spam,
# first_seen, last_seen, links_to_target, dofollow_refdomains,
# positions_source_domain, is_root_domain, ip_source
```
