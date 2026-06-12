# DEAR_AGENT

Context for AI agents (and future humans) working on this repo. Read this
before changing anything.

## What this repo is

A fork of [gpakosz/.tmux](https://github.com/gpakosz/.tmux) ("Oh my tmux!"),
deployed as `~/.config/tmux/`. `origin` is rreagan3/.tmux-conf, `upstream` is
gpakosz/.tmux, working branch is `master`.

The upstream rule still applies: **all customization goes in
`tmux.conf.local`**, never `tmux.conf` — with one deliberate exception
documented below.

## Local changes vs upstream

### 1. Plugin manager: tpack, not tpm

Plugins are managed by [tpack](https://github.com/tmuxpack/tpack) (a drop-in
tpm replacement with a TUI), installed two ways on purpose:

- the `tpack` binary via Homebrew (`brew install tmuxpack/tpack/tpack`) — the
  TUI and CLI;
- a clone of the tpack repo at `plugins/tpm` — its `tpm` shim script makes
  Oh my tmux!'s "seamless tpm integration" launch tpack without knowing the
  difference.

Because of that, the three `tmux_conf_*_plugins_*` flags in `tmux.conf.local`
are **false** and must stay false: Oh my tmux!'s updater would `git reset` the
`plugins/tpm` clone back to real tpm and perl-patch scripts that don't exist
in tpack. Update tpack with `tpack self-update`; manage plugins with
`<prefix> + T` (TUI), `<prefix> + I` (install), `<prefix> + u` (update), or the
`tpack` CLI with `TMUX_PLUGIN_MANAGER_PATH=~/.config/tmux/plugins/` exported.

Do not run `tpack clean` blindly — it removes anything not currently
discoverable as declared, and discovery depends on the tmux server being up
with `@tpm_plugins` populated.

### 2. One patch to tmux.conf (upstream PR candidate)

`__apply_plugins` in `tmux.conf` stored the discovered plugin list
newline-separated in the `@tpm_plugins` option. tmux >= 3.6 sanitizes control
characters in option values to `_`, garbling the list into one unparseable
spec. The patch joins with spaces instead (search for `paste -s -d ' '` in
`tmux.conf`). This belongs upstream; until then, preserve it across upstream
merges.

### 3. Session persistence (the point of all this)

Declared in `tmux.conf.local`:

| Plugin | Role |
|---|---|
| tmux-plugins/tmux-resurrect | save/restore layout, cwd, programs (`<prefix> + C-s` / `C-r`) |
| tmux-plugins/tmux-continuum | autosave every 5 min, auto-restore on server start |
| timvw/tmux-assistant-resurrect | re-attach Claude Code sessions after restore |

How Claude resume works: the plugin installs `SessionStart`/`SessionEnd` hooks
in `~/.claude/settings.json` that write/remove
`$TMPDIR/tmux-assistant-resurrect/claude-<pid>.json` containing the session
UUID. Resurrect's post-save hook joins those to panes and writes
`~/.local/share/tmux/resurrect/assistant-sessions.json`; the post-restore hook
sends `claude --resume <uuid>` to each restored pane.

Consequences:
- Claude sessions started **before** the hooks existed are not resumable
  (logged as "no session ID available" at save time). Coverage is complete
  once every long-lived Claude session has been started/restarted under the
  hooks.
- Do **not** add `claude` (or any assistant) to `@resurrect-processes` — the
  plugin would then type resume commands into an already-running TUI.

Auto-start at boot is intentionally off: `@continuum-boot` only knows
Terminal.app/iTerm/kitty/alacritty, and this machine mainly uses Ghostty.
After a reboot, the first `tmux` invocation restores everything.

### 4. Claude-aware window naming

`automatic-rename-format` in `tmux.conf.local` names each window after its
active pane's title when the pane set one, else the cwd basename. Claude Code
titles its pane `<status glyph> <session name>`, so windows track the most
recently focused Claude pane. Manually renaming a window turns this off for
that window (tmux behavior); re-enable with `setw automatic-rename on`.

## Verifying after changes

```sh
tmux source ~/.config/tmux/tmux.conf       # reload (or <prefix> + r)
tmux show -gv '@tpm_plugins'                # space-separated, no '_' garbage
tmux show -gv status-right | grep continuum # autosave hooked in
bash plugins/tmux-resurrect/scripts/save.sh # manual save; check the log lines
```
