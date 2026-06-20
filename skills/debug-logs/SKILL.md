---
name: debug-logs
description:
  "Store all debug logs in ~/dev/shannon/logs/. Use when running apps,
  configuring log paths, or troubleshooting output."
---

# Debug Logs

All debug logs go in `~/dev/shannon/logs/`. This directory is gitignored. Never
write logs to `/tmp/` or any other location outside the repo.

## Log Directory

```
~/dev/shannon/logs/
```

Create it if it doesn't exist:

```bash
mkdir -p ~/dev/shannon/logs
```

## Naming Convention

Log files are named for the command, test, or issue being investigated:

| Work                  | Log file                         |
| --------------------- | -------------------------------- |
| Shannon run           | `shannon-run.log`                |
| Nushell upgrade issue | `issue-0041-nushell-upgrade.log` |
| Signal handling test  | `signal-handling.log`            |

## Per-App-Type Redirection

### Launchd plists

Set `StandardOutPath` and `StandardErrorPath` in the plist XML:

```xml
<key>StandardOutPath</key>
<string>/Users/astrohacker/dev/shannon/logs/<name>.log</string>
<key>StandardErrorPath</key>
<string>/Users/astrohacker/dev/shannon/logs/<name>.log</string>
```

### CLI binaries and scripts

```bash
./binary args > ~/dev/shannon/logs/<name>.log 2>&1
```

Or to see output live while also logging:

```bash
./binary args 2>&1 | tee ~/dev/shannon/logs/<name>.log
```

### Website or docs scripts

```bash
bun run server.ts > ~/dev/shannon/logs/<name>.log 2>&1
```
