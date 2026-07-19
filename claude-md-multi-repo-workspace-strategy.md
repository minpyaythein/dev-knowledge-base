# CLAUDE.md placement in a multi-repo workspace

Claude Code loads CLAUDE.md from ancestor directories too, so a workspace-root file covers subrepo sessions — but only committed per-repo files reach teammates and CI.

## Why it matters

A single workspace-root CLAUDE.md feels sufficient because it always loads locally, yet it lives outside every git repo. Teammates, CI, and standalone clones never see it.

## How it works

Discovery runs in both directions, but git visibility decides where content belongs.

1. At launch, Claude Code reads CLAUDE.md in the cwd and every ancestor directory up to `/`.
2. CLAUDE.md files in subdirectories load on demand when Claude touches files there.
3. A file outside the git repos is invisible to clones, CI, and teammates.

So split by audience: the workspace root holds the cross-repo picture (system map, data flows, dependency order between repos); each repo holds only its own commands and conventions. Keep every fact in exactly one place — except a contract shared by two repos, which goes in both, because either repo may be edited standalone.

## Example

| File | Content | Who sees it |
|---|---|---|
| `<workspace-root>/CLAUDE.md` (no git) | multi-repo map, request flow, pipeline | only this machine |
| `<backend-repo>/CLAUDE.md` (committed) | make targets, logging + error-code rules | every clone, CI, team |
| both function repos (committed) | shared queue-message contract, cross-referenced | whoever edits either repo |

The two function repos share a queue schema, so that one paragraph is duplicated in both repos deliberately.

## Gotchas

- Committing CLAUDE.md applies it to every teammate using Claude Code on that repo — agree on content and language (English vs. team language) first.
