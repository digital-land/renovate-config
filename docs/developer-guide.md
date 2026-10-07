# Dependency updates with Renovate: developer guide

This guide explains how automated dependency updates work across our repositories, what you'll see in your repo, and what you need to do when an update needs attention.

---

## In short

- **Renovate** keeps our dependencies up to date by opening pull requests.
- Routine **minor and patch** updates arrive **once a week** in a single grouped PR for a developer to review and merge. Nothing merges automatically.
- **Major** upgrades are **only listed** on the dashboard. No PR is created until someone on the team ticks one.
- **Security** fixes arrive as soon as they're found, whatever the schedule.
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
| Weekly minor/patch PR | Monday morning | Grouped into one PR per repo |
| Lock file maintenance PR | Monday morning | Refreshes lockfiles to pick up transitive updates |
| Major upgrades | Listed on the dashboard as soon as they're found | No PR until someone ticks one; the PR then opens on the next run |
| Dashboard added to board | At the end of each Renovate run | Only needed once per repo; later runs do nothing |
| Security updates | Next Renovate run after detection | Not held back by the weekly schedule |

New releases are only proposed once they're at least **3 days old**. This avoids broken or compromised releases that get pulled shortly after publishing. Renovate also keeps **no more than 5 of its PRs open** at once per repo.

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

Major upgrades waiting in "Pending Approval" count as pending, so **the card stays open while majors are available**, even after the weekly PR is merged. That's deliberate: an open card with no weekly PR means "major upgrades to look at". Open the dashboard to see which ones.

### Weekly non-major PR: "Update weekly non-major updates"

All minor and patch updates for the repo, grouped into one PR.

- It **does not merge automatically**. Once CI passes, someone on the team should review it and merge it. It's listed on the Dependency Dashboard until it's merged.
- If CI **fails**, see [When the weekly PR fails](#when-the-weekly-pr-fails).
- If it isn't merged before the next Monday, Renovate updates the same PR with that week's new updates rather than opening a second one.

### Lock file maintenance PR

Regenerates lockfiles (`package-lock.json`, pip-compile `requirements.txt` files and so on) so indirect dependencies get updated too. Treat it like the weekly PR.

### Major upgrades (listed, not created)

Major version bumps **don't open PRs on their own**. They're listed on the Dependency Dashboard under **"Pending Approval"**, each with a checkbox. When the team decides to take one on, tick its box and Renovate opens the PR, labelled `dependencies` and `major`, on its next run. See [Working on a major upgrade](#working-on-a-major-upgrade).

### Security updates

These come from GitHub's Dependabot alerts (alerts only; Dependabot's own update PRs are switched off). Renovate raises them straight away and labels them as security updates. Prioritise them.

---

## Project boards

- Each repo has **one card**: its Dependency Dashboard issue. It's added to the **[central dependency maintenance board](https://github.com/orgs/digital-land/projects/44)** and can also be added to **team boards**.
- An issue can be on **several boards at once**. Each board has its **own Status**, so moving a card on one board doesn't move it on the others.
- **When the dashboard closes, it moves to Done on every board**, through each project's built-in "Item closed → Done" workflow. When it reopens, the "Item reopened" workflow moves it back to Backlog.
- **The team's job each week** is to merge the weekly PR (and lock file maintenance PR). Don't close the dashboard or drag the card to Done by hand. The card clears itself on Renovate's next run once nothing is pending.
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

## When the weekly PR fails

If the grouped minor/patch PR fails CI:

1. Check the failing job to see which update broke it. The PR description lists every package in the group.
2. Then either:
   - **fix it** by pushing a change to the PR branch, then review and merge it as normal once CI passes, **or**
   - **hold back the problem package** for now with a rule in your repo's `renovate.json` (see below), and raise an issue to deal with it properly.

Until it's fixed, the rest of that week's updates for the repo are held up too, so please don't leave a failing weekly PR open for long.

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
Anything in your repo's `renovate.json` overrides the shared preset. For example, to get the weekly PR on a Wednesday instead:

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
- **Pass every source file explicitly** in your `pip-compile` command (for example `pip-compile --output-file=requirements.txt requirements.in`).
- Renovate updates the **`.in`** (or `pyproject.toml`) source files and regenerates the `.txt` output. Don't edit the compiled `.txt` files by hand.

If your repo uses pip-tools, its `renovate.json` needs these settings:

```json
{
  "extends": ["local>digital-land/renovate-config"],
  "pip-compile": {
    "managerFilePatterns": ["/(^|/)requirements(-[\\w]+)?\\.txt$/"]
  },
  "pip_requirements": { "enabled": false },
  "pip_setup": { "enabled": false }
}
```

Change the file pattern if your compiled files are named differently.

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

For reference, this is what `default.json` in `digital-land/renovate-config` sets for every repo:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "timezone": "Europe/London",
  "schedule": ["before 12pm on monday"],
  "minimumReleaseAge": "3 days",
  "prConcurrentLimit": 5,
  "labels": ["dependencies"],
  "dependencyDashboardLabels": ["dependencies"],
  "dependencyDashboardAutoclose": true,
  "lockFileMaintenance": { "enabled": true, "schedule": ["before 12pm on monday"] },
  "packageRules": [
    {
      "matchUpdateTypes": ["minor", "patch"],
      "groupName": "weekly non-major updates"
    },
    {
      "matchUpdateTypes": ["major"],
      "dependencyDashboardApproval": true,
      "labels": ["dependencies", "major"]
    }
  ]
}
```

Automerge is deliberately switched off for now: every Renovate PR needs a developer to review and merge it.

Further reading: [Renovate documentation](https://docs.renovatebot.com/)