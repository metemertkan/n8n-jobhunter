# n8n: LinkedIn job alerts -> JobHunter

Self-hosted n8n on this server. Once a day it reads the LinkedIn job-alert emails in Gmail and hands each one
to JobImporter, which imports the jobs into JobHunter. Then it moves the email to Gmail Trash.

```
Gmail (from:jobalerts-noreply@linkedin.com in:inbox)
  -> n8n "LinkedIn job alerts -> JobHunter" (daily 09:00 Europe/Amsterdam)
  -> POST http://job_importer:8765/linkedin-alerts   (private Docker network "jobalerts", bearer token)
       JobImporter: job IDs from the email (regex; Gemini only if no links are found)
       -> https://www.linkedin.com/jobs/view/<id>/ guest page -> NL-or-EU/US-remote + title filters
       -> dedup (all JobImporter regions + JobHunter) -> JobHunter create_job_manually_api
          (status NEW; exclude-list titles as SKIPPED with skip_reason)
  -> every job handled: email -> Gmail Trash (recoverable for 30 days)
     anything failed:   label "JobHunter/import-failed", email stays in the inbox and is retried next day
```

## What runs

| | |
|---|---|
| Container | `n8n` (image `n8nio/n8n:2.48.7`), compose project `n8n` |
| Data | Docker volume `n8n_data` (SQLite + settings). No other container or volume is used. |
| Ports | `127.0.0.1:5678` and `100.72.60.5:5678` (Tailscale) only, never 0.0.0.0 |
| Networks | `default` (internet/Gmail), `jobalerts` (to `job_importer:8765`), `jobhunter_net` (so `agentic_job_hunter` can call the webhook; no port published) |
| URLs | http://localhost:5678 (on the server) · http://100.72.60.5:5678 or http://server-x570-aorus-master.tail526ddc.ts.net:5678 (tailnet) |
| Secrets | `.env` (chmod 600): `N8N_ENCRYPTION_KEY`. The JobImporter token is stored encrypted as the n8n credential "JobImporter alerts token". |
| Hardening | public API off, env access from nodes blocked, Execute Command / SSH / Local File Trigger nodes removed, telemetry off, executions pruned after 14 days |

Workflow file: `workflows/linkedin-alerts-to-jobhunter.workflow.json`. It is imported but not published or active.

## Manual trigger from JobHunter ("Check LinkedIn alerts now", JobHunter PR #6)

- Webhook node "JobHunter button (webhook)": `POST http://n8n:5678/webhook/jobhunter-linkedin-alerts`, reachable
  only on the Docker network `jobhunter_net`. It needs the header `X-JobHunter-Token`, checked against the n8n
  Header Auth credential "JobHunter webhook token". It answers `{"status":"started"}` immediately and then runs the same path as the daily schedule.
  A wrong or missing token gets 403.
- JobHunter's `.env` holds `N8N_ALERTS_WEBHOOK_URL` and `N8N_ALERTS_WEBHOOK_TOKEN` (same secret; never in git).
- A trigger is a real run: it imports the jobs and trashes the processed alert emails.
- To rotate the token, put a new value in JobHunter's `.env` and in the n8n credential, then restart job_hunter.

## One-time setup (Mete)

1. **Create the n8n owner account.** Open http://100.72.60.5:5678 and create your owner account. Use a strong password, then turn on 2FA under Settings → Personal.
2. **Create a Google OAuth client.** Go to https://console.cloud.google.com.
   - Create or pick a project, then go to *APIs & Services → Library* and enable the **Gmail API**.
   - In *OAuth consent screen*, choose *External* and fill in the app name and your email. Under *Audience → Test users*, add your Gmail address. While the app is in *Testing*, Google expires the refresh token after 7 days. To avoid that, set it to *In production*; it is unverified, but that is fine for your own account.
   - In *Credentials → Create credentials → OAuth client ID*, choose *Web application*.
     - Authorized redirect URI: `http://localhost:5678/rest/oauth2-credential/callback`
     - Copy the Client ID and Client secret.
3. **Connect Gmail in n8n.** This step must use `localhost`, because Google allows plain http only for localhost. Either work on the server itself, or open a tunnel from your laptop with `ssh -L 5678:127.0.0.1:5678 server@100.72.60.5`. Then:
   - Open http://localhost:5678 and go to *Credentials → "Gmail (Mete)"*.
   - Paste the Client ID and Client secret, click **Sign in with Google**, allow access, and save.
4. **Publish the workflow.** Open *"LinkedIn job alerts -> JobHunter"*.
   - Optional test: click *Execute workflow* (manual trigger). With alerts in the inbox, this does a real import and trashes the emails.
   - Then **Publish** (activate) it. It runs daily at 09:00 Europe/Amsterdam.

## Operations

```
cd /home/server/Projects/n8n
docker compose ps                    # health
docker compose logs -f n8n
docker compose pull && docker compose up -d     # after changing the image tag
docker exec n8n n8n import:workflow --input=/workflows/linkedin-alerts-to-jobhunter.workflow.json   # re-import
```

- Failures carry the Gmail label `JobHunter/import-failed`. The run's details are under *Executions* in n8n, and JobImporter's report is in `/home/server/Projects/JobImporter/reports/alerts/`.
- Dry run without n8n: `docker exec job_importer /app/run.sh --country alerts --linkedin-alert-ids 4454269627 --dry-run`.
- Backups: the volume `n8n_data` and `.env`. Example: `docker run --rm -v n8n_data:/d -v $PWD:/b alpine tar czf /b/n8n_data.tgz -C /d .` (stop n8n first).
- The Tailscale port binding needs tailscaled up before Docker starts n8n. `restart: unless-stopped` retries it.

## Application tracking -> JobHunter (`workflows/application-tracking-to-jobhunter.workflow.json`)

Searches the inbox (`newer_than:3d` on scheduled/manual runs, application keywords, excluding LinkedIn job alerts, promotions, own mail
and already labelled mail), sends each mail to `http://job_importer:8765/track-application` and applies the
returned label (`JobHunter/tracked` or `JobHunter/needs-review`). Never trashes or archives mail.
Triggers: daily at 09:30 Amsterdam (3-day window, so a missed day is still covered; mail that was already
handled is skipped), manual, and
webhook `POST /webhook/jobhunter-application-tracking` (header `X-JobHunter-Token`, body
`{"dry_run": true, "newer_than": "30d"}`). Live status changes also need `[tracking] mode = "live"` in
JobImporter's config.toml.

Schedule changes imported with the CLI (`n8n import:workflow` + `n8n publish:workflow`) only take effect after
`docker compose restart n8n`; webhook changes apply without a restart.

## Weekly shortlist / Interview prep / Watchlist

| Workflow (id) | File | Schedule (Europe/Amsterdam) | Other triggers |
|---|---|---|---|
| Weekly shortlist -> email (`WeeklyShortlist01`) | `workflows/weekly-shortlist-email.workflow.json` | Sun 10:00, **disabled until Mete approves** | manual, webhook `jobhunter-weekly-shortlist` |
| Interview prep pack -> email (`InterviewPrep001`) | `workflows/interview-prep-pack-email.workflow.json` | daily 09:45 (after tracking 09:30), **disabled until approved** | manual, webhook `jobhunter-interview-prep` `{job_id?, force?}` |
| Target company watchlist -> JobHunter (`WatchlistToJH001`) | `workflows/target-company-watchlist.workflow.json` | Mon + Thu 08:30, published | manual, webhook `jobhunter-watchlist` `{dry_run?}` |

- JobImporter builds everything (see its README). n8n only calls it and sends the Gmail message.
- Email safety: "Recipient check (to = me)" reads `users/me/profile` and refuses any recipient other than the
  mailbox owner. Nothing is trashed or relabelled.
- Webhooks use the same `X-JobHunter-Token` header credential and are reachable only on the Docker network.
- The watchlist stays a dry run while JobImporter `countries/watchlist.toml` has `mode = "dry-run"`.
- To go live with the emails: in the n8n UI enable the schedule node and publish (or use `n8n publish:workflow --id=...`), then
  `docker compose restart n8n`.
