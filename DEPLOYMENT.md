# Deployment Guide — PR Auto-Reviewer

This workflow runs entirely on your own infrastructure. Your GitHub PAT, Discord webhook URL, Slack token, Telegram bot token, and all other credentials stay in your environment. They are never shared with anyone.

---

## Prerequisites (all methods)

Before choosing a deployment method, prepare:

**GitHub Personal Access Token**
- GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
- Generate new token (classic), set expiration, select scope: `repo` (full)
- Copy the token immediately — you cannot view it again

**Platform credentials (only for platforms you enable)**

Discord: Create a webhook in your server → channel settings → Integrations → Webhooks → New Webhook → copy URL

Slack: Create a Slack App at api.slack.com/apps, add `chat:write` scope under OAuth, install to workspace, copy Bot Token. Also get the channel ID (right-click channel → View channel details → copy ID at the bottom).

Telegram: Message @BotFather on Telegram → /newbot → follow prompts → copy the bot token. Get your chat ID by messaging @userinfobot.

---

## Method 1 — Local machine + ngrok (testing only)

n8n runs on your laptop. ngrok creates a temporary public HTTPS tunnel to it. Suitable only for personal testing or live demos. Not suitable for anything that needs to stay up when your machine is off or sleeping.

**ngrok free tier (verified June 2026):**
- 1 static subdomain that persists across restarts: `yourname.ngrok-free.app`
- 1 simultaneous tunnel
- No bandwidth limit on free tier
- Sessions do not expire automatically on the free static domain

**Steps:**

1. Ensure n8n is running via Docker Desktop on port 5678
2. Sign up at ngrok.com, install the CLI, authenticate: `ngrok config add-authtoken YOUR_TOKEN`
3. Claim your free static domain at ngrok.com/dashboard → Domains
4. Start the tunnel: `ngrok http --domain=yourname.ngrok-free.app 5678`
5. Your webhook URL: `https://yourname.ngrok-free.app/webhook/github-pr`
6. Keep the terminal open — closing it kills the tunnel

**Limitation:** Requires your laptop to be on and the terminal running. If your machine sleeps, GitHub webhook deliveries fail until n8n is reachable again. GitHub retries failed deliveries for up to 3 days.

---

## Method 2 — Cloudflare Tunnel (permanent URL, free, no session limits)

Cloudflare Tunnel creates a persistent outbound connection from your machine to Cloudflare's edge network. No session timeouts, no bandwidth caps, free forever. Requires a domain managed by Cloudflare DNS.

**Requirements:**
- A domain name (~$10/year via Cloudflare Registrar or any registrar)
- Free Cloudflare account
- `cloudflared` daemon installed

**Steps:**

1. Add your domain to Cloudflare and set its nameservers to Cloudflare's
2. Install cloudflared from developers.cloudflare.com/cloudflare-one/connections/connect-apps
3. Authenticate: `cloudflared tunnel login`
4. Create a named tunnel: `cloudflared tunnel create pr-reviewer`
5. In the tunnel config file, route traffic to `localhost:5678`
6. Install as a system service: `cloudflared service install`
7. Your webhook URL: `https://yourdomain.com/webhook/github-pr`

The tunnel runs as a background service and starts automatically on reboot. URL never changes.

**Limitation:** Still requires your machine to be on and n8n running. Laptop sleeping = workflow offline.

---

## Method 3 — Self-hosted n8n on a VPS (recommended for production)

n8n runs 24/7 on a cloud server. No dependency on your local machine. Permanent public HTTPS URL with no tunneling.

**Recommended servers and cost (verified June 2026):**

| Provider | Spec | Cost |
|---|---|---|
| Hetzner CX22 | 2 vCPU, 4 GB RAM, 40 GB NVMe, 20 TB bandwidth | ~€4.35/month |
| DigitalOcean Basic | 1 vCPU, 1 GB RAM, 25 GB SSD | $6/month |
| Vultr Cloud Compute | 1 vCPU, 1 GB RAM, 25 GB SSD | $6/month |

Hetzner CX22 is the best value. 4 GB RAM is comfortable for n8n with room to spare.

**What you install:**
- Docker + Docker Compose (to run n8n)
- Nginx (reverse proxy to handle HTTPS)
- Certbot (free SSL certificate via Let's Encrypt)

**Steps:**

1. Create a VPS, select Ubuntu 24.04 LTS
2. SSH in: `ssh root@YOUR_IP`
3. Install Docker: `curl -fsSL https://get.docker.com | sh`
4. Create `docker-compose.yml`:

```yaml
version: '3.8'
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=yourdomain.com
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://yourdomain.com/
      - DISCORD_PR_WEBHOOK_URL=your_discord_webhook_url
      - SLACK_PR_CHANNEL_ID=your_slack_channel_id
      - TELEGRAM_BOT_TOKEN=your_telegram_bot_token
      - TELEGRAM_CHAT_ID=your_telegram_chat_id
      - GITHUB_WEBHOOK_SECRET=your_generated_secret
      - NODE_FUNCTION_ALLOW_BUILTIN=crypto
    volumes:
      - n8n_data:/home/node/.n8n
volumes:
  n8n_data:
```

5. Start n8n: `docker compose up -d`
6. Point your domain's A record to the VPS IP
7. Install Nginx, configure a reverse proxy to port 5678, get a free SSL cert via Certbot
8. Your webhook URL: `https://yourdomain.com/webhook/github-pr`

For the Nginx + Certbot configuration, search "n8n self-hosted nginx ssl" — the n8n docs have an exact guide for this step.

---

## Method 4 — n8n Cloud (no infrastructure to manage)

n8n hosts everything. You log in, import the workflow, add credentials, and activate. No servers, no Docker, no SSL.

**Pricing (verified June 2026):**
- Starter: €24/month — 2,500 workflow executions/month
- Pro: €60/month — 10,000 executions/month
- 14-day free trial, no permanent free tier

**Execution math for this workflow:**
This workflow is event-driven and only executes when a PR is opened or a commit is pushed to an open PR. A repo with 100 PR events per month uses 100 executions. The Starter plan is sufficient for any small-to-medium team.

**Steps:**

1. Sign up at app.n8n.cloud
2. Import the workflow: top-right menu → Import from file → upload `PR-Reviewer.json`
3. Add the GitHub credential: Credentials → Add → HTTP Header Auth
   - Name: `Github PAT v2`
   - Header Name: `Authorization`
   - Header Value: `Bearer YOUR_GITHUB_PAT`
4. Add environment variables: Settings → Variables
   - `DISCORD_PR_WEBHOOK_URL` — your Discord webhook URL
   - `SLACK_PR_CHANNEL_ID` — your Slack channel ID
   - `TELEGRAM_BOT_TOKEN` — your Telegram bot token
   - `TELEGRAM_CHAT_ID` — your Telegram chat ID
5. Add Slack credential: Credentials → Add → Slack API, name it `Slack PR-Assistant`, paste Bot Token
6. Activate the workflow using the toggle in the top right
7. Your webhook URL: `https://your-instance.app.n8n.cloud/webhook/github-pr`
8. GITHUB_WEBHOOK_SECRET — your generated secret
9. NODE_FUNCTION_ALLOW_BUILTIN — set this to crypto

---

## Adding the GitHub Webhook (same for all methods)

Once your n8n instance is running and the workflow is active:

1. Go to your GitHub repository → Settings → Webhooks → Add webhook
2. Fill in:
   - Payload URL: `https://YOUR_N8N_URL/webhook/github-pr`
   - Content type: `application/json`
   - Secret: paste your GITHUB_WEBHOOK_SECRET here
   - Events: select "Let me select individual events" → check **Pull requests** only
   - Active: checked
3. Click Add webhook
4. GitHub sends a ping event. Check n8n Executions — it should appear within seconds

---

## Setting up credentials in n8n (self-hosted)

**GitHub PAT:**
- Credentials → New → HTTP Header Auth
- Name: `Github PAT v2` (must match exactly — the workflow references this name)
- Header Name: `Authorization`
- Header Value: `Bearer YOUR_GITHUB_PAT`

**Slack:**
- Credentials → New → Slack API
- Name: `Slack PR-Assistant` (must match exactly)
- Access Token: your Bot Token starting with `xoxb-`

**Discord and Telegram:** configured via environment variables only — no n8n credential needed

---

## Enabling platforms

Open the Edit Fields node in the workflow. Set the three boolean values:

| Field | Set to `true` to enable |
|---|---|
| `discord` | Send Discord embed on every PR event |
| `slack` | Send Slack Block Kit message on every PR event |
| `telegram` | Send Telegram Rich Message on every PR event |

You can enable any combination. All three simultaneously is supported. Setting all three to `false` means only the GitHub comment is posted — no external notifications.

---

## Verifying the setup

1. Activate the workflow (toggle top-right, green = active)
2. Open a pull request in your repository
3. In n8n → Executions — a new execution should appear within 3–5 seconds
4. Check the PR on GitHub — the bot comment should be posted
5. Check each enabled platform for the notification

If nothing appears in Executions, go to GitHub repo → Settings → Webhooks → click your webhook → Recent Deliveries. GitHub shows exactly what it sent and the HTTP response it got back. A non-200 response or timeout means n8n is not reachable at the URL you configured.

---

## Choosing the right method

| Situation | Method |
|---|---|
| Testing locally, laptop stays on | Method 1 — ngrok |
| Permanent URL, low cost, own a domain | Method 2 — Cloudflare Tunnel |
| Production, 24/7, full control, comfortable with Linux | Method 3 — VPS (Hetzner recommended) |
| Production, no infrastructure management, willing to pay | Method 4 — n8n Cloud |

---

## Common issues

**Webhook delivers but nothing appears in n8n Executions**
The workflow is not active. Toggle it active (green) — saved is not the same as active.

**Bot posts a new comment instead of patching the existing one**
The Get PR Comments node is not fetching all pages. Verify pagination is enabled on that node with the correct completion expression.

**Discord fires but Slack or Telegram does not**
Check the Switch node Output Mode — it must be set to All Matching Rules, not First Matching Rule. If it is on First Match, only the first enabled platform fires.

**Slack returns `channel_not_found`**
The bot is not a member of the channel. In Slack, go to the channel → click the channel name → Integrations → Add apps → find your bot and add it.

**Telegram returns 400 Bad Request**
The `sendRichMessage` endpoint requires Bot API 10.1 or newer. Verify your bot token is correct and your Telegram client is up to date.

**GitHub comment fails but Discord fires**
The GitHub PAT credential name does not match exactly. It must be `Github PAT v2`. Check Credentials and verify the name character-for-character.
