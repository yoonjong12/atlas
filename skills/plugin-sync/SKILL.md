---
name: plugin-sync
description: "Sync a Bitbucket-hosted Claude Code plugin from remote to local. Pull latest, refresh cache, verify. Trigger on: 'plugin sync', 'plugin update', '플러그인 싱크', '플러그인 업데이트', 'sync plugin', 'pull plugin', '플러그인 최신화'"
argument-hint: "[plugin-name or local-clone-path]"
allowed-tools: Bash, Read
---

# Plugin Sync — Bitbucket-hosted Claude Code plugins

Pull latest code from a Bitbucket remote into the local plugin clone, refresh the Claude Code plugin cache, and verify all layers match.

## Prerequisites

Git SSH or HTTPS access to the Bitbucket remote. Plugin already installed via `claude plugin marketplace add` + `claude plugin install`.

## Steps

1. **Locate clone.** Find the local plugin clone:
   ```bash
   bash ${CLAUDE_PLUGIN_ROOT}/scripts/bb_plugin_sync.sh locate [plugin-name]
   ```
   Searches `~/.local/share/` and `~/.claude/plugins/marketplaces/` for repos with Bitbucket remotes.

2. **Sync.** Pull latest and refresh cache:
   ```bash
   bash ${CLAUDE_PLUGIN_ROOT}/scripts/bb_plugin_sync.sh sync <clone-path> [branch]
   ```
   Default branch: `main`. Detects if HEAD is detached (tag checkout) and handles accordingly.

3. **Verify.** Compare local vs remote:
   ```bash
   bash ${CLAUDE_PLUGIN_ROOT}/scripts/bb_plugin_sync.sh verify <clone-path>
   ```

## One-shot

```bash
bash ${CLAUDE_PLUGIN_ROOT}/scripts/bb_plugin_sync.sh all [plugin-name] [branch]
```

Runs locate → sync → verify in sequence.

## Notes

- Opposite of `/publish` (outbound push). This is inbound pull.
- For GitHub-hosted plugins, use `claude plugin update` directly.
- After sync, tell user to run `/reload-plugins` if session is active.
