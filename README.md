# dsh-update — DSH prerelease updater

Keeps the globally installed `@deepseek-ai/dsh` on the newest npm prerelease
and reloads the Web GUI onto it. Everything lives in `~/.dsh/updater/`; the
command is on `PATH` as `dsh-update` (`~/bin/dsh-update`).

**Version 0.1.0**

## Install

```sh
git clone https://github.com/kenjoe41/dsh-updater ~/.dsh/updater
cp ~/.dsh/updater/config.env.example ~/.dsh/updater/config.env
mkdir -p ~/bin && ln -sf ~/.dsh/updater/dsh-update ~/bin/dsh-update
~/.dsh/updater/dsh-update systemd install
```

`config.env` is user-local and gitignored; `config.env.example` documents every
setting. If `dsh`/`npm` are not on `PATH`, set `DSH_NVM_BIN` to the directory
that holds them. `systemd install` writes the updater units and the GUI unit;
the first `auto` (or `reload --force`) moves the GUI under systemd.

## What runs

| Unit | What it does |
|---|---|
| `dsh-web.service` | The Web GUI itself (`dsh --profile web --port 3080 --no-open`), enabled at login, restarted on failure. |
| `dsh-update.timer` | Fires every 15 min. |
| `dsh-update.service` | Oneshot: runs `dsh-update auto` — check, stage, reload when idle. |

Logs: `~/.dsh/updater/update.log` (updater), `npm.log` (npm output),
`web.log` (the web server, including the `dsh web: http://…?token=…` line).

## How an update happens

1. **Check** — reads the npm packument once and resolves the channel
   (`next` by default). The GitHub releases page is linked but never required.
2. **Stage** — `npm install -g --prefix ~/.dsh/updater/stage/<ver>` into an
   isolated prefix. The live npm tree is untouched, so a slow/flaky registry
   can take as long as it needs. Three attempts, then stop.
3. **Smoke** — the staged copy must print its version and compose the whole
   profile (`dsh --profile web --help`), which catches plugin/peer
   incompatibilities before anything is swapped.
4. **Activate (idle-gated)** — only when no session has written for five
   minutes: stop the server, move the current package to `prev/`, move the
   staged package in, start `dsh-web.service`, wait for the port. If the new
   server does not answer within 90 s, the previous package is put back and the
   service restarted (rollback). The browser cookie is signed with a persisted
   secret, so the open tab reconnects instead of asking for a new URL. If a
   switch lands mid-turn anyway (forced reload, crash, or a session that stayed
   busy), `dsh-autocontinue`'s restart-recovery scan reopens every session whose
   persisted log was left inside an open turn and queues a continuation round,
   so work in other workspaces is not orphaned.

## Commands

```bash
dsh-update check [--json]     # installed vs newest; exit 10 = update available
dsh-update releases [N]       # recent releases (GitHub, npm fallback)
dsh-update status [--json]    # versions, service, timer, last outcomes
dsh-update stage [--force]    # download + smoke-test only
dsh-update activate [--force] # swap the staged version in and restart
dsh-update install [--force]  # stage + activate
dsh-update reload [--force]   # restart the GUI on the installed version
dsh-update auto               # what the timer runs
dsh-update log [-n N]         # tail update.log
dsh-update config             # effective configuration
dsh-update systemd install|refresh|uninstall|status
```

`--force` ignores the five-minute idle window.

## Configuration

`~/.dsh/updater/config.env`:

```bash
DSH_UPDATE_CHANNEL="next"     # next | alpha | latest | newest | tag:<name>
DSH_UPDATE_AUTO_INSTALL="1"
DSH_UPDATE_AUTO_RELOAD="1"
DSH_UPDATE_QUIET_SECS="300"
DSH_UPDATE_INTERVAL_MIN="15"  # re-run `dsh-update systemd refresh` after changing
DSH_UPDATE_PORT="3080"
DSH_UPDATE_PROFILE="web"
DSH_UPDATE_KEEP_PREV="1"      # previous installs kept for rollback (~0.5 GB each)
DSH_UPDATE_NOTIFY="1"
```

## Recovery notes

- If the GUI is down: `systemctl --user status dsh-web` and
  `tail -50 ~/.dsh/updater/web.log`.
- If the session was interrupted by a reload, reopen the printed URL:
  `grep 'dsh web:' ~/.dsh/updater/web.log | tail -1`.
- A second manual `dsh --profile web` will fail on port 3080 while the service
  is active; use the service instead (`systemctl --user restart dsh-web`).
- The units hard-code the current nvm Node path. After `nvm alias default`
  changes, edit the two unit files or re-run `dsh-update systemd refresh` with
  `DSH_NVM_BIN` pointing at the new version.
- `prev/` holds the previous package for rollback; `stage/` holds the next one
  until it is activated. Prune with `DSH_UPDATE_KEEP_PREV` (0 disables keeping
  any).
