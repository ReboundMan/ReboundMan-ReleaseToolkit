# ReboundMan Release Toolkit

> Reusable GitHub Actions workflow + composite actions for building, signing,
> and publishing **ReboundMan** Windows desktop apps (WordMD, Token Tray, and
> future apps) on a consistent, low-touch cadence.

## Status

🚧 **Phase 0 + Phase 1 drafted.** Reusable workflow + composite actions + rollback workflow are implemented and YAML-validated. Pilot against WordMD pending two human-only blockers:

1. Azure Trusted Signing identity validation (1–3 day Microsoft wait — start at any time)
2. Org-level GitHub secrets in `ReboundMan` org settings

Once both are done, **tag this repo `v1.0.0`** and onboard the WordMD pilot per [`docs/ONBOARDING.md`](./docs/ONBOARDING.md).

**Decisions locked:** see [`PLAN.md`](./PLAN.md) §7.

**Implemented (this repo):**
- `.github/workflows/win-app-release.yml` — reusable release workflow (Modes A/B/C)
- `.github/workflows/win-app-rollback.yml` — reusable yank workflow (Flavor A in PLAN §13)
- `.github/actions/setup-windows-build/` — .NET + optional Node + Inno Setup
- `.github/actions/azure-trusted-signing/` — wraps `azure/trusted-signing-action` + verifies signatures post-sign
- `.github/actions/inno-compile/` — ISCC.exe wrapper
- `.github/actions/generate-release-notes/` — Conventional Commits → grouped Markdown changelog
- `templates/release.yml.template` — drop-in stub for new app repos
- `CONTRIBUTING.md` — commit-message conventions

**Private (gitignored, ask the maintainer):**
- `docs/private/SIGNING.md` — Azure Trusted Signing setup runbook

## What this repo contains

```
.github/
  workflows/
    win-app-release.yml         ← reusable workflow consumed by app repos
    win-app-rollback.yml        ← reusable yank workflow (see PLAN §13)
  actions/
    setup-windows-build/        ← .NET + Node + Inno Setup
    azure-trusted-signing/      ← signtool wrapper for Azure Trusted Signing
    inno-compile/               ← ISCC.exe invocation
    generate-release-notes/     ← changelog from PRs/commits since last tag
docs/
  ONBOARDING.md                 ← how to wire a new app into the toolkit
  RUNBOOK.md                    ← what life looks like week-to-week
  private/                      ← gitignored: SIGNING.md and other ops detail
templates/
  release.yml.template          ← copy-paste stub for new app repos
PLAN.md                         ← architecture + phased rollout + rollback design
CONTRIBUTING.md                 ← commit message conventions
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
