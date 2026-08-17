# Releasing

This repository uses [Changesets](https://github.com/changesets/changesets) for version management and an automated GitHub Actions workflow for NuGet releases.

Packages are published to this repository's **GitHub Packages** NuGet feed
(`https://nuget.pkg.github.com/<owner>/index.json`), authenticated with the workflow's
built-in `GITHUB_TOKEN`. No additional secrets are required.

## Packages

This repo publishes three NuGet packages independently:

| NuGet package | Changeset package | Version source | Project file |
| --- | --- | --- | --- |
| `PostHog` | `PostHog` | `src/PostHog/package.json` | `src/PostHog/PostHog.csproj` |
| `PostHog.AspNetCore` | `PostHog.AspNetCore` | `src/PostHog.AspNetCore/package.json` | `src/PostHog.AspNetCore/PostHog.AspNetCore.csproj` |
| `PostHog.AI` | `PostHog.AI` | `src/PostHog.AI/package.json` | `src/PostHog.AI/PostHog.AI.csproj` |

The package-specific `package.json` files are Changesets metadata only. They intentionally do not declare internal package dependencies, so Changesets releases only the packages selected by changesets. The actual NuGet package dependencies come from the `.csproj` project references when packages are built.

## How to release

### 1. Add a changeset

When making a releasable change, run:

```bash
pnpm changeset
```

Select the package or packages that changed:

- Core SDK change: select `PostHog`
- ASP.NET Core integration change: select `PostHog.AspNetCore`
- AI Observability change: select `PostHog.AI`

Then choose the bump type:

- `patch`: bug fixes, documentation updates, dependency-only updates, and internal changes
- `minor`: backwards-compatible features
- `major`: breaking changes

Commit the generated `.changeset/*.md` file with your PR.

Example changeset for an AI-only patch release:

```md
---
"PostHog.AI": patch
---

Fix OpenAI tracing metadata.
```

### 2. Merge the PR

After review, merge the PR to `main`. A push to `main` that includes `.changeset/*.md` changes automatically starts the release workflow. The workflow then:

1. Checks for pending changesets.
2. Runs `pnpm changeset version` to update the selected package `package.json` versions and changelogs.
3. Syncs those versions into the matching `.csproj` files.
4. Builds and tests the solution.
5. Commits the version bump to `main` as `chore: update package versions [version bump] [skip ci]`.
6. Publishes only the packages whose package versions changed, to GitHub Packages.
7. Creates package-specific GitHub releases and tags, for example `PostHog-v2.6.1` or `PostHog.AI-v0.1.1`, with the `.nupkg` attached as a release asset.

The release is fully automatic once the changeset lands on `main` — there is no approval gate.

### Manual trigger

You can manually trigger the release workflow from the Actions tab with `workflow_dispatch`. Manual runs still require pending changesets.

## Repository prerequisites

One-time setup, required for the workflow to be able to commit and publish:

- **Settings → Actions → General → Workflow permissions**: set to *Read and write permissions*.
  Without this, `GITHUB_TOKEN` cannot commit the version bump or push release tags.
- If `main` is a protected branch, allow the version-bump commit through (or exempt GitHub Actions),
  otherwise the version-bump step fails.

## Consuming the published packages

GitHub Packages requires authentication even for public packages. Consumers need a personal access
token with the `read:packages` scope, and a NuGet source pointing at the feed:

```bash
dotnet nuget add source "https://nuget.pkg.github.com/<owner>/index.json" \
  --name github \
  --username <github-username> \
  --password <token-with-read:packages> \
  --store-password-in-clear-text
```

Alternatively, download the `.nupkg` attached to the corresponding GitHub release.

## Important notes

- Do not edit `.csproj` package versions manually for a release. Edit the relevant package's `package.json` only if doing a one-off correction; normal releases should use `pnpm changeset`.
- The root `package.json` is tooling-only and is not released.
- The workflow publishes packages sequentially with `PostHog` first, because the other packages depend on it.
- If only `PostHog.AI` changes, only `PostHog.AI` is versioned and published.
- If only `PostHog` changes, only `PostHog` is versioned and published, including for major releases.
- Do not add internal `dependencies` entries to the package-specific `package.json` files unless you intentionally want Changesets to couple those packages' releases.

## Troubleshooting

### No changesets found

If the release workflow reports that no changesets were found, make sure your PR includes at least one releasable `.changeset/*.md` file.

### A package did not publish

Check whether that package's `package.json` version changed in the version-bump commit. The publish job intentionally skips packages whose versions did not change.
