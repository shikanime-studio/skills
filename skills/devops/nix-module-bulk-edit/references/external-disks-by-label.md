# External disks by label on NixOS hosts

Use this pattern when a host receives one or more already-formatted external
disks and you want them mounted declaratively.

## Recommended approach

- Prefer `fileSystems` when the disks are already partitioned/formatted.
- Prefer `disko` only when NixOS should own partitioning and filesystem
  creation.
- Use stable identifiers:
  - `/dev/disk/by-label/<label>` when labels are already present and intentional
  - `/dev/disk/by-uuid/<uuid>` when labels may change or are not trustworthy
- Avoid `/dev/sdX` in host config.

## Safe mount options for removable disks

- `nofail` so boot does not fail if a disk is absent
- `x-systemd.automount` so mounts happen on first access
- `x-systemd.idle-timeout=600` to unmount after inactivity

## Example

```nix
fileSystems."/mnt/remilia" = {
  device = "/dev/disk/by-label/remilia";
  fsType = "xfs";
  options = [ "nofail" "x-systemd.automount" "x-systemd.idle-timeout=600" ];
};

fileSystems."/mnt/flandre" = {
  device = "/dev/disk/by-label/flandre";
  fsType = "xfs";
  options = [ "nofail" "x-systemd.automount" "x-systemd.idle-timeout=600" ];
};
```

## Verification

- Check labels with `lsblk -f` before editing.
- Re-read the host module after patching to confirm the stanza is in the right
  place.
- Validate syntax with `nix-instantiate --parse <file>` or a module build when
  practical.
