# Gmail Listener Bot with OpenClaw Agent Integration

## Executive Summary

A Gmail listener bot is a Python application that monitors a Gmail inbox for new emails and automatically generates and sends AI-powered responses. When integrated with OpenClaw, you get a powerful multi-channel AI agent that can handle email alongside your other communication channels (Slack, Discord, WhatsApp, etc.) through a unified agent framework.

This guide covers building a production-ready Gmail auto-responder using Python, Google Gmail API, and OpenClaw as the agent brain.

---

## What Is a Gmail Listener Bot?

A Gmail listener bot is a background process that:

1. **Monitors** — Watches an inbox for new (unread) emails
2. **Processes** — Extracts sender, subject, body, and thread context
3. **Decides** — Determines if/how to respond (AI agent reasoning)
4. **Responds** — Sends a draft or fully automated reply
5. **Archives** — Marks as read, applies labels, maintains inbox hygiene

### Two Monitoring Approaches

| Approach | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **IMAP Polling** | Loop checking inbox every N seconds | Simple, no GCP setup | Not real-time, API overhead |
| **Gmail Push Notifications** | Google Cloud Pub/Sub webhooks | Real-time, scalable | More complex setup |

---

## Architecture Overview

```
[Gmail API / IMAP]
       ↓
[Python Listener Service]
       ↓
[OpenClaw Agent (via gog skill)]
       ↓
[AI Reasoning → Response Generation]
       ↓
[Gmail API → Send Reply]
```

### Alternative: LangChain/LangGraph as Agent Brain

```
[Gmail API]
    ↓
[Python Service]
    ↓
[LangGraph Agent (with memory)]
    ↓
[LLM (GPT-4o, Claude, DeepSeek)]
    ↓
[Gmail API → Send Reply]
```

---

## Prerequisites

### Google Cloud Setup

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Create a new project (e.g., "gmail-agent")
3. Enable **Gmail API**
4. Configure **OAuth Consent Screen**:
   - For personal Gmail → "External" + add yourself as test user
   - For Google Workspace → "Internal"
5. Add scopes:
   - `gmail.readonly` — Read emails
   - `gmail.send` — Send replies
   - `gmail.modify` — Mark read, archive, label
   - `gmail.compose` — Create drafts
6. Create **OAuth Client ID** credentials (Desktop app type)
7. Download `credentials.json`

### Python Environment

```bash
python3 -m venv gmail-bot
source gmail-bot/bin/activate

pip install \
  google-api-python-client \
  google-auth-oauthlib \
  google-auth-httplib2 \
  imaplib2 \
  smtplib \
  email \
  beautifulsoup4 \
  python-dotenv \
  loguru \
  schedule
```

---

## Implementation 1: IMAP Polling Bot

Simpler approach using IMAP. Good for personal use or testing.

```python
"""
Gmail Auto-Responder with IMAP polling
"""
import os
import time
import imaplib
import email
from email.header import decode_header
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
import smtplib
from dotenv import load_dotenv
from openai import OpenAI
import loguru as logger

load_dotenv()

# ── Configuration ──────────────────────────────────────────────
EMAIL = os.getenv("GMAIL_EMAIL")
APP_PASSWORD = os.getenv("GMAIL_APP_PASSWORD")
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
CHECK_INTERVAL = 30  # seconds
ALLOWED_DOMAINS = ["@gmail.com"]  # Restrict to specific senders (optional)

client = OpenAI(api_key=OPENAI_API_KEY)


def decode_email_header(header):
    """Decode email subject/sender headers properly."""
    decoded_parts = decode_header(header)
    result = []
    for part, charset in decoded_parts:
        if isinstance(part, bytes):
            charset = charset or "utf-8"
            result.append(part.decode(charset, errors="replace"))
        else:
            result.append(part)
    return "".join(result)


def get_email_body(msg):
    """Extract plain text body from email message."""
    body = ""
    if msg.is_multipart():
        for part in msg.walk():
            content_type = part.get_content_type()
            if content_type == "text/plain":
                charset = part.get_content_charset() or "utf-8"
                body = part.get_payload(decode=True).decode(charset, errors="replace")
                break
    else:
        charset = msg.get_content_charset() or "utf-8"
        body = msg.get_payload(decode=True).decode(charset, errors="replace")
    return body.strip()


def fetch_unread_emails():
    """Connect to Gmail via IMAP and fetch unread emails."""
    try:
        mail = imaplib.IMAP4_SSL("imap.gmail.com")
        mail.login(EMAIL, APP_PASSWORD)
        mail.select("inbox")

        # Search for unread emails
        status, messages = mail.search(None, "UNSEEN")
        email_ids = messages[0].split()

        emails = []
        for email_id in email_ids:
            _, msg_data = mail.fetch(email_id, "(RFC822)")
            raw_email = msg_data[0][1]
            msg = email.message_from_bytes(raw_email)

            sender = decode_email_header(msg.get("From", ""))
            subject = decode_email_header(msg.get("Subject", ""))
            body = get_email_body(msg)
            thread_id = msg.get("Thread-Id", "")

            emails.append({
                "id": email_id,
                "sender": sender,
                "subject": subject,
                "body": body,
                "thread_id": thread_id,
                "message_obj": msg,
            })

        mail.logout()
        return emails
    except Exception as e:
        logger.error(f"Failed to fetch emails: {e}")
        return []


def generate_auto_reply(sender: str, subject: str, body: str) -> str:
    """Use OpenAI to generate a context-aware auto-reply."""
    prompt = f"""You are an AI email assistant helping the owner of this inbox.
Generate a professional, concise auto-reply email.

Sender: {sender}
Subject: {subject}
Email Body:
{body[:2000]}

Write a helpful reply that:
- Acknowledges receipt of the email
- Is warm but professional
- Provides useful information if applicable
- Keeps it concise (2-4 sentences)
- Uses a friendly but professional tone

Only output the email body text, no subject line or headers."""

    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}],
        max_tokens=300,
        temperature=0.7,
    )

    return response.choices[0].message.content.strip()


def send_email(to_email: str, subject: str, body: str, thread_ref: str = None):
    """Send email via Gmail SMTP."""
    try:
        msg = MIMEMultipart()
        msg["From"] = EMAIL
        msg["To"] = to_email
        msg["Subject"] = subject

        # Add threading headers if available
        if thread_ref:
            msg["In-Reply-To"] = thread_ref
            msg["References"] = thread_ref

        msg.attach(MIMEText(body, "plain"))

        with smtplib.SMTP_SSL("smtp.gmail.com", 465) as server:
            server.login(EMAIL, APP_PASSWORD)
            server.sendmail(EMAIL, to_email, msg.as_string())

        logger.info(f"Replied to {to_email}: {subject}")
    except Exception as e:
        logger.error(f"Failed to send email: {e}")


def should_respond(sender: str) -> bool:
    """Filter: decide if we should respond to this sender."""
    # Skip auto-responders and newsletters
    skip_patterns = [
        "no-reply", "noreply", "notification", "newsletter",
        "donotreply", "auto-", "no.reply",
    ]
    sender_lower = sender.lower()
    return not any(p in sender_lower for p in skip_patterns)


def mark_as_read(email_id, mail):
    """Mark email as read after processing."""
    try:
        mail.select("inbox")
        mail.store(email_id, "+FLAGS", "\\Seen")
    except Exception as e:
        logger.warning(f"Could not mark as read: {e}")


def main():
    logger.info("Gmail Auto-Responder Bot started")
    logger.info(f"Checking every {CHECK_INTERVAL} seconds")

    while True:
        emails = fetch_unread_emails()
        logger.info(f"Found {len(emails)} unread emails")

        for email_data in emails:
            sender = email_data["sender"]
            subject = email_data["subject"]
            body = email_data["body"]

            if not should_respond(sender):
                logger.info(f"Skipping non-responsive sender: {sender}")
                continue

            logger.info(f"Processing email from {sender}: {subject}")

            # Generate AI reply
            reply_body = generate_auto_reply(sender, subject, body)

            # Extract email address from sender
            import re
            email_match = re.search(r"<(.+?)>", sender)
            to_email = email_match.group(1) if email_match else sender.split()[0]

            # Send the reply
            send_email(
                to_email=to_email,
                subject=f"Re: {subject}",
                body=reply_body,
            )

        time.sleep(CHECK_INTERVAL)


if __name__ == "__main__":
    main()
```

### .env file

```bash
GMAIL_EMAIL=your-email@gmail.com
GMAIL_APP_PASSWORD=your-16-char-app-password
OPENAI_API_KEY=sk-...
```

---

## Implementation 2: Gmail API + OpenClaw Agent

Production approach using the official Gmail API and OpenClaw as the agent brain via the `gog` skill.

### Setup OpenClaw gog Skill

```bash
# In your OpenClaw config, enable the gog skill:
npx playbooks add skill openclaw/skills --skill gmail
```

Or configure manually:

```yaml
# openclaw config
skills:
  gog:
    enabled: true
    credentials_path: /path/to/credentials.json
    token_path: /path/to/token.json
    scopes:
      - gmail.readonly
      - gmail.send
      - gmail.modify
```

### Listener Service with OpenClaw

```python
"""
Gmail Listener with OpenClaw Agent Integration
Uses Gmail API for reliable email access + OpenClaw for AI reasoning
"""
import os
import time
import json
import loguru as logger
from datetime import datetime, timedelta
from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build

# ── Gmail API Setup ──────────────────────────────────────────────
SCOPES = [
    "https://www.googleapis.com/auth/gmail.readonly",
    "https://www.googleapis.com/auth/gmail.send",
    "https://www.googleapis.com/auth/gmail.modify",
]

CREDENTIALS_PATH = os.path.expanduser("~/.config/gmail-agent/credentials.json")
TOKEN_PATH = os.path.expanduser("~/.config/gmail-agent/token.json")


def get_gmail_service():
    """Authenticate and return Gmail API service."""
    creds = None
    if os.path.exists(TOKEN_PATH):
        creds = Credentials.from_authorized_user_file(TOKEN_PATH, SCOPES)

    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())
        else:
            flow = InstalledAppFlow.from_client_secrets_file(CREDENTIALS_PATH, SCOPES)
            creds = flow.run_local_server(port=0)

        with open(TOKEN_PATH, "w") as token:
            token.write(creds.to_json())

    return build("gmail", "v1", credentials=creds)


def get_unread_emails(service, max_results=10):
    """Fetch unread emails from inbox."""
    try:
        results = service.users().messages().list(
            userId="me",
            q="is:unread newer_than:1d",
            maxResults=max_results
        ).execute()

        messages = results.get("messages", [])
        emails = []

        for msg_ref in messages:
            msg = service.users().messages().get(
                userId="me",
                id=msg_ref["id"],
                format="full"
            ).execute()

            headers = {h["name"]: h["value"] for h in msg["payload"]["headers"]}
            snippet = msg.get("snippet", "")

            # Get full body
            body = ""
            if "parts" in msg["payload"]:
                for part in msg["payload"]["parts"]:
                    if part["mimeType"] == "text/plain":
                        data = part["body"].get("data", "")
                        if data:
                            import base64
                            body = base64.urlsafe_b64decode(data).decode("utf-8", errors="replace")
                            break

            emails.append({
                "id": msg["id"],
                "thread_id": msg["threadId"],
                "sender": headers.get("From", ""),
                "subject": headers.get("Subject", ""),
                "date": headers.get("Date", ""),
                "body": body or snippet,
            })

        return emails
    except Exception as e:
        logger.error(f"Failed to fetch emails: {e}")
        return []


def mark_as_read(service, msg_id):
    """Mark message as read."""
    try:
        service.users().messages().modify(
            userId="me",
            id=msg_id,
            body={"removeLabelIds": ["UNREAD"]}
        ).execute()
    except Exception as e:
        logger.warning(f"Could not mark as read: {e}")


def send_reply(service, to_email: str, subject: str, body: str, thread_id: str = None):
    """Send email reply."""
    from email.mime.text import MIMEText
    import base64

    message = MIMEText(body)
    message["To"] = to_email
    message["Subject"] = subject
    if thread_id:
        message["In-Reply-To"] = thread_id
        message["References"] = thread_id

    raw = base64.urlsafe_b64encode(message.as_bytes()).decode("utf-8")

    try:
        service.users().messages().send(
            userId="me",
            body={"raw": raw, "threadId": thread_id}
        ).execute()
        logger.info(f"Sent reply to {to_email}")
    except Exception as e:
        logger.error(f"Failed to send reply: {e}")


def query_openclaw(prompt: str) -> str:
    """Send email to OpenClaw agent for AI-powered response."""
    import httpx

    # OpenClaw API endpoint (adjust port/host as needed)
    OLLAMA_BASE = os.getenv("OLLAMA_BASE_URL", "http://localhost:11434")
    OLLAMA_MODEL = os.getenv("OLLAMA_MODEL", "llama3.3")

    try:
        response = httpx.post(
            f"{OLLAMA_BASE}/api/generate",
            json={
                "model": OLLAMA_MODEL,
                "prompt": prompt,
                "stream": False,
                "options": {
                    "temperature": 0.7,
                    "num_predict": 300,
                }
            },
            timeout=30.0,
        )
        return response.json().get("response", "").strip()
    except Exception as e:
        logger.error(f"OpenClaw query failed: {e}")
        return "Thank you for your email. I will get back to you shortly."


def generate_smart_reply(email_data: dict) -> str:
    """Generate AI-powered auto-reply using OpenClaw."""
    prompt = f"""You are an AI assistant managing email for the owner of this inbox.
Generate a professional auto-reply email.

=== EMAIL DETAILS ===
From: {email_data['sender']}
Subject: {email_data['subject']}
Body: {email_data['body'][:1500]}
===

Requirements:
- Warm, professional tone
- 2-4 sentences maximum
- Acknowledge receipt
- If a question was asked, provide a helpful answer
- If it's a meeting request, express interest and suggest alternatives
- If it's urgent, mention you'll respond fully soon
- Keep it concise and natural

Auto-reply body:"""

    return query_openclaw(prompt)


def main():
    logger.info("Gmail OpenClaw Agent started")

    service = get_gmail_service()
    logger.info("Gmail API connected")

    while True:
        emails = get_unread_emails(service)
        logger.info(f"Found {len(emails)} unread emails")

        for email_data in emails:
            sender = email_data["sender"]
            subject = email_data["subject"]

            logger.info(f"Processing: {sender} — {subject}")

            # Generate AI reply
            reply_body = generate_smart_reply(email_data)

            # Extract email address
            import re
            email_match = re.search(r"<(.+?)>", sender)
            to_email = email_match.group(1) if email_match else sender.split()[0]

            # Send reply
            send_reply(
                service=service,
                to_email=to_email,
                subject=f"Re: {subject}",
                body=reply_body,
                thread_id=email_data["thread_id"],
            )

            # Mark as read
            mark_as_read(service, email_data["id"])

        time.sleep(30)  # Check every 30 seconds


if __name__ == "__main__":
    main()
```

---

## Implementation 3: LangGraph Agent (Alternative Agent Brain)

For teams wanting a dedicated email agent framework without OpenClaw:

```python
"""
Gmail Auto-Responder using LangGraph
Stateful multi-step reasoning with memory
"""
import os
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, SystemMessage
from googleapiclient.discovery import build
from google.auth.transport.requests import Request
from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
import base64

# ── Gmail Setup (same as above) ──────────────────────────────────
SCOPES = ["https://www.googleapis.com/auth/gmail.readonly",
          "https://www.googleapis.com/auth/gmail.send",
          "https://www.googleapis.com/auth/gmail.modify"]

# ── Agent State ──────────────────────────────────────────────────
class AgentState(TypedDict):
    email: dict
    analysis: str
    response: str
    decision: str  # "reply" | "escalate" | "ignore"


# ── LangGraph Nodes ──────────────────────────────────────────────
llm = ChatOpenAI(model="gpt-4o", temperature=0.7)


def analyze_email(state: AgentState) -> AgentState:
    """Step 1: Analyze the email content."""
    email = state["email"]

    prompt = f"""Analyze this email and determine the best course of action.

Email from: {email['sender']}
Subject: {email['subject']}
Body: {email['body'][:1000]}

Respond with:
1. Summary (1 sentence)
2. Intent (question, request, notification, meeting, spam)
3. Urgency (low/medium/high)
4. Action: reply | escalate | ignore
5. Reasoning

Format: Summary: ... Intent: ... Urgency: ... Action: ..." ""

    response = llm.invoke([HumanMessage(content=prompt)])
    state["analysis"] = response.content
    return state


def decide_action(state: AgentState) -> AgentState:
    """Step 2: Decide what to do based on analysis."""
    analysis = state["analysis"].lower()

    if "action: escalate" in analysis:
        state["decision"] = "escalate"
    elif "action: ignore" in analysis:
        state["decision"] = "ignore"
    else:
        state["decision"] = "reply"

    return state


def generate_response(state: AgentState) -> AgentState:
    """Step 3: Generate the email response."""
    email = state["email"]

    prompt = f"""Draft a professional auto-reply email.

Original email:
From: {email['sender']}
Subject: {email['subject']}
Body: {email['body'][:1500]}

Analysis: {state['analysis']}

Draft a concise, warm, professional reply (3-5 sentences):"""

    response = llm.invoke([HumanMessage(content=prompt)])
    state["response"] = response.content
    return state


def escalate(state: AgentState) -> AgentState:
    """Mark for human review."""
    state["response"] = "[ESCALATE] Flagged for human review"
    return state


# ── Build Graph ──────────────────────────────────────────────────
workflow = StateGraph(AgentState)

workflow.add_node("analyze", analyze_email)
workflow.add_node("decide", decide_action)
workflow.add_node("reply", generate_response)
workflow.add_node("escalate", escalate)

workflow.set_entry_point("analyze")
workflow.add_edge("analyze", "decide")

workflow.add_conditional_edges(
    "decide",
    lambda state: state["decision"],
    {
        "reply": "reply",
        "escalate": "escalate",
        "ignore": END,
    }
)

workflow.add_edge("reply", END)
workflow.add_edge("escalate", END)

graph = workflow.compile()


def process_email(email_data: dict) -> str:
    """Run the agent graph on a single email."""
    initial_state = {"email": email_data, "analysis": "", "response": "", "decision": ""}
    final_state = graph.invoke(initial_state)
    return final_state["response"]
```

---

## Production Considerations

### Security

- **Never commit credentials** — Use `.env` files + `.gitignore`
- **OAuth tokens** — Store in encrypted config, set file permissions `600`
- **App Passwords** (IMAP) — Generate separate from main password at [myaccount.google.com](https://myaccount.google.com)
- **Rate limiting** — Gmail allows ~100 emails/minute; add delays
- **Avoid reply loops** — Skip auto-replies to no-reply addresses
- **Thread awareness** — Always include `In-Reply-To` and `References` headers

### Gmail API vs IMAP

| Factor | Gmail API | IMAP |
|--------|----------|------|
| OAuth | Required | App Password |
| Speed | Faster | Slower |
| Reliability | Official | Can be flaky |
| Setup complexity | Higher | Lower |
| Gmail-specific features | Full access | Limited |
| Cost | Free (quota) | Free |

### Monitoring

```python
# Add health checks
from loguru import logger
import psutil

logger.add("gmail-agent.log", rotation="10 MB")

def health_check():
    cpu = psutil.cpu_percent()
    memory = psutil.virtual_memory().percent
    logger.info(f"Health: CPU={cpu}% MEM={memory}%")
    if cpu > 90 or memory > 90:
        logger.warning("High resource usage!")
```

### Deployment

**Option 1: Systemd service (Linux/VPS)**
```ini
[Unit]
Description=Gmail Auto-Responder
After=network.target

[Service]
Type=simple
User=youruser
WorkingDirectory=/home/youruser/gmail-bot
ExecStart=/home/youruser/gmail-bot/venv/bin/python main.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Option 2: Docker**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "main.py"]
```

**Option 3: Cloud Run (for push notifications)**
- More complex but handles real-time Gmail push notifications via Pub/Sub
- Scales to zero, pay per use

---

## OpenClaw Integration Benefits

When you use OpenClaw as the agent brain instead of a standalone LLM:

- **Multi-channel** — Email agent works alongside Slack, Discord, WhatsApp
- **Skills** — Reuse existing OpenClaw skills (calendar, notes, search)
- **Memory** — OpenClaw's memory system tracks email context across sessions
- **Scheduling** — Built-in cron for follow-up reminders
- **Security** — Credential encryption, access controls
- **Monitoring** — Unified logging and error handling

Example OpenClaw skill call from the listener:
```python
# When escalation is needed, notify via OpenClaw's alert system
import httpx

httpx.post(
    "http://localhost:18789/api/notify",
    json={
        "channel": "slack",
        "message": f"⚠️ Email needs your attention:\n{subject}\nFrom: {sender}",
    }
)
```

---

## Key Takeaways

1. **Start simple** — Use IMAP polling + OpenAI for quick prototype
2. **Production needs OAuth** — Gmail API is more reliable than IMAP
3. **OpenClaw wins** — Unified agent framework handles email + all other channels
4. **Thread carefully** — Always include threading headers to avoid reply loops
5. **Filter aggressively** — Skip newsletters, no-reply, auto-notifications
6. **Test safely** — Start with draft mode before enabling auto-send
7. **Monitor continuously** — Set up health checks and alerting

---

## Sources

- [Gmail API Documentation](https://developers.google.com/gmail/api)
- [Google Cloud Pub/Sub Push Notifications](https://developers.google.com/workspace/gmail/api/guides/push)
- [OpenClaw Setup Guide](https://openclawsetup.dev/blog/connect-ai-agent-to-gmail)
- [AI-WORKERS1729/GmailAI](https://github.com/mytestlab1729/gmailai)
- [LangGraph Documentation](https://langchain.readthedocs.io/langgraph/)
- [Google OAuth 2.0 Guide](https://developers.google.com/identity/protocols/oauth2)
- [CrewAI Email Agent Example](https://github.com/UmaiRyden/Email-Auto-Responder)
