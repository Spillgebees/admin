# admin

Central configuration hub for the [Spillgebees](https://github.com/Spillgebees) organization.

## What's in here

- **Renovate shared preset** (`default.json`) — extended by all repos via `local>Spillgebees/admin`
- **safe-settings** (`.github/settings.yml`, `.github/repos/*.yml`) — enforces repository settings and branch protection across all org repos via [github/safe-settings](https://github.com/github/safe-settings)
- **GitHub Actions workflow** (`.github/workflows/safe-settings-sync.yml`) — runs safe-settings on a schedule (every 6 hours), on config changes, and on manual trigger

## safe-settings

Policy-as-code enforcement for all Spillgebees repositories. Runs via GitHub Actions — no self-hosted infrastructure.

### Prerequisites

The workflow requires a **GitHub App** installed on the Spillgebees organization with the following permissions:

- **Repository**: Administration (R&W), Checks (R&W), Commit statuses (R&W), Contents (R&W), Issues (R&W), Metadata (Read), Pull requests (R&W)
- **Organization**: Administration (R&W), Members (R&W)

Configure these in the repo's Actions settings:

| Type | Name | Value |
|---|---|---|
| Variable | `SAFE_SETTINGS_GH_ORG` | `Spillgebees` |
| Variable | `SAFE_SETTINGS_APP_ID` | GitHub App ID |
| Secret | `SAFE_SETTINGS_PRIVATE_KEY` | GitHub App private key (`.pem` contents) |

### What it enforces

**Repository settings** (org-wide defaults):
- Squash merge only (merge commits and rebase disabled)
- Auto-delete head branches after merge
- Auto-merge enabled

**Branch protection** on `main`:
- Require pull request with 1 approval
- Dismiss stale reviews on new push
- Require CODEOWNERS review
- Require approval from someone other than the last pusher
- Require conversation resolution
- Require linear history
- Block force pushes and branch deletion
- Org admin can bypass PR requirements (`enforce_admins: false`)
- Status checks are managed per-repo — existing checks are preserved, new repos start with none

### Known limitations

- **Private repos on the Free plan** cannot have branch protection. safe-settings will log errors for those repos, which is expected.
- **Org-level rulesets** (including merge queue) require the GitHub Team plan. Branch protection rules are used instead.

### Configuration hierarchy

Settings are merged with **highest precedence first**:

1. **Repo-level**: `.github/repos/<repo-name>.yml`
2. **Org-level defaults**: `.github/settings.yml`

No per-repo overrides are configured yet. To add repo-specific settings (e.g., required status checks), create `.github/repos/<repo-name>.yml`.

### Sync schedule

| Trigger | When |
|---|---|
| Push to `main` | When `.github/settings.yml`, `.github/repos/**`, or `deployment-settings.yml` changes |
| Cron | Every 6 hours (drift prevention) |
| Manual | `workflow_dispatch` — trigger from the Actions tab |

## Renovate

Shared preset for automated dependency updates. Each repo extends it:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "local>Spillgebees/admin"
  ]
}
```

See [`default.json`](default.json) for the full preset configuration.
