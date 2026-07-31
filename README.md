# 🤖 AI Job Alert Bot (n8n workflow)

An n8n automation that checks a free job board every day, uses an LLM
(Groq) to score how well each new listing matches your profile, and
sends you a Slack alert only for the good matches.

Unlike the other repos in this series, this one isn't a Streamlit app —
it's a single importable **n8n workflow** (`AI_Job_Alert_Bot.json`)
plus this guide.

## What it does

```
Schedule (daily)
   → Fetch job listings (Arbeitnow API, free, no auth)
   → Keyword pre-filter (cheap, local -- keeps AI calls low)
   → Skip jobs already seen (deduped via n8n's built-in workflow storage)
   → Score each job against your profile with an LLM (Groq)
   → If score >= threshold → Slack alert
   → Otherwise → do nothing
```

## Why this design

- **No external database needed for dedup.** It uses n8n's built-in
  `$getWorkflowStaticData('global')`, which persists with the workflow
  itself -- no Postgres/Redis/Sheets setup required just to remember
  which jobs you've already seen.
- **Keyword pre-filter before the AI call.** Job boards return a lot of
  irrelevant postings; filtering locally first keeps your Groq API
  usage (and the run time) low.
- **Secrets live in n8n Credentials, not the workflow JSON or a Code
  node's `process.env`.** This intentionally avoids the
  `N8N_BLOCK_ENV_ACCESS_IN_NODE` issue -- n8n blocks Code nodes from
  reading environment variables directly, so this workflow uses n8n's
  proper Credentials system for the Groq API key instead.

## Setup

### 1. Import the workflow

In n8n: **Workflows → Import from File** → select `AI_Job_Alert_Bot.json`.

### 2. Create the Groq credential

1. In n8n, go to **Credentials → New → HTTP Header Auth**.
2. Name it something like `Groq API`.
3. Set:
   - **Header Name:** `Authorization`
   - **Header Value:** `Bearer YOUR_GROQ_API_KEY` (get a free key at [console.groq.com/keys](https://console.groq.com/keys))
4. Open the **"Score Match with AI"** node in the workflow and select this credential under Authentication.

### 3. Edit your candidate profile

Open the **"Score Match with AI"** node and find the `CANDIDATE PROFILE
(EDIT ME)` line inside the JSON body. Replace it with a short
description of your skills, experience, and what you're looking for.

### 4. Set your Slack webhook (or swap the notification channel)

1. Create a [Slack Incoming Webhook](https://api.slack.com/messaging/webhooks) for a channel.
2. Open the **"Send Slack Alert"** node and replace `YOUR_SLACK_WEBHOOK_URL_HERE` with your webhook URL.
3. Prefer Telegram, Discord, or email instead? Delete this node and
   swap in n8n's built-in Telegram/Discord/Email node -- the incoming
   item already has `title`, `company_name`, `score`, `reason`,
   `location`, `remote`, and `url` fields ready to use.

### 5. Adjust the keyword filter and match threshold

- **"Filter & Prep Jobs"** node: edit the `keywords` array to match
  the roles/technologies you care about.
- **"Good Match?"** node: change `65` (the tested default) to your
  preferred minimum match score (0-100) if you want it stricter or looser.

### 6. Test, then activate

Run the workflow manually first (the "Execute Workflow" button) to
confirm it fetches jobs and posts to Slack correctly. Once it looks
right, toggle the workflow **Active** so the daily schedule takes over.

## Customization ideas

- Swap the Arbeitnow API for RemoteOK, Adzuna, or an RSS feed from any
  other job board.
- Add a Google Sheets node to log every scored job (not just matches)
  for later review.
- Add a second AI step that drafts a tailored cover letter opener for
  each strong match.
- Lower the keyword pre-filter list to widen the net, or raise the
  match threshold to be more selective.

## A note on the AI model

This workflow calls Groq's `openai/gpt-oss-20b` model. Groq
deprecated `llama-3.1-8b-instant` (shutdown date: August 16, 2026), so
this avoids that model. Check
[console.groq.com/docs/deprecations](https://console.groq.com/docs/deprecations)
periodically and update the model name in the "Score Match with AI"
node's JSON body if needed.

## License

MIT — free to use, modify, and share.
