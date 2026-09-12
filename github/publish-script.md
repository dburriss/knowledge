---
description: Reference publish pipeline for .NET CLI tools — a local version-bump/tag script plus GitHub Actions workflows that publish to NuGet and create a GitHub Release, with steps to adopt it in a new repo.
tags: [github, nuget, dotnet, release]
---

# Publish Script (NuGet + GitHub Releases)

A reusable release pipeline for a .NET CLI tool distributed both as a `dotnet tool` on NuGet and as self-contained binaries on GitHub Releases. Used as-is in [eru](https://github.com/dburriss/eru) and [orcai](https://github.com/dburriss/orcai).

Reference files live alongside this doc in [`publish-script/`](publish-script/):

| File | Role |
|---|---|
| [`publish.sh`](publish-script/publish.sh) | Thin bash wrapper — the entry point you actually run (`./publish.sh`) |
| [`publish.fsx`](publish-script/publish.fsx) | The real logic, run via `dotnet fsi`: bumps version, updates changelog, builds/tests, commits, tags, pushes |
| [`publish-nuget.yml`](publish-script/publish-nuget.yml) | GitHub Actions workflow: packs and pushes to nuget.org on tag push |
| [`publish-gh.yml`](publish-script/publish-gh.yml) | GitHub Actions workflow: builds self-contained binaries for linux/win/macos and attaches them to a GitHub Release on tag push |

## How the pieces fit together

```
./publish.sh
  → dotnet fsi scripts/publish.fsx
      1. read <Version> from the .fsproj
      2. read the "## [Unreleased]" section of CHANGELOG.md
      3. preflight: dotnet build + dotnet test (Release)
      4. prompt: major / minor / patch (or prerelease suffix)
      5. write new <Version> + <PackageReleaseNotes> into the .fsproj
      6. rewrite CHANGELOG.md (stable releases only)
      7. final release build
      8. git add / commit / tag vX.Y.Z
      9. prompt: push to origin?
           → pushing the tag triggers CI
                ├─ publish-nuget.yml  → dotnet pack + dotnet nuget push
                └─ publish-gh.yml     → dotnet publish (per-OS) + GitHub Release with binaries
```

The script never talks to nuget.org or the GitHub API directly — it only produces a signed, tagged commit. The tag push is the trigger; NuGet and GitHub Release publishing happen entirely in CI. This split means a `--dry-run` of the local script is enough to sanity-check a release without any risk of actually publishing.

### Retag flow

If `CHANGELOG.md` has nothing under `## [Unreleased]`, the script assumes you're re-cutting a release from the current commit (e.g. CI failed after tagging) rather than bumping version. It asks for confirmation, deletes the local tag, recreates it at `HEAD`, and force-pushes if you agree — no version or changelog changes happen in this path.

### Prereleases

Answering "yes" to "Is this a prerelease?" appends a suffix (e.g. `0.7.0-beta1`) and **skips** the CHANGELOG.md rewrite, so unreleased entries keep accumulating under `## [Unreleased]` until the eventual stable release. Only the `.fsproj` version is bumped for a prerelease.

## Adopting this in a new repository

1. **Prerequisites in the target repo:**
   - A single `.fsproj` for the CLI tool with `<Version>` and `<PackageReleaseNotes>` elements in its first `<PropertyGroup>`.
   - A `CHANGELOG.md` at the repo root with a `## [Unreleased]` heading (Keep a Changelog style).
   - A solution file (`.sln` or `.slnx`).

2. **Copy the files** from [`publish-script/`](publish-script/) into the target repo:
   ```
   publish.sh        → <repo root>/publish.sh
   publish.fsx       → <repo root>/scripts/publish.fsx
   publish-nuget.yml → <repo root>/.github/workflows/publish-nuget.yml
   publish-gh.yml    → <repo root>/.github/workflows/publish-gh.yml
   ```

3. **Edit `publish.sh`** — set `--fsproj` and `--solution` to the real paths, e.g.:
   ```bash
   dotnet fsi scripts/publish.fsx \
     --allow-dirty \
     --fsproj "src/MyTool.Cli/MyTool.Cli.fsproj" \
     --solution "MyTool.sln" \
     "$@"
   ```
   `publish.fsx` itself needs no edits — it's fully parameterized via `--fsproj`/`--solution`.

4. **Edit `publish-nuget.yml`** — update the `dotnet pack` path to point at the real `.fsproj`.

5. **Edit `publish-gh.yml`** — replace every `yourtool` placeholder with the actual binary/artifact name (it appears in the `matrix.include` entries and the rename/upload steps), and update the `dotnet publish` path to the real `.fsproj`.

6. **Make the wrapper executable:**
   ```bash
   chmod +x publish.sh
   ```

7. **Configure Trusted Publishing on nuget.org** (replaces long-lived API keys):
   - Sign in to nuget.org → your username → **Trusted Publishing** → add a new policy.
   - **Repository Owner:** your GitHub org/user (e.g. `dburriss`)
   - **Repository:** the repo name (e.g. `orcai`)
   - **Workflow File:** `publish-nuget.yml` (file name only, not the `.github/workflows/` path)
   - **Environment:** leave empty unless the workflow uses `environment: release`
   - A policy on a private repo starts in a 7-day "pending" state and only becomes permanent after the first successful publish — this is expected.
   - `publish-nuget.yml` uses the [`NuGet/login`](https://github.com/NuGet/login) action to exchange the job's OIDC token for a 1-hour temporary API key, so it needs `permissions: id-token: write` (already set in the reference workflow) and no `NUGET_DEPLOY_KEY` secret.
   - Optionally set a `NUGET_USER` repo secret with your nuget.org username (profile name, not email) — otherwise hardcode it in the `user:` input.
   - `GITHUB_TOKEN` is provided automatically by Actions; no setup needed for `publish-gh.yml`.

8. **Verify** with `./publish.sh --dry-run` before ever running it for real — it prints every file change and git/gh command it would run without touching disk or git.

## NuGet package metadata (.fsproj)

All package metadata is inline MSBuild properties in the `.fsproj`'s first `<PropertyGroup>` — no separate `.nuspec` file. Example, from [orcai](https://github.com/dburriss/orcai/blob/main/src/OrcAI.Tool/OrcAI.Tool.fsproj):

```xml
<PackageId>OrcAI.Tool</PackageId>
<PackAsTool>true</PackAsTool>
<Version>0.10.5</Version>
<Authors>Devon Burriss</Authors>
<Description>...</Description>
<Title>orcai</Title>
<ToolCommandName>orcai</ToolCommandName>
<Copyright>Copyright © 2025 Devon Burriss</Copyright>
<PackageProjectUrl>https://github.com/dburriss/orca</PackageProjectUrl>
<RepositoryUrl>https://github.com/dburriss/orca</RepositoryUrl>
<RepositoryType>git</RepositoryType>
<PackageLicenseExpression>MIT</PackageLicenseExpression>
<PackageTags>github copilot cli project-management automation</PackageTags>
<PackageReadmeFile>README.md</PackageReadmeFile>
<PackageReleaseNotes>...</PackageReleaseNotes>
<PublishRepositoryUrl>true</PublishRepositoryUrl>
<EmbedUntrackedSources>true</EmbedUntrackedSources>
```

Notes:
- `PackAsTool` + `ToolCommandName` mark it as a .NET global/local tool and set the installed CLI command name.
- `PackageLicenseExpression` uses an SPDX identifier (`MIT`) rather than embedding a license file.
- `PublishRepositoryUrl` + `EmbedUntrackedSources` enable SourceLink / reproducible builds alongside `RepositoryUrl`.
- `PackageReadmeFile` names the file NuGet.org displays, but the actual source can be remapped away from the repo's root README via an `ItemGroup`:
  ```xml
  <ItemGroup>
    <None Include="..\..\NUGET.md" Pack="true" PackagePath="README.md" />
  </ItemGroup>
  ```
  This packs `NUGET.md` into the nupkg as `README.md`, letting the NuGet-facing readme differ from the repo's GitHub readme.
- `publish.fsx` (see above) writes `<Version>` and `<PackageReleaseNotes>` directly into these same properties on each release — `PackageReleaseNotes` ends up as the per-version changelog baked into the package itself.

## Notes / gotchas

- `publish-nuget.yml` uses NuGet [Trusted Publishing](https://learn.microsoft.com/en-us/nuget/nuget-org/trusted-publishing) (OIDC) instead of a long-lived API key secret — the job requests a GitHub OIDC token, exchanges it for a 1-hour nuget.org API key via the `NuGet/login` action, then pushes with that. This requires `permissions: id-token: write` on the job and a matching Trusted Publishing policy registered on nuget.org (see step 7 above).
- The workflows target **.NET 10** (`dotnet-version: '10.0.x'`) — bump this if the target repo is on a different SDK.
- `publish-gh.yml` builds for `linux-x64`, `win-x64`, and `osx-x64` only; add matrix entries for other RIDs (e.g. `osx-arm64`) if needed.
- `dotnet nuget push` uses `--skip-duplicate` so re-running the workflow (e.g. after a retag) won't fail if the package version already exists on nuget.org.
- The changelog-extraction `awk` in `publish-gh.yml` matches `## [VERSION]` exactly against the tag (stripped of its `v` prefix) — keep changelog headings in that exact format or the release body will come out empty.
