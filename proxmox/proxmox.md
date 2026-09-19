# Proxmox VE — Installation on ZFS

## Setup
- **Hypervisor:** Proxmox VE
- **Root filesystem:** ZFS
- **Disk layout:** single disk, no redundancy (ZFS "stripe" — no mirror/RAIDZ protection against a disk failure)
- **Encryption:** ZFS native encryption, added **after** installation as a separate step (see the linked ZFS encryption guide below for the exact procedure)

## How it was installed

1. Booted the official Proxmox VE installer ISO and, on the installation target screen, chose **ZFS** as the filesystem instead of the default ext4/LVM.
2. Selected the single disk as the pool target. With only one disk, the installer creates a ZFS pool with no redundancy (mirror/RAIDZ aren't possible) — the goal here wasn't protection against a disk failure, but data integrity and ZFS's advanced features.
3. Finished the base installation with a plain (unencrypted) ZFS root pool.
4. Installer-created datasets under the pool (e.g. `rpool/ROOT`, `rpool/data`), later used by Proxmox as storage for VM disks (as ZVOLs) and containers.

## Encryption, added afterward

After the base install was up and running, ZFS native encryption was applied to the pool as a separate step (rather than at pool creation). See **[ZFS encryption](ZFS%20encryption.md)**

## Why ZFS instead of ext4/LVM

- **Fast, cheap snapshots**: ZFS's copy-on-write snapshots are near-instant and don't need pre-reserved space like LVM thin volumes. Handy before system upgrades or risky changes — snapshot, try the change, roll back if something breaks.
- **End-to-end checksumming**: every block written is checksummed; ZFS catches silent data corruption that ext4/LVM wouldn't, and `zfs scrub` periodically verifies the integrity of everything on the pool.
- **Transparent compression**: real disk space savings with no manual effort, usually at negligible CPU cost with `lz4`.
- **Native encryption built into the filesystem**: rather than LUKS under ZFS (which adds another layer), ZFS native encryption works at the dataset level, so:
  - encrypted and unencrypted datasets can coexist in the same pool with different keys if needed;
  - it plays well with snapshots (snapshots of an encrypted dataset stay encrypted);
  - no separate LUKS layer to manage on top of ZFS.
- **send/receive**: `zfs send`/`zfs receive` enables efficient incremental backup and replication (including of encrypted datasets, via `zfs send -w` for raw replication without decrypting) — useful for backing up to a second host later without external tools.
- **Simple VM storage management**: Proxmox integrates ZFS as a native storage backend, so VM disks become ZVOLs with all the above benefits (snapshots, compression, checksumming) with no extra configuration.

## Trade-offs accepted with this setup

- **No hardware redundancy**: with a single disk, a physical disk failure means losing everything on the pool. ZFS here protects against *silent data corruption* (via checksums), not against the disk itself failing — an external backup is still needed for that (e.g. `zfs send` to another host/disk, or traditional Proxmox backups via PBS/vzdump).
- **Passphrase required at boot**: with encryption enabled, the system won't fully start on its own without the passphrase being entered (unless an automatic unlock via a keyfile is configured, with the corresponding security trade-off).
- **RAM overhead**: ZFS uses RAM for the ARC cache; on systems with limited RAM it may be necessary to cap `zfs_arc_max` to leave enough memory for VMs/LXC.

