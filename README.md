# ReboundMan Release Toolkit

> Reusable GitHub Actions workflow + composite actions for building, signing,
> and publishing **ReboundMan** Windows desktop apps (WordMD, Token Tray, and
> future apps) on a consistent, low-touch cadence.

## Status

🚧 **Phase 0 in progress.** See [`PLAN.md`](./PLAN.md) for the full
architecture. Azure Trusted Signing setup runbook is intentionally kept
private (`docs/private/SIGNING.md`, gitignored) — ping the maintainer for
access.

**Decisions locked:**
- Repo name: `ReboundMan/ReboundMan-ReleaseToolkit` (public)
- Signing: Azure Trusted Signing via federated OIDC
- Schedule: `0 6 * * 6` (Sat 06:00 UTC = Fri 11pm PDT)
- Versioning: git tags as single source of truth
- Conventional commits: encouraged, never enforced
- Rollback: yank-via-workflow + forward-fix philosophy (see PLAN §13)

**Still pending (Phase 0 human-only steps):**
- [ ] Choose Azure subscription (MSDN vs PAYGO)
- [ ] Choose publisher identity name on cert
- [ ] Complete Azure Trusted Signing identity validation (1–3 day Microsoft wait)
- [ ] Drop org-level GitHub secrets in `ReboundMan` org settings

## What this repo will contain (once Phase 1 lands)

```
.github/
  workflows/
    win-app-release.yml         ← reusable workflow consumed by app repos
  actions/
    setup-windows-build/        ← .NET + Node + Inno Setup
    azure-trusted-signing/      ← signtool wrapper for Azure Trusted Signing
    inno-compile/               ← ISCC.exe invocation
    generate-release-notes/     ← changelog from PRs/commits since last tag
    winget-pr/                  ← (Phase 5) winget-pkgs manifest PR
docs/
  ONBOARDING.md                 ← how to wire a new app into the toolkit
  SIGNING.md                    ← Azure Trusted Signing setup runbook (PRIVATE — gitignored, lives at docs/private/SIGNING.md)
  RUNBOOK.md                    ← what life looks like week-to-week
templates/
  release.yml.template          ← copy-paste stub for new app repos
PLAN.md                         ← architecture + phased rollout
README.md                       ← (this file)
LICENSE                         ← MIT
```

## Consumers (planned)

| Repo | Onboarded |
|---|---|
| [`ReboundMan/ReboundMan-WordMD`](https://github.com/ReboundMan/ReboundMan-WordMD) | Phase 2 (pilot) |
| `ReboundMan/TokenTray` | Phase 3 |
| future apps | Add a ~30-line `release.yml`, done |

## License

[MIT](./LICENSE) © ReboundMan.
