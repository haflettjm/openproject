# Local OpenProject and beads synchronization

The owner requested a running local OpenProject Community instance with automatic task imports and done/reopen synchronization back to beads. The fork starts at OpenProject v17.9.4. The separate bridge targets the installed snjax/beads v0.62.1 CLI. This document specifies the first working integration.

## Hosting

Use Docker Compose with the pinned OpenProject slim release, PostgreSQL 17, and memcached. Web, worker, cron, and initial seeder share application settings and persistent assets. A systemd user proxy binds loopback at localhost:8088 and connects to a Puma Unix socket in an owner-private host directory. This avoids host-to-Docker bridge connectivity problems while preserving VPN/firewall settings. Generate unique local admin, database, and Rails secrets in an ignored file with mode 0600. No public exposure, enterprise feature changes, or modifications to Tithe source code are required.

Runtime starts from the official image matching the fork baseline. Application changes to the fork will need a custom image build before they affect the running instance. The bridge uses the API and can develop independently of Rails and frontend dependencies.

## Ownership and identity

Register each absolute repository path, its beads project UUID, and the target OpenProject numeric project ID. Check the UUID in the repository metadata on every read. The source identity is the UUID plus the complete bead ID, never an issue prefix or title. Store a unique mapping and last confirmed source/target snapshots in a local SQLite ledger. A second independent repository can use the same bead IDs without collisions.

Beads owns titles, descriptions, priorities, types, parentage, and ordinary status. Project-manager edits to those fields do not write back. Completion is the explicit exception: a transition from a nonclosed OpenProject status to a closed one requests `bd close`; a closed-to-open transition requests `bd reopen`. Reopening restores open, not the previous in-progress state. Closing an existing decision task through the owner-operated PM is an explicit user action, not an automated design decision. No status change asserts separate verification or owner acceptance.

## Reconciliation

A single host service polls every 15 seconds. Polling is the reliable event catch-up mechanism; this version does not require a public webhook listener or modify existing repository hooks. Use supported CLI JSON queries including closed records and no default 50-record limit. Project content and per-task status are mirrored; aggregate project readiness is not inferred from task completion alone.

Fetch OpenProject work packages through paginated project-scoped API requests. Each mirrored description contains a durable source marker so an ambiguous create can be recovered without creating another copy. Duplicate markers, missing mapped objects, repository identity mismatch, unmapped statuses, and simultaneous divergent completion edits create visible conflicts and stop writes for the affected task.

For a mapped task compare both current states with the last confirmed pair. Source-only changes project outward. A target-only change in closedness requests a beads close/reopen. Divergent changes affecting completion are quarantined, while identical completion is converged. Keep a durable pending operation before any write. After ambiguous responses, reread and verify before advancing the ledger. OpenProject updates include lockVersion, and rejected stale updates are retried only after fresh reconciliation. Never automatically delete tasks in either system.

The CLI provides no general expected-version update. Immediately reread source before a completion write and verify after it; conflicts detected during this window remain visible. This reduces but cannot eliminate races with an independent CLI writer. No exactly-once or linearizable cross-system guarantee is made. Do not use forced closure to bypass beads dependency checks.

## Security and operation

Use a dedicated non-admin OpenProject service user scoped to registered projects. Keep its API token in an owner-readable local credential file, never Git or process arguments. Allow API credentials only to the configured origin; refuse redirects and unsafe pagination links. Build CLI argument arrays without a shell and use full bead IDs. Log identities and outcomes, not task descriptions or credentials. Acquire a process lock so two bridge instances cannot write the same ledger concurrently.

Retry transient network/server errors on later poll cycles with backoff. Persist conflicts and expose status through a CLI report. Restart automatically after failures and host login/reboot where user services are enabled. Preserve the ledger, application configuration, database, assets, and beads backups together. Missing credentials or invalid configuration must stop writes visibly.

## Validation

First test import, repeated polling without duplicates, same bead ID in two projects, source status changes, PM close/reopen, competing edits, lost create responses, stale lockVersion, malformed mappings, and process restart. Use temporary synthetic beads tasks and projects for destructive/state-changing tests. Then import Tithe and prove an idle poll does not mutate its task states.

Test health and authenticated project access in the actual local instance, run real synthetic done/reopen round trips through both APIs, and check the installed service. Scale performance and sophisticated portfolio reporting remain separate validation work; a successful small import does not establish 10k-task capacity.
