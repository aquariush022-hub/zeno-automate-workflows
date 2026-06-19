# ZenoAutomate — n8n Workflow Portfolio

Technical documentation of automation workflows built by Shubham Patel.  
All workflows are deployed on Railway and run 24/7 with no local machine dependency.

- [Upwork Profile](https://www.upwork.com/freelancers/~010edc3e1651bcf87b)
- zenoautomate@gmail.com

---

## AI Outreach Suite — 3-Workflow Chained Pipeline

The most complex system here. Three separate n8n workflows chained together via Execute Workflow nodes to form one fully automated B2B outreach pipeline. A single chat message triggers the entire sequence — finding businesses, finding their emails, and sending them personalised cold emails — with no manual input at any stage.

```
Chat Trigger (natural language query)
        |
        v
[Workflow 1] AI Business Lead Scraper
        |
        | → Google Sheets (business data written)
        |
        v
[Workflow 2] Lead Email Enricher
        |
        | → Google Sheets (email column filled)
        |
        v
[Workflow 3] AI Cold Email Sender
        |
        v
Gmail (personalised email sent per business)
```

---

### Workflow 1 — AI Business Lead Scraper

**Trigger:** Chat node (n8n built-in)  
**Output:** Google Sheets row per business found

**How it works:**

The user types a natural language query like "find cafes in Pune." An AI Agent node interprets the intent and extracts the location and business type. It then calls the Serper API (Google Maps data) via HTTP Request node and processes the raw response through a JavaScript Code node that structures each result into clean fields — name, address, phone, website, category, rating — and writes them to Google Sheets row by row.

**Key technical decisions:**

- Used AI Agent node instead of a simple HTTP Request so the query doesn't need to follow a fixed format — the AI handles intent extraction
- Serper API returns Google Maps results which include website URLs and social links — critical for the next workflow
- Added a `category` column manually to the output so Workflow 3 can read business type without doing another API call
- JavaScript Code node handles cases where Serper returns incomplete data (missing phone, no website) by filling those fields with empty strings instead of breaking the flow

**Stack:** Chat Trigger · AI Agent · Serper API · HTTP Request · JavaScript Code · Google Sheets

---

### Workflow 2 — Lead Email Enricher

**Trigger:** Executes after Workflow 1 completes (chained via Execute Workflow node)  
**Output:** Email column filled in the same Google Sheets

**How it works:**

Reads all rows from the Google Sheets output of Workflow 1 using a Google Sheets node. Passes them into a Loop Over Items node which processes each business one at a time. For each business, an HTTP Request node visits the website URL from the sheet and attempts to extract an email address from the page content using a JavaScript Code node with regex pattern matching. If an email is found, it writes it back to the correct row in Google Sheets using the row index.

**Key technical decisions:**

- Loop Over Items was necessary because HTTP requests to external websites need to be sequential — parallel requests caused rate limiting and incomplete responses
- Referenced upstream Code node data using `$('Code in JavaScript').first().json` to carry business name and row index through the loop without losing context
- Handles sites with no email gracefully — if regex finds nothing, the email cell stays blank and the loop continues rather than throwing an error
- Some businesses only list emails on their Facebook or Instagram pages, not their website — the enricher checks both the website field and the social link field from the scraper output

**Stack:** Google Sheets · Loop Over Items · HTTP Request · JavaScript Code · Regex

---

### Workflow 3 — AI Cold Email Sender

**Trigger:** Executes after Workflow 2 completes (chained via Execute Workflow node)  
**Output:** Personalised cold email sent via Gmail per enriched lead

**How it works:**

Reads the enriched Google Sheets, filters out rows where the email column is empty, then passes each remaining lead into a Groq AI HTTP Request node. The prompt is dynamically built using the business category field — a gym gets a different email than a café or a motel. Groq returns the generated email as a JSON response which is parsed using `response.generations[0][0].text` and then `JSON.parse`. The parsed subject and body are passed directly into a Gmail node which sends the email.

**Key technical decisions:**

- Dynamic prompting using `$('Code in JavaScript').item.json.category` means no templates — every email is written fresh for that business type by the AI
- Groq API used over OpenAI because it's free, fast, and LLaMA 3.3 70B produces good enough output for cold email copy
- Gmail OAuth was blocked on the new zenoautomate@gmail.com account (Google restricts OAuth on new accounts) — temporarily using a secondary account with a custom From Name set in the Gmail node to maintain professional appearance
- Filter node placed before the AI call to skip empty emails — avoids wasting Groq API calls on leads the enricher couldn't find emails for

**Stack:** Google Sheets · Filter · HTTP Request (Groq) · JavaScript Code · Gmail API

---

## AI Lead Management System

**Trigger:** Webhook (receives POST from any contact form)  
**Output:** Routed response via Gmail + Google Sheets + Telegram depending on lead score

**How it works:**

A webhook URL is embedded in a contact form. When someone submits it, n8n receives the payload and passes the lead data to Groq AI which scores it 1–10 based on the message content and contact details. An IF node splits the flow into three paths based on score range. High scores (7–10) trigger an instant Telegram alert to the business owner and a personalised Gmail reply. Medium scores (4–6) get logged to a Google Sheets CRM and a follow-up email is queued. Low scores (1–3) receive a polite auto-response. A separate Schedule Trigger fires at 9pm IST daily, reads the Sheet, counts total leads, finds the highest-scored one, and sends a summary to Telegram.

**Key technical decisions:**

- Railway server runs in America/New_York timezone — 9pm IST required cron `0 30 15 * * *` to account for the UTC offset
- Webhook node set to respond immediately with a 200 status before the AI processing starts — prevents form submission timeouts on slow Groq responses
- Groq prompt instructs the model to return only a JSON object with `score` and `reason` fields — response is parsed directly without needing to strip markdown fences

**Stack:** Webhook · Groq AI · IF Node · Gmail API · Google Sheets · Telegram · Schedule Trigger · Railway

---

## AI Gmail Assistant

**Trigger:** Gmail Trigger (polls every minute)  
**Output:** Telegram notification per email + auto-reply for urgent ones

**How it works:**

Gmail Trigger node polls the inbox every 60 seconds for new messages. Each email is passed to Groq AI with a classification prompt that returns one of four categories: urgent, inquiry, spam, follow-up. A Switch node routes based on the category. All emails get a formatted Telegram message with sender, subject, AI summary, and suggested reply. Urgent emails additionally trigger a Gmail reply node that sends an immediate response without any human input.

**Key technical decisions:**

- Gmail Trigger returns capitalized field keys (`From`, `Subject`, `Body`) — expressions must match this exactly or the node returns undefined
- Switch node used instead of multiple IF nodes to keep the canvas clean and make it easier to add new categories later
- Groq prompt uses strict JSON output format — classification, summary, and suggested reply all returned in one API call to avoid multiple requests per email

**Stack:** Gmail Trigger · Groq AI · Switch Node · Telegram Bot API · Gmail API · Railway

---

## Notes

- All workflows deployed on Railway using Docker
- n8n instance: `n8n-production-96112.up.railway.app`
- Screenshots and execution logs available on request
- Workflow JSON exports available on request for verified clients
