# ReboundMan Release Toolkit

> Reusable GitHub Actions workflow + composite actions for building, signing,
> and publishing **ReboundMan** Windows desktop apps (WordMD, Token Tray, and
> future apps) on a consistent, low-touch cadence.

## Status

🚧 **Planning / pre-implementation.** See [`PLAN.md`](./PLAN.md) for the
full architecture, three operating modes (manual / scheduled-with-approval /
fully automated), phased rollout, and open decisions.

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
  SIGNING.md                    ← Azure Trusted Signing setup runbook
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
| `ReboundMan/ReboundMan-TokenTray` | Phase 3 |
| future apps | Add a ~30-line `release.yml`, done |

## License

[MIT](./LICENSE) © ReboundMan.
