# Canvas: Feedback Processor — Screenshot-to-Issue Pipeline

**Date**: 2026-03-21
**Status**: Draft — pending user review
**Scope**: Platform-level tool (not Five Pandas-specific)
**Repos**: New repo `sunj-labs/feedback-processor` or module in `platform-docs`

---

## The Problem (one paragraph)

Family members encounter bugs or confusing UX in the Five Pandas app across three time zones. Their only feedback channel is calling or texting the operator. Screenshots lose context in WhatsApp threads. By the time the operator sees the message, the UI state has changed, the exact page is unknown, and reproduction steps are lost. There's no tracking, no dedup, no acknowledgment that the issue was received, and no way for the submitter to know when it's fixed.

---

## The Bet (one paragraph)

A dedicated email address (`feedback@fivepandas.com`) that accepts screenshots + text, uses vision LLM to parse what's wrong, creates a GitHub issue with labels and context, checks for similar open issues, and replies to the submitter with a confirmation and status. Zero new infrastructure — Cloudflare email routing to existing Gmail, existing Gmail API for ingest, Haiku for parsing. Cost: ~$0.003 per submission. The family gets a "send and forget" feedback channel; the operator gets structured, triaged, deduplicated issues.

---

## Architecture

```
Family member                     Platform
─────────────                     ────────
Take screenshot
  ↓
Share to feedback@fivepandas.com
  ↓
                                  Cloudflare Email Routing
                                    ↓
                                  fivepandascapital@gmail.com
                                    ↓
                                  Gmail Ingest (BullMQ job)
                                    ↓ classify as "feedback"
                                  Feedback Processor
                                    ├─ Extract: image, text, sender
                                    ├─ Haiku Vision: parse screenshot
                                    │   → page, component, UI state
                                    │   → what looks wrong
                                    │   → suggested category
                                    ├─ Haiku Text: parse user message
                                    │   → intent (bug/feedback/feature/question)
                                    │   → severity estimate
                                    ├─ Dedup: search open GitHub issues
                                    │   → similar issues by title/description
                                    │   → if match: add comment to existing
                                    │   → if new: create new issue
                                    ├─ Create GitHub Issue
                                    │   → title, body, labels, screenshot
                                    │   → label: user-feedback, bug/enhancement
                                    │   → body: screenshot, parsed context,
                                    │     user message, page, suggested repro
                                    └─ Reply Email
                                        → "Got it — looks like a [deals page]
                                           issue. Created ticket #N.
                                           [M] similar reports open.
                                           We'll update you when it's fixed."
```

---

## Email Routing Setup (Cloudflare)

1. Cloudflare dashboard → `fivepandas.com` → Email Routing
2. Enable email routing (adds MX records automatically)
3. Add route: `feedback@fivepandas.com` → `fivepandascapital@gmail.com`
4. Optional additional routes:
   - `sanjay@fivepandas.com` → `sunjay.pandey@gmail.com`
   - `deals@fivepandas.com` → `fivepandascapital@gmail.com`

**Cost**: Free (Cloudflare email routing is included in all plans)
**Setup time**: 5 minutes

---

## Feedback Processing Pipeline

### Step 1: Email Classification

The existing email classifier (`src/agents/email-classifier.ts`) already categorizes emails. Add a new classification:

```
feedback_submission — emails to feedback@fivepandas.com
  Detection: To header contains "feedback@fivepandas.com"
  Priority: classify before other rules (explicit address)
```

### Step 2: Screenshot Extraction

MMS/email attachments come as MIME parts. Extract:
- Image attachments (PNG, JPG, HEIC) — screenshots
- Text body — user's description of the issue
- Sender email — for reply and tracking
- Subject line — often empty from phone shares, but useful if present

### Step 3: Vision Parse (Haiku)

Prompt Haiku with the screenshot:

```
You are analyzing a screenshot from the Five Pandas deal sourcing platform.

Identify:
1. Which page is shown (Dashboard, Deals list, Deal detail, Rubric, Sign-in, other)
2. What UI state is visible (loading, error, empty, populated, filtered)
3. What appears wrong or confusing (layout issue, data issue, missing content, UX confusion)
4. Suggested category: bug | ux-feedback | feature-request | question
5. Estimated severity: critical | high | medium | low
6. One-sentence summary suitable as a GitHub issue title

Return as JSON.
```

**Cost**: Haiku vision ~$0.002 per image (1000 tokens in, 200 out)

### Step 4: Text Parse (Haiku)

If the user included a text message, parse intent:

```
The user sent this feedback about the Five Pandas app:
"[user message]"

Classify:
1. Intent: bug-report | feature-request | ux-feedback | question | praise
2. Key details mentioned (specific deal, page, action they were trying to do)
3. Urgency implied (frustrated? confused? just suggesting?)

Return as JSON.
```

**Cost**: Haiku text ~$0.001 per message

### Step 5: Dedup Check

Before creating a new issue, search existing open issues:

```bash
gh issue list --state open --search "[parsed title keywords]" --json number,title,labels
```

If a match scores > 80% similarity (title + labels):
- Add a comment to the existing issue with the new screenshot + user message
- Note: "Also reported by [sender] on [date]"
- Don't create a duplicate

If no match: create new issue.

### Step 6: Create GitHub Issue

```markdown
## User Feedback — [parsed title]

**Submitted by**: [sender name/email]
**Date**: [timestamp]
**Page**: [parsed from screenshot]
**Category**: [bug | ux-feedback | feature-request]
**Severity**: [parsed estimate]

### Screenshot
[attached image]

### User Message
> [original text from email]

### AI Analysis
- **Page identified**: [deals list / deal detail / dashboard]
- **UI state**: [filtered view, showing 47 deals]
- **Issue detected**: [score sort appears incorrect — high-scoring deals not visible]
- **Similar open issues**: [#105, #106]

### Suggested Labels
`user-feedback`, `bug`, `priority:high`, `deals-page`
```

**Labels applied automatically:**
- `user-feedback` (always)
- `bug` | `enhancement` | `question` (from classification)
- `priority:*` (from severity estimate)
- Component label if identifiable (e.g., `deals-page`, `dashboard`)

### Step 7: Reply to Submitter

Send reply via Gmail (same API, same auth):

```
Subject: Re: [original subject or "Five Pandas Feedback"]

Hi [first name],

Got your feedback — thanks for flagging this!

📸 Looks like a [deals page] issue: [one-sentence summary]
🎫 Created ticket #[N]: [link to GitHub issue]
📊 We have [M] similar reports on this

We'll update you when it's resolved. In the meantime, you can
track progress at [link].

— Five Pandas
```

---

## What's Platform-Level

The feedback processor is generic. The Five Pandas-specific parts are:

| Component | Platform | App-specific |
|-----------|----------|-------------|
| Email routing | Cloudflare (any domain) | feedback@fivepandas.com |
| Email ingest | Gmail API (any inbox) | fivepandascapital inbox |
| Screenshot parser | Haiku vision prompt | Five Pandas page identification |
| Issue creation | GitHub API (any repo) | sunj-labs/poa |
| Reply template | Email send (any content) | Five Pandas branding |

**Config per app:**
```json
{
  "feedbackEmail": "feedback@fivepandas.com",
  "githubRepo": "sunj-labs/poa",
  "appName": "Five Pandas",
  "pages": ["Dashboard", "Deals", "Deal Detail", "Rubric", "Sign-in"],
  "replyFrom": "fivepandascapital@gmail.com"
}
```

A new app would just change the config — same pipeline.

---

## Open Source Angle

This could be a standalone open-source tool:

**Name ideas**: `screenshot-to-issue`, `feedback-bot`, `bug-snap`

**Value prop**: "Your users send a screenshot. AI creates a GitHub issue. They get a reply. No Jira, no forms, no friction."

**Stack**: Node.js + Gmail API + Anthropic Haiku + GitHub CLI
**Deployment**: Single Docker container or BullMQ job in existing worker

**Blog post**: "We Built a Feedback Pipeline That Turns Family Screenshots into GitHub Issues for $0.003 Each"

---

## Risks

| Risk | Mitigation |
|------|-----------|
| Haiku misidentifies the page | Include app page list in prompt; fall back to "Unknown page" |
| Screenshot is not from the app | Haiku detects non-app content; label as "unrecognized" for manual triage |
| Spam to feedback address | Rate limit by sender; only process from known family email addresses initially |
| Gmail API rate limits | Already handling; feedback volume is <10/day |
| Large image attachments | Resize before sending to Haiku; strip EXIF data |

---

## Success Criteria

1. Family member sends screenshot + text to `feedback@fivepandas.com`
2. Within 60 seconds: GitHub issue created with screenshot, parsed context, labels
3. Within 60 seconds: reply email confirms receipt with ticket number
4. Duplicate submissions add to existing issue, not create new ones
5. Operator sees structured, labeled, triaged issues — not raw screenshots

---

## Implementation Sequence

1. **Cloudflare email routing** — 5 min setup
2. **Email classifier update** — add `feedback_submission` type
3. **Feedback processor agent** — new BullMQ job type
4. **Haiku vision + text prompts** — tune on 5-10 real screenshots
5. **GitHub issue creation** — via `gh` CLI or GitHub API
6. **Reply email** — via existing Gmail send
7. **Dedup logic** — search open issues before creating
8. **Test with family** — send real screenshots, verify issue quality

**Estimated effort**: 1-2 sessions
**Estimated cost per submission**: ~$0.003 (Haiku) + $0 (Gmail, GitHub, Cloudflare)

---

## Next Steps

1. **Review this canvas** — does the flow match your expectations?
2. **Set up Cloudflare email routing** — feedback@fivepandas.com → fivepandascapital@gmail.com
3. **Build the processor** — new agent in POA worker (or separate repo)
4. **Test with real screenshots** — tune Haiku prompts
5. **Blog post** — open source the tool
