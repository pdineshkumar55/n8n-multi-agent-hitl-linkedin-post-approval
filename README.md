# LinkedIn Post Approval — n8n Workflow

An n8n workflow that uses AI to draft LinkedIn posts, with a human in the loop: nothing is saved without your explicit approval.

You type a topic into a web form → a **Writer Agent** drafts the post → a separate **Reviewer Agent** improves it → the final post is emailed to you with **Approve / Reject** buttons → the run pauses until you click → only approved posts are written to a Google Sheet.

---

## Workflow diagram

```
┌──────────────────┐   ┌──────────────┐   ┌────────────────┐   ┌────────────────────┐   ┌────────────┐   true  ┌────────────────────┐
│ LinkedIn Topic   │──▶│ Writer Agent │──▶│ Reviewer Agent │──▶│ Email for Approval │──▶│ Approved?  │───────▶│ Save Approved Post │
│ Form (trigger)   │   └──────┬───────┘   └───────┬────────┘   │ (Gmail send & wait)│   │  (IF node) │         │  (Google Sheets)   │
└──────────────────┘          │                   │            └────────────────────┘   └─────┬──────┘         └────────────────────┘
                     ┌────────┴────────┐ ┌────────┴─────────┐                                 │ false
                     │ OpenAI Model    │ │ OpenAI Model     │                                 ▼
                     │ (Writer, t=0.8) │ │ (Reviewer, t=0.3)│                        ┌────────────────────┐
                     └─────────────────┘ └──────────────────┘                        │ Rejected - Discard │
                                                                                     └────────────────────┘
```

---

## Tools used

| Tool | Purpose |
|---|---|
| n8n | Workflow orchestration |
| n8n Form Trigger | Web form to enter the topic |
| AI Agent (×2) + OpenAI Chat Model | Writer and Reviewer agents (`gpt-4.1-mini`) |
| Gmail | Sends the post and waits for an Approve/Reject response |
| Google Sheets | Stores approved posts |

---

## Nodes

| # | Node | Type | What it does |
|---|---|---|---|
| 1 | **LinkedIn Topic Form** | Form Trigger | Web form with one required field: **Topic**. Responds immediately with a "check your Gmail" message. |
| 2 | **Writer Agent** | AI Agent | Writes the first draft from the topic. |
| 3 | **OpenAI Model (Writer)** | OpenAI Chat Model | `gpt-4.1-mini`, temperature **0.8** (more creative). |
| 4 | **Reviewer Agent** | AI Agent | Judges the draft against fixed standards and returns the **rewritten, final post**. |
| 5 | **OpenAI Model (Reviewer)** | OpenAI Chat Model | `gpt-4.1-mini`, temperature **0.3** (more precise / consistent). |
| 6 | **Email for Approval** | Gmail — *Send and Wait for Response* | Emails the topic + final post with **Approve** and **Reject** buttons. The execution pauses here until a button is clicked. |
| 7 | **Approved?** | IF | Checks `{{ $json.data.approved }}` is `true`. |
| 8 | **Save Approved Post** | Google Sheets — Append Row | Writes Date, Topic, Final Post, Status = `Approved`. |
| 9 | **Rejected - Discard** | No Operation | End of the reject branch — nothing is saved. |

---

## Design decisions

### 1. Two agents, two jobs
The Writer and Reviewer are separate AI Agent nodes with their own chat model and system prompt — not one prompt doing both. The Writer optimises for getting ideas down (higher temperature); the Reviewer optimises for quality control (lower temperature).

### 2. What standards the Reviewer judges against
A reviewer can only improve what it is allowed to judge, so it is given an explicit checklist:

1. **Hook** — the first line must stop the scroll; rewrite if weak or generic.
2. **Relevance** — stays on topic and delivers one clear, concrete takeaway.
3. **Clarity** — short sentences and paragraphs, no unexplained jargon, no clichés ("in today's fast-paced world", "game-changer").
4. **Authenticity** — first-person practitioner voice, no hype, no invented statistics or claims.
5. **Length & format** — 150–250 words, plain text only (no markdown/asterisks/headings), max 2 emojis.
6. **Engagement** — ends with a specific question or call to action.
7. **Hashtags** — 3–5 relevant hashtags on the final line.
8. **Grammar & spelling**.

### 3. Feedback vs. rewritten post
The Reviewer returns the **rewritten, ready-to-publish post**, not feedback. That way the email shows exactly what will be saved, and what you approve is exactly what lands in the sheet.

### 4. What "Reject" means
**Reject = drop the post.** Nothing is written to the sheet, so the sheet stays a clean list of approved, publishable posts. The rejected post is still visible in n8n's **Executions** history if you need it.

> **Alternative:** to keep an audit trail of rejections, replace the *Rejected - Discard* node with a second Google Sheets append that writes `Status = Rejected`.

---

## Google Sheet

**Sheet:** `LinkedIn Post Approval Log` (first tab, `gid=0`)

| Date | Topic | Final Post | Status |
|---|---|---|---|
| 2026-10-27 10:15 | Lessons from building white-label Roku apps | *(final post text)* | Approved |

- **Date** — `{{ $now.setZone('Asia/Kolkata').toFormat('yyyy-MM-dd HH:mm') }}`
- **Topic** — from the form
- **Final Post** — Reviewer Agent output
- **Status** — always `Approved` (only approved posts reach this node)

---

## Credentials

All credentials live in the **n8n credential store** — none are hard-coded in the workflow.

| Credential | Type | Used by |
|---|---|---|
| OpenAI account | `openAiApi` | Both OpenAI Chat Model nodes |
| Gmail account | `gmailOAuth2` | Email for Approval |
| Google Sheets account | `googleSheetsOAuth2Api` | Save Approved Post |

---

## Setup (importing into another n8n instance)

1. **Import** the workflow JSON into n8n (*Workflows → Import from file*).
2. **Create credentials** in *Credentials*: OpenAI API key, Gmail OAuth2, Google Sheets OAuth2.
3. Open each OpenAI, Gmail and Google Sheets node and **select your credential**.
4. In **Email for Approval**, set **To** to your own email address.
5. Create a Google Sheet with headers `Date | Topic | Final Post | Status` in row 1, then select it in **Save Approved Post**.
6. **Save** the workflow as `LinkedIn Post Approval`.

---

## How to use

1. Click **Test workflow** (or **Activate** the workflow to get a permanent production form URL).
2. Open the form URL, enter a topic, and submit.
3. Within a minute you'll get an email: **"LinkedIn post for approval: \<topic\>"**.
4. Read the post and click:
   - **Approve** → a new row appears in the Google Sheet.
   - **Reject** → the post is discarded.

> The execution shows as **Waiting** in n8n until you click a button.

---

## Example prompts for the form

- Lessons from building a JSON-config-driven white-label Roku app
- Why every OTT team should automate Roku certification checks
- What I learned from completing a Generative AI architect program
- How AI coding assistants changed my code review process

---

## Possible improvements

- Log rejected posts with `Status = Rejected` (and an optional reason via a custom-form response).
- Add a "Request changes" option that sends feedback back to the Reviewer for another pass.
- Set a wait timeout on the Gmail node so stale approvals expire.
- Add an Error Trigger workflow to alert on failures.
- Post approved content directly to LinkedIn via the LinkedIn node.
