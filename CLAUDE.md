# CLAUDE.md — Notetux++ Ecosystem Orchestrator

This file is the **root context** for the entire Notetux++ project. It describes how the repositories
relate to each other and how to navigate and contribute to each one. Read this first before opening
any sub-repo.

---

## Repository layout

```
/home/andrea/dev/npp/
├── notetux/          ← main application repo  (git: notetux-plus-plus/notetux-plus-plus)
├── nppPluginList/    ← plugin registry repo   (serves the installable-plugin catalogue)
└── plugins/          ← plugin repos (one sub-directory per plugin, each its own git repo)
    ├── hello_world/  ← HelloPlugin   — reference implementation (complete)
    ├── npp_ftp/      ← NppFTP        — FTP/SFTP/SCP remote editing (not started)
    ├── compare/      ← Compare       — side-by-side visual diff (not started)
    └── json_viewer/  ← JSON Viewer   — JSON tree panel + format/minify (not started)
```

---

## Repo 1 — `notetux/` (main application)

**Remote:** `git@github.com:notetux-plus-plus/notetux-plus-plus.git`

A native Linux port of Notepad++ written in C11 with a thin C++ wrapper for Lexilla.
Targets GTK3 and vendors both Scintilla and Lexilla.

See `notetux/CLAUDE.md` for the full source map, build instructions, design rules, and
next-steps backlog. That file is the single source of truth for the application itself.

**Key facts for cross-repo work:**
- Plugin ABI is defined by five mandatory C exports: `getName`, `getFuncsArray`, `beNotified`,
  `messageProc`, `isUnicode`; plus an optional `setInfo(NppData)`.
- `NppData` carries `nppHandle`, `scintillaMainHandle`, `scintillaSecondHandle`,
  the `hostMsg` function pointer, and `configDir[1024]` (pre-filled by the host with
  `~/.config/notetux/plugins/<Name>/config/` before `setInfo` is called).
- Plugins are loaded from `~/.config/notetux/plugins/<Name>/<Name>.so`
  and `/usr/lib/notetux/plugins/<Name>/<Name>.so`.
- The Plugins Admin dialog (`pluginsadmin.c`) fetches the catalogue from the URL stored in
  `g_prefs.plugin_list_url`; default URL points to the raw JSON served by the `nppPluginList`
  repo (see below).
- NPPM messages currently routed by `plugin_host_message()`:
  `NPPM_GETCURRENTSCINTILLA`, `NPPM_GETNBOPENFILES`, `NPPM_GETFULLCURRENTPATH`,
  `NPPM_GETFILENAME`, `NPPM_GETDIRECTORYPATH`, `NPPM_GETPLUGINSCONFIGDIR`.

---

## Repo 2 — `nppPluginList/` (plugin registry)

**Remote:** `git@github.com:notetux-plus-plus/nppPluginList.git` *(create when ready)*

This repo serves the **machine-readable catalogue** of installable plugins. The Plugins Admin
dialog in notetux fetches it over HTTPS and renders it as a browsable list.

See `nppPluginList/CLAUDE.md` for the full JSON schema, validation rules, and how to add
a new plugin entry.

**Quick summary:**
- One JSON file: `v1/notetux_plugin_list.json`
- Each entry has: `name`, `displayName`, `version`, `author`, `description`, `homepage`,
  `repository` (with `download` URL and `sha256` hash), `minAppVersion`.
- The file is fetched at raw GitHub URL and must remain valid JSON at all times on `main`.
- A GitHub Actions workflow validates the JSON on every PR.

---

## Repo 3 — `plugins/<PluginName>/` (individual plugins)

Each plugin lives in its own sub-directory of `plugins/` and is an independent git repo.

**Naming convention:** directory name = plugin identifier, e.g. `hello_world`. Inside it, the
`.so` file and main source file use the PascalCase plugin name, e.g. `HelloPlugin/HelloPlugin.c`.

**Canonical plugin repo layout:**
```
plugins/<plugin_id>/
├── <PluginName>/
│   ├── <PluginName>.c       ← plugin source (C11, includes plugin_api.h)
│   └── Makefile             ← build: produces <PluginName>.so
├── plugin_api.h             ← copy of the public ABI header (from notetux/src/plugin_api.h)
├── CLAUDE.md                ← plugin-specific context (see template below)
├── README.md
└── .github/
    └── workflows/
        └── build.yml        ← CI: compile + upload .so release artifact
```

See `plugins/hello_world/` for the reference implementation.
See `plugins/CLAUDE.md` for plugin development rules and the per-plugin CLAUDE.md template.

### Plugin configuration directory — mandatory convention

Every plugin that persists any state **must** store it under its own config directory:

```
~/.config/notetux/plugins/<PluginName>/config/
```

The host creates this directory automatically before calling `setInfo`. Plugins receive it
in two ways — use whichever is convenient:

**1. Via `NppData.configDir` in `setInfo` (preferred):**
```c
static char s_config_dir[1024] = "";

void setInfo(NppData data)
{
    s_host_msg = data.hostMsg;
    snprintf(s_config_dir, sizeof(s_config_dir), "%s", data.configDir);
}
```

**2. Via `NPPM_GETPLUGINSCONFIGDIR` at any time:**
```c
char dir[1024] = "";
s_host_msg(NPPM_GETPLUGINSCONFIGDIR,
           (unsigned long)(intptr_t)"MyPlugin",
           (long)(intptr_t)dir);
/* dir is now ~/.config/notetux/plugins/MyPlugin/config/ */
```

Rules:
- Never hard-code the config path — always use what the host provides.
- Never write config files outside this directory.
- The directory is guaranteed to exist when `setInfo` returns.
- Uninstalling a plugin = deleting `~/.config/notetux/plugins/<PluginName>/` in its entirety.

---

## CI/CD pipeline (per plugin)

Every plugin repo has `.github/workflows/build.yml` with two behaviours:

- **Pull request** — compiles the `.so` (compile-check gate, no artifact uploaded).
- **Tag push `v*.*.*`** — compiles the `.so`, computes `sha256sum`, and uploads both files
  as assets to an auto-created GitHub Release.

To release a new plugin version:
```sh
git tag v1.2.0 && git push origin v1.2.0
```
CI produces `<PluginName>.so` and `<PluginName>.so.sha256` as release assets. Copy the hash
from the `.sha256` file into `nppPluginList/v1/notetux_plugin_list.json`.

### Cross-distro binary compatibility

All workflows build on **Ubuntu 20.04** (glibc 2.31) so the `.so` runs unchanged on any
distro that can run notetux: Ubuntu 20.04+, Debian 11+, Fedora 33+, openSUSE Leap 15.3+.

Rules enforced in every plugin:
- `-Wl,--as-needed` strips unused dynamic deps from the `.so`.
- Only link dynamically against **glibc + GTK3** — both are guaranteed present on any system
  running notetux.
- Optional runtime libraries (e.g. `libssh2` in NppFTP, `libenchant` for spell-check) are
  loaded with `dlopen` at runtime — **never linked with `-l`**. The CI pipeline for NppFTP
  actively checks that `libssh2` does not appear in `ldd` output and fails the build if it does.
- Never use `ubuntu-latest` as the runner — pin to `ubuntu-20.04` to keep the glibc baseline stable.

---

## Cross-repo workflow

### Adding a new plugin

1. Create `plugins/<plugin_id>/` as a new git repo (copy structure from `hello_world`).
2. Implement the plugin (five mandatory exports + `setInfo`).
3. Push to `git@github.com:notetux-plus-plus/<plugin_id>.git`.
4. Tag `v0.1.0` → CI builds and uploads the `.so` release asset automatically.
5. Copy the SHA-256 from the `.sha256` release asset.
6. Add an entry to `nppPluginList/v1/notetux_plugin_list.json` (PR to `nppPluginList` repo).
7. The plugin becomes visible in Plugins Admin on the next catalogue refresh.

### Updating an existing plugin

1. Bump `version` in `<PluginName>.c` (and README).
2. Tag the new version → CI builds and uploads the `.so` automatically.
3. Copy the new SHA-256 from the `.sha256` release asset.
4. Update `version`, `repository.download`, and `repository.sha256` in `nppPluginList`.
5. PR the catalogue change; CI validates JSON; merge to `main`.

### Updating the plugin ABI in notetux

If `NppData` or the plugin API changes:
1. Update `notetux/src/plugin_api.h`.
2. Bump `minAppVersion` for all affected entries in `nppPluginList`.
3. Update the `plugin_api.h` copy in each plugin repo.

---

## Orchestrator notes for Claude

When working **in this root directory**, your role is to understand the full system:
- If the user asks about a plugin → read `plugins/<plugin_id>/CLAUDE.md` and the plugin source.
- If the user asks about the catalogue → read `nppPluginList/CLAUDE.md` and the JSON file.
- If the user asks about the application → read `notetux/CLAUDE.md`.
- Cross-repo changes (new plugin, ABI change, catalogue update) always touch at least two repos.
- Never modify `notetux/` vendored directories (`scintilla/`, `lexilla/`) unless the change is
  intentional and clearly motivated.
- Plugin repos are independent; a change in one plugin does not require changes in others.
- The `nppPluginList` repo is the **only** authoritative source for what is installable via
  Plugins Admin — keep it accurate and validated.
