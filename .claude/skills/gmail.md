---
name: gmail
description: >
  Full Gmail manager - read, summarize, draft, send, and search emails via Google Gmail API.
  Use when the user says "check email", "gmail", "inbox", "send email", "email this person",
  "compose email", "mail", "read emails", "draft email", "search email", or wants to manage
  their Gmail inbox.
---

# Gmail Manager

Full inbox management via Google Gmail API: read, summarize, draft, send, and search emails.

## Prerequisites

1. Google OAuth must be set up: `python3 ~/.claude/skills/_google-auth/scripts/setup.py`
2. Gmail API enabled in Google Cloud project
3. Python packages: `google-auth-oauthlib`, `google-api-python-client`

## Command Routing

| User Says | Action |
|-----------|--------|
| "check email", "inbox", "read emails" | Read & Summarize |
| "check email last 3 days" | Read with timeframe |
| "send email to X", "email X" | Draft → Review → Send |
| "search email about X" | Search |
| No args / just "/gmail" | Read today's unread |

## Workflow: Read & Summarize Inbox

1. Run the read script:
```bash
python3 ~/.claude/skills/gmail/scripts/read_emails.py --since today --unread
```
Options: `--since "2 days ago"`, `--since "2026-02-20"`, `--max 20`

2. Parse the JSON output (array of email objects with: from, subject, snippet, date, labels, unread, important)

3. Categorize each email:
   - **Urgent**: Important flag, from known contacts, interview/offer keywords
   - **Needs Reply**: Questions directed at user, action items
   - **FYI**: Newsletters, notifications, automated emails
   - **Promotions**: Marketing, sales emails

4. Present summary:
```
## Inbox Summary (Feb 25, 2026)

### Urgent (2)
- **Amazon Recruiter** - "Next steps for your application" - 10:30 AM
- **Professor Smith** - "Lab 5 deadline extended" - 9:15 AM

### Needs Reply (1)
- **GAUDON Client** - "Revised project timeline" - Yesterday

### FYI (3)
- GitHub notification - PR merged
- Canvas - Assignment 4 graded
- LinkedIn - 5 new connection requests
```

## Workflow: Send Email (ALWAYS Draft-First)

**CRITICAL: Never send without user confirmation.**

**MUST READ `references/email-rules.md` before composing any email.**

### Composition Rules
- **New emails**: Use time-appropriate greeting (Good morning / Good afternoon / Good evening) based on current time
- **Replies/follow-ups**: Use respectful acknowledgment ("Thank you for your response", "I appreciate you getting back to me")
- **NEVER** use dashes (--) or hyphens (-) as separators or bullet points in email body
- Keep emails concise and well structured. Short paragraphs, one idea per paragraph.
- Signature is auto-appended from `references/signature.html`. Use `--no-signature` to skip.

### Steps
1. User provides: recipient, subject, body (or conversational intent)
2. Read `references/email-rules.md` for formatting rules
3. Compose a professional email draft following the rules
4. Show draft to user for confirmation
5. Only after user confirms, run:
```bash
python3 ~/.claude/skills/gmail/scripts/send_email.py \
  --to "recipient@email.com" \
  --subject "Subject line" \
  --body "Full email body here"
```
Optional: `--cc`, `--bcc`, `--attach file1.pdf file2.pdf`, `--no-signature`

6. Report confirmation with message ID.

## Workflow: Search Emails

```bash
python3 ~/.claude/skills/gmail/scripts/search_emails.py "search query" --max 10
```

Gmail search supports:
- `from:sender@email.com`
- `subject:keyword`
- `has:attachment`
- `filename:pdf`
- `after:2026/02/01 before:2026/02/25`
- Free text search across all fields

## Email Templates

See `references/templates.md` for pre-built templates:
- Interview follow-up / thank-you
- Recruiter response
- Cold outreach
- Professional inquiry

When composing, pick the appropriate template and personalize it.

## Error Handling

| Error | Solution |
|-------|----------|
| "OAuth credentials not found" | Run setup: `python3 ~/.claude/skills/_google-auth/scripts/setup.py` |
| "Token expired" | Script auto-refreshes; if fails, re-run setup |
| "Insufficient permissions" | Re-run setup to re-authorize with Gmail scopes |
| Empty inbox result | Widen the date range or check query syntax |

## Safety Rules

- **NEVER send email without showing draft and getting explicit user confirmation**
- Never expose full email content of third parties without user request
- Truncate long email bodies in summaries (show snippet only)
- Don't auto-reply to any emails
