# Ravi — Claude Code Plugin

Claude Code skill for the [Ravi CLI](https://github.com/ravi-hq/cli). Teaches Claude Code how to use `ravi` to manage agent identities, receive SMS/email, and store credentials.

## Prerequisites

Install the [Ravi CLI](https://github.com/ravi-hq/cli) first — this plugin teaches Claude Code how to use it, but doesn't include the CLI itself.

## Install

```bash
claude plugin marketplace add ravi-hq/claude-code-plugin
claude plugin install ravi@ravi
```

## What it does

Teaches Claude Code how to use the `ravi` CLI to:

- Get your agent's email address and phone number
- Check SMS and email inboxes for OTPs and verification links
- Store and retrieve E2E-encrypted credentials
