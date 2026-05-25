# Email Intelligence Automation

**An AI-powered inbox triage agent. It reads, classifies, drafts a reply, and only interrupts you when something actually needs you.**

Built with [n8n](https://n8n.io), Google Gemini, and Gmail. Self-hosted, free to run, reviewer-friendly to set up.

---

## Quick start

```bash
git clone <this-repo>
cd <this-repo>
cp .env.example .env
docker compose up -d
```

n8n boots on **http://localhost:5678**. Create an owner account, then follow [Run it locally](#run-it-locally) for credentials + workflow import.

---

## The problem

Inboxes don't scale. Most "AI inbox" tools just move the noise. You go from 50 unread emails to 50 Slack pings saying "you have unread emails." That's not automation, that's a fancier todo list.

## The approach

Treat triage as the AI's actual job. For every new email:

1. **Classify** — actionable, informational, promotional, automated, or spam
2. **Prioritize** — 1–5 based on urgency and time sensitivity
3. **Summarize** — one or two sentences, no fluff
4. **Draft a reply** — written back to Gmail as a real threaded draft (not Slack text)
5. **Notify Slack only if it deserves attention** — priority ≥ 3 or AI flags it urgent

Everything else gets a quiet label. Your inbox becomes a sorted queue, not a notification firehose.

---

## What this submission does differently

Four engineering choices that separate this from a generic "AI summarizes my email" demo:

### 1. AI replies land as Gmail drafts, not Slack text

Most demos dump the suggested reply into a Slack message or Notion page. You then copy → open Gmail → find thread → paste → edit → send. Five steps.

Here the AI writes the reply directly into a **threaded Gmail draft** attached to the original message. Open Gmail → review → hit send. One step.

### 2. Priority gates notifications, so the workflow reduces attention cost

The AI sets a `slack_notify` flag and a `priority 1–5`. Slack only fires for priority ≥ 3 or AI-flagged urgent. Newsletters, automated alerts, and FYI receipts get a quiet Gmail label, never a ping. Notifications get *fewer* the more you use this, not more.

### 3. Structured output enforcement with an auditable reasoning field

Gemini returns strict JSON: `category`, `priority`, `summary`, `needs_reply`, `suggested_reply`, `gmail_label`, `slack_notify`, and a `reasoning` field. Every classification is one-sentence-explainable. Useful for debugging, demoing, and trusting the system.

### 4. Production-grade email body parsing, not vibes

The Code node decodes base64url-encoded MIME parts, walks nested multipart structures, falls back from text/plain → text/html → snippet, strips quoted reply history and signatures, and parses From headers properly. Built like you'd ship to production, not a demo that breaks on the second email.

---

## Architecture

```
┌────────────────────────────────────────────────┐
│ Gmail Inbox                                    │
│ (unread, excludes Promotions/Social/Updates)   │
└─────────────────────┬──────────────────────────┘
                      │ poll on schedule
                      ▼
┌────────────────────────────────────────────────┐
│ Code: Clean Body                               │
│ MIME walk · base64url · HTML→text · strip      │
│ signatures · normalize sender                  │
└─────────────────────┬──────────────────────────┘
                      ▼
┌────────────────────────────────────────────────┐
│ AI Agent — Gemini 2.5 Flash-Lite               │
│ + Structured Output Parser                     │
│ ──> { category, priority, summary,             │
│       needs_reply, suggested_reply,            │
│       gmail_label, slack_notify, reasoning }   │
└─────────────────────┬──────────────────────────┘
                      ▼
              ┌───────────────┐
              │ Gmail: Label  │  AI/Urgent · AI/Needs Reply
              │  (always)     │  AI/FYI · AI/Ignore
              └───────┬───────┘
                      ▼
            ┌─────────────────────┐
            │ needs_reply == true │
            └────┬───────────┬────┘
            yes  │           │  no
                 ▼           ▼
        ┌──────────────┐   ┌──────────────┐
        │ Gmail: Draft │   │ Skip draft   │
        │ threaded     │   │              │
        └──────┬───────┘   └──────┬───────┘
               │                  │
               └────────┬─────────┘
                        ▼
        ┌──────────────────────────┐
        │ priority ≥ 3 OR          │
        │ slack_notify == true?    │
        └────────┬─────────────────┘
                 │ yes
                 ▼
        ┌──────────────┐
        │ Slack ping   │   summary + priority + draft link
        └──────────────┘
```

## Stack

| Component | Choice | Why |
|---|---|---|
| **Workflow engine** | n8n Community Edition, self-hosted via Docker | Free, unlimited executions, owns the data, deploys anywhere. n8n Cloud's free tier was removed; Starter (\$24/mo) caps at 2,500 executions per month, which a polling workflow exhausts quickly. |
| **LLM** | Google Gemini 2.5 Flash-Lite | Free-tier friendly and fast. Task is bounded (classify, summarize, short reply), so Flash-Lite is the right tier. The Structured Output Parser constrains format, drafts are human-reviewed before sending. |
| **Email I/O** | Gmail OAuth 2.0 | Native n8n node, supports threaded drafts and label application. |
| **Notifications** | Slack Incoming Webhook (via HTTP Request node) | Simpler than Slack OAuth. The webhook URL is the auth. Discord webhook is a one-node swap if Slack workspace is restricted. |

---

## What was required vs what I added

**The assessment asked for** (mandatory):
- Pull emails from Gmail
- Pass to AI with relevant context
- Generate a short summary
- Draft a suggested reply
- Deliver output somewhere useful

**What I added** (the mandatory personal extension):

| Addition | Why |
|---|---|
| **Priority-aware triage labels** (`AI/Urgent`, `AI/Needs Reply`, `AI/FYI`, `AI/Ignore`) applied to every processed email | Turns the Gmail inbox itself into a sorted queue. Operational, not just chronological. |
| **Selective Slack notifications** — only priority ≥ 3 or AI-flagged urgent | Reduces attention cost. Most "AI inbox" automations create more notifications than they remove; this one creates fewer. |
| **`reasoning` field** in the AI's structured output | Every classification is explainable in one sentence. Makes the AI auditable rather than a black box. |

These are not feature bolt-ons. They change the *topology* of the workflow: the inbox itself becomes the primary surface (labels + drafts), Slack becomes a selective high-signal channel. Different design, not just more nodes.

---

## What I deliberately kept out of scope

The assessment is graded on judgment, not feature count. Each of the items below would add value, but also adds credential setup, failure surface, or testing complexity. For a clean v1, I held them back:

- **Calendar availability check** — if an email mentions a meeting time, the AI could cross-reference Google Calendar. Useful, but adds another OAuth flow and another credential for the reviewer to set up.
- **Persistent sender memory** — track past senders to detect VIPs, recurring threads, or frequent contacts. Useful, but needs a DB layer and a lookup step.
- **RAG over sent emails** — match the user's writing style by retrieving similar past sends as in-context examples. Useful, but adds embedding storage and a vector DB.
- **Auto-send replies** — the AI's reply lands as a draft, never an outgoing message. Auto-send would multiply the cost of any AI mistake. Drafts force human review, which is the right tradeoff for v1.

## What I'd add for production

- **n8n Error Trigger workflow** to catch failures (Gemini timeout, Gmail OAuth expired, Slack 5xx) and route them to a dedicated logging channel with retry. The current workflow fails individual executions on error; production needs visibility and resilience.
- **Multi-account routing** with a credential-per-row table for team deployments: one workflow, many inboxes.
- **Published OAuth consent screen** (after Google's verification process) so refresh tokens don't periodically expire.
- **Sender allowlist/denylist** for finer control over what gets processed.

## Portability

Credentials are intentionally not committed in `workflow.json`. The repo is portable: reviewers clone, run `docker compose up`, import the workflow, set up their own credentials in n8n's UI, and they're running. No data leakage, no rebuilds required.

---

## Run it locally

Most of the setup is waiting on Google OAuth propagation; the click steps themselves are quick.

### Prerequisites

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed
- A Google account with Gmail (fresh test account recommended)
- A Slack workspace with permission to install apps ([create a free one](https://slack.com/get-started))

### 1. Start n8n

```bash
git clone <this-repo>
cd <this-repo>
docker compose up -d
```

Open **http://localhost:5678** → create an owner account (email + password). First-time setup persists in the Docker volume.

Optional: copy `.env.example` to `.env` only if you want to override local n8n infrastructure settings. Credentials do **not** go in `.env`.

### 2. Get a Gemini API key

1. Go to **https://aistudio.google.com/apikey**
2. **Create API key** → "in a new project" or use existing
3. Copy the key (starts with `AIzaSy...`)

In n8n:
4. **Credentials** → **Add credential** → search `gemini` → **Google Gemini (PaLM) API**
5. Paste the key → **Credential Name:** `Gemini AI Studio` → **Save**

### 3. Set up Gmail OAuth

The longest step. You'll bounce between n8n and [Google Cloud Console](https://console.cloud.google.com).

**In n8n:** Credentials → Add → **Gmail OAuth2 API**. The form shows an **OAuth Redirect URL** at the top:

```
http://localhost:5678/rest/oauth2-credential/callback
```

Keep this n8n tab open. Now in GCP:

1. **Create a project** (top bar → New Project)
2. **Enable Gmail API** — APIs & Services → Library → search `Gmail API` → **Enable**
3. **OAuth consent screen** — APIs & Services → OAuth consent screen → **External** → fill app name, support email, dev email → save through Scopes (leave empty) → **Test Users** → **+ Add Users** → enter your Gmail. ⚠️ **Without this, sign-in later fails with "access blocked".**
4. **Create OAuth client** — APIs & Services → Credentials → **+ Create Credentials** → **OAuth client ID** → **Application type: Web application** → name: anything → **Authorized redirect URIs** → paste the n8n redirect URL above → **Create**
5. Copy **Client ID** and **Client Secret** from the popup using the copy icons

**Back in n8n:** paste both → click **Sign in with Google**.

⚠️ **If you see "invalid_client / The provided client secret is invalid" — wait for propagation.** Google takes a while to propagate new OAuth clients. The error is misleading; the cause is propagation delay. GCP shows a banner about this.

When the popup opens:
- Pick your Gmail
- "Google hasn't verified this app" → **Advanced** → **Go to <app> (unsafe)** — expected for Testing-mode OAuth (it's your own app)
- Grant the permissions
- **Credential Name:** `Gmail OAuth` → **Save**

> **Refresh tokens periodically expire** while consent screen is in Testing mode. Re-click Sign in when needed. For production: publish consent screen → Google verification process.

### 4. Create a Slack Incoming Webhook

1. **https://api.slack.com/apps** → **Create New App** → **From scratch** → pick workspace
2. Left sidebar → **Incoming Webhooks** → toggle **On**
3. **Add New Webhook to Workspace** → pick a channel → **Allow**
4. Copy the URL and paste it into a notes app (not into n8n's credential vault, we use it differently)

### 5. Create the 4 Gmail labels

In **https://mail.google.com** → Settings ⚙️ → See all settings → **Labels** tab → **Create new label** → create each:

```
AI/Urgent
AI/Needs Reply
AI/FYI
AI/Ignore
```

The `/` creates a nested label under a parent `AI`.

### 6. Import the workflow

In n8n: **Workflows** → **+ Add Workflow** → drag `workflow.json` from this repo onto the empty canvas.

### 7. Re-bind credentials (5 nodes)

Click each red-warning node and pick the matching credential from the dropdown:

| Node | Credential |
|---|---|
| Gmail Trigger | `Gmail OAuth` |
| Gmail: Get Labels | `Gmail OAuth` |
| Gmail: Add Label | `Gmail OAuth` |
| Gmail: Create Draft | `Gmail OAuth` |
| Google Gemini Chat Model | `Gemini AI Studio` |

### 8. Paste the Slack webhook URL

Click **Slack: Webhook Notification** → in **URL** field, replace `__REPLACE_WITH_SLACK_WEBHOOK_URL__` with your real webhook URL from step 4 → **Save** the workflow (Cmd+S).

Leave **Discord: Webhook Notification (Fallback)** disabled unless Slack setup is blocked. If enabling it, replace `__REPLACE_WITH_DISCORD_WEBHOOK_URL__` with a Discord webhook URL.

### 9. Test it

Send yourself emails from another account to exercise the three main paths. Run **Execute Workflow** after each one arrives in your inbox.

**Example 1 — actionable, needs reply (high priority)**

> **Subject:** Quick question about Tuesday
> **Body:** Can you confirm we're still meeting at 3pm to go over the proposal?

Expect: `AI/Needs Reply` label, threaded Gmail draft created, Slack notification fires.

**Example 2 — informational, no reply needed (low priority)**

> **Subject:** Receipt for your Stripe payment
> **Body:** Thanks for your payment of $49 to Acme Co. Your receipt is attached.

Expect: `AI/FYI` label applied, **no draft**, **no Slack notification** (priority < 3 + needs_reply=false).

**Example 3 — promotional / noise**

> **Subject:** 50% off all annual plans this week only
> **Body:** Limited-time offer for new subscribers...

Expect: `AI/Ignore` label, no draft, no Slack. Most of the time this gets filtered out by the Gmail Trigger's `-category:promotions` filter before it even reaches the workflow.

Verify after each:
- ✅ **Gmail Drafts** → reply present only for actionable cases
- ✅ **Original email** → correct `AI/...` label
- ✅ **Slack channel** → only the high-priority case rings; the others stay quiet

### 10. Activate

When the test passes, flip the **Inactive → Active** toggle (top right). n8n polls the inbox on schedule. Walk away.

---

## Known limitations

- **OAuth refresh tokens periodically expire** in Testing mode. Re-click Sign in when needed. Production fix: publish consent screen → Google verification.
- **Polling latency** matches the configured poll interval. Real-time would need Gmail Pub/Sub + GCP project setup.
- **Single Gmail account** — workflow is wired to one credential. Multi-user requires credential-per-row routing.
- **No persistent sender memory** — every email is processed independently. Adding a sender history layer would need a DB and lookup step.
- **No error workflow wired** — if Gemini, Gmail, or Slack fails for a message, that execution fails. In production, wire an n8n Error Trigger workflow to log + retry.

---

## Troubleshooting

| Symptom | Cause / Fix |
|---|---|
| `invalid_client / client secret is invalid` on Gmail Sign in | Google hasn't propagated your new OAuth client yet. Wait for propagation, then retry. |
| `access blocked: This app's request is invalid` | You're not added as a Test User on the OAuth consent screen. Fix: GCP → OAuth consent screen → Test Users → Add Users. |
| Slack returns 400 | Body must be JSON `{"text": "..."}`. The workflow already sends correct format; only an issue if you edited the body. |
| Workflow runs but no draft appears | Email may have classified as non-actionable or `needs_reply: false`. Inspect the AI Agent node's output in the Executions panel. |
| `Invalid label: AI/Needs Reply` | You are running an older workflow version that passed the display name directly to Gmail. Re-import `workflow.json`; the current version resolves Gmail label names to internal label IDs before applying them. |
| Label not applied | Make sure all 4 labels exist in Gmail before running (`AI/Urgent`, `AI/Needs Reply`, `AI/FYI`, `AI/Ignore`) and that `Gmail: Get Labels` is bound to `Gmail OAuth`. |
| `Invalid email address` in Gmail Draft | Re-import the current `workflow.json` or update `Code: Clean Body`. The current parser extracts the email address from `From`, `Reply-To`, and related Gmail fields before creating drafts. |
| Gemini connection test fails | Verify full API key copied. Check rate limits in [AI Studio dashboard](https://aistudio.google.com/). |

---

## Repo contents

| File | Purpose |
|---|---|
| `README.md` | This file — what + why + how to run |
| `workflow.json` | The n8n workflow (importable) |
| `docker-compose.yml` | n8n container config |
| `.env.example` | Template for n8n config overrides |
| `.gitignore` | Excludes `.env` from commits |
