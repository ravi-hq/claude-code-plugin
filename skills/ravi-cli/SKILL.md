---
name: ravi-cli
description: Use the Ravi CLI to manage agent identities, receive SMS/email, and store credentials
---

# Ravi CLI — Identity for AI Agents

You have access to `ravi`, a CLI that gives you your own phone number, email address, and credential vault. Use it to sign up for services, receive OTPs, complete 2FA, and store passwords — all autonomously.

## Prerequisites

Verify you're authenticated before using any command:

```bash
ravi auth status --json
```

If `"authenticated": false`, tell the user to run `ravi auth login` (requires browser interaction — you cannot do this yourself).

## Your Identity

Get the email and phone number assigned to you:

```bash
# Your email address (use this for signups)
ravi get email --json
# → {"id": 1, "email": "janedoe@ravi.app", "created_dt": "..."}

# Your phone number (use this for SMS verification)
ravi get phone --json
# → {"id": 1, "phone_number": "+15551234567", "provider": "twilio", "created_dt": "..."}

# The human who owns this account
ravi get owner --json
# → {"first_name": "Jane", "last_name": "Doe"}
```

## Receiving Messages

After triggering a verification (signup form, 2FA, etc.), wait a few seconds then check your inbox.

### SMS (OTPs, verification codes)

```bash
# List SMS conversations (grouped by sender)
ravi inbox sms --json

# Only conversations with unread messages
ravi inbox sms --unread --json

# View a specific conversation (all messages)
ravi inbox sms <conversation_id> --json
# conversation_id format: {phone_id}_{from_number}, e.g. "1_+15559876543"
```

**JSON shape — conversation list:**
```json
[{
  "conversation_id": "1_+15559876543",
  "from_number": "+15559876543",
  "phone_number": "+15551234567",
  "preview": "Your code is 847291",
  "message_count": 3,
  "unread_count": 1,
  "latest_message_dt": "2026-02-25T10:30:00Z"
}]
```

**JSON shape — conversation detail:**
```json
{
  "conversation_id": "1_+15559876543",
  "from_number": "+15559876543",
  "messages": [
    {"id": 42, "body": "Your code is 847291", "direction": "incoming", "is_read": false, "created_dt": "..."}
  ]
}
```

### Email (verification links, confirmations)

```bash
# List email threads
ravi inbox email --json

# Only threads with unread messages
ravi inbox email --unread --json

# View a specific thread (all messages with full content)
ravi inbox email <thread_id> --json
```

**JSON shape — thread detail:**
```json
{
  "thread_id": "abc123",
  "subject": "Verify your email",
  "messages": [
    {
      "id": 10,
      "from_email": "noreply@example.com",
      "to_email": "janedoe@ravi.app",
      "subject": "Verify your email",
      "text_content": "Click here to verify: https://example.com/verify?token=xyz",
      "direction": "incoming",
      "is_read": false,
      "created_dt": "..."
    }
  ]
}
```

### Individual Messages (flat, not grouped)

Use these when you need messages by ID rather than by conversation:

```bash
ravi message sms --json              # All SMS messages
ravi message sms --unread --json     # Unread only
ravi message sms <message_id> --json # Specific message

ravi message email --json              # All email messages
ravi message email --unread --json     # Unread only
ravi message email <message_id> --json # Specific message
```

## Sending Emails

### Compose a new email

```bash
ravi email compose --to "recipient@example.com" --subject "Subject" --body "<p>HTML content</p>" --json
```

**Flags:**
- `--to` (required): Recipient email address
- `--subject` (required): Email subject line
- `--body` (required): Email body (HTML supported — use tags like `<p>`, `<h2>`, `<ul>` for formatting)
- `--cc`: CC recipients (comma-separated)
- `--bcc`: BCC recipients (comma-separated)
- `--attach`: File path to attach (can be repeated for multiple files)

**Example with HTML formatting and attachment:**
```bash
ravi email compose --to "user@example.com" --subject "Monthly Report" \
  --body "<h2>Monthly Report</h2><p>Key findings:</p><ul><li>Revenue up 15%</li><li>Churn down 3%</li></ul>" \
  --attach report.pdf --json
```

### Reply to an email

```bash
# Reply to sender only
ravi email reply <message_id> --subject "Re: Original Subject" --body "<p>Reply content</p>" --json

# Reply to all recipients
ravi email reply-all <message_id> --subject "Re: Original Subject" --body "<p>Reply content</p>" --json
```

**Flags:** `--subject` (required), `--body` (required), `--attach` (optional)

**Note:** The subject must be provided because the original is E2E encrypted on the server.

### Attachments

Attachments are uploaded automatically when you use `--attach`. The CLI:
1. Validates the file (blocked extensions like `.exe` rejected instantly)
2. Requests a presigned upload URL from the server
3. Uploads the file directly to cloud storage
4. Includes the attachment UUID in the email

**Blocked extensions:** `.exe`, `.dll`, `.bat`, `.cmd`, `.msi`, `.iso`, `.dmg`, `.apk`, and other dangerous file types. Developer files (`.py`, `.sh`, `.js`, `.rb`) are allowed.

**Max size:** 10 MB per attachment.

### Rate Limits

Email sending is rate-limited per user account:
- **60 emails/hour** and **500 emails/day**
- **200 attachment uploads/hour**

On hitting a rate limit, you'll get a 429 error with a `retry_after_seconds` value. Wait that many seconds before retrying.

**Best practices for agents:**
- Avoid tight loops of email sends — batch work where possible
- On 429: parse `retry_after_seconds` from the error, wait, then retry
- For bulk operations, add a 1-2 second delay between sends

## Credential Vault

Store and retrieve passwords for services you sign up for. All fields are E2E encrypted.

```bash
# Create entry (auto-generates password if --password not given)
ravi vault create example.com --json
ravi vault create example.com --username "me@ravi.app" --password 'S3cret!' --json

# List all entries
ravi vault list --json

# Retrieve (decrypted)
ravi vault get <uuid> --json

# Update
ravi vault edit <uuid> --password 'NewPass!' --json

# Delete
ravi vault delete <uuid> --json

# Generate a password without storing it
ravi vault generate --length 24 --json
# → {"password": "xK9#mL2..."}
```

**Create flags:** `--username`, `--password`, `--notes`, `--generate`, `--length` (default 16), `--no-special`, `--no-digits`, `--exclude-chars`

## Common Workflows

### Sign up for a service

```bash
# 1. Get your credentials
EMAIL=$(ravi get email --json | jq -r '.email')
PHONE=$(ravi get phone --json | jq -r '.phone_number')

# 2. Use $EMAIL and $PHONE in the signup form

# 3. Generate and store a password
CREDS=$(ravi vault create example.com --username "$EMAIL" --json)
PASSWORD=$(echo "$CREDS" | jq -r '.password')
# Use $PASSWORD in the signup form

# 4. Wait for verification
sleep 5
ravi inbox sms --unread --json   # Check for SMS OTP
ravi inbox email --unread --json # Check for email verification
```

### Extract an OTP code from SMS

```bash
# Get unread SMS, extract 4-8 digit codes
ravi inbox sms --unread --json | jq -r '.[].preview' | grep -oE '[0-9]{4,8}'
```

### Extract a verification link from email

```bash
# Get the latest unread email thread, pull URLs from text content
THREAD_ID=$(ravi inbox email --unread --json | jq -r '.[0].thread_id')
ravi inbox email "$THREAD_ID" --json | jq -r '.messages[].text_content' | grep -oE 'https?://[^ ]+'
```

### Complete 2FA login

```bash
# After triggering 2FA on a website:
sleep 5
CODE=$(ravi inbox sms --unread --json | jq -r '.[0].preview' | grep -oE '[0-9]{4,8}' | head -1)
# Use $CODE to complete the login
```

### Send a follow-up email after signup

```bash
# 1. Compose a welcome email
ravi email compose --to "partner@example.com" --subject "Following up" \
  --body "<p>Hi, I just signed up for your service using my Ravi identity.</p><p>Looking forward to getting started!</p>" --json

# 2. Reply to an existing thread
MSG_ID=$(ravi message email --json | jq -r '.[0].id')
ravi email reply "$MSG_ID" --subject "Re: Welcome to Service" \
  --body "<p>Thanks for the welcome! I have a question about setup.</p>" --json
```

## Important Notes

- **Always use `--json`** — all commands support it. Human-readable output is not designed for parsing.
- **Poll, don't rush** — SMS/email delivery takes 2-10 seconds. Use `sleep 5` before checking.
- **Auth is automatic** — token refresh happens transparently. If you get auth errors, ask the user to re-login.
- **E2E encryption is transparent** — the CLI encrypts vault fields before sending and decrypts on retrieval. You see plaintext.
- **Domain cleaning** — `ravi vault create` auto-cleans URLs to base domains (e.g., `https://mail.google.com/inbox` becomes `google.com`).
- **HTML email bodies** — The `--body` flag accepts HTML. Use tags for formatting: `<p>`, `<h2>`, `<ul>`, `<a href="...">`. No `<html>` or `<body>` wrapper needed.
- **Rate limits** — 60 emails/hour, 500/day, 200 attachment uploads/hour. On 429 errors, wait `retry_after_seconds` then retry.
