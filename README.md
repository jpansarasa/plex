# plex

Plex Media Server, run as a Docker container under systemd.

## Layout

| File | Role |
| --- | --- |
| `install` | Idempotent installer. Run it for the first deploy and after every change. |
| `plex.service` | Owns the container. Host network, `/dev/dri` passthrough, no pull on start. |
| `plex-update.service` | Oneshot wrapper around `check-update`. No `[Install]` section — the timer resolves it by name. |
| `plex-update.timer` | Daily update check, off the boot critical path. |
| `check-update` | Pulls `:latest`, compares to the running image, **stages** (never applies). |

Paths are pinned: the unit hardcodes `/opt/plex` and `/tank/plex`, and
`install` refuses to run from anywhere else.

## Storage

| Path | What | On |
| --- | --- | --- |
| `/tank/plex` | Config + library database | ZFS (`tank/plex`) |
| `/var/plex` | Transcode scratch | NVMe, ext4 |
| `/export/{tv,movies,music}` | Media, read by Plex | ZFS |

`install` creates `/var/plex` owned by `plex:plex`. That matters because
`docker run` creates a missing bind-mount source as `root:root`, and Plex
runs as uid 2004, so on a fresh machine it could not write transcodes.

### Media access is by group, not ownership

**Everything under `/export/{tv,movies,music}` belongs to the group `media`
(gid 1500).** Plex owns none of it — those trees are owned by whichever
accounts write them — so its entire read path is the `media` group membership
that `install` grants:

```bash
usermod --append --groups media plex
```

This is the invariant to protect. A file that lands as `user:user` (a manual
copy, an unpacked archive, a restore) drops out of the group and that episode
or album quietly never appears in Plex — no error in any log, on either side.

`install` therefore checks both the roots and, recursively, everything under
them, and reports drift rather than fixing it: those trees belong to the
services that write them, so correcting them is their owners' call. To fix:

```bash
sudo find /export/music ! -group media -print0 | sudo xargs -0r chgrp media
```

## Recreating on a fresh machine

Prerequisites: Docker, ZFS with a pool named `tank`, systemd, and the `media`
group (created by the `nzb` install).

```bash
sudo git clone https://github.com/jpansarasa/plex.git /opt/plex

# Creates the plex user, the dataset, the transcode dir, registers the
# units, and starts the container.
sudo /opt/plex/install
```

`install` does not back up or restore `/tank/plex`, which holds the library
database, watch history and server identity. Restore it from whatever copy you
keep, or claim a fresh server at <https://plex.tv/claim> and re-add the
libraries.

## Day-to-day

```bash
# Check for an image update by hand (stages, does not apply)
sudo systemctl start plex-update.service
journalctl -u plex-update.service -n 20

# Apply a staged update (interrupts active streams)
sudo systemctl restart plex.service

# Is an update waiting?
cat /run/plex-update-available
```

`install` is safe to run repeatedly; it converges rather than erroring on
things that already exist. The only side effect of a no-change run is the
service restart in the final step.

## Notes

- `plex.service` deliberately does **not** pull on start. Pulling is the update timer's job; a start that waits on the network makes boot order fragile.
- `check-update` sends no push notification; Plex surfaces its own update notices in the UI.
- The container runs with `--rm`, so a stopped container leaves nothing behind; all state is in the two mounts above.
