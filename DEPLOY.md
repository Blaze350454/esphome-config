# Deploying ESPHome Config Changes

This repo is the dev/backup layer for ESPHome running on the VM at
`192.168.2.14` (dashboard: http://esphome.mynetwork.home:6052 or
http://192.168.2.14:6052). Flow:

```
Windows edit  →  git push  →  GitHub Actions validates  →  VM git pull  →  Dashboard flashes device
```

## 1. Edit on Windows

Work in `D:\Claude\Projects\esphome-config\`.

```bash
git pull --rebase        # sync first — VM's 2:15 AM cron may have pushed
# ...edit *.yaml...
git add <file>.yaml
git commit -m "what changed"
git push
```

## 2. Watch CI

https://github.com/Blaze350454/esphome-config/actions

`validate.yml` runs `esphome config` against every device YAML using a fake
secrets file (`ci/fake_secrets.yaml`). If a device fails, do **not** deploy
— fix the YAML and push again.

## 3. Deploy to the ESPHome VM

```bash
ssh root@192.168.2.14
cd /root/config
git pull --rebase
```

## 4. Flash the device

Two options:

**A. Dashboard (recommended)** — open http://192.168.2.14:6052, find the
device, click **Install** → **Wirelessly** (OTA). The dashboard reads the
updated YAML from `/root/config/` automatically.

**B. CLI from the VM:**

```bash
cd /root/config
esphome run <device>.yaml    # compile + upload via OTA
# or: esphome compile <device>.yaml  # compile only (dry run)
```

## 5. Rollback

If the new firmware is bad, on the VM:

```bash
cd /root/config
git log --oneline -5
git reset --hard <good-commit-sha>
esphome run <device>.yaml    # reflash good version
```

Then on Windows, revert and push so GitHub matches:

```bash
git revert <bad-commit-sha>
git push
```

Note: an unflashable (bricked) device may need USB recovery — OTA can't
reach a non-booting ESP. Keep a known-good commit tagged for this case.

## Nightly auto-backup

`/root/backup-esphome.sh` runs at 2:15 AM via cron (offset from the HA VM's
2:00 AM job to avoid contention). It:

1. `git pull --rebase` — picks up Windows pushes from the day.
2. `git add -A && git commit` — captures dashboard edits.
3. `git push` — sends VM-side changes back to GitHub.

Log: `/root/backup.log`.

## What's NOT in git

Per `.gitignore`:

- `secrets.yaml` — real WiFi/API credentials
- `.esphome/` — ESPHome build cache (500 MB+ of compiled platformio artifacts)
- `*.bin`, `*.elf` — compiled firmware binaries
- `*.bak*`, `*.swp`, `.DS_Store` — editor/OS scratch

Firmware rebuilds cleanly from YAML, so losing the cache is harmless.
`secrets.yaml` is reproduced on a fresh install by copying values from the
operator's password manager.

## Adding a new device

1. Create `<device-name>.yaml` in the repo root with standard ESPHome blocks.
2. Reference secrets via `!secret wifi_ssid` etc.; add any new secret keys to
   both:
   - `/root/config/secrets.yaml` on the VM (real values), and
   - `ci/fake_secrets.yaml` in this repo (CI-safe placeholders)
3. Commit, push, wait for CI, pull on the VM, flash from the dashboard.

## Archiving a device

Move the YAML to `archive/` instead of deleting it — keeps the config
history for reference when the same board is re-used later.
