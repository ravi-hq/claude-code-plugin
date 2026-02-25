---
name: ravi-signup
description: End-to-end workflows for signing up, logging in, and completing 2FA using your Ravi identity
---

# Ravi Signup & Login Workflows

Step-by-step recipes that combine identity, vault, inbox, and email skills for common agent tasks.

## Sign up for a service

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

## Complete 2FA login

```bash
# After triggering 2FA on a website:
sleep 5
CODE=$(ravi inbox sms --unread --json | jq -r '.[0].preview' | grep -oE '[0-9]{4,8}' | head -1)
# Use $CODE to complete the login
```

## Extract a verification link from email

```bash
THREAD_ID=$(ravi inbox email --unread --json | jq -r '.[0].thread_id')
ravi inbox email "$THREAD_ID" --json | jq -r '.messages[].text_content' | grep -oE 'https?://[^ ]+'
```

## Send a follow-up email after signup

```bash
# 1. Compose a new email
ravi email compose --to "partner@example.com" --subject "Following up" \
  --body "<p>Hi, I just signed up for your service.</p><p>Looking forward to getting started!</p>" --json

# 2. Reply to an existing thread
MSG_ID=$(ravi message email --json | jq -r '.[0].id')
ravi email reply "$MSG_ID" --subject "Re: Welcome to Service" \
  --body "<p>Thanks for the welcome! I have a question about setup.</p>" --json
```

## Retrieve stored credentials for login

```bash
# Find the credential entry
ENTRY=$(ravi vault list --json | jq -r '.[] | select(.service_name == "example.com")')
UUID=$(echo "$ENTRY" | jq -r '.id')

# Get decrypted credentials
CREDS=$(ravi vault get "$UUID" --json)
USERNAME=$(echo "$CREDS" | jq -r '.username')
PASSWORD=$(echo "$CREDS" | jq -r '.password')
# Use $USERNAME and $PASSWORD to log in
```

## Tips

- **Poll, don't rush** — SMS/email delivery takes 2-10 seconds. Use `sleep 5` before checking.
- **Store credentials immediately** — create a vault entry during signup so you don't lose the password.
- **Use HTML in emails** — `--body` accepts HTML tags like `<p>`, `<h2>`, `<ul>`. No wrapper needed.
- **Rate limits apply to sending** — 60 emails/hour, 500/day. See `ravi-email-send` skill for details.
