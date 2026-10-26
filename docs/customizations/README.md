# Our OpenProject fork

This fork is based on upstream v17.9.4. Keep `origin` pointed at `haflettjm/openproject` and `upstream` at `opf/openproject`. Put our changes on `custom/beads-foundation`; keep upstream release history intact.

## Local instance

The Compose definition is `customizations/local/compose.yaml`. It runs OpenProject Community at http://localhost:8088 with PostgreSQL and separate web, background worker, and cron services. A systemd user service forwards loopback TCP to a private Puma Unix socket because VPN firewall rules block host access to Docker bridge ports on this machine. VPN and firewall settings remain unchanged. Its runtime image matches the fork baseline; changing Rails or frontend code requires building a new image before those changes appear.

The file references the owner-private configuration at `~/.config/openproject/local.env`. It contains generated admin, database, and Rails secrets. Keep it outside Git and preserve it across restarts. The admin login is `admin`; read `OPENPROJECT_SEED_ADMIN_USER_PASSWORD` from that private file to retrieve the password. This instance is bound to loopback and is intended for this machine. LAN or internet hosting needs a separate hostname and TLS configuration.

Docker named volumes `openproject-local_database` and `openproject-local_assets` preserve data across ordinary container restarts. Do not remove these volumes to upgrade or troubleshoot.

## Beads integration

The bridge is maintained separately from Rails application changes. See [the sync design](beads-sync-design.md) for ownership, project mapping, completion rules, and concurrency limits. Beads remains authoritative for task content; OpenProject can send explicit done/reopen changes back.

Current status: local hosting is implemented and verified. The bridge described in the design and [implementation plan](beads-sync-plan.md) is pending design review and implementation. No automatic synchronization is enabled yet.

## Service controls

Start or update the containers:

```sh
docker compose -f /home/fivek777/git/personal/openproject/customizations/local/compose.yaml up -d
```

Install or update the local proxy service:

```sh
mkdir -p /home/fivek777/.config/systemd/user
cp /home/fivek777/git/personal/openproject/customizations/local/openproject-local-proxy.service /home/fivek777/.config/systemd/user/openproject-local-proxy.service
systemctl --user daemon-reload
systemctl --user enable --now openproject-local-proxy.service
```

Check health and service state:

```sh
curl --noproxy '*' --fail --max-time 10 http://localhost:8088/health_checks/default
systemctl --user status openproject-local-proxy.service
docker compose -f /home/fivek777/git/personal/openproject/customizations/local/compose.yaml ps
```

The proxy can start before the application finishes booting. Wait for the health check before using the browser. Docker restart policies and the enabled user service restore ordinary operation after reboot; user lingering is enabled on the current machine. The absolute host paths in this local configuration must be adapted on another machine.

Task completion and owner acceptance remain different concepts. A research task can be done while the design decision it informs remains open. Art import checks do not establish visual approval. Store evidence in the existing wiki or asset systems and link it from the task.

Community basic boards do not update task status when a card changes columns. Use the task's actual Status field for done/reopen synchronization. Enterprise feature gates remain unchanged.

## Upstream maintenance

Before upgrading, back up the application database, attachments, private configuration, bridge ledger and configuration, and the beads databases. Test restoration into separate targets. Fetch the next supported upstream release, merge it into our customization branch, and review the diff. Run affected tests and the synthetic sync round trip before updating the running image. Keep image digests and application baseline together in the change.

The public fork must not contain task dumps, database files, private project mappings, credentials, or backup archives. Local tracking files are excluded through Git's local exclude file. A source checkout alone is not a backup of the running application.
