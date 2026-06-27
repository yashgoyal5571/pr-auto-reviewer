# Deployment Guide — PR Auto-Reviewer

This workflow runs on **your own infrastructure**. Your GitHub PAT, Discord webhook URL, and all credentials stay in your environment. They are never shared with anyone.

---

## Prerequisites (all deployment methods)

Recommended n8n version: 1.0 or higher.
Before choosing a deployment method, you need three things ready:

**1. GitHub Personal Access Token**
- Go to GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
- Click "Generate new token (classic)"
- Set expiration to your preference (90 days minimum recommended)
- Select scope: `repo` (full)
- Copy the token — you will not see it again

**2. Discord Webhook URL**
- Open Discord → your server → a channel → Edit Channel → Integrations → Webhooks → New Webhook
- Copy the webhook URL

**3. n8n instance with a public HTTPS URL**
- Your n8n must be reachable from the internet so GitHub can send webhook payloads to it
- The method you choose below determines how you get this public URL

---

## Method 1 — Local machine + ngrok (testing and demos only)

**What it is:** n8n runs on your laptop. ngrok creates a public HTTPS tunnel to it. Suitable for personal testing or showing a demo. Not suitable for anything that needs to stay up when your machine is off.

**ngrok free tier facts (as of 2026):**
- 1 assigned static subdomain that persists across restarts (format: `yourname.ngrok-free.dev`)
- 1 simultaneous tunnel
- 1 GB bandwidth per month
- 2-hour session limit per tunnel start — you must restart ngrok after 2 hours
- No custom domain names on free tier

**Steps:**

1. Make sure n8n is running via Docker Desktop on port 5678
2. Sign up at ngrok.com and install the ngrok CLI
3. Authenticate: `ngrok config add-authtoken YOUR_TOKEN`
4. Start the tunnel: `ngrok http --domain=yourname.ngrok-free.dev 5678`
5. Your public webhook URL is: `https://yourname.ngrok-free.dev/webhook/github-pr`
6. Keep this terminal window open — closing it kills the tunnel

**Limitation:** The 2-hour session limit means you must restart ngrok every 2 hours. GitHub will retry failed webhook deliveries for up to 3 days, but if ngrok is down when a PR opens, the event is missed until GitHub retries.

---

## Method 2 — Cloudflare Tunnel (free, permanent URL, no session limits)

**What it is:** Cloudflare Tunnel creates a permanent outbound connection from your machine to Cloudflare's edge. No session timeouts, no bandwidth limits, free forever. Requires a domain name managed by Cloudflare DNS.

**Requirements:**
- A domain name (can be purchased for ~$10/year via Cloudflare Registrar or any registrar)
- Cloudflare account (free)
- `cloudflared` daemon installed on your machine

**Steps:**

1. Add your domain to Cloudflare and point its nameservers to Cloudflare
2. Install cloudflared: download from `developers.cloudflare.com/cloudflare-one/connections/connect-apps`
3. Authenticate: `cloudflared tunnel login`
4. Create a named tunnel: `cloudflared tunnel create pr-reviewer`
5. Configure the tunnel to route to localhost:5678 in the config file
6. Run as a background service: `cloudflared service install`
7. Your webhook URL is: `https://yourdomain.com/webhook/github-pr`

**Key difference from ngrok:** The tunnel runs as a system service and starts automatically on reboot. No session limits. Your URL never changes.

**Limitation:** Still requires your machine to be on and n8n to be running. If your laptop sleeps, the workflow stops processing until you wake it.

---

## Method 3 — Self-hosted n8n on a VPS (recommended for production)

**What it is:** n8n runs 24/7 on a cloud server you control. No dependency on your laptop. Permanent public URL with no tunneling needed.

**Recommended provider and cost:**
- Hetzner CX22: 2 vCPU, 4 GB RAM, 40 GB NVMe, 20 TB bandwidth — ~€4.35/month (~$4.70/month at current rates). More than enough for n8n.
- DigitalOcean Basic: $6/month, 1 vCPU, 1 GB RAM — sufficient but tighter on memory
- Vultr Cloud Compute: $6/month, 1 vCPU, 1 GB RAM

**What you install on the VPS:**
- Docker + Docker Compose (to run n8n)
- Nginx (reverse proxy for HTTPS)
- Certbot (free SSL via Let's Encrypt)

**Steps:**

1. Create a VPS at your chosen provider. Select Ubuntu 24.04 LTS.
2. SSH into the server: `ssh root@YOUR_SERVER_IP`
3. Install Docker:
   ```bash
   curl -fsSL https://get.docker.com | sh
   ```
4. Create a `docker-compose.yml`:
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
         - DISCORD_PR_WEBHOOK_URL=your_discord_webhook_url_here
       volumes:
         - n8n_data:/home/node/.n8n
   volumes:
     n8n_data:
   ```
5. Start n8n: `docker compose up -d`
6. Install Nginx and Certbot, configure a reverse proxy to port 5678, and issue a free SSL certificate
7. Point your domain's A record to the VPS IP
8. Your webhook URL is: `https://yourdomain.com/webhook/github-pr`

**Resources needed for Nginx + Certbot setup:** Search "n8n self-hosted nginx ssl tutorial" — the n8n docs cover this exactly.

---

## Method 4 — n8n Cloud (no infrastructure to manage)

**What it is:** n8n hosts and manages everything. You log in, import the workflow, add credentials, and activate. No servers, no Docker, no SSL.

**Current pricing (verified June 2026):**
- Starter: €24/month (~$26) — 2,500 executions/month
- Pro: €60/month (~$65) — 10,000 executions/month
- No permanent free tier. 14-day free trial available.

**Execution math for this workflow:**
- This workflow is event-driven (webhook-triggered). It only executes when a PR is opened or updated.
- 1 PR open or push = 1 execution
- On a repo with 100 PRs/month, that is 100 executions
- The Starter plan at 2,500 executions/month is more than sufficient for any small-to-medium repo

**Steps:**

1. Sign up at app.n8n.cloud
2. Import the workflow: open n8n → top-right menu → Import from file → upload `PR-Reviewer.json`
3. Add the GitHub PAT credential: Settings → Credentials → Add → Header Auth
   - Name: `Github PAT v2`
   - Name field: `Authorization`
   - Value field: `token YOUR_GITHUB_PAT`
4. Add the Discord webhook URL: Settings → Variables → Add variable
   - Key: `DISCORD_PR_WEBHOOK_URL`
   - Value: your Discord webhook URL
5. Activate the workflow using the toggle in the top-right
6. Your webhook URL is: `https://your-instance.app.n8n.cloud/webhook/github-pr`

---

## Adding the GitHub Webhook (same for all methods)

Once you have a running n8n instance with a public URL:

1. Go to your GitHub repository
2. Settings → Webhooks → Add webhook
3. Fill in:
   - Payload URL: `https://YOUR_N8N_URL/webhook/github-pr`
   - Content type: `application/json`
   - Secret: leave empty
   - Which events: select "Let me select individual events" → check **Pull requests** only
   - Active: checked
4. Click Add webhook
5. GitHub will send a ping event. In n8n, go to Executions — you should see it arrive

---

## Setting Up Credentials in n8n (self-hosted)

**GitHub PAT:**
1. n8n left sidebar → Credentials → Add credential → Header Auth
2. Name: `Github PAT v2` (must match exactly)
3. Name field: `Authorization`
4. Value field: `token YOUR_GITHUB_PAT`
5. Save

**Discord Webhook URL:**
- On self-hosted Docker: add `DISCORD_PR_WEBHOOK_URL=your_url` to the environment section in `docker-compose.yml` and restart with `docker compose up -d`
- On n8n Cloud: Settings → Variables → add key `DISCORD_PR_WEBHOOK_URL` with the URL as value

---

## Verifying the Setup

1. Activate the workflow in n8n (toggle top-right, must be green)
2. Open a pull request in your repository (or push a new commit to an existing open PR)
3. In n8n, go to Executions — you should see a new execution appear within seconds
4. Check the PR on GitHub — the bot should have posted a comment
5. Check your Discord channel — the embed should appear

If nothing happens, go to your GitHub repo → Settings → Webhooks → click your webhook → Recent Deliveries. GitHub shows you exactly what it sent and whether it got a 200 response back.

---

## Choosing the Right Method

| Situation | Recommended method |
|---|---|
| Testing or personal use, laptop stays on | Method 1 (ngrok) or Method 2 (Cloudflare Tunnel) |
| Always-on, own a domain, want full control | Method 2 (Cloudflare Tunnel) or Method 3 (VPS) |
| Production, 24/7, no infrastructure maintenance | Method 3 (VPS) or Method 4 (n8n Cloud) |
| Non-technical team, willing to pay | Method 4 (n8n Cloud) |
| Budget-conscious, comfortable with Linux | Method 3 (Hetzner VPS ~$5/month) |

---

## Common Issues

**Webhook delivers but nothing appears in n8n executions**
- Check the workflow is activated (not just saved)
- Confirm the webhook path in n8n matches `/webhook/github-pr` exactly

**Bot posts a new comment instead of updating the existing one**
- The "Get PR Comments" node pagination may not be fetching all comments
- Verify pagination is enabled on that node

**Discord embed fires but GitHub comment fails**
- Check the GitHub PAT credential name matches `Github PAT v2` exactly
- Verify the PAT has `repo` scope and has not expired

**n8n shows execution but GitHub shows no comment**
- Open the execution in n8n and check the "Post Comment" or "Patch Comment" node output for error details
