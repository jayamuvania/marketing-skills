---
name: link-building-outreach
description: >
  Runs a full link-building outreach campaign for Final Round AI (or any domain).
  Use this skill whenever the user says "run outreach", "find backlink opportunities",
  "competitor backlink analysis", "who links to our competitors", "link building",
  "find sites to pitch", or "send outreach emails". Operates as an SEO expert:
  identifies the exact page on each target site to get a link from, chooses the right
  FRAI product page to link to, writes direct outreach pitches (link insertion or guest
  post), fixes any bad emails from previous batches, replies to any incoming responses,
  and sends everything via Gmail SMTP using multiple parallel agents.
---

# Link-Building Outreach Skill

## SEO Master Mindset — Read This First

You are operating as an **SEO expert**, not a PR team. The goal of every email is
one of two things:

1. **Link insertion** — get a link added to an existing page on their site that
   already ranks or has authority, pointing to a specific FRAI product page
2. **Guest post** — pitch a specific article topic that you will write for their
   site, which will naturally include a link to a FRAI product page AND will itself
   rank for queries that help FRAI appear in AI Overviews and LLM answers

Every decision — which domain to target, which page to pitch, which product to link
to, what topic to propose — must be made through this lens.

---

## FRAI Products to Focus On (in order of priority)

| Product | Page | What it does |
|---|---|---|
| **AI Interview Copilot** | finalroundai.com/interview-copilot | Real-time AI coaching during live interviews |
| **AI Mock Interview** | finalroundai.com/ai-mock-interview | Practice interviews with realistic AI interviewers |
| **Homepage** | finalroundai.com | General brand / all products |

> **Do not pitch the Resume Builder or Question Bank as the primary ask.** Every
> outreach must drive traffic and links to Interview Copilot, AI Mock Interview, or
> the homepage. Mention other products only in passing.

---

## Guest Post Topics That Build AI Overview Visibility

When pitching a guest post, choose a topic from this list. These are designed to
rank for queries where ChatGPT, Gemini, and Google AI Overview will cite the
article — and where the article naturally links to FRAI.

| Topic | Links to | Why it builds AI Overview visibility |
|---|---|---|
| "Best AI Interview Copilot Tools in [year]" | /interview-copilot | Direct category match; LLMs cite "best X" roundups constantly |
| "How AI Interview Copilots Work: Real-Time Coaching Explained" | /interview-copilot | Definitional content; AI Overview loves explaining "what is X" |
| "AI Mock Interview vs Real Interview: What to Expect" | /ai-mock-interview | Comparison content ranks well + gets cited for "how to prepare" queries |
| "How to Use AI to Prepare for Technical Interviews" | /ai-mock-interview | High-intent how-to; cited when users ask AI assistants for interview advice |
| "Top AI Tools for Job Seekers in [year]" | homepage | Roundup format; frequently cited by LLMs for "AI job search tools" |
| "Does AI Interview Coaching Actually Work? Data and Evidence" | /interview-copilot | Data-driven = high citation value in AI Overviews |
| "The Candidate's Guide to AI-Screened Hiring" | homepage | Timely; gets cited when AI answers questions about modern hiring |

Pick the topic that best matches the target site's existing content style and audience.

---

## Workflow — Every Campaign Follows This Order

### Step 0: Check inbox for replies (ALWAYS first)

Before doing anything else, read the `alex@finalroundai.net` inbox for any replies
to previous outreach emails. Use Gmail IMAP to read the inbox.

**Star emails that need action** — after reading the inbox, use IMAP to star any email
that requires a manual response or action from Jaya. This includes:
- Positive replies (they said yes, asked for rates, requested a draft)
- Paid deal confirmations (they quoted a price)
- Emails where Jaya needs to write and submit a draft
- Any reply that can't be handled automatically

```python
import imaplib, email
from email.header import decode_header

SENDER = "alex@finalroundai.net"
APP_PASSWORD = "rwlnedjhkqurdskp"  # Claude Outreach 2 — generated May 25 2026

mail = imaplib.IMAP4_SSL("imap.gmail.com")
mail.login(SENDER, APP_PASSWORD)
mail.select("inbox")

_, msgs = mail.search(None, 'SINCE "SINCE-DATE"')
ids = msgs[0].split()

for uid in ids:
    _, data = mail.fetch(uid, "(RFC822)")
    msg = email.message_from_bytes(data[0][1])
    sender = msg.get("From", "")
    subject = decode_header(msg["Subject"] or "")[0][0]
    if isinstance(subject, bytes):
        subject = subject.decode(errors="replace")
    # Star if it's a real reply from an outreach target (not a bounce, spam, or auto-reply)
    if needs_action(sender, subject):  # use your judgement
        mail.store(uid, "+FLAGS", "\\Flagged")
        print(f"⭐ Starred: {sender} | {subject}")

mail.logout()
```

For each reply, respond **immediately as an SEO expert**:

- If they say yes to a guest post → confirm the title, target URL (Interview Copilot
  or AI Mock Interview page), word count (1,200–1,500 words), and deadline
- If they ask for more info about what we want → be direct: "We'd love a link to
  our AI Interview Copilot page (finalroundai.com/interview-copilot) from your
  [specific article]. If you'd prefer, we can write a guest post instead."
- If they ask for payment → confirm you're open to it, ask for their rate card
- If they decline → thank them, ask if there's a different format that works

Do not move to Step 1 until all replies are handled.

### Step 1: Fix bad emails from previous batch

Pull the list of domains that were previously contacted with guessed or unverified
emails. Launch parallel agents (2 domains each) to find the correct email for each.

For each domain where a better email is found:
- Send a **fresh, improved email** to the verified address (not a re-send of the
  same generic email — write a sharper, more direct version)
- Log the correction

### Step 2: Run new batch (Ahrefs → gap → research → send)

Follow Phases 1–5 below for the new set of domains.

### Step 3: Send Slack summary to #seo-backlink-alerts

After everything above is complete, post the full summary. Never skip this.

---

## Phase 1 — Pull Ahrefs referring domains (parallel)

```bash
AHREFS_KEY="your_key_here"
mkdir -p /tmp/outreach

fetch_refdomains() {
  local label=$1
  local target=$2
  curl -s \
    "https://api.ahrefs.com/v3/site-explorer/refdomains?target=${target}&select=domain_rating,domain,traffic_domain,dofollow_links,is_spam&limit=100&order_by=domain_rating:desc" \
    -H "Authorization: Bearer ${AHREFS_KEY}" \
    > "/tmp/outreach/${label}.json"
  echo "Fetched ${label}: $(jq '.refdomains | length' /tmp/outreach/${label}.json) domains"
}

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
echo "All fetched."
```

---

## Phase 2 — Build gap list (Python only — bash times out on 600+ rows)

```python
import json

EXCLUDE = {
    "youtube.com","google.com","google.co.uk","github.com","wikipedia.org",
    "linkedin.com","twitter.com","x.com","facebook.com","instagram.com",
    "reddit.com","tiktok.com","apple.com","microsoft.com","amazon.com",
    "trustpilot.com","g2.com","capterra.com","producthunt.com",
    "ycombinator.com","web.archive.org",
    "interviewsidekick.com","interviewcoder.co","parakeet-ai.com",
    "vervecopilot.com","senseicopilot.com","interview-assistant-ai.com",
    "lockedinai.com","beyz.ai","linkjob.ai","finalroundai.com"
}

MIN_DR = 25
base = "/tmp/outreach"

with open(f"{base}/own.json") as f:
    own_domains = {r["domain"] for r in json.load(f).get("refdomains", [])}

competitors = ["sidekick","coder","parakeet","verve","sensei","intasst","lockedin","beyz","linkjob"]
all_rows = []
for comp in competitors:
    with open(f"{base}/{comp}.json") as f:
        for r in json.load(f).get("refdomains", []):
            if not r.get("is_spam"):
                all_rows.append({"src": comp, "domain": r["domain"],
                    "dr": float(r.get("domain_rating") or 0),
                    "traffic": int(r.get("traffic_domain") or 0),
                    "dofollow": int(r.get("dofollow_links") or 0)})

seen, gap = set(), []
for r in sorted(all_rows, key=lambda x: -x["dr"]):
    d = r["domain"]
    if d in seen or d in own_domains or d in EXCLUDE: continue
    if any(d.endswith("."+e) for e in EXCLUDE): continue
    if r["dr"] >= MIN_DR and r["dofollow"] >= 1:
        seen.add(d); gap.append(r)

gap.sort(key=lambda x: -x["dr"])
with open(f"{base}/gap_domains.tsv", "w") as f:
    for r in gap:
        f.write(f"{r['domain']}\t{r['dr']}\t{r['traffic']}\t{r['dofollow']}\t{r['src']}\n")
print(f"Gap domains: {len(gap)}")
```

**Second-pass filter before curating targets:**
Remove: DR > 85 (massive platforms), news syndication (NBC affiliates, regional TV),
non-English sites, e-commerce/SaaS unrelated to careers/AI/developers, dofollow < 1.

**Target sweet spot: DR 30–85, dofollow ≥ 1, content covers AI tools / career / HR / dev.**

Curate top 20 into `/tmp/outreach/outreach_targets.tsv`.

---

## Phase 3 — Research (10 parallel agents, 2 domains each)

**NEVER use a single agent for all domains.** Always launch 10 agents in parallel
(2 domains each). This cuts research from ~15 min to ~3 min.

### What each agent must return (JSON per domain)

```json
{
  "domain": "ranktracker.com",
  "existing_rankable_page": "URL of their page that ranks for interview/AI/career queries",
  "page_ranking_for": "what query/topic that page ranks for",
  "link_target_page": "which FRAI page to link to — interview-copilot | ai-mock-interview | homepage",
  "link_target_reason": "why this specific FRAI page is the right fit for their audience",
  "pitch_type": "link_insertion | guest_post",
  "guest_post_topic": "exact proposed title if pitch_type is guest_post",
  "contact_email": "verified email address",
  "email_confidence": "found | guessed | MANUAL_FOLLOWUP",
  "email_source": "where the email was found (e.g. /contact page, WebSearch, guessed pattern)"
}
```

### Research agent prompt template

```
You are an SEO expert doing link-building research for Final Round AI (finalroundai.com).

FRAI's two main products:
- AI Interview Copilot (finalroundai.com/interview-copilot) — real-time AI coaching during live interviews
- AI Mock Interview (finalroundai.com/ai-mock-interview) — practice interviews with AI interviewers

Your job for each domain:
1. Find the specific existing page on their site that is most relevant to interview prep,
   AI tools, or job search — ideally one that already ranks in search or gets traffic.
   This is the page we want a link insertion on, OR the topic area for a guest post.
2. Decide: link insertion (add FRAI to an existing article) or guest post (write new content)?
   Prefer link insertion if they have a relevant ranking page. Choose guest post if they
   have no relevant page but accept contributions.
3. If guest post: pick the exact title from this list that best fits their audience:
   - "Best AI Interview Copilot Tools in 2026"
   - "How AI Interview Copilots Work: Real-Time Coaching Explained"
   - "AI Mock Interview vs Real Interview: What to Expect"
   - "How to Use AI to Prepare for Technical Interviews"
   - "Top AI Tools for Job Seekers in 2026"
   - "Does AI Interview Coaching Actually Work? Data and Evidence"
   - "The Candidate's Guide to AI-Screened Hiring"
4. Find the correct contact email. Try: WebFetch on /contact, /about, /write-for-us,
   /advertise. Also WebSearch: "editor email site:<domain>" and "write for us <domain>".
   Mark confidence as: found (confirmed on page), guessed (pattern inference), MANUAL_FOLLOWUP.

Domains: <domain1>, <domain2>

Output ONLY a JSON array with this exact structure per domain:
[{
  "domain": "...",
  "existing_rankable_page": "URL or null",
  "page_ranking_for": "query/topic or null",
  "link_target_page": "interview-copilot | ai-mock-interview | homepage",
  "link_target_reason": "one sentence",
  "pitch_type": "link_insertion | guest_post",
  "guest_post_topic": "exact title or null",
  "contact_email": "email or MANUAL_FOLLOWUP",
  "email_confidence": "found | guessed | MANUAL_FOLLOWUP",
  "email_source": "where found"
}]
```

---

## Phase 4 — Write direct outreach emails

### The fundamental rule: be direct and specific

**Bad (what was sent in Wave 1 — do not repeat):**
> "We thought your readers might find Final Round AI useful — it gives real-time coaching
> during live interviews. Would you be open to including it in your roundup?"

**Good (what to write instead):**
> "I saw your article '[exact title]' — it ranks well for '[query]'. We'd love to get a
> mention there linking to our AI Interview Copilot (finalroundai.com/interview-copilot).
> Happy to pay for the placement if that's easier."

The email must answer three questions in the first two sentences:
1. Which specific page on their site are we talking about?
2. Which specific FRAI page do we want linked?
3. What exactly are we asking them to do?

### Email templates by pitch type

#### Link insertion pitch
```
Subject: Link to [FRAI product] from your "[their article title]" post?

Hi [Name],

I came across your article "[exact title]" — it ranks well for [query] and is exactly
the kind of content our audience is reading when they're looking for tools like ours.

We'd love to get a mention there linking to our [AI Interview Copilot / AI Mock Interview]
page (finalroundai.com/[path]). We could provide a short description you can drop in,
or if you'd prefer a sponsored placement we're open to that too — just let us know your rate.

[Alex]
Final Round AI
alex@finalroundai.net
https://www.finalroundai.com
```

#### Guest post pitch
```
Subject: Guest post for [site]: "[exact proposed title]"

Hi [Name],

I'd like to write a guest post for [site name] titled "[exact proposed title]".

The piece would cover [1-sentence description of what the article covers and why it's
useful for their audience]. It would include a link to our [AI Interview Copilot /
AI Mock Interview] page as a recommended tool — which fits naturally given the topic.

I can deliver [1,200–1,500] words, fully edited, within [timeframe]. Happy to do it as
a sponsored post too if that's a cleaner path for your team.

[Alex]
Final Round AI
alex@finalroundai.net
https://www.finalroundai.com
```

### Rules (every email must follow these)

- **Name the specific article** — never pitch "your blog" or "your site" generically
- **Name the specific FRAI page** — always say which URL we want linked
- **State the ask in sentence 2** — don't bury it
- **Mention paid option** — "Happy to pay for the placement / do it as a sponsored post"
- **150 words maximum** — tight, direct, no fluff
- **No bullet points in body**
- **No "I hope this finds you well", "synergies", "value-add"**
- **Adjust for site type** — investigative journalism = story pitch only (no link ask)

### Tone by site type

| Site type | Tone adjustment |
|---|---|
| SEO / marketing blog | Most direct — name the page, name the link, ask clearly |
| HR / recruiting pub | Professional but specific — name their article, pitch the article we'd write |
| Developer community | Technical angle — coding interviews, technical prep |
| AI tools directory | Even simpler — "we'd like to be listed, here are our details, happy to pay for a featured slot" |
| Investigative journalism | Do NOT ask for a link — pitch a story angle about AI and hiring |
| Warm lead (already mentions FRAI) | Escalate — reference existing mention, pitch partnership/co-marketing directly |

---

## Phase 5 — Send via Gmail SMTP

```python
import smtplib, time, json
from email.mime.text import MIMEText

SENDER = "alex@finalroundai.net"
APP_PASSWORD = "rwlnedjhkqurdskp"  # Claude Outreach 2 — generated May 25 2026

with open("/tmp/outreach/emails.json") as f:
    emails = json.load(f)

with smtplib.SMTP_SSL("smtp.gmail.com", 465) as server:
    server.login(SENDER, APP_PASSWORD)
    for e in emails:
        msg = MIMEText(e["body"])
        msg["Subject"] = e["subject"]
        msg["From"] = SENDER
        msg["To"] = e["to"]
        try:
            server.sendmail(SENDER, [e["to"]], msg.as_string())
            print(f"✓ SENT  {e['domain']:<25} → {e['to']}")
        except Exception as ex:
            print(f"✗ FAIL  {e['domain']:<25} → {ex}")
        time.sleep(0.5)
```

Save every email as `/tmp/outreach/drafts/<domain>.txt`.
**Send without confirmation prompt** — show results table after.

---

## Phase 6 — Summary report

```
## Outreach Campaign Summary — [DATE]

**Replies handled:** [N] (from previous batch)
**Bad emails fixed and resent:** [N]
**New domains contacted:** [N]
**Total emails sent this session:** [N]
**Manual follow-up needed:** [N]

### Link Insertions Pitched
| Domain | DR | Their Page | FRAI Link Target | Email |
|---|---|---|---|---|

### Guest Posts Pitched
| Domain | DR | Proposed Title | FRAI Link Target | Email |
|---|---|---|---|---|

### Replies Received & Handled
| Domain | Reply summary | Response sent |
|---|---|---|

### Manual Follow-up Needed
| Domain | DR | Suggested action |
|---|---|---|
```

---

## Phase 7 — Post to #seo-backlink-alerts (MANDATORY, always last)

Channel: `#seo-backlink-alerts` — ID: `C0ANV0PHERJ`

Keep it short. The team reads this on their phone. Under 40 lines, no tables, one line per domain.

```
🔗 *Link Building | [DATE]*

📤 *Emails sent:* [N]
🔁 *Re-reached (corrected addresses):* [N]
💬 *Replies received:* [N] — [site names]

💰 *Paid confirmed:* $[total] ([site] $[N] + [site] $[N])

🎯 *Sites contacted & ask:*
• [domain] — link insertion → /interview-copilot
• [domain] — guest post: "[Title]" → /ai-mock-interview
• [domain] — directory listing → /interview-copilot
(one line per domain, every domain contacted this session)

📊 *Campaign stats:*
• Already reached out: [N] domains total
• Remaining in pipeline: [N] domains

📊 *Stats:* Already reached: [N] domains | Remaining: [N] domains

👩 *Action items for Jaya:*
• [domain] — [exactly what Jaya needs to do, one line]
• [domain] — [exactly what Jaya needs to do, one line]
```

### Rules
- One line per domain — just the name + what we asked + which FRAI page
- No tables, no descriptions, no nested bullets
- Paid deals clearly visible with dollar amounts
- Always include the stats block (total reached + remaining pipeline)
- Omit sections that are empty (no replies = no replies line, no paid = no paid line)
- The "Action items for Jaya" section is ALWAYS last and ALWAYS present if there are any manual actions
- Each action item must say exactly what to do — not just the domain name. Examples:
  - "unite.ai — DM Antoine Tardif on LinkedIn: linkedin.com/in/antoine-tardif — ask him to add link to /interview-copilot on their existing FRAI review page"
  - "airbyte.com — submit guest post via airbyte.com/community/write-for-the-community (they pay $300–500)"
  - "cloudways.com — submit via cloudways.com/en/write-for-us.php — topic: How to Use AI to Prepare for Technical Interviews"
- Under 40 lines total — if it goes longer, cut domain descriptions not domains

---

## Gmail setup (one-time)

App Password: `piidifgzcvxhrgge` (already configured for alex@finalroundai.net)

If it stops working: myaccount.google.com → Security → 2-Step Verification → App passwords → create new.

---

## Error handling

| Situation | Action |
|---|---|
| Ahrefs 401 | API key expired — ask user for new one |
| No email found | Mark MANUAL_FOLLOWUP — suggest LinkedIn/X DM to editor |
| Gmail send fails | Save to `/tmp/outreach/drafts/<domain>.txt`, log as failed |
| Research agent >10 min | Kill and relaunch as 2-domain batches |
| Slack send fails | Show formatted message in chat, ask user to post manually |
| Fewer than 5 gap domains | Relax DR filter to 15 |

---

## Quick reference

```bash
# Referring domains (top 100 by DR)
curl -s "https://api.ahrefs.com/v3/site-explorer/refdomains?target=DOMAIN&select=domain_rating,domain,traffic_domain,dofollow_links,is_spam&limit=100&order_by=domain_rating:desc" \
  -H "Authorization: Bearer $AHREFS_KEY"
```

---

## Lessons learned

- **Python only for gap filtering** — bash while-loop times out on 600+ rows
- **10 parallel agents (2 domains each)** — cuts research from 15 min to 3 min
- **Send before research finishes** — initial send from domain knowledge, follow-ups when agents return verified emails
- **Second-pass filter is critical** — removes NBC affiliates, spa software, crypto tools
- **Jobscan already mentions FRAI** — warm lead, escalate to partnerships immediately
- **404media = story pitch only** — never send a guest post or ad pitch to investigative outlets
- **Fazier = no email** — submit via @FazierHQ on X or buy $39 paid slot
- **Slack channel: `#seo-backlink-alerts`** (ID: `C0ANV0PHERJ`)
- **Wave 1 emails were too generic** — name the specific article, name the specific FRAI page, state the ask in sentence 2
- **Half the Wave 1 emails had wrong addresses** — always prioritise `found` confidence emails; resend to verified addresses before running next batch
