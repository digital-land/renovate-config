# renovate-config

Central setup for automated dependency updates across `digital-land`, using [Renovate](https://docs.renovatebot.com/).

This repo holds:

- the **shared Renovate config** (preset) that every opted-in repo extends
- the **list of repos** Renovate runs on
- the **scheduled workflow** that runs Renovate across those repos and adds each repo's Dependency Dashboard to our project board

Renovate is open source and runs entirely on our GitHub Actions runners. No code or data goes to a third-party service.

> **Developers:** see the [developer guide](docs/developer-guide.md) for what Renovate does in your repo and how to work with its PRs and issues.

---

## Contents

```
.
├── default.json                              shared preset, extended by every repo
├── repos.json                                repos Renovate runs on
├── docs/
│   └── developer-guide.md                    guide for developers
└── .github/workflows/
    ├── renovate.yml                          runs Renovate on a schedule, then updates the board
    └── test.yml                              checks default.json on pull requests
```

## How it works

1. `renovate.yml` runs on weekdays at 06:00, 10:00 and 14:00 UTC. It authenticates as the **Rennovate-App** GitHub App (`rennovate-app`) and runs Renovate against every repo listed in **`repos.json`**. Each listed repo must also have the App installed.
2. Each repo has a `renovate.json` that extends this repo's preset:
   ```json
   {
     "$schema": "https://docs.renovatebot.com/renovate-schema.json",
     "extends": ["local>digital-land/renovate-config"]
   }
   ```
3. Renovate opens routine minor/patch updates once a week (Monday morning), with **one PR per type** (Python, JavaScript, CI, Docker), and keeps a **Dependency Dashboard** issue in each repo. **Major upgrades are only listed** on the dashboard; a PR is created only when someone ticks one. **Security fixes** are grouped into one PR per repo and raised on the next run, whatever the schedule. **Python itself** is held at 3.13 or below.
4. At the end of each run, `renovate.yml` adds every listed repo's Dependency Dashboard to the [project board](https://github.com/orgs/digital-land/projects/44), so each repo has **one card**. The dashboard closes itself when nothing is pending (moving the card to Done) and reopens when there's new work. This step is skipped if `UPGRADES_PROJECT_ID` isn't set.
5. `test.yml` (the **Test** workflow) runs Renovate's config validator on every pull request to this repo.

**Automerge is currently off.** Every Renovate PR needs a developer to review and merge it. We plan to switch it on for minor/patch updates once we're confident in the process.

## Configuration

### Actions variables and secrets

| Name | Type | Purpose |
|---|---|---|
| `RENOVATE_APP_ID` | Variable | ID of the Rennovate-App GitHub App |
| `RENOVATE_APP_PRIVATE_KEY` | Secret | Private key for the App (full `.pem` contents). Not the App's client secret. |
| `UPGRADES_PROJECT_ID` | Variable | ID of the [dependency maintenance project board](https://github.com/orgs/digital-land/projects/44) (`PVT_...`, not the number in its URL). To look it up: `gh project view 44 --owner digital-land --format json --jq .id` (needs `gh auth refresh -s read:project` first). |

### GitHub App

The App (**Rennovate-App**, slug `rennovate-app`) is owned by the organisation and installed on **selected repositories** only. That installation is the hard limit on what Renovate can access. It must include **this repo** as well as every repo in `repos.json`, because Renovate reads the shared preset from here.

Permissions:

- **Repository:** Contents, Issues, Pull requests, Workflows, Checks, Commit statuses (read and write); Administration, Dependabot alerts, Metadata (read)
- **Organisation:** Members (read); Projects (read and write)

Webhooks are disabled, because Renovate runs on a schedule.

## Common tasks

### Add a repo
1. Add the repo to the App installation: **Org settings → GitHub Apps → Rennovate-App → Configure**.
2. Add the repo to `repos.json` in a PR, and merge it:
   ```json
   { "repo": "digital-land/<repo-name>" }
   ```
3. Run the **Renovate** workflow (or wait for the next scheduled run). Renovate opens an onboarding PR in the repo. Review it, add any repo-specific settings (for example for pip-tools; see the developer guide), and merge it.

### Remove a repo
Remove it from `repos.json`. To fully revoke access, also remove the repo from the App installation.

### Change the shared policy
Edit `default.json` and open a PR. Changes apply to every repo on Renovate's next run. Repos can override anything in their own `renovate.json`.

### Run Renovate now
**Actions → Renovate → Run workflow.** Use this after merging config changes, or to onboard a repo straight away. You can enter a single repo from `repos.json` to run just that one, and tick **Create PRs now** to skip the Monday schedule for this run.

### Debug a problem
Open the latest **Renovate** workflow run and search the logs for the repo's name. Renovate processes repos one after another, so each repo's output appears in its own section of the log. Debug logging (`LOG_LEVEL: debug` in `renovate.yml`) is currently on while the setup beds in. Remove it once things are running smoothly, as it makes the public logs much more detailed.

### Rotate the App's private key
1. On the App's settings page, generate a new private key.
2. Update the `RENOVATE_APP_PRIVATE_KEY` secret.
3. Run the workflow to check it works, then delete the old key from the App.

## Notes

- **Public repo:** this repo is public, so its Actions logs are public. Renovate's logs name every repo it processes and their dependencies. **Only install the App on public repos.** Before adding a private repo, make this repo private.
- **Actions minutes:** Actions minutes are free for public repos. If this repo becomes private, check usage under **Org settings → Billing** and reduce the run frequency in `renovate.yml` if needed. Fewer runs only slows down dashboard checkbox requests and security fixes; the weekly PRs still open on Mondays.
- **Dependabot:** keep Dependabot **alerts** enabled (Renovate uses them for security PRs), but don't add `dependabot.yml` files, as Dependabot version updates would duplicate Renovate's PRs.
- **Python repos using pip-tools** need extra settings in their `renovate.json`. See the developer guide.
- **Run length:** the App's token lasts one hour, and the job times out after 60 minutes. If runs start getting close to that as more repos are added, split the list across separate jobs or runs.

## Ownership

Maintained by: _team or people responsible_. Questions or problems: _channel or contact_.
