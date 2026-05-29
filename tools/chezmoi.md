---
description: Reference for chezmoi, a dotfile and config management tool for keeping configs consistent across machines with support for templating, secrets, and per-machine differences.
tags: [tools, chezmoi, dotfiles, configuration, reference]
---

# chezmoi

**chezmoi** is a dotfile and config management tool. Think “declarative home directory,” with strong support for secrets, templating, and multi-machine drift.

## What it’s for

- Manage dotfiles (`.zshrc`, `.gitconfig`, Hyprland configs, etc.)
- Keep configs consistent across machines
- Handle per-machine differences cleanly
- Manage secrets without committing them in plain text

## Core model

- **Source of truth** lives in `~/.local/share/chezmoi`
- Files there are **applied** to `$HOME`
- You edit the _source_, not the target files

## Typical workflow

```bash
chezmoi init <repo>     # clone dotfiles repo
chezmoi apply           # sync to $HOME
chezmoi edit ~/.zshrc   # edits the source version
chezmoi diff            # see pending changes
chezmoi apply           # apply changes
```

Key point: `chezmoi edit` is preferred over editing files directly.

## File types

- `dot_zshrc` → `~/.zshrc`
- `dot_config/hypr/hyprland.conf` → `~/.config/hypr/hyprland.conf`
- Executables: `run_onchange_*`, `run_once_*`

Naming encodes intent.

## Templating

Uses Go templates.

Example:

```tmpl
{{ if eq .chezmoi.hostname "laptop" }}
set -o vi
{{ end }}
```

Data sources:

- `chezmoi data`
- OS, hostname, username
- Encrypted secrets

This is how you avoid branching repos per machine.

## Secrets

First-class feature.

Options:

- `chezmoi secret keyring`
- `chezmoi secret gopass`
- `chezmoi secret pass`
- Age-encrypted files

Example:

```tmpl
export API_KEY={{ .my_api_key }}
```

Secret never appears in the repo in plaintext.

## Drift control

- `chezmoi diff` → detect config drift
- `chezmoi status` → see unmanaged files
- `chezmoi re-add` → pull live edits back into source

This matters when tools mutate configs behind your back.

## Hooks / automation

- `run_once_*` → bootstrap scripts
- `run_onchange_*` → re-run when inputs change

Common uses:

- Package installs
- Plugin managers
- Recompiling configs

## Why use it over alternatives

**vs bare git repo**

- Handles secrets
- Safer diffs
- No accidental overwrite
- Per-machine logic

**vs Ansible**

- Local, fast
- No inventory
- Designed for `$HOME`, not servers

## Trade-offs / pitfalls

- Go templates add cognitive load
- Debugging template failures can be annoying
- Overuse of conditionals leads to unreadable configs
- Not a full config management system—scope it to user env

## When it’s a good fit

- Multiple Linux/macOS machines
- Tiling WM / heavy dotfile setups
- You care about reproducibility
- You want infra-like discipline for your shell
