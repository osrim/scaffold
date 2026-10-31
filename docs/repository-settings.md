# Repository settings checklist

Use this checklist to set up a new repository, and again before you make a
repository public. A repository created from this template gets the files, but
not the settings, rulesets, environments, or secrets, so check each of them
with this list.

For a private repository, skip [Audit the content](#audit-the-content),
[Change the visibility](#change-the-visibility), and
[After you go public](#after-you-go-public). Steps marked **Optional** depend
on the repository.

## Before you start

- You need the admin role on the repository.
- A private repository needs GitHub Pro or a higher plan for rulesets and
  environments. Required reviewers on an environment need GitHub Enterprise.
- Run the `gh` commands from a clone of the repository. `gh api` fills
  `{owner}` and `{repo}` from the current clone.
- Breadcrumbs start from the repository page, for example
  **Settings > General > Pull Requests**.

## Audit the content

Making the repository public publishes its full history, its branches, tags,
issues, pull requests, releases, and Actions run logs. Once it's public, any
clone or fork keeps a copy, so assume nothing can be taken back.

- [ ] Scan the full Git history for secrets with
      [gitleaks](https://github.com/gitleaks/gitleaks). Rotate every secret it
      finds. Rewriting history doesn't help, because the secret is already
      exposed to anyone who had access.

  ```sh
  gitleaks git --redact --verbose
  ```

- [ ] Check the author names and emails in the history. To hide your email in
      future commits, use **Profile > Settings > Emails > Keep my email
      addresses private**.

  ```sh
  git log --all --format='%an <%ae>' | sort -u
  ```

- [ ] Delete stale branches and tags.

  ```sh
  git ls-remote --heads --tags origin
  git push origin --delete <branch>
  ```

- [ ] Review open and closed issues, pull requests, and releases for internal
      information.
- [ ] Delete Actions runs whose logs show internal information. Logs of past
      runs become public with the repository.

  ```sh
  gh run list --limit 200
  gh run delete <run-id>
  ```

- [ ] Check that `.gitignore` covers local files such as `.env` and
      `mise.local.toml`.
- [ ] Check that the repository has a `README.md` and a `LICENSE`.
- [ ] Add a `SECURITY.md` that tells reporters to use
      **Security > Report a vulnerability** instead of a public issue. You turn
      this on in [Advanced Security](#advanced-security).
- [ ] **Optional:** Add a `CONTRIBUTING.md`, a `CODE_OF_CONDUCT.md`, issue and
      pull request templates, and a `.github/CODEOWNERS` file.

## Pull requests

- [ ] In **Settings > General > Pull Requests**, set the merge options:
  - Clear **Allow merge commits**.
  - Select **Allow squash merging**, and set **Default commit message** to
    **Pull request title and description**.
  - Clear **Allow rebase merging**.
  - Select **Automatically delete head branches**.

  ```sh
  gh repo edit \
    --enable-merge-commit=false \
    --enable-rebase-merge=false \
    --enable-squash-merge \
    --squash-merge-commit-message pr-title-description \
    --delete-branch-on-merge
  ```

- [ ] **Optional:** Select **Allow auto-merge**, so Renovate merges through
      GitHub's native auto-merge.

  ```sh
  gh repo edit --enable-auto-merge
  ```

- [ ] **Optional:** Select **Always suggest updating pull request branches**.

  ```sh
  gh repo edit --allow-update-branch
  ```

## Protect the default branch

The ruleset lets only repository admins push to `main` directly. Everyone else
opens a pull request and merges it with a squash merge. Renovate merges its own
pull requests, so the ruleset requires no approvals.

- [ ] In **Settings > Rules > Rulesets**, check whether a `main` ruleset
      already exists.

  ```sh
  gh api repos/{owner}/{repo}/rulesets
  ```

- [ ] If it doesn't exist, create it in **Settings > Rules > Rulesets > New
      ruleset > New branch ruleset**:
  - **Ruleset name**: `main`.
  - **Enforcement status**: **Active**.
  - **Bypass list**: add **Repository admin**, set to **Always allow**.
  - **Target branches**: **Add target > Include default branch**.
  - Select **Restrict deletions** and **Block force pushes**.
  - Select **Require a pull request before merging**. Set **Required
    approvals** to `0`, and set **Allowed merge methods** to **Squash** only.

  ```sh
  gh api -X POST repos/{owner}/{repo}/rulesets --input - <<'EOF'
  {
    "name": "main",
    "target": "branch",
    "enforcement": "active",
    "bypass_actors": [
      { "actor_id": 5, "actor_type": "RepositoryRole", "bypass_mode": "always" }
    ],
    "conditions": {
      "ref_name": { "include": ["~DEFAULT_BRANCH"], "exclude": [] }
    },
    "rules": [
      { "type": "deletion" },
      { "type": "non_fast_forward" },
      {
        "type": "pull_request",
        "parameters": {
          "allowed_merge_methods": ["squash"],
          "required_approving_review_count": 0,
          "dismiss_stale_reviews_on_push": false,
          "require_code_owner_review": false,
          "require_last_push_approval": false,
          "required_review_thread_resolution": false
        }
      }
    ]
  }
  EOF
  ```

- [ ] **Optional:** If the repository has CI, select **Require status checks to
      pass** and add the CI jobs. Renovate then merges only after they pass.
- [ ] **Optional:** If other people review code, set **Required approvals** to
      `1`. Renovate's automatic merges then wait for a review.
- [ ] **Optional:** If the repository publishes releases, add a tag ruleset in
      **Settings > Rules > Rulesets > New ruleset > New tag ruleset** that
      targets `v*` and selects **Restrict updates** and **Restrict deletions**.

## Actions

- [ ] In **Settings > Actions > General > Actions permissions**, select
      **Allow _owner_, and select non-_owner_, actions and reusable
      workflows**. Then:
  - Select **Allow actions created by GitHub**.
  - Add each third-party action you use under **Allow specified actions and
    reusable workflows**, for example `jdx/mise-action@*`.
  - Select **Require actions to be pinned to a full-length commit SHA**.
    Renovate's `helpers:pinGitHubActionDigests` preset already pins them.

  ```sh
  gh api -X PUT repos/{owner}/{repo}/actions/permissions \
    -F enabled=true -f allowed_actions=selected -F sha_pinning_required=true

  gh api -X PUT repos/{owner}/{repo}/actions/permissions/selected-actions --input - <<'EOF'
  {
    "github_owned_allowed": true,
    "verified_allowed": false,
    "patterns_allowed": ["jdx/mise-action@*"]
  }
  EOF
  ```

- [ ] In **Settings > Actions > General > Workflow permissions**, select
      **Read repository contents and packages permissions**, and clear **Allow
      GitHub Actions to create and approve pull requests**.

  ```sh
  gh api -X PUT repos/{owner}/{repo}/actions/permissions/workflow \
    -f default_workflow_permissions=read -F can_approve_pull_request_reviews=false
  ```

- [ ] **Optional:** In **Settings > Actions > General > Artifact and log
      retention**, lower the retention, for example to 30 days.

  ```sh
  gh api -X PUT repos/{owner}/{repo}/actions/permissions/artifact-and-log-retention -F days=30
  ```

- [ ] Review each file in `.github/workflows/`:
  - Set `permissions:` at the top of each workflow, and grant more per job only
    where the job needs it.
  - Don't check out or run pull request code in a `pull_request_target`
    workflow.
  - Don't put `${{ github.event.* }}` values directly in a `run:` script. Pass
    them through `env:` instead.
  - Don't use self-hosted runners. A fork's pull request can run code on them.
  - Run [zizmor](https://github.com/zizmorcore/zizmor) to find other issues.

  ```sh
  zizmor .github/workflows
  ```

## Environments and secrets

Any workflow on any branch can read repository secrets. Only a job that names
an environment can read that environment's secrets, and the environment limits
which branches can deploy and who must approve.

- [ ] Create an environment in **Settings > Environments > New environment**,
      for example `production`.

  ```sh
  gh api -X PUT repos/{owner}/{repo}/environments/production --input - <<'EOF'
  {
    "deployment_branch_policy": {
      "protected_branches": false,
      "custom_branch_policies": true
    }
  }
  EOF
  ```

- [ ] In **Settings > Environments > production > Deployment branches and
      tags**, select **Selected branches and tags** and add `main`.

  ```sh
  gh api -X POST repos/{owner}/{repo}/environments/production/deployment-branch-policies \
    -f name=main -f type=branch
  ```

- [ ] **Optional:** In **Settings > Environments > production > Deployment
      protection rules**, select **Required reviewers** and add yourself, so
      each deployment waits for your approval.

  ```sh
  gh api -X PUT repos/{owner}/{repo}/environments/production --input - <<EOF
  {
    "deployment_branch_policy": {
      "protected_branches": false,
      "custom_branch_policies": true
    },
    "reviewers": [{ "type": "User", "id": $(gh api user --jq .id) }]
  }
  EOF
  ```

- [ ] Move each secret from **Settings > Secrets and variables > Actions >
      Repository secrets** to **Settings > Environments > production >
      Environment secrets**. Then delete the repository secret.

  ```sh
  gh secret list
  gh secret set <NAME> --env production
  gh secret delete <NAME>
  ```

- [ ] Do the same for variables that only deployments use.

  ```sh
  gh variable list
  gh variable set <NAME> --env production
  gh variable delete <NAME>
  ```

- [ ] Add `environment: production` to each job that uses the secrets.

  ```yaml
  jobs:
    deploy:
      environment: production
  ```

- [ ] **Optional:** If a workflow deploys to a cloud provider, replace
      long-lived keys with
      [OpenID Connect](https://docs.github.com/en/actions/concepts/security/openid-connect).

## Access and integrations

- [ ] In **Settings > Collaborators**, remove people who no longer need
      access.

  ```sh
  gh api repos/{owner}/{repo}/collaborators --jq '.[] | {login, role_name}'
  ```

- [ ] In **Settings > Deploy keys**, delete unused keys, and check that the
      others are read-only.

  ```sh
  gh repo deploy-key list
  gh repo deploy-key delete <key-id>
  ```

- [ ] In **Settings > Webhooks**, delete unused webhooks.

  ```sh
  gh api repos/{owner}/{repo}/hooks --jq '.[] | {id, url: .config.url, events}'
  ```

- [ ] In **Settings > GitHub Apps**, remove apps the repository doesn't use,
      and check that Renovate is installed.
- [ ] **Optional:** If the repository publishes to GitHub Packages, set each
      package's visibility on its package settings page. Package visibility
      doesn't follow the repository.
- [ ] **Optional:** If the repository uses GitHub Pages, check what
      **Settings > Pages** publishes.

## Repository features

- [ ] In **Settings > General > Features**, clear the features the repository
      doesn't use. A public wiki lets any GitHub user edit it unless you select
      **Restrict editing to collaborators only**.

  ```sh
  gh repo edit --enable-wiki=false --enable-projects=false --enable-discussions=false
  ```

- [ ] On the repository page, select the gear next to **About**, and set the
      description, website, and topics.

  ```sh
  gh repo edit --description "<description>" --homepage "<url>" --add-topic <topic>
  ```

- [ ] **Optional:** If the repository publishes releases, select **Enable
      release immutability** in **Settings > General > Releases**, so a
      published release's assets and tag can't change.

  ```sh
  gh api -X PUT repos/{owner}/{repo}/immutable-releases
  ```

## Change the visibility

- [ ] In **Settings > General > Danger Zone > Change repository visibility**,
      select **Change visibility > Make public**.

  ```sh
  gh repo edit --visibility public --accept-visibility-change-consequences
  ```

## After you go public

### Advanced Security

Renovate handles dependency updates, including security fixes. It reads the
Dependabot alerts to open security pull requests, so turn the alerts on and
leave the Dependabot pull requests off to avoid duplicate pull requests.

- [ ] In **Settings > Advanced Security > Dependency graph**, check that it's
      **On**. You can only change it in the UI.
- [ ] In **Settings > Advanced Security > Dependabot alerts**, turn it **On**.

  ```sh
  gh api -X PUT repos/{owner}/{repo}/vulnerability-alerts
  ```

- [ ] In **Settings > Advanced Security > Dependabot security updates**, leave
      it **Off**. **Grouped security updates** then stays off too.

  ```sh
  gh api -X DELETE repos/{owner}/{repo}/automated-security-fixes
  ```

- [ ] In **Settings > Advanced Security > Dependabot version updates**, don't
      create a `dependabot.yml`.
- [ ] In **Settings > Advanced Security > Private vulnerability reporting**,
      select **Enable**.

  ```sh
  gh api -X PUT repos/{owner}/{repo}/private-vulnerability-reporting
  ```

- [ ] In **Settings > Advanced Security > Secret Protection**, select
      **Enable**, then enable **Push protection**.

  ```sh
  gh repo edit --enable-secret-scanning --enable-secret-scanning-push-protection
  ```

- [ ] **Optional:** If CodeQL supports the repository's languages, go to
      **Settings > Advanced Security > Code scanning > CodeQL analysis** and
      select **Set up > Default**.

  ```sh
  gh api -X PATCH repos/{owner}/{repo}/code-scanning/default-setup -f state=configured
  ```

### Fork pull requests

- [ ] In **Settings > Actions > General > Approval for running fork pull
      request workflows from contributors**, select **Require approval for all
      external contributors**. A fork's pull request then runs no workflow
      until you approve it.

  ```sh
  gh api -X PUT repos/{owner}/{repo}/actions/permissions/fork-pr-contributor-approval \
    -f approval_policy=all_external_contributors
  ```

### Moderation

- [ ] **Optional:** In **Settings > Moderation options > Code review limits**,
      select **Limit to users explicitly granted read or higher access**, so
      only collaborators can approve or request changes.
- [ ] **Optional:** If the repository gets spam, set a temporary limit in
      **Settings > Moderation options > Interaction limits**.

  ```sh
  gh api -X PUT repos/{owner}/{repo}/interaction-limits \
    -f limit=existing_users -f expiry=one_week
  ```

### Check the result

- [ ] Open **Insights > Community standards**, and add the missing files that
      apply.

  ```sh
  gh api repos/{owner}/{repo}/community/profile --jq '{health_percentage, files}'
  ```

- [ ] Open the repository in a private browser window, signed out, and check
      what a visitor sees.
