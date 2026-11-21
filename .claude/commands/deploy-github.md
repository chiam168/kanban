---
description: Security-scan, push to GitHub, set up Pages + CI/CD via Actions, and fill in the repo About section
argument-hint: <owner/repo or GitHub URL> [--private] [--branch main]
allowed-tools: Bash(git:*), Bash(gh:*), Bash(grep:*), Bash(curl:*), Bash(ls:*), Bash(cat:*), Bash(mkdir:*), Read, Write, Edit
---

# Deploy to GitHub + GitHub Pages

Target repo info from the user: `$ARGUMENTS`

Work through the phases below in order. Stop and ask the user whenever a phase says so — do not guess. Keep a short running log of what you did; print it as the final report.

## Phase 0 — Inputs and prerequisites

1. Parse `$ARGUMENTS` into `OWNER/REPO`. Accept `owner/repo`, `https://github.com/owner/repo(.git)` or `git@github.com:owner/repo.git`. If it's empty or ambiguous, ask the user for it and stop until they answer.
2. Default branch is `main` unless `--branch` was given. Visibility is **public** unless `--private` was given (note: GitHub Pages on a free plan requires a public repo — if `--private`, warn the user Pages may need a paid plan and ask whether to continue).
3. Check tooling: `git --version`, `gh --version`, `gh auth status`. If `gh` is missing or not logged in, tell the user to run `gh auth login` themselves and stop. Never ask for, type or print a token.
4. Check whether the repo already exists: `gh repo view OWNER/REPO --json name,visibility,defaultBranchRef,homepageUrl,description 2>/dev/null`. Remember whether this is a **create** or **edit** run.

## Phase 1 — Security scan (gate: runs BEFORE anything leaves the machine)

Nothing is pushed until this phase passes. Scan every file that would be committed (respect `.gitignore`; if not a git repo yet, scan the working tree excluding `.git/`).

1. **Project hard-constraint check** (from CLAUDE.md):
   `grep -nEi "localStorage|sessionStorage|indexedDB|document\.cookie|https?://|!important" index.html`
   The only allowed hit is the `FORMSUBMIT_ENDPOINT` line. Anything else → report and stop.
2. **Outbound network calls**: grep all files for `fetch(`, `XMLHttpRequest`, `navigator.sendBeacon`, `WebSocket`, `EventSource`, `<script src=`, `<link href=`, `<img src=http`, `@import`, `url(http`. The only permitted outbound call is the FormSubmit AJAX `fetch` to `FORMSUBMIT_ENDPOINT`. Report anything else and stop.
3. **Secrets**: grep for common secret patterns and fail on any match:
   - `AKIA[0-9A-Z]{16}`, `ghp_[A-Za-z0-9]{36}`, `github_pat_`, `gho_`, `xox[baprs]-`, `sk-[A-Za-z0-9]{20,}`, `sk-ant-`, `AIza[0-9A-Za-z_-]{35}`
   - `-----BEGIN (RSA |EC |OPENSSH |)PRIVATE KEY-----`
   - `(api[_-]?key|secret|password|passwd|token)\s*[:=]\s*["'][^"']{8,}`
   If `gitleaks` or `trufflehog` is installed locally, run it too (`gitleaks detect --no-git -s .` / `trufflehog filesystem .`); don't install them.
4. **Sensitive files**: make sure none of these would be committed: `.env*`, `*.pem`, `*.key`, `id_rsa*`, `credentials*.json`, `.claude/settings.local.json`, `.DS_Store`, `*.log`. Create/extend `.gitignore` to cover them.
5. **Personal data / email exposure**: read the value of `FORMSUBMIT_ENDPOINT`.
   - If it's the placeholder `YOUR_EMAIL@example.com`: note that notifications won't work after deploy, continue.
   - If it contains a **real email address**: warn the user that the address will be publicly visible in the repo and on the live Pages site (scrapable by spammers). Suggest switching to FormSubmit's random-string alias (shown in the FormSubmit activation email / dashboard: `https://formsubmit.co/ajax/<random-string>`). Ask whether to (a) swap in the alias, (b) proceed with the email visible, or (c) stop. Do not continue without an answer.
   - Also grep for any other email addresses, phone numbers or names outside seed/demo data and report them.
6. **Branding constraint**: confirm no real bank logos or official-system imitation were added (per CLAUDE.md). Report anything suspicious.
7. Print a scan summary table (check → pass/fail → details). Any fail → stop and ask the user how to proceed.

## Phase 2 — Push the code

1. If not a git repo: `git init -b <branch>`. If it is, make sure you're on `<branch>` and the tree has no surprises (`git status`).
2. Ensure `.gitignore` exists (Phase 1 step 4).
3. Add a short `README.md` if missing: one-paragraph description, "open `index.html` in a browser", the live-site link placeholder (filled in Phase 5), and the "no persistence — refresh resets the board" note. Don't overwrite an existing README; edit only the link line if present.
4. Stage and review: `git add -A && git status --short`. Show the user the file list. Re-run the Phase 1 secret grep against `git diff --cached` as a final check.
5. Commit with a clear message (e.g. `Initial commit: IT PMO Kanban board` or `Update Kanban board and deploy config`).
6. Remote:
   - **create**: `gh repo create OWNER/REPO --public|--private --source=. --remote=origin --push` (confirm with the user before creating the repo).
   - **edit**: add/verify `origin` points to `OWNER/REPO`, then `git push -u origin <branch>`.
   - Never `--force` push. If the push is rejected (remote has commits), stop and ask the user whether to pull/rebase.

## Phase 3 — CI/CD via GitHub Actions

Create or update `.github/workflows/pages.yml`. It must:
- Trigger on `push` to `<branch>`, on `pull_request` (checks only, no deploy), and `workflow_dispatch`.
- Use least-privilege `permissions` (`contents: read` at top; `pages: write` + `id-token: write` only on the deploy job) and a `concurrency` group `pages` with `cancel-in-progress: false`.
- **check job**: re-run the Phase 1 constraint grep and secret-pattern grep in shell (no third-party actions) and fail the build on a hit; verify `index.html` exists and contains `<title>`.
- **deploy job** (needs: check; only on push/dispatch to `<branch>`): copy **only the public site files** (`index.html`, plus any assets it references) into `_site/` — never upload `CLAUDE.md`, `.claude/`, `.github/` or other repo files — then use the official actions pinned to major versions: `actions/checkout@v7`, `actions/configure-pages@v6`, `actions/upload-pages-artifact@v5` (path `_site`), `actions/deploy-pages@v5`, with `environment: github-pages` and `url: ${{ steps.deployment.outputs.page_url }}`.
- **Before writing the workflow, check those are still the latest majors**: `gh api repos/<action>/releases/latest --jq .tag_name` for each of the four, and use the newest major. `upload-pages-artifact` and `deploy-pages` must be from the same generation. A mismatch makes the deploy fail with `No artifacts named "github-pages" were found`.

If a workflow already exists, edit it rather than duplicating; keep any user additions.

Commit (`ci: add GitHub Pages deploy workflow`) and push.

## Phase 4 — Create/edit GitHub Pages

1. Check: `gh api repos/OWNER/REPO/pages 2>/dev/null`.
2. Not enabled → `gh api -X POST repos/OWNER/REPO/pages -f build_type=workflow`.
   Enabled but `build_type` is `legacy` → `gh api -X PUT repos/OWNER/REPO/pages -f build_type=workflow`.
3. Optionally enforce HTTPS: `gh api -X PUT repos/OWNER/REPO/pages -F https_enforced=true` (ignore failure if the cert isn't ready yet).
4. Read back the site URL: `gh api repos/OWNER/REPO/pages --jq .html_url` (normally `https://OWNER.github.io/REPO/`).

## Phase 5 — Deploy and verify

1. Trigger/locate the run: `gh run list --workflow pages.yml -L 1`; if nothing ran after enabling Pages, `gh workflow run pages.yml --ref <branch>`.
2. Watch it in the background: `gh run watch <id> --exit-status` (run_in_background; you'll be notified). On failure, `gh run view <id> --log-failed`, fix the cause, push, and retry once; if it still fails, report the log excerpt and stop.
3. Verify the live site: `curl -sSfL -o /dev/null -w "%{http_code}" <pages_url>` should be `200`, and `curl -sL <pages_url> | grep -o "<title>[^<]*"` should match the app title. Pages can take a minute after deploy — retry a few times (use a Monitor/until-loop, not foreground sleep).
4. **Post-deploy exposure check**: fetch the deployed site and confirm `CLAUDE.md`, `.claude/`, `.github/` and `.gitignore` return 404 under the Pages URL, and that the served `index.html` passes the Phase 1 secret grep.
5. Update the README live-site link to the Pages URL if it changed; commit + push.

## Phase 6 — Create/edit the repo About section (with the Pages link)

1. Description: one line, ≤ 350 chars, e.g. `Single-file Kanban board for an IT PMO demo — vanilla HTML/CSS/JS, no build, no persistence.` If a description already exists, show it and only replace it if the user agrees or it's empty.
2. Run:
   `gh repo edit OWNER/REPO --description "<desc>" --homepage "<pages_url>" --add-topic kanban --add-topic vanilla-js --add-topic github-pages --add-topic project-management`
   (Merge with existing topics; don't remove any.)
3. Verify: `gh repo view OWNER/REPO --json description,homepageUrl,repositoryTopics`. `homepageUrl` must equal the Pages URL — this is the link shown in the About sidebar.

## Final report

Print:
- Security scan summary (Phase 1 table + Phase 5 exposure check).
- Repo URL, branch, commit SHA(s) pushed.
- Workflow file path and the latest run URL + status.
- Live Pages URL (and HTTP status from verification).
- About section: description, homepage, topics.
- Anything skipped, failed, or waiting on the user (e.g. FormSubmit activation email still needs clicking before notifications are delivered).
