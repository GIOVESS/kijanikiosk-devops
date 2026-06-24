# KijaniKiosk Deployment Recovery Runbook

## When to use this runbook
A deployment has failed or the application is returning errors after a release.

## Step 1 — Confirm the failure scope
Check whether the issue affects all users or a subset.
`curl -o /dev/null -s -w "%{http_code}" http://localhost/api/health`
Expected: 200. Anything else: proceed.

## Step 2 — Identify the failing service
`systemctl status kk-api kk-payments kk-logs`
Note which service is not active. Check its journal:
`journalctl -u kk-api -n 50 --no-pager`

## Step 3 — Check disk and memory pressure
`df -h / && free -h`
If disk > 90%: `du -sh /opt/kijanikiosk/shared/logs/` — force logrotate if needed.
`sudo logrotate --force /etc/logrotate.d/kijanikiosk`

## Step 4 — Roll back the deployment
`git -C /opt/kijanikiosk/app log --oneline -5`
Identify the last stable commit. Redeploy that version and restart the service.

## Step 5 — Verify recovery
`curl -o /dev/null -s -w "%{http_code}" http://localhost/api/health`
`systemctl is-active kk-api kk-payments kk-logs`
All three must return `active`. Open a postmortem issue within 24 hours.
