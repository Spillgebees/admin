# admin

Central configuration hub for the [Spillgebees](https://github.com/Spillgebees) organization.

## What's in here

- **Renovate shared preset** (`default.json`) — extended by all repos via `local>Spillgebees/admin`
- **safe-settings** (`.github/settings.yml`, `.github/suborgs/*.yml`, `.github/repos/*.yml`) — enforces repository settings, a default-branch ruleset, and labels across all org repos via [github/safe-settings](https://github.com/github/safe-settings)
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

**Default-branch ruleset** (`default-branch-protection`, defined in `.github/suborgs/all-repos.yml`, applied to every repo):
- Require a pull request with 1 approval before merging
- Dismiss stale reviews on new push
- Require CODEOWNERS review
- Require approval from someone other than the last pusher
- Require conversation resolution
- Require linear history
- Block force pushes and branch deletion
- **Org admin can bypass PR requirements but cannot push directly to the default branch** (bypass mode `pull_request`)
- Status checks are not centrally required — add a `required_status_checks` rule per repo when needed

**Labels** (`.github/suborgs/all-repos.yml`) — the org-standard issue/PR label set, synced to every repo. This is the single source of truth: safe-settings **deletes** any label on a repo that is not in the list.

#### Why a ruleset instead of classic branch protection?

The old setup used classic branch protection with `enforce_admins: false`, which let the org admin **push directly to `main`** — an accidental push to `main` is exactly what that allowed. A repository ruleset with bypass mode `pull_request` keeps the flexibility (admins can still merge a PR without an approval, merge their own PR, or get past a stuck check) while blocking direct pushes for everyone, admins included — a direct push isn't a pull request, so the `pull_request` rule blocks it and the admin's PR-only bypass doesn't apply.

### Known limitations

- **Private repos on the Free plan** cannot have rulesets or branch protection. All current repos are public, so this does not apply today.
- **Org-level rulesets** require the GitHub Team plan. We use **repository rulesets** (applied per repo via the `*` suborg) instead, which are free for public repos.

### Configuration hierarchy

Settings are merged with **highest precedence first**:

1. **Repo-level**: `.github/repos/<repo-name>.yml`
2. **Suborg-level**: `.github/suborgs/<name>.yml` (e.g. `all-repos.yml`, which matches every repo)
3. **Org-level defaults**: `.github/settings.yml`

To add repo-specific settings (e.g., required status checks), create `.github/repos/<repo-name>.yml`.

> **Note:** `rulesets:` in `.github/settings.yml` would create an *organization* ruleset (Team plan only) and is stripped before per-repo processing. Keep rulesets in a suborg or repo config so they are created as *repository* rulesets.

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
