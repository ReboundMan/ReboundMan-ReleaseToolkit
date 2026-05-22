# Release Runbook

> Day-to-day operational guide for the ReboundMan release pipeline. Things you
> do (or don't do) on any given week.

## The weekly rhythm

### Friday 11pm PT — cron tick

| Repo state | What happens |
|---|---|
| No commits since last tag | Workflow exits silently in <30 sec. No release, no notification. |
| 1+ commits since last tag | Workflow builds, signs, drafts `v<next-patch>`. GitHub emails you "Draft release created." |

### Mon/Tue — your turn

If a draft is waiting:

1. **Skim the auto-generated changelog.** It's grouped by `feat:` / `fix:` / `perf:` / `docs:` / `other`. Reorder, drop noise, add a "Notable" callout if a feature is worth highlighting.
2. **Sanity-check the version.** Patch (`1.4.1 → 1.4.2`) is the default; bump to minor (`1.4.1 → 1.5.0`) if there are user-visible features. Major bumps need a manual override.
3. **Click Publish.** That's it.

If you don't want to publish this week:

- Just don't click Publish. The draft sits there. Next Friday, if there are *more* commits, a new draft supersedes it (the old one auto-discards).
- Or click **Delete draft** to keep things tidy.

### Time budget

- No-change week: 0 min from you
- Normal week: ~3 min reviewing + publishing
- Big-feature week: ~10 min if you want to handcraft notes/screenshots

---

## Triggering off-cycle releases

### Manual draft (Mode A) — "I want to test what the release would look like right now"

```bash
gh workflow run release.yml -R ReboundMan/ReboundMan-WordMD
# or, with explicit version:
gh workflow run release.yml -R ReboundMan/ReboundMan-WordMD -f version=1.4.99-test
```

Produces a draft, doesn't touch tags. Safe to delete after.

### Tag push (Mode C) — "Ship this NOW"

```bash
# In your app repo
git tag v1.4.3
git push --tags
```

Workflow runs end-to-end and **publishes** within ~10 min. No draft step. Use for hotfixes.

---

## Rollback

> If you can ship a forward fix in <30 min, do that instead. The yank workflow
> is for cases where the bug is bad enough that you'd rather stop new installs
> than ship a fix tonight.

### Yanking a bad release

```bash
gh workflow run win-app-rollback.yml -R ReboundMan/ReboundMan-WordMD \
  -f version=1.4.2 \
  -f reason="Crash on file open with non-ASCII filenames"
```

This:
- Marks the release as draft (drops it from "Latest" and the public listing)
- Prepends a `⚠️ YANKED` banner to the release notes
- Files a tracking issue titled `Rollback: v1.4.2`

The release page stays accessible (anyone with the URL can still download the `Setup.exe`), so users who need to revert can manually reinstall the prior version. Just nobody discovers the bad one fresh.

### Deleting the tag (rare)

If you also need `git describe` to skip the yanked version:

```bash
gh workflow run win-app-rollback.yml -R ReboundMan/ReboundMan-WordMD \
  -f version=1.4.2 \
  -f reason="..." \
  -f delete_tag=true
```

> ⚠️ This permanently rewrites tag history. Don't do it if any user has already
> built/cloned against the tag — their tooling will start failing.

### Forward fix after a yank

1. Branch, fix, PR, merge.
2. `git tag v1.4.3 && git push --tags`.
3. Mode C ships it.
4. Close the rollback tracking issue with a link to the new release.

---

## Skipping/pausing the schedule

- **Skip one week:** ignore the draft (or delete it). Done.
- **Pause cron for a month:** comment out the `schedule:` block in `.github/workflows/release.yml` of the affected app, push, set a calendar reminder.
- **Disable an app's release pipeline entirely:** Actions tab → Workflows → `Release` → `⋯` → Disable workflow.

---

## What to do if a release fails mid-flight

Common failure points and triage:

| Failure | What it means | What to do |
|---|---|---|
| `dotnet publish` fails | Build broken on `main` | Fix the build, push, re-run via `gh run rerun <id>` |
| Inno Setup compile fails | `.iss` references a missing file | Check `SourceDir` in `.iss` matches what `publish` produced. Tag version stamps can leave stray `MyAppVersion=""` if the regex misses. |
| Signing step fails | `AADSTS70021` → federation issue; `403` → role missing | See private signing runbook (`docs/private/SIGNING.md` §Troubleshooting) |
| Release creation fails (`gh release create`) | Tag already exists with a different SHA | Means a previous run partially succeeded. Either delete the existing tag (if test) or bump to next patch (if real) |
| Everything succeeds but no installer attached | `installer_glob` matches zero files | Check the path. Workflow logs print "Uploading: <files>". |

`gh run watch <id>` from the Actions tab UI gives you live logs.

---

## Health checks (do once a month)

- [ ] Open WordMD's latest release on a clean Windows VM, click through SmartScreen, confirm no "Unknown publisher" warning
- [ ] Verify the cert chain hasn't been revoked: `signtool verify /pa /v WordMD-Setup-*.exe`
- [ ] Check Azure Trusted Signing portal → Account → ensure no quota warnings
- [ ] Glance at the toolkit repo for any Dependabot PRs (composite actions can pin action SHAs)

---

## Things that should never happen (and what to do if they do)

| Event | Response |
|---|---|
| Workflow publishes a release outside Mode C without your click | Stop. Audit `release.yml`. Some condition allowed a non-tag push to publish. |
| Signing succeeds but with a different publisher name | Check the certificate profile in Azure Trusted Signing. May have been re-bound to a different identity validation. |
| Multiple installer artifacts get uploaded to one release | Probably a `installer_glob` that's too broad. Fix the glob, delete the extras manually. |
| Secret leak (unlikely with OIDC) | Rotate the Service Principal: delete + recreate per private signing runbook §4 (`docs/private/SIGNING.md`). Federated creds re-establish trust without re-issuing secrets. |
