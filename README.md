# Human-in-the-Loop Browser Workspace

`humaninloopbw` is a practical remote-browser assistant stack for running a **persistent, human-controlled Chromium session** on your own VM while allowing an assistant such as OpenClaw, Playwright, or another local automation layer to attach to that same browser through Chrome DevTools Protocol/CDP.

The purpose is not to build a spam bot or an anti-detection system. The purpose is to create a workflow where the human remains in control, the browser session stays logged in, and the assistant helps with navigation, drafting, review prep, page opening, and other approved actions.

In plain English: this project gives you a browser running on your VM that you can see through noVNC, while your local assistant can also connect to that exact same browser session. You log in manually, the browser profile persists, and the assistant works inside that already-authenticated browser only when you approve the flow.

---

## Project Philosophy

This stack is built around a simple rule:

> The assistant can prepare and navigate, but the human stays in the loop for sensitive or final actions.

Good use cases include:

- Opening pages for human review.
- Drafting posts, captions, comments, or messages.
- Preparing browser tabs for the user.
- Summarizing visible page content.
- Letting a human manually log in once, then preserving the profile.
- Pausing before final actions.
- Pairing OpenClaw with a visible browser so the user can intervene at any moment.
- Using Telegram, curl, or another control layer to say “continue”, “stop”, “approve”, or “deny”.

Bad use cases that this project should not be used for:

- Spam.
- Mass posting.
- Credential theft.
- CAPTCHA bypass.
- Anti-detection bypass.
- Hidden account farming.
- Rapid liking/following/commenting.
- Scraping private user data.
- Evading platform checkpoints or enforcement systems.

This repo should remain a **human-supervised assistant workspace**, not a stealth automation kit.

---

## High-Level Architecture

The stack runs a full desktop browser inside a lightweight Linux container:

```text
[ You on phone/laptop browser ]
            |
            | noVNC over HTTP/WebSocket
            v
[ noVNC + websockify ]
            |
            v
[ x11vnc ] ---> [ Xvfb virtual display :1 ] ---> [ Fluxbox window manager ]
                                                   |
                                                   v
                                      [ Chromium headed browser ]
                                                   |
                                                   | CDP on 127.0.0.1:9222
                                                   v
                                      [ OpenClaw / Playwright / Agent ]
```

The key design choice is that there is only one browser profile and one visible browser session.

That means:

- You can open the browser through noVNC.
- You can log in manually.
- Chromium stores the session in `./chrome-profile`.
- OpenClaw or Playwright connects to the same Chromium process through CDP.
- The assistant sees the same pages you see.
- You can take over manually at any time.

---

## Components

### 1. Xvfb

`Xvfb` creates a fake Linux display inside the container. Chromium needs a display because this project runs Chromium in visible/headed mode, not pure headless mode.

Default display:

```text
:1
```

### 2. Fluxbox

`Fluxbox` is a tiny window manager. It gives Chromium a normal desktop environment without installing a heavy full desktop.

### 3. Chromium

Chromium is launched with:

- a persistent profile directory
- a CDP port
- a visible window size
- normal browser storage

Important idea:

```text
Persistent profile = fewer repeated logins
```

The browser profile is stored outside the container in:

```text
./chrome-profile
```

Inside the container it is mounted as:

```text
/data/chrome-profile
```

### 4. x11vnc

`x11vnc` exports the Xvfb display as a VNC session.

### 5. noVNC + websockify

`noVNC` lets you control the VNC desktop from a normal web browser.

Default noVNC port:

```text
6080
```

You open it from your local browser like:

```text
http://YOUR_VM_IP:6080/vnc.html
```

### 6. CDP / Chrome DevTools Protocol

CDP lets OpenClaw, Playwright, or another browser-control tool attach to the already-running Chromium session.

Default CDP endpoint:

```text
http://127.0.0.1:9222
```

Important security rule:

```text
CDP must stay bound to 127.0.0.1 unless you absolutely know what you are doing.
```

The compose file intentionally maps it like this:

```yaml
- "127.0.0.1:9222:9222"
```

That means it is reachable from the VM itself, but not publicly exposed to the internet.

### 7. Approval API

The approval API is the control layer that lets a human approve, deny, reset, or check status.

Default API port:

```text
8787
```

This is where the human-in-the-loop behavior lives.

---

## Current Repository Files

Expected structure:

```text
humaninloopbw/
├── README.md
├── .gitignore
├── .env.example
├── docker-compose.yml
├── Dockerfile
├── supervisord.conf
├── start-browser.sh
└── app/
    ├── requirements.txt
    └── server.py
```

At minimum, the project needs:

```text
.env
Dockerfile
docker-compose.yml
supervisord.conf
start-browser.sh
app/requirements.txt
app/server.py
```

`.env` should not be committed.

Use `.env.example` as the safe public template.

---

## Environment Variables

Create your real `.env` from the example:

```bash
cp .env.example .env
```

Then edit it:

```env
VNC_PASSWORD=change_this_password_now
APPROVAL_TOKEN=change_this_approval_token_now
CHROME_PROFILE_DIR=/data/chrome-profile
DISPLAY=:1
CDP_URL=http://127.0.0.1:9222
```

### VNC_PASSWORD

Password used when opening the noVNC browser desktop.

Use a strong password.

Bad:

```text
123456
password
admin
```

Better:

```text
long-random-password-here
```

### APPROVAL_TOKEN

Secret token required for approval API commands.

Every approval/deny/reset/open request should include:

```text
x-approval-token: YOUR_TOKEN
```

### CHROME_PROFILE_DIR

Where Chromium stores profile data inside the container.

Default:

```text
/data/chrome-profile
```

This is mapped to:

```text
./chrome-profile
```

on the VM.

### DISPLAY

Linux display used by Xvfb and Chromium.

Default:

```text
:1
```

### CDP_URL

Where the API connects to Chromium.

Default:

```text
http://127.0.0.1:9222
```

---

## Security Model

This project exposes three important surfaces:

```text
6080 = noVNC browser UI
8787 = approval API
9222 = Chromium CDP
```

### Port 9222: CDP

CDP is powerful. Anyone who can access it may control the browser.

So this repo binds it to localhost only:

```yaml
- "127.0.0.1:9222:9222"
```

Do not expose `9222` publicly.

### Port 6080: noVNC

noVNC gives visual control of the browser. Protect it.

Recommended options:

- Use a firewall.
- Use Cloudflare Tunnel.
- Use Cloudflare Access.
- Use Tailscale or a private VPN.
- Use a strong VNC password.

### Port 8787: Approval API

The approval API can instruct the browser to open URLs and approve flows.

Protect it too.

At minimum:

- Use a strong `APPROVAL_TOKEN`.
- Avoid exposing it publicly without a tunnel/access layer.
- Prefer localhost-only when integrating with Telegram or another local bot.

---

## Recommended VM

Minimum practical VM:

```text
2 vCPU
4 GB RAM
20–40 GB SSD
Ubuntu or Debian host
Docker + Docker Compose
```

1 GB RAM can technically run tiny containers, but Chromium plus noVNC plus an assistant can feel terrible. 4 GB RAM is the realistic starting point.

AMD vs Intel does not matter much here. RAM matters more.

---

## Install Docker on Ubuntu/Debian

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Check:

```bash
docker --version
docker compose version
```

Optional, allow current user to run Docker:

```bash
sudo usermod -aG docker $USER
```

Then log out and back in.

---

## Clone and Run

```bash
git clone https://github.com/wealthykenny/humaninloopbw.git
cd humaninloopbw
cp .env.example .env
nano .env
```

Start:

```bash
docker compose up -d --build
```

View logs:

```bash
docker logs -f openclaw-human-browser
```

Stop:

```bash
docker compose down
```

Restart:

```bash
docker compose restart
```

Rebuild after code changes:

```bash
docker compose up -d --build
```

---

## Open noVNC

In your browser:

```text
http://YOUR_VM_IP:6080/vnc.html
```

Use your `VNC_PASSWORD` from `.env`.

Inside noVNC, you should see the Chromium browser running in the virtual desktop.

Login manually to the services you personally use.

Because the profile is mounted to `./chrome-profile`, the login session should survive container restarts as long as:

- you do not delete `./chrome-profile`
- the service does not revoke the session
- the site does not require fresh verification
- the VM IP/session remains acceptable to the site

---

## Approval API Usage

Health check:

```bash
curl http://127.0.0.1:8787/health
```

Status:

```bash
curl http://127.0.0.1:8787/status
```

Open a page:

```bash
curl -X POST http://127.0.0.1:8787/open \
  -H "Content-Type: application/json" \
  -H "x-approval-token: YOUR_APPROVAL_TOKEN" \
  -d '{"url":"https://example.com"}'
```

Approve:

```bash
curl -X POST http://127.0.0.1:8787/approve \
  -H "x-approval-token: YOUR_APPROVAL_TOKEN"
```

Deny:

```bash
curl -X POST http://127.0.0.1:8787/deny \
  -H "x-approval-token: YOUR_APPROVAL_TOKEN"
```

Reset:

```bash
curl -X POST http://127.0.0.1:8787/reset \
  -H "x-approval-token: YOUR_APPROVAL_TOKEN"
```

Draft text into a page:

```bash
curl -X POST http://127.0.0.1:8787/draft-post \
  -H "Content-Type: application/json" \
  -H "x-approval-token: YOUR_APPROVAL_TOKEN" \
  -d '{
    "url":"https://example.com",
    "text":"This is a human-reviewed draft prepared by my assistant.",
    "require_approval":true
  }'
```

Important: the draft endpoint should prepare content, not blindly publish. Final publishing should remain human-approved.

---

## OpenClaw / Playwright Connection

OpenClaw should connect to the already-running Chromium browser through CDP:

```text
http://127.0.0.1:9222
```

Example Playwright connection:

```python
from playwright.async_api import async_playwright

async with async_playwright() as p:
    browser = await p.chromium.connect_over_cdp("http://127.0.0.1:9222")
    context = browser.contexts[0]
    page = context.pages[0]
    await page.goto("https://example.com")
```

This is the key difference from launching a fresh browser:

```text
Fresh browser = no login session
CDP attach = same visible browser session
```

---

## Human-in-the-Loop Workflow

Recommended workflow:

```text
1. Start the container.
2. Open noVNC.
3. Log in manually.
4. Leave Chromium running.
5. OpenClaw connects through CDP.
6. OpenClaw navigates or prepares a draft.
7. Approval API marks the state as waiting.
8. Human reviews through noVNC.
9. Human sends approve or deny.
10. Assistant continues only if approved.
```

This gives you the practical “assistant with a visible handbrake” system.

---

## Telegram Integration Idea

A Telegram bot can sit in front of the approval API.

Example commands:

```text
/status
/approve
/deny
/reset
/open https://example.com
```

The Telegram bot should call the local API:

```text
http://127.0.0.1:8787
```

Recommended security rule:

Only allow your Telegram user ID to trigger commands.

Pseudo-flow:

```text
OpenClaw: Draft prepared. Waiting for approval.
Telegram bot: Sends message to Sir Trey.
Sir Trey: /approve
Telegram bot: POST /approve
OpenClaw: Continues.
```

---

## Cloudflare Tunnel Deployment Pattern

For remote access, prefer Cloudflare Tunnel instead of exposing ports directly.

Recommended public routes:

```text
browser.yourdomain.com -> http://localhost:6080
control.yourdomain.com -> http://localhost:8787
```

Then protect both with Cloudflare Access.

Do not expose:

```text
9222
```

CDP should stay local to the VM.

---

## Files That Should Never Be Committed

Do not commit:

```text
.env
chrome-profile/
logs/
secrets/
*.sqlite with sensitive sessions
```

`chrome-profile/` can contain browser cookies, local storage, saved sessions, and other sensitive data.

That is why `.gitignore` should include:

```gitignore
.env
chrome-profile/
*.log
__pycache__/
.pytest_cache/
.DS_Store
```

---

## Operational Notes

### First Run

The first build may take a while because Debian packages and Python packages are installed.

After that, rebuilds should be faster.

### Browser Seems Blank

Check logs:

```bash
docker logs -f openclaw-human-browser
```

Restart:

```bash
docker compose restart
```

### noVNC Connects But Browser Is Missing

Possible causes:

- Chromium failed to start.
- Xvfb failed.
- Fluxbox failed.
- Display variable mismatch.

Check:

```bash
docker exec -it openclaw-human-browser ps aux
```

You should see processes for:

```text
Xvfb
fluxbox
chromium
x11vnc
websockify
uvicorn
```

### CDP Not Connecting

From the VM:

```bash
curl http://127.0.0.1:9222/json/version
```

If working, it should return browser metadata.

### Approval API Not Working

Check health:

```bash
curl http://127.0.0.1:8787/health
```

Check logs:

```bash
docker logs -f openclaw-human-browser
```

### Login Does Not Persist

Check that the volume exists:

```bash
ls -la chrome-profile
```

Do not run:

```bash
rm -rf chrome-profile
```

unless you intentionally want to wipe the browser profile.

---

## Suggested Next Features

Good next features for this repo:

### 1. Telegram Approval Bot

Add a bot that maps:

```text
/approve -> POST /approve
/deny -> POST /deny
/status -> GET /status
/open URL -> POST /open
```

### 2. Action Queue

Instead of one global state, store pending actions in a queue:

```text
pending
approved
rejected
completed
failed
```

### 3. Screenshots on Demand

Add an endpoint:

```text
POST /screenshot
```

The assistant can send the current browser screenshot to Telegram for quick review.

### 4. Per-Action Approval

Instead of one global approve/deny, create action IDs:

```text
approve action_123
deny action_123
```

### 5. Local-Only Control Mode

Bind approval API to localhost only and control it through Telegram bot running on the VM.

### 6. Safer Draft System

Separate endpoints:

```text
/open
/fill-draft
/request-approval
/manual-confirmation-required
```

Avoid any endpoint named `/post-now` unless it still requires explicit human approval.

---

## Important Design Boundaries

This project is allowed to make browser work easier for the VM owner.


- 

The correct identity of this project is:

```text
Remote browser workspace + human approval gate + assistant control bridge
```

That is powerful enough.

---

## VM-Boss Notes

For Sir / VM-Boss workflow:

Use this repo as the base layer.

Then add OpenClaw on the same VM and point it to:

```text
http://127.0.0.1:9222
```

The browser is the shared working area.

The human is the final decision maker.

OpenClaw is the assistant.

noVNC is the remote hand.

The approval API is the brake pedal.

That is the architecture.
