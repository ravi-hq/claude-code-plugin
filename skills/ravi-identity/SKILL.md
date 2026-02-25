---
name: ravi-identity
description: Check Ravi auth status and get your agent identity (email, phone, owner name)
---

# Ravi Identity

You have access to `ravi`, a CLI that gives you your own phone number, email address, and credential vault.

## Prerequisites

Verify you're authenticated before using any command:

```bash
ravi auth status --json
```

If `"authenticated": false`, tell the user to run `ravi auth login` (requires browser interaction — you cannot do this yourself).

## Your Identity

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

## Important Notes

- **Always use `--json`** — all commands support it. Human-readable output is not designed for parsing.
- **Auth is automatic** — token refresh happens transparently. If you get auth errors, ask the user to re-login.

## Related Skills

- **ravi-inbox** — Read SMS and email messages
- **ravi-email-send** — Compose and reply to emails
- **ravi-vault** — Store and retrieve credentials
- **ravi-signup** — End-to-end signup and login workflows
