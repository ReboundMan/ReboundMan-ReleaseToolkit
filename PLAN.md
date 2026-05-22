# ReboundMan Release Automation — Plan

> **Goal:** A single, reusable Windows-app release pipeline that any ReboundMan
> app (WordMD, Token Tray, future apps) can opt into in <30 lines of YAML, with
> three operating modes (manual, scheduled-with-approval, fully automated on
> change). Plus an optional Copilot CLI skill that gives a unified, terminal-
> driven release experience across all apps.

---

## 1. Guiding principles

1. **GitHub Actions is the engine.** Cron, dispatch, secrets, OIDC, artifacts,
   draft-vs-published releases — it already does everything we need. No new
   infra to host or pay for.
2. **One toolkit, many consumers.** Build/sign/release logic lives once in a
   `ReboundMan/release-toolkit` repo as a *reusable workflow* + composite
   actions. Each app repo holds a ~30-line `.github/workflows/release.yml`
   that just calls into it.
3. **Drafts are the human-approval gate.** Modes that need human review create
   a **draft** GitHub Release with installer attached + auto-generated notes.
   Modes that don't, publish straight away.
4. **Versioning is opinionated.** Single source of truth = git tags (`v1.4.2`).
   Workflow stamps `WordMD.iss`, `csproj`, and the installer filename from the
   tag at build time. No more manual `MyAppVersion` edits.
5. **Signing is mandatory before any automated publish.** Unsigned weekly
   releases train users to click past SmartScreen. Signing comes first.
6. **The skill is a human convenience layer.** All real work happens in GH
   Actions. The skill (`release ...`) just orchestrates `gh workflow run`,
   shows status across repos, and helps edit release notes.

---

## 2. Architecture

```
┌─────────────────────────────────────────────┐
│ ReboundMan/ReboundMan-ReleaseToolkit (new)  │
│                                             │
│  .github/workflows/                         │
│    win-app-release.yml   ← reusable wf     │
│  .github/actions/                           │
│    setup-windows-build/                     │
│    azure-trusted-signing/                   │
│    generate-release-notes/                  │
│    inno-compile/                            │
│    winget-pr/             (optional)        │
│  docs/                                      │
│    ONBOARDING.md                            │
│    SIGNING.md                ← (gitignored: docs/private/) │
│    RUNBOOK.md                               │
│  templates/                                 │
│    release.yml.template   ← per-app stub   │
└────────────────┬────────────────────────────┘
                 │ uses:
   ┌─────────────┼─────────────┬─────────────┐
   ▼             ▼             ▼             ▼
┌──────┐    ┌────────┐    ┌────────┐    ┌────────┐
│WordMD│    │Token   │    │ App 3  │    │ App N  │
│      │    │Tray    │    │        │    │        │
│.github/workflows/release.yml (30 lines)        │
└──────┘    └────────┘    └────────┘    └────────┘
                 │
                 ▼
        ┌────────────────────────┐
        │ Copilot CLI skill      │
        │   release status       │
        │   release kickoff <app>│
        │   release publish <app>│
        │   release notes  <app> │
        └────────────────────────┘
```

---

## 3. The three modes (mapped to triggers)

| Mode | Trigger | Output | Human action |
|---|---|---|---|
| **A. Manual** | `workflow_dispatch` (via `gh workflow run` or skill) | Draft release | Reviews draft, clicks Publish |
| **B. Scheduled + approval** | `schedule: cron 0 6 * * 6` (Sat 06:00 UTC = Fri 11pm PT) — only proceeds if `git rev-list <last-tag>..HEAD` is non-empty | Draft release | Reviews draft, clicks Publish (or ignores → skip-week) |
| **C. Fully automated** | `push: tags: ['v*']` | Published release | None (push the tag → users see it) |

All three call the **same** reusable workflow with different `release_mode`
input (`draft` vs `publish`). The "change detection" lives in a tiny pre-job
that runs `git rev-list --count $(git describe --tags --abbrev=0)..HEAD` and
short-circuits if zero.

---

## 4. Reusable workflow contract

`ReboundMan/ReboundMan-ReleaseToolkit/.github/workflows/win-app-release.yml`

Inputs (each app overrides as needed):

```yaml
inputs:
  app_name:           { required: true,  type: string }   # "WordMD"
  publish_command:    { required: true,  type: string }   # dotnet publish ...
  publish_output_dir: { required: true,  type: string }   # path to publish/
  inno_script:        { required: true,  type: string }   # installer/WordMD.iss
  primary_exe:        { required: true,  type: string }   # WordMD.exe (to sign)
  installer_glob:     { required: true,  type: string }   # dist/*Setup*.exe
  web_bundle:         { required: false, type: boolean, default: false }
  release_mode:       { required: true,  type: string }   # draft|publish
  version:            { required: false, type: string }   # default: from tag
secrets:
  AZURE_TENANT_ID:    { required: true }
  AZURE_CLIENT_ID:    { required: true }
  AZURE_SUBSCRIPTION_ID: { required: true }
  TRUSTED_SIGNING_ACCOUNT: { required: true }
  TRUSTED_SIGNING_PROFILE: { required: true }
```

Steps (in order):
1. **Checkout** + fetch tags
2. **Determine version** (input or `git describe`) and export
3. **Setup tools** — `.NET 8`, Node 18 (if `web_bundle`), Inno Setup 6
4. **Stamp version** into `*.iss` and `*.csproj` (sed/yq)
5. **Build** — run `publish_command`
6. **Sign exe** — Azure Trusted Signing on `primary_exe`
7. **Compile installer** — `ISCC.exe $inno_script`
8. **Sign installer** — Trusted Signing on `installer_glob`
9. **Generate release notes** — PRs merged + commits since last tag, grouped
   by conventional-commit prefix (`feat:`, `fix:`, `docs:`, etc.)
10. **Create release** — `gh release create vX.Y.Z` with `--draft` or not,
    upload installer + SHA256 checksum
11. **(Optional) WinGet manifest PR** — only on `release_mode=publish`

---

## 5. Per-app onboarding (the ~30-line stub)

For any new app (WordMD example):

```yaml
# .github/workflows/release.yml
name: Release
on:
  workflow_dispatch:
    inputs:
      version: { description: 'Override version (default = tag)', required: false }
  push:
    tags: ['v*']
  schedule:
    - cron: '0 6 * * 6'   # Sat 06:00 UTC = Fri 11pm PDT (10pm PST in winter)

jobs:
  release:
    uses: ReboundMan/ReboundMan-ReleaseToolkit/.github/workflows/win-app-release.yml@v1
    with:
      app_name:           WordMD
      publish_command: |
        dotnet publish src/WordMD -c Release -r win-x64 -p:Platform=x64
      publish_output_dir: src/WordMD/bin/x64/Release/net8.0-windows10.0.26100.0/win-x64/publish
      inno_script:        installer/WordMD.iss
      primary_exe:        WordMD.exe
      installer_glob:     dist/WordMD-Setup-*.exe
      web_bundle:         true
      release_mode:       ${{ github.event_name == 'push' && 'publish' || 'draft' }}
      version:            ${{ inputs.version }}
    secrets: inherit
```

That's it. The reusable workflow handles change-detection short-circuit for
the schedule trigger internally.

---

## 6. Copilot CLI skill — `release`

A separate, optional layer that gives you a unified terminal UX:

```
release status                # show pending drafts across all known apps
release kickoff wordmd        # gh workflow run release.yml -R ReboundMan/ReboundMan-WordMD
release kickoff wordmd 1.4.2  # with explicit version
release notes wordmd          # open the draft's notes in $EDITOR, push edits back
release publish wordmd        # promote latest draft to public
release dry-run wordmd        # show "what would the next release contain?"
                              # = git log since last tag, grouped + categorized
release skip-week wordmd      # mark current week skipped (just an issue comment)
```

The skill is a thin wrapper over `gh` CLI + a config file
(`~/.copilot/skills/release/apps.yaml`) that lists each app's repo, primary
branch, version prefix, and any per-app quirks. New apps onboard by adding a
3-line block to the yaml.

**This is genuinely optional** — everything is doable directly with `gh`. The
skill is for "I have 4 apps and want to see what's pending across all of them
in one command."

---

## 7. Decisions

### Locked
| # | Decision | Choice |
|---|---|---|
| 1 | Toolkit repo name | **`ReboundMan/ReboundMan-ReleaseToolkit`** |
| 2 | Toolkit repo visibility | **Public** (no secrets in repo; simplifies reusable-workflow consumption) |
| 3 | Signing provider | **Azure Trusted Signing** |
| 4 | Schedule day/time | **`0 6 * * 6`** — Saturday 06:00 UTC = **Friday 11pm PDT** (drifts to 10pm PST in winter) |
| 5 | Version source of truth | **Git tags `vX.Y.Z`**. Workflow stamps `.iss` + `.csproj` + installer filename at build time. Mode C fails fast if HEAD isn't tagged. Modes A/B compute next-patch from the last final tag (`git tag --list 'v*' \| grep -E '^v[0-9]+\.[0-9]+\.[0-9]+$'`). Pre-release tags exist but don't bump the baseline. Explicit `inputs.version` override always wins. |
| 6 | Conventional commits | **Encouraged, never enforced.** Workflow parses `feat:` `fix:` `perf:` `docs:` `refactor:` `test:` `build:` `ci:` `chore:` (and `!` for breaking) and auto-groups release notes into Features / Fixes / Performance / Docs / Other. Un-prefixed commits land under Other. No bot rejects PRs. |
| 7 | Build skip-week mechanism | Just don't publish the draft. Cron exits silently if no commits since last tag. |
| 8 | Rollback model | Yank-via-workflow (Flavor A) + forward-fix philosophy (Flavor B). See §13. |
| 9 | "Draft ready" notification | GitHub's default email for v1. Re-evaluate after a few weeks. |

### Still TBD (do not block planning — decide at execution time)
| # | Decision | Decide by |
|---|---|---|
| 10 | Azure subscription (MSDN vs PAYGO) | When standing up Trusted Signing account (Phase 0 step 1) |
| 11 | Publisher identity on the cert | When submitting identity validation (Phase 0 step 2) — "ReboundMan" individual recommended for fastest validation |

### Deferred to later phases
| # | Decision | Phase |
|---|---|---|
| 12 | Copilot CLI `release` skill | Phase 4 |
| 13 | WinGet manifest auto-PR | Phase 5 |
| 14 | arm64 build matrix | Phase 5 |
| 15 | SBOM / SHA256 attachment | Phase 5 |
| 16 | Teams/Slack webhook notifications | Phase 5 |

---

## 8. Phased rollout

### Phase 0 — Prereqs (you decide signing first) ~2–7 days
- Stand up Azure Trusted Signing account
- Complete identity validation (1–3 days for individual publisher)
- Create certificate profile, capture identifiers
- Create Azure SP + federated OIDC trust for `ReboundMan/*` repos
- Save SP coordinates as org-level secrets in `ReboundMan` GitHub org

### Phase 1 — Toolkit repo (~1 day after Phase 0)
- Create `ReboundMan/ReboundMan-ReleaseToolkit`
- Land `win-app-release.yml` + composite actions
- Tag `v1.0.0`
- Document onboarding in `docs/ONBOARDING.md`

### Phase 2 — WordMD pilot (~half day)
- Drop in `release.yml` stub
- Test with `workflow_dispatch` (Mode A) → verify draft + signed installer
- Cut `v1.4.2` tag → verify Mode C end-to-end
- Wait one Friday → verify Mode B
- Update `INSTALL.md` (remove "Code-signing (deferred)" section)

### Phase 3 — TokenTray onboarding (~1 hour)
- Copy stub, adjust 4 paths, tag `v1.0.0`
- Validates that the contract really is reusable
- Repo: `ReboundMan/TokenTray` (note: bare name, no `ReboundMan-` prefix)

### Phase 4 — Copilot CLI skill (~1 day)
- `release` skill with the verbs in §6
- Config file under `~/.copilot/skills/release/apps.yaml`
- Auto-discovery from `gh repo list ReboundMan` as a stretch goal

### Phase 5 — Polish (~ongoing)
- WinGet manifest PR step
- arm64 build matrix
- Auto-attached SHA256 + SBOM
- Release notes templating per app (e.g., screenshots auto-pulled from /samples)

---

## 9. Operational runbook (what life looks like after Phase 4)

**Normal week (no changes):**
- Friday 11pm PT, cron runs, sees no commits, exits silently.

**Normal week (a few bug fixes):**
- Friday 11pm PT, cron builds + signs + drafts `v1.4.2`.
- You get a GitHub email "New draft release."
- `release notes wordmd` → tweak the changelog if needed.
- `release publish wordmd` → live on GitHub Releases.
- Done in ~3 minutes.

**Hotfix (you want it out now):**
- `git tag v1.4.3 && git push --tags`
- Mode C builds, signs, publishes automatically.
- ~10 minutes end-to-end.

**Big feature week:**
- Land PRs, tag `v1.5.0`, Mode C ships it.
- Or use Mode A `release kickoff wordmd 1.5.0` for a draft pass first.

---

## 10. What I'm NOT recommending (and why)

- **Per-repo duplicated workflows.** Tempting for "simplicity" but every
  signing tweak becomes an N-repo edit. The toolkit pattern is worth the
  one-time setup.
- **Self-hosted runner.** GitHub-hosted `windows-latest` covers WinUI + Inno
  builds fine. No infra to babysit.
- **Building the skill first.** The workflow alone is the high-value piece.
  The skill is dessert.
- **EV code-signing cert purchase.** Azure Trusted Signing gets you the same
  SmartScreen reputation for ~$10/mo with no hardware token.
- **Auto-publish on every push to main.** Too aggressive — easy to ship a
  half-finished feature. Tags are the explicit "this is ready" signal.

---

## 11. Estimated effort

| Phase | Calendar time | My hands-on-keyboard time |
|---|---|---|
| 0 — Signing prereqs | 2–7 days (mostly waiting on identity validation) | ~1 hr to walk you through it |
| 1 — Toolkit repo | 1 day | ~3 hrs |
| 2 — WordMD pilot | 0.5 day | ~1 hr + 1 week observation |
| 3 — TokenTray | 1 hour | 30 min |
| 4 — Skill | 1 day | ~3 hrs |
| **Total active work** | | **~8 hrs over ~2 weeks** |

---

## 12. Next steps once approved

1. Confirm decisions in §7 (especially #1, #3, #6).
2. I'll write a detailed `SIGNING.md` runbook for Phase 0 you can follow
   yourself, since the Azure portal clicks are human-only. **This runbook is
   kept private** (gitignored under `docs/private/`) so the specific Azure
   resource/account naming, role assignments, and federated-subject syntax
   don't become public reconnaissance for our infra.
3. Once Trusted Signing is live, I'll scaffold the `ReboundMan-ReleaseToolkit` repo.
4. Pilot it against WordMD before touching TokenTray.

---

## 13. Rollback workflow

> Best rollback is a forward fix. The workflow below is for the rare case
> where you genuinely need to stop people downloading a bad version.

### Flavor A — Yank a bad release (workflow)

`.github/workflows/win-app-rollback.yml` (reusable, lives in this toolkit):

**Trigger:** `workflow_dispatch` only — never automatic.

**Inputs:**
- `version` (required) — e.g. `1.4.2`
- `reason` (optional) — short text, prepended to the release notes
- `delete_tag` (optional, default `false`) — remove the git tag so `git describe` skips it

**Steps:**
1. Verify the release exists. Fail fast if not.
2. Re-flag the release as **`--draft`** via `gh release edit`. It disappears from the "Latest" pointer and from the public release listing.
3. Prepend a banner to the release body: `> ⚠️ **YANKED on YYYY-MM-DD** — <reason>. See vX.Y.Z+1 for the fix.`
4. (If `delete_tag: true`) `git push --delete origin vX.Y.Z`.
5. (Phase 5, if WinGet is live) PR to remove the manifest version from `winget-pkgs`.
6. Auto-file a tracking issue titled `Rollback: vX.Y.Z` with the reason, linking to the yanked release.

Safety: requires manual `workflow_dispatch` click. We may later gate it behind a GitHub **Environment** with required reviewers — even though there's just one of you, the extra click is good muscle memory for destructive ops.

### Flavor B — Forward fix (philosophy, no automation)

The yank above only stops *new* downloads. Users on the bad version are already stuck. The cure is:

1. Land the fix on `main` (cherry-pick or revert as appropriate).
2. Tag `vX.Y.Z+1`.
3. Mode C ships the fix automatically within ~10 minutes.

WordMD has no in-app updater today, so users learn about the fix via:
- The yank banner on the bad release page (if they revisit it).
- The new release showing up on the project's Releases page / RSS feed.
- A pinned issue from step 6 of the yank workflow.

### Skill verbs (Phase 4)

- `release rollback wordmd 1.4.2 --reason "Crash on file open"` — triggers Flavor A
- `release download wordmd 1.4.1` — fetches the prior installer for a user who needs to manually revert
- `release status wordmd` — shows any active yanks

### What we explicitly DON'T do

- **Auto-rollback on telemetry signals.** Overkill at single-developer scale; risk of false-positive yanks is worse than the bad release.
- **Force-downgrade installer.** No in-app updater means we'd have to ship a separate "WordMD-Downgrade-1.4.1.exe" — not worth it. Manual reinstall of the prior `Setup.exe` works.
- **Delete the release entirely.** Keeps the prior `Setup.exe` artifact accessible for users who need to revert (and for forensics).
