# Target initramfs archive evidence

The maintainer supplied `lsinitrd -m -k "$(uname -r)"` output and a filtered archive-entry listing for both targets during issue #5's interview (Q24). This is evidence about files on disk, not proof of runtime behavior, exact embedded script contents, or which image was last booted. No key contents were provided.

| Target | Image | Dracut version |
| --- | --- | --- |
| First target | `/boot/initrd-7.2.6-1-default` | `112+suse.51.gf078a84-1` |
| Musashi | `/boot/initrd-7.1.8-1-default` | `112+suse.34.g35e16b7-1` |

## Common contents

Both module lists include `systemd`, `systemd-initrd`, `dracut-systemd`, `usrmount`, and `zfs`. Both archive listings contain:

- `etc/cmdline.d/20-root-dev.conf` and `etc/zfs/zpool.cache`.
- `usr/lib/systemd/system-generators/dracut-zfs-generator`.
- `usr/lib/dracut-zfs-lib.sh` and the `95-parse-zfs.sh`, `98-mount-zfs.sh`, and `90-zfs-load-key.sh` dracut hooks.
- `zfs-import-cache.service`, `zfs-import-scan.service`, and `zfs-env-bootfs.service` with initrd dependencies.
- `zfs-nonroot-necessities.service`, required by `initrd-root-fs.target`.
- `zfs-snapshot-bootfs.service` and `zfs-rollback-bootfs.service`, wanted by `initrd.target`. Their presence alone does not establish whether their trigger conditions are satisfied.

Neither filtered listing contains `zfs-mount-generator` or any `zfs-list.cache` path. The filter would include those names. `dracut-zfs-generator` is a different component, and `zpool.cache` is a pool-import cache, not the mount generator's dataset cache.

Musashi's archive additionally contains `etc/zfs/rpool.key`, mode `0600`. This confirms filename presence and archive permissions, improving on the earlier observation that dracut configuration only requested inclusion. It does not verify the key's contents, validity, or successful unlocking after cloning.

## Interpretation and limits

The observed separation supports considering per-environment host-side mount-generator caches without assuming those caches are already embedded in these initramfs images. Private mounting still needs explicit isolation in the host, and installed or embedded initrd hooks must not bypass it. Boot tests must cover direct alternate selection, required-mount failure, encrypted unlocking, rename, rollback, and subsequent initramfs regeneration.

Both commands warned that an EFI system partition/XBOOTLDR could not be located, but successfully identified and listed the explicit `/boot/initrd-*` images above. This output does not establish the deployed ZFSBootMenu image or its effective preferences.
