# Automatic beads synchronization implementation plan

Goal: run a local Community instance and mirror registered beads repositories automatically, with OpenProject done/reopen changes sent back through supported CLI commands.

Architecture: a separate Python standard-library service calls the beads CLI and OpenProject REST API. SQLite stores durable identities, checkpoints, pending operations, and conflicts. A systemd user service owns the single polling process. No Rails changes are needed for this first integration.

## Deliverable 1: local hosting

Use `customizations/local/compose.yaml` with immutable image digests, owner-private secrets, named database and asset volumes, loopback web binding, readiness checks, and automatic container restart. Verify HTTP health, authenticated admin access, and a dedicated service user's project-scoped permissions. Record source/runtime version equality.

## Deliverable 2: bridge contracts and tests

The separate bridge repository contains configuration and identity validation, CLI adapter, HTTP adapter, reconciliation engine and SQLite state, a command-line service runner, and behavioral tests. Add tests before implementing each behavior. Prove full-list retrieval, duplicate ID isolation, recovery of ambiguous creates, restart checkpoint recovery, no echo loops, source projection, PM done/reopen, conflict detection, stale API version rejection, wrong-project refusal, unsafe redirect refusal, and source identity mismatch.

Use explicit absolute repository paths and the installed CLI path. Check project UUIDs before every read. Never shell-interpolate task content. Do not put credentials in command arguments or Git. A process lock prevents a second writer. The initial synchronization never treats target status as a command without a prior confirmed checkpoint.

## Deliverable 3: real verification and automatic operation

Create synthetic verification tasks in isolated tracking data, map them to private test projects, and prove close/reopen in both directions. Verify ordinary nonclosed status changes and content updates. Simulate service restart and show stable mirror count and no unintended source state changes.

Create the private Tithe PM project and register its existing UUID. Import tasks, then verify an idle poll leaves source statuses unchanged. Enable the polling service and verify it remains active. Store an owner-readable validation summary outside Git with timestamps and synthetic test outcomes.

## Deliverable 4: operation and delivery

Document local URL and password retrieval, adding repositories, conflict inspection and resolution, service controls, credential rotation, backup and restoration, supported fields, and the residual race with independent CLI writers. Run behavior tests, live round trips, whitespace and secret checks. Commit and push only source/configuration templates and documentation; keep private runtime state local. No public deployment is included.
