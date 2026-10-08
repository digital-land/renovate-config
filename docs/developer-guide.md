# Dependency updates with Renovate: developer guide

This guide explains how automated dependency updates work across our repositories, what you'll see in your repo, and what you need to do when an update needs attention.

---

## In short

- **Renovate** keeps our dependencies up to date by opening pull requests.
- Routine **minor and patch** updates arrive **once a week**, with one PR per type (Python, JavaScript, CI, Docker), for a developer to review and merge. Nothing merges automatically.
- **Major** upgrades are **only listed** on the dashboard. No PR is created until someone on the team ticks one.
- **Security** fixes arrive as soon as they're found, whatever the schedule, grouped into one PR (plus a second for any that need a major upgrade).
- **Python itself** stays on 3.13 or below. Moving between Python versions is only listed on the dashboard.
- Each repo has one **Dependency Dashboard** issue that lists everything pending. That issue is the repo's **one card** on the project board(s).
- When everything pending has been dealt with, the dashboard closes itself and moves to **Done on every board** it's on.

---

## How it's set up

```
digital-land/renovate-config  (central repo)
├── default.json                          shared Renovate config ("preset") used by every repo
├── repos.json                            the repos Renovate runs on
└── .github/workflows/
    └── renovate.yml                      runs Renovate across those repos, then adds
                                          each repo's Dependency Dashboard to the board

digital-land/<any repo>
└── renovate.json                         one line: extends the shared preset
```

**Renovate runs centrally.** A single scheduled GitHub Actions workflow in `renovate-config` runs Renovate against every opted-in repo. Your repo doesn't need its own workflow or secrets. Renovate is open source and runs entirely on our GitHub Actions runners, so no code or data goes to a third-party service.

**It authenticates as a GitHub App** (Rennovate-App). PRs, commits and dashboard issues come from `rennovate-app[bot]`. Because it's an App, Renovate's PRs trigger your normal CI checks.

**Repos are listed centrally.** Renovate only processes repos that:
1. are listed in **`repos.json`** in `digital-land/renovate-config`, and
2. have the GitHub App installed.

**Policy is shared.** Each repo's `renovate.json` just extends the central preset:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["local>digital-land/renovate-config"]
}
```

Changes to `default.json` in the central repo apply to every repo on Renovate's next run.

---

## When things happen

| What | When | Notes |
|---|---|---|
| Renovate runs | Weekdays at about 06:00, 10:00 and 14:00 UTC | Picks up dashboard checkbox ticks and new security fixes |
| Weekly minor/patch PRs | Monday morning | One PR per type: Python, JavaScript, CI, Docker |
| Lock file maintenance PRs | Monday morning | Refreshes lockfiles to pick up transitive updates. Python (pip-compile) gets its own "Refresh pip-compile outputs" PR |
| Major upgrades | Listed on the dashboard as soon as they're found | No PR until someone ticks one; the PR then opens on the next run |
| Dashboard added to board | At the end of each Renovate run | Only needed once per repo; later runs do nothing |
| Security updates | Next Renovate run after detection | Grouped into one PR, plus a second for any fixes that need a major upgrade; not held back by the weekly schedule |

New releases are only proposed once they're at least **3 days old**. This avoids broken or compromised releases that get pulled shortly after publishing. Renovate also keeps **no more than 5 of its PRs open** at once per repo, and opens **no more than 3 new PRs an hour**, so the Monday PRs are spread across the 06:00 and 10:00 UTC runs. Security PRs aren't counted towards either limit.

---

## What you'll see in your repo

### The Dependency Dashboard (one issue per repo)

Renovate keeps a single issue titled **"Dependency Dashboard"** up to date. It lists:

- updates waiting for the weekly schedule
- **major upgrades available**, under "Pending Approval"
- PRs currently open
- anything that's errored or been ignored
- every dependency Renovate has detected

Each pending item has a **checkbox**. Tick it and Renovate will create (or rebase) that PR on its next run. Use this if you want an update before Monday.

The dashboard **closes itself when nothing is pending** and reopens when new updates arrive. On the project board, that means the card moves to Done when the repo is up to date and comes back when there's new work. Don't close it by hand, because Renovate will just reopen it.

Major upgrades waiting in "Pending Approval" count as pending, so **the card stays open while majors are available**, even after the weekly PRs are merged. That's deliberate: an open card with no weekly PRs means "major upgrades to look at". Open the dashboard to see which ones.

### Weekly PRs, one per type

Minor and patch updates are grouped by type, so a problem in one doesn't hold up the others. Your repo only gets the ones it needs:

| PR title | Covers |
|---|---|
| Update weekly Python updates | Python packages (pip-compile, pip) |
| Update weekly JavaScript updates | npm packages |
| Update weekly CI updates | Actions used in `.github/workflows` |
| Update weekly Docker updates | `Dockerfile` and compose image tags |
| Update weekly non-major updates | Anything that doesn't fit the types above |

- They **don't merge automatically**. Once CI passes, someone on the team should review and merge each one. They're listed on the Dependency Dashboard until merged.
- If CI **fails** on one, see [When a weekly PR fails](#when-a-weekly-pr-fails). The others can still be merged.
- If one isn't merged before the next Monday, Renovate updates the same PR with that week's new updates rather than opening a second one.

### Lock file maintenance PR

Regenerates lockfiles (`package-lock.json`, pip-compile `requirements.txt` files and so on) so indirect dependencies get updated too. Treat it like the weekly PRs.

### Major upgrades (listed, not created)

Major version bumps **don't open PRs on their own**. They're listed on the Dependency Dashboard under **"Pending Approval"**, each with a checkbox. When the team decides to take one on, tick its box and Renovate opens the PR, labelled `dependencies` and `major`, on its next run. See [Working on a major upgrade](#working-on-a-major-upgrade).

### Security updates

These come from GitHub's Dependabot alerts (alerts only; Dependabot's own update PRs are switched off). Renovate groups the repo's security fixes, labels them `security`, and raises them straight away rather than waiting for Monday. Prioritise them. There are up to two PRs:

| PR title | Contains |
|---|---|
| Update security updates [SECURITY] | Fixes within the package's current major version. Usually safe to merge once CI passes. |
| Update security updates [SECURITY] (major) | Fixes that are only available in a new major version. May need code changes. |

Keeping them apart means the straightforward fixes don't wait on the ones that need work.

- **Each fix moves the package to the lowest version that fixes the vulnerability**, not the latest. Once it's merged, the package goes back into the normal weekly PR, which moves it on to the latest minor or patch.
- **A package is only ever in one PR.** While it's vulnerable it's in the security PR and left out of the weekly PRs. If both PRs touch the same file, Renovate rebases whichever is merged second.
- **The "(major)" security PR doesn't wait for a dashboard tick**, unlike other majors, because a known vulnerability shouldn't wait. Check the release notes before merging.

### Python version (held at 3.13)

Renovate won't propose Python 3.14 or later, in Docker images or in `actions/setup-python`. Moving between Python versions (for example 3.10 to 3.13) is only **listed on the dashboard**, because it also needs `runtime.txt`, CI and pip-compile output updating together. Tick it when you're ready, then make those other changes on the same PR branch. Patch updates within your current version still come through in the weekly PRs.

---

## Project boards

- Each repo has **one card**: its Dependency Dashboard issue. It's added to the **[central dependency maintenance board](https://github.com/orgs/digital-land/projects/44)** and can also be added to **team boards**.
- An issue can be on **several boards at once**. Each board has its **own Status**, so moving a card on one board doesn't move it on the others.
- **When the dashboard closes, it moves to Done on every board**, through each project's built-in "Item closed → Done" workflow. When it reopens, the "Item reopened" workflow moves it back to Backlog.
- **The team's job each week** is to merge the weekly PRs (and lock file maintenance PR). Don't close the dashboard or drag the card to Done by hand. The card clears itself on Renovate's next run once nothing is pending.
- In table view, add the **Repository** field and group by it. Every dashboard issue is titled "Dependency Dashboard", so the repository is what tells them apart.

---

## Working on a major upgrade

1. **Decide to take it on.** Open the repo's Dependency Dashboard and find the upgrade under "Pending Approval". If it needs planning, create your own issue or task for it on your team board.
2. **Tick its checkbox.** Renovate opens the PR on its next run (within a few hours on a weekday), or straight away if you run the Renovate workflow manually.
3. **Read the release notes** and changelog that Renovate includes in the PR.
4. **Check out the PR branch** (named something like `renovate/<package>-<version>.x`) and make any code changes the upgrade needs. Push them to the same branch.
5. When CI passes, get it reviewed and merge it. The dashboard updates on Renovate's next run.

**Important: once you've pushed to a Renovate branch, Renovate stops updating it.** That's deliberate, so it won't overwrite your work. Don't tick the "rebase/retry" checkbox in the PR description after you've pushed changes: it **resets the branch and discards your commits**. If you need the latest `main`, merge or rebase it yourself.

**If you've decided not to do a major for now**, you can leave it on the dashboard, but the card will stay open. To clear it, hold the package at its current major in your repo's `renovate.json` (see [Hold a package at its current major](#hold-a-package-at-its-current-major)) and remove the rule when you're ready.

---

## When a weekly PR fails

If one of the weekly PRs fails CI:

1. Check the failing job to see which update broke it. The PR description lists every package in the group.
2. Then either:
   - **fix it** by pushing a change to the PR branch, then review and merge it as normal once CI passes, **or**
   - **hold back the problem package** for now with a rule in your repo's `renovate.json` (see below), and raise an issue to deal with it properly.

Until it's fixed, the rest of that PR's updates are held up too, so please don't leave a failing weekly PR open for long. The other weekly PRs aren't affected.

---

## Common tasks

### Get an update now instead of waiting for Monday
Tick its checkbox on the Dependency Dashboard. The PR appears on the next run.

### Rebase or retry a Renovate PR
Tick the "rebase/retry" checkbox in the PR description, **as long as you haven't pushed your own commits** to it.

### Skip a particular version
Close the PR without merging. Renovate won't propose that version again, and will propose the next release when there is one. It's listed under "ignored" on the dashboard.

### Ignore a package for now
In your repo's `renovate.json`:

```json
{
  "extends": ["local>digital-land/renovate-config"],
  "packageRules": [
    {
      "matchPackageNames": ["some-package"],
      "enabled": false
    }
  ]
}
```

Add a comment or link to an issue explaining why, and remove the rule when you can.

### Hold a package at its current major
To keep getting minor and patch updates for a package but stop it listing a new major (for example, staying on Django 4.2 for now):

```json
{
  "extends": ["local>digital-land/renovate-config"],
  "packageRules": [
    {
      "description": "Staying on Django 4.2 until <reason / issue link>",
      "matchPackageNames": ["django"],
      "matchUpdateTypes": ["major"],
      "enabled": false
    }
  ]
}
```

### Change behaviour for your repo only
Anything in your repo's `renovate.json` overrides the shared preset. For example, to get the weekly PRs on a Wednesday instead:

```json
{
  "extends": ["local>digital-land/renovate-config"],
  "schedule": ["before 12pm on wednesday"],
  "lockFileMaintenance": { "enabled": true, "schedule": ["before 12pm on wednesday"] }
}
```

Before adding something that would help every repo, suggest it as a change to the shared preset instead.

### Turn Renovate off for a repo
Add `"enabled": false` to the repo's `renovate.json`, or open a PR in `digital-land/renovate-config` removing it from `repos.json`.

### Add a new repo
1. Ask whoever manages the GitHub App to install it on the repo.
2. Open a PR in `digital-land/renovate-config` adding the repo to `repos.json`.
3. Once that's merged, on its next run Renovate opens an **onboarding PR** that adds `renovate.json` and lists the dependencies it found and the PRs it plans to open. Review it and merge it.

---

## Python repos using pip-tools

Renovate uses its `pip-compile` manager for these repos. It reads the command from the header at the top of each compiled `requirements*.txt` file and reruns `pip-compile` with the same arguments. For that to work:

- **Keep the default header.** Don't compile with `--no-header`.
- **Compile with `--strip-extras`.** Without it, recent pip-tools versions write `--no-strip-extras` into the header, and Renovate can't read a header with that option, so it skips the file.
- **Always pass `--output-file=<path>`, with an `=` sign.** Renovate works out which folder to run `pip-compile` from by comparing this path with where the file actually is. Without it, Renovate assumes the command was run from the compiled file's own folder, and fails to find your `.in` files if they're in a subfolder.
- **Pass every source file explicitly** in your `pip-compile` command.

For example, with compiled files in a `requirements/` folder, run from the repo root (these are the commands in `local-plans-explorer`'s `Makefile`):

```make
python -m piptools compile --strip-extras --output-file=requirements/dev-requirements.txt requirements/dev-requirements.in
python -m piptools compile --strip-extras --output-file=requirements/requirements.txt requirements/requirements.in
```

Some other notes:

- **Use these same commands to upgrade by hand.** Renovate reruns exactly the command in the header. Add `--upgrade` to refresh everything within your limits, or `--upgrade-package=<name>==<version>` for one package.
- **Limit direct dependencies in the `.in` files with `~=`**, for example `Flask~=3.1` (any 3.x from 3.1) or `pytest-playwright~=0.5.2` (0.5.x only). The Monday "Refresh pip-compile outputs" PR and a manual `--upgrade` both stay within these limits. When a new release falls outside a limit, Renovate proposes changing the limit: on the dashboard for a major, or in the weekly Python PR for a minor.
- Renovate updates the **`.in`** (or `pyproject.toml`) source files and regenerates the `.txt` output. Don't edit the compiled `.txt` files by hand.

If your repo uses pip-tools, its `renovate.json` needs these settings:

```json
{
  "extends": ["local>digital-land/renovate-config"],
  "pip-compile": {
    "managerFilePatterns": ["/(^|/)[\\w-]*requirements\\.txt$/"]
  },
  "pip_requirements": { "enabled": false },
  "pip_setup": { "enabled": false }
}
```

The pattern must match **only the compiled files** (for example `requirements.txt` and `dev-requirements.txt`). If your repo also has plain requirements files that pip-compile didn't generate, narrow it. For example, `local-plans-explorer` keeps its compiled files in a `requirements/` folder and uses `"/^requirements/[\\w-]*requirements\\.txt$/"`. Check the onboarding PR's "Detected Package Files" list shows each compiled file under `pip-compile`.

---

## Troubleshooting

**No PRs are appearing in my repo**
- Check the repo is listed in `repos.json` in `digital-land/renovate-config`, and the App is installed on it.
- Check `renovate.json` is valid. If it isn't, Renovate opens an issue explaining the error.
- Remember the 3-day release-age delay and the Monday schedule. The dashboard shows what's waiting.

**CI didn't run on a Renovate PR**
- PRs from the App should trigger workflows. If they don't, check your workflow's `on:` triggers include `pull_request`, and raise it with whoever manages the Renovate setup.

**A PR has merge conflicts**
- Renovate rebases its own PRs automatically. If you've pushed to the branch, it won't, so resolve the conflicts yourself.

**The dashboard shows an error for a dependency**
- Usually a private registry Renovate can't reach, or a lockfile that won't regenerate. Check the Renovate workflow logs in `digital-land/renovate-config` (search for your repo's name), or ask whoever manages the Renovate setup.

---

## Shared preset (current)

The settings every repo starts from are in [`default.json`](https://github.com/digital-land/renovate-config/blob/main/default.json) in `digital-land/renovate-config`. Each setting has a description explaining it. In summary:

- Renovate's recommended defaults (`config:recommended`), on London time
- weekly PRs before 12pm on Monday, one per type, plus lock file maintenance
- new releases only once they're at least 3 days old
- at most 5 Renovate PRs open per repo at once, and at most 3 new ones an hour (security PRs aren't counted)
- major upgrades, and moving between Python versions, listed on the dashboard only
- Python held at 3.13 or below
- security fixes grouped into one PR (plus one for fixes needing a major), raised straight away
- a Dependency Dashboard that closes itself when nothing is pending

Automerge is deliberately switched off for now: every Renovate PR needs a developer to review and merge it.

Further reading: [Renovate documentation](https://docs.renovatebot.com/)