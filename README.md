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
| URLs | http://localhost:5678 (on the server) · http://100.72.60.5:5678 or http://server-x570-aorus-master.tail526ddc.ts.net:5678 (tailnet) |
| Secrets | `.env` (chmod 600): `N8N_ENCRYPTION_KEY`. The JobImporter token is stored encrypted as the n8n credential "JobImporter alerts token". |
| Hardening | public API off, env access from nodes blocked, Execute Command / SSH / Local File Trigger nodes removed, telemetry off, executions pruned after 14 days |

Workflow file: `workflows/linkedin-alerts-to-jobhunter.workflow.json`. It is imported but not published or active.

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
