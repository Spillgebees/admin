# admin

Central configuration hub for the [Spillgebees](https://github.com/Spillgebees) organization.

## What's in here

- **Renovate shared preset** (`default.json`) — extended by all repos via `local>Spillgebees/admin`
- **Personal repo rulesets** (`rulesets/personal-*.json`), the same default-branch and tag rules for `igotinfected` repos, applied by hand
- **safe-settings** (`.github/settings.yml`, `.github/suborgs/*.yml`, optional per-repo `.github/repos/<repo>.yml`) enforces repository settings, default-branch and tag rulesets, deployment environments, and labels across all org repos via [github/safe-settings](https://github.com/github/safe-settings)
- **GitHub Actions workflow** (`.github/workflows/safe-settings-sync.yml`) — runs safe-settings on a schedule (every 6 hours), on config changes, and on manual trigger
- **Reusable zizmor workflow** (`.github/workflows/zizmor.yml`) audits GitHub Actions workflows, org repos call it with `uses: Spillgebees/admin/.github/workflows/zizmor.yml@<sha>`

## safe-settings

Policy-as-code enforcement for all Spillgebees repositories. Runs via GitHub Actions — no self-hosted infrastructure.

### Prerequisites

The workflow requires a **GitHub App** installed on the Spillgebees organization with the following permissions:

- **Repository**: Actions (Read), Administration (R&W), Checks (R&W), Commit statuses (R&W), Contents (R&W), Environments (Read), Issues (R&W), Metadata (Read), Pull requests (R&W)
- **Organization**: Administration (R&W), Members (R&W)

Actions and Environments are only read to diff environments. Without them the environments plugin errors on every repo with a `.github/repos/<repo>.yml` and changes nothing.

Configure these in the repo's Actions settings. The private key goes in the `safe-settings` environment, whose deployment branches are restricted to `main`, so only the sync on `main` can read it:

| Type | Name | Value |
|---|---|---|
| Variable | `SAFE_SETTINGS_GH_ORG` | `Spillgebees` |
| Variable | `SAFE_SETTINGS_APP_ID` | GitHub App ID |
| Environment secret (`safe-settings`) | `SAFE_SETTINGS_PRIVATE_KEY` | GitHub App private key (`.pem` contents) |

### What it enforces

**Repository settings** (org-wide defaults):
- Squash merge only (merge commits and rebase disabled)
- Auto-delete head branches after merge
- Auto-merge enabled

**Default-branch ruleset** (`default-branch-protection`, defined in `.github/suborgs/all-repos.yml`, applied to every repo):
- Restrict updates: only bypass actors can update the default branch, and merging a PR counts as an update. Without this, anyone with write access, including GitHub Apps with Contents write such as agent bots, could merge PRs since no approvals are required
- Require a pull request before merging (no approvals required while the owner is the only person with write access; see the comment in `all-repos.yml`)
- Dismiss stale reviews on new push
- Require conversation resolution
- Require linear history
- Block force pushes and branch deletion
- **Org admin can bypass PR requirements but cannot push directly to the default branch** (bypass mode `pull_request`). Because of the update restriction, every merge goes through the merge box's bypass option, auto-merge most likely won't complete, and stacked PRs merge one at a time from the bottom
- Status checks are not centrally required — add a `required_status_checks` rule per repo when needed

**Tag ruleset** (`tag-protection`, also in `all-repos.yml`, applied to every repo):
- Only bypass actors (org admin, mode `always`) can create, move or delete any tag
- Releases publish NuGet packages from their tag, so this keeps anyone with Contents write, bot apps included, from tagging a commit of their choosing and releasing it

**Environments** (`.github/repos/<repo>.yml`, see [Environments](#environments)):
- `nuget` on `Blazor.Map`, `Blazor.RichTextEditor`, `Blazor.Docs.Sdk` and `Transmodel`: tags only, `igotinfected` must approve every run
- `coolify` on `lux-railway-map-overlay`: `main` only, no reviewer so the monthly scheduled run still deploys
- `github-pages`: `main` only

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

### Environments

Anyone who can push a branch can also push a workflow, and that workflow can read every repo and org secret the repo has access to. Pushing a branch only needs Contents and Workflows write, which bot apps have. Environment secrets are only handed to jobs that run on a branch or tag the environment allows, after any required reviewer approves, so every publish or deploy secret lives in an environment, never at repo or org level.

safe-settings manages environments per repo in `.github/repos/<repo>.yml`. Once a repo has an `environments:` list, safe-settings **deletes** every environment on that repo that isn't in it, secrets included. So:

- list every environment the repo has, including the ones GitHub creates (`github-pages`, `copilot`)
- add a new environment here first and let the sync create it, then add its secrets by hand
- never put `environments:` in a suborg, it would apply to repos with environments that aren't listed and delete them (`admin`'s `safe-settings` environment is managed by hand for that reason)

safe-settings can't set secrets. After the sync creates an environment, add its secrets under **Settings → Environments**, or with `gh secret set SECRET_NAME --repo Spillgebees/REPO_NAME --env ENV_NAME`.

NuGet publishing uses [trusted publishing](https://learn.microsoft.com/en-us/nuget/nuget-org/trusted-publishing) instead of an API key. The nuget.org policy is pinned to the repo, the workflow file and the `nuget` environment, so only a job that the environment let through can get a key. A publish job looks like this:

```yaml
publish:
  needs: [build-and-test]
  runs-on: ubuntu-latest
  environment: nuget
  permissions:
    id-token: write
  steps:
    # download the packages ...
    - id: login
      uses: NuGet/login@8d196754b4036150537f80ac539e15c2f1028841 # v1.2.0
      with:
        user: igotinfected
    - run: dotnet nuget push "*.nupkg" --api-key "$NUGET_API_KEY" --source https://api.nuget.org/v3/index.json
      env:
        NUGET_API_KEY: ${{ steps.login.outputs.NUGET_API_KEY }}
```

### Sync schedule

| Trigger | When |
|---|---|
| Push to `main` | When `.github/settings.yml`, `.github/suborgs/**`, `.github/repos/**`, `deployment-settings.yml`, or the sync workflow changes |
| Cron | Every 6 hours (drift prevention) |
| Manual | `workflow_dispatch` — trigger from the Actions tab |

## Personal repos

safe-settings only manages the org, so `igotinfected` repos get the ruleset by hand from [`rulesets/personal-default-branch-protection.json`](rulesets/personal-default-branch-protection.json). It has the same `update`, `pull_request` (0 approvals), `non_fast_forward` and `deletion` rules, without the org's linear history and conversation resolution. Bypass goes to the `igotinfected` user (`15004223`) with mode `pull_request`, since `OrganizationAdmin` doesn't exist on personal repos.

[`rulesets/personal-tag-protection.json`](rulesets/personal-tag-protection.json) is the org's tag ruleset with the same `User` bypass, in mode `always`.

Apply both to one repo, replacing `REPO_NAME`:

```sh
gh api --method POST repos/igotinfected/REPO_NAME/rulesets --input rulesets/personal-default-branch-protection.json
gh api --method POST repos/igotinfected/REPO_NAME/rulesets --input rulesets/personal-tag-protection.json
```

Or import the files under **Settings → Rules → Rulesets → New ruleset → Import a ruleset**. Rulesets on a free personal account only work on public repos. Bot apps never go on the bypass list.

Personal repos get environments by hand too. Create one limited to `main`, replacing `REPO_NAME` and `ENV_NAME`:

```sh
echo '{"deployment_branch_policy":{"protected_branches":false,"custom_branch_policies":true}}' \
  | gh api --method PUT repos/igotinfected/REPO_NAME/environments/ENV_NAME --input -
gh api --method POST repos/igotinfected/REPO_NAME/environments/ENV_NAME/deployment-branch-policies -f name=main -f type=branch
```

For a release environment limited to tags with `igotinfected` as reviewer:

```sh
echo '{"reviewers":[{"type":"User","id":15004223}],"deployment_branch_policy":{"protected_branches":false,"custom_branch_policies":true}}' \
  | gh api --method PUT repos/igotinfected/REPO_NAME/environments/nuget --input -
gh api --method POST repos/igotinfected/REPO_NAME/environments/nuget/deployment-branch-policies -f name='*' -f type=tag
```

Rulesets layer with classic branch protection, so drop any classic rule on the repo once the ruleset is in place (`burser` and `MatomoProvider` still have one).

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
