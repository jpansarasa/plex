# plex

Plex Media Server, run as a Docker container under systemd.

## Layout

| File | Role |
| --- | --- |
| `install` | Idempotent installer. Run it for the first deploy and after every change. |
| `plex.service` | Owns the container. Host network, `/dev/dri` passthrough, no pull on start. |
| `plex-update.service` | Oneshot wrapper around `check-update`. No `[Install]` section — the timer resolves it by name. |
| `plex-update.timer` | Daily update check, off the boot critical path. |
| `check-update` | Pulls the configured tag, compares to the running image, **stages** (never applies). |

Paths are pinned: the unit hardcodes `/opt/plex` and `/tank/plex`, and
`install` refuses to run from anywhere else.

## Storage

| Path | What | On |
| --- | --- | --- |
| `/tank/plex` | Config + library database | ZFS (`tank/plex`) |
| `/var/plex` | Transcode scratch | NVMe, ext4 |
| `/export/{tv,movies,music}` | Media, read by Plex **read-only** | ZFS |

`install` creates `/var/plex` owned by `plex:plex`, because `docker run` creates
a missing bind-mount source as `root:root` and the server process runs as uid
2004, so on a fresh machine it could not write transcodes.

One optional file lives on the dataset and is not in git:
`/tank/plex/image.env`, holding `PLEX_TAG` when a release is pinned. Absent in
the normal case. `install` renders it to `/etc/plex/image.env`, and that
generated copy is the only one systemd and `check-update` read — systemd opens
`EnvironmentFile=` in PID 1 itself, and `tank` is `failmode=wait`, so a pin file
on the pool would let a faulted pool wedge PID 1.

## Backups — read this one

**`/tank/plex` is the only irreplaceable thing this repo touches, and until
recently nothing was protecting it.** The `Media/` and `Metadata/` trees (about
70 GB of generated thumbnails and downloaded artwork) are expensive to rebuild
but not *lost*. The irreplaceable part is roughly 565 MB:
`com.plexapp.plugins.library.db`, its `blobs` companion, and `Preferences.xml`.
Between them they hold every view count, resume offset, rating, playlist,
collection and managed-user association, plus the server's `MachineIdentifier`.
None of it is derivable from the media files and none of it can be re-entered by
hand. Losing `Preferences.xml` specifically mints a new server identity, which
means plex.tv treats this as a different server: every shared-library invitation
has to be re-issued and re-accepted.

Three things are true about its protection, and they are easy to confuse:

**Local snapshots: yes, now.** `tank/plex` carries
`com.sun:auto-snapshot:daily=true` set *locally* (it must be local to override
the `false` inherited from `tank`, and a local property is what travels with a
`zfs send`). `install` sets it and takes a first snapshot when it *creates* the
dataset. The label is `daily` with keep=31, not the `monthly` the ntfy and
searxng repos use: this pool reserves monthly for cheap regenerable service
config and daily for user state, and watch history is user state.

**Plex's own database backups: yes, but they are not a backup.** Plex's Butler
task writes dated copies into `Plug-in Support/Databases/` every few days. Those
are genuinely useful — they cover database corruption and a failed schema
migration — but they sit on the same dataset, the same pool and the same
chassis. A directory full of rotating dated `.db` files is exactly what a backup
regime looks like, which is how the absence of one goes unnoticed.

**Offsite: no.** `/opt/zfs/zfs-replicate` runs `zfs-backup.ssh -x tank/plex`, so
this dataset is deliberately excluded from the nightly replication. **Do not
simply remove that exclusion.** The backup host already has a dataset named
`tank/plex` — it is a second, independent, *running* Plex server, not a copy of
this one — and `zfs-backup.ssh` receives with `zfs recv -eF`, which forces the
destination name. Removing the `-x` aims a full stream at a live filesystem.
The exclusion is correct; what was missing is this paragraph explaining why.

Getting the 565 MB offsite without touching that exclusion is the open item. The
cheapest correct approach is to copy the newest database pair and
`Preferences.xml` into `tank/backup`, which *is* replicated nightly.

## Media access is by the world bits, not the group

**Correcting a claim this README used to make.** It said Plex reads
`/export/{tv,movies,music}` through membership of the shared `media` group (gid
1500). It does not, and never did. The host `plex` account's group membership
never reaches the container: the image starts the server via `s6-setuidgid`,
which sets supplementary groups from the *image's* own `/etc/group`, where
`media` does not exist. Measured on the running server,
`/proc/<pms>/status` reports groups `44 100 993 2004` — no 1500. Adding
`--group-add 1500` to the container was tried and does not help; it reaches
container PID 1 and is then discarded by the same `setgroups()`.

What actually makes the library readable is that all three roots are
`drwxrwxr-x` and everything under them is world-readable. So the invariant to
protect is the **`o=rx` bit**, and `install` checks exactly that.

This matters because the old group-based check failed in both directions. A
tree at `radarr:media` mode `2770` passed every check while Plex saw nothing; a
tree at `sonarr:sonarr` mode `755` was read perfectly and produced a warning.
The remediation the old README suggested — `chgrp media` — fixed nothing, and
its natural follow-on of dropping the world bits would have emptied the library
while every check stayed green.

The realistic way to lose the bit is a umask change in a writing service: under
`umask 007` new files land `0660` and new directories `0770`, and Plex loses
them one at a time, silently. `install` reports drift rather than fixing it —
those trees belong to the services that write them. To fix:

```bash
sudo find /export/music -type f ! -perm -o=r  -exec chmod o+r  {} +
sudo find /export/music -type d ! -perm -o=rx -exec chmod o+rx {} +
```

The media mounts are `:ro`. Plex needs no write access and effectively had none
before, but only because `other` happens to get `r-x` — a single edit setting
`PLEX_GID=1500` would have handed it unlink rights over the whole library. A
read-only bind is enforced at the VFS level, so unlike permission bits it also
constrains container root.

## Recreating on a fresh machine

Prerequisites: Docker, ZFS with a pool named `tank`, systemd, `findmnt`,
`curl`, and the `media` group (created by the `nzb` install).

```bash
sudo git clone https://github.com/jpansarasa/plex.git /opt/plex
sudo /opt/plex/install
```

`install` creates the plex user, the dataset, the transcode dir, registers the
units and starts the container, and does not exit 0 until Plex actually answers.

It does **not** restore `/tank/plex`. Restore that first if you have it — the
unit now refuses to start without `Preferences.xml`, precisely so that a missing
restore fails loudly instead of silently creating a new server. Starting without
it means claiming a fresh server at <https://plex.tv/claim> and re-adding the
libraries, and accepting that watch history is gone.

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

`/run` is tmpfs, so that marker does not survive a reboot. Normally that is
right, because a reboot applies the staged image — but a reboot that did *not*
apply leaves the marker gone while the update is still staged.
`journalctl -u plex-update.service` is the authoritative record.

### Pinning a bad release

This matters more here than for the sibling services. Plex migrates the library
schema forward on first start of a new binary, and not every migration has a
rollback, so "just restart the old image" is not always a way back.

```bash
sudo sh -c 'echo PLEX_TAG=1.43.3.10828-00f62d37d > /tank/plex/image.env'
sudo /opt/plex/install      # re-renders /etc/plex/image.env, then restarts

# Un-pin once upstream supersedes the bad release
sudo rm /tank/plex/image.env
sudo /opt/plex/install
```

The pin source lives on the dataset so it travels with the pool; the rendered
copy is what systemd reads. While pinned, `check-update` tracks the pinned tag
and separately reports when upstream moves past it.

## Notes

- `plex.service` deliberately does **not** pull on start. Pulling is the update
  timer's job; a start that waits on the network makes boot order fragile.
- The container runs with `--rm`, so a stopped container leaves nothing behind.
  All persistent state is in `/tank/plex` and `/var/plex`.
- The container's PID 1 is still root — the image requires it to set
  `PLEX_UID`/`PLEX_GID` and to supervise — but it now runs with `cap_drop=ALL`
  plus a minimal re-add and `no-new-privileges`. The add list is not decorative:
  `KILL` is required because s6 runs as uid 0 and signals a process running as
  2004, and without it the container can only ever die by SIGKILL.
- `tank/plex` and the three media datasets carry `setuid=off devices=off`. A
  root container writing to a `setuid=on` bind mount can leave a setuid-root
  binary that any host account can execute; no media file or database has any
  business being setuid or a device node.

### Waiting, not skipping

Every prerequisite that can be *late* — the config dataset, the media datasets,
dockerd — is checked in a way that **fails and retries**, never one that skips.

`Condition*` directives (and a failed `Requires=`) abort the start *job*. The
unit never enters start, `Restart=` is never armed, it sits at `inactive (dead)`
with `Result=success`, and nothing ever re-evaluates it. This unit used to gate
on `ConditionPathIsDirectory=/tank/plex`, which is worse than useless: a **bare
mountpoint satisfies it**, so it passed in exactly the fail-open case it looked
like it was guarding — and for Plex that case means starting a brand-new
unclaimed server, serving the whole library to the network, reporting healthy.

So the dataset checks are `ExecStartPre=`, docker is `Wants=` rather than
`Requires=`, and `StartLimitIntervalSec=0` means it never gives up.

The cost: a unit that retries forever never reaches `failed`, so
`systemctl --failed` stays empty while Plex is down. Look instead at:

```bash
systemctl is-active plex            # "activating" while stuck
journalctl -p err -u plex -n 20     # the guards log why, every cycle
```

## Removing this service

Order matters — removing the tree first strands the container, because
`ExecStop` runs `docker stop` and the unit file is gone.

```bash
sudo systemctl disable --now plex.service plex-update.timer
sudo docker stop plex
sudo rm -f /etc/systemd/system/plex-update.service
sudo systemctl daemon-reload
sudo rm -rf /opt/plex /etc/plex
# The dataset holds the library database and server identity. Deliberate step:
# sudo zfs destroy -r tank/plex
```
