# copilotcli-fossil

A [GitHub Copilot CLI](https://docs.github.com/en/copilot/github-copilot-in-the-cli) extension that adds **Fossil SCM** awareness — both to the AI agent's context and to the TUI status bar.

Without this extension, Copilot CLI only detects Git repositories. In a Fossil checkout it shows no branch info and the agent doesn't know to use `fossil` commands. This extension fixes both.

## What it does

1. **TUI branch display** — Shows your Fossil branch in the status bar (e.g. `[⎇ trunk·fossil]`)
2. **Stays fresh** — Automatically updates when you switch branches, even mid-session
3. **AI context** — Injects Fossil branch, dirty status, and repo path into the system prompt
4. **Self-contained** — No shell wrappers, no `.bashrc` changes, no persistent files. Just the extension.

## How it works

```
Session start    → creates temporary .git/ spoof → TUI reads branch
Branch switch    → fs.watch detects .fslckout change → updates .git/HEAD
Each turn        → onUserPromptSubmitted hook → refreshes .git/HEAD
Fossil commands  → onPostToolUse hook → refreshes .git/HEAD
Session end      → onSessionEnd hook → removes .git/ entirely
```

The spoofed `.git/` is minimal (empty commit + HEAD pointing to `<branch>·fossil`), clearly marked as fake in its `description` file, and removed automatically.

## Requirements

- GitHub Copilot CLI v1.0.40+
- [Fossil SCM](https://fossil-scm.org) installed and on `PATH` (or at `~/.local/bin/fossil`)
- A Fossil checkout (directory containing `.fslckout` or `_FOSSIL_`)

## Installation

### User-wide (all Fossil repos)

```bash
mkdir -p ~/.copilot/extensions/fossil-vcs
curl -o ~/.copilot/extensions/fossil-vcs/extension.mjs \
  https://raw.githubusercontent.com/turgs/copilotcli-fossil/main/extension.mjs
```

### Per-project

```bash
mkdir -p .github/extensions/fossil-vcs
curl -o .github/extensions/fossil-vcs/extension.mjs \
  https://raw.githubusercontent.com/turgs/copilotcli-fossil/main/extension.mjs
```

Then restart Copilot CLI or type `/clear`.

## Safety

- Only creates `.git/` if one doesn't already exist (won't clobber real git repos)
- If a real `.git/` is present, the extension only injects AI context — no filesystem changes
- The spoofed `.git/description` contains a clear marker (`Spoofed by copilotcli-fossil`)
- Cleanup fires on session end; even if it doesn't (crash/SIGKILL), the spoof is inert and will be replaced on next session start

## Caveats

- **IDE detection**: While a CLI session is active, VS Code / JetBrains may briefly detect `.git/`. It's harmless (exclude-all is set) and removed when the session ends.
- **One extension, one cwd**: The extension operates on `process.cwd()` at launch. If you open Copilot CLI from a different directory, restart the session.

## Compared to the old approach

The previous version (`turgs/copilot-fossil`) required:
- A `.bashrc` shell wrapper to keep `.git/HEAD` in sync
- Persistent `.fossil-dirty` / `.fossil-extras` marker files
- A `.gitignore` to suppress VS Code warnings

This version needs none of that. It's a single file that handles everything.

## Compatibility

Built for the Copilot CLI extension system (v1.0.40+). Uses `@github/copilot-sdk/extension` with:
- `systemMessage.customize` for AI context
- `hooks.onUserPromptSubmitted` for per-turn refresh
- `hooks.onPostToolUse` for post-fossil-command refresh
- `hooks.onSessionEnd` for cleanup

## License

MIT
