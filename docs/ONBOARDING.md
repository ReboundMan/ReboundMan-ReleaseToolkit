# Onboarding a new app to the Release Toolkit

> **Audience:** future-you, when you're standing up app #3, #4, #N. The pattern
> should take ~15 minutes per app once Phase 0 + Phase 1 are done.

## Prerequisites

- [ ] Azure Trusted Signing is live and identity-validated (private runbook: `docs/private/SIGNING.md`, ask the maintainer)
- [ ] Org-level GitHub secrets are in place (see SIGNING §5)
- [ ] `ReboundMan/ReboundMan-ReleaseToolkit` has a tagged `v1.x.x` release of the reusable workflow
- [ ] Your app already has:
  - [ ] An Inno Setup script (`installer/<App>.iss`) that produces a Setup.exe
  - [ ] A working `dotnet publish` (or equivalent) command that emits a self-contained build
  - [ ] At least one `vX.Y.Z` git tag on `main`

---

## Step 1 — Add a federated credential for the new repo (one-time)

In your Azure CLI shell (see the private signing runbook §4 for context — `docs/private/SIGNING.md`):

```bash
APP_ID="<the github-actions-reboundman-signing app id>"
REPO="ReboundMan/<your-new-app>"

az ad app federated-credential create --id "$APP_ID" --parameters "{
  \"name\": \"github-$(echo $REPO | tr '/' '-')-main\",
  \"issuer\": \"https://token.actions.githubusercontent.com\",
  \"subject\": \"repo:$REPO:ref:refs/heads/main\",
  \"audiences\": [\"api://AzureADTokenExchange\"]
}"

az ad app federated-credential create --id "$APP_ID" --parameters "{
  \"name\": \"github-$(echo $REPO | tr '/' '-')-tags\",
  \"issuer\": \"https://token.actions.githubusercontent.com\",
  \"subject\": \"repo:$REPO:ref:refs/tags/v*\",
  \"audiences\": [\"api://AzureADTokenExchange\"]
}"
```

## Step 2 — Drop in the release workflow stub

Copy [`templates/release.yml.template`](../templates/release.yml.template) to
`.github/workflows/release.yml` in your app repo and adjust the inputs:

| Input | What to set it to |
|---|---|
| `app_name` | Display name, e.g. `WordMD` |
| `publish_command` | The exact `dotnet publish` (or similar) command that builds your self-contained output |
| `publish_output_dir` | Path the publish command writes to |
| `inno_script` | Path to your `.iss` file |
| `primary_exe` | Filename of the main exe to sign (e.g. `WordMD.exe`) |
| `installer_glob` | Glob that matches the installer output (e.g. `dist/*Setup*.exe`) |
| `web_bundle` | `true` if your build needs `npm install && npm run build` first |

## Step 3 — Scope GitHub secrets to the new repo

If you used org-level secrets (recommended), edit each one's "Repository access" list to include the new repo. No values to copy.

## Step 4 — Smoke test in each mode

| Mode | Test |
|---|---|
| **A — Manual** | Actions tab → `Release` → `Run workflow` with no inputs → verify a draft release appears with a signed installer attached |
| **C — Tag push** | `git tag v0.0.1-test && git push --tags` → verify a published release appears within ~10 min. **Delete the test release + tag afterwards.** |
| **B — Scheduled** | Just wait for next Friday 11pm PT. Or temporarily change the cron to a few minutes out, push, observe, revert. |

## Step 5 — Update the toolkit's consumer table

Edit [`../README.md`](../README.md) "Consumers" section and add a row for your new app with the date you onboarded.

---

## App-specific tweaks (the few things the template can't handle)

### Versioning files

The toolkit stamps `MyAppVersion` in `.iss` and `<Version>` in `.csproj`. If your app reads its version from somewhere else (e.g. `version.txt`, a `Constants.cs` literal, an `assemblyinfo.cs`), add a `pre_build_command` input that runs a sed/regex stamp before `publish_command`.

### Multi-project solutions

If your `dotnet publish` needs to target a specific project, just include the project path in `publish_command`:

```yaml
publish_command: |
  dotnet publish src/MyApp.Desktop/MyApp.Desktop.csproj -c Release -r win-x64 -p:Platform=x64
```

### Apps with native dependencies

The runner image (`windows-latest`) has the .NET SDK, Node, Inno Setup (via composite action), and the Windows App SDK. If your app needs anything else (CMake, MSBuild SDKs, vcpkg), add a `setup_extra_command` input with the install command.

### Apps without an installer

If your app ships as a portable zip rather than a Setup.exe, set `installer_glob: dist/*.zip` and the workflow will sign the contained exe before zipping. (Phase 1+ feature — until then, write your own `.iss` that wraps the zip.)

---

## Common pitfalls

| Pitfall | Fix |
|---|---|
| Workflow runs, fails at signing with `AADSTS70021` | Federated credential subject doesn't match. Recheck Step 1; make sure you set both `main` and tags. |
| Inno Setup compile fails with "version mismatch" | Your `.iss` has a hardcoded `MyAppVersion`. The toolkit's stamp step expects the `#define MyAppVersion "..."` line; check the format matches the regex in `templates/release.yml.template` |
| Draft release has empty changelog | No commits since last tag, OR the workflow couldn't read PRs (check `permissions: contents: write, pull-requests: read`) |
| Release ships with arch=`any` instead of `x64` | Inno Setup script missing `ArchitecturesAllowed=x64compatible`. See WordMD's `.iss` for reference. |
