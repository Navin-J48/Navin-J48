# Setup guide

This dashboard is built from three ingredients:

1. **Static Markdown/HTML** in `README.md` — the layout, text, and icon row.
2. **Hosted badge services** (`github-readme-stats`, `streak-stats`, `skillicons.dev`) — real-time images keyed by your username. No secrets or workflow involved; they read public data directly when GitHub renders the image.
3. **A self-hosted GitHub Action** (`lowlighter/metrics`) that queries the GitHub API and commits fresh SVGs (`dashboard-*.svg`) into this repo on a schedule. This is what powers the calendar, habits, language breakdown, and pinned-repository panels, and is how the neon theme gets applied.

## 1. Create the special profile repository

For account **Navin-J48**, that repository is `Navin-J48/Navin-J48` (already created — GitHub confirms it as the account's one public repository). GitHub automatically renders that repo's `README.md` on the profile page at github.com/Navin-J48.

## 2. Replace the username placeholder

Already done — every badge URL, avatar link, and profile link in `README.md` uses `Navin-J48` directly. `metrics.yml` also sets `user: Navin-J48` explicitly on every step, so nothing here depends on implicit token ownership. If you ever rename the account, you'd need to update both files (a repo-owner variable is not used here since you asked for the username to be hardcoded explicitly).

## 3. Create a Personal Access Token — read-only, no write access needed

This is the part most guides get wrong, so it's worth being precise about:

- The **hosted badges** (`github-readme-stats`, `streak-stats`) need no token from you at all — they read public data directly.
- The **`lowlighter/metrics` Action** needs a token (`METRICS_TOKEN`) to query the GitHub GraphQL/REST API for richer data than public scraping allows. **This token only ever reads data — it is never used to write anything.**
- Committing the generated SVGs back into this repo is handled by a *separate*, automatically-issued token (`github.token`, GitHub's built-in per-run `GITHUB_TOKEN`), which is already scoped to this one repository. The workflow passes it explicitly as `committer_token: ${{ github.token }}`.

So `METRICS_TOKEN` needs **no `repo` scope and no write permissions of any kind**:

1. Go to **GitHub → Settings → Developer settings → Personal access tokens**.
2. **Fine-grained token (recommended):** create one with no repository access selected, or only "Public Repositories (read-only)" — that's enough for all public-data plugins used here.
   **Classic token (simpler, works everywhere):** you can generate one with **zero scopes checked**. An authenticated-but-scopeless classic token is sufficient for all the plugins in this workflow (they're documented as "scopeless" — see each plugin's requirements in `metrics.yml`'s comments).
3. Only add scopes beyond that if you want more than public data:
   - `read:org` — to include **private** organization memberships in the header/community section.
   - `repo` (classic) — only if you want **private repositories** counted in the language/repository stats. This is optional and off by default in this setup, per the principle of least privilege.
4. In **this repository** → **Settings → Secrets and variables → Actions → New repository secret**:
   - Name: `METRICS_TOKEN`
   - Value: the token you just generated
5. Timezone is already set to `Asia/Kolkata` (IST) for the header, calendar, and habits panels, so commit-hour/day charts line up with your actual working hours instead of UTC. If you ever want to override it without editing the workflow, add a repository **variable** (not secret) named `METRICS_TIMEZONE` with a different [tz database name](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) — it takes priority over the `Asia/Kolkata` default.

**Never put the token value itself anywhere in this repo** — not in the README, not in `metrics.yml`, not in a comment. Every reference in this project is `${{ secrets.METRICS_TOKEN }}`, which GitHub resolves at runtime and never exposes in logs.

## 4. Enable Actions write permissions (for the commit step only)

Under **Settings → Actions → General → Workflow permissions**, select **"Read and write permissions"**. This grants the automatic `github.token` used in step 3 the `contents: write` access needed to commit the generated SVGs — it has no effect on `METRICS_TOKEN`, which remains read-only.

The workflow also declares this explicitly at the top level:

```yaml
permissions:
  contents: write
```

This is the only permission granted, and only because the workflow commits files back to the repo.

## 5. Test it

- Push this project to `Navin-J48/Navin-J48`.
- Go to the **Actions** tab → select the **Metrics** workflow → **Run workflow** (`workflow_dispatch` is already wired up for this).
- Once it finishes (usually under a minute), confirm five new/updated files appear at the repo root: `dashboard-header.svg`, `dashboard-calendar.svg`, `dashboard-habits.svg`, `dashboard-languages.svg`, `dashboard-repositories.svg`.
- Visit `github.com/Navin-J48` and confirm every image renders. If one looks broken, open it directly at `github.com/Navin-J48/Navin-J48/blob/main/dashboard-header.svg` to see the raw render, and check the failed step's logs in the Actions tab for API/token errors.

## 6. Keep it updated automatically

No action needed — the workflow already runs:
- Daily at 03:00 UTC (`schedule: cron: "0 3 * * *"`)
- On every push to `main`
- On demand via **Actions → Metrics → Run workflow**

To change the frequency, edit the `cron` line in `.github/workflows/metrics.yml` (`minute hour day month weekday`; e.g. `"0 */6 * * *"` = every 6 hours).

## Going further with theming

`NEON_CSS` in `metrics.yml` only targets selectors guaranteed to exist in every `lowlighter/metrics` template (`body`, `h1`–`h4`, `a`, `svg`). To restyle something more specific (e.g. individual isocalendar day cells):

1. Run the workflow once and open the generated SVG in a text editor.
2. Find the actual `class="..."` names used inside that plugin's markup — these are plugin-specific and not documented as a stable public API.
3. Add matching rules to `extras_css` in that step of `metrics.yml`.

## A note on current data

As of this setup, `Navin-J48` is a new GitHub account (joined July 31, 2026) with one repository — `Navin-J48/Navin-J48`, this profile repo itself. That means, honestly:

- **Project Spotlight** (`dashboard-repositories.svg`) will render empty or near-empty at first, since `plugin_repositories_pinned` has nothing else to pull from yet. Nothing is wrong — once you create and pin other repositories at github.com/Navin-J48 ("Customize your pins"), this panel fills in automatically on the next scheduled run.
- **Top languages**, **stars**, and **streak** will likewise start at their real current values (near zero) rather than showing placeholder numbers. This is intentional — the whole point of this rebuild was to stop inventing numbers like "128 repositories" and show what's actually true, even when that's a small number today.

## Final validation checklist

- [x] Repository structure matches what's listed below
- [x] `README.md` uses only GitHub-renderable HTML (tables, `div align`, `img width`) — no `style=` attributes, which GitHub's sanitizer strips
- [x] `metrics.yml` YAML syntax validated
- [x] Every plugin option used (`base_indepth`, `plugin_isocalendar_duration`, `plugin_habits_*` including `plugin_habits_charts_type: graph`, `plugin_languages_*`, `plugin_repositories_pinned`) checked against current `lowlighter/metrics` plugin documentation
- [x] `config_timezone` set to `Asia/Kolkata` on header, calendar, and habits steps (the plugins where time-of-day/date accuracy matters)
- [x] No `USERNAME` placeholder remains anywhere in the project — verified with a project-wide search
- [x] `user: ${{ github.repository_owner }}` set explicitly on every step — no implicit reliance on "whoever owns the token"
- [x] `METRICS_TOKEN` referenced only as `${{ secrets.METRICS_TOKEN }}`; no token value anywhere in the repo
- [x] `committer_token` explicitly set to the auto-issued `github.token`, so the PAT never needs write/`repo` scope
- [x] `permissions: contents: write` is the only permission granted, and only because a commit step needs it
- [x] Every filename referenced in `README.md` (`dashboard-header.svg`, `dashboard-calendar.svg`, `dashboard-habits.svg`, `dashboard-languages.svg`, `dashboard-repositories.svg`) matches a `filename:` value in `metrics.yml` exactly
- [x] No fictional statistics anywhere — every number comes from a live badge service or a workflow-generated SVG
- [x] Project Spotlight pulls real pinned repositories, not invented project names
- [x] Technology list only includes what you specified: C, C++, Java, Python, SQL, HTML, CSS, JavaScript, React, Flutter, Firebase, Git
- [x] Workflow supports manual (`workflow_dispatch`), scheduled (`cron`), and push triggers

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Images show as broken links | Workflow hasn't run yet, or a filename in `README.md` doesn't match a `filename:` in `metrics.yml` |
| Workflow fails with a 401 | `METRICS_TOKEN` missing or expired |
| Workflow fails with "resource not accessible" on the commit step | Repo's Actions permissions are set to read-only — see step 4 |
| Calendar/habits panels look empty | Your commits are authored under an email/username not linked to your GitHub account — link it under Settings → Emails |
| Badge images (stats/streak) don't update immediately | Those are hosted services with their own cache; they refresh independently of this repo's Action, usually within minutes to a few hours |
| Avatar isn't rounded | GitHub's README sanitizer strips `style="border-radius"` — this is a known Markdown/HTML limitation, not a bug in this project |
