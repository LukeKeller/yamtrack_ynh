# Yamtrack (LukeKeller fork) — YunoHost package

This is a **fork** of the official YunoHost-Apps/yamtrack_ynh package, modified to install Yamtrack from `https://github.com/LukeKeller/Yamtrack` instead of the upstream `FuzzyGrim/Yamtrack` repository. Everything else (install/upgrade scripts, OIDC/Dex wiring, conf templates) is identical to upstream — credit to the original maintainer.

Use this when you want to test feature work (e.g., the in-flight Hardcover sync) on a YunoHost server before it's merged or released upstream — including alongside the official `yamtrack` package, since this build uses a different YunoHost app id.

## App identity (different from the official package)

| Field | This fork build | Official package |
|---|---|---|
| YunoHost id | `yamtrack_fork` | `yamtrack` |
| Display name | `Yamtrack (LukeKeller fork)` | `Yamtrack` |
| Default install path | `/yamtrack-fork` | `/yamtrack` |
| Default reverse-proxy port | `8096` | `8095` |
| System user | `yamtrack_fork` | `yamtrack` |
| systemd services | `yamtrack_fork`, `yamtrack_fork-celery`, `yamtrack_fork-celery-beat` | `yamtrack`, ... |
| PostgreSQL DB | `yamtrack_fork` | `yamtrack` |
| nginx vhost include | `/etc/nginx/conf.d/<domain>.d/yamtrack_fork.conf` | `.../yamtrack.conf` |

Because everything namespaces off the id, you can install both packages on the same YunoHost server without collision.

## Why a separate `_ynh` repo

YunoHost validates the URL passed to `yunohost app install` with the regex
`^https://[host]/[owner]/[repo]_ynh(/tree/[branch])?$` — the segment immediately
after the owner must end in `_ynh`. Our main fork is `LukeKeller/Yamtrack`, which
doesn't match, so URL installs against this repo are rejected. To get a working
URL install we publish these packaging files to a dedicated repo named
`LukeKeller/yamtrack_ynh`. The main fork keeps a copy under `yunohost-package/`
for visibility, and an orphan `yunohost-package` branch on the main fork holds
the same files at the root for the local-path install.

## Install on your VPS

### A. From a single URL (recommended)

Requires the dedicated `LukeKeller/yamtrack_ynh` repo to exist (see "Publishing"
below). On the VPS:

```bash
sudo yunohost app install https://github.com/LukeKeller/yamtrack_ynh
```

### B. From a local path (works without the dedicated repo)

```bash
ssh root@<your-vps>
git clone -b yunohost-package --single-branch \
  https://github.com/LukeKeller/Yamtrack.git /tmp/yamtrack-pkg
sudo yunohost app install /tmp/yamtrack-pkg
```

Or, if you'd rather have the whole Yamtrack source tree on hand:

```bash
git clone -b main --single-branch \
  https://github.com/LukeKeller/Yamtrack.git /tmp/yamtrack
sudo yunohost app install /tmp/yamtrack/yunohost-package
```

## Publishing the dedicated `_ynh` repo (one-time setup)

1. On github.com, create an **empty** public repo named exactly `yamtrack_ynh`
   under your account (no README/license/.gitignore — just the bare repo).
2. On your dev machine, push this directory's contents to it as `main`:

   ```bash
   # Pull just the packaging files (the orphan branch on Yamtrack)
   git clone -b yunohost-package --single-branch \
     https://github.com/LukeKeller/Yamtrack.git /tmp/yamtrack_ynh
   cd /tmp/yamtrack_ynh
   git push https://github.com/LukeKeller/yamtrack_ynh.git yunohost-package:main
   ```

After step 2 the URL install (method A above) works.

## Shipping flow (Yamtrack feature → YunoHost upgrade)

The canonical flow has two repos, each with `main` as its integration branch:

- **`LukeKeller/Yamtrack`** → integration branch is `main`. Every feature lands on `main`, followed by an empty bump-marker commit (`Bump fork package to 0.25.2~ynhNN (<feature>)`). Upstream (`FuzzyGrim/Yamtrack`) uses `dev`/`main` separately; this fork doesn't.
- **`LukeKeller/yamtrack_ynh`** (this repo) → integration branch is `main`. `manifest.toml` is the source of truth for the version YunoHost installs.

Both branches should always reflect the currently-deployed state. Do not park bumps on long-lived feature branches.

### Steps

1. **Finish the feature in Yamtrack** and merge it into `main` (rebase/fast-forward preferred). Add the empty bump-marker commit on `main` and push. Record the resulting `main` HEAD SHA.
2. **In this repo, on `main`**, update the pin and version together. **Derive `NN` from the live remote — both repos — every time**, not from a number you remember:
   ```bash
   git fetch origin main
   git log --oneline origin/main --grep="Bump to 0\.25\.2~ynh" -1   # this repo's last bump
   # Cross-check that Yamtrack's last "Bump fork package to 0.25.2~ynhNN" matches.

   git checkout main && git pull --ff-only
   ./bump-source.sh <yamtrack-main-head-sha>    # rewrites url + sha256 in manifest.toml
   sed -i 's/^version = ".*"/version = "0.25.2~ynhNN"/' manifest.toml   # bump NN
   git add manifest.toml
   git commit -m "Bump to 0.25.2~ynhNN: <short feature description>"
   git push origin main
   ```
   `NN` here must equal `NN` in the Yamtrack bump marker — the two repos stay aligned by convention. If they're off by one, something went wrong upstream; stop and reconcile before pushing.
3. **Upgrade on the VPS** (the YunoHost app id is `yamtrack_fork`, not `yamtrack`):
   ```bash
   # Method A — pull straight from the catalog repo
   sudo yunohost app upgrade yamtrack_fork -u https://github.com/LukeKeller/yamtrack_ynh

   # Method B — local working copy
   cd /tmp/yamtrack-pkg && git pull
   sudo yunohost app upgrade yamtrack_fork -u /tmp/yamtrack-pkg
   ```

After the upgrade succeeds, the feature branches in both repos can be deleted locally and on the remote.

### `bump-source.sh` details

`./bump-source.sh <commit-or-branch>` resolves the ref to a commit SHA on `LukeKeller/Yamtrack`, downloads the matching GitHub tarball, computes its SHA256, and rewrites `[resources.sources.main]` in `manifest.toml`. Running it with no argument uses the current local HEAD of *this* repo, which is rarely what you want — always pass the Yamtrack SHA explicitly.

### Mirroring to the `yunohost-package` branch on Yamtrack

The Yamtrack repo keeps a copy of these files under `yunohost-package/` and on an orphan `yunohost-package` branch, used by the "local path install" method (Method B in the install section above). When you change anything in this repo that affects the install (manifest, scripts, conf), also refresh that mirror — or leave a note in the bump-marker commit if you skipped it.

## What the package does

Same as upstream:
- Provisions a Python venv from `requirements.txt`, runs `manage.py migrate` + `collectstatic`.
- Sets up nginx + three systemd units (`yamtrack`, `yamtrack-celery`, `yamtrack-celery-beat`).
- Provisions a PostgreSQL database and a Redis logical DB.
- Optionally registers Yamtrack as an OIDC client in Dex (`enable_sso=true`) so YunoHost SSO can sign users in. With `enable_sso=false` it falls back to Yamtrack's local accounts.
- Creates a Yamtrack admin matching the YunoHost user passed to `--admin`.

See `doc/ADMIN.md` (or `doc/ADMIN_fr.md`) for runtime configuration knobs (`TRAKT_API`, `SIMKL_ID`, etc.).

## Caveats specific to running a fork build

- **`autoupdate.strategy` is removed** from `manifest.toml` because the fork doesn't tag GitHub releases. The YunoHost CI auto-updater would have nothing to track. If you start tagging releases on `LukeKeller/Yamtrack`, you can re-add `autoupdate.strategy = "latest_github_release"` and drop the per-commit pin.
- **You're responsible for keeping the source pin moving forward.** Upstream's package gets bumped automatically; this one does not.
- **Don't submit this fork upstream.** This file lives only in your fork, not in `YunoHost-Apps/yamtrack_ynh`. If you want to contribute back, open a PR against the upstream repo with only the genuinely-shared changes.
