+++
title = "VASController — Lifecycle Manager for Greenbone Community Edition"
date = 2026-10-18T00:00:00Z
wip = true  # shows a red DRAFT badge next to the title (see extend_head.html)
draft = false
tags = ["tool", "openvas", "greenbone", "docker", "pentest"]
summary = "Bash tool to install, run and archive Greenbone Community Edition (OpenVAS) in Docker: feed import monitoring, version locking, mission backups, offline bundles."
+++

## Summary

**VASController** (`vasctl`) is a single Bash script that manages the full lifecycle of **Greenbone Community Edition** (OpenVAS) running in Docker on Debian and Ubuntu: installation, day-to-day operation, feed monitoring, version locking, per-mission backup and restore, offline deployment and clean removal.

It drives the **official Greenbone container stack** and its unmodified `compose.yaml`. It does not run scans or generate reports; that remains the job of the Greenbone Security Assistant (GSA).

| | |
|---|---|
| **Repository** | [github.com/fabdotnet/VASController](https://github.com/fabdotnet/VASController) |
| **Version** | 1.1.0 |
| **License** | MIT |
| **Language** | Bash (single file, runs as root) |
| **Platforms** | Debian 12+, Ubuntu 22.04+ — amd64 |
| **Recommended resources** | 4 vCPU, 8 GB RAM, 60 GB disk minimum |

## Motivation

Vulnerability scanning in pentest engagements has a recurring constraint: every client should start from a **clean scanner**, while past engagements must stay **retrievable** in case questions come up after delivery.

The previous setup was a virtual machine running a **source-built OpenVAS**. It worked, but with two drawbacks:

- **Upgrades were costly.** Moving a compiled stack to a newer Greenbone release meant rebuilding it, whereas the official containers are upgraded with an image pull.
- **Archiving was heavy.** Keeping an engagement meant keeping the whole VM, i.e. **several GB** per client on local or cloud storage.

A third gap appeared once moving to containers: after an install or a feed update, `gvmd` imports the feed in the background for a long time, with no progress indicator. Greenbone's documentation points to the logs to follow the feed loading ([Workflows](https://greenbone.github.io/docs/latest/22.4/container/workflows.html)), and there is no clear signal telling when the scanner is actually ready.

Greenbone's container guide covers the installation well ([Greenbone Community Containers](https://greenbone.github.io/docs/latest/22.4/container/index.html)). VASController covers what comes after.

## Features

| Area | Commands | Purpose |
|---|---|---|
| Installation | `install`, `passwd`, `self-install` | Docker from the official APT repository, official Greenbone compose, images; installs `vasctl` as a system command |
| Operation | `start`, `stop`, `restart [--hard]`, `status`, `versions`, `disk`, `logs`, `services` | Stack control, soft or hard restart, real disk usage |
| Diagnostics | `doctor` | Pre-scan checks: OS, CPU/RAM, NTP, ports, Docker, container health, feed age, **hypervisor NAT detection** |
| Feed | `feed-status [--watch]`, `update-feed [--wait]`, `schedule-feed` | Import dashboard; optional nightly update via a systemd timer |
| Reproducibility | `lock`, `unlock`, `update` | Pins every image (software **and** feed) to its digest |
| Missions | `backup`, `restore`, `backup-scan`, `restore-scan` | Full backup, or a lightweight end-of-mission archive restorable on a blank machine |
| Offline | `export-bundle [--with-docker]`, `import-bundle` | Whole stack in one archive, installable without Internet |
| Removal | `uninstall [--keep-images \| --keep-data \| --purge-docker]` | Full cleanup, or a blank database without re-downloading images |

`sudo vasctl help <command>` documents each command in detail.

## Feed Import Monitoring

`gvmd` does not expose an import percentage. `feed-status` **infers** progress from several sources and refreshes every 15 seconds with `--watch`:

- **Scanner**: the VT loading phase of `ospd-openvas`, read from its logs. Until it completes, the NVT count on the `gvmd` side stays at zero.
- **gvmd**: the current background task, read from the `gvmd` process titles.
- **NVTs**: rows in the database versus `.nasl` files shipped by the feed, giving a progress bar, a smoothed rate and an ETA.
- **CVEs / CPEs**: counters from the `scap` schema. While `gvmd` rebuilds it into a temporary `scap2` schema, the line shows **building**.
- **CERT-Bund / DFN-CERT**: counters from the `cert` schema.

Changing values are highlighted with their delta. A counter frozen for 10 minutes without being complete is flagged `stalled?`. Completion is detected heuristically: no `gvmd` activity, NVTs at 98 % or more, SCAP and CERT populated, and stable counters across several samples.

```bash
sudo vasctl feed-status --watch               # live dashboard
sudo vasctl feed-status --watch --timeout 90  # gives up after 90 min (exit code 1)
```

## Mission Backups

Two archive types are available:

- **`backup`** — a full backup: `gvmd` database dump, `gvmd` GnuPG key (required to decrypt stored credentials), compose file, lock file if any, and a manifest.
- **`backup-scan --label <mission>`** — the end-of-mission archive. It keeps everything the operator created (targets, credentials, scan configs, tasks, reports, results, overrides, notes) and **excludes the `scap` and `cert` schemas**, which `gvmd` rebuilds from the feed. Compression is zstd level 19 when the PostgreSQL in `pg-gvm` supports it (16+), gzip -9 otherwise.

In practice, a mission archive stays **under 100 MB**, where archiving the equivalent VM took several GB.

Both archive types can be restored **on a blank machine**: if Greenbone is not installed, `vasctl` installs it (online, or from an offline bundle with `--bundle`), starts the stack and restores the database. After a `restore-scan`, `gvmd` rebuilds the SCAP and CERT data, which takes a while; reports remain incomplete on the CVE/CPE side until it finishes.

By default, a restore on a fresh, connected machine installs the latest stable images, so the feed differs from the one used during the mission. Two options keep it identical:

- `--same-versions` reinstalls the exact image digests recorded in the archive (requires an archive taken on a locked stack, and digests still available in the registry).
- `--bundle <file>` installs from a bundle exported at the time of the mission.

## Typical Mission Workflow

```bash
# Before the engagement: fresh feed, then freeze everything
sudo vasctl update-feed --wait
sudo vasctl lock
sudo vasctl doctor

# ... scans in GSA ...

# After the engagement: archive, unlock, blank database
sudo vasctl backup-scan --label client-x
sudo vasctl unlock
sudo vasctl uninstall --keep-images -y

# Later, on any machine
sudo bash vasctl.sh restore-scan vasctl-scan-client-x-<date>.tar --watch
```

After `uninstall --keep-images`, the next `start` brings up a clean stack from the images already on disk. The database is empty and `gvmd` re-imports the feed shipped in those images.

Mission archives contain credentials and their decryption key: they should be stored on encrypted media, or encrypted (e.g. `gpg -c`) before being moved.

## Design Decisions

- **Frozen feed by default.** No automatic update unless explicitly enabled with `schedule-feed`. During an engagement, a stable and known baseline matters more than the latest NVTs. `lock` makes this explicit: while the stack is locked, `update`, `update-feed` and `schedule-feed enable` are refused.
- **The official stack stays untouched.** The lock is a separate override file (`docker-compose.lock.yml`) loaded on top of Greenbone's `compose.yaml`; no modified image is shipped.
- **Guard rails on destructive actions.** `uninstall`, `restore`, an import over an existing installation and `--purge-docker` require the expected word to be typed in full. Archives and bundles embed a `SHA256SUMS` file verified before any change. Manifests are parsed as data, never sourced.
- **Restrictive permissions.** Backups are written `0600` in a `0700` directory; `self-install` installs the script `root:root 0755`.
- **Local access only.** The official compose binds GSA to `127.0.0.1` (ports 443 and 9392). This is kept, and remote access goes through an SSH tunnel:

```bash
ssh -L 8443:127.0.0.1:443 <user>@<scanner-vm>   # then https://127.0.0.1:8443
```

## Limitations

- **Best effort.** There is no roadmap; the tool is maintained as time allows.
- **Platforms**: Debian 12+ and Ubuntu 22.04+ on amd64 only. Other distributions, Kali included, are refused. Greenbone's own container guide documents Debian stable and Ubuntu 24.04 LTS among others.
- **Root required**: the tool drives Docker, APT and systemd.
- **`feed-status` is an estimate**: progress is inferred from the database and completion is a heuristic.
- **NAT detection is a heuristic**: a `doctor` warning is a hint, not a verdict. Bridged networking is preferable for host discovery.
- **Offline bundles** require the same CPU architecture, and the same distribution and release when Docker is installed from the bundle. The source lock is not re-applied on import; each image is verified by ID instead.

## Getting Started

```bash
git clone https://github.com/fabdotnet/VASController.git
cd VASController
sudo bash vasctl.sh install          # Docker + official Greenbone compose + images
sudo bash vasctl.sh self-install     # installs /usr/local/sbin/vasctl
sudo vasctl start
sudo vasctl feed-status --watch      # wait for the import to complete before scanning
sudo vasctl passwd                   # change the default admin password
```

Issues and pull requests are welcome on the [repository](https://github.com/fabdotnet/VASController).

## References

- [VASController repository](https://github.com/fabdotnet/VASController) — source, README, [LICENSE](https://github.com/fabdotnet/VASController/blob/main/LICENSE) (MIT)
- [Greenbone Community Containers](https://greenbone.github.io/docs/latest/22.4/container/index.html) — official container guide
- [Greenbone Community Containers — Workflows](https://greenbone.github.io/docs/latest/22.4/container/workflows.html) — feed updates and loading
- [Official compose.yaml](https://greenbone.github.io/docs/latest/_static/compose.yaml)

## Disclaimer

VASController is not affiliated with or endorsed by Greenbone AG. Greenbone and OpenVAS are trademarks of Greenbone AG.

Only use this tool against systems you are authorized to scan.

## Credits

Developed by Fabien MAISONNETTE (FabDotNET).
