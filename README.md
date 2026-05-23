# Notetux++ Orchestrator

This repo is the **single source of truth** for the state of the entire Notetux++ ecosystem.
It contains only one meaningful file — `CLAUDE.md` — which is the shared context document
read by Claude Code across all machines and all sub-projects.

## Why this repo exists

The Notetux++ project spans multiple independent git repositories:

| Repo | Purpose |
|------|---------|
| [notetux-plus-plus](https://github.com/notetux-plus-plus/notetux-plus-plus) | Main application (C11/GTK3 Linux port of Notepad++) |
| [nppPluginList](https://github.com/notetux-plus-plus/nppPluginList) | Plugin registry — JSON catalogue fetched by Plugins Admin |
| [hello_world](https://github.com/notetux-plus-plus/hello_world) | HelloPlugin — minimal reference plugin |
| [npp_ftp](https://github.com/notetux-plus-plus/npp_ftp) | NppFTP — FTP/SFTP/SCP remote editing |
| [compare](https://github.com/notetux-plus-plus/compare) | Compare — side-by-side visual diff |
| [json_viewer](https://github.com/notetux-plus-plus/json_viewer) | JSON Viewer — tree panel + format/minify |

`CLAUDE.md` keeps a high-level description of all these repos, their relationships, the plugin
ABI, the cross-repo workflows, and the current status of each component. When work is done in
any sub-repo, `CLAUDE.md` here is updated so every machine working on the project stays in sync.

## What is tracked

Only two files are committed here:

```
.gitignore    ← ignores everything except itself and CLAUDE.md
CLAUDE.md     ← orchestrator context for the full ecosystem
```

The actual code lives in the sub-repos above.
