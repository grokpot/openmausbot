# OpenMausBot on Railway

OpenMausBot — an open-source AI bot team (Claude Code, Codex and API-key engines) where every bot gets its own browser, files and tools. Deploy on Railway with one click.

[![Deploy on Railway](https://railway.com/button.svg)](https://railway.com/deploy/LxzuVp)

This repo packages the official [milind-soni/OpenMausBot](https://github.com/milind-soni/OpenMausBot) server image (`ghcr.io/milind-soni/openmausbot:v0.1.165`, Apache-2.0) for Railway as `ghcr.io/onepush-dev/openmausbot:0.1.165`.

## UI patches

This image rebuilds OpenMausBot's web UI from the pinned upstream tag with:

- `patches/sidebar-attention-sort.patch` — Grok-style sidebar: bots waiting on you or with unread replies float to the top (newest first) with a row marker; the Inbox discovery row is hidden while that sort is on ([milind-soni/OpenMausBot#5627](https://github.com/milind-soni/OpenMausBot/issues/5627)). Opt out in **Settings → Appearance → Bring bots that need you to the top**.

## What's inside

Upstream's `compose.yaml` runs two containers in one network namespace: the `omb` server and a Caddy edge. Railway services can't share a network namespace, so this image runs both in one service:

| Process | Listens on | Role |
|---|---|---|
| OpenMausBot server | `127.0.0.1:8799` (app), `127.0.0.1:8800` (webhooks) | loopback only, never exposed |
| Caddy | Railway `PORT` (8080) | the only public listener; forwards the real host and `X-Forwarded-*`, so every request needs a **paired session** |
| Pairing gate | `127.0.0.1:8801`, routed at `/_railway/pair` | password-protected one-time pairing link, so no terminal is needed |

Also baked in: the Claude Code and Codex CLIs (upstream's `ENGINES` build arg), Chrome for Testing for the bots' browsers (from the upstream image), and `tini` as PID 1 (compose `init: true`).

- Volume: `/data` (HOME: chats, routines, bot workspaces, browser profiles, engine logins, paired sessions)
- Healthcheck: `/api/health`
- Restart policy: always

## First sign-in

1. Open `https://<your-domain>/_railway/pair`.
2. Sign in with user `admin` and the `OMB_ADMIN_PASSWORD` value from the service's **Variables** tab.
3. You land on `/pair` with a one-time code filled in: click **Connect**. The browser is now paired (admin) for 30 days, renewed on use.

Then connect an AI engine in **Settings → Engines** (Claude: "Sign in to Claude"; Codex: "Connect ChatGPT" with a device code) or add a provider key in **Settings → Connections → Model providers** (Anthropic, OpenAI-compatible/OpenRouter, xAI). API-key engines currently support chat only; agent tools and computer use need a signed-in engine CLI.

More devices: **Settings → Remote access** (pairing codes) or visit `/_railway/pair` again.

## Variables

| Variable | Default | Notes |
|---|---|---|
| `OMB_ADMIN_PASSWORD` | generated per deploy (`${{secret(32)}}`) | protects `/_railway/pair`; not passed to the server or bots |
| `OMB_ADMIN_USER` | `admin` (image) | user for `/_railway/pair` |
| `OMB_PUBLIC_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | used in pairing links |
| `OMB_WEBHOOK_PUBLIC_URL` | `https://${{RAILWAY_PUBLIC_DOMAIN}}` | base printed for `/hooks/...` URLs |
| `OMB_ANTHROPIC_API_KEY` | unset (optional) | workspace Anthropic key, instead of a Claude login |
| `OMB_SIGNIN_EMAILS` | unset (optional) | allow-list for "Sign in with your email" (full access) |
| `OMB_SIGNIN_MEMBER_EMAILS` | unset (optional) | allow-list for chat-only email sign-in |
| `PORT`, `OMB_PORT`, `OMB_WEBHOOK_PORT`, `OMB_PAIR_GATE_PORT`, `OMB_DATA_DIR` | 8080, 8799, 8800, 8801, `/data/.openmausbot` (image) | internal ports and data dir |

## Automated Updates

This repo uses Dependabot to automatically track new releases of the upstream `ghcr.io/milind-soni/openmausbot` image. When a new version is released:

1. **Dependabot** opens a PR daily (checks at 09:00 UTC) with the version bump
2. **Build CI** (`build-pr.yml`) builds the Docker image and runs smoke tests
3. **Auto-merge** (`auto-merge-dependabot.yml`) automatically merges patch/minor updates once the build passes
4. **Publish** (`publish-ghcr.yml`) builds and publishes the new image to `ghcr.io/onepush-dev/openmausbot` on merge to `main`
5. **Railway** rebuilds and redeploys your service automatically on the next push to `main`

Major version updates require manual review and are not auto-merged.

### Rolling Back

If a version causes issues, revert to a previous version:

1. Identify the commit that bumped the version (e.g., `git log --oneline Dockerfile`)
2. Revert that commit: `git revert <commit-sha>`
3. Push to main: `git push origin main`
4. Railway will automatically redeploy with the previous version

Or manually edit the `Dockerfile` to pin a specific version and push.

## Notes

- Security: the server trusts session-less loopback callers as the owner, so ports 8799/8800 are never published. Only Caddy is public, and through it every request needs a paired session (webhooks under `/hooks/*` use their own per-hook secrets).
- Railway has no `--shm-size`; Chrome falls back to `/tmp` for shared memory. Give the service at least 2 GB RAM if several bots browse at once.
- Desktop-only features (dictation, controlling a host desktop, Local VMs that need Docker/KVM) don't apply on a server.

## Manual Updates

To manually update the base image version, edit the `FROM` line in `Dockerfile` with the new version tag and its multi-arch digest (get it from GHCR), then push to main.

License: this packaging is MIT; OpenMausBot is Apache-2.0 (its `enterprise/` folder has its own license).
