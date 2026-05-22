# Contributing

This repo holds shared release infrastructure for ReboundMan Windows apps. Most
changes here are YAML edits to workflows and composite actions.

## Commit message style

We use [Conventional Commits](https://www.conventionalcommits.org/) prefixes,
**encouraged but not enforced**. The `generate-release-notes` composite parses
these to auto-group changelog entries. Anything un-prefixed lands under
"Other" — no PR will be blocked.

| Prefix      | Use for                                                          |
|-------------|------------------------------------------------------------------|
| `feat:`     | New capability for consumers (new workflow input, new action)    |
| `fix:`      | Bug fix in existing behavior                                     |
| `perf:`     | Performance improvement                                          |
| `docs:`     | Documentation only                                               |
| `refactor:` | Code restructuring with no behavior change                       |
| `test:`     | Tests, dry-runs, smoke checks                                    |
| `build:`    | Changes to how the toolkit itself is built/packaged              |
| `ci:`       | Changes to CI configuration that aren't part of the public API   |
| `chore:`    | Maintenance (dependency bumps, file moves)                       |
| `style:`    | Formatting/whitespace only                                       |

Append `!` (e.g. `feat!:`) for a breaking change to the public input contract.
Breaking changes get a 💥 section at the top of the release notes.

### Examples

```
feat: add web_bundle input to setup-windows-build
fix(release): handle empty installer_glob without crashing
docs: clarify Mode B skip-week behavior in RUNBOOK
feat!: rename TRUSTED_SIGNING_ACCOUNT secret to TRUSTED_SIGNING_ACCOUNT_NAME
```

## Versioning

The toolkit follows semver. Tags are `vMAJOR.MINOR.PATCH`. Consumers pin to a
major version in their stub:

```yaml
uses: ReboundMan/ReboundMan-ReleaseToolkit/.github/workflows/win-app-release.yml@v1
```

Breaking changes bump major; we also maintain a moving `v1`, `v2`, etc. tag
that points at the latest non-breaking minor.

## Testing changes

Pre-tag changes are tested by pointing a consumer at `@main` temporarily:

```yaml
uses: ReboundMan/ReboundMan-ReleaseToolkit/.github/workflows/win-app-release.yml@main
```

Confirm the consumer's workflow runs cleanly, then merge here, then bump the
moving major tag (e.g. `git tag -f v1 main && git push --force origin v1`).
