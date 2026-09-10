# knowledge

A collection of reference documents intended for use with [eru](https://github.com/dburriss/eru), a CLI tool for sharing knowledge files across projects via manifests. Pull individual docs or whole collections into any project with `eru add`, or use an MCP.

---

## software-design

Opinionated style guides and design principles for structuring software projects. These documents cover architecture patterns, language-specific conventions, and testing philosophy — intended to be pulled into a project or served via an MCP as a starting point.

| Document | Description |
|---|---|
| [clean-architecture-style-guide](software-design/clean-architecture-style-guide.md) | Opinionated style guide for structuring projects using clean/onion/ports-and-adapters architecture, with naming conventions, use case patterns, and anti-patterns. |
| [fsharp-cli-design](software-design/fsharp-cli-design.md) | Opinionated style guide for designing F# CLI applications using Argu, with patterns for argument parsing, command routing, result handling, and composition. |
| [stratified-design](software-design/stratified-design.md) | How stratified design applies to F# projects using clean/ports-and-adapters architecture, covering layer mapping, rate of change, layer-skipping smells, and a code-review checklist. |
| [testing-style-guide](software-design/testing-style-guide.md) | Style guide for test design using ABC testing (Acceptance, Building, Communication) and behaviour-focused unit tests. |

## tools

Cheat sheets and usage references for developer tools. These are practical, command-focused documents covering installation, common workflows, and configuration — the kind of thing you reach for when setting up a new machine or onboarding onto a tool for the first time.

| Document | Description |
|---|---|
| [chezmoi](tools/chezmoi.md) | Reference for chezmoi, a dotfile and config management tool for keeping configs consistent across machines with support for templating, secrets, and per-machine differences. |
| [ck-search](tools/ck-search.md) | A semantic code search tool that enables searching codebases by meaning rather than just keywords. |
| [gh-cli](tools/gh-cli.md) | Reference for GitHub CLI commands covering projects, issues, and pull requests, with example invocations and output. |
| [git-worktrees](tools/git-worktrees.md) | Git worktrees let you have multiple branches checked out at once, each in its own directory, without extra clones. |
| [mise](tools/mise.md) | Cheat sheet for mise, a polyglot tool version manager for runtimes, env vars, and tasks. |

## github

Reference documents specific to GitHub features and integrations. These cover platform-level topics like GitHub Apps, Copilot configuration, and CLI usage — useful when setting up automation, workflows, or tooling that hooks into the GitHub ecosystem.

| Document | Description |
|---|---|
| [copilot-dotnet-environment](github/copilot-dotnet-environment.md) | Guide for customizing GitHub Copilot's environment for .NET projects, including SDK setup and tool configuration. |
| [github-apps](github/github-apps.md) | Overview of GitHub Apps — what they are, how they differ from OAuth apps, and how they authenticate using installation access tokens. |
| [publish-script](github/publish-script.md) | Reference publish pipeline for .NET CLI tools — a local version-bump/tag script plus GitHub Actions workflows that publish to NuGet and create a GitHub Release. |
