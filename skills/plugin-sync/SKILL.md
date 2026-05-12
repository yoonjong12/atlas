---
name: plugin-sync
description: "Sync Claude Code plugins from remote to local. Pull latest from ~/.claude/plugins/marketplaces/, refresh cache, verify. Git-host agnostic. Trigger on: 'plugin sync', 'plugin update', '플러그인 싱크', '플러그인 업데이트', 'sync plugin', 'pull plugin', '플러그인 최신화'"
argument-hint: "[plugin-name or marketplace-clone-path]"
allowed-tools: Bash, Read
---

# Plugin Sync — Claude Code plugin marketplace sync

Pull latest from remote into `~/.claude/plugins/marketplaces/<name>/`, refresh Claude Code plugin cache, verify.

Single source of truth: `~/.claude/plugins/marketplaces/`. Git-host agnostic (GitHub, Bitbucket, any git remote).

## Steps

1. **Locate.** List all plugin clones:
   ```bash
   bash ${CLAUDE_PLUGIN_ROOT}/scripts/bb_plugin_sync.sh locate [filter]
   ```

2. **Sync.** Pull latest + refresh cache:
   ```bash
   bash ${CLAUDE_PLUGIN_ROOT}/scripts/bb_plugin_sync.sh sync <clone-path> [branch]
   ```

3. **Verify.** Compare local vs remote:
   ```bash
   bash ${CLAUDE_PLUGIN_ROOT}/scripts/bb_plugin_sync.sh verify <clone-path>
   ```

## One-shot (all plugins)

```bash
bash ${CLAUDE_PLUGIN_ROOT}/scripts/bb_plugin_sync.sh all [filter] [branch]
```

## Notes

- Opposite of `/publish` (outbound push). This is inbound pull.
- After sync, tell user to run `/reload-plugins` if session is active.
