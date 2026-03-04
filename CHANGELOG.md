# Changelog

## 2026-03-04

### Fixed
- **Signal handling in Makefile:** Added `exec` to all interactive/long-running targets (`dc.server`, `dc.shell`, `dc.claude`, `dc.tidewave`, `dc.logs.*`) to prevent orphaned processes on CTRL+C
- **`dc.stop`/`dc.down` without running container:** No longer errors when no devcontainer is running; prints a message instead

## 2026-03-02

### Fixed
- **First-run `.claude` directory:** Added `initializeCommand` to create the `.claude` folder before the container starts, preventing a bind-mount failure on fresh projects

## 2026-02-26

### Added
- **Git identity:** Bind-mounts the host's `~/.gitconfig` into the container so git commits use your name and email
- **Git support:** Pre-configured `safe.directory` so git works inside the container; enabled git status in the zsh prompt
- **Git worktrees:** `make dc.worktree.new/list/remove` targets and `wt` shell function for switching between worktrees
- **Parallel Claude workflow:** Run multiple Claude instances in separate worktrees via multiple `make dc.shell` sessions
- **Zsh tab completion:** Enabled `compinit` and make target completion (`make <TAB>`)
- **Download script:** One-liner curl/tar command in README to fetch `.devcontainer/` into any project
- **Stop command:** `make dc.stop` stops the devcontainer without removing it, allowing fast restart with `make dc.up`
- **Tidewave logs command:** `make dc.logs.tidewave` now shows the tidewave logs

### Fixed
- **No more orphaned volumes:** Removed named Docker volumes for mix, hex, and bash history; these now live in the container layer and are cleaned up automatically on `make dc.down`

### Changed
- **Smaller image:** Replaced `vim` with `vim-tiny`, `build-essential` with `gcc`/`make`/`libc-dev`, cleaned docs/man pages and npm cache (~95-145MB savings)
