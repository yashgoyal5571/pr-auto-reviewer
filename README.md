# PR Auto-Reviewer

An n8n workflow that automatically reviews GitHub pull requests and posts structured feedback as a comment. Triggers on every PR open and every subsequent push. Maintains a single comment per PR using an idempotent update pattern.

Live demo sandbox: [pr-reviewer-test](https://github.com/yashgoyal5571/pr-reviewer-test)

---

## What it does

When a pull request is opened or updated, the workflow:

1. Fetches all changed files from the GitHub REST API
2. Runs 6 checks against the PR data
3. Checks if it has already commented on this PR
4. Posts a new comment or patches the existing one
5. Sends a color-coded embed to Discord

The GitHub comment is updated in place on every push. No duplicate comments accumulate across multiple commits to the same PR.

---

## Checks

| Check | Logic |
|---|---|
| Title format | Validates against Conventional Commits prefixes: feat, fix, docs, style, refactor, test, chore, perf, ci, build |
| PR size | Small under 50 lines, Medium 50 to 199, Large 200 to 499, Very Large 500 and above |
| Files changed | Per-file breakdown with filename, status, additions, and deletions |
| File types | All unique extensions present in the PR |
| Sensitive files | Flags package.json, package-lock.json, yarn.lock, .env, .env.local, .env.production, requirements.txt, Dockerfile, docker-compose.yml, .github/workflows |
| Idempotency | Scans existing comments for a hidden HTML marker, patches if found, posts if not |

Sensitive file detection overrides the size-based color. Any PR touching a sensitive file gets a red Discord embed regardless of size.

---

## Discord embed colors

| Color | Condition |
|---|---|
| Red | One or more sensitive files modified |
| Green | Small PR, no sensitive files |
| Yellow | Medium PR, no sensitive files |
| Orange | Large or Very Large PR, no sensitive files |
| Blue | No files detected (edge case default) |

---

## Workflow diagram

![PR Auto-Reviewer workflow canvas](./assets/workflow.png)

```
GitHub PR event (opened or synchronize)
              |
         Webhook (n8n)
              |
    Fetch PR Files — GitHub REST API
    Uses repository.full_name, fork-safe
              |
         Analyze PR
    Runs all checks, builds comment body
    and Discord embed payload
              |
      Get PR Comments — GitHub REST API
              |
       Check Idempotency
    Scans for <!-- pr-auto-reviewer-v1 -->
              |
     Comment Exists? (IF node)
      |               |
   true             false
      |               |
 Patch Comment    Post Comment
      |               |
      +-------+-------+
              |
       Notify Discord
       Continue On Fail enabled
```

---

## Example GitHub comment

```
🤖 PR Auto-Review

Title Format: ✅ Follows conventional commit format

Size: 🟢 Small (34 lines)
Changes:
- `dockerfile` (modified): +15/-0
- `package.json` (modified): +4/-9
- `test.md` (modified): +6/-0
Files Changed: 3
File Types: (no ext), .json, .md

⚠️ Sensitive files modified:
- package.json

---
Posted by your n8n PR Auto-Reviewer
Last updated: 2026-06-26 20:22
```

---

## Tech stack

| Tool | Role |
|---|---|
| n8n | Workflow orchestration, self-hosted via Docker Desktop |
| Docker Desktop | Runs n8n locally |
| ngrok | Exposes local n8n instance to GitHub webhooks |
| GitHub Webhooks | Triggers the workflow on PR events |
| GitHub REST API | Fetches changed files, posts and patches comments |
| Discord Webhooks | Sends color-coded embed notifications |

---

## Setup

### Prerequisites

- Docker Desktop installed and running
- n8n running locally via Docker
- ngrok installed
- A GitHub account and a target repository
- A Discord server with a webhook URL

### Step 1 — Import the workflow

Download `PR-Reviewer.json` from this repo and import it into your n8n instance via the workflow menu.

### Step 2 — Create credentials

In n8n, go to Credentials and create one credential:

**HTTP Header Auth** named `Github PAT v2`
- Header Name: `Authorization`
- Header Value: `Bearer YOUR_GITHUB_PAT`

Your GitHub PAT needs `repo` scope (full repository access) to read files and write comments.

Attach this credential to: Fetch PR Files, Get PR Comments, Post Comment, Patch Comment.

### Step 3 — Set the Discord webhook as an environment variable

Add this to your Docker environment:

```
DISCORD_PR_WEBHOOK_URL=https://discord.com/api/webhooks/your/webhook/url
```

The Notify Discord node reads `$env.DISCORD_PR_WEBHOOK_URL`. The URL is never stored in the workflow JSON.

### Step 4 — Start ngrok

```bash
ngrok http 5678
```

Copy the forwarding URL. It will look like `https://abc123.ngrok-free.app`.

### Step 5 — Configure the GitHub webhook

In your target GitHub repository, go to Settings → Webhooks → Add webhook.

- Payload URL: `https://your-ngrok-url/webhook/github-pr`
- Content type: `application/json`
- Events: select Pull requests only

### Step 6 — Activate the workflow

In n8n, activate the PR Auto-Reviewer workflow using the toggle in the top right. The webhook URL switches from test mode to production mode on activation.

### Step 7 — Test

Open a pull request in your target repository. The bot should comment within a few seconds.

---

## Project structure

```
pr-auto-reviewer/
├── PR-Reviewer.json       n8n workflow export
├── assets/
│   └── workflow.png       Workflow canvas screenshot
└── README.md
```

---

## Known limitations

- Runs locally on Docker Desktop. The bot is only active while the machine is on and ngrok is running.
- ngrok free tier generates a new URL on every restart. The GitHub webhook must be updated each time.
- No database. The bot has no memory across workflow restarts beyond what GitHub comment history provides.

---

## Author

Yash Goyal — [github.com/yashgoyal5571](https://github.com/yashgoyal5571)
