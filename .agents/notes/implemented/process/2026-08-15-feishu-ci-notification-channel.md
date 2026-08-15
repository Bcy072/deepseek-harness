# Agent Note: Feishu CI notification channel

Status: implemented

## Problem

CI failures on `deepseek-ai/deepseek-harness` across many workflows (CI, E2E, release, landlock, docs, issue lifecycle) were only visible through the GitHub Actions web UI or email. Developers who primarily work in Feishu had no in-band notification when a workflow failed or when a master push succeeded, delaying response to broken builds.

## Decision

A new `workflow_run`-triggered workflow (`.github/workflows/feishu-notify.yml`) sends an interactive card message to a configured Feishu user via the Feishu Open Platform bot API. The workflow:

- Triggers after any listed workflow completes (`workflow_run` event, `completed` type), avoiding modification of existing workflows.
- Notifies on every failure and cancellation; for successes, only notifies on master-push runs (not PR runs), reducing noise.
- Supports `workflow_dispatch` for manual testing with simulated conclusion, name, and URL inputs.
- Obtains a `tenant_access_token` using app credentials (`app_id` + `app_secret`), then sends an interactive card to the target user's `open_id` with repo, branch, event, commit, and conclusion metadata plus a "View run" button linking to the run.
- Uses no third-party Action or CLI in the runner — only `curl` and `jq` with the Feishu REST API, keeping the runner dependency-free.

Credentials are stored as GitHub repository configuration:

| Secret or variable | Kind | Purpose |
|---|---|---|
| `DSH_FEISHU_APP_ID` | variable | Feishu self-built app ID (`cli_…`) |
| `DSH_FEISHU_APP_SECRET` | secret | Feishu self-built app secret |
| `DSH_FEISHU_OPEN_ID` | variable | Target user `open_id` (`ou_…`) |

The bot identity (`tenant_access_token`) is used rather than a user OAuth token, because it does not expire interactively and works in CI without browser-based login.

## Alternatives considered

### Why not use the `lark-cli` in the runner?

The `lark-cli` npm package requires installation and configuration steps in the runner, adding a Node dependency and startup latency. The Feishu bot REST API is two `curl` calls (token + send), needs no install, and runs in plain bash with `jq`. Using raw API calls keeps the runner minimal.

### Why not modify each existing workflow to add a notification step?

Every workflow would need a duplicated notification step or a reusable composite action. The `workflow_run` trigger centralizes notification logic in one file and fires after any triggering workflow completes, regardless of its internal structure. This avoids cross-workflow duplication and keeps existing workflows unchanged.

### Why not GitHub email notifications?

GitHub email notifications are per-user, not configurable per-workflow, and cannot be routed to a Feishu group or user. The Feishu channel provides in-band delivery to the team's primary messaging platform.

### Why not a group chat instead of P2P?

The initial configuration targets a single user (`open_id`). A group chat (`chat_id`) can be configured by changing `DSH_FEISHU_OPEN_ID` to a group chat ID and switching `receive_id_type` — the design does not lock P2P-only delivery.

## Consequences

CI failures and master-push successes now arrive as interactive Feishu cards with a direct link to the run. PR-success noise is suppressed by the `if` condition. The workflow adds one new GitHub Actions job per triggering workflow completion; this job is fast (two HTTP calls, no checkout, no build) and runs on `ubuntu-latest`. The credentials are repo-scoped and can be rotated by updating the GitHub variable/secret without touching the workflow file.