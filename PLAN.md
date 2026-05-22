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
│    SIGNING.md                               │
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
| **B. Scheduled + approval** | `schedule: cron Friday 19:00 UTC` — only proceeds if `git rev-list <last-tag>..HEAD` is non-empty | Draft release | Reviews draft, clicks Publish (or ignores → skip-week) |
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
    - cron: '0 19 * * 5'   # Fridays 11am PT

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

## 7. Open decisions (need your input)

| # | Decision | Default I'd pick |
|---|---|---|
| 1 | Toolkit repo name | `ReboundMan/ReboundMan-ReleaseToolkit` (matches existing `ReboundMan-WordMD` naming) |
| 2 | Toolkit repo visibility | Public (it has no secrets, just reusable YAML) |
| 3 | Signing provider | Azure Trusted Signing (vs. SSL.com EV cert) |
| 4 | Version source of truth | Git tags (`v1.4.2`), workflow stamps everything |
| 5 | Conventional commits required? | **Encouraged, not enforced** — un-prefixed commits land under "Other" in notes |
| 6 | Schedule day/time | Fridays 19:00 UTC (Friday 11am PT) for Mode B |
| 7 | Build skip-week mechanism | Just don't publish the draft. Cron exits if no commits. |
| 8 | WinGet manifest PR step | **Phase 2** — skip until after first signed release lands |
| 9 | Multi-arch (x64 + arm64) | **Phase 2** — x64 only for v1 |
| 10 | Build the skill? | **Yes, but after the workflow ships** — workflow alone covers 80% of the value |

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

### Phase 3 — Token Tray onboarding (~1 hour)
- Copy stub, adjust 4 paths, tag `v1.0.0`
- Validates that the contract really is reusable

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
- Friday 11am PT, cron runs, sees no commits, exits silently.

**Normal week (a few bug fixes):**
- Friday 11am PT, cron builds + signs + drafts `v1.4.2`.
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
| 3 — Token Tray | 1 hour | 30 min |
| 4 — Skill | 1 day | ~3 hrs |
| **Total active work** | | **~8 hrs over ~2 weeks** |

---

## 12. Next steps once approved

1. Confirm decisions in §7 (especially #1, #3, #6).
2. I'll write a detailed `SIGNING.md` runbook for Phase 0 you can follow
   yourself, since the Azure portal clicks are human-only.
3. Once Trusted Signing is live, I'll scaffold the `ReboundMan-ReleaseToolkit` repo.
4. Pilot it against WordMD before touching Token Tray.
