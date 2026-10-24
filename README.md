# admin

Repository settings, rulesets and shared tooling for the [Spillgebees](https://github.com/Spillgebees) organization and the `igotinfected` personal repos. Nothing in here is specific to them beyond names and IDs, so feel free to copy any of it for your own org.

## What's in here

- **safe-settings** (`.github/settings.yml`, `.github/suborgs/*.yml`, `.github/repos/<repo>.yml`) enforces repository settings, branch and tag rulesets, deployment environments and labels on every org repo via [github/safe-settings](https://github.com/github/safe-settings)
- **Sync workflow** (`.github/workflows/safe-settings-sync.yml`) runs safe-settings every 6 hours, on config changes and on manual trigger
- **Personal repo rulesets** (`rulesets/personal-*.json`), the same rules for repos on a personal account, applied by hand
- **Reusable zizmor workflow** (`.github/workflows/zizmor.yml`) audits GitHub Actions workflows, repos call it with `uses: Spillgebees/admin/.github/workflows/zizmor.yml@<sha>`
- **Renovate preset** (`default.json`), extended by every repo via `local>Spillgebees/admin`

## The model

The setup assumes one owner, plus bot apps (agents, review bots, Renovate) that open PRs and push branches. The goals:

- the owner merges their own PRs without a second person, but never pushes straight to the default branch
- a bot PR merges only once the owner approved exactly that change, and the owner doesn't need to bypass anything to merge it
- only the owner can create, move or delete tags, so only the owner can trigger a release
- publish and deploy credentials only reach jobs running on the default branch or a release tag, never a branch a bot pushed

### Merging

| PR from | How it merges |
|---|---|
| the owner | nobody else can approve it, so merge with the merge box's bypass option |
| a bot app | the owner approves it, then merges normally or lets auto-merge finish it |

The default-branch ruleset requires 1 approval from a code owner and approval of the most recent push. Approvals are dismissed on new pushes and when the merge base moves, for example after a retarget. `CODEOWNERS` lists only the owner, and apps can't be code owners, so a bot approving another bot's PR doesn't count.

An approval means the PR may merge. An app with Contents write can merge a PR once it's approved, but only the exact commits that were approved.

Every repo needs a `CODEOWNERS` file with `* @<owner>`. Without one the code owner check does nothing and any approval from someone with write access counts, apps included.

Required status checks aren't set centrally since check names differ per repo. Add a `required_status_checks` rule per repo, otherwise auto-merge merges as soon as the PR is approved, whatever state CI is in.

### Tags and releases

The tag ruleset lets only bypass actors create, move or delete tags. Release workflows build from the release's tag, so a bot with Contents write can't tag a commit of its choosing and release it.

A bot can still publish a draft release, or re-create a release on an existing tag. That's why publish jobs also run in an environment with the owner as required reviewer, see below.

### Secrets

Pushing a branch only takes Contents and Workflows write, and a workflow on that branch can read every repo and org secret. So publish and deploy secrets go in [environments](https://docs.github.com/en/actions/reference/workflows-and-actions/deployments-and-environments), never at repo or org level:

- release jobs use an environment limited to tags, with the owner as required reviewer
- deploy jobs on the default branch use an environment limited to it, with a reviewer only if nothing scheduled deploys through it
- `github-pages` is limited to the default branch. A leftover `gh-pages` rule would let anyone who can push that branch deploy the site

For NuGet, [trusted publishing](https://learn.microsoft.com/en-us/nuget/nuget-org/trusted-publishing) removes the API key entirely. Pin the nuget.org policy to the repo, the workflow file and the release environment, so only a job the environment let through gets a short-lived key:

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
        # nuget.org profile name, not the email address
        user: ${{ vars.NUGET_USER }}
    - run: dotnet nuget push "*.nupkg" --api-key "$NUGET_API_KEY" --source https://api.nuget.org/v3/index.json
      env:
        NUGET_API_KEY: ${{ steps.login.outputs.NUGET_API_KEY }}
```

Merge the safe-settings change that creates an environment before any workflow references it. GitHub creates a referenced environment that doesn't exist yet, without any protection.

Environments don't cover `GITHUB_TOKEN`. A pushed workflow can still ask for any permission the repo allows, for example `packages: write`.

## safe-settings

### Prerequisites

A **GitHub App** installed on the organization with these permissions:

- **Repository**: Actions (Read), Administration (R&W), Checks (R&W), Commit statuses (R&W), Contents (R&W), Environments (Read), Issues (R&W), Metadata (Read), Pull requests (R&W)
- **Organization**: Administration (R&W), Members (R&W)

Actions and Environments are only read to diff environments. Without them the environments plugin fails on every repo that has one and changes nothing.

Configure these in this repo's Actions settings. The private key goes in the `safe-settings` environment, limited to `main`, so only the sync on `main` can read it:

| Type | Name | Value |
|---|---|---|
| Variable | `SAFE_SETTINGS_GH_ORG` | org name |
| Variable | `SAFE_SETTINGS_APP_ID` | GitHub App ID |
| Environment secret (`safe-settings`) | `SAFE_SETTINGS_PRIVATE_KEY` | GitHub App private key (`.pem` contents) |

### What it enforces

**Repository settings** (`.github/settings.yml`): squash merge only, auto-delete head branches, auto-merge enabled.

**Rulesets** (`.github/suborgs/all-repos.yml`, every repo):
- `default-branch-protection`: pull request with 1 code owner approval and last push approval, stale approvals dismissed, conversations resolved, linear history, no force pushes or deletion. The org admin bypasses in `pull_request` mode only
- `tag-protection`: only the org admin (mode `always`) can create, move or delete tags

**Environments** (`.github/repos/<repo>.yml`), see below.

**Labels** (`.github/suborgs/all-repos.yml`), the org label set. safe-settings **deletes** any label that isn't in the list.

### Environments

safe-settings 2.1.17 manages environments per repo. Once a repo has an `environments:` list, safe-settings **deletes** every environment on that repo that isn't in it, secrets included. So:

- list every environment the repo has, including the ones GitHub creates (`github-pages`, `copilot`)
- add a new environment to the config first and let the sync create it, then add its secrets
- never put `environments:` in a suborg, it would delete environments on repos that don't list them (this repo's `safe-settings` environment is managed by hand for that reason)

safe-settings can't set secrets. Add them under **Settings → Environments**, or with `gh secret set SECRET_NAME --repo OWNER/REPO_NAME --env ENV_NAME`.

### Known limitations

- Rulesets and environment protection rules on the Free plan only work on public repos.
- Org-level rulesets need the Team plan, so the rulesets are repository rulesets applied to every repo through the `*` suborg.

### Configuration hierarchy

Highest precedence first:

1. Repo: `.github/repos/<repo-name>.yml`
2. Suborg: `.github/suborgs/<name>.yml` (`all-repos.yml` matches every repo)
3. Org defaults: `.github/settings.yml`

> **Note:** `rulesets:` in `.github/settings.yml` would create an *organization* ruleset (Team plan only) and is stripped before per-repo processing. Keep rulesets in a suborg or repo config.

### Sync schedule

| Trigger | When |
|---|---|
| Push to `main` | When `.github/settings.yml`, `.github/suborgs/**`, `.github/repos/**`, `deployment-settings.yml`, or the sync workflow changes |
| Cron | Every 6 hours (drift prevention) |
| Manual | `workflow_dispatch` from the Actions tab |

## Personal repos

safe-settings only manages orgs, so personal repos get the same rules by hand. `OrganizationAdmin` doesn't exist there, so the JSON files use a `User` bypass actor. Replace `15004223` with your own user ID (`gh api user -q .id`).

- [`rulesets/personal-default-branch-protection.json`](rulesets/personal-default-branch-protection.json): the org's default-branch rules without linear history and conversation resolution, bypass mode `pull_request`
- [`rulesets/personal-tag-protection.json`](rulesets/personal-tag-protection.json): the org's tag rules, bypass mode `always`

Apply both, replacing `OWNER` and `REPO_NAME`:

```sh
gh api --method POST repos/OWNER/REPO_NAME/rulesets --input rulesets/personal-default-branch-protection.json
gh api --method POST repos/OWNER/REPO_NAME/rulesets --input rulesets/personal-tag-protection.json
```

Or import them under **Settings → Rules → Rulesets → New ruleset → Import a ruleset**. Never put bot apps on the bypass list. Rulesets layer with classic branch protection, so remove any classic rule once the ruleset is in place.

Then add `CODEOWNERS` with `* @OWNER`, and stop `GITHUB_TOKEN` from approving PRs. The org-level setting doesn't apply to personal repos:

```sh
gh api --method PUT repos/OWNER/REPO_NAME/actions/permissions/workflow -f default_workflow_permissions=read -F can_approve_pull_request_reviews=false
```

Environments are by hand too. One limited to the default branch:

```sh
echo '{"deployment_branch_policy":{"protected_branches":false,"custom_branch_policies":true}}' \
  | gh api --method PUT repos/OWNER/REPO_NAME/environments/ENV_NAME --input -
gh api --method POST repos/OWNER/REPO_NAME/environments/ENV_NAME/deployment-branch-policies -f name=main -f type=branch
```

A release environment limited to tags, with you as reviewer. `**/*` matches every tag, `*` alone would skip tags with a `/`:

```sh
gh api user -q '{reviewers:[{type:"User",id:.id}],deployment_branch_policy:{protected_branches:false,custom_branch_policies:true}}' \
  | gh api --method PUT repos/OWNER/REPO_NAME/environments/nuget --input -
gh api --method POST repos/OWNER/REPO_NAME/environments/nuget/deployment-branch-policies -f name='**/*' -f type=tag
```

## Renovate

Shared preset for dependency updates. Each repo extends it:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "local>Spillgebees/admin"
  ]
}
```

See [`default.json`](default.json) for the full preset.
